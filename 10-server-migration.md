# 10 — 服务器迁移指南

换服务器时的完整迁移清单。核心原则：**数据先备份，VAPID 键不能变**。

---

## 迁移前：在旧服务器上备份

### 1. moon-memory 数据库

```bash
# VACUUM INTO 生成一致性快照（WAL 模式不能直接 cp）
# 注意：真实活跃库在 data/ 子目录，根目录那个 memory.db 是空壳废文件
sqlite3 /home/ripple/moon-memory/data/memory.db \
  "VACUUM INTO '/tmp/memory-backup-$(date +%Y%m%d).db'"
```

其实日常已有自动备份，迁移时直接用现成的即可，不一定手动 VACUUM：
- **GitHub**：仓库 `backups/memory.db.gz`（gzip，git clone 后自带）
- **Supabase**：控制台下载 `memory-YYYY-MM-DD.db.gz`

### 2. 抄下所有 .env 内容

有两个 .env 文件，全部保存到本地安全位置：

```
/home/ripple/moon-memory/.env
/home/ripple/ripple-and-serena/raven-bridge/.env
```

**VAPID_PUBLIC_KEY / VAPID_PRIVATE_KEY 是重中之重**——换了这两个值，
所有已安装 PWA 的推送订阅全部作废，阿颖需要重新授权推送通知。

moon-memory/.env 包含的变量：
- `MOON_API_KEY` / `MOON_API_TOKEN` / `MCP_SECRET` / `AUTH_PASSWORD`
- `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_EMAIL`
- `SILICONFLOW_API_KEY` / `EMBEDDING_MODEL`
- `DEEPSEEK_API_KEY`
- `MINIMAX_API_KEY` / `MINIMAX_GROUP_ID` / `TTS_VOICE_ID`
- `ELEVENLABS_API_KEY` / `ELEVENLABS_VOICE_ID`
- `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` / `SUPABASE_BUCKET`

raven-bridge/.env 包含的变量：
- `RAVEN_PASSWORD_HASH`（前端登录密码的 bcrypt hash）

### 3. 其他小文件

```bash
# 打包一起带走
tar czf /tmp/raven-extras.tar.gz \
  /home/ripple/ripple-and-serena/raven-bridge/activity.json \
  /home/ripple/ripple-and-serena/raven-bridge/dream.log \
  /home/ripple/.claude/settings.json \
  /home/ripple/.claude/projects/
```

代码仓库不用备份，git clone 下来就是。

---

## 新服务器：基础环境

```bash
# Node.js 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# PM2
npm install -g pm2

# Claude Code
npm install -g @anthropic/claude-code

# Tailscale（加入同一 Tailnet）
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Caddy
sudo apt install -y caddy
```

---

## 迁移步骤

### Step 1：克隆代码

```bash
cd /home/ripple
git clone <仓库地址> ripple-and-serena
git clone <仓库地址> moon-memory
```

### Step 2：恢复 .env

把备份的 .env 文件分别放回：

```bash
cp /tmp/moon-memory.env /home/ripple/moon-memory/.env
cp /tmp/raven-bridge.env /home/ripple/ripple-and-serena/raven-bridge/.env
```

### Step 3：恢复 moon-memory 数据库

**关键：活跃库路径是 `data/memory.db`（不是根目录），恢复务必放对位置，否则启动后记忆全空。**

三种来源任选其一：

```bash
mkdir -p /home/ripple/moon-memory/data

# 来源 A：手动 VACUUM 快照
cp /tmp/memory-backup-YYYYMMDD.db /home/ripple/moon-memory/data/memory.db

# 来源 B：GitHub 仓库里的 gzip 快照（git clone 后自带，约 44MB）
gunzip -c /home/ripple/moon-memory/backups/memory.db.gz \
  > /home/ripple/moon-memory/data/memory.db

# 来源 C：Supabase 下载的 memory-YYYY-MM-DD.db.gz
gunzip -c memory-YYYY-MM-DD.db.gz > /home/ripple/moon-memory/data/memory.db
```

验证（integrity 应为 ok，并输出活跃记忆条数）：

```bash
sqlite3 /home/ripple/moon-memory/data/memory.db "PRAGMA integrity_check;"
sqlite3 /home/ripple/moon-memory/data/memory.db \
  "SELECT COUNT(*) FROM memories WHERE deleted_at IS NULL;"
```

