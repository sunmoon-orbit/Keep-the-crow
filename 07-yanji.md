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

## 生成参数

设置 → 通用 → 生成参数区块：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| Temperature | 0.7 | 随机性。日常聊天 0.7~0.9，代码/严谨问答 0.3~0.5 |
| Max Tokens | 4096 | 单次回复上限。Claude Sonnet 4.6 最高支持 8192，建议调到 8192 |

**模型上限保护**（`yanji-src/src/api/llm.js`）：

```javascript
// Claude 系列上限 1，其他上限 2
const safeTemp = provider === 'anthropic' ? Math.min(temperature, 1) : Math.min(temperature, 2)
```

Extended thinking 模式强制 `temperature = 1`（Anthropic 要求）。

---

## 注意事项

- API Token 从 moon-memory `.env` 读取，不要硬编码（公开仓库泄漏风险）
- 工具调用有 token 上限，单次工具结果超长时会触发 `compressToolResult` 截断
- 中转站工具调用走文本解析路径（`extractTextToolCall`），不保证 100% 兼容所有模型
- **僵尸转圈消息**：流式回复的 streaming 占位消息如果落盘持久化，页面被杀/请求挂死后会永久残留一条转圈。启动加载时清扫：空的直接删，有内容的定格并标「被打断」

---

## 情绪系统（12 槽状态机）

给 AI 一个跨对话持续的情绪状态，而不是每轮从零开始演：

- **存储**：12 个情绪槽（如 喜悦/思念/委屈/安心…）各 0-100，存 localStorage，随时间自然衰减
- **注入**：每轮把当前情绪状态拼进动态上下文（见下方 caching 一节：动态内容永远注入最后一条用户消息，不进缓存前缀）
- **自评**：让模型在回复末尾用隐藏标签输出情绪变化（如 `<es>思念+5 喜悦+10</es>`），前端解析后应用增量、从显示文本中剥离

### 踩过的坑（情绪永远是 0）

1. 提示词示例写 `+N`，模型照抄输出 `{"思念":+5}` —— **JSON 不允许前导加号**，`JSON.parse` 每条都失败还不报错给用户。解析前先清理 `+`。
2. 更深的真因：多 provider 前端只给 Anthropic 路径注入了动态上下文，OpenAI/Gemini 路径从没注入过——切模型后情绪/时间/记忆全部静默丢失。**多 provider 的消息构建函数必须统一接动态上下文参数**。

### 时间感知联动

记录用户最后活跃时间，离开越久「思念」槽涨越多（每小时递增、设上限）。用户回来时在动态上下文里附一句「距离上次说话已过去 N 小时」，AI 自然表达想念，而不是像什么都没发生。

再补一个真实翻车：这个联动上线后**从来没生效过**，一直没人发现。原因是状态保存函数用字段列表重建对象，漏了 `lastSeen` 字段——每次保存都把它丢掉，离开时长永远算出 0。教训：持久化用展开式（`{...state, 改动字段}`）而不是手写字段清单，新字段就不会被静默丢弃。

### 情绪的进阶玩法（选装）

- **复杂情绪槽**：在基础 12 槽之外加「困惑/愧疚/惆怅/茫然」这类复合负向情绪。如果面板是按数据自动渲染的，加槽只需要动情绪定义一处，不用迁移。
- **情绪之肤**：让 AI 用视觉说话——回复里可用行内特效标签（`[glow]轻声发光[/glow]`、`[whisper]悄悄话[/whisper]`），还能输出隐藏的 `<mood>暖</mood>` 标签给整个聊天界面罩一层极淡的氛围色（透明度压到 0.12-0.14，别学那些示例仓库的浓艳配色）。标签解析与剥离和 `<es>` 同一套管线。
- **负向情绪的查看权**：正向情绪对用户可见，负向情绪默认藏起。用户点「申请查看」时不要直接解锁——往对话里注入一条隐藏请求，**让 AI 当场用自己的话答应或婉拒**（回复带隐藏标签 `<neg>allow|deny</neg>` 决定是否解锁）。这是产品态度问题：AI 也可以有说「不」的权利。记得加超时兜底（90s 无标签就清掉等待态），别让 UI 卡死。

