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

**不设全局代理变量。逐工具诊断。独立修复。记住你的偏好。**

<br>

9通道连通性检测 · 意图感知诊断 · 逐步交互修复 · 持久化自动修复 · 完整镜像知识库

[English →](README.md) &nbsp;·&nbsp; [效果演示](#效果演示) &nbsp;·&nbsp; [安装](#安装) &nbsp;·&nbsp; [检测矩阵](#检测矩阵) &nbsp;·&nbsp; [修复知识库](#修复知识库)

</div>

---

## 这解决了什么

在中国大陆 GFW 网络环境下，开发工具面临碎片化的网络局面——有些通道通，有些不通。传统做法是一行 `export HTTPS_PROXY=http://127.0.0.1:7890` 搞定，但带来一个致命后果：

> **代理一关，全部瘫痪——包括 AI Agent 自己。网络断了 = 无法自救。死循环。**

这个 skill 反其道而行：**每个工具独立管理自己的连通性**，一个坏了不会拖死全家。

---

## 效果演示

```
用户: npm install 一直超时，帮我看看

Agent: [加载 network-diagnostics skill]
       → Phase 0: 意图识别 → npm → 锁定 ①②⑦ 通道
       
       ━━━ 网络诊断 ━━━
       ① 代理   127.0.0.1:7890   ✅ 通
       ② curl   GitHub HTTPS      ✅ 通
       ⑦ npm    registry          ⚠️ ENOTFOUND
       
       结果: 2通 / 1不通 / 1项可修复

       ⑦ npm 无法连接 registry (npmjs.org DNS 被墙)。
       
       执行: npm config set registry https://registry.npmmirror.com
       
       [是(Y)/否(N)/全部是(A)/以后都自动(F)]？
       
       → 用户选 F（记住偏好，以后自动）
       → memory: network-diagnostics.auto_fix = true
       
       ━━━ 复验 ━━━
       ⑦ npm    registry          ✅ 通
       
       全部修复成功 ✓
```

> 每一步对应 skill 的 Phase 机制：Phase 0 意图锁定 → Phase 1 针对性诊断 → Phase 2 交互修复 → Phase 3 复验确认。

---

## 核心机理

### 为什么不设全局环境变量

原始的 v1.0 版本包含 `export HTTPS_PROXY`。实战中发现：

```
代理退出
  → 127.0.0.1:7890 变死端口
  → HTTPS_PROXY 还在环境变量里
  → Python(curl/pip/npm/Agent自身API调用) → 全走死代理 → 全部超时
  → Agent 自己断了 ← 加载不了 skill ← 无法自救
```

**v2.0+ 方案**：各工具独立配置，互不影响。

| 工具 | 配置方式 | 影响范围 |
|------|---------|---------|
| git | `git config --global http.proxy` | 仅 git |
| npm | `npm config set registry` | 仅 npm |
| pip | `pip config set global.index-url` | 仅 pip |
| cargo | `~/.cargo/config.toml` | 仅 cargo |
| go | `go env -w GOPROXY` | 仅 go |

代理退出后，只有 git 会受影响。其他走镜像的工具照常工作。

---

## 检测矩阵

| # | 检测点 | 测试命令 | 不通时修复 |
|---|--------|---------|-----------|
| ① | 代理存活 | `curl --proxy` → GitHub | ⛔ 阻断 — 先开代理 |
| ② | curl → GitHub | `curl -sI` GitHub | 检查代理配置 |
| ③ | git → GitHub | `git ls-remote` | `git config http.proxy` |
| ④ | Python → GitHub | `urllib` codeload | 检查 hermes .env |
| ⑤ | pip → PyPI | `pip install --dry-run` | 清华镜像 |
| ⑥ | uv → PyPI | `uv pip install --dry-run` | `UV_INDEX_URL` |
| ⑦ | npm → registry | `npm ping` | npmmirror |
| ⑧ | cargo → crates.io | `cargo search` | USTC 镜像 |
| ⑨ | go → proxy | `go env GOPROXY` | goproxy.cn |

---

## 安装

### 方式一：一行命令（推荐）

```bash
npx skills add marvinsui/network-diagnostics-in-GFW
```

基于 [vercel-labs/skills](https://github.com/vercel-labs/skills) 通用安装器，支持 55+ runtime 自动识别。

### 方式二：手动安装（Hermes Agent）

```bash
git clone https://github.com/marvinsui/network-diagnostics-in-GFW.git
mkdir -p ~/AppData/Local/hermes/skills/devops/network-diagnostics
cp network-diagnostics-in-GFW/SKILL.md ~/AppData/Local/hermes/skills/devops/network-diagnostics/
cp -r network-diagnostics-in-GFW/references/ ~/AppData/Local/hermes/skills/devops/network-diagnostics/
```

### 使用

```
> 帮我测一下网络，npm 一直超时
> 全面诊断一下开发环境的外网连通性
> 代理关了，帮我看看哪些工具还能用
```

---

## 修复知识库

### GitHub 通道

| 方案 | 命令 | 影响范围 |
|------|------|---------|
| git 代理 | `git config --global http.proxy http://127.0.0.1:7890` | 仅 git |
| Python urllib | `urllib.request.urlopen(codeload_url)` | Python 脚本 |
| jsDelivr CDN | `curl -sL "https://cdn.jsdelivr.net/gh/..."` | 单文件 |

### 已验证镜像

| 服务 | 镜像 URL | 状态 |
|------|---------|------|
| npm registry | `https://registry.npmmirror.com` | ✅ |
| PyPI | `https://pypi.tuna.tsinghua.edu.cn/simple` | ✅ |
| Electron 二进制 | `https://npmmirror.com/mirrors/electron/` | ✅ |
| Playwright 浏览器 | `https://npmmirror.com/mirrors/playwright/` | ✅ |
| Cargo (Rust) | `https://mirrors.ustc.edu.cn/crates.io-index` | ✅ |
| Go modules | `https://goproxy.cn` | ✅ |

---

## 自动修复开关

通过 Hermes memory 持久化：

```
memory key: network-diagnostics.auto_fix
值: true / false（默认）
```

随时说「以后自动修复网络问题」或「关闭网络自动修复」翻转。

---

## 许可证

MIT
