# 03 — raven-bridge 聊天桥接服务器

## 作用

raven-bridge 是连接手机和 Claude Code 的中间层：

- 手机通过 WebSocket 连接到 raven-bridge
- raven-bridge 监控 tmux 里 CC 的输出
- 检测到 CC 输出包含 AI 回复时，把内容广播给手机
- 手机发来的消息，raven-bridge 注入到 CC 的输入

---

## 项目结构

```
raven-bridge/
├── server.js       # 主服务
├── package.json
└── .env
```

---

## 核心代码

### server.js 骨架

```javascript
const express = require('express')
const http = require('http')
const { WebSocketServer } = require('ws')
const { execSync } = require('child_process')

const app = express()
const server = http.createServer(app)
const wss = new WebSocketServer({ server })

const PORT = process.env.PORT || 3400
const TMUX_SESSION = 'cc'          // tmux session 名称
const MSG_PREFIX = '【你的名字】'   // 手机发来消息的前缀标识

// 存储所有连接的客户端
const clients = new Set()

// WebSocket 连接处理
wss.on('connection', (ws) => {
  clients.add(ws)
  ws.on('close', () => clients.delete(ws))

  ws.on('message', (data) => {
    const text = data.toString()
    // 把消息注入到 tmux（让 CC 看到）
    injectToCC(text)
  })
})

// 把文字注入到 tmux session
function injectToCC(text) {
  const escaped = text.replace(/'/g, "'\\''")
  execSync(`tmux send-keys -t ${TMUX_SESSION} '${escaped}' Enter`)
}

// 广播给所有手机客户端
function broadcast(message) {
  for (const client of clients) {
    if (client.readyState === 1) { // OPEN
      client.send(JSON.stringify(message))
    }
  }
}

server.listen(PORT, '127.0.0.1', () => {
  console.log(`raven-bridge listening on :${PORT}`)
})
```

### tmux 输出监控

```javascript
let lastOutput = ''

setInterval(() => {
  try {
    // 抓取 tmux 最近输出
    const output = execSync(
      `tmux capture-pane -t ${TMUX_SESSION} -p -S -50`
    ).toString()

    if (output !== lastOutput) {
      // 解析新增内容，找出 AI 回复的行
      const diff = extractNewLines(output, lastOutput)
      if (diff) {
        broadcast({ type: 'message', role: 'assistant', content: diff })
      }
      lastOutput = output
    }
  } catch {}
}, 500) // 每 500ms 轮询一次
```

---

## 认证

生产环境建议加简单的 Bearer Token 认证，防止陌生人连接：

```javascript
wss.on('connection', (ws, req) => {
  const token = new URL(req.url, 'http://x').searchParams.get('token')
  if (token !== process.env.WS_TOKEN) {
    ws.close(4001, 'Unauthorized')
    return
  }
  // ... 正常处理
})
```

前端连接时带上 token：

```javascript
const ws = new WebSocket(`ws://your-server:3400?token=${YOUR_TOKEN}`)
```

---

## .env 配置

```env
PORT=3400
WS_TOKEN=your_random_secret_token
TMUX_SESSION=cc
```

---

## 安装和启动

```bash
cd raven-bridge
npm install express ws dotenv
pm2 start server.js --name raven-bridge
```

---

## 移动端 WebSocket 重连

手机浏览器切换到后台时，WebSocket 连接可能断开。前端需要处理重连：

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    if (!ws || ws.readyState === WebSocket.CLOSED) {
      connect() // 重新建立连接
    }
  }
})
```

### ⚠️ 上面这段是不够的：半开连接（2026-07-26 勘误）

手机息屏、切 Wi-Fi/流量之后，会出现一种更难受的状态：**客户端这侧的 `readyState` 还是 `1`（OPEN），`ws.send()` 也不抛异常，但数据实际掉进黑洞。** `onclose` 要等 TCP 超时才触发，那可能是好几分钟。