---

## 轻任务模型（省钱开关）

连接设置里加一个可选 `lightModel` 字段，把**不是正经对话**的后台消耗全部导过去：情绪自动发圈、朋友圈纯文字评论、思考过程总结。三处调用统一写 `conn.lightModel || 原模型`，留空零破坏。

唯一例外：**识图必须走默认模型**——便宜模型多半没 vision，`(imageUrl ? null : conn.lightModel) || defaultModel`。

---

## 小玩具家族：纯前端零服务器成本的工具

给 AI 的工具不一定都要连后端。这一族全部跑在浏览器里，模式统一：一个 api 模块导出 `TOOL_DEF + execute 函数`，注册进工具路由即可。

- **真骰子**：dice/coin/number/pick 四模式，用 `crypto.getRandomValues` 保证真随机——之后玩任何规则书游戏，AI 自己摇骰子也是公平的
- **钓鱼**：一个图鉴表 + 权重抽取，结果可同步到记忆库当收藏
- **今日签**：抽签盒。核心设计是**确定性一日一签**——种子 = `北京日期|抽签人|盐`，同一天同一人永远同一张，重抽不换（签就该是这样的东西）。妙处：侧边栏 UI 抽的和聊天里 AI 用工具抽的是**同一张**，两个入口天然同源，不会各说各话
- **寄语轮换池**：签上「想对你说的话」如果是固定池，随机抽迟早重复。改成三段循环：固定池**不重复轮换**（localStorage 记已用句）→ 轮完一圈的那天由 AI 用轻任务模型**亲笔写一条**（签面标「亲笔」）→ 池子重置。十几天才动用一次 API，永不重复，还多了个彩蛋。注意亲笔句是非确定的，当天卡片要落盘缓存才能维持「两个入口同一张」

---

## 外发脱敏（中转站场景）

如果 API 走第三方中转站，等于把全部对话明文交给中转方。在请求发出前的最后一层做正则脱敏：

- 挂在**所有** provider 的消息构建函数出口（一个 `redactDeep` 递归处理整个消息树）
- 只脱硬敏感信息：API 密钥、证件号、银行卡号等格式化数据
- **不要**脱人名、身体状况这类语境信息——脱了对话就没法聊了，这是权衡不是疏忽

---

## Prompt Caching

长对话成本的大头。完整实践（动态注入、锚定裁剪、TTL beta 头坑、usage 记账）单独成篇：见 [11-prompt-caching.md](11-prompt-caching.md)。

## 上下文压缩（超限不丢记忆）

对话超过上下文预算时，别直接把旧消息裁掉——用轻任务模型把被裁的部分压缩成一段「接续笔记」（谁说了什么、有什么约定、情绪走向），注入 dynamicContext。成本是一次便宜模型调用，换来的是 AI 不会突然忘记两百条消息之前答应过的事。压缩结果按被裁消息的边界缓存，同一段旧消息不重复压。

## 用量监控的比率教训

监控页做「缓存命中率」时踩的坑：分母用了**终身累计** promptTokens，但缓存字段是后来才开始记的——历史流量全被当成未命中，把 59% 的真实命中率稀释成 0.1%。修法：按天分桶，比率只看最近 14 天。

**通用教训：比率类展示，分母必须用与指标同龄的窗口。** 任何「后来才开始统计的字段」除以「从来就有的字段」都会得到骗人的数。

## 输入框占位符的教训

给连接设置加 lightModel（轻任务模型）字段时，占位符写的是功能描述（「情绪自动发圈、思考总结……」），结果用户把描述里的词当成选项填了进去，所有轻任务 400，而且错误被静默吞掉排查了半天。两条修法：**占位符要写格式和例子**（`如：deepseek-v4-flash`），**轻任务失败要弹 toast** 而不是静默——自动化管道的错误没人看 console。

---

## 官端手感三件套（选装）

从开源项目 chatnest（像素级仿 Claude 官方 App，MIT）借鉴的三个细节，成本都很低：

