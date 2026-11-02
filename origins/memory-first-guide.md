# 给 AI 伴侣搭一个记忆库
> 低成本、自托管、MCP + REST 双接口

---

## 这是什么

一个跑在自己服务器上的记忆存储系统，让你的 AI 伴侣可以：

- **读取历史记忆**：每次新窗口开始前，AI 能看到你们过去的重要内容
- **写入新记忆**：对话中产生的重要片段可以存起来
- **按类型检索**：日记、交接信、关系核心、日常……分类存、分类取
- **跨平台使用**：Claude、API 聊天工具都能接

---

## 需要什么

### 最低资源
| 资源 | 要求 | 参考成本 |
|------|------|---------|
| VPS | 1 核 1G 内存，10G 硬盘以上 | 谷歌云免费小鸡（e2-micro）够用 |
| 域名 | 可选，有更方便 | Cloudflare 购买约 ¥70/年 |
| Claude Pro | 使用 MCP 连接需要 | ¥140/月左右 |

**谷歌云免费额度**：e2-micro 实例永久免费（美国区），30G 硬盘，够跑记忆库后端 + Caddy + pm2。

### 技术门槛
- 会基本的 SSH 操作
- 能跟着 AI 一步一步做（不会没关系，让 Claude Code 来）

---

## 架构

```
你/AI
  │
  ├── MCP 协议（Claude 用）
  │     └── /sse 端点 → 调用工具：memory_list / memory_write / memory_filter...
  │
  └── REST API（前端/其他工具用）
        └── /memories / /filter / /import / /export...
              │
              └── 数据库（KV 存储 或 SQLite）
```

两种接口跑在同一台服务器上，同一套数据。

---

## 核心模块

- **后端**：Node.js 服务，提供 REST API + MCP SSE 端点
- **数据库**：KV 存储（Cloudflare Workers KV）或 本地 SQLite 均可
- **前端**（可选）：网页界面，查看/写入/导出记忆
- **反向代理**：Caddy（自动 HTTPS，配置简单）
- **进程守护**：pm2，服务挂了自动重启

---

## 搭建步骤（简化版）

### 1. 准备服务器
```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 装 Node.js、npm、pm2
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pm2
```

### 2. 部署后端
把后端代码放到服务器上（可以用 GitHub 仓库），安装依赖，配置环境变量：

```bash
# 主要环境变量
API_TOKEN=你自己设的密钥     # REST API 认证用
PORT=3000
```

用 pm2 启动：
```bash
pm2 start index.js --name memory-server
pm2 save
pm2 startup
```

### 3. 配置 Caddy 反向代理
```
your-domain.com {
    reverse_proxy localhost:3000
}
```

Caddy 自动申请 SSL，https 直接生效。

### 4. 在 Claude 里连接 MCP

Claude 设置 → 连接器 → 添加自定义 MCP：
```
URL: https://your-domain.com/sse
```

连接成功后 Claude 就能调用记忆工具了。

---

## 踩过的坑

### 坑1：Claude Code workspace trust bug（v2.1.143）
**现象**：`claude remote-control` 报 `Workspace not trusted`，接受弹窗后设置没存进去。  
**原因**：`.claude.json` 里 `hasTrustDialogAccepted` 一直是 false，CC 不写入。  
**解法**：打开 `claude`，告诉它"remote-control 报 workspace not trusted，帮我修复"，让 CC 自己找到 `.claude.json` 改掉。

### 坑2：前端 API 路径跟后端对不上
**现象**：前端加载一直转圈，控制台报 404。  
**原因**：手写路径猜错了，比如 `/list` 实际是 `/memories`，`/filter` 实际是 `/memories/filter`。  
**解法**：让 Claude Code 直接读后端路由文件，对照修前端。

### 坑3：MCP 连接 401
**现象**：MCP 工具调用返回 `unauthorized`。  
**原因**：更换了 API_TOKEN 后，Claude 连接器里的认证没跟着更新。  
**解法**：删掉旧连接器，重新添加（把新 token 带进连接配置里）。

### 坑4：记忆备份导入格式不对
**现象**：前端导入旧备份报"文件格式错误"。  
**原因**：旧备份字段名（`category`、`created`）和新后端期望的字段名（`type`、`created_at`）不一致。  
**解法**：用脚本转换字段名，或者让 AI 帮你批量导入（MCP 写入比前端上传更稳）。

---

## 进阶：给 API 聊天工具接记忆（以言叽为例）

如果你有一个自己的前端聊天工具（比如跑在 GitHub Pages 上调 Claude API 的页面），可以接入记忆库让它自动检索：

**核心思路：**
```
用户发消息
  → 前端用消息内容调 /memories/filter 或语义搜索
  → 拿到相关记忆（3-5条）
  → 拼进 system prompt
  → 再调模型 API
```

**省 token 的缓存策略：**
- 相同搜索词 5 分钟内只查一次，命中缓存直接用
- 长期记忆（关系核心、锚点）单独缓存，不随对话刷新

---

## 记忆类型参考

| 类型 | 用途 |
|------|------|
| `memory` | 一般记忆 |
| `daily` | 日常（可设遗忘曲线，久了提示归档） |
| `diary` | 日记 |
| `handoff` | 交接信（窗口结束时写给下一个实例） |
| `deep` | 关系核心、共识 |
| `anchor` | 里程碑 |
| `treasure` | 珍藏 |

---

## 总结

最低成本：**谷歌云免费小鸡 + Cloudflare 域名 ≈ ¥70/年**（域名费）

整套搭下来不需要很强的技术背景，跟着 Claude Code 一步步做就行。最难的部分是第一次 debug，之后基本不用管它，pm2 会自动守着。

---

*写于 2026.5.19 · 涟言 🐦‍⬛*
