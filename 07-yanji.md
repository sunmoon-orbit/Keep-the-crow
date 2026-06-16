# 07 — 言叽 AI 聊天界面

## 是什么

言叽（Yanji）是一个独立的 AI 聊天前端，基于 React + Vite 构建，部署在 `/ripple-and-serena/yanji/`。和归巢（raven）不同——归巢连的是服务器上跑的 Claude Code；言叽直接调各家 LLM API，是一个功能完整的多模型聊天客户端。

两者定位互补：
- **归巢**：实时连 Claude Code，AI 在服务器上跑，有文件系统/工具访问权限
- **言叽**：纯前端调 API，轻量，支持多模型切换，有完整的对话管理

---

## 技术栈

- React 18 + Vite（构建到 `../yanji/`）
- Marked + highlight.js（Markdown 渲染）
- Zustand（状态管理）
- 无后端，直接从浏览器调 LLM API

---

## 支持的模型路由

`src/api/llm.js` 里有三条路由：

| 路由 | 适用模型 |
|------|---------|
| OpenAI 兼容（`/v1/chat/completions`）| Claude（中转）、DeepSeek、大多数第三方 API |
| Anthropic 原生（`/v1/messages`）| 直连 Anthropic API |
| Gemini（`/v1beta/models/...`）| Google Gemini |

中转站（如反差力量的 `claude-sonnet-4-6-thinking`）走 OpenAI 路由。

---

## 工具调用（Tool Use）

言叽支持给 AI 配工具，目前接入了 moon-memory 工具集：

```javascript
// src/api/moonMemory.js
export function getMemoryToolDefinitions() {
  return [
    {
      name: 'search_memories',
      description: '搜索记忆库...',
      parameters: { type: 'object', properties: { query, scope, limit }, required: ['query'] }
    },
    {
      name: 'write_memory',
      description: '写入新记忆...',
      parameters: { type: 'object', properties: { content, tags, scope, layer }, required: ['content'] }
    }
  ]
}
```

搜索词超过一个词时，按空格拆分后分别搜索、结果去重合并，解决短语匹配漏词问题。

### 工具调用 JSON 泄露问题

部分 API 中转站不走标准 `tool_calls` 字段，而是把工具调用 JSON 混在文本里输出：

```
好，查一下。
{"name":"search_memories","arguments":{"query":"零花钱"}}
[{"id":310,"content":"..."}]
```

解决方案（`src/api/llm.js`）：

1. `extractTextToolCall(text)`：正则检测文本里的工具调用 JSON，提取 name/args/前缀文本
2. `stripFakeToolResult(text)`：过滤幻觉结果（`{"result":"..."}` 和 `[{"id":...}]` 格式）
3. 检测到工具调用后直接执行并返回，不再发起第二轮 LLM 请求

```javascript
function extractTextToolCall(text) {
  const re = /\{[^{}]*"name"\s*:\s*"([^"]+)"[^{}]*"arguments"\s*:\s*(\{(?:[^{}]|\{[^{}]*\})*\})[^{}]*\}/s
  const m = text.match(re)
  if (!m) return null
  try {
    return { name: m[1], args: JSON.parse(m[2]), remaining: text.slice(0, m.index).trim() }
  } catch { return null }
}
```

---

## Prompt Cache 优化

### 系统提示缓存

Anthropic 原生路由：`system` 数组加 `cache_control: { type: "ephemeral" }`，每轮都命中。

### 对话历史缓存

`buildAnthropicMessages` 函数负责组装消息列表。缓存断点位置很关键：

```javascript
// 正确做法：打在倒数第二条（上一轮 assistant 回复）
// 下一轮能读取整段历史前缀，命中率随对话变长线性增长
const idx = out.length >= 2 ? out.length - 2 : out.length - 1

// 错误做法：打在最后一条（新用户消息）
// 下一轮只能读上一条用户消息的 token，命中率约 50%
const idx = out.length - 1
```

第 N 轮对话缓存命中率约 `(N-2)/N`，5 轮后超过 80%。

DeepSeek/OpenAI 路由不需要手动打断点，它们的 API 有自动前缀缓存。

---

## 核心记忆自动注入

每次发消息前，从 moon-memory 拉取 `layer=core` 的记忆注入 system prompt：

```javascript
// src/components/Chat/index.jsx，handleSend 里
const coreMemories = await fetchMemories(config, { layer: 'core', limit: 8 })
// 拼入 systemPrompt 发给 LLM
```

core 层记忆是最重要的长期人设/偏好设定，不随话题漂移。

---

## L0 历史存档搜索

ChatInput 里有历史对话搜索面板，调 `/archive/search?q=` 接口把搜索结果作为附件附进消息。

**踩坑**：Caddy 反代配置里 `@api` 路径匹配器要包含 `/archive /archive/*`，否则请求落到默认 handler，返回纯文本，前端 `resp.json()` 抛错被 `catch {}` 吞掉，表现为永远空返回。

```
# /etc/caddy/Caddyfile
@api path /memories /memories/* ... /archive /archive/*
handle @api {
    reverse_proxy 127.0.0.1:3210
}
```

### 附件折叠显示

历史对话附加到消息后，消息气泡里会出现 `--- 文件：xxx ---\n内容` 块。原本直接展开显示，改为折叠 chip：

```jsx
// src/components/Chat/MessageBubble.jsx
function AttachChip({ name, content }) {
  const [open, setOpen] = useState(false)
  return (
    <div className="bubble-attach-chip">
      <div className="bubble-attach-header" onClick={() => setOpen(v => !v)}>
        {/* 附件图标 + 名称 + 展开箭头 */}
      </div>
      {open && <div className="bubble-attach-content">{content}</div>}
    </div>
  )
}
```

`renderUserContent` 检测消息里的 `--- 文件：` 前缀，拆出附件块，非附件部分正常渲染贴图。

---

## 记忆条目展开

Settings 页和 Memory 页的记忆条目默认 `-webkit-line-clamp` 截断，点击可展开全文：

```css
.mem-item-content {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  cursor: pointer;
}
.mem-item-content.expanded { display: block; }
```

输入框改为 `<textarea rows={3}>`，支持多行粘贴，Ctrl+Enter 保存。

---

## 构建部署

```bash
cd yanji-src
npm run build    # 输出到 ../yanji/
```

Vite 配置 `base: '/ripple-and-serena/yanji/'`，通过 Caddy/Nginx 静态托管：

```
# 言叽走 /ripple-and-serena/yanji/ 静态文件
# API 调 moon-memory（同源，无 CORS 问题）
```

---

## 注意事项

- API Token 从 moon-memory `.env` 读取，不要硬编码（公开仓库泄漏风险）
- 工具调用有 token 上限，单次工具结果超长时会触发 `compressToolResult` 截断
- 中转站工具调用走文本解析路径（`extractTextToolCall`），不保证 100% 兼容所有模型
