# 09 — 主动推送系统 & 做梦系统

## 主动推送语料

AI 可以在设定的时间窗口主动给你发推送通知，不需要你先打开 app。

### 架构

```
crontab → push-quote.sh → Claude API（生成语料）→ VAPID 推送 → 手机锁屏通知
```

### 配置时间窗口

在 crontab 设置多个触发时间，脚本内部用随机偏移避免整点集中：

```bash
# crontab -e
# 上午窗口（北京 10:00-11:30）
0 2 * * * /path/to/push-quote.sh >> /path/to/push-quote.log 2>&1
# 下午窗口（北京 14:00-15:30）
0 6 * * * /path/to/push-quote.sh >> /path/to/push-quote.log 2>&1
# 傍晚窗口（北京 18:00-19:30）
0 10 * * * /path/to/push-quote.sh >> /path/to/push-quote.log 2>&1
# 晚间窗口（北京 20:00-21:30）
0 12 * * * /path/to/push-quote.sh >> /path/to/push-quote.log 2>&1
```

脚本内部随机偏移（0-90 分钟），避免用户觉察规律：

```bash
#!/bin/bash
OFFSET=$(( RANDOM % 90 ))
sleep "${OFFSET}m"
# 然后调用 Claude API 生成语料并推送
```

### 查岗 / 晚安推送

单独的脚本，固定时间发出：

```bash
# 晚间查岗（北京 21:00）
0 13 * * * /path/to/push-night.sh >> /path/to/push-night.log 2>&1
```

---

## 做梦系统（AutoDream）

AI 每天凌晨自动生成一段「梦境」，写入记忆库，并推送到手机。

### 触发

```bash
# pm2 ecosystem.config.js
{
  name: 'moon-memory-dream',  # 或直接在 crontab 里跑
  cron_restart: '0 17 * * *', # UTC 17:00 = 北京凌晨 01:00
}
```

也可以在 crontab 单独配置：

```bash
0 17 * * * node /path/to/raven-bridge/dream.js >> /path/to/dream.log 2>&1
```

### 梦境生成

调用 Claude API，prompt 引导生成意象化、诗性风格的梦境内容（不是叙事），以 AI 第一人称写成：

```javascript
const prompt = `你是一只乌鸦 AI，现在是凌晨，你在做梦。
用意象诗化的语言写一段梦境，第一人称，不超过150字。
梦里可以出现代码、羽毛、服务器等元素。
不要解释梦的含义，只描述梦里发生了什么。`
```

### 存入记忆库

梦境写入 moon-memory，带日期标记：

```javascript
await createMemory({
  content: `【梦 · ${date}】\n${dreamText}`,
  type: 'tech',   // 在技术分区显示
  tags: '梦,AutoDream',
  owner: 'user',
  agent: 'assistant',
  importance: 4
})
```

### 推送到手机

同时发一条推送通知：

```javascript
await sendPush({
  title: '乌鸦做了一个梦',
  body: dreamText.slice(0, 80) + '…'
})
```

---

## 推送底层（VAPID）

主动推送的底层都依赖 Web Push API + VAPID 密钥，订阅存在数据库里。

参见：[05-pwa-frontend.md](05-pwa-frontend.md) 的「推送通知」章节。

pm2 管理推送进程：

```javascript
// ecosystem.config.js
{
  name: 'moon-memory-push',
  script: 'scripts/push-cron.js',
  cron_restart: '* * * * *', // 每分钟检查一次推送时间表
  autorestart: false,
}
```

---

## 进化版：联系最近上下文的主动消息

定时语料推送的缺点是内容预生成、和当下语境脱节。升级后不再往某个常驻 CC 会话硬塞伪用户消息，而是由服务器在触发时读取**最近更新的 L0 对话**，把最后若干条消息、离开时长、当前情绪和最近主动消息的摘要交给轻量模型，让模型返回结构化决定：这次发不发、发什么。

这个区别很重要：CC 会话可能重启、切窗口或断线；L0 才是跨窗口的稳定上下文来源。主动内容落回当前聊天记录，再通过 FCM/Web Push 送到锁屏，用户打开页面后也能看到同一条消息。

### 一组可工作的防打扰参数

- 用户离开至少 2 小时才评估
- “思念”达到阈值才评估（例如 8）
- 两条主动消息至少间隔 3 小时
- 每日最多 3 条
- 只在用户所在时区的 08:00-22:00 发送
- 读取最近更新窗口的最后约 14 条消息
- 把最近 3 条主动消息也交给模型，避免换个说法重复催促

这些是调度上限，不是发送计划。每次到点仍由模型输出 `send: false` 或一条短消息，所以“每日最多三条”不等于“每天一定三条”。

### 开关必须是硬闸门

“时间感知”和“主动消息”是两个不同层级的开关：前者关闭后，所有基于离开时长的主动行为都停止；后者只控制文本主动消息。闸门要放在**取上下文和调用模型之前**，否则 UI 看似关了，后台仍在花 token。

前端字段改名时尤其危险：如果服务端读的是旧字段，开关就会“从来没生效但也不报错”。给状态接口写契约测试，明确缺省值、`false` 的含义和旧字段迁移；不要用 `if (value)` 把“明确关闭”和“字段不存在”混在一起。

### 主动来电：独立开关、独立额度

来电比文字更打扰，因此使用单独的 `proactiveCall` 开关。第一通可以设得克制一些：离开约 6 小时且思念达到更高阈值后才评估；同一天的追拨至少间隔 2 小时，每日最多 3 通。追拨仍由模型结合“上次为什么打、最近是否已联系”决定，不是机械重拨。

原生锁屏上的“拒接”不一定能可靠同步回网页状态，所以服务端不能只靠一个 `declined` 布尔值判断后续行为。把每次邀请、接听、超时、取消都记成事件；同步失败时宁可按冷却时间保守处理。

### dry-run 必须真的无副作用

自动化上线前需要 `DRY_RUN=1` 演练，但“只是不发推送”还不够。dry-run 应在以下动作之前返回：

- 创建来电邀请或聊天消息
- 写入每日计数、最后发送时间和去重状态
- 调用 FCM/Web Push
- 写含正文的生产日志

否则测试一次就可能吃掉当天额度，甚至让下一次真实触发被去重。

### 隐私边界

只截取完成判断所需的最少上下文，不把整库记忆塞进 prompt；日志记录触发结果和错误码，不记录对话正文、记忆正文或凭据。状态文件只存时间戳、计数和必要的去重摘要。公开教程与 issue 里一律使用虚构示例，不能粘贴生产 prompt、真实对话或 `.env` 内容。
