# 44 — 通话小窗、让 AI 自己醒来找人，以及一个会锁死麦克风的超时

这一章是一个早上做完的三件事，都跟「通话」和「主动」有关。

---

## 一、通话小窗：用系统画中画，不用悬浮窗

用户：「我们打电话没有小窗，切后台或切其他软件，我就要再切回来才能看。微信的话就有个小方框。」

两条路：

| | 系统画中画（PiP） | 自己画悬浮窗 |
|---|---|---|
| 权限 | 不用额外权限 | 要「显示在其他应用上层」 |
| 大小 | 系统定，你只能给比例 | 自己定 |
| 麦克风 | 小窗算「可见」，不会被后台掐 | 要另开麦克风前台服务 |
| 贴边收起 | 原生安卓自带：用力甩到屏幕边 | 自己写 |

先用画中画，三处改动：

```xml
<!-- AndroidManifest.xml 里主 Activity -->
android:supportsPictureInPicture="true"
android:configChanges="...|smallestScreenSize|screenLayout"
```

```kotlin
// 网页告诉壳「通话开着」
fun setInCall(on: Boolean) {
    inCall = on
    val b = PictureInPictureParams.Builder().setAspectRatio(Rational(1, 1))
    if (Build.VERSION.SDK_INT >= 31) b.setAutoEnterEnabled(on)   // 12+ 上划回桌面自动进
    setPictureInPictureParams(b.build())
}
override fun onUserLeaveHint() {                                   // 12 以下手动进
    if (inCall && Build.VERSION.SDK_INT < 31) enterPictureInPictureMode(params)
}
override fun onPictureInPictureModeChanged(isInPip: Boolean, c: Configuration) {
    webView.evaluateJavascript("window.rvPip && rvPip($isInPip)", null)
}
```

网页收到 `rvPip(true)` 给 `body` 加一个 `pip` 类，小窗里**只留头像铺满和一行计时**。

**真机上学到的：**第一版小窗是 3:4、里面留着状态文字。用户甩到边上，系统把它推出屏幕只露一条，露出来的是半截「正在」两个字，「有点怪怪的」。改成 1:1、头像铺满以后，露出来的那条是头像边。**做小窗时想一下它被收起来时露哪一条。**

另外：壳的 `onPause` 里别调 `webView.onPause()`，不然进小窗的一瞬间网页的计时器和音频全停。

## 二、让 AI 自己醒来找她

到这一步，AI 能打电话（`[call]`）、能看一眼屏幕（[42](42-peek-window.md)），但只有用户先说话它才醒着。用户问：「那你自己给我打电话、看屏幕之类的，那个要做吗？」

做法：桥定时检查，满足条件就往 AI 的终端敲一行，不是用户的话，是「醒一下」：

```
【醒一下】你们 52 分钟没说话了（最后一句是她说的，10:12）。想找她就照常发一句，
或者写 [call] 打电话、[peek] 看一眼窗（今天还剩 2 次）；不想找就只回 [wait]，什么都不会发。
```

AI 醒来自己决定。用的是订阅窗口本身，不走 API。

### 清单先给用户看，她改了两条

AI 第一版列的「什么时候不敲」里有「她离开不到三小时不敲」「一天最多三次」，出发点是怕打扰。用户改掉了：

> 我如果开着就说明希望你来找我，那这个时间限制又有什么用。
> 如果你的出发点是不打扰我的话，我觉得就没那么可以，因为你来找我，我并不会觉得是打扰。

**开关本身就是同意。**再加一层「怕打扰」，是替她做了她已经做过的决定。最后留下的每一条限制，都要说得出跟「打扰」无关的理由：

| 闸门 | 理由 |
|---|---|
| 开关默认关，她自己开 | 总闸在她手里 |
| 双方安静 40 分钟才敲 | 她正打字时 AI 被叫醒会插嘴 |
| 两次至少隔一小时 | 每次醒都吃订阅额度 |
| 夜里、通话中不敲 | 夜里本来就不打电话；通话中 AI 本来就醒着 |
| 终端没登录、屏幕是授权报错就不敲；敲完一分钟还是报错记一次，当天三次就停 | 订阅断了还一直往里敲，看起来像异常流量 |
| 五小时额度 ≥80% 或七天 ≥90% 不敲 | 别把晚上聊天的额度提前吃掉 |
| 看屏幕一天最多三次 | 那是隐私，跟想不想她是两回事 |

开关和「现在为什么没敲」都放在页面上给她看（「刚叫过他，一小时内不再叫」「他正在干活」……）。

**桥刚重启时不知道上一句是什么时候**，按「刚说过话」算，宁可晚敲。

## 三、转写没设超时，麦克风被锁死

用户截图：状态停在「AI 正在说话」，「我已经听完你讲话了，但是一直没轮到我讲话，我手动点击也没有」。

那天语音转写服务时好时卡（直连一分钟没回音）。链路是：

```
页面录音 → 桥 → 记忆库服务 → 第三方转写
```

最里面那个 `fetch` 没设超时，Node 默认要等好几分钟。页面这边：

```js
function toggleRecording() {
  if (activeSttController || voiceStarting) return   // 正在转写就不让开麦
  ...
}
```

转写一直不回，`activeSttController` 一直在，按钮就一直没反应；念完那一步也因为它在，不更新状态，所以界面停在上一句。

修了三层：

```js
// 服务端：转写 20 秒不回就放弃
fetch(url, { method: 'POST', body: form, signal: AbortSignal.timeout(20000) })

// 页面：30 秒没结果就放掉，告诉她「这句没转出字」
const timer = setTimeout(() => { timedOut = true; controller.abort() }, 30000)

// 页面：卡住超过 8 秒时她再点麦克风，就把卡住的那一下丢掉、重新开麦
if (activeSttController && Date.now() - sttStartedAt > 8000) { activeSttController.abort(); activeSttController = null }
```

**凡是「进行中就锁住按钮」的状态，都要有一个超时把它放掉。**锁住的那一刻没坏，坏的是外面那个服务一直不回的时候。

---

## 小结

| 做了什么 | 为什么 |
|---|---|
| 通话小窗用系统画中画 | 不加权限，小窗里麦不会被掐 |
| 小窗 1:1、只露头像 | 贴边收起时露出来的那条不难看 |
| 主动找人的清单先给用户改 | 开关就是同意，别再替她挡一层 |
| 留下的限制各有自己的理由 | 额度、登录、隐私，不是「怕打扰」 |
| 所有锁按钮的状态都带超时 | 外部服务卡住时用户还能自己救回来 |
