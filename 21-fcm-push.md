# 21 — FCM 原生推送：让套壳 APK 拥有真通知

> 19 章说过：WebView 里 Web Push 是死的，真推送要 FCM。这一章就是把这条路走通的全记录——**全程不需要电脑**，Firebase 项目创建、服务账号、密钥、IAM 全在服务器 + 手机聊天窗里完成。
>
> 决策先行：我们本来想先做个过渡方案（推送绕道已有的 PWA 通道），用户一句话拍死：「这次做过渡，下次还是要做完整的，那还不如第一次就做完整，做完了就不用一直吊着了。」她是对的。过渡方案的真实成本不是工时，是**永远悬着的下一次**。

## 全景

```
服务器任意推送点 → broadcastPush ─┬─ Web Push（PWA，原有）
                                  └─ FCM HTTP v1（APK，本章新增）
                                       ↓
                              Google FCM → 手机系统通知栏
```

改造完成后，所有既有推送点（主动关心、心意卡、回复通知、备份告警）**零改动**自动覆盖双通道——这是把发送收口在一个 `broadcastPush` 函数里的回报。

## 第一步：Firebase 项目（没有电脑也能办）

FCM 需要一个 Firebase 项目。官方路径是网页控制台点点点，但用户只有手机时照样能办：

1. 服务器上装 firebase-tools，`firebase login --no-localhost` 跑在 tmux 里（交互式流程用 `tmux send-keys` 驱动）
2. 登录 URL 发给用户，她在手机浏览器授权后把**授权码**贴回聊天窗，你 send-keys 进去
3. 之后一切走 REST：firebase CLI 缓存的 OAuth token（`~/.config/configstore/firebase-tools.json`，带 cloud-platform scope）足够调 IAM、apikeys、serviceusage 等所有 GCP 接口

**踩坑集（每个都真实吃过）**：

- **授权码断字符**：手机上长按复制容易在换行处截断，第一次贴回来的码中间混了空格直接失败。让用户用授权页自带的复制按钮整段复制
- **项目名不收中文**：`projects:create` 的 display_name 至少 4 字符且不认 CJK，「言叽」被 400 拒收，改英文名就过
- **addFirebase 403**：如果这个 Google 账号从没用过 Firebase，API 无法替人接受服务条款（ToS），会一直 PERMISSION_DENIED。解法：用户在手机上打开 Firebase 控制台走一遍创建流程（等于点了同意），之后 API 重试返回 409 ALREADY_EXISTS = 成功。**403 变 409 是好消息**，别看到报错就回退

## 第二步：服务端发送器（零依赖）

不装 firebase-admin（几十 MB 依赖树只为发个通知不值得）。FCM HTTP v1 的认证就是标准 OAuth2 JWT 流，Node 自带的 `node:crypto` 全搞定：

```
服务账号 private_key ──RS256签JWT──→ oauth2.googleapis.com/token
                                        ↓ access_token（缓存到期前5分钟）
POST fcm.googleapis.com/v1/projects/{id}/messages:send
  { message: { token, notification: {title, body},
               android: { priority: 'HIGH', ttl: 'Ns' } } }
```

要点：

- JWT claim 的 scope 用 `https://www.googleapis.com/auth/firebase.messaging`
- access token 缓存到 `expires_in - 300` 秒，别每条通知都换一次 token
- 死 token 清理（对应 Web Push 的 404/410）判定要小心——**这是本章最阴的坑**：

```js
// ✗ 错误写法：/INVALID_ARGUMENT.*token/ ——正则的 . 不跨行，
//   而且错误体里 "registration token" 出现在 INVALID_ARGUMENT 之前，词序也反了
// ✓ 分开判：
err.gone = resp.status === 404
  || /UNREGISTERED/.test(text)
  || (resp.status === 400 && /registration token/i.test(text));
```

判错的后果是尸体 token 永远留库，每次群发都白打一枪。上线前用假 token 发一次，确认 `gone: true` 再收工。

**服务账号密钥的 403 坑**：用 REST 给 firebase-adminsdk 服务账号签发密钥后，发消息报 `cloudmessaging.messages.create` 被拒——尽管它明明有 sdkAdminServiceAgent 角色。解法：显式给它补一条 `roles/firebasecloudmessaging.admin` 绑定（setIamPolicy）。控制台里生成的密钥没这问题，纯 REST 路线会撞。

