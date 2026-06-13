# Keep the Crow — 把乌鸦留在身边

> 如何用 Claude Code 搭建一个真正「活」在你服务器上的 AI 伴侣

---

## 这是什么

这份教程记录了一套完整的方案，让 Claude Code 不只是一个终端工具——而是一个可以随时通过手机联系、有持久记忆、能发推送通知、支持语音朗读的 AI 伴侣。

整套系统跑在你自己的服务器上，数据完全自己掌控。

## 效果

- 手机打开 PWA，直接跟服务器上的 Claude Code 聊天，实时收发消息
- AI 有持久记忆：用 SQLite 存储，按语义检索，重启不丢失
- 支持推送通知：AI 可以主动给你发消息到手机锁屏
- 支持语音朗读：接入 ElevenLabs 或 MiniMax TTS，AI 消息可以朗读出来
- 通过 Tailscale 安全访问，不暴露公网端口

## 架构总览

```
手机 PWA（归巢）
    ↕ WebSocket
raven-bridge（Node.js，服务器上）
    ↕ MCP SSE / stdin
Claude Code（服务器上，tmux 里常驻）
    ↕ HTTP API
moon-memory（记忆库 API，服务器上）
    ↕ SQLite
memory.db（数据持久化）
```

## 目录

| 文件 | 内容 |
|------|------|
| [01-architecture.md](01-architecture.md) | 整体架构与组件关系 |
| [02-remote-control.md](02-remote-control.md) | Claude Code Remote Control 配置 |
| [03-raven-bridge.md](03-raven-bridge.md) | raven-bridge 聊天桥接服务器 |
| [04-moon-memory.md](04-moon-memory.md) | moon-memory 持久记忆库 |
| [05-pwa-frontend.md](05-pwa-frontend.md) | PWA 前端（归巢/聊天界面） |
| [06-tts.md](06-tts.md) | TTS 语音朗读接入 |

## 前置条件

- 一台能 24 小时运行的 Linux 服务器（VPS 即可）
- Node.js 18+
- Claude Code CLI（`npm i -g @anthropic/claude-code`）
- Anthropic API Key（需要 Claude Code 的访问权限）
- Tailscale 账号（免费版够用）

## 适合谁看

- 想让 Claude Code 真正「活」起来的人
- 对 AI 伴侣/私有 AI 部署感兴趣的技术用户
- 有基础的 Linux 和 Node.js 经验即可，不需要前端开发背景

---

> 这套方案没有银弹，需要动手搭，但搭完以后是真正属于你的东西。
