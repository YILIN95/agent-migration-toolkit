# 迁移后系统审计 — 扩展子项清单

> 配合 `hermes-openclaw-migration` 第六阶段「迁移后验证清单」使用。
> 基本 10 步检查通过后，用此表做深层验证。

## 1. config.yaml 深层检查

- [ ] `terminal.cwd` 指向正确 Linux 路径（如 `/home/ubuntu`），非 Windows 路径
- [ ] `model.default` 与实际使用的 provider 匹配
- [ ] `model.provider` 对应的 `providers.<name>.base_url` 可达
- [ ] `providers` 下所有 `api_key` 引用 `${ENV_VAR}` 且 `.env` 中该变量存在
- [ ] `delegation` 的 model/provider/base_url/api_key 配置完整且可用
- [ ] `memory.provider` 指定值在 `hermes memory status` 中显示 `available ✓`
- [ ] `tools.web_search.provider` 存在且对应 API key 有效
- [ ] 无重复 key（如同时有 `tools.web_search.provider` 和顶层 `web_search.provider`）

## 2. SOUL.md vs config.yaml 一致性

- [ ] SOUL.md 模型选择规则中的 base_url 与 config.yaml 一致
- [ ] SOUL.md 模型表列出的模型在 config.yaml 的 providers 中均存在
- [ ] SOUL.md 中「可用模型」表与 config 中实际 models 列表同步（版本升级后模型名可能变化）

**常见不一致**：
- SOUL.md 写 `base_url: .../coding/v3`，config 实际是 `.../plan/v3`
- SOUL.md 列 `glm-4.7`，config 里是 `glm-5.1`
- SOUL.md 说「所有模型走 Coding Plan」，config 实际主力是 DeepSeek 官方 API

## 3. .env 变量完整性

- [ ] `DEEPSEEK_API_KEY` — 如果 config 中有 deepseek provider
- [ ] `VOLCENGINE_API_KEY` — 如果 config 中有 volcengine provider
- [ ] `FIRECRAWL_API_KEY` — 如果 `tools.web_search.provider: firecrawl`
- [ ] `FEISHU_APP_ID` + `FEISHU_APP_SECRET` — 飞书集成
- [ ] `S2_API_KEY` — 学术搜索（如已配置）
- [ ] `API_SERVER_ENABLED` / `API_SERVER_HOST` / `API_SERVER_PORT` / `API_SERVER_KEY` — Dashboard 如已开启
- [ ] 无残留旧机器的无效变量

## 4. Python 环境 — web_search 可用性

web_search 失败分两层诊断：

**第一层——Provider 配置**：
- [ ] `hermes config migrate | grep web_search` 确认 provider 被 Hermes 支持
- [ ] `curl -s https://api.firecrawl.dev/ --max-time 10` 端点可达（国内直连）
- [ ] `curl` + API key 独立验证 key 有效（非 401/402/403）

**第二层——Python 包安装**（Ubuntu 特有）：
- [ ] `python3 -c "import firecrawl" 2>&1` 不报 ModuleNotFoundError
- [ ] 如报错且 Python 是 externally-managed：`pip install firecrawl-py --break-system-packages`
- [ ] 或：`pipx install firecrawl-py` + 确保 Gateway 的 PATH 能找到

## 5. OpenClaw 共存的深层检查

- [ ] `openclaw.json` 大小 > 10KB（能正常解析）
- [ ] `openclaw.json` 中 `agents.defaults.workspace` 指向正确的 Linux 路径
- [ ] 飞书插件已安装：`ls ~/.openclaw/npm/node_modules/@openclaw/feishu/` 存在
- [ ] 飞书插件版本 ≤ OpenClaw 版本（版本不匹配导致 `Cannot find module`）
- [ ] 无 `C:\Users\*` 路径残留目录
- [ ] 无 `BOOTSTRAP.md` 幽灵文件（会导致 OpenClaw 启动失忆）

## 6. fact_store 重建指引

迁移后 fact_store 通常为空。文本记忆（MEMORY.md + USER.md）保留，但结构化事实需重建。以下事实应优先录入：

- [ ] 用户身份（姓名、职业、教育背景、研究方向）
- [ ] 家庭成员（配偶姓名、预产期、宝宝昵称）
- [ ] 项目约定（项目目录结构、文件命名规范）
- [ ] 工具偏好（模型选择策略、成本优化规则）
- [ ] 日程关键节点（毕业晚会、塔里木演出等）

方法：从 MEMORY.md 和 USER.md 提取事实 → `fact_store(action='add')` 逐条录入。
