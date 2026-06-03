# OpenAI Codex for Open Source — 申请文案

> 用于填写 https://openai.com/form/codex-for-oss/ 申请表

---

## GitHub 仓库

https://github.com/YILIN95/agent-migration-toolkit

---

## 问题 1：Why does this repository qualify?（≤500 字符）

```
I am the creator and sole maintainer of agent-migration-toolkit, a
battle-tested migration playbook for AI agents (Hermes + OpenClaw)
across platforms. The project documents 18 real-world production
pitfalls with verified fixes, covering data packaging, SSH transfer,
systemd deployment, Feishu integration, and post-migration auditing.
It serves the growing community of self-hosted AI agent operators
who need reliable cross-platform migration guidance.
```

**中文版（参考）：**
```
我是 agent-migration-toolkit 的创建者和唯一维护者。该项目是一份
经过生产环境验证的 AI Agent（Hermes + OpenClaw）跨平台迁移完整指南，
记录了 18 个真实生产环境踩坑案例及修复方案，覆盖数据打包、SSH 传输、
systemd 部署、飞书集成和迁移后审计。服务于日益增长的自托管 AI Agent
运维社区，提供可靠的跨平台迁移参考。
```

---

## 问题 2：How will you use API credits for your project?（≤500 字符）

```
API credits will be used to automate the maintenance of this
project: reviewing community-contributed migration cases via PR
review, generating structured changelogs from commit history,
classifying and triaging GitHub issues, suggesting documentation
improvements, and validating new migration procedures against
the existing playbook for consistency. All AI-generated outputs
will be reviewed by me before merging.
```

**中文版（参考）：**
```
API 额度将用于自动化该项目的维护工作：通过 PR 审查审核社区贡献的
迁移案例、从提交历史生成结构化 changelog、分类和分流 GitHub Issues、
建议文档改进、以及验证新的迁移流程与现有指南的一致性。所有 AI 生成
的内容将由我在合并前审核。
```

---

## 问题 3：Additional context（≤500 字符，选填）

```
This project was born from a real production migration of two AI
agents from Windows to Ubuntu on Tencent Cloud, serving a university
faculty member's daily workflow. The agents handle diary automation,
calendar management, academic literature search across 78 databases,
and multi-platform messaging. The migration involved solving 18 unique
production issues over 2 days — every fix in the guide comes from
actual debugging, not theory.
```

**中文版（参考）：**
```
该项目源于一次真实的生产环境迁移——将两台 AI Agent 从 Windows 迁移到
腾讯云 Ubuntu，服务于一位大学教师的日常工作流。这些 Agent 负责日记
自动化、日历管理、78 个科学数据库的文献检索和多平台消息推送。迁移过程
中在两天内解决了 18 个独特的生产环境问题——指南中的每个修复方案都来自
实际调试，而非理论推演。
```

---

## 申请步骤 Checklist

- [ ] 在 GitHub 创建仓库 `agent-migration-toolkit`
- [ ] Push 代码到 GitHub
- [ ] 确认仓库是 Public
- [ ] 打开 https://openai.com/form/codex-for-oss/
- [ ] 填写姓名、邮箱（与 ChatGPT 账号一致）
- [ ] 填写 GitHub 用户名
- [ ] 粘贴仓库 URL
- [ ] 选择 Maintainer role: Primary/Core maintainer
- [ ] 粘贴问题 1 文案（Why qualify）
- [ ] 选择 Interests: ChatGPT Pro + API credits
- [ ] 粘贴问题 2 文案（How use credits）
- [ ] 粘贴问题 3 文案（Additional context，选填）
- [ ] 提交，等邮件通知
