# 11 — Prompt Caching 实战：长对话省钱且不失忆

## 为什么需要

AI 伴侣的对话动辄几百轮，每轮都把全部历史重新计费一遍是最大的成本来源。Anthropic 的 prompt caching 可以让重复的前缀按 0.1 倍价格计费——但它有一堆隐形的坑，配错了会**静默失效**（不报错，只是默默按全价算）。

三条铁律：

1. **严格前缀匹配**：缓存命中要求从第一个 token 开始逐字一致。前缀里任何一个字符变了，后面全部作废。
2. **写贵读贱**：写缓存是 1.25 倍（1h TTL 是 2 倍）价格，读是 0.1 倍。频繁毁缓存比不用缓存更贵。
3. **`cache_read_input_tokens` 是唯一可信的命中判据**：不要靠感觉，每次请求都看 usage 里的这个数字。

---

## 正确的消息结构

```
[system prompt]            ← 纯静态，打缓存断点（ttl 1h）
[历史消息 1..N-1]           ← 稳定不变
[倒数第二条消息]             ← 打第二个缓存断点（ttl 1h）
[最后一条用户消息]           ← 动态内容注入在这里，永远在断点之后
```

关键原则：**每轮变化的东西（当前时间、检索到的记忆、情绪状态）绝不能进缓存前缀**。把它们全部拼在最后一条用户消息的开头：

```javascript
// 动态内容注入到最后一条用户消息前——不写进缓存前缀，不毁历史命中
if (dynamicContext?.trim() && out.length > 0) {
  const last = out[out.length - 1]
  if (last.role === 'user') {
    const dynBlock = { type: 'text', text: `[以下为系统自动注入的实时上下文，并非用户发送]\n${dynamicContext}\n[实时上下文结束]` }
    out[out.length - 1] = { ...last, content: [dynBlock, { type: 'text', text: last.content }] }
  }
}
```

注意：注入只发生在**发请求时**，不要把动态内容写回数据库/前端存储——否则下一轮重建历史时它变成了前缀的一部分，每轮都不一样，缓存全毁。

断点打在倒数第二条（上一轮的 assistant 回复）：

```javascript
// 深拷贝再打断点，避免 cache_control 写回存储——
// 否则旧消息会累积断点，超过 Anthropic 的 4 个上限直接报错
const idx = out.length >= 2 ? out.length - 2 : out.length - 1
const last = out[idx]
if (typeof last.content === 'string' && last.content) {
  out[idx] = { ...last, content: [{ type: 'text', text: last.content, cache_control: { type: 'ephemeral', ttl: '1h' } }] }
}
```

---

## 坑一：TTL 1h 需要 beta 头，缺了静默退回 5 分钟

`cache_control` 里写 `ttl: '1h'` 只是许愿，真正生效需要请求头带上 beta 标志：

```javascript
headers: {
  'anthropic-beta': 'prompt-caching-2024-07-31,extended-cache-ttl-2025-04-11',
}
```

缺了这个头**不会报错**，缓存照样工作——但 TTL 是默认的 5 分钟。AI 伴侣的对话间隔经常超过 5 分钟，等于每次回来都重写一遍缓存（2 倍价格写、0 命中）。这是我们踩过的最隐蔽的坑：代码看起来全对，账单不对。

验证方法：隔 10 分钟发两条消息，看第二条的 `cache_read_input_tokens` 是否大于 0。

---

## 坑二：滑动窗口是缓存杀手

最常见的历史裁剪写法：

```javascript
messages.slice(-50)  // ← 千万别这么写
```

每来一条新消息，窗口就往前滑一格，**开头那条消息变了** → 前缀匹配失败 → 全量重写。这个写法让缓存 100% 失效，而且每轮都付写缓存的钱。

正确做法是**锚定切点**：切点按步长对齐，只在跨过边界时才移动。

```javascript
// 量化切点：窗口内多轮对话共享同一个开头，前缀保持稳定
const max = maxRounds * 2
if (messages.length <= max) return messages
const step = Math.max(2, Math.floor(max / 4) * 2)
const cut = Math.ceil((messages.length - max) / step) * step
return messages.slice(cut)
```

代价是窗口比 max 略小，收益是切点在多轮内不动，缓存持续命中。切点移动的那一轮会付一次全量写——这是计划内的，比每轮都写便宜一个数量级。

---

## 坑三：历史里的图片

base64 图片占大量 token，留在历史里每轮都要重新计费+缓存。只保留最近几条消息的图片，更早的降级成占位文本：

```javascript
const IMG_KEEP_RECENT = 4
const limited = allMsgs.map((m, i, arr) => {
  const keepImages = i >= arr.length - IMG_KEEP_RECENT
  const baseContent = !keepImages && m.images?.length && !m.content ? '[图片]' : m.content
  return { ...m, content: baseContent, images: keepImages ? m.images : undefined }
})
```

注意降级也要**稳定**：一条消息一旦降级就永远保持降级形态，否则又是前缀变化。

---

## 坑四：多 provider 时动态注入漏了谁

如果前端支持多个模型（Anthropic / OpenAI 兼容 / Gemini），每个 provider 都有自己的消息构建函数。动态注入必须**三处都接**——我们踩过的真实事故：只给 Anthropic 路径接了 dynamicContext，切到其他模型时情绪状态、当前时间、核心记忆全部静默丢失，排查了很久才发现不是提示词的问题。

---

## 记账：让命中率可见

每次响应都取 usage 并展示（我们直接显示在消息气泡上）：

```javascript
const cr = data.usage.cache_read_input_tokens || 0
const cw = data.usage.cache_creation_input_tokens || 0
// 气泡显示：缓存命中率 = cr / (input + cr + cw)
```

健康状态：日常对话命中率应该在 80-95%。突然掉到 0，按顺序查：beta 头还在吗？前缀里混进时间戳了吗？裁剪切点动了吗？有人改了 system prompt 吗？

---

## 什么不用做

网上一些方案是给**网关/中转**设计的（六段式缓存分层、55 分钟保活心跳定时给 API 发空请求续缓存等）。如果你的架构是纯前端直连 API：

- 保活心跳做不了也不该做（浏览器关了就停，且纯烧钱续一个可能根本用不上的缓存）
- 分段所有权无从谈起（没有网关这个角色）
- 用 1h TTL + 上面四条，已经覆盖绝大部分收益
