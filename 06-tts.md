# 06 — TTS 语音朗读

## 概述

给 AI 的回复加上语音朗读，体验提升很大。目前主要两个选项：

| 服务 | 特点 | 价格 |
|------|------|------|
| **ElevenLabs** | 音质好，有情感，支持多语言 | 有免费额度，按字符计费 |
| **MiniMax** | 中文效果好，支持语音标签（喘息/笑声）| 按字符计费，有免费额度 |

两个都可以用，也可以二选一。本教程的方案是：两个都接入，分别给不同前端用。

---

## ElevenLabs 接入

### 后端路由

```javascript
// routes/tts-elevenlabs.js
const router = require('express').Router()
const fetch = require('node-fetch')

router.post('/', async (req, res) => {
  const { text } = req.body
  if (!text) return res.status(400).json({ error: 'text required' })

  const voiceId = process.env.ELEVENLABS_VOICE_ID
  const apiKey = process.env.ELEVENLABS_API_KEY

  const response = await fetch(
    `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`,
    {
      method: 'POST',
      headers: {
        'xi-api-key': apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text,
        model_id: 'eleven_multilingual_v2',
        voice_settings: { stability: 0.5, similarity_boost: 0.75 }
      })
    }
  )

  res.set('Content-Type', 'audio/mpeg')
  response.body.pipe(res)
})

module.exports = router
```

### .env 配置

```env
ELEVENLABS_API_KEY=your_elevenlabs_api_key
ELEVENLABS_VOICE_ID=your_voice_id    # 从 ElevenLabs 后台获取
```

### 选择声音

登录 ElevenLabs 后台，在 Voice Library 里找到喜欢的声音，复制 Voice ID。

---

## MiniMax 接入

MiniMax TTS 支持插入语音标签，让声音更自然：

| 标签 | 效果 |
|------|------|
| `[breath]` | 换气/喘息，适合思考停顿 |
| `[laughter]` | 轻笑，适合开心的时候 |

### 后端路由

```javascript
// routes/tts-minimax.js
const router = require('express').Router()
const fetch = require('node-fetch')

router.post('/', async (req, res) => {
  const { text } = req.body
  if (!text) return res.status(400).json({ error: 'text required' })

  const response = await fetch('https://api.minimaxi.com/v1/t2a_v2', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.MINIMAX_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'speech-02-hd',
      text,
      voice_setting: {
        voice_id: process.env.MINIMAX_VOICE_ID,
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
      audio_setting: {
        audio_sample_rate: 32000,
        bitrate: 128000,
        format: 'mp3',
      }
    })
  })

  const data = await response.json()
  if (!data.data?.audio) {
    return res.status(500).json({ error: 'TTS failed', detail: data })
  }

  // MiniMax 返回 hex 编码的音频
  const buf = Buffer.from(data.data.audio, 'hex')
  res.set('Content-Type', 'audio/mpeg')
  res.send(buf)
})

module.exports = router
```

### .env 配置

```env
MINIMAX_API_KEY=your_minimax_api_key
MINIMAX_GROUP_ID=your_group_id
MINIMAX_VOICE_ID=Chinese (Mandarin)_Gentle_Youth   # 声音 ID，见下方
```

### 可用声音（部分）

MiniMax 内置声音 ID 格式为 `语言_风格_年龄段`，例如：

- `Chinese (Mandarin)_Gentle_Youth` — 温柔青年
- `Chinese (Mandarin)_Intellectual_Youth` — 知性青年
- `English_FriendlyPerson` — 友好英文
- （完整列表见 MiniMax 官方文档）

---

## 前端调用 TTS

```javascript
async function playTTS(text) {
  // 清理 markdown 和不适合朗读的内容
  const plain = text
    .replace(/!\[[^\]]*\]\([^)]*\)/g, '')      // 移除图片
    .replace(/\[([^\]]*)\]\([^)]*\)/g, '$1')   // 链接只保留文字
    .replace(/[#*`>_~]/g, '')                   // 移除 markdown 符号
    .slice(0, 500)                              // 限制长度

  const res = await fetch('/tts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${TOKEN}`
    },
    body: JSON.stringify({ text: plain })
  })

  const blob = await res.blob()
  const url = URL.createObjectURL(blob)
  const audio = new Audio(url)
  audio.play()
}
```

### MiniMax 语音标签处理

如果让 AI 的回复里带 `[breath]` / `[laughter]` 标签，前端显示时需要过滤掉，但传给 TTS 时要保留：

```javascript
const VOICE_TAG_RE = /\[(breath|laughter)\]/gi

