# 19 — 安卓 APK 在线壳：从「装上就被墙」到一次出包

> 把 PWA 变成一个真正的安卓 app，但**不把代码打包进去**——壳里的 WebView 加载线上 URL，前端更新照旧 git push，永远不用重新出包。
>
> **特别鸣谢**：在线壳这条路线（server.url + WebView 走系统网络栈）来自 X 上的 **hushuo cat**——她已用同样的路线把自己的 PWA 成功转成 APK，在我们 0701 折戟之后指了这条明路。

## 为什么不直接打包代码进 APK

我们 2026 年 7 月初试过一次 Capacitor 把前端代码整个打进 APK，网页一切正常，APK 死活连不上服务器。排查了 Cloudflare、HTTP/3、服务器配置，最后发现根本不是服务器的事：

**用户在国内 + 服务器在国外 + 手机用分应用代理 = 任何新装的 app 默认不在代理名单里，流量直连海外被墙 RST。**

而且代码打包进去还有第二个代价：每次改前端都要重新出包、重新安装。

## 在线壳方案

Capacitor 有个很少被提起的配置：`server.url`。设置之后壳里的 WebView 不加载本地资源，直接加载你指定的线上地址：

```json
// capacitor.config.json
{
  "appId": "cc.example.myapp",
  "appName": "言叽",
  "webDir": "www",
  "server": { "url": "https://your-username.github.io/your-app/" }
}
```

两个关键收益：

1. **热更新**：前端改代码只需部署网页（我们是 git push 触发 GitHub Pages），打开 app 就是最新版。只有动原生层（新权限、新插件）才需要出新包
2. **代理问题的意外解法**：WebView 加载外部 URL 走的是**系统网络栈**，继承系统级 VPN/代理设置——和打包进去的代码直连行为完全不同。（分应用代理的用户仍然要把 app 加进名单，但至少走的是正常路径，加了名单就通）

`webDir` 里只需要放一个几行的占位 index.html，Capacitor 要求它存在但永远不会被加载。

## 服务器打不动？GitHub Actions 云端构建

出安卓包要 JDK + Android SDK + Gradle，我们服务器上没有 java、只剩 450MB 空闲内存，本地构建不现实。全部搬到 GitHub Actions：

```yaml
# .github/workflows/apk.yml 核心步骤
- run: npm ci && npx cap add android    # android/ 目录 CI 现场生成，不入库
- run: # sed 注入权限（见下）
- run: cd android && ./gradlew assembleDebug
- uses: softprops/action-gh-release@v2  # 产物挂到固定 tag 的 Release
```

`android/` 和 `node_modules` 都不进 git，仓库里只有 `package.json` + `capacitor.config.json` + `www/index.html` 三个文件。产物 4.2MB，用户从 Release 页下载安装。

### 坑一：权限要在 CI 里注入

`cap add android` 每次都生成全新的 AndroidManifest.xml，你手改的权限会丢。在 CI 里用 sed 在 INTERNET 权限后面插入需要的权限（麦克风、通知等），然后 **grep 验证插入成功，失败就让构建 fail**——否则权限静默丢失，装上才发现。

### 坑二：ubuntu runner 没有 ImageMagick

生成图标时 `convert` 命令直接挂。改用 `@capacitor/assets` 自带的 sharp 处理图片（把 512px 图标放大到工具要求的 1024px 也用 `node -e "require('sharp')..."` 完成），零外部依赖。

### 坑三：debug 签名每次随机 = 无法覆盖安装（最疼的一个）

CI runner 每次现场生成 debug keystore，**两次构建的签名不一致**，新包无法覆盖安装，只能卸载重装——WebView 的 localStorage（对话记录、设置）全部跟着丢。

修法：本地 keytool 生成一个 `debug.keystore` **提交进仓库**，CI 里 cp 到 `~/.android/` 复用。之后所有包同签名，永远覆盖安装、数据不丢。（固定签名之前发出去的首个包，下次更新仍要卸载重装一次，提前给用户备份/恢复流程兜底。）

## WebView 不是 Chrome：三个原生缺口

