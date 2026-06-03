---
name: hermes-openclaw-migration
description: Hermes + OpenClaw 服务器迁移全流程。从 Windows 迁移到 Ubuntu，包含数据打包、传输、安装、飞书集成。触发：迁移 Hermes、迁移 OpenClaw、换服务器、部署到新机器。
version: 1.0.0
---

# Hermes + OpenClaw 服务器迁移

## 适用场景
从一台服务器迁移 Hermes 和 OpenClaw 到另一台（Windows → Linux，或其他跨平台迁移）。

## 前置条件
- 源服务器和目标服务器都可访问
- 目标服务器有 Python 3.12+ / Node.js 22+ / pip / npm
- 飞书开放平台权限（创建企业自建应用）

---

## 第一阶段：数据分级打包

### 数据分级

| 级别 | 内容 | 处理 |
|------|------|:--:|
| 🔴 核心 | config.yaml, SOUL.md, auth.json, cron/, memories/, skills/(不含.hub), openclaw.json, agents/, memory/, workspace/ | 打包直传 |
| 🟡 可选 | sessions/, logs/ | 建议传 |
| 🟢 重建 | .hub/(skill 缓存 ~1.3GB) | 新服务器重装 |

### 打包命令（Windows → tar.gz）

```bash
cd /c/Users/Administrator && tar -czf hermes-openclaw-core.tar.gz \
  --exclude='.hermes/skills/.hub' \
  --exclude='.hermes/sessions' \
  --exclude='.hermes/logs' \
  --exclude='.hermes/*.lock' --exclude='.hermes/*.pid' --exclude='.hermes/*.db' \
  --exclude='.openclaw/logs' --exclude='.openclaw/backups' --exclude='.openclaw/cron' \
  .hermes/SOUL.md .hermes/config.yaml .hermes/auth.json .hermes/.env .hermes/cron .hermes/memories .hermes/skills \
  .openclaw/openclaw.json .openclaw/agents .openclaw/memory

# 可选：sessions + logs
tar -czf hermes-sessions-logs.tar.gz .hermes/sessions/ .hermes/logs/ .openclaw/logs/

# 可选：workspace
tar -czf openclaw-workspace.tar.gz .openclaw/workspace/
```

**注意：Windows tar 会保留 NTFS 路径结构，但 Linux 解压后可能有路径混乱（如 `C:\Users\...` 路径名变成文件名）。**

---

## 第二阶段：传输

### SSH 密钥设置（首次）
```bash
# 本地生成密钥
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "migration"

# 把公钥给老板 → 腾讯云控制台绑定到实例
# 或手动添加到目标服务器：cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# 测试连接
ssh -i ~/.ssh/yilin.pem ubuntu@<目标IP> "echo connected"
```

### 验证性传输（防数据泄露）
```bash
# 先传 104 字节测试文件确认通道正确
echo "TEST" | ssh ubuntu@<IP> "cat > /home/ubuntu/migration-test.txt"
# 老板在目标服务器确认文件存在后再传大包
```

### 传数据
```bash
scp -i ~/.ssh/key.pem *.tar.gz ubuntu@<IP>:/home/ubuntu/
# 传完验证 MD5
md5sum *.tar.gz                        # 本地
ssh ubuntu@<IP> "md5sum /home/ubuntu/*.tar.gz"  # 远端
```

---

## 第三阶段：安装

### 环境依赖（Ubuntu 24.04）
```bash
# pip + Node.js 22
sudo apt install -y python3-pip
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash
sudo apt install -y nodejs
```

### Hermes
```bash
# PyPI 直接安装（国内可用）
pip3 install --break-system-packages hermes-agent
# 验证：hermes --version

# 加 PATH（如果提示找不到命令）
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
```

### OpenClaw
```bash
# ⚠️ 必须用 sudo，ubuntu 24.04 的 --user prefix 会权限不足
sudo npm install -g openclaw@2026.5.26
# 不装 5.27 — 有飞书 dispatch 崩溃 bug
```

### 解压数据覆盖配置
```bash
cd /home/ubuntu
tar -xzf hermes-openclaw-core.tar.gz
tar -xzf hermes-sessions-logs.tar.gz
tar -xzf openclaw-workspace.tar.gz
```

