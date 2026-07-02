# 12 — 共读书架：两个人（一人一 AI）在同一本书上划线

## 这是什么

一个轻量的共读系统：把书上架到记忆库后端，人和 AI 都能在正文上划线、写批注，互相能看到对方留下的痕迹，还有一个共享书签记录读到哪了。

- 前端跑在 GitHub Pages（零服务器负担）
- 数据存在记忆库后端（SQLite，只加四张表）
- AI 侧用 curl 就能留批注，不需要打开浏览器

## 数据模型

```sql
books        (id, title, author, intro, cover_color, added_by, created_at)
book_chapters(book_id, idx, title, content, UNIQUE(book_id, idx))
book_annotations(book_id, chapter_idx, start_off, end_off, quote, author, color, note, created_at)
book_bookmarks(book_id PRIMARY KEY, chapter_idx, updated_by, updated_at)  -- 每本书一个共享进度
```

设计要点：

- **正文按章存储，按章单取**。书详情接口只返回目录+每章字数，点开某章才拉正文，几百 KB 的书也不卡。
- **划线锚定 = 章内字符偏移 + 引文双保险**。`start_off/end_off` 定位，`quote` 存原文片段兜底（正文若被修订还能人工对上）。

## 划线锚定的核心实现

这是整个系统最容易做错的地方。要让「选中的文字」精确映射到「存储正文的字符偏移」，前提是**渲染文本与存储正文逐字一致**：

1. 正文用**单个** `white-space: pre-wrap` 容器渲染，不做任何 markdown/HTML 加工
2. 监听 `selectionchange`，用 Range 计算选区起点在容器内的字符偏移：

```javascript
const range = sel.getRangeAt(0)
const pre = range.cloneRange()
pre.selectNodeContents(containerEl)
pre.setEnd(range.startContainer, range.startOffset)
const start = pre.toString().length      // 选区起点的字符偏移
const quote = range.toString()
// 存 { start, end: start + quote.length, quote }
```

3. 渲染划线时，把所有批注的 start/end 收集成边界点，把正文切成 segment，落在批注区间内的 segment 包 `<mark>`。多条划线重叠也能正确渲染。

中文注意：只要文本都在 BMP 内（绝大多数中文场景），Python 的 `len()` 偏移和 JS 的 UTF-16 偏移一致——AI 侧脚本算好偏移直接 POST 即可：

```bash
curl -X POST http://127.0.0.1:PORT/books/1/annotations \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"chapter_idx":0,"start_off":91,"end_off":130,"quote":"...","author":"AI名","color":"blue","note":"..."}'
```

## 自助上架（txt 上传）

让用户自己传书，不用每次麻烦 AI：

- **编码检测**：国内 txt 大多是 GBK，直接 `readAsText` 必乱码。正确姿势：

```javascript
const buf = await file.arrayBuffer()
try { text = new TextDecoder('utf-8', { fatal: true }).decode(buf) }
catch { text = new TextDecoder('gbk').decode(buf) }
```

- **自动分章**：行首正则识别 `第[数字/中文数字]+[章回卷节集部篇]` 和 `序章|楔子|引子|前言|后记|尾声|终章|番外`。识别不到 2 个标题就整本一章；无标题且超长则按段落边界每 ~1 万字切「第N部分」。
- **归一化要在存储前做**（`\r\n → \n`、全角空格），否则前端渲染文本和存储偏移对不上，所有划线错位。

## 后端的三个坑

1. **大请求体**：整本 txt 可能几 MB，全局 `express.json({ limit: '5mb' })` 会拦。给上架路由单独配大上限 parser，并在全局 parser 中间件里跳过该路径——两边都要做，全局 parser 先跑到就没有然后了。
2. **CORS 缺 PUT**：书签接口用 PUT，如果 CORS `methods` 列表里没有它，从 GitHub Pages 跨域调用会被预检（OPTIONS）拦下，报错还很含糊。加之前先 `curl -X OPTIONS` 验证预检返回 204。
3. **反代白名单**：如果反代（Caddy/Nginx）按路径白名单放行 API，新增的 `/books /books/*` 要记得加，否则前端永远拿到空响应或 404，还以为是代码问题。

## 附赠的坑：端口被幽灵进程抢占

我们部署时遇到 `EADDRINUSE` 反复出现：杀掉监听进程后端口又被占，而且占用者跑的是**旧代码**。真凶是很久之前用 root 跑过一次 pm2，留下了 `pm2-root.service`，里面挂着一份旧拷贝，每次服务重启都跟正常实例抢端口（root 侧重启计数 500+，意味着这场无声的抢椅子游戏持续了几周）。

排查线索：`pm2 pid <app>` 返回 0 但端口有人听 → `ps aux | grep server.js` 看进程属主。根治：

```bash
sudo pm2 delete all && sudo pm2 save --force
sudo systemctl disable --now pm2-root.service
```

教训：永远不要用 root 跑 pm2「试一下」，它会用 systemd 把自己钉进开机自启。