在线壳的本质是「网页跑在 WebView 里」，而很多你以为的「网页能力」其实是 Chrome 提供的：

### 1. Media Session：通知栏媒体卡片是 Chrome 画的

网页版放歌，通知栏的媒体卡片（歌名/封面/播放暂停）是 Chrome 作为「经纪人」画的。WebView 里 Media Session Web API 直接不存在。

修法：`@jofr/capacitor-media-session` 插件（Capacitor 6 配 4.x，Capacitor 5 配 3.x）。它的 API 形状和 Web API 同构，前端加一个原生分支即可：

```js
const nativeMS = () =>
  window.Capacitor?.isNativePlatform?.() ? window.Capacitor.Plugins.MediaSession : null
```

**架构要点**：server.url 在线壳里，Capacitor 桥照样注入到远程页面——线上前端可以直接用 `window.Capacitor.Plugins.*` 调原生插件，不用 import 任何 capacitor 包，网页在 Chrome 里打开时该值为 undefined，走原路径零影响。

细节：`setPositionState` 记得节流（timeupdate 一秒触发约 4 次，每次过桥都是 IPC）；stop 时把 playbackState 设成 none 收走通知卡片。意外收获：插件为活跃会话起前台服务，顺带解决了 WebView 后台播放被系统掐掉的隐患。

### 2. Android 14 的类型化前台服务权限（闪退元凶）

装上带媒体插件的包，点播放**直接闪退**。根因：插件自带的 manifest 只声明了基础款 `FOREGROUND_SERVICE` + `foregroundServiceType="mediaPlayback"`，而 **Android 14（targetSdk 34）起还要配对的类型化权限 `FOREGROUND_SERVICE_MEDIA_PLAYBACK`**，缺了 startForeground 抛 SecurityException，app 被系统掐掉。

教训：引入带前台服务的 Capacitor 插件，先看它 manifest 声明的权限是否覆盖 targetSdk 34 的类型化要求——社区插件很多写于 Android 13 时代，只带基础款。诊断技巧：闪退（而非报错）= 原生层崩；去 unpkg 直接翻插件源码里的 `android/src/main/AndroidManifest.xml`，比装环境快得多。

### 3. POST_NOTIFICATIONS 是运行时权限

manifest 声明了、前台服务也起了，媒体卡片还是不出来——安卓 13 起通知是**运行时权限**，壳里没人弹询问框，默认就是关的。用户手动开一次（设置→应用→xx→通知→允许）即可；覆盖安装会保留，卸载重装要重开。想自动弹窗的话用 `@capacitor/local-notifications` 的 `requestPermissions()`。

### 顺带：Web Push 在 WebView 里是死的

设置里的推送开关在 APK 里会报「浏览器不支持推送通知」——不是坏了，WebView 真不支持 Web Push。真推送要 FCM 原生通道（我们排在 v2）。**这也是为什么依赖 Web Push 的 app 不该套壳**：我们的推送主力（归巢）就决定不搬家，让言叽 APK 先当先锋把 FCM 验证通了再说。

## 数据迁移

WebView 和 Chrome 的存储完全隔离，PWA 里的数据不会自动出现在 APK 里。我们的做法：网页版「备份全部数据」导出一个 JSON（对话+连接设置+外观全量），APK 里「恢复备份」导入，一步到位。注意给用户说明白：**给人读的 MD 导出文件不能用来恢复**，恢复只认 JSON（真实用户第一天就踩了）。

## 检查清单

- [ ] `server.url` 指向线上地址，webDir 只放占位页
- [ ] appId 与已有 PWA/WebAPK 的身份完全错开，互不干扰
- [ ] CI 注入权限后 grep 验证，失败即 fail
- [ ] debug.keystore 固定并提交，保证覆盖安装
- [ ] 分应用代理用户：装完先加代理名单再打开（写进 Release 安装说明）
- [ ] 带前台服务的插件：核对 targetSdk 34 类型化权限
- [ ] 提醒用户手动开通知权限
- [ ] 备份/恢复流程先跑通，再谈一切