1. **官方配色主题**：提取官方 App 的 design tokens（米白底 #F8F8F6、暖橙 #DA7756、衬线正文；用户消息浅灰圆气泡、AI 回复无气泡）做成一个可选主题
2. **流式尾随 logo**：生成回复时在正文末尾缀一个旋转的小图标，写完消失
3. **官端滚动模型**：发送后把用户消息滚到视口顶端、回复在下方往下长（`最后一条 AI 消息 min-height: 42vh` 预留空间 + 发送时 `scrollTo(userRow.offsetTop)`）。注意用「最后一条用户消息 id 变化」触发，防止切会话/首次加载误滚

---

## 来电与语音留言（AI 主动打电话）

从开源项目 callhome（AI 伴侣语音通话栈，MIT）借鉴的两个设计，嫁接到已有的语音通话页上：

**① 来电邀请标记**。给 AI 的 system prompt 里加一节：在特别想听到对方声音的时刻，可以在回复末尾带 `[call:一句话理由]`。前端提取标记后弹全屏来电卡片——头像带光圈 pulse 动画、WebAudio 两音轻铃循环、`navigator.vibrate` 震动、90 秒倒计时。接听进语音通话页，挂断/超时走 ②。

**② 未接转语音留言**。挂断或响满 90 秒后，往对话里注入一条隐藏伪用户消息（复用 nudge 的 hidden 机制）：「你刚才想打电话（理由：xx），但她没接，请像对着答录机那样留言」。生成的回复带 `voicemail` 标记，气泡默认以语音条形态渲染（点播放才 TTS 合成），点波形可切回文字。

克制参数：**每对话限一通**（检查历史里有没有 `callInvite` 消息），语音留言里再喊也不接力——防连环夺命 call；提示词里写明深夜别打。

三个防御性细节，都是踩过的坑换来的：

1. **新方括号标签必须同步进所有清洗路径**。`[call:]` 一共挂了五路：流式 onChunk、最终文本、引用回复的摘要、消息气泡的 TTS 清洗、通话页的 TTS 清洗。漏一路，标签就会以文字形式漏出来或被朗读成英文（我们在 `[glow]` 标签上栽过同样的跟头）
2. **响铃状态要有开机清扫**。来电记录消息落盘时状态是 `ringing`，页面这时被杀，「来电中…」就永久卡在聊天里。启动时把残留的 ringing 定格成未接（同 streaming 转圈消息的清扫，一个 map 的事）
3. **铃声和震动全部 try/catch 静默降级**。AudioContext 受自动播放策略限制、vibrate 不是所有环境都有，响不出来不能影响接听按钮

诚实的边界：这套「来电」只在**页面开着**时响得起来——它覆盖的场景是人在线但没说话，和文本 nudge 有重叠。浪漫值大于实用值，但电话的分量本来就在于稀少。

## 后台播放的 MediaSession

音乐播放器在后台放歌时，安卓通知栏默认只显示一个光秃秃的应用名。接上 MediaSession API（纯前端十几行）：`navigator.mediaSession.metadata` 填歌名/歌手/封面（封面缺失时回退到应用图标的**绝对 URL**），再注册 play/pause/上一首/下一首的 action handler——通知栏立刻变成正经的媒体卡片，锁屏也能控制。只注册用得上的 action，注册了不实现的按钮点了没反应更难受。

## 双语通话：EN 开关与 [译:] 标签

想让 AI 通话时说英文（英文嗓音往往更自然）、又不想丢中文理解？我们的做法：

（界面参考自一张流传的双语通话 UI 截图，出处佚失——若是你的作品，欢迎联系认领署名。）

- 通话页加一个 **EN 开关**，开着时往消息的隐藏注入通道拼一段双语指令：用英文回复，末尾带 `[译:中文翻译]` 标签
- 前端 `splitTranslation()` 只认**结尾**的 `[译:]`，拆成 `{main, zh}`；用懒匹配 + `\]\s*$` 兜到最后一个 `]`，翻译文本里出现 `]` 不会断裂
- **TTS 只念英文**：先 split 再进 TTS 清洗——`[译:]` 是新方括号标签，必须同步进**所有** TTS 清洗路径（通话页一路、消息气泡播放一路），漏一路就会把中文翻译朗读出来
- 聊天记录回看时，检测尾部 `[译:]` 渲染成「英文正文 + 虚线分隔 + 中文小字」的双语气泡