---

## 第四阶段：配置修正

### OpenClaw 路径修正
```bash
# 删除 Windows 路径制造的 BOOTSTRAP.md 幽灵文件
find /home/ubuntu/.openclaw -name 'BOOTSTRAP.md' -delete
```

### API Key 配置
```bash
# 从 models.json 提取 API key 写入 .env（OpenClaw 启动需要）
python3 -c "
import json
with open('/home/ubuntu/.openclaw/agents/main/agent/models.json') as f:
    d = json.load(f)
keys = {}
for p in ['volcengine-plan','volcengine','deepseek']:
    ak = d['providers'][p].get('apiKey','')
    if ak and not ak.startswith('\$'):
        keys[p] = ak
with open('/home/ubuntu/.openclaw/.env','w') as f:
    if 'volcengine-plan' in keys:
        f.write(f'VOLCENGINE_API_KEY={keys[\"volcengine-plan\"]}\n')
    if 'deepseek' in keys:
        f.write(f'DEEPSEEK_API_KEY={keys[\"deepseek\"]}\n')
print('done')
"
```

### 用户身份（必须）
**所有操作要用 `sudo -u ubuntu`，别用 root！**

临时 nohup 启动（仅用于迁移验证，正式部署用 systemd — 见下方 systemd 章节）：
```bash
sudo -u ubuntu bash -c 'export PATH=$HOME/.local/bin:/usr/bin:$PATH && nohup hermes gateway run > /home/ubuntu/hermes-gateway.log 2>&1 &'
sudo -u ubuntu bash -c 'export PATH=$HOME/.local/bin:/usr/bin:$PATH && nohup openclaw gateway run --force > /home/ubuntu/openclaw-gateway.log 2>&1 &'
```

**正式部署用 systemd**：安装 systemd unit 文件后，两个 Gateway 崩溃自动恢复、开机自启。详见下方「systemd 自启动配置」章节。

---

## 第五阶段：飞书集成

### 创建飞书应用（两个独立 App）

