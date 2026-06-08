# Network Diagnostics in GFW — 网络连通性诊断与修复

[![Version](https://img.shields.io/badge/version-2.1.0-blue)](SKILL.md)
[![Hermes](https://img.shields.io/badge/Hermes-skill-6C2BD9)](https://hermes-agent.nousresearch.com)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

> A Hermes Agent skill for diagnosing and fixing developer toolchain connectivity behind the Great Firewall.
> 一个用于在 GFW 网络环境下诊断并修复开发工具链连通性的 Hermes Agent 技能。

[中文详细文档 →](README_CN.md)

---

## What It Does

In mainland China, developer tools (git, pip, npm, curl, Python, cargo, go...) often lose connectivity to foreign servers. Some channels work, some don't. This skill:

1. **Intent-aware diagnosis** — detects which tools you need, tests only those channels
2. **Step-by-step repair** — explains the problem, offers a fix, asks for confirmation
3. **Remembers your preferences** — "auto-fix from now on" via persistent memory

## Design Principle

**No global proxy environment variables.** Setting `HTTPS_PROXY` globally means every tool (including the agent's own API calls) routes through the proxy. When the proxy goes down, everything dies — including the agent itself. Instead, each tool configures its own proxy or mirror independently.

## Quick Example

```
User: npm install keeps timing out!

Agent: [loads network-diagnostics skill]
       Phase 0: Intent → npm → lock proxy/curl/npm channels
       Phase 1: Diagnostics
       
       ═══ Network Diagnostics ═══
       ① Proxy   127.0.0.1:7890   ✅ OK
       ② curl    GitHub HTTPS      ✅ OK
       ⑦ npm     registry          ⚠️ ENOTFOUND
       
       Phase 2:
       ⑦ npm DNS blocked → auto-fix → npm config set registry npmmirror.com
       
       Phase 3: Re-verify ✓
```

## Detection Matrix

| # | Channel | Test | Fix |
|---|---------|------|-----|
| ① | Proxy alive | `curl --proxy` to GitHub | ⛔ Blocker — open proxy first |
| ② | curl → GitHub | `curl -sI` GitHub HTTPS | Check proxy config |
| ③ | git → GitHub | `git ls-remote` | `git config http.proxy` |
| ④ | Python → GitHub | `urllib` to codeload | Check hermes `.env` |
| ⑤ | pip → PyPI | `pip install --dry-run` | Tsinghua mirror |
| ⑥ | uv → PyPI | `uv pip install --dry-run` | `UV_INDEX_URL` |
| ⑦ | npm → registry | `npm ping` | npmmirror |
| ⑧ | cargo → crates.io | `cargo search` | USTC mirror |
| ⑨ | go → proxy | `go env GOPROXY` | goproxy.cn |

## Repair Flow

```
Phase 1: Diagnose → report (✅/⚠️/❌ per channel)
Phase 2: For each broken channel:
  → Explain the problem
  → Show the fix command
  → Ask: [Y]es / [A]ll / [F]orever auto
    Y = this one only
    A = this + all remaining
    F = save "auto_fix: true" to memory
Phase 3: Re-verify fixed channels
```

## Installation

### Via Hermes Skills Hub (coming soon)

```bash
hermes skills install network-diagnostics
```

### Manual

```bash
mkdir -p ~/AppData/Local/hermes/skills/devops/network-diagnostics
cp SKILL.md ~/AppData/Local/hermes/skills/devops/network-diagnostics/
cp -r references/ ~/AppData/Local/hermes/skills/devops/network-diagnostics/
```

## Files

```
network-diagnostics-in-GFW/
├── SKILL.md              # Skill definition (loaded by Hermes)
├── README.md             # This file (English)
├── README_CN.md          # Detailed Chinese documentation
├── LICENSE               # MIT
└── references/
    └── error-signatures.md   # GFW error pattern cheat sheet
```

## Verified Mirrors

| Service | Mirror URL | Status |
|---------|-----------|--------|
| npm registry | `https://registry.npmmirror.com` | ✅ |
| PyPI | `https://pypi.tuna.tsinghua.edu.cn/simple` | ✅ |
| Electron binaries | `https://npmmirror.com/mirrors/electron/` | ✅ |
| Playwright browsers | `https://npmmirror.com/mirrors/playwright/` | ✅ |
| Cargo (Rust) | `https://mirrors.ustc.edu.cn/crates.io-index` | ✅ |
| Go modules | `https://goproxy.cn` | ✅ |
| jsDelivr CDN | `https://cdn.jsdelivr.net/gh/` | ✅ (single files only) |

## License

MIT
