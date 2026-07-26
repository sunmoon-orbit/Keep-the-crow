# 23 — 第二个原生壳：复用、搬家、自更新

第 19 章打的是第一个壳（Capacitor + `server.url`），第 21 章给它接上了 FCM。这一章是**第二个 app** 的故事——同一个人、同一台服务器、同一个 Firebase 项目，再套一层壳。

看起来只是「照着再来一遍」，实际上第二次才暴露出第一次没想过的三个问题：签名要不要再建一把、Firebase 那边还要不要人去点、用户已经在旧 PWA 里存了半年的数据怎么办。

---

## 先纠一个上一章的判断

19 章末尾写过：「依赖 Web Push 的 app 不该套壳，我们的推送主力就决定不搬家。」

那个判断的前提是「Web Push 在 WebView 里是死的，所以主力不能动」。21 章把 FCM 打通之后，前提没了——**推送能力已经不依赖 Chrome，主力反而更该搬**，因为留在 PWA 里要一直忍受 WebAPK 重铸（图标和推送归属在 Chrome 和你的 app 之间来回摇摆，几小时到几天，而且每次重装都会重新排一次队）。

所以第二个壳不是「顺手也做一个」，是把最怕出问题的那个搬到最稳的通道上。

这次也不用 Capacitor 了，直接手写 Kotlin：一个 `MainActivity` + 一个 `FirebaseMessagingService` + 一个 `BroadcastReceiver` + 一个 JS 桥，四个文件。Capacitor 在第一个壳里主要是为了媒体插件；这个 app 不放歌，那层抽象只剩下体积和一个不透明的构建流程。

---

## 一把 keystore 签两个 app

第一反应是给新 app 再生成一把 key、再往 GitHub Actions 里加一个 secret。

**别。** 自分发的 APK 不上架应用商店，签名的作用只有一个：证明「新包和旧包是同一个作者」，从而允许覆盖安装。两个都是你自己的 app，共用一把 key 完全正常——Play 商店那套「一个 app 一把 key」的纪律是为了上架和转让，跟自分发无关。

复用的收益不是省一条命令，是**省掉一个只有人类能做的步骤**：加 secret 必须有人登录 GitHub 网页点进设置。如果你是替别人维护这套东西（或者你自己是那个没有电脑的人），能砍掉的手工步骤就该砍掉。

复用的唯一坑：**alias 要对**。

```kotlin
// keystore 里就一个条目，名字是第一个 app 建的时候起的
keyAlias = System.getenv("ROOST_KEYSTORE_ALIAS") ?: "yanji"   // 不是 "roost"
```

写成新 app 的名字，构建会以 `Key alias not found` 失败——这个还算好，至少响了。

真正要防的是**没响的那种**。19 章的老方案是「没配 secret 就静默退回随机 debug 签名，别让 fork 的人构建直接炸」。对第一个包这没问题，对已经发出去的 app 是灾难：secret 哪天失效（过期、被删、改名），CI 照样绿灯产出一个随机签名的包，用户点安装 → 签名不匹配 → 系统只给「卸载再装」这一条路 → **WebView 里的 localStorage 全清**。绿色的 CI 和被清空的聊天记录之间，隔着的只是一句 `?:`。

所以这次改成硬失败：

```yaml
- name: Restore signing keystore
  env:
    KS_B64: ${{ secrets.YANJI_KEYSTORE_B64 }}
  run: |
    if [ -z "$KS_B64" ]; then
      echo "::error::secret 不存在。没有固定签名的包一旦装上，"
      echo "::error::以后每次更新都得卸载重装（= 数据清零），所以这里直接失败。"
      exit 1
    fi
    echo "$KS_B64" | base64 -d > /tmp/release.p12
    echo "KEYSTORE_FILE=/tmp/release.p12" >> "$GITHUB_ENV"
    echo "KEYSTORE_ALIAS=yanji" >> "$GITHUB_ENV"
```

