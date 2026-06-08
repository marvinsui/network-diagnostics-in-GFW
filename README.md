<div align="center">

# Network Diagnostics in GFW

<p align="center">
  <em>网络连通性诊断与修复</em>
</p>

> *「代理一死，全员瘫痪。你得让每个工具自己学会走路，别把命拴在一根绳上。」*

[![Version](https://img.shields.io/badge/version-2.1.0-blue)](SKILL.md)
[![Hermes](https://img.shields.io/badge/Hermes-skill-6C2BD9)](https://hermes-agent.nousresearch.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Multi-Runtime](https://img.shields.io/badge/Runtime-Hermes%20Agent-blueviolet)](https://hermes-agent.nousresearch.com)

<br>

**Don't set global proxy env vars. Diagnose each tool. Fix individually. Remember your preferences.**

<br>

9-channel connectivity detection · Intent-aware diagnosis · Step-by-step repair · Persistent auto-fix memory · Full mirror knowledge base

[中文详细文档 →](README_CN.md)

</div>

---

## What It Solves

In mainland China, developer tools face a fragmented network: some channels work, some don't. The traditional fix — `export HTTPS_PROXY=http://127.0.0.1:7890` — creates a fatal side effect:

> **When the proxy goes down, everything dies — including the AI agent itself. No network = no self-rescue. Death loop.**

This skill takes the opposite approach: **each tool manages its own connectivity**, so one failure can't cascade.

---

## Quick Demo

```
User: npm install keeps timing out!

Agent: [loads network-diagnostics skill]
       Phase 0: Intent → npm → lock proxy/curl/npm channels
       
       ═══ Network Diagnostics ═══
       ① Proxy   127.0.0.1:7890   ✅ OK
       ② curl    GitHub HTTPS      ✅ OK
       ⑦ npm     registry          ⚠️ ENOTFOUND
       
       Result: 2/3 OK, 1 fixable

       ⑦ npm can't reach npmjs.org (DNS blocked).
       
       Fix: npm config set registry https://registry.npmmirror.com
       
       [Y]es / [N]o / [A]ll / [F]orever auto?
       
       → User chose F (remember forever)
       → memory: network-diagnostics.auto_fix = true
       
       ═══ Re-verify ═══
       ⑦ npm     registry          ✅ OK
       
       All fixed ✓
```

> Each step annotated with the underlying mechanism: Phase 0 intent-lock, Phase 1 targeted diagnosis, Phase 2 interactive repair, Phase 3 re-verification.

---

## Detection Matrix

| # | Channel | Test | Failure Fix |
|---|---------|------|-------------|
| ① | Proxy alive | `curl --proxy` to GitHub | ⛔ Blocker — start proxy first |
| ② | curl → GitHub | `curl -sI` GitHub HTTPS | Check proxy config |
| ③ | git → GitHub | `git ls-remote` | `git config http.proxy` |
| ④ | Python → GitHub | `urllib` to codeload | Check hermes `.env` |
| ⑤ | pip → PyPI | `pip install --dry-run` | Tsinghua mirror |
| ⑥ | uv → PyPI | `uv pip install --dry-run` | `UV_INDEX_URL` |
| ⑦ | npm → registry | `npm ping` | npmmirror |
| ⑧ | cargo → crates.io | `cargo search` | USTC mirror |
| ⑨ | go → proxy | `go env GOPROXY` | goproxy.cn |

---

## Design Principles

### 1. No Global Proxy Env Vars

Never `export HTTPS_PROXY`. Each tool independently:

| Tool | Config Method | Blast Radius |
|------|--------------|-------------|
| git | `git config --global http.proxy` | git only |
| npm | `npm config set registry` | npm only |
| pip | `pip config set global.index-url` | pip only |
| cargo | `~/.cargo/config.toml` | cargo only |
| go | `go env -w GOPROXY` | go only |

Proxy down? Only git breaks. Everything else still works.

### 2. Intent-Aware Diagnosis

Activate → ask "what do you need?" → lock only relevant channels. Never run the full matrix unless asked.

### 3. Ask Before Fix

Every fix: explain the problem → show the command → wait for confirmation. User can choose "remember forever" to persist via Hermes memory.

---

## Installation

### Option 1: One-liner (recommended, cross-runtime)

```bash
npx skills add marvinsui/network-diagnostics-in-GFW
```

Works with [vercel-labs/skills](https://github.com/vercel-labs/skills) (55+ runtime support). Auto-detects your runtime.

### Option 2: Manual (Hermes Agent)

```bash
git clone https://github.com/marvinsui/network-diagnostics-in-GFW.git
mkdir -p ~/AppData/Local/hermes/skills/devops/network-diagnostics
cp network-diagnostics-in-GFW/SKILL.md ~/AppData/Local/hermes/skills/devops/network-diagnostics/
cp -r network-diagnostics-in-GFW/references/ ~/AppData/Local/hermes/skills/devops/network-diagnostics/
```

### Verify

```bash
hermes skills list | grep network-diagnostics
```

### Usage

```
> 帮我测一下网络，npm 一直超时
> 全面诊断一下开发环境的外网连通性
> 代理关了，帮我看看哪些工具还能用
```

---

## Repair Knowledge Base

### GitHub Channels

| Method | Command | Scope |
|--------|---------|-------|
| git proxy | `git config --global http.proxy http://127.0.0.1:7890` | git only |
| Python urllib | `urllib.request.urlopen(codeload_url)` | Python scripts |
| jsDelivr CDN | `curl -sL "https://cdn.jsdelivr.net/gh/..."` | single files |

### Verified Mirrors

| Service | Mirror URL | Status |
|---------|-----------|--------|
| npm registry | `https://registry.npmmirror.com` | ✅ |
| PyPI | `https://pypi.tuna.tsinghua.edu.cn/simple` | ✅ |
| Electron binaries | `https://npmmirror.com/mirrors/electron/` | ✅ |
| Playwright browsers | `https://npmmirror.com/mirrors/playwright/` | ✅ |
| Cargo (Rust) | `https://mirrors.ustc.edu.cn/crates.io-index` | ✅ |
| Go modules | `https://goproxy.cn` | ✅ |

---

## Repository Structure

```
network-diagnostics-in-GFW/
├── SKILL.md                       # Skill definition (loaded by Hermes)
├── README.md                      # This file (English)
├── README_CN.md                   # Detailed Chinese documentation
├── LICENSE                        # MIT
└── references/
    └── error-signatures.md        # GFW error pattern cheat sheet
```

---

## License

MIT
