# 02 — Claude Code Remote Control 配置

## 什么是 Remote Control

Claude Code 有一个 Remote Control 功能，允许外部服务通过 MCP SSE 协议向 CC 发送指令、接收输出。

配合一个桥接服务器（raven-bridge），就能让手机实时和 CC 对话。

---

## 配置步骤

### 1. 找到 Claude Code 配置文件

```bash
# 全局配置目录（所有项目共用）
~/.claude/settings.json

# 或项目级配置
your-project/.claude/settings.json
```

### 2. 开启 Remote Control

在 `settings.json` 里加入：

```json
{
  "remoteControlAtStartup": true
}
```

这会让 CC 在启动时自动开启 Remote Control 监听，不需要手动操作。

### 3. 在 tmux 里启动 CC

```bash
tmux new-session -d -s cc -c /path/to/your/project
tmux send-keys -t cc "claude" Enter
```

CC 启动后会在终端显示 Remote Control 的连接地址（通常是 `http://127.0.0.1:xxxxx/sse`）。

---

## 配合 raven-bridge 使用

raven-bridge 的职责是：

1. 监听 tmux session 的输出（用 `tmux capture-pane` 轮询）
2. 检测到以 `【特定前缀】` 开头的消息，判定为用户消息
3. 通过 Remote Control / MCP 把消息注入给 CC
4. CC 回复后，把回复广播给所有 WebSocket 客户端（手机）

具体搭建见 [03-raven-bridge.md](03-raven-bridge.md)。

---

## 注意事项

- `remoteControlAtStartup: true` 只在 CC 启动时生效一次，之后重启 CC 才会重新连接
- CC 需要保持在 tmux 里运行，不要直接在 SSH 里跑（断开连接会终止进程）
- Remote Control 监听的是本地端口，不直接暴露到外网

---

## 验证 CC 正在运行

```bash
tmux list-sessions          # 确认 cc session 存在
tmux attach -t cc           # 进入 session 查看状态
# Ctrl+B, D 退出但不关闭 session
```