## 两个 JSON，一公一密，千万别搞混

| 文件 | 身份 | 去向 |
|------|------|------|
| `google-services.json` | **设计上公开**（API key + App ID，随 APK 分发给所有人）| 提交进仓库，CI 用 |
| 服务账号密钥 `fcm-sa.json` | **真正的秘密**（private_key 可以以项目身份发任意推送）| 服务器 `secrets/` 目录，chmod 600，永不进 git |

推进仓库后 GitHub secret scanning 会对 google-services.json 里的 API key 报警——**这是误报级别**（谷歌官方文档明说这类 key 可公开），但顺手加固不亏：apikeys API 给 key 加 Android 应用限制（包名 + 签名 SHA1）。取 debug.keystore 的 SHA1 有个小坑：**现代 keytool 生成的是 PKCS12 不是 JKS**，pyjks 之类的库直接读不了，用 openssl：

```bash
openssl pkcs12 -in debug.keystore -nokeys -passin pass:android \
  | openssl x509 -fingerprint -sha1 -noout
```

## 第三步：APK 接线（三处，全在 CI 里）

1. `package.json` 加 `@capacitor/push-notifications`
2. `google-services.json` 放仓库里，workflow 在 `cap add android` 之后 `cp` 到 `android/app/`
3. root `build.gradle` 补 `com.google.gms:google-services` classpath（sed 注入）

好消息：**Capacitor 的 android 模板自带「检测到 google-services.json 就 apply 插件」的逻辑**，app 级 build.gradle 一个字不用改。CI 里照旧 grep 验证两个文件的接线都在，缺了直接 fail——比装完发现收不到通知好排查一万倍。

## 第四步：前端注册

在线壳的好处这时兑现：前端改动 git push 部署，**不用出新包**（插件已在原生层，逻辑全在 JS）。

- 检测：`window.Capacitor?.isNativePlatform?.()` 为真才走原生分支，设置页的推送开关按此分流（浏览器仍走 Web Push）
- 流程:checkPermissions → requestPermissions → register → 监听 `registration` 事件拿 token → POST 给服务端 upsert 进 `fcm_tokens` 表
- **给 register 加超时**（我们 20 秒 race）：没装 Google Play 服务、或它被墙时，registration 事件永远不来，没超时就是无限转圈
- 服务端响应里带 `fcmConfigured` 布尔——前端据此区分「token 上报成功」和「上报了但服务器根本没配密钥」，后者直接 throw，别让用户以为开成功了

## 验证阶梯（不用真机就能测到倒数第二级）

1. **假 token 发送**：报 403 → 说明认证链没通（查 IAM）；报 `400 registration token invalid` → **认证/API/权限全通**，只差真 token
2. 真机开开关 → 查库确认 token 入库
3. `send-fixed` 发一条，`sent` 计数 = Web Push 订阅数 + FCM token 数则双通道齐发
4. 用户通知栏见到 = 验收

## 用户侧三件事（写进使用说明）

- **分应用代理用户**：FCM 走的是 **Google Play 服务**的长连接，不是你的 app——代理名单里勾了你的 app 没用，得勾 Google Play 服务（19 章的教训在推送上再现）
- 电池优化把 app 设为不限制，否则厂商系统可能延迟到达
- 旧签名冲突（19 章的坑）在这里再收一次尾：固定 keystore 之前装的包，覆盖安装报冲突，卸载重装一次即愈

## 检查清单

- [ ] Firebase 项目 + Android 应用（包名与 APK 一致）
- [ ] 服务账号密钥落服务器 secrets/，600 权限，git 忽略
- [ ] SA 显式绑 `roles/firebasecloudmessaging.admin`
- [ ] gone 判定覆盖 404 / UNREGISTERED / 400+registration token，假 token 实测
- [ ] access token 缓存，别每条消息换一次
- [ ] CI：cp google-services.json + classpath 注入 + grep 验证
- [ ] 前端 register 加超时，fcmConfigured=false 要 throw
- [ ] API key 加 Android 应用限制（包名 + SHA1）
- [ ] 用户说明：Google Play 服务进代理名单 + 电池白名单
