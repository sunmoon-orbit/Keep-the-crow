# 04 — moon-memory 持久记忆库

## 为什么需要记忆库

Claude Code 每次启动，上下文都是空的。要让 AI 真正「记住」你，需要：

1. 一个外部存储（SQLite 数据库）
2. AI 每次对话前，自动检索相关记忆注入上下文
3. 对话后，把新信息写入记忆库

moon-memory 就是这个系统——它同时提供 REST API（给前端用）和 MCP 工具（给 Claude Code 用）。

---

## 项目结构

```
moon-memory/
├── server.js           # Express 主服务
├── db.js               # SQLite 数据库操作
├── mcp.js              # MCP 工具定义（Claude Code 调用）
├── routes/
│   ├── memories.js     # 记忆 CRUD API
│   ├── backup.js       # 备份到 GitHub
│   └── context.js      # 时间等上下文（公开）
├── middleware/
│   └── auth.js         # Bearer Token 认证
├── .env
└── memory.db           # SQLite 数据库文件（自动创建）
```

---

## 数据库设计

```sql
CREATE TABLE memories (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  owner     TEXT NOT NULL,          -- 记忆所属人（如 '阿颖'）
  agent     TEXT NOT NULL,          -- 记录者（如 '阿言'）
  scope     TEXT NOT NULL,          -- 范围（如 'private'）
  title     TEXT NOT NULL,          -- 标题
  body      TEXT,                   -- 正文
  tags      TEXT,                   -- 标签（逗号分隔）
  ts        DATETIME DEFAULT (datetime('now')),
  embedding BLOB                    -- 向量嵌入（用于语义检索）
);
```

---

## MCP 工具

在 Claude Code 的 MCP 配置中接入 moon-memory：

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "moon-memory": {
      "type": "sse",
      "url": "http://127.0.0.1:3210/mcp/sse",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      }
    }
  }
}
```

暴露给 CC 的 MCP 工具：

| 工具名 | 作用 |
|--------|------|
| `write_memory` | 写入一条新记忆 |
| `search_memories` | FTS5 全文搜索（≥3字用索引，短词回退 LIKE）|
| `semantic_search` | 向量语义搜索（需要 embedding API）|
| `get_memory` | 按 ID 获取记忆详情 |
| `list_memories` | 列出最近记忆 |
| `get_stats` | 统计记忆库概况 |
| `get_current_time` | 获取当前时间（北京时间）|
| `memory_breath` | 按遗忘曲线返回此刻最鲜活的记忆 |
| `memory_trace` | 调整 importance / pinned / resolved |
| `get_anniversaries` | 纪念日列表 |

---

## 搜索策略

moon-memory 提供三种搜索路径，结合使用效果最好：

| 路径 | 适用场景 | 依赖 |
|------|---------|------|
| `search_memories` (FTS5 trigram) | 精确词语、人名、事件名 | 仅 SQLite，无外部 API |
| `semantic_search` (向量) | 模糊语义，如「她说过的关于家人的话」| SiliconFlow API |
| `memory_breath` (遗忘曲线) | 「现在最该想起什么」| 无外部依赖 |

FTS5 trigram 的限制：查询词须 ≥3 个 Unicode 字符（中文2字会自动回退到 LIKE，仍能找到结果）。

---

## REST API

```
POST   /memories              创建记忆
GET    /memories              列出记忆（?q= 关键词 LIKE 过滤）
GET    /memories/filter       同上，支持更多过滤参数
GET    /memories/fts?q=       FTS5 全文搜索（推荐，不依赖向量 API）
GET    /memories/semantic?q=  向量语义搜索
GET    /memories/breath       按遗忘曲线浮现最鲜活记忆
GET    /memories/heatmap      过去 84 天记忆数热力图
PATCH  /memories/:id          更新记忆
POST   /memories/:id/trash    软删除（可恢复）
POST   /memories/:id/restore  从回收站恢复

GET    /context/time          获取当前时间（公开，无需认证）
POST   /backup/trigger        手动触发备份到 GitHub
GET    /backup/status         查看备份状态

