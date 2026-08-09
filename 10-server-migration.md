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

### 备份演练：你备的到底是不是个能用的库

「备份有没有问题」这个问题，看备份文件的时间戳是答不了的。
**备份最常见的失败模式不是没备，是一直在备、从没试过还原。**

一次完整演练四步，十分钟能做完：

```bash
# 1. gzip 自身完整性
gunzip -t memory.db.gz

# 2. 分块拼回来，和原 gz 是不是同一个东西
cat memory.db.gz.part-* | sha256sum
sha256sum memory.db.gz            # 两个哈希必须一致

# 3. 真的还原出来，让 SQLite 自己判
cat memory.db.gz.part-* | gunzip > /tmp/restore-test.db
sqlite3 /tmp/restore-test.db "PRAGMA integrity_check;"     # 期望 ok

# 4. 抽查内容，和实时库对比
sqlite3 /tmp/restore-test.db "select count(*) from memories;"
```

第 4 步有个容易吓自己一跳的地方：**备份和实时库的数字本来就该不一样**，
差额应该正好等于备份时间点之后新增的量。我们演练时还遇到备份里少了一整张表——
一查是那张表在备份之后才建的，**这是对的，不是缺失**。
所以第 4 步真正要确认的是「差异能被解释」，不是「数字相等」。

（另一个自摆乌龙：查询写错表名报 `no such table`，第一反应是备份坏了。
先 `.tables` 看一眼再下结论。）

---

## 告警设计：先分清「异常」和「有人故意这么设的」

我们有两个会定时产出内容的功能（做梦、独处时间），某天发现它们三周没动静了。
第一反应是加个巡检：超过 14 天没产出就推送提醒。

写完当天问了本人，答案是：**她自己主动关掉的**，因为那阵子换了个入口聊天，
那两个功能用不上了。

于是这个告警从「有用」变成了「唠叨」——对一个人有意做的决定，每 14 天念一次。

问题出在判据选错了。日志里，「用户关掉了开关」和「脚本坏了」长得一模一样：

```
[dream] 总闸关闭，今晚不做梦        ← 她关的
[idle]  开关关着，继续睡            ← 她关的
```

两者都是打印一行然后 `exit 0`。而**开关状态本来就是读得到的**，
只是第一版没去读，图省事拿「有没有产出」当判据。改成：

```js
// 开关关着 → 她的选择，闭嘴
// 开关开着却 14 天没产出 → 这才是真坏了，报
if (on && days >= QUIET_DAYS) broken.push(`${name}开关是开的，但已 ${days} 天没有新的`);
```

> **做告警前先问：我要报的这个状态，有没有可能是有人故意设成这样的？**
> 如果有，就必须找到一个能区分「故障」和「有意为之」的信号，
> 而不是挑那个容易拿到的信号。判据错了的告警比没有告警更糟——
> 它会训练用户忽略通知。

同一套「台阶」写法值得复用：越过阈值报一次，之后每爬升一档才再报，
恢复正常就把档位文件删掉复位。别天天轰炸同一件事。

---

## 日常运维：数据库不会自己瘦

SQLite 有两个「空间不会自动还你」的特性，长期运行必踩：

1. **大批量删除后必须 `VACUUM`**：删几万条重复消息，文件一个字节都不会小——碎片还在文件里。我们的库 126MB VACUUM 完 57MB。
2. **FTS5（尤其 trigram 分词）索引会膨胀**，需要 `INSERT INTO xxx_fts(xxx_fts) VALUES('optimize')` 合并 b-tree。

做成每月 cron（错开备份时间——我们排在备份前一小时，既避免锁冲突，当天备份还能拿到瘦身后的库）：

```bash
# 每月 1 日，备份任务之前
0 2 1 * * cd /path/to/moon-memory && node scripts/vacuum.js >> logs/vacuum.log 2>&1
```

VACUUM 期间服务在线无恙（WAL 模式下读写照常），不用停机。

另一个不还空间的家伙是 git：squash/force-push 之后，**本地旧对象照占磁盘**，要手动 `git reflog expire --expire=now --all && git gc --prune=now`。我们两个仓库回收了 1.2GB。

### 容量心态

先测算再焦虑：纯文本对话的增长比想象小得多——我们全部历史消息正文才 5MB，含全文索引等开销实际增长约 15-20MB/月，22GB 空闲够用几十年。真正吃盘的是备份产物（要有界、滚动清理）和 git 历史（见上）。

---

## 监控：不能和被监控者同生共死

我们装过自托管的 uptime-kuma，它教了两课才被删掉：

1. **任何服务都必须配自启**（pm2 save / systemd enable）。kuma 没配，一次服务器重启后就死了，**近一个月无人察觉**——因为它死了就没人报警「它死了」。
2. 监控者跑在被监控的同一台机器上，整机挂掉时它一起挂，恰好监控不了最致命的情形。

正确形态：外部监控用免费第三方（UptimeRobot / Better Stack）从公网 ping 你的域名；机内健康检查（服务在不在、备份新不新鲜）做成 AI 能调用的巡检工具 + 失败时推送到手机，比养一个监控面板实在。