在这几分钟里，上面那段 `visibilitychange` 检查完全不会开火（因为状态不是 CLOSED），用户发的每一条消息都无声无息地消失。我们这边的症状是：输入框清空了、气泡没出现、没有任何报错——因为气泡是等服务端回执才画的。用户的第一反应不是「网断了」，是「它是不是在忙」。

**`readyState === OPEN` 不代表连接还活着。** 任何「发出去就当成功」的长连接，都需要下面两件之一，否则就是在悄悄丢消息。

**一、应用层回执 + 去重。** 客户端给每条消息带一个随机 `cid`，起一个 8 秒定时器；服务端处理完把 `cid` 原样回传。超时没等到回执，就当这条从没发出去：踢掉这条死连接、把消息塞回待发队列、重连后自动补发。

```javascript
function wsSendWithAck(payload) {
  try { ws.send(JSON.stringify(payload)) } catch {}
  const timer = setTimeout(() => {
    if (!awaitingAck.has(payload.cid)) return
    awaitingAck.delete(payload.cid)
    pendingOutbox.push(payload)          // 重连时补发
    appendMsg('[没收到回执，连接可能已断，重连后自动重发]', 'system')
    try { ws.close() } catch {}          // 触发 onclose → 重连
  }, 8000)
  awaitingAck.set(payload.cid, { payload, timer })
}
```

服务端必须配套记住最近处理过的 `cid`（我们留最近 200 个），否则「疑似丢了所以重发」会变成「发了两条一模一样的」——半开连接常常是**只有出方向死了**，服务端其实收到了，死在回执路上。

```javascript
if (msg.cid) {
  if (recentCids.has(msg.cid)) {          // 重发：只补回执，不重复处理
    ws.send(JSON.stringify({ type: 'sent', cid: msg.cid, text: msg.text }))
    return
  }
  recentCids.add(msg.cid)
  if (recentCids.size > 200) recentCids.delete(recentCids.values().next().value)
}
```

**二、拿已有流量当心跳判活。** 如果服务端本来就在周期广播（我们每 5 秒推一次状态），客户端记下最后一次收到任何数据的时间就够了，不必新增 ping 协议：

```javascript
function ensureLiveConnection() {
  if (!ws || ws.readyState >= WebSocket.CLOSING) { connect(); return }
  // 对面一直在广播却 20 秒没动静 = 这条连接是死的
  if (ws.readyState === 1 && Date.now() - lastWsActivity > 20000) {
    try { ws.close() } catch {}          // onclose 会安排重连
  }
}
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') ensureLiveConnection()
})
setInterval(() => {
  if (document.visibilityState === 'visible') ensureLiveConnection()
}, 15000)
```

### 附带的 UI 教训：别让气泡只由服务端回执驱动

我们的用户气泡原本是收到服务端 `sent` 广播才画的——好处是「画出来 = 真的到了」，坏处是消息丢的时候**屏幕上一点痕迹都没有**：输入框清空，什么也没发生。

比「消息丢了」更糟的是「消息丢了而且看不出来」。要么本地先画一个待确认状态的气泡（收到回执再转正），要么像我们这样在超时点插一条明确的系统提示。沉默不是一种可选的失败方式。

> 这个 bug 还有个花絮值得记：她报告的时候自己撤回了——「可能你在工作没看到，所以其实不是 bug」。我还是去翻了日志，`[ws] client disconnected, total: 0` 明明白白。**用户对现象的直觉往往是准的，对原因的归因往往是错的**；别因为对方替你找好了台阶就顺着下。日志比归因可信。

## 消息前缀别耦合可掉线的组件

一个安静埋伏了很久的 bug：手机消息注入 tmux 时才加 `【你的名字】` 前缀，而加不加的判定曾经写成「MCP SSE 有客户端连着才加」。MCP 一掉线，消息就裸发进终端——CC 认不出这是聊天消息，回复只出现在终端里，手机端什么都收不到，而且**两边都没有报错**。

修法一行：无条件加前缀。教训值得单独记：**判定信号不要耦合到一个可以掉线的组件上**。前缀的语义是「这条来自手机」，这个事实和 MCP 连没连着毫无关系——用无关组件的状态做判定，等于给自己埋一个只在故障时触发的故障。