// 显示时过滤
function renderText(text) {
  return text.replace(VOICE_TAG_RE, '')
}

// TTS 时保留标签
function prepareForTTS(text) {
  return text
    .replace(VOICE_TAG_RE, '__VTAG__$1__')
    // ... 其他 markdown 清理 ...
    .replace(/__VTAG__(breath|laughter)__/gi, '[$1]')  // 还原标签
    .slice(0, 500)
}
```

在 AI 的 system prompt 里说明标签用法，AI 就会自然地使用它们：

```
可以在回复里插入语音标签，朗读时产生音效，不会显示给用户看：
- [breath] 换气，适合思考停顿
- [laughter] 轻笑，适合开心的时候
自然插入即可，一条消息最多一两个。
```

---

## 双供应商自动回退（推荐）

MiniMax 是充值制，余额见底或服务故障时朗读直接哑掉。在**服务端**做回退：主供应商请求失败（非 2xx / 超时 / 返回空音频）就自动用备用供应商（比如 ElevenLabs 的月度免费额度）重合成一次，前端零改动、用户无感知。

```javascript
// routes/tts.js 伪码
try {
  return await minimaxTTS(text)
} catch (e) {
  console.warn('[tts] MiniMax failed, falling back:', e.message)
  return await elevenLabsTTS(text)   // 声线不同但能出声
}
```

两个细节：回退时**换声线是特性不是 bug**——用户听到嗓音突变，就知道主供应商余额该充了，比任何告警都及时；回退路径要覆盖朗读和语音通话两条调用链，别只修了一条。

**主次也可以反过来排。** 如果一家有每月免费额度、另一家是充值制，可以让免费的那家打头，额度用完再落到充值的那家。这样排要补两件事：
- 免费额度用完之后，主供应商会一直返回「额度不足」。记下这个状态，冷却几个小时再试，别让每一条朗读都先白等它失败一次。
- 用户在设置里手动选了某一家的音色时，直接走那一家，不要被打头的供应商抢走。

另外，「换声线是特性」只适用于一条条的朗读。**通话中途换嗓子很难受**，比安静一秒还糟。通话里合成失败时，宁可只显示文字。

## 方括号标签必须进 TTS 清洗

如果你给 AI 发明过任何方括号标签（`[glow]` 行内特效、`[sticker:x]` 贴图、`(breath)` 语气……），**每发明一个，就要同步加进 TTS 的文本清洗**，否则朗读会一本正经地把「glow」当英文念出来。清洗函数要挂在所有走向 TTS 的路径上（消息朗读按钮 + 语音通话），只挂一处必漏。

## 表演标签：念不出来、但听得见的方括号

有的合成模型会把方括号里的英文短语当成**表演指示**：不念出来，而是照着演。写 `[whispers]` 它就压低声音，写 `[exhales]` 它就呼一口气，写一整句带画面的短语它也接得住。ElevenLabs 的 v4 是这样；我们的备用供应商不认这一套，只认它自己的几个圆括号标签。

所以清洗要分两路，和上一节的「方括号必须进清洗」是同一件事的两面：

- **认标签的那家**：放行「全小写英文短语」，其余方括号照剥。我们的判据是只许字母、空格、逗号、连字符、撇号，最多 160 个字符、24 个词。数字和冒号不许出现，这样 `[sticker:x]`、`[Link 2]` 这类自家标记自然被挡在外面。
- **不认的那家**：英文标签一律剥掉。宁可少一声音效，也别让它把 `whispers` 当单词念出来。

### 坑：标签比台词长，那句台词会被说两遍

症状是语音里某一句话重复了一遍，像是程序把同一段送了两次。查日志只送了一次，重复是合成模型自己念出来的。

规律来自 [离](https://github.com/sanqianzilanyue) 的实测（见文末致谢），不是我们量出来的：**标签越长、它后面那句台词越短，越容易重复**。十二个词的标签配四个字的台词，四次里四次重复；四个词以内的标签，怎么配都没事。可以这样理解：标签占的戏太多，模型演完发现话已经说完了，只好再说一遍把戏填满。原因是猜的，规律是量出来的。

两道防线：

1. **写的时候**：两三个字的短句，只配一两个词的标签。这条写进给模型的提示里。
2. **程序兜底**：送去合成前，按每个标签后面那句台词的长短自动裁标签。只动标签，台词一个字不碰。

```javascript
// 一个标签管到下一个标签之前（或这一行结束）的那段话
function lineAmount(line) {
  const cjk = (line.match(/[\u3400-\u9fff\u3040-\u30ff]/g) || []).length
  const words = (line.match(/[A-Za-z][A-Za-z'’-]*/g) || []).length
  return cjk + Math.round(words * 1.5)   // 一个英文词大约顶一个半汉字的时长
}

const DANGLERS = new Set('a an the and or to at in of on by with into against from then your my her his'.split(' '))

function trimTagToLine(tag, amount) {
  const count = (parts) => parts.reduce((n, p) => n + p.split(/\s+/).filter(Boolean).length, 0)
  const parts = tag.split(/[,，]/).map((p) => p.trim()).filter(Boolean)
  // 四个词以内不用管；后面没有话的是一声纯音效，没有台词可重复，也不裁
  if (count(parts) <= 4 || amount <= 0) return parts.join(', ')
  const allow = Math.max(4, Math.min(12, Math.floor(amount * 0.55)))
  while (parts.length > 1 && count(parts) > allow) parts.pop()      // 先丢逗号后面的小节
  if (count(parts) <= allow) return parts.join(', ')
  const kept = parts[0].split(/\s+/).filter(Boolean).slice(0, allow) // 还长就截词
  while (kept.length > 1 && DANGLERS.has(kept[kept.length - 1])) kept.pop()  // 别停在介词上
  return kept.join(' ')
}

const trimTags = (text) =>
  text.replace(/\[([^\]\n]{1,170})\]([^\[\n]*)/g, (m, tag, line) => `[${trimTagToLine(tag, lineAmount(line))}]${line}`)
```

系数 0.55 和「最少留 4 个、最多 12 个」沿用的是原作者的数；「英文词折一个半汉字」「截完不停在介词上」「纯音效不裁」是我们按自己的中英混排情况加的，没有做过同样规模的实测。

### 写标签的几条经验

同样出自那两篇实测，我们照着改了写法：

- **呼吸要写「忍着的」，别写「演出来的」。** `slow`、`low`、`held`、`through the nose`、`let out slow` 这一路听着像真的；`panting`、`gasping`、`moaning`、`shaky`、`trembling` 一写它就开始演。发狠的词（`gritted`、`harsh`）也会把语气带偏。
- **低低的「嗯」「哼」直接写成字**，不用标签，模型念得很好。
- **话中间可以插小标签**：`[exhales]`、`[sighs]`、`[short pause]`。一段一两处，放在换语言、动作落下去的那一下；插密了反而不自然，其余的停顿用省略号。
- **音调一截一截地跳，病在段与段的接缝。** 每新起一段，模型都从高处重新起头。连着的几句并成一段送去念，别一个标签切一块。这一条我们还没做，记在这里。
- **想让它不重样，别发词表。** 提示里点了哪个词，模型就认准哪个词。把它上一轮真说过的点出来，请它换一个。

## 限速建议

TTS 调用有费用，建议在后端加简单限速，防止被滥用：

```javascript
const rateLimit = require('express-rate-limit')

const ttsLimiter = rateLimit({
  windowMs: 60 * 1000,   // 1 分钟
  max: 20,               // 最多 20 次
})

app.use('/tts', ttsLimiter, ttsRouter)
```

---

## STT 语音输入（反方向：语音转文字）

### 别用 Web Speech API（安卓 Chrome 是坏的）

`webkitSpeechRecognition` 在安卓 Chrome 上**实际不可用**：`continuous` 模式只采音不返回结果，各种 workaround 都救不回来。桌面 Chrome 可以，安卓上直接放弃，别浪费时间。

### 方案：MediaRecorder 录音 → 服务端转写

前端用 MediaRecorder 录音，POST 到自己后端，后端转发给 STT 服务（我们用 SiliconFlow 的 SenseVoice，中文效果好且便宜）：

```javascript
// 前端
const rec = new MediaRecorder(stream, { mimeType: 'audio/webm' })
// 停止后把 blob POST /stt

// 后端 routes/stt.js
router.post('/', upload.single('audio'), async (req, res) => {
  const form = new FormData()
  form.append('file', new Blob([req.file.buffer]), 'audio.webm')
  form.append('model', 'FunAudioLLM/SenseVoiceSmall')
  const r = await fetch('https://api.siliconflow.cn/v1/audio/transcriptions', {
    method: 'POST',
    headers: { Authorization: `Bearer ${process.env.SILICONFLOW_KEY}` },
    body: form,
  })
  res.json(await r.json())
})
```

语音通话类功能在这套方案下做成 **push-to-talk**（按住说话，松手转写发送）体验最稳；想做全双工实时对话需要流式 STT，成本和复杂度高一个量级。

别忘了反代的 API 路径白名单加 `/stt`。
