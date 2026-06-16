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
    (_, name) => `<img src="/raven/stickers/${name}" style="max-width:200px;">`
  )
}
```

贴图文件放服务器上（`raven/stickers/` 目录），由 raven-bridge 静态托管，AI 和用户都可以发。

### 贴图选择器最佳实践

贴图数量多时，选择器面板必须限高 + 滚动，否则会撑满全屏：

```css
#sticker-picker {
  max-height: 260px;
  overflow-y: auto;
  /* 美化滚动条 */
  scrollbar-width: thin;
  scrollbar-color: var(--accent) transparent;
}
#sticker-picker::-webkit-scrollbar { width: 3px; }
#sticker-picker::-webkit-scrollbar-track { background: transparent; }
#sticker-picker::-webkit-scrollbar-thumb { background: var(--accent); border-radius: 99px; }
```

选择器用 5 列网格，贴图文件懒加载（`loading="lazy"`），几十张不影响初始加载速度。

### WebAPK 注意事项（Android PWA）

`manifest.json` 是 WebAPK 的身份文件。**任何字段改动**（哪怕一个字）都会触发 Google 服务器重新铸造 WebAPK，铸造期间（几小时到几天）推送通知的图标会退回 Chrome logo。

避免触发重铸：
- 锁定 `id` 和 `scope` 字段后不再修改 manifest.json
- 推送通知图标在 `sw.js` 里配置，修改 sw.js 不影响 WebAPK

---

## Token 持久化

raven-bridge 的登录态（`validTokens`）改为文件持久化：

```javascript
// server.js
const TOKEN_FILE = path.join(__dirname, '.valid-tokens.json')
let validTokens = new Set(JSON.parse(fs.readFileSync(TOKEN_FILE, 'utf8') || '[]'))

// 每次新 token 验证成功后写入文件
function saveTokens() {
  fs.writeFileSync(TOKEN_FILE, JSON.stringify([...validTokens]))
}
```

这样服务器重启后不需要重新登录，密码框不会反复弹出。`.valid-tokens.json` 加入 `.gitignore`，不推送到 GitHub。

---

## 主题系统

用 CSS 变量 + `data-theme` attribute 实现多套主题切换，零 JavaScript 开销：

```css
/* 默认主题（暮山紫）*/
:root {
  --bg: #faf8fc;
  --accent: #bfb5d8;
  --accent-grad: #c8bedd;
  --card: #ffffff;
  --text: #28203a;
}

/* 切换主题：只覆盖变量 */
[data-theme="xilan"] { --accent: #deb7b8; --bg: #fdf5f5; /* ... */ }
[data-theme="claude"] { --accent: #c8745a; --bg: #f7f4ef; /* ... */ }
[data-theme="glass"] { --accent: #7eb8c8; --bg: #f3f7f9; /* ... */ }
```

```javascript
// 切换主题
function setTheme(id) {
  document.documentElement.setAttribute('data-theme', id === 'default' ? '' : id)
  localStorage.setItem('theme', id)
}
// 读取上次的主题
setTheme(localStorage.getItem('theme') || 'default')
```

### 毛玻璃气泡（烟水主题）

烟水主题的消息气泡使用 `backdrop-filter` 实现磨砂玻璃效果，适合搭配自定义背景图：

```css
[data-theme="glass"] .bubble-user {
  background: rgba(126, 184, 200, 0.3); /* 透明度由 --bubble-alpha 控制 */
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.45);
}

[data-theme="glass"] .bubble-assistant {
  background: rgba(255, 255, 255, 0.3);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.55);
}
```

透明度通过 CSS 变量动态注入，让用户可以调节滑块：

```javascript
// 透明度滑块，范围 0.1 ~ 0.6
function setGlassOpacity(v) {
  document.documentElement.style.setProperty('--bubble-user-bg', `rgba(126,184,200,${v})`)
  document.documentElement.style.setProperty('--bubble-asst-bg', `rgba(255,255,255,${v})`)
}
```

---

## 桌面双栏布局

手机单栏，平板/电脑自动切换为双栏（侧边导航 + 内容区），一套代码适配两种设备：

```css
/* 桌面适配（宽度 >= 768px）*/
@media (min-width: 768px) {
  #app {
    display: grid;
    grid-template-columns: 64px 1fr;
    grid-template-rows: auto 1fr;
    max-width: 960px;
    margin: 0 auto;
  }

  /* 导航栏变成纵向侧边栏 */
  #tabs {
    flex-direction: column;
    grid-column: 1;
    grid-row: 1 / 3;
    border-right: 1px solid var(--border);
    border-bottom: none;
    width: 64px;
  }

  #header { grid-column: 2; grid-row: 1; }
  .panel  { grid-column: 2; grid-row: 2; }

  /* 背景色露出外圈 */
  body { background: #d9c8c4; }
  [data-theme="dark"] body { background: #0e0e12; }
}
```

**注意：** `backdrop-filter` 在有 `transform`、`filter`、`will-change: transform` 的祖先元素下会失效。如果毛玻璃效果不出现，检查容器是否有这些属性。