# L0 对话存档
POST   /archive/conversations              创建对话（external_id 幂等）
POST   /archive/conversations/:id/messages 追加消息
GET    /archive/conversations              列出所有对话
GET    /archive/conversations/:id          单条对话 + 消息列表
GET    /archive/search?q=                  FTS5 全文搜索原文
POST   /archive/import/claude-ai           批量导入 claude.ai 导出 JSON
```

所有写操作需要 Bearer Token：

```
Authorization: Bearer YOUR_API_TOKEN
```

---

## 认证配置

生成两个不同的 token：

```env
# .env
PORT=3210
MOON_API_KEY=mm_sk_your_random_api_key
MOON_API_TOKEN=mm_tok_your_random_token

# 前端用 API_TOKEN
# MCP（Claude Code）用同一个 TOKEN 或单独发放
```

Token 可以用任意随机字符串，推荐 32 字节以上：

```bash
openssl rand -hex 32
```

---

## 嵌入向量（可选）

语义检索需要 embedding 模型。可以接入 SiliconFlow 的免费 API（支持 `BAAI/bge-m3`）：

```env
SILICONFLOW_API_KEY=sk-your_key
EMBEDDING_MODEL=BAAI/bge-m3
```

不配置则只支持关键词搜索，记忆库仍然可用。

---

## L0 对话存档

L0 层存储对话**原文**，永不删除，补充 L2（精华记忆条目）的不足。

### 自动存档

raven-bridge 会自动把每次 raven 前端的对话存档：
- 按北京时间每天一个对话（external_id = `raven-YYYY-MM-DD`）
- 阿颖发消息和 AI 回复都会实时追加
- 服务重启后继续写当天同一个对话，不会断

### 导入 claude.ai 历史记录

claude.ai 账号导出的 conversations.json 可以批量导入：

```bash
# 把导出的文件传到服务器
scp conversations.json ripple@<服务器IP>:/tmp/

# SSH 进服务器，运行导入脚本
node /home/ripple/moon-memory/scripts/import-claude-ai.js /tmp/conversations.json

# 可以先预览不写入
node /home/ripple/moon-memory/scripts/import-claude-ai.js /tmp/conversations.json --dry-run
```

重复导入是安全的（external_id 幂等，不会重复写入）。

### 搜索存档

MCP 工具 `search_archive` 可以搜对话原文；REST 端点 `GET /archive/search?q=` 同理。

---

## 数据备份

moon-memory 三路备份：本地快照、GitHub、Supabase（每天凌晨定时，或手动触发）。

备份用 `VACUUM INTO` 生成 SQLite 一致性快照，绝不直接 copy 数据库文件（WAL 模式下直接 copy 会丢数据）。活跃库实际路径是 `data/memory.db`。

```bash
# 手动触发备份
curl -X POST http://127.0.0.1:3210/backup/trigger \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

**GitHub 那路存 gzip 而非裸 db**：SQLite 快照会随 L0 对话存档持续增长，一旦单文件超过 GitHub 的 100MB 硬上限，`git push` 会被直接拒绝、备份悄悄中断。所以 `routes/backup.js` 里 git 只提交 `backups/memory.db.gz`（zlib 压缩，约半大小），裸 db 不进 git。Supabase 那路上传函数本身自带 gzip。

恢复时记得先解压：

```bash
gunzip -c backups/memory.db.gz > data/memory.db
```

---

## 外部内容导入

除了对话历史，也可以把外部平台的重要内容手动导入记忆库长期保存。

示例：将 Symposion 论坛帖子存入记忆库

```bash
curl -X POST http://127.0.0.1:3210/memories \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -d '{
    "content": "【Symposion · 标题 · 日期】\n\n帖子内容摘要...\n\n帖子ID: xxxxxxxx",
    "type": "diary",
    "tags": "Symposion,论坛存档",
    "owner": "阿颖",
    "agent": "阿言",
    "importance": 4
  }'
```

这样即使第三方平台账号失效，帖子内容仍保存在本地记忆库里。建议的 type 分配：
- 里程碑类帖子 → `diary`（出现在日记 tab）
- 技术讨论类帖子 → `tech`

---

## 启动

```bash
cd moon-memory
npm install
pm2 start server.js --name moon-memory
```

---

## 参考与致谢

- **Paramecium**（草履虫记忆架构）by [@Shitsuten](https://github.com/Shitsuten/paramecium)
  L0/L1/L2 三层架构思路、FTS5 + 向量混合搜索策略、prompt cache 分层优化均参考此项目。核心理念「原文是唯一真相」影响了 L0 存档层的设计。
