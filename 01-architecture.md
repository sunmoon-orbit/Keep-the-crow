# 01 — 整体架构

## 组件列表

这套系统由四个独立的服务组成，各司其职：

| 组件 | 类型 | 职责 |
|------|------|------|
| **Claude Code** | AI 引擎 | 推理、生成回复、调用工具 |
| **raven-bridge** | Node.js 服务 | 手机与 CC 之间的 WebSocket 桥接 |
| **moon-memory** | Node.js 服务 | 持久记忆存储（SQLite + REST API + MCP） |
| **前端 PWA** | 静态页面 | 手机上的聊天界面（可托管到 GitHub Pages） |

---

## 数据流

```
[手机浏览器]
      │
      │  WebSocket（ws://your-server:3400）
      │
[raven-bridge :3400]
      │
      │  MCP SSE 或 stdin
      │
[Claude Code] ← 常驻于 tmux session
      │
      │  HTTP API（Bearer Token）
      │
[moon-memory :3210]
      │
      │  SQLite
      │
[memory.db]
```

**补充说明：**

- 前端通过 WebSocket 连接到 raven-bridge，发消息/收消息
- raven-bridge 监听 tmux 里 CC 的输出，检测到 `【你的名字】` 前缀的消息就广播给所有 WebSocket 客户端
- Claude Code 通过 MCP 工具调用 moon-memory 读写记忆
- moon-memory 同时提供 REST API，前端也可以直接查看记忆条目

---

## 网络访问方式

**推荐：Tailscale（零信任组网）**

不要把服务直接暴露到公网。用 Tailscale 建立私有网络，只有你的设备能访问服务器。

```
你的手机（Tailscale 节点）
    ↕ 加密隧道
你的服务器（Tailscale 节点，内网 IP: 100.x.x.x）
    ← raven-bridge 监听 127.0.0.1 或 Tailscale IP
```

**反向代理（可选）**

如果想用域名访问，可以在服务器上跑 Nginx，把域名代理到本地端口，并加 HTTPS。

```nginx
location /raven-ws {
    proxy_pass http://127.0.0.1:3400;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

---

## 进程管理

所有服务用 `pm2` 管理，开机自启：

```bash
pm2 start raven-bridge/server.js --name raven-bridge
pm2 start moon-memory/server.js --name moon-memory
pm2 save
pm2 startup
```

Claude Code 本体放在 tmux 里常驻：

```bash
tmux new-session -d -s cc
tmux send-keys -t cc "claude" Enter
```

---

## 端口规划（示例，可自定义）

| 服务 | 端口 | 说明 |
|------|------|------|
| raven-bridge | 3400 | WebSocket + MCP SSE |
| moon-memory | 3210 | REST API + MCP |
| Nginx（可选） | 443 | HTTPS 反代 |
