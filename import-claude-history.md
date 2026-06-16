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
