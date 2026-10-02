# 13 — 安全加固：你的桥接服务大概率在裸奔

## 为什么有这篇

照着 03-raven-bridge 搭出来的桥接服务，最初的版本是**没有鉴权**的——我们自己的也是。跑了一个多月之后做了一次完整审计，发现了五个公网可达的洞，其中最严重的一个能让任何陌生人**实时偷听你和 AI 的全部对话**。

如果你已经把系统跑起来了，这篇按顺序过一遍。全部修完大约一个下午。

---

## 先说基础面：这些应该早就做了

审计不是从应用层开始的。先确认服务器本身：

```bash
# 防火墙：默认拒入站，公网只开 80/443
sudo ufw status verbose

# SSH：禁密码、禁 root、只走 Tailscale 网段
grep -E "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config

# fail2ban 在跑
systemctl status fail2ban

# 自动安全更新
systemctl status unattended-upgrades

# 依赖漏洞
cd raven-bridge && npm audit
```

这一层没问题了，才轮到应用层。下面是我们真实踩到的五个洞，按严重程度排序。

---

## 洞一：WebSocket 偷听（最严重）

**问题**：桥接服务的 WS 端点谁都能连。连上的瞬间，服务端热情地把「终端最后 80 行输出 + 最近 10 条 AI 回复」作为欢迎数据发过去，之后的每条广播也照发不误。任何知道你域名的人都可以挂一个 WS 客户端，实时旁观你和 AI 的所有对话。

**修法**：认证先行，未认证的连接什么都收不到。

```javascript
// 连接建立后不发任何数据，等首条 auth 消息
ws.on('message', (raw) => {
  const msg = JSON.parse(raw)
  if (!ws.authed) {
    if (msg.type === 'auth' && validTokens.has(msg.token)) {
      ws.authed = true
      clients.add(ws)          // 认证通过才进广播列表
      sendWelcome(ws)          // 才发欢迎数据
    }
    return                     // 未认证的消息一律丢弃
  }
  // ...正常处理
})

// 15 秒不认证直接断开，防止挂着占连接
setTimeout(() => { if (!ws.authed) ws.terminate() }, 15000)
```

广播函数同样只发给 `clients` 集合（已认证的），不要用 `wss.clients`（全部连接）。

前端配合：`onopen` 里第一件事发 `{type:'auth', token}`。注意一个坑——**用户开着的旧页面**重连后收不到广播（旧 JS 不会发 auth），需要刷新一次。上线这个改动时提前说一声，省得对方以为坏了。

---

## 洞二：回复接口冒充

**问题**：`/raven/reply` 是 AI 通过本机 curl 给前端发消息用的。它没鉴权时，任何外人都可以 POST 一条消息**冒充你的 AI** 跟你说话——还会触发推送通知，可信度拉满。同理还有 `/thinking`、`/mcp/*` 这类内部接口。

**修法**：这类接口只应该被本机调用，用一个很省事的判定：

```javascript
// 经 Caddy/Nginx 反代进来的请求必带 X-Forwarded-For；
// 本机直连（AI 的 curl、hooks）没有这个头
function isLocalDirect(req) {
  return !req.headers['x-forwarded-for']
}

if (!isLocalDirect(req)) return res.writeHead(403).end()
```

前提是这些端口（3400/3210）**没有直接暴露公网**，外部流量只能从反代进来——ufw 那一层保证了这一点。两层配合，判定才成立。

---

## 洞三：记忆接口裸奔

**问题**：`/raven/memory-random` 这类给前端首页展示「随机记忆」的接口没鉴权，返回的是记忆库原文。你和 AI 之间最私密的数据，公网可拉。

**修法**：所有会返回内容的接口统一挂 token：

```javascript
function requireToken(req, res) {
  if (isLocalDirect(req)) return true                    // 本机直连放行
  const auth = req.headers.authorization || ''
  const token = auth.replace(/^Bearer /, '') || parseQuery(req).token
  if (validTokens.has(token)) return true
  res.writeHead(401).end()
  return false
}
```

支持 `?token=` 是给 `<img>`、`<audio>` 这类没法带请求头的标签留的口子。

---

## 洞四：推送订阅劫持

**问题**：`/push/subscribe` 无鉴权。外人可以把**自己的**推送端点注册进你的订阅列表，从此你的 AI 每发一条通知，他的设备也收到一份。这是最容易被忽略的一类洞——订阅接口看起来「只是写入」，但写入的是收件人列表。

**修法**：同洞三，token 化。subscribe 和 unsubscribe 都要。

---

## 洞五：审计本身炸出来的老 bug

审计时用 curl 逐个打接口，打到 `/raven/health` 时进程直接崩了。排查发现处理函数写完响应后**漏了 return**，代码继续往下掉进静态文件处理器，二次写响应头，`ERR_HTTP_HEADERS_SENT` 未捕获直接炸进程。

这个 bug 存在了几个月没暴露——因为从来没人打过这个接口。两个教训：

1. Node 的 http 处理链里，`res.end()` 之后必须 `return`，这类漏写靠 code review 很难扫出来，靠打一遍全部接口很容易；
2. **审计的副产品往往是稳定性 bug**。把「用 curl 把所有路由打一遍」当作常规体检，不只是安全动作。

---

## 收尾杂项

```bash
chmod 600 .valid-tokens.json     # token 持久化文件别给组/其他人读
# sshd_config: X11Forwarding no  # 用不上的转发一律关
```

## 后来补的两条

**花钱的接口一个都不能免鉴权。** 我们有过一个免 token 的朗读接口，只按 IP 限速。换 IP 就能绕过去，一个人一小时就能烧光整个月的免费额度。修法是让它先过前端自己的登录验证，再由服务端转发到真正花钱的那条路。关接口之前，先 `grep` 一遍谁在调用它，日志里 0 次调用不等于没人在用。

**想把仓库开源，先查提交历史，不是只查当前文件。** 私有仓库常常兼做备份：每天把数据快照推上去，注释和配置里也写着真实的名字。当前文件删干净了，历史里还在，一公开就全部出去，事后删提交也收不回来。正确做法是另起一个**没有历史**的新仓库，只放代码，换成示例数据，再附一份隐私说明。

## 验证：修完必须打一遍

写一个断言脚本，把每个接口按「外网无 token / 外网带 token / 本机直连」三种身份各打一遍，预期分别是 401(403) / 200 / 200。我们的版本是 17 项断言，全绿才算完。

WS 单独测：不带 auth 连上挂 4 秒，收到 0 条消息才算堵住；带 token 应正常收到欢迎数据。

---

## 心法

- **默认所有接口都是公网可达的**，除非你验证过不是。「这个接口没人知道」不是防护。
- 鉴权分三级想：公开（真正无害的，如 /health 探活）、token（用户的前端）、仅本机（AI 与桥接的内部通道）。每个路由明确属于哪一级。
- 洞一到洞四有个共同点：**都是功能上线时「先跑通再说」留下的**。跑通和上线之间，差的就是这篇。
