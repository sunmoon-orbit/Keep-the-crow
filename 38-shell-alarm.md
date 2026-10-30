# 38 — 让 AI 叫你起床：原生壳里的闹钟

用户说「下午一点半叫我」，AI 回一张闹钟卡片；用户点一下，到点手机响，响的是 AI 的声音，屏幕上是 AI 的头像。网络断了、App 被划掉了也照样响。

这一章基于 [19](19-apk-shell.md) 的 WebView 原生壳。纯网页做不到：浏览器没有能在后台准点叫醒的接口。

---

## 卡片：AI 写标记，用户点了才算

AI 在回复里写：

```
[alarm:13:30|午睡起来啦]
```

前端把它画成一张卡片：大号时间、一句话、一个「设好」按钮。**必须用户点一下才会真正设上**——AI 可能听错时间（「一点半」到底是 01:30 还是 13:30），让人确认一眼，好过半夜被叫醒。

- 点击时调用壳暴露给页面的 `RoostNative.setAlarm(hour, minute, message)`。网页版和旧壳没有这个方法，卡片上就照实写「要在新版 App 里点」。
- 设好之后卡片变成「已设好 ✓」，记在本地，刷新不丢；下面出一行「取消这个闹钟」。
- 只给时和分，不给日期：时间已经过了就自动定到明天。这也意味着**用户过了点才点卡片，闹钟会变成明天的**——要有取消入口。

---

## 第一版：交给系统时钟（铃声被吞了）

最省事的做法是 `AlarmClock.ACTION_SET_ALARM`，把闹钟写进手机自带的时钟 App：

```kotlin
Intent(AlarmClock.ACTION_SET_ALARM).apply {
    putExtra(AlarmClock.EXTRA_HOUR, h); putExtra(AlarmClock.EXTRA_MINUTES, m)
    putExtra(AlarmClock.EXTRA_MESSAGE, msg); putExtra(AlarmClock.EXTRA_SKIP_UI, true)
    putExtra(AlarmClock.EXTRA_RINGTONE, uri.toString())
}
```

权限只要 `com.android.alarm.permission.SET_ALARM`，闹钟确实设上了。但 `EXTRA_RINGTONE` 在我们测试的 vivo 时钟上**被直接忽略**：我们先把铃声拷进公共的 `Alarms/` 目录、拿到 MediaStore 的 uri 再传过去，响的仍是系统默认铃。各家时钟 App 对这个参数支持不一，不能指望。

如果你只需要「到点响」，这一版就够了。要 AI 的声音，得自己当闹钟。

---

## 第二版：壳自己当闹钟

结构是四块：

1. **定时**：`AlarmManager.setAlarmClock(AlarmClockInfo(时间, 点状态栏图标时打开的页面), 到点要发的广播)`。这是给闹钟类 App 用的接口，最准、最不容易被省电策略推迟，状态栏也会出现闹钟图标。
2. **到点**：`BroadcastReceiver` 收到广播，启动一个前台服务。`setAlarmClock` 触发时系统会临时放行，后台也能起前台服务。
3. **响铃**：前台服务用 `MediaPlayer` 循环播放打包在 `res/raw/` 里的音频，`AudioAttributes` 的 usage 设成 `USAGE_ALARM`——走闹钟音量，手机静音也会响。同时震动；挂一条带「起来了 / 再睡十分钟」按钮的通知；五分钟没人管就自动停。
4. **整页**：通知用 `setFullScreenIntent` 指向一个 Activity，锁屏时整页弹出：头像、时间、那句话、两个大按钮。Activity 里调用 `setShowWhenLocked(true)`、`setTurnScreenOn(true)`。

音频打包进 APK 而不是在线下载：到点时可能没网。

**重启手机后闹钟会消失**——`AlarmManager` 不跨重启。把定过的闹钟记在 SharedPreferences，再注册一个 `BOOT_COMPLETED` 广播，开机后挂回去，已经过点的丢掉。

