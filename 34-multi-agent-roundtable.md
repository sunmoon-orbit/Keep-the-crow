# 34 — 多 Agent 圆桌：一条消息，两份投递，有限次接力

把两个 Agent 拉进同一个聊天页，难点不在画三个头像，而在于回答四个工程问题：消息由谁保存、每个 Agent 如何独立消费、掉线重启后怎样续上，以及怎样阻止两个 Agent 互相 `@` 到永远。

本篇给出一套适合小型 VPS 的实现。示例角色统一写成 `user`、`agent_a`、`agent_b`，不依赖具体模型厂商。

---

## 一、消息和投递必须拆成两张表

一条用户消息可能同时叫醒两个 Agent，但它仍然只是一条消息。不要为每个消费者复制正文；把不可变消息和可变投递状态分开：

```sql
CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  sender TEXT NOT NULL,
  text TEXT NOT NULL,
  root_id TEXT NOT NULL,
  reply_to TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE deliveries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  message_id TEXT NOT NULL,
  target TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  attempts INTEGER NOT NULL DEFAULT 0,
  lease_until TEXT,
  runtime_thread_id TEXT,
  runtime_turn_id TEXT,
  last_error TEXT DEFAULT '',
  UNIQUE(message_id, target)
);
```

用户不点名时可以创建两份 delivery；只 `@agent_a` 时只建一份。Agent 的普通回复只上桌，不自动叫醒另一位；只有显式点名才继续接力。

这层分离带来三个好处：

- 一个 Agent 忙或离线，不影响另一个先回答；
- UI 只按消息 id 去重，不会看到两份正文；
- 每条投递可以独立重试、阻塞、恢复和统计排队数。

---

## 二、幂等不能只靠“文本一样”

手机断线后会重发 outbox，Agent 完成回调也可能因 bridge 重启重复送达。至少准备两层去重：

1. **请求幂等键**：客户端每次发送生成随机 `cid`，服务端以 `(sender, cid)` 建唯一约束；
2. **Agent 回复指纹**：对 `sender + root_id + 规范化正文 + 附件 id + reply_to` 做哈希，挡住同一轮被不同恢复路径重复落库。

文本规范化只用于生成指纹，原文仍按原样保存：

```js
function normalizeForFingerprint(text) {
  return String(text || '')
    .normalize('NFKC')
    .replace(/\s+/g, ' ')
    .trim()
}
```

不要只按正文全局去重。同一句“收到”在不同话题里完全可能都是合法回复，指纹必须带稳定的 `root_id`。

---

## 三、消费队列要用 lease，不要“取出即删除”

worker 领取投递时，把它从 `pending` 原子地改成 `leased`，并写一个短租约：

```sql
SELECT * FROM deliveries
WHERE target = ?
  AND (status = 'pending' OR (status = 'leased' AND lease_until < ?))
ORDER BY id ASC
LIMIT 1;
```

之后的状态可以是：

```text
pending → leased → submitted/running → done
                  ↘ blocked / failed
```

- bridge 在把任务真正交给 runtime 后才标 `running`；
- 保存 runtime 的 thread id 和 turn id；
- bridge 重启后读取仍在 `running` 的记录，向 runtime 查询结果；
- 找到最终回复就幂等落库，找不到则标 `blocked`，不要猜成成功或盲目重跑。

盲目重跑可能重复执行命令、重复写文件，也可能让按次或按量模型重复计费。

---

## 四、两个 Agent 互相点名必须有熔断器

如果 Agent A 的回复点名 B，B 又点名 A，一个普通话题就会变成无限自动对话。给每个 `root_id` 保存 `ai_wake_count`：

```js
if (sender !== 'user' && targets.length) {
  const used = wakeCount(rootId)
  if (used >= MAX_AI_WAKES) {
    targets = []                  // 回复照常上桌，但不再创建投递
    appendSystemNotice(rootId)
  } else {
    incrementWakeCount(rootId)
  }
}
```

这里限制的是“AI 叫醒 AI”的次数，不是消息总数。用户再说一句时会创建新的 root，并从零开始计数。

上限不宜藏在 prompt 里。prompt 只能提醒，真正的熔断必须在数据库事务里完成，否则并发写入时两个 worker 都可能以为自己还没超限。

---

## 五、忙时不要把七条排队消息回答七遍

Agent 忙碌期间，用户可能继续补充、纠正或换话题。它空下来后若逐条开七个 turn，会浪费额度，还会产生彼此过时的回答。

