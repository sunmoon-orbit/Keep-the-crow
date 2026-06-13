# 05 — PWA 前端（聊天界面）

## 技术选型

前端尽量保持简单：

- 单 HTML 文件（也可以用 React/Vue，看个人习惯）
- Service Worker（满足 PWA 安装条件）
- WebSocket 连接 raven-bridge
- 可以托管到 GitHub Pages（免费、零运维）

---

## 关键功能点

### WebSocket 连接

```javascript
let ws
let reconnectTimer = null

function connect() {
  ws = new WebSocket(`wss://your-domain.com/raven-ws?token=${TOKEN}`)

  ws.onopen = () => {
    console.log('connected')
    if (reconnectTimer) {
      clearTimeout(reconnectTimer)
      reconnectTimer = null
    }
  }

  ws.onmessage = (e) => {
    const msg = JSON.parse(e.data)
    if (msg.type === 'message') {
      appendMessage(msg)
    }
  }

  ws.onclose = () => {
    // 断线重连
    reconnectTimer = setTimeout(connect, 3000)
  }
}

connect()
```

### 手机后台返回前台时重连

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    if (!ws || ws.readyState === WebSocket.CLOSED || ws.readyState === WebSocket.CLOSING) {
      if (reconnectTimer) {
        clearTimeout(reconnectTimer)
        reconnectTimer = null
      }
      connect()
    }
  }
})
```

### 发送消息

```javascript
function sendMessage(text) {
  // 加上前缀，让 raven-bridge / CC 识别这是用户消息
  const formatted = `【你的名字】${text}`
  ws.send(JSON.stringify({ type: 'chat', text: formatted }))
}
```

---

## PWA 配置

### manifest.json

```json
{
  "name": "你的 AI 伴侣",
  "short_name": "AI",
  "start_url": ".",
  "scope": ".",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#ffffff",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**注意：** `theme_color` 控制 Android 状态栏颜色，PWA 安装后改这个值会触发 WebAPK 重新铸造（Android Chrome），期间图标可能暂时恢复成 Chrome logo，等一两天会自动恢复。**尽量一开始就定好颜色，之后不要改。**

### sw.js（最简版）

```javascript
self.addEventListener('install', () => self.skipWaiting())
self.addEventListener('activate', () => self.clients.claim())
self.addEventListener('fetch', (e) => e.respondWith(fetch(e.request)))
```

---

## 推送通知

接收 AI 主动发来的推送，需要：

1. 在前端申请推送权限，获取 subscription
2. 把 subscription 发到服务器保存
3. 服务器用 web-push 库发送推送

```javascript
// 前端：申请推送权限
const reg = await navigator.serviceWorker.ready
const sub = await reg.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: VAPID_PUBLIC_KEY
})

// 把 sub 发给服务器保存
await fetch('/push/subscribe', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json',
             'Authorization': `Bearer ${TOKEN}` },
  body: JSON.stringify(sub)
})
```

```javascript
// sw.js：收到推送时显示通知
self.addEventListener('push', (e) => {
  const data = e.data.json()
  e.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/icon-192.png',
      vibrate: [200, 100, 200]
    })
  )
})
```

VAPID 密钥生成：

```bash
npx web-push generate-vapid-keys
```

---

## 托管到 GitHub Pages

1. 把前端文件放到仓库的 `/docs` 目录（或独立分支）
2. GitHub 仓库设置 → Pages → Source 选对应目录
3. 推送后自动部署，几分钟生效

免费、自动 HTTPS，无需服务器。只有静态文件放这里，WebSocket 还是连到你的服务器。

---

## 贴图支持

在前端加一个贴图选择器，格式用 `[sticker:filename.png]`，收到消息时渲染成图片：

```javascript
function renderMessage(text) {
  return text.replace(
    /\[sticker:([^\]]+)\]/g,
    (_, name) => `<img src="/stickers/${name}" style="max-width:200px;">`
  )
}
```

贴图文件放服务器上（或一起托管到 GitHub Pages），这样手机 PWA 和 AI 回复都能用同一套贴图。
