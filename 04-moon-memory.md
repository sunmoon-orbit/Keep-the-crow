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

暴露给 CC 的 MCP 工具示例：

| 工具名 | 作用 |
|--------|------|
| `write_memory` | 写入一条新记忆 |
| `search_memories` | 关键词搜索记忆 |
| `semantic_search` | 语义相似度搜索（需要 embedding 模型） |
| `get_memory` | 按 ID 获取记忆详情 |
| `list_memories` | 列出最近记忆 |
| `get_stats` | 统计记忆库概况 |
| `get_current_time` | 获取当前时间（注入上下文） |

---

## REST API

```
POST   /memories          创建记忆
GET    /memories          列出记忆（支持分页、过滤）
PATCH  /memories/:id      更新记忆
DELETE /memories/:id      删除记忆

GET    /context           获取当前时间等上下文（公开）
POST   /backup/trigger    手动触发备份到 GitHub
GET    /backup/status     查看备份状态
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

## 数据备份

moon-memory 支持自动备份到 GitHub（每天凌晨定时，或手动触发）。

备份用 `VACUUM INTO` 生成 SQLite 一致性快照，绝不直接 copy 数据库文件（WAL 模式下直接 copy 会丢数据）。

```bash
# 手动触发备份
curl -X POST http://127.0.0.1:3210/backup/trigger \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

---

## 启动

```bash
cd moon-memory
npm install
pm2 start server.js --name moon-memory
```