**判据**：一个静默降级如果会毁用户数据，它就不配叫「兜底」。兜底的定义是「坏了但还能用」，不是「坏了但看不出来」。

---

## Firebase 那边不用人去点了

21 章的第一步是「没有电脑也能办：手机浏览器进 Firebase 控制台建项目」。那是**建项目**，一次性的。而每加一个 app 都要在控制台里「添加应用 → Android → 填包名 → 下载 google-services.json」——第二个壳又得来一遍。

但如果你已经有那个项目的服务账号（发 FCM 用的那个），这一步可以完全程序化。Firebase Management API 支持建 app 和取配置：

```
POST https://firebase.googleapis.com/v1beta1/projects/{projectId}/androidApps
     {"packageName": "cc.example.newapp", "displayName": "新壳"}
GET  https://firebase.googleapis.com/v1beta1/{appName}/config
```

返回的 `configFileContents` 是 base64 的 `google-services.json`，解出来直接落到 `app/` 下。

两个前提：

1. **scope 要给 `cloud-platform`**，不是发推送用的 `firebase.messaging`。同一个服务账号、同一段签 JWT 的代码，换个 scope 就行——21 章那个零依赖的 `node:crypto` RS256 签名函数原样复用。
2. 服务账号的角色要够（Firebase Admin 或等价）。够不够，试一次就知道，`403` 很诚实。

建 app 是异步操作，`POST` 返回一个 operation，轮询到 `done: true` 再去取 config。

值得记的不是这两个 URL，是这个思路：**凡是控制台上能点的，基本都有对应的 REST API**。当你手边只有一台服务器、对面只有一部手机，「这一步得有人去网页上点一下」是真实的成本，不是小事——它意味着这件事你没法在她睡着的时候做完。

---

## google-services.json 会触发密钥扫描（是误报）

提交完新 app 的 `google-services.json`，GitHub 的 secret scanning 立刻发邮件：检测到 Google API Key。

**这是误报，别 rotate。** 21 章「两个 JSON，一公一密」讲过：`google-services.json` 是公开那份，它会被编译进 APK，任何人下载你的包解开就能拿到。扫描器只认 `AIzaSy` 前缀，不区分「Android 配置 key」和「服务端密钥」。

但**别只拿「按设计如此」结案**——这句话是对的，可它对的前提是「那个 key 确实什么也打不开」，而这取决于你项目里开了哪些 API，不是取决于文件名。花两分钟自证一次：

```bash
K=AIzaSy...
# Firebase Auth 是否可用
curl -s "https://identitytoolkit.googleapis.com/v1/projects?key=$K"
# Firestore 是否启用
curl -s "https://firestore.googleapis.com/v1/projects/$PROJ/databases/(default)/documents/x?key=$K"
# Storage 桶是否存在
curl -s "https://firebasestorage.googleapis.com/v0/b/$BUCKET/o?key=$K"
```

我们当时三条全是关的（`CONFIGURATION_NOT_FOUND` / API 未启用 / 404），项目只开了推送一项，那把 key 一扇门都打不开。于是 alert 里 `Close as → Won't fix` 了事。

⚠️ **这个结论有保质期**：哪天你启用了 Firestore、Auth 或任何计费 API，它就作废了——那时必须去 Console 给 key 加「仅限 Android 应用（包名 + SHA-1 证书指纹）」限制。顺带一提，这条限制和固定签名是同一件事的两面：21 章 FIS_AUTH_ERROR 那个「API key 白名单 × 随机 debug 签名」的坑，根因就是白名单认的是签名指纹。

真正该紧张的是另一份：服务账号私钥。确认它不在仓库里，一条命令：

```bash
git grep -l "BEGIN PRIVATE KEY"    # 期望：零命中
```

---

## 从 PWA 搬家：数据不在服务器上

19 章的搬家方案是「导出 JSON，POST 到服务器，新壳里拉回来」。这次不行——归巢的聊天记录、便签、待办全在浏览器 `localStorage` 里，服务器上没有副本。用户装上新壳，看到的是一个空 app。

