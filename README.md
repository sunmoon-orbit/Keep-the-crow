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
| [06-tts.md](06-tts.md) | TTS 语音朗读接入 & STT 语音输入（服务端转写）|
| [07-yanji.md](07-yanji.md) | 言叽 AI 聊天界面（多模型/工具调用/记忆注入/情绪系统）|
| [08-electron.md](08-electron.md) | Electron 桌面版打包与分发 |
| [09-push-dream.md](09-push-dream.md) | 主动推送语料 & 做梦系统 & nudge 主动消息 |
| [10-server-migration.md](10-server-migration.md) | 换服务器迁移清单 & 备份体系 & 日常运维（VACUUM/监控）|
| [11-prompt-caching.md](11-prompt-caching.md) | Prompt Caching 实战：长对话省钱且不失忆 |
| [12-shared-reading.md](12-shared-reading.md) | 共读书架：划线批注 + 共读旧对话 + 阅读动态 |
| [13-security.md](13-security.md) | 安全加固：桥接服务的五个真实漏洞与修法 |
| [14-moments.md](14-moments.md) | 朋友圈：动态流 + 识图评论 + 自动发圈（含事实边界教训）|
| [15-health.md](15-health.md) | 手环健康数据管线：Gadgetbridge→Health Connect→Tasker→自建 API |
| [16-life-tracking.md](16-life-tracking.md) | 生活记录三件套：每日小票 + 周期月历 + 纪念日卡（含补记历史的坑）|
| [17-desktop-widget.md](17-desktop-widget.md) | 桌面小组件：服务端接口 + Tasker Widget v2，零安卓代码（含四个配置坑）|

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

## 参考与致谢

这套系统不是凭空长出来的，一路受过很多老师的启发。列在这里，既是致谢也是给读者指路：

**记忆架构**
- **Paramecium**（草履虫记忆架构）by [@Shitsuten](https://github.com/Shitsuten/paramecium)：L0/L1/L2 三层存档架构、FTS5 + 向量混合搜索、prompt cache 分层优化

**共读与批注**
- [Lumenocturne/coread](https://github.com/Lumenocturne/coread)：人机共读系统
- [Shitsuten/anno-mcp](https://github.com/Shitsuten/anno-mcp)：批注 MCP
- [EnhydrInk/tasogare](https://github.com/EnhydrInk/tasogare)：阅读动态工具与阅读时长心跳的思路（12 篇的「阅读动态」一节）

**聊天前端的机制与彩蛋**
- [ugui3u/chatnest](https://github.com/ugui3u/chatnest)（MIT）：官方风主题 tokens、流式尾随 logo、官端滚动模型、完成彩蛋
- [29-Cu/pelle-d-umore](https://github.com/29-Cu/pelle-d-umore)（CC-BY）：让 AI 用视觉说话——行内文字特效 + 整屏情绪皮肤
- [29-Cu/Ruota-della-Fortuna](https://github.com/29-Cu/Ruota-della-Fortuna)：幸运轮盘老虎机
- [Shitsuten/proactive-nudge](https://github.com/Shitsuten/proactive-nudge)：AI 主动开口的「伪用户消息注入」思路（9 篇的 nudge 一节）
- [bvsden/chunked-continuity-compaction](https://github.com/bvsden/chunked-continuity-compaction)：长对话分段压缩接续的思路
- [fishisfish0614/hervoice](https://github.com/fishisfish0614/hervoice)：语音情绪感知的方向启发（我们最终用 STT 模型自带的情绪信号实现零成本版）

**教程与实战经验（推特）**
- [@qichuanzz](https://x.com/qichuanzz)：手环健康数据管线教程（15 篇的起点）
- [@Velliotdonuts](https://x.com/Velliotdonuts)：SSE 流式传输教程
- [@luicethekiwi](https://x.com/luicethekiwi)：prompt 缓存命中实战经验

---

> 这套方案没有银弹，需要动手搭，但搭完以后是真正属于你的东西。