Hermes 和 OpenClaw 各需独立飞书 App。去 [open.feishu.cn](https://open.feishu.cn) 创建企业自建应用。

### 飞书应用配置清单（每个 App 都要）

| 步骤 | 位置 | 操作 |
|------|------|------|
| 1. 添加能力 | 添加应用能力 | 勾选「机器人」 |
| 2. 权限-消息 | 权限管理 | `im:message` 获取与发送消息 |
| 3. 权限-机器人回复 | 权限管理 | `im:message:send_as_bot`（应用身份） |
| 4. 权限-群聊 | 权限管理 | `im:chat:readonly`（应用身份） |
| 5. 事件订阅 | 事件与回调 | 添加 `im.message.receive_v1`（应用身份） |
| 6. 允许用户 | 应用可用性 | 所有人可见 |
| 7. 发布 | 版本管理与发布 | 创建版本 → 上线 |

**关键坑：**
- 权限分「用户身份」和「应用身份」两种。机器人发消息需要「应用身份」的 `im:message:send_as_bot`
- 只开 `im:message` 不够——那是用户身份权限，能收不能回
- 每次改权限后必须重新发布版本才能生效

### 写入飞书凭据

**Hermes**（改 ~/.hermes/.env）：
```bash
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=xxxx
FEISHU_ALLOW_ALL_USERS=true
```

**OpenClaw**（改 openclaw.json channels.feishu）：
```json
{
  "enabled": true,
  "appId": "cli_xxxx",
  "appSecret": "xxxx",
  "dmPolicy": "open",
  "groupPolicy": "open"
}
```

### 安装飞书插件（OpenClaw）
```bash
sudo -u ubuntu bash -c 'cd /home/ubuntu/.openclaw && openclaw plugins install feishu'
```

---

## 踩坑记录

| 坑 | 现象 | 根因 | 修复 |
|------|------|------|------|
| GitHub 在腾讯云不可达 | git clone 超时 | 直连网络无 VPN | 用 PyPI pip install 代替 |
| root vs ubuntu 用户 | 找不到配置/进程启动失败 | root 家目录是 /root 而非 /home/ubuntu | 所有操作用 `sudo -u ubuntu` |
| Windows 路径混入 tar | `C:\Users\Administrator\` 变成文件路径 | Windows tar 保留完整路径 | `find -name 'C:*' -delete` |
| API Key 被 LLM 过滤 | .env 写入 `***` 而非真实 key | 系统安全拦截 API key 明文 | 用 Python 本地脚本提取 models.json |
| OpenClaw 5.27 bug | 飞书 dispatch 崩溃 | 版本兼容问题 | 降级到 2026.5.26 |
| BOOTSTRAP.md 残留 | 松果启动失忆 | 残留初始化标志 | 删除该文件 |
| 飞书权限类型错误 | 能收不能回 | 开了「用户身份」而非「应用身份」 | 重新开通应用身份权限 |
| 事件未订阅 | 显示"不能发消息" | 未加 receive_v1 事件 | 事件与回调中添加 |
| dmPolicy 白名单 | 消息被 block | 新飞书 open_id 不在 allowFrom | 改为 open 或加入白名单 |
| npm 全局安装权限 | EACCES | ubuntu 24.04 默认 npm prefix 保护 | 用 sudo npm install -g |
| terminal.cwd 残留 | 所有 terminal 调用报 `cd: C:\\Users\\...: No such file or directory` | config.yaml 的 `terminal.cwd` 仍是 Windows 路径 | `hermes config set terminal.cwd /home/ubuntu` |
| web_search 不可用 | `Error searching web: Feature 'search.firecrawl' unavailable` + `externally-managed-environment` | firecrawl-py 未安装；Python 3.12+ PEP 668 禁止全局 pip install | `pip install firecrawl-py --break-system-packages` 或在虚拟环境中安装 |
| fact_store 清空 | `fact_store list` 返回 0 条 | Holographic 记忆库在迁移中丢失（MEMORY.md/USER.md 文本保留） | 从 MEMORY.md+飞书文档重建；新事实会在使用中自然积累 |
| OpenClaw workspace 幽灵目录 | `~/C:\\Users\\Administrator\\.openclaw\\workspace/` 存在且含文件 | Windows tar 解压时把完整 NTFS 路径当成目录名 | 确认 `/home/ubuntu/.openclaw/workspace/` 内容完整后 `rm -rf` 删除 |
| openclaw.json 明文 Key | Brave API Key、飞书 App Secret、Gateway Token 等以明文写在配置中 | 旧配置习惯；迁移后需审计 | 迁移到 `.env` 环境变量引用；注意 OpenClaw 在计划任务上下文中可能无法读取 env var（参见 openclaw-admin skill 的 SecretRefResolutionError） |
| 国内服务器网络约束 | Google / Wikipedia / DuckDuckGo / arXiv API 被墙，web_search 后端（如 exa）可能不可达 | 腾讯云轻量服务器无 VPN/代理出口 | 搜索后端改用 Tavily（API 走海外节点，国内可达）+ AnySearch（学术垂类）；Firecrawl 作为备选但需验证额度 |
| web_search backend 不可用 | `web_search` 返回 `Feature 'search.firecrawl' unavailable` 或超时 | 原后端（firecrawl/exa）在国内不可达或包未安装 | 切换为 Tavily：`hermes config set tools.web_search.provider tavily`；确认 `TAVILY_API_KEY` 在 `.env` 中；验证 `curl -s https://api.tavily.com --max-time 5` 返回 200 |
| OpenClaw 搜索后端失效 | Exa/Brave API key `INVALID_API_KEY`，松果无法搜索 | 原 key 已过期 | `tools.web.search.provider: "exa"` → `"tavily"` + 将 `TAVILY_API_KEY` 从 Hermes .env 复制到 OpenClaw .env（两套 .env 互相独立） |
| `gateway install` sudo 失败 | `hermes gateway install --system` 报 `ModuleNotFoundError`；`openclaw gateway install` 报找不到 config | sudo 丢失 PATH 和 HOME 环境变量 | 手动写 systemd unit 文件 → `sudo cp` 到 `/etc/systemd/system/`（详见 systemd 自启动配置章节） |

---

## 第六阶段：迁移后验证清单（10 步系统审计）

执行完迁移后，必须逐项确认以下状态。这是基于实际迁移踩坑总结出来的固定检查表。

| # | 检查项 | 操作 | 通过标准 |
|:--:|------|------|------|
| 1 | **核心配置文件** | 读 `config.yaml`、`SOUL.md`、`auth.json` | config 完整可读、SOUL.md 行数≥150、auth.json 存在且受保护 |
| 2 | **Model/Provider** | `hermes model` + 检查 `model.default` 和 `model.provider` | 模型名和 provider 与 `.env` 中 key 匹配 |
| 3 | **持久记忆** | 读 `~/.hermes/memories/MEMORY.md` + `USER.md` + `fact_store list` | MEMORY.md/USER.md 非空（≥10 行）；fact_store 可能为空需重建 |
| 4 | **Skills 清单** | `ls ~/.hermes/skills/ | wc -l` | ≥50 个目录；关键 skill（feishu-integration、academic-paper 等）存在 |
| 5 | **Cron 任务** | `cronjob list` | 任务数 ≥ 10；周期任务（如 DHA 提醒）enabled=true |
| 6 | **项目体系** | `ls ~/projects/` + `ls ~/papers/` | 如源服务器有此目录则必须存在；不存在则需重建 |
| 7 | **飞书集成** | 检查 `.env` 中 FEISHU_* 变量 + 发送测试消息 | 凭据齐全；消息能收发 |
| 8 | **工具链** | `terminal(pwd)` + `web_search("test")` | terminal 不报 cd 错误；web_search 返回结果（非 pip install 错误） |
| 9 | **Gateway 进程** | `ps aux | grep -E 'hermes|gateway'` | Hermes + OpenClaw Gateway 均在运行 |
| 10 | **OpenClaw 残留** | `ls ~/C:\\\\* 2>/dev/null` | 无 Windows 路径残影；workspace 在正确位置 |

### 验证后输出格式

用以下三态标记汇总：

```
✓ 正常    — 通过检查
✗ 必须修复 — 影响正常使用
⚠ 建议修复 — 不影响当前但应尽快处理
```

### 迁移后常见空壳 Skill 修复

部分 Skill 目录在 tar 打包/解压时可能变成空壳（目录存在但无文件）。**务必检查以下高价值 Skill**：

| Skill | 文件数（正常） | 说明 |
|-------|:---:|------|
| `database-lookup` | ≥78（含 SKILL.md + 78 个数据库 reference） | 78 个科学数据库直查——科研效率核武器 |
| `scientific-visualization` | ≥10（含 SKILL.md + scripts + assets） | 论文级图表生成 |

**修复方法**：从 Windows 旧服务器拉取（如果还在运行）：
1. 在旧服务器打包：`tar czf /tmp/database-lookup.tar.gz -C ~/.hermes/skills database-lookup/`
2. 上传到飞书云盘（用 lark-cli）
3. 在新服务器下载并解压：`cd ~/.hermes/skills && tar xzf /path/to/database-lookup.tar.gz`

验证：`find ~/.hermes/skills/database-lookup -type f | wc -l` 应 ≥ 78。

**扩展清单**：更详细的子检查项（模型表一致性、SOUL.md vs config 差异、OpenClaw config 完整性等）见 [`references/post-migration-audit-extended.md`](references/post-migration-audit-extended.md)。

**双代理能力全景审计**：10 步基础检查 + extended audit 通过后，如需设计协同工作流或评估完整能力矩阵，见 [`references/dual-agent-capability-audit.md`](references/dual-agent-capability-audit.md)。包含：进程与资源审计、配置深度审计（含安全扫描）、OpenClaw Agent 级联审计、能力矩阵构建、协同工作流设计。

---

## systemd 自启动配置

迁移完成后，用 systemd 管理两个 Gateway 的自动启动和崩溃恢复。

### ⚠️ 内置 `gateway install` 在 sudo 下会失败

两个工具都有内置的 `gateway install --system` 命令，但在 Ubuntu 上用 `sudo` 执行时都会失败：

| 工具 | 命令 | 失败原因 |
|------|------|------|
| Hermes | `sudo hermes gateway install --system` | `ModuleNotFoundError: hermes_cli` — sudo 的 Python 路径不包含用户 pip 安装的模块 |
| OpenClaw | `sudo openclaw gateway install --force` | sudo 丢失 HOME 环境变量 → 找不到 `~/.openclaw/openclaw.json` + `systemctl enable` 失败 |

**正确方案**：手动写 systemd unit 文件。

### Hermes Gateway Unit

创建 `/etc/systemd/system/hermes-gateway.service`：

```ini
[Unit]
Description=Hermes Gateway
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
EnvironmentFile=/home/ubuntu/.hermes/.env
ExecStart=/home/ubuntu/.local/bin/hermes gateway run
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### OpenClaw Gateway Unit

创建 `/etc/systemd/system/openclaw-gateway.service`：

```ini
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
EnvironmentFile=/home/ubuntu/.openclaw/.env
ExecStart=/home/ubuntu/.local/bin/openclaw gateway run --force --bind lan
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### 部署步骤

```bash
# 1. 停掉旧 nohup 进程（如果有）
pkill -f "openclaw gateway" 2>/dev/null
hermes gateway stop 2>/dev/null

# 2. 安装 unit 文件
sudo cp /tmp/hermes-gateway.service /etc/systemd/system/
sudo cp /tmp/openclaw-gateway.service /etc/systemd/system/

# 3. 重载并启动
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-gateway openclaw-gateway

# 4. 验证
sudo systemctl status hermes-gateway openclaw-gateway --no-pager
```

### 关键参数说明

| 参数 | 说明 |
|------|------|
| `Restart=on-failure` | 进程崩溃后自动重启（exit code ≠ 0） |
| `RestartSec=5` | 崩溃后等 5 秒再拉（防止重启风暴） |
| `EnvironmentFile=` | 从 .env 加载 API Key 等环境变量 |
| `User=ubuntu` | ⚠️ 必须指定，systemd 默认 root 会找不到配置 |

### 常用运维命令

```bash
sudo systemctl status hermes-gateway      # 看状态
sudo systemctl restart hermes-gateway     # 重启
sudo journalctl -u hermes-gateway -f      # 实时日志
sudo journalctl -u hermes-gateway -n 50   # 最近 50 行
```

---

## Cron 进程互检

迁移后设置 Hermes cron 每 5 分钟检查两个 Gateway 进程存活状态。

### 检查脚本

创建 `~/.hermes/scripts/check-gateways.sh`：

```bash
#!/bin/bash
ALERT=""
if ! pgrep -f "openclaw" > /dev/null; then
  ALERT="${ALERT}⚠️ 松果（OpenClaw）进程挂了！\n"
fi
if ! pgrep -f "hermes gateway" > /dev/null; then
  ALERT="${ALERT}⚠️ Hermes Gateway 进程异常！\n"
fi
if [ -n "$ALERT" ]; then
  echo -e "$ALERT请 SSH 检查服务器。"
fi
```

### 创建 Cron 任务

```bash
# 用 Hermes cronjob 工具，no_agent=true 模式（不消耗 LLM token）
hermes cron create \
  --name "互检-检查 Gateway 进程" \
  --schedule "every 5m" \
  --no-agent \
  --script "check-gateways.sh"
```

**设计要点**：
- `no_agent=true` — 纯脚本，不调用 LLM，零 token 消耗
- 脚本输出非空时 cron 自动发飞书通知（空输出 = 静默 = 一切正常）
- 脚本路径必须相对于 `~/.hermes/scripts/`，不能用绝对路径
- OpenClaw 的 cron 系统不支持 no_agent 脚本模式，所以互检集中在 Hermes 侧
- 局限性：如果 Hermes 自己挂了，互检也停摆。但 systemd 会在 5 秒内自动拉起来

---

## 启动与验证

```bash
# 检查 systemd 服务状态
sudo systemctl status hermes-gateway openclaw-gateway --no-pager

# 检查端口监听
ss -tlnp | grep -E '18789|9119'

# 健康检查
curl -s http://localhost:9119/health  # Hermes
curl -s http://localhost:18789/health # OpenClaw

# 检查 systemd 日志
sudo journalctl -u hermes-gateway -n 20
sudo journalctl -u openclaw-gateway -n 20
```