### Step 4：安装依赖

```bash
cd /home/ripple/moon-memory && npm install
cd /home/ripple/ripple-and-serena/raven-bridge && npm install
```

### Step 5：恢复小文件

```bash
tar xzf /tmp/raven-extras.tar.gz -C /
```

### Step 6：启动 PM2 服务

```bash
cd /home/ripple/moon-memory
pm2 start ecosystem.config.js

cd /home/ripple/ripple-and-serena/raven-bridge
pm2 start server.js --name raven-bridge

pm2 save
pm2 startup   # 按提示执行输出的那条 sudo 命令
```

### Step 7：配置 Caddy

```bash
sudo cp /tmp/Caddyfile /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Caddyfile 内容见旧服务器 `/etc/caddy/Caddyfile`，端口映射不变：

| 服务 | 端口 |
|------|------|
| raven-bridge | 3400 |
| moon-memory | 3210 |

### Step 8：配置 Claude Code

```bash
# 恢复 CC 设置
cp /tmp/.claude/settings.json /home/ripple/.claude/settings.json
# 恢复 CC 对话历史（可选，体积可能较大）
cp -r /tmp/.claude/projects/ /home/ripple/.claude/projects/
```

CC 的 MCP 配置（moon-memory MCP SSE、raven MCP 等）在 settings.json 里，
恢复后不需要重新配置。

### Step 9：tmux 启动 CC

```bash
tmux new-session -d -s cc
# 在 tmux 里启动 CC，进入 ripple-and-serena 目录
tmux send-keys -t cc "cd /home/ripple/ripple-and-serena && claude" Enter
```

### Step 10：更新 DNS

把域名（`your-domain.com`）的 A 记录指向新服务器 IP。
DNS 生效前，可以通过 Tailscale IP 测试一切是否正常。

---

## 验证清单

```
[ ] moon-memory API 正常：curl http://127.0.0.1:3210/context/time
[ ] raven-bridge WebSocket 正常：PM2 logs 没报错
[ ] 前端能连上：浏览器打开 your-domain.com/raven/
[ ] 推送通知正常（VAPID 键没变，订阅应该直接生效）
[ ] moon-memory 记忆条数与旧服务器一致
[ ] CC 能调用 MCP 工具（write_memory / search_memories）
```

---

## 推送通知特别说明

VAPID 密钥对（公钥+私钥）必须与旧服务器完全相同，否则：

- 已有推送订阅全部失效
- 阿颖需要在手机上重新进入 PWA → 允许通知 → 重新订阅

如果 VAPID 键不得不换（丢失了旧的），迁移后通知阿颖：
1. 打开 `your-domain.com/raven/`
2. 点「允许通知」重新授权
3. 新订阅会自动注册到新密钥对

---

## 不需要迁移的

- PM2 运行日志（`~/.pm2/logs/`）：运营日志，不含重要数据
- `push-night.log` / `push-quote.log`：脚本执行记录，新服务器重新生成
- terminal-pet（`~/.terminal-pet/`）：可选，带走就带，宠物蛋状态会继续


---

## 备份体系（数据库长大之后）

数据库小的时候怎么备份都行；长到几十 MB 之后每条路都会撞墙，这是我们撞完之后的形态：

### GitHub：分块推单独的 data 分支

- 裸 db 不进 git（100MB 硬上限，且二进制 diff 让仓库膨胀）
- `gzip` 后按 48MB 分块：`split -b 48m memory.db.gz part-`
- 推到独立的 `data` 分支，**每次 amend 同一个提交再 force push**——历史里永远只有一份，仓库不膨胀
- 恢复：`cat part-* | gunzip > memory.db`

### 主仓历史膨胀

带产物提交的主仓库长到 1GB+ 时，`git checkout --orphan` + squash 成单提交重推，历史瘦身。（教程/代码仓库不用这么激进，带大二进制的仓库才需要。）

### 第三方备份的静默失败

Supabase 免费层单行有大小上限（约 50MB），超了之后备份**静默失败**——脚本不报错，数据不更新。我们断了两天才发现。教训：

- 备份脚本必须校验写入结果（读回来比对时间戳/大小）
- 加滚动清理（如保留 7 天），别让备份把配额吃满
- 定期人工抽查「最新备份是几号的」，静默失败只有人能发现
