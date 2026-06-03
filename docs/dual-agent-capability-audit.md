# 双代理能力全景审计方法论

> 配合 `hermes-openclaw-migration` 第六阶段使用。在基本 10 步检查 + extended audit 后执行此深度审计，产出完整的双代理协作方案。

## 适用场景

- 迁移后首次全面评估两个 Agent 的能力矩阵
- 设计协同工作流前需了解各自边界
- 系统资源紧张时评估优化空间
- 定期审计（建议每月一次）

---

## 阶段 A：进程与资源审计

### 1. 进程清单
```bash
ps aux | grep -E 'hermes|openclaw' | grep -v grep
```
记录：PID、内存（RSS）、CPU%、启动时间、命令行参数

### 2. 系统资源基线
```bash
free -h && nproc && df -h /
```
记录：总内存、可用内存、CPU 核数、磁盘使用率

### 3. 资源占比计算
- 各进程内存 / 总内存 = 百分比
- 总 AI 进程内存 / 总内存 = 系统 AI 负载
- 判断：<50% 安全，50-80% 警戒，>80% 需优化

---

## 阶段 B：配置深度审计

### Hermes（~/.hermes/config.yaml）

| 检查项 | 关键字段 | 目标值 |
|--------|---------|--------|
| 默认模型 | `model.default` + `model.provider` | 应与 SOUL.md 一致 |
| 搜索后端 | `tools.web_search.provider` | tavily / firecrawl 等 |
| Web 提取后端 | `web.extract_backend` | 应与搜索后端配套 |
| 委派模型 | `delegation.model` | 应与主力模型一致 |
| 委派并发 | `delegation.max_concurrent_children` | 3（1-2核）或更高 |
| CWD | `terminal.cwd` | 非 Windows 路径 |
| 记忆引擎 | `memory.provider` | holographic |
| 记忆容量 | 读 MEMORY.md/USER.md 字数 | MEMORY ≤2200, USER ≤1375 |

### OpenClaw（~/.openclaw/openclaw.json）

| 检查项 | 关键字段 | 目标值 |
|--------|---------|--------|
| 默认模型 | `agents.defaults.model` | deepseek/deepseek-v4-pro |
| 上下文窗口 | `agents.defaults.contextTokens` | 500000（适配 1M 模型） |
| 超时 | `agents.defaults.timeoutSeconds` | ≥1800 |
| 压缩 | `agents.defaults.compaction` | mode=safeguard, timeout≥600 |
| 搜索 | `tools.web.search.provider` | 需在国内可达 |
| 工具 profile | `tools.profile` | coding |

### 安全扫描（重要！）

**检查 openclaw.json 中是否有明文 key：**

```bash
grep -nE '(apiKey|appSecret|token|secret).*:.*"[a-zA-Z0-9_-]{20,}"' ~/.openclaw/openclaw.json
```

常见暴露位置：
- `channels.feishu.appSecret` — 飞书 App Secret
- `gateway.auth.token` — Gateway 认证 Token
- `plugins.entries.*.config.*.apiKey` — 各种插件 API key
- `models.providers.*.apiKey` — 不应直接写明文，应引用 `${ENV_VAR}`

⚠️ 如果发现明文 key：记录位置和 key 前缀（不完整回显），通知老板决定是迁移到 `.env` 还是保持（trade-off：安全 vs 启动可靠性——部分 OpenClaw 版本在计划任务上下文中无法读取 env var，明文 key 反而是唯一可用方式）。

---

## 阶段 C：OpenClaw Agent 级联审计

### 检查每个子 Agent 的 SKILL.md

```bash
for agent in main research reviewer explore; do
  echo "=== $agent ===" 
  cat ~/.openclaw/agents/$agent/agent/SKILL.md 2>/dev/null | head -5
done
```

验证点：
- Agent 角色定义是否与预期一致
- 反幻觉铁律是否存在（research/reviewer 必须）
- 数据保真规则是否存在
- 上下文保护规则是否存在

### 检查 Skills 启用状态

读 `openclaw.json` → `skills.entries`，统计：
- `enabled: true` 数量
- `enabled: false` 数量
- 关键 skill 是否启用（至少：arxiv-cli-tools）

### 检查插件

读 `openclaw.json` → `plugins.entries`，确认：
- feishu: enabled
- memory-core: enabled
- 关键 provider 插件（deepseek 等）

### 检查 Cron

```bash
cat ~/.openclaw/cron/jobs.json | python3 -m json.tool
```

验证：
- 周期性任务是否 enabled
- schedule 是否正确
- delivery 目标是否可达

### 检查 Memory

- MEMORY.md 是否有内容（空=失忆）
- 有 dreaming 系统的，检查最近 dreaming 日记：
  ```bash
  ls -lt ~/.openclaw/workspace/memory/dreaming/deep/ | head -3
  ```

---

## 阶段 D：能力矩阵构建

### 维度定义

| 维度 | 指标 | 数据来源 |
|------|------|---------|
| 推理深度 | 默认模型+上下文窗口 | config |
| 搜索能力 | 搜索后端+连通性 | config + curl 实测 |
| 记忆持久性 | 记忆系统类型+内容量 | read MEMORY.md/USER.md |
| 自动化能力 | Cron 数量+类型 | cronjob list |
| Skills 丰富度 | Skills 数量+关键覆盖 | ls skills dir |
| 子代理能力 | 委派并发数+Kanban | config delegation |
| 飞书集成 | 连接模式+凭证状态 | config + 日志 |
| 多媒体 | VLM/TTS/生图 | toolsets enabled |

### 输出格式

用 ✅/⚠️/❌ 三态标记，每个维度给出两列对比。

---

## 阶段 E：协同工作流设计

在此阶段，基于前四个阶段的发现，设计具体协同场景：

1. **分工原则** — 哪方主推理、哪方主执行
2. **场景化流水线** — 学术研究、公文写作、日常管理各怎么配合
3. **避免重复** — 哪些能力只在一方启用就够了
4. **互保机制** — 一方崩了另一方怎么救

---

## 阶段 F：安全检查清单

### 明文 Key 扫描（必须）

| 文件 | 扫描命令 | 风险 |
|------|---------|------|
| openclaw.json | `grep -nE 'apiKey.*"[a-zA-Z0-9_-]{20,}"'` | 🔴 |
| auth-profiles.json | `grep -n 'key' ~/.openclaw/agents/*/agent/auth-profiles.json` | 🟡（此文件不应被版本控制）|
| .env | `grep -c '_API_KEY\|_TOKEN\|_SECRET'` | 确认引用完整性 |
| config.yaml | 确认所有 api_key 都是 `${ENV_VAR}` 引用 | 🟡 |

### 互保可行性

```bash
# 验证 Hermes 可以重启 OpenClaw
which openclaw && echo "OK: openclaw binary found"

# 验证 OpenClaw 可以重启 Hermes  
which hermes && echo "OK: hermes binary found"
```
