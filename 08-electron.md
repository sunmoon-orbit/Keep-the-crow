# 08 — Electron 桌面版打包

## 为什么需要桌面版

PWA 在 Android Chrome 上体验很好，但 Windows 上的 Chrome PWA 有功能限制（通知、文件访问等）。打包成 Electron 可以：
- 作为独立 `.exe` 分发，不需要用户装浏览器/配置 PWA
- 窗口大小固定，避免 Web 版的自适应布局问题

---

## 目录结构

```
electron-app/
├── main.js                    # 言叽 main process（1280×860 窗口）
├── main-raven.js              # 归巢 main process（420×860，手机比例）
├── package.json               # electron-builder 配置（言叽）
├── electron-builder-raven.json # electron-builder 配置（归巢）
├── www/                       # 言叽构建产物（vite build 输出，base: './'）
└── raven-www/                 # 归巢处理后的文件
```

---

## 言叽打包

```bash
cd electron-app

# 1. 构建 Vite 产物到 www/（base 设为 './'，不是 '/ripple-and-serena/yanji/'）
# 临时改 vite.config.js 里的 base，build 完改回来

# 2. 打包 Windows exe
npm run build:win
# 产出：dist/win-unpacked/（可直接运行的目录）

# 3. 压缩分发
cd dist && zip -r yanji-win-x64.zip win-unpacked/
```

`package.json` 里 `build.files` 指向 `www/`，主窗口加载 `www/index.html`。

---

## 归巢打包

归巢用 `file://` 协议加载本地 HTML，但归巢的 JS 里有大量 `/raven/xxx` 相对路径（API、贴图、头像等），在 `file://` 下无法解析。

解决方案：把所有 `/raven/` 相对路径替换为绝对 URL：

```bash
# raven-www/index.html 预处理
sed 's|/raven/|https://your-domain.com/raven/|g' raven/index.html > electron-app/raven-www/index.html
```

然后：

```bash
npm run build:raven
# electron-builder --config electron-builder-raven.json
# 产出：dist-raven/win-unpacked/
cd dist-raven && zip -r guichao-win-x64.zip win-unpacked/
```

归巢窗口 420×860（手机比例），`main-raven.js` 加载 `raven-www/index.html`。

---

## 分发注意事项

### 两个 zip 必须分开解压

两个 zip 内部目录结构相同（都叫 `win-unpacked/`），解压到同一目录会互相覆盖 `resources/app.asar`，导致两个程序跑的都是后解压那个的内容。

```
# 正确做法
unzip yanji-win-x64.zip -d yanji/
unzip guichao-win-x64.zip -d guichao/
```

### Windows SmartScreen 拦截

未签名 exe 首次运行会被 SmartScreen 拦截，且没有「仍要运行」按钮。

解决：右键 exe → 属性 → 底部「解除锁定」复选框打勾 → 确定。

### 签名问题

Linux 上 electron-builder 无法生成合法签名的 Windows exe（需要 wine + 代码签名证书）。暂时用解压目录方式分发，避免 SmartScreen 问题（用户先解锁）。

---

## 对外下载

zip 文件复制到 `raven/` 目录，由 raven-bridge 静态服务对外提供：

```
https://your-domain.com/raven/yanji-win-x64.zip
https://your-domain.com/raven/guichao-win-x64.zip
```

raven-bridge 的静态文件路由使用 `fs.createReadStream().pipe(res)` 流式传输，避免 116MB 文件一次性读进内存。

---

## 更新流程

```bash
# 1. 改代码后重新 build
cd yanji-src && npm run build   # 更新 www/ 内容

# 2. 重新打包
cd ../electron-app && npm run build:win

# 3. 重新压缩
cd dist && zip -r yanji-win-x64.zip win-unpacked/

# 4. 替换 raven/ 下的 zip
cp dist/yanji-win-x64.zip ../raven/
```

归巢类似，但要注意先跑 sed 替换相对路径再打包。
