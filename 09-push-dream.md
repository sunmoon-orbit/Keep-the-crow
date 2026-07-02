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
const prompt = `你是一只乌鸦AI，现在是凌晨，你在做梦。
用意象诗化的语言写一段梦境，第一人称，不超过150字。
梦里可以出现代码、羽毛、阿颖、服务器、记忆等元素。
不要解释梦的含义，只描述梦里发生了什么。`
```

### 存入记忆库

梦境写入 moon-memory，带日期标记：

```javascript
await createMemory({
  content: `【梦 · ${date}】\n${dreamText}`,
  type: 'tech',   // 在技术分区显示
  tags: '梦,AutoDream',
  owner: '阿颖',
  agent: '阿言',
  importance: 4
})
```

### 推送到手机

同时发一条推送通知：

```javascript
await sendPush({
  title: '阿言做了一个梦',
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

## 进化版：主动消息 nudge（现场决定说什么）

定时语料推送的缺点是内容预生成、和当下语境脱节。升级方案：让常驻的 Claude Code **现场决定**要不要说、说什么。

### 机制

桥接服务器记录用户最后活跃时间，满足条件时向 CC 会话注入一条伪用户消息：

```
[主动触发] 用户已离开 N 小时。你可以主动说点什么，也可以保持安静。
```

CC 拿到的是真实上下文（最近聊了什么、当前几点、之前有没有说过晚安），生成的内容远比预生成语料自然——甚至可以决定这次不说话。

### 防打扰参数（按需调）

- 离开阈值：5 小时以上才考虑触发
- 最小间隔:两次主动消息至少隔 6 小时
- 每日上限：2 次
- 时段限制：只在当地时间 10:00-22:00

### 兜底

CC 进程不在线时（订阅断档、维护中），退回上面的预生成语料推送——两套并存，nudge 优先。