可以在同一目标的 worker 空闲时，一次领取一个有界批次，例如最多 10 条：

```text
你忙的时候排了 4 条，按时间顺序合并后统一回复：
1. user：补充条件……
2. agent_a：已经查到……
3. user：不用继续查那个……
4. user：改看这一项……
```

回复挂在批次最后一条上，批次内所有 delivery 共用同一 runtime turn，并一起进入 `running/done`。附件也要按原顺序合并，但仍受数量和大小上限约束。

合并是交互层策略，不应破坏消息存档。数据库里仍保留原来的四条消息，只有送给模型的 envelope 被临时合并。

---

## 六、同桌状态要表达“在哪里忙”，不只表达在线

对用户有用的状态至少包括：

- `idle`：在线且没有工作；
- `queued`：已有待处理投递；
- `working`：正在圆桌或另一个入口执行；
- `waiting_approval`：卡在用户授权；
- `reconnecting`：runtime 断线但仍在宽限期；
- `standby`：为省内存尚未拉起，收到消息即可启动；
- `offline`：当前确实无法消费任务。

把懒启动写成“离线”会制造错误警报。小 VPS 上让重型 Agent 待命、按需拉起是正常优化，UI 应诚实区分 `standby` 与故障。

同样，状态里最好带工作表面：正在圆桌、终端还是另一个聊天入口。另一个 Agent 看见同桌已经在做同一件事时，应补充或审阅，而不是重复开工。

---

## 七、实时消息必须有 HTTP 补缝

只靠 WebSocket 会遇到一个经典缝隙：客户端先拉历史，再订阅实时流，两步之间刚好写入的消息两边都看不到。

可靠顺序是：

1. 浏览器保存最后看到的稳定消息 id；
2. WebSocket 认证后订阅 `after=<last_id>`；
3. 同时用 HTTP 再拉一次增量；
4. 前端按消息 id 去重；
5. 本地 outbox 在断线后带原 `cid` 重发。

审批卡也要在重连时重建。先清掉旧连接留下的卡，再由服务端重发仍有效的审批；审批响应必须同时匹配 request、thread、turn 和 item id，不能只凭一段命令文本。

---

## 八、附件只传引用，模型侧再做受控解析

浏览器消息里只保存：

```json
{
  "id": "random-file-id",
  "name": "photo.jpg",
  "mime": "image/jpeg",
  "size": 123456,
  "url": "/uploads/random-file-id"
}
```

不要把服务器绝对路径送回浏览器，也不要相信客户端传来的路径。bridge 给 Agent 投递前只取 `id` 的 basename，再限定到唯一上传目录，确认文件真实存在。

不同 runtime 的附件能力可能不同：一个接受本地图片对象，另一个只理解文本路径。适配器应分别投递；图片能力失败时可以降级为“有附件但当前模型未读取”，不能把整条消息卡死，也不能让服务器代抓任意远程 URL。

---

## 九、手机端的小交互也要共享状态

圆桌和私聊若在同一站点，可以复用同一组本地收藏与使用次数：

- 贴图按钮插入受白名单约束的 `[sticker:file]` 标记；
- 颜文字通过 `textContent` 渲染，不能拼进 `innerHTML`；
- 自定义颜文字设长度与数量上限；
- 长按消息的 reaction 用稳定消息 id 做 key，而不是正文；
- 手机长按计时器在移动超过阈值、滚动、抬手或取消时立即清掉，避免用户滚列表时误触。

贴图和 reaction 是两种语义：前者是一条消息，可能叫醒 Agent；后者是附着在旧消息上的轻量反馈，通常不创建新 turn。

---

## 最小验收清单

- [ ] 一条用户消息只存一次，对两个 Agent 生成独立 delivery
- [ ] WebSocket 重发同一 `cid` 不会重复建消息
- [ ] Agent 恢复路径重复回调不会重复上桌
- [ ] lease 过期可重新领取，`running` 可在 bridge 重启后对账
- [ ] AI 互相点名达到上限后只上桌、不再叫醒
- [ ] 忙时多条消息合成一次模型调用，原始消息仍分别保存
- [ ] `standby`、`offline`、`waiting_approval` 与其他入口工作状态可区分
- [ ] 历史补拉与实时订阅之间没有漏消息，前端按 id 去重
- [ ] 审批响应绑定完整 runtime 身份，重连不残留旧卡
- [ ] 附件不向浏览器暴露绝对路径，不接受任意路径或任意远程 URL
- [ ] reaction 不创建新对话轮，移动/滚动不会误触长按
