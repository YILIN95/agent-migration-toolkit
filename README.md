# Agent Migration Toolkit

> A battle-tested, step-by-step guide for migrating AI agents (Hermes + OpenClaw) across servers and platforms. **Windows → Ubuntu, with all the real-world pitfalls documented.**

[![License: MIT](https://img.shields.io/badge/License-MIT-red.svg)](LICENSE)

---

## What This Is

A complete migration playbook born from a real production migration of two AI agents (Hermes and OpenClaw) from a Windows server to a Tencent Cloud Ubuntu instance. Every step, every error, and every fix in this guide came from actual production debugging — not hypothetical scenarios.

**Covers:**
- 📦 Data classification & tar packaging strategies
- 🔐 SSH key setup & verified file transfer
- 🐧 Ubuntu 24.04 environment setup
- 🛠️ Hermes & OpenClaw installation from source
- 🔧 Post-migration config fixes (12+ documented edge cases)
- 🚀 Systemd auto-start with crash recovery
- 📱 Feishu (Lark) integration for both agents
- ✅ 10-step post-migration audit checklist
- 🔍 Dual-agent capability audit framework

---

## Why This Matters

Migrating AI agents is not just "copy files and run." Production agents have:
- Persistent memory databases
- API credentials across multiple providers
- Platform-specific cron jobs
- Cross-agent health checks
- Message platform integrations (Feishu/WeChat)

One wrong step = silent failures, data loss, or security leaks. This toolkit documents the **complete truth** of what actually happens during migration — including the `C:\Users\...` ghost directories, the API key redaction traps, and the systemd sudo failures.

---

## Quick Start

### Prerequisites
- Source server with Hermes + OpenClaw running
- Target Ubuntu 24.04 server
- Python 3.12+, Node.js 22+
- Feishu Open Platform admin access

### 5-Phase Migration

| Phase | What | Time |
|-------|------|:---:|
| 1. Package | Classify data, tar core configs | 30 min |
| 2. Transfer | SSH key setup, verified SCP | 15 min |
| 3. Install | Hermes + OpenClaw on Ubuntu | 30 min |
| 4. Fix | Path corrections, API keys, permissions | 1-2 hrs |
| 5. Verify | 10-step audit, systemd, Feishu test | 30 min |

**Full guide:** [`docs/migration-guide.md`](docs/migration-guide.md)

---

## Documented Pitfalls (18 real issues)

| # | Issue | Category |
|:--:|-------|----------|
| 1 | GitHub unreachable on Tencent Cloud | Network |
| 2 | Root vs ubuntu user confusion | Permission |
| 3 | Windows paths contaminating Linux tar | Packaging |
| 4 | API keys redacted by LLM safety filter | Security |
| 5 | OpenClaw v5.27 Feishu dispatch crash | Version |
| 6 | BOOTSTRAP.md residue causing amnesia | Config |
| 7 | Feishu permission type mismatch | Integration |
| 8 | Event subscription not configured | Integration |
| 9 | dmPolicy whitelist blocking messages | Integration |
| 10 | npm global install EACCES | Permission |
| 11 | terminal.cwd Windows path residue | Config |
| 12 | web_search unavailable after migration | Dependencies |
| 13 | fact_store cleared during transfer | Memory |
| 14 | OpenClaw workspace ghost directories | Packaging |
| 15 | Plaintext API keys in config files | Security |
| 16 | Domestic server network restrictions | Network |
| 17 | Search backend unreachable (Firecrawl/Exa) | Dependencies |
| 18 | gateway install sudo failure | Systemd |

---

## Repository Structure

```
agent-migration-toolkit/
├── README.md                          # This file
├── LICENSE                            # MIT
└── docs/
    ├── migration-guide.md             # Full 6-phase migration playbook
    ├── post-migration-audit-extended.md  # Extended audit checklist
    └── dual-agent-capability-audit.md    # Post-migration capability matrix
```

---

## Who Should Use This

- DevOps engineers migrating AI agents between servers
- Open-source maintainers running self-hosted AI assistants
- Anyone moving from Windows → Linux for AI workloads
- Teams managing multi-agent deployments (Hermes, OpenClaw, Claude Code, Codex)

---

## Real-World Impact

This guide was used to successfully migrate two production AI agents serving a university faculty member. The agents handle:
- Daily diary automation (voice-to-text → cloud docs)
- Calendar management & event reminders
- Academic literature search across 78 scientific databases
- Multi-platform messaging (Feishu, WeChat)
- 15+ scheduled cron jobs for deadline tracking

**Zero downtime after migration. All cron jobs verified. Memory intact.**

---

## Contributing

Found another migration pitfall? PRs welcome. This is a living document — every real production migration surfaces new edge cases.

---

## License

MIT © 2026 Tianchu
