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
