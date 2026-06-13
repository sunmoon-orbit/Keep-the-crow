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