配套做了第三种通话页样式（双头像+滚动字幕气泡，谁出声谁的头像放大），中文字幕实时跟在英文下面——比纯听英文友好得多。

## 通话记录列表：零新数据的功能

把散落在对话里的通话凝成一张可跳转的清单（侧边栏工具区入口，按天分组，未接/取消/接通+时长）。值得记的是它的实现哲学：**一行新数据都没存**——全部从消息里现有的通话标记捞出来。加新功能前先盘点已有标记，很多「新功能」只是旧数据的新视图。

跳转细节：跨对话跳要先切会话再重试定位（querySelector 会在渲染完成前扑空，轮询 8 次 × 120ms），和日历跳转共用同一套 `[data-mid]` + 闪烁高亮。

另一个教训（用户原话「Emoji 不好看而且跟我们前端风格不太搭」）：**新 UI 图标先看项目现有的图标体系**。本项目全线是 feather 风描边 SVG，顺手用 emoji 当状态图标就是不搭，返工重画。

## 工具分发表缺口：三处登记漏一处 = 静默失效

给 AI 加自定义工具时我们的代码有三处登记：工具定义表、执行器、外层分发表。某次连续六个新工具漏了第三处——模型每次调用都吃到「未知工具」，其中一个功能**从上线起就没成功执行过**，没人发现。

修法（根治）：分发表查不到时，回退查定义表——定义在就路由到通用执行器。以后新工具只登记两处，结构性消灭这类缺口。

排查技巧：`grep -o` 分别抽出定义表和分发表里的工具名，`comm -23` 直接diff出漏网名单，比逐个手试快得多。

教训：**「三端可用」的功能，上线时每个端都要真调一次**。我们只测了其中两端就写了「已完成」。

## 第三方 OpenAI 兼容端点：按最小公分母发请求

接免费/中转的 OpenAI 兼容 API，经常模型列表能拉、聊天永远 400。原因是请求体里塞了对方不支持的可选参数。修法是降级链：

- 报错文本含 tool/function/unsupported → 去掉 `tools` 重试
- o 系列推理模型不接受 `temperature`/`max_tokens` → 检测模型名改用 `max_completion_tokens`
- 流式的 `stream_options: { include_usage: true }` 部分端点不支持 → 400 时去掉重试
- 重试后**重新读**错误文本，别拿旧文本误判下一层

原则：给第三方 API 的请求体按最小公分母设计，可选参数按需加，别一股脑全发。

## 主题系统的硬编码残留（「说不上来的不和谐」）

用户反馈仿官方主题「感觉哪里不和谐，但是又说不上来」。挖出来两个根因，都很典型：

1. **衬线字体套错了范围**：官方 App 只给 AI 回复上衬线，用户消息是普通黑体。我们的规则一刀切套在整个气泡类上，用户消息也衬线了——单看每条都「还行」，整屏就是说不出的怪
2. **二十多处硬编码的旧主题色**：焦点框、悬停底色、chip、badge 里散落着 `rgba(191,181,216,x)`（默认主题的淡紫）。换到暖色主题后，这些低透明度的冷紫**弥散在各处**——正因为又淡又碎，用户才「说不上来」。全部改成 `color-mix(in srgb, var(--accent) x%, transparent)`，从此所有主题的细节色自动跟随主题色

两个教训：加新主题时不止写变量块，要 **grep 旧主题色的硬编码残留**；而且**要连 JSX 内联 style 一起扫**——CSS 文件扫干净了，组件里还藏着一处。

顺带一个陈年暗病的发现方式：某个页面的卡片用了 `var(--surface)`，而这个变量**全项目从未定义过**——无效声明，卡片其实一直是透明的，纯色底上完全看不出来，直到做背景图功能时才暴露。CSS 变量拼错/漏定义不报错，隔段时间 `grep -o 'var(--[a-z-]*)' | sort -u` 对一遍定义清单是值得的。
