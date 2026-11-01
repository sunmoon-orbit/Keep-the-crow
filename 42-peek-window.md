# 42 — 窗：在用户允许的时候，让 AI 看一眼她的手机屏幕

用户刷到别人做的「让 AI 偷看你手机屏幕」：一种是装一个陌生的小 APK，另一种是用自动化 App 监听聊天通知、截图存相册、再用 curl 传上服务器。她问 AI：「你想不想做这个？」

AI 的回答是想，但不照那两种做。这一章讲为什么，以及做进自己原生壳里的版本。

---

## 一、为什么不装别人的

两条路最后都要开安卓的**无障碍服务**。这个权限能看到屏幕上的一切，也能替你点按钮，等于把整台手机交给那个 App。

- 陌生 APK：代码看不到，写得再好也是把钥匙交给不认识的人。
- 自动化 App 那条：要绕聊天软件的通知，截图会躺在相册里，还要 `sleep 15` 等文件写完。链路长，相册里也会攒一堆截图。

如果你已经有自己的原生壳（[38](38-shell-alarm.md) 那种），把这件事做进去：代码在自己仓库里，截图只走自己的服务器。

## 二、链路

```
AI 回复里写 [peek]
  → 桥：剥掉标记，过闸门（夜里不敲、一天上限、同时只敲一次）
  → 推送一条 data 消息 {type: "raven_peek", peekId}
  → 壳里的推送服务收到 → PeekService.knock()
      开关关着 / 没权限 / 锁屏 / 安全窗口 → 回桥一个原因
      否则 takeScreenshot → 缩到 720 宽 JPEG → POST /peek/upload?id=
  → 桥：存进私有目录，最多留 5 张、超过一天删
  → 往 AI 的终端送一行：【窗·看到了】…[附件: 路径]
  → 页面广播一条，聊天里加一行「AI 看了一眼你的屏幕」
```

## 三、壳里的无障碍服务：只用来截图

```xml
<!-- res/xml/peek_accessibility.xml -->
<accessibility-service
    android:description="@string/peek_desc"
    android:accessibilityEventTypes="typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:canRetrieveWindowContent="false"
    android:canTakeScreenshot="true" />
```

- `canRetrieveWindowContent="false"`：不读界面上的字。
- `onAccessibilityEvent` 留空，不响应任何事件，也不代点按钮。
- 截图用 `AccessibilityService.takeScreenshot()`（安卓 11+），拿到的是 `HardwareBuffer`，要先 `Bitmap.wrapHardwareBuffer()` 再 `copy(ARGB_8888)` 才能压 JPEG。
- `minSdk` 低于 30 的话，截图函数标 `@RequiresApi(R)`，调用处显式判断版本，不然 lint 会拦。

```kotlin
fun knock(ctx: Context, id: String) = when {
    !isEnabled(ctx)                 -> report(ctx, id, "off")
    Build.VERSION.SDK_INT < 30      -> report(ctx, id, "old")
    instance == null                -> report(ctx, id, "noperm")
    !powerManager.isInteractive ||
        keyguard.isKeyguardLocked   -> report(ctx, id, "locked")
    else -> instance?.shoot(id)
}
// takeScreenshot 的 onFailure 里，ERROR_TAKE_SCREENSHOT_SECURE_WINDOW 报 "secure"：
// 银行、密码这类 FLAG_SECURE 页面，系统本来就不让截。
```

每次截成功都弹一条通知「AI 看了一眼」。**不做「偷偷看」**：原教程里有一条「回复时不要提截图」，这里反过来，看了就说。

## 四、闸门放在服务端

开关在壳里（她随时能关），但「什么时候不敲」放在桥里，这样换壳、换窗口都管得住：

- 北京时间 23–7 点不敲，直接回 AI 一行「夜里不看」。
- 一天最多三次，计数存文件，第二天清零。
- 同一时间只敲一次，45 秒没回音就告诉 AI「没回音」，不重试。
- 截图目录权限 700，不进聊天存档，不进公开上传目录。

AI 这边的规矩写进它每个窗口都会读的说明：想知道她在干嘛**先问**，窗是问不到时的那一眼；她说在学习、在忙、在难过不看。

## 五、装好以后的那一步：「允许受限制的设置」

安卓 13 起，不是从应用商店装的 App，无障碍那一项是灰的，点不了。要先：

> 设置 → 应用 → 你的 App → 右上角 ⋮ → **允许受限制的设置**

再回无障碍里打开。把这句写进 App 里的开关说明，不然用户卡在这一步不知道为什么。

## 六、关代理时会怎样

如果推送走的是谷歌 FCM，而用户在国内：

- 不开代理时推送基本到不了，手机不知道有人敲门。
- 上传地址如果也在代理规则里，截图也传不回来。

AI 那边只会收到「45 秒没回音」。这也算一道保险：代理一关，窗就自动关上了。

---

## 小结

| 决定 | 理由 |
|---|---|
| 做进自己的壳，不装别人的 | 无障碍权限等于整台手机 |
| 服务只截图、不读内容、不点按钮 | 权限能小就小 |
| 每次都弹通知、聊天里留一行 | 偷看就不叫窗了 |
| 闸门在服务端 | 换窗口、换壳都还管得住 |
| 截图私有目录、限量、自动删 | 看一眼就够，不攒 |
