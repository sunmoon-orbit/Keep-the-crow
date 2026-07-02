# 导入 claude.ai 对话历史 — 操作步骤

每次导出备份后，按以下步骤把历史导入服务器 L0 存档。

---

## 第一步：从 claude.ai 导出数据

1. 打开 claude.ai → 右上角头像 → **Settings**
2. **Account** → **Export data**
3. 点 **Export** → 等邮件（Anthropic 会发一封含下载链接的邮件）
4. 下载 ZIP，解压，找到 `conversations.json`

---

## 第二步：把文件传到服务器

在电脑终端运行（把文件路径替换成实际位置）：

```bash
# Mac/Linux：
scp -i ~/.ssh/id_ed25519 ~/Downloads/conversations.json ripple@100.93.7.53:/tmp/

# Windows PowerShell（在 conversations.json 所在目录运行）：
scp -i $env:USERPROFILE\.ssh\id_ed25519 conversations.json ripple@100.93.7.53:/tmp/
```

**注意：用 Tailscale IP `100.93.7.53`，公网 IP `107.174.40.67` SSH 进不去。**

---

## 第三步：SSH 进服务器，运行导入脚本

```bash
ssh -i ~/.ssh/id_ed25519 ripple@100.93.7.53
```

进去后：

```bash
# 先预览（不写入）
node /home/ripple/moon-memory/scripts/import-claude-ai.js /tmp/conversations.json --dry-run

# 确认没问题，正式导入
node /home/ripple/moon-memory/scripts/import-claude-ai.js /tmp/conversations.json
```

看到 `导入完成：xxx 条对话写入 L0 存档` 就成功了。

---

## 注意事项

- **重复导入是安全的**：每条对话有唯一 ID（external_id），重复导入不会重复写入
- **raven 聊天**：从现在起自动存档，不需要手动操作
- **Claude Code 对话**：数据在服务器本地（`~/.claude/projects/`），暂未自动导入，后续可补

---

## 搜索存档

导入后，在 raven 前端可以让阿言用 `search_archive` 工具搜历史原文，或者直接问他"我们之前聊过……"。

---

## 踩坑补遗（导入大文件/新版导出格式）

### 1. 新版导出的正文可能不在 text 字段

claude.ai 导出格式悄悄变过：消息正文有时在 `content` 数组里而不是顶层 `text` 字段。只读 `text` 的导入脚本会产出一堆**空对话**且不报错。解析时两个都要兜：

```javascript
const text = msg.text || (Array.isArray(msg.content)
  ? msg.content.map(c => c.text || '').join('') : '')
```

另外「导出最近 30 天」有时不含正文，要导就导全部数据。

### 2. 大文件导入 Failed to fetch = 撞了 body 上限

几十 MB 的 conversations.json 直接 POST，会被 express 的 json 上限掐断连接，浏览器只报一句 `Failed to fetch`。放开导入路由的上限只是止痛；根本解法是**客户端分批上传**（每批 10-20 条对话），文件再大也不撞墙。

### 3. 「重复导入安全」的前提是 ID 归一化 + 消息级去重

两个真实事故：

- 老脚本给 external_id 加了前缀（`claude_ai_<uuid>`）、新脚本用裸 uuid——同一对话在库里存了两份，去重完全失配。**导入 ID 的格式定了就不要改**；改了就要写迁移把旧 ID 归一化。
- 对话级幂等 ≠ 消息级幂等：脚本判断「对话已存在就跳过」，但另一条代码路径对已存在对话**追加插入全部消息**——重复导入后每条消息 ×2。去重要做到消息级（对话内容 hash 或消息指纹）。

清理重复时保留内容更全的版本，并记得把批注/标注迁移过去再删。以及：**别拿生产库测试导入脚本**。