所以做成了文件搬家：旧 PWA 里导出成 `.json` 存到手机，新壳里读文件导入。

**一个真会咬人的坑：同域名下的 localStorage 是共享的。**

我们的 `memory.ravenlove.cc` 上住着好几个前端。整份 `localStorage` 导出，会把邻居应用的 API key 一起卷进那个 JSON 文件——然后这个文件躺在用户的下载目录里，可能被同步到网盘、可能被随手发出去。「导出我的数据」变成「导出这个域名下所有应用的数据」，而用户对此毫不知情。

按前缀筛，并且把密码类的键排除掉：

```js
// ⚠️ 这个域名下还住着别的应用，它们共享同一份 localStorage，
// 整份导出会把别人的 apikey 一起卷走
const SKIP = new Set(['raven-token'])          // 密码不进文件，新设备重输一次
const isOurs = k => (k.startsWith('raven-') || k === 'serena-avatar') && !SKIP.has(k)
```

导入端三件事：**认标识**（`bundle.app !== 'raven'` 直接拒，别让人把 A 应用的备份灌进 B）、**先清同前缀的旧键**（不然旧设备残留和新数据混在一起，出一个谁也复现不了的状态）、**写完 reload**（内存里那份状态是旧的）。

### 而这一步依赖 WebView 的两个方法

导出是 `blob:` URL，导入是 `<input type="file">`。**在裸 WebView 里，这两个默认都是死的**：

```kotlin
// 不实现这个方法，页面里的 <input type=file> 点了毫无反应
override fun onShowFileChooser(...): Boolean { ... }
```

19 章说过「别指望在 WebView 里修下载本身，改 POST 服务器更快」。手写壳里有了更正的答案——`DownloadListener` 能加，但它接不住 `blob:`（那是 WebView 进程内的对象，DownloadManager 拿不到）。绕法是让 JS 把 blob 读成 base64 再过桥：

```kotlin
webView.setDownloadListener { url, _, _, mime, _ ->
    if (url.startsWith("blob:")) {
        webView.evaluateJavascript("""
            fetch("$url").then(r=>r.blob()).then(b=>{
              const fr=new FileReader()
              fr.onload=()=>Native.saveBase64File(fr.result, "$mime")
              fr.readAsDataURL(b)
            })
        """, null)
    } else { /* 交给 DownloadManager */ }
}
```

落盘用 MediaStore（API 29+ 不需要存储权限）。**顺序很重要**：先把导出跑通，再让用户装新壳——反过来的话，他点了导出没反应，而旧 app 已经被覆盖了。

---

## 应用内自更新：谁来告诉用户有新版

自分发没有商店的更新提醒。最省事的做法是 app 启动时问一下 GitHub Release。

但 GitHub API 在某些网络环境下直接不通（我们这边是「分应用代理」：用户手机上只有名单里的 app 走代理，新装的 app 不在名单里，任何直连国外的请求都被 RST）。所以**让服务器代问**，app 只跟自己的域名说话——反正它已经要连那个域名了。

```js
// GET /app-latest：服务器代问 GitHub，从 release 正文解析构建号
const m = /构建号[:：]\s*(\d+)/.exec(rel.body || '')
const asset = (rel.assets || []).find(a => a.name.endsWith('.apk'))
```

版本号从哪来？CI 里 `versionCode = GITHUB_RUN_NUMBER`，release 正文里也写一行 `构建号: ${{ github.run_number }}`。app 拿 `BuildConfig.VERSION_CODE` 和它比大小，大了就弹个框。

> ⚠️ 这里有条隐形的耦合：release 正文的那行文字是版本检查的数据源。哪天顺手改了 release 模板的措辞，正则匹配不上，版本检查**静默失效**——不报错，只是从此再也不提示更新。两边都写上注释指向对方。