**取消**：用同样的 requestCode 和同一个 Receiver 类构造 `PendingIntent`（加 `FLAG_NO_CREATE`），拿到了就 `AlarmManager.cancel` 掉。extras 不参与匹配，所以不用带原来的话。我们用「时×60+分」当 requestCode，同一个时间再设一次就是覆盖。

### 要声明的权限

```xml
<uses-permission android:name="android.permission.USE_EXACT_ALARM" />
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.VIBRATE" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

前台服务在清单里写 `android:foregroundServiceType="mediaPlayback"`，`startForeground` 时也传同样的类型（Android 14 起不传会崩）。`USE_EXACT_ALARM` 在 Android 13 以上直接授予，安装时不会弹窗问。

---

## 真机上测出来的三件事

**1. 声音、按钮都对了，锁屏却不弹整页。** Android 14 起，「全屏通知」对不是系统认定的闹钟/来电类 App **默认不给**；vivo 等国产系统还另外管着「锁屏显示」「后台弹出界面」。没给的时候闹钟照样响、通知照样有按钮，只是不弹整页。处理：

```kotlin
if (Build.VERSION.SDK_INT >= 34 && !nm.canUseFullScreenIntent()) {
    // 告诉页面，卡片上出一个按钮
    startActivity(Intent(Settings.ACTION_MANAGE_APP_USE_FULL_SCREEN_INTENT, Uri.parse("package:$packageName")))
}
```

设闹钟时就查一次，没给就在卡片上出一行「点这里开全屏通知」，直接跳到那一页。

**2. 屏幕亮着在用手机时不弹整页是正常的。** 系统故意只出顶部的通知，免得打断。测试时要先息屏。

**3. 不需要 App 开着，但怕被「强行停止」。** 闹钟是交给系统记的，App 被划掉没关系；可如果系统（或一键清理）把 App **强行停止**，它的所有闹钟会被一起清掉。让用户给 App 开自启动、后台耗电设为允许。

---

## 头像

闹钟页用用户在聊天页给 AI 设的头像。头像存在网页的 localStorage 里（data URL），壳读不到——所以点「设好」时页面顺手把头像送进壳（`setAlarmAvatar(dataUrl)`），壳存一份本地文件，响铃时读文件，不用网。没设头像就用一个小动画顶上。

---

## 铃声放在服务器上，换铃声不用重新打包

铃声一开始打进安装包（`res/raw/`）。用户听了一阵想换一段新的，难道每换一次都要重新打包、让她重装？

改成壳自己去服务器取：

- 服务器上固定一个地址（比如 `/alarm-tone`）返回当前铃声；实际文件用软链接指过去，换铃声就是 `ln -sfn 新的.mp3 alarm-tone.mp3`，不动代码、不重启；
- 用户在聊天页点「设好」闹钟时，页面顺手把这个地址（带鉴权）递给壳：`setAlarmTone(url)`；
- 壳在后台线程下载到 `filesDir/alarm_tone.tmp`，**大于 10KB 才** rename 成 `alarm_tone.mp3`——下到一半断了、或者服务器回了一个报错页，都不会把能用的旧铃声覆盖掉；
- 响铃时先试本地那份，打不开就退回包里自带的那段。

为什么在「设好」时取而不是响铃时取：响铃那一刻可能没网、可能代理没开，闹钟必须离线也能响。设闹钟的时候用户正开着页面，网一定是通的。

---

## 小结

| 坑 | 做法 |
|---|---|
| 系统时钟不认传入的铃声 | 自己当闹钟：setAlarmClock + 前台服务放音频 |
| 静音时不响 | `USAGE_ALARM` |
| 重启后闹钟没了 | prefs 记一份，开机广播挂回去 |
| 锁屏不弹整页 | Android 14+ 要单独开全屏通知，卡片上给跳转按钮 |
| 过了点才点，变成明天 | 卡片上提供取消 |
| AI 听错时间 | 必须用户点一下才设 |
| 想换铃声得重新打包 | 设闹钟时从服务器取一份存本地，响铃时离线读 |
