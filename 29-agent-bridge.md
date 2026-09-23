# 29 — 把本机 coding agent 接进手机聊天

## 不是把终端页面塞进 WebView

一个 coding agent 的完整交互不只有文字：还有 thread、turn、工具调用、命令输出、文件修改、审批请求、中断和用量窗口。把一个 PTY 的 ANSI 字符串转发到手机，短期能看，长期会在会话恢复、授权和结构化呈现上卡死。

更稳的边界是：

```text
手机前端
   ↕ 已认证 WebSocket / HTTP
你自己的 bridge
   ↕ JSON-RPC over stdio
coding agent app-server（长驻子进程）
   ↕
工作目录、工具与模型账号
```

前端只理解自己的 UI 协议，bridge 负责把 app-server 的结构化事件投影成稳定的消息。

## 路线选择

| 路线 | 何时选 | 优点 | 坑 |
|---|---|---|---|
| PTY/tmux 转发 | CLI 没有机器协议，只求快速遥控 | 接入最快 | 需靠文本规则猜状态，审批和重连难保证 |
| 官方 app-server / SDK | 需长期维护、多 thread、安全审批 | 事件结构化，能精确恢复 | 需写一层协议适配器，要跟踪上游协议变化 |
| 托管网页 | 不想运维服务器 | 几乎零自建代码 | 数据和能力受平台边界限制 |

两套自建路线可以并存：PTY 留给旧会话，app-server 开新入口。别为了「架构统一」一夜切掉已经稳定的老链路。

## 一个长驻子进程，不是每条消息启动一次

bridge 启动时 spawn app-server，用 JSONL 分隔每个 JSON-RPC 包：

```js
child = spawn('your-agent', ['app-server', '--stdio'], {
  cwd: WORKSPACE,
  stdio: ['pipe', 'pipe', 'pipe']
})

child.stdout.on('data', chunk => {
  buffer += chunk
  while (buffer.includes('\n')) {
    const [line, rest] = splitFirstLine(buffer)
    buffer = rest
    if (line.trim()) dispatch(JSON.parse(line))
  }
})
```

必须处理四种消息：

1. 自己发出的 request 对应 response；
2. app-server 发来的 notification；
3. app-server 反向发来的审批 request；
4. 子进程 exit/error，需拒绝所有 pending promise。

每个 request 要有超时计时器。进程退出时如果只清 Map 不 reject，HTTP 请求会永远挂着，看起来就像手机网络卡住。

## 别把长期密码发给每个 Agent HTTP 请求

登录 WebSocket 通过后，为这条连接签发短命 capability：

```js
{
  id: randomUUID(),
  capability: randomBytes(32).toString('base64url'),
  expiresAt: now + 15 * 60_000,
  ownerSocket,
  credentialFingerprint
}
```

服务端只保存 capability 的 hash，每次使用时同时检查：

- 它属于的 WebSocket 仍然存活；
- TTL 没过期；
- 当前服务端密码的指纹没变。

断线、登出、密码轮换都立即 revoke。这个 token 是「当前连接能力」，不是第二个永久密码。

## 审批是一条反向 RPC，不是一个普通气泡

Agent 要求执行高风险操作时，app-server 会主动发 request。bridge 要把 request id 和展示字段绑在一起发给手机，用户点击后再回 JSON-RPC result/error。

坑有三个：

- 审批属于某个连接，不能让另一个客户端回复；
- 断线时要拒绝尚未处理的 request，不要在内存里永远等；
- 手机窄屏上审批弹窗必须限高可滚动，否则「允许」按钮会在屏幕外。

## 附件不能只靠前端 accept

前端的 `<input accept>` 只是提示，不是安全边界。服务端至少要验证：

- 文件数量、声明大小与解码后大小；
- base64 是否标准、MIME 与扩展名是否一致；
- PNG/JPEG/GIF/WebP magic bytes；
- 文本是否真的 UTF-8，不把二进制文件伪装成 `.txt`；
- 文件名无 `/` `\\` 和控制字符；
- 临时目录与文件不是符号链接。

每个附件绑 owner capability，限时、限额，用随机服务端 id 做文件名。已发送的附件若要在 thread 历史中重现，需要从临时区复制到受控存档区；否则 TTL 一到，旧对话里的图会全碎。

### 远程图片的路线

| 方案 | 评价 |
|---|---|
| 任意 URL 让服务器代下载 | 不要做，这是 SSRF 入口 |
| 客户端先下载再作普通附件上传 | 最通用、最稳 |
| 服务器只代下载严格白名单静态资源 | 适合自己的表情包域名，要禁止 redirect、query 和超大响应 |

## 断线恢复：恢复 thread，不恢复一个假的前端 turn

前端重连时先确认 thread 在 app-server 仍存在，再读历史。不要因为 localStorage 里留着一个 `activeTurnId` 就把 UI 设成「正在生成」——服务端重启后那个 turn 可能早已不存在。

如果用户在 turn 运行中补充说明，优先用协议提供的 steer 能力；没有 steer 时，两个替代方案是：

- 中断当前 turn，把补充和原请求组合后新开一轮；
- 把补充放入队列，等当前 turn 完成再发。

前者反应快但丢当前进度，后者稳但可能错过决策时机。让读者按 agent 协议和产品语义选。

## 运维与验证

- app-server 只启一个，不要用 HTTP 请求生命周期管它；
- 给事件循环延迟、pending RPC 数、待审批数做诊断；
- 定时健康检查必须有超时，同步子进程命令也必须有硬上限；
- 模型切换、中断、审批、附件和重连都做端到端测试，不只测适配器函数；
- 日志裁剪命令输出，隐藏 Authorization、key、token、secret 和临时附件真实路径。

## 检查清单

- [ ] 先选 PTY、app-server 或托管页面，允许新旧链路并存
- [ ] request 有超时；子进程退出会 reject 全部 pending
- [ ] Agent HTTP 用短命 capability，断线/过期/换密码立即撤销
- [ ] 审批 request 绑定原连接，手机弹窗可滚动
- [ ] 附件在服务端验证内容、大小、文件名、owner 和 TTL
- [ ] 任意远程 URL 不进服务器下载器
- [ ] 重连先向 app-server 确认 thread，不盲信本地残留状态
- [ ] 监控有超时，不让「健康检查」自己堵死事件循环