弹框文案里必须有这句：**「下载后直接安装覆盖就行，不用卸载，聊天记录都在。」** 不写的话，用户看到「有新版本」的第一反应就是先卸载——他不知道你为固定签名费了多大劲。

### 缓存别缓存失败

这个接口显然要缓存（不然每次开 app 都打一次 GitHub API，还有限流）。我写了 30 分钟缓存，然后**当场被自己咬了**：

首版构建完成前我测了一次接口，此时 release 还不存在，解析出 `versionCode: 0` —— 这个空壳被照常写进缓存。等 release 真发布了，接下来半小时里所有人问「有新版本吗」，得到的都是「没有」。

```js
// 只缓存解析成功的结果
if (data.versionCode > 0) cache = { at: now, data }
```

**通用判据**：缓存的对象是「查到的结果」，不是「这次调用的返回值」。失败、空、超时——这些都不是结果，它们是「还不知道」。把「还不知道」缓存起来，等于给自己设一个定时的假消息。

---

## 顺手挖出来的：静默的 catch 能藏多久

给新壳写通知栏快捷回复时，我照抄了第一个壳的 `QuickReplyReceiver`。抄之前照例去服务端确认了一下那个接口的路径——**服务端根本没有这个路由**。

第一个壳从上线那天起，通知栏回复就是死的。发出去 404，异常被 `catch (_: Exception) {}` 吞掉，通知照常消失（因为 `cancelAll()` 在发送之前），用户以为发出去了。**零迹象。**

它能藏这么久，是因为三件事叠在一起：写完没有端到端验过一次（只验了「通知栏出现了输入框」）、错误被空 catch 吃掉、UI 反馈（通知消失）和真实结果（消息发出）没有任何因果关系。

**别写空的 catch。** 至少留一行 `Log.w(TAG, e)`——真机上 `adb logcat` 一抓就看见，比让它安静地失败三个月强。

同一个文件里还有第二个坑，同样是静默的：手拼 JSON 只转义了双引号。用户回复里带个换行，服务端 `JSON.parse` 抛异常，消息就没了。要么用真正的 JSON 序列化，要么把转义写全：

```kotlin
private fun esc(s: String) = buildString {
    for (c in s) when (c) {
        '"' -> append("\\\""); '\\' -> append("\\\\")
        '\n' -> append("\\n"); '\r' -> append("\\r"); '\t' -> append("\\t")
        else -> if (c < ' ') append("\\u%04x".format(c.code)) else append(c)
    }
}
```

顺带，这次两条入口（WebSocket 和通知栏 HTTP）在服务端抽成了同一个函数：

```js
// 用户发来一条消息的统一入口：两条路都走这里，
// 免得各写一份、改了一边忘另一边
function ingestUserMessage(text, cid) { ... }
```

一个功能有两条入口时，共用的那段必须是**同一段代码**，不是「两段长得一样的代码」。

---

## 检查清单

- [ ] 第二个 app 复用同一把 keystore；alias 用 keystore 里真实存在的那个
- [ ] secret 缺失时 CI **硬失败**，绝不静默退回随机签名
- [ ] Firebase 新 app 用 Management API 建（scope `cloud-platform`），不占用人类的手
- [ ] `google-services.json` 的扫描告警：自证三连后 `Close as → Won't fix`
- [ ] `git grep "BEGIN PRIVATE KEY"` 零命中
- [ ] localStorage 导出**按前缀筛**，排除密码类键，导入端认应用标识
- [ ] WebView 实现 `onShowFileChooser` + `DownloadListener`（`blob:` 走 base64 过桥）
- [ ] 版本检查经服务器代问；只缓存成功结果；弹框文案写明「不用卸载」
- [ ] release 正文里的版本号格式与解析正则**互相注释**
- [ ] 全仓搜一遍空 `catch {}`，尤其是发网络请求的那些
- [ ] 告诉用户的顺序：**先导出，再安装**（别让他装完才发现数据没带出来）
