# Network Diagnostics in GFW — 网络连通性诊断与修复

> 一个用于在 GFW 网络环境下诊断并修复开发工具链连通性的 Hermes Agent 技能。

---

## 目录

- [背景与问题](#背景与问题)
- [设计理念](#设计理念)
- [核心机理](#核心机理)
- [功能特性](#功能特性)
- [检测矩阵](#检测矩阵)
- [使用流程](#使用流程)
- [交互模式](#交互模式)
- [修复知识库](#修复知识库)
- [自动修复开关](#自动修复开关)
- [安装方法](#安装方法)
- [陷阱与注意事项](#陷阱与注意事项)

---

## 背景与问题

在中国大陆 GFW 网络环境下，开发工具访问国外服务器时常出现**部分通道不通、部分通道通**的复杂局面：

```
✅ PyPI (pypi.org)      直连 OK
✅ npm (npmmirror镜像)   通
✅ git (走代理)          通
❌ npm (npmjs.org)       DNS 被墙
❌ GitHub HTTPS          直连被墙
❌ Electron 二进制        ECONNRESET
```

传统的做法是 `export HTTPS_PROXY=http://127.0.0.1:7890` 一设了之，但这带来一个致命副作用：

> **代理一旦退出，所有工具（包括 AI Agent 自身的 API 调用）全部瘫痪。**

Agent 本身都无法联网，也就无法自救——形成死循环。

---

## 设计理念

### 原则一：不设全局环境变量

不使用 `export HTTPS_PROXY`。各工具独立配置自己的代理或镜像：

| 工具 | 配置方式 | 影响范围 |
|------|---------|---------|
| git | `git config --global http.proxy` | 仅 git |
| npm | `npm config set registry` | 仅 npm |
| pip | `pip config set global.index-url` | 仅 pip |
| cargo | `~/.cargo/config.toml` | 仅 cargo |
| go | `go env -w GOPROXY` | 仅 go |

代理退出后，只有 git 走代理会失败，其他工具（pip/npm/cargo等走镜像的）不受影响。

### 原则二：按需诊断

激活 skill 后首先问：「这次需要用到哪些工具？」根据意图锁定检测项，**不跑全矩阵**。

| 用户意图 | 锁定检测项 |
|---------|-----------|
| `git clone/pull` | ①代理 ②curl ③git |
| `npm install` | ①代理 ②curl ⑦npm |
| `hermes update` | ①代理 ②curl ③git ④Python |
| 全面体检 | ①-⑨ 全矩阵 |

### 原则三：先告知，再修复

每项修复都告知用户问题、展示命令、等待确认。用户可选择「以后都自动」，通过 Hermes memory 持久化偏好。

---

## 核心机理

### 通道检测原理

每条通道的检测是最小化的连通性测试，不消耗大量带宽：

```
① 代理存活：  curl --proxy 到 GitHub → HTTP 200/301/302
② curl 通道：  curl 直连 GitHub HTTPS → HTTP 状态码
③ git 通道：   git ls-remote → commit hash
④ Python 通道： urllib 请求 codeload → HTTP 200
⑤ pip 通道：   pip install --dry-run → Would install...
⑥ uv 通道：    uv pip install --dry-run → 正常输出
⑦ npm 通道：   npm ping → Pong
⑧ cargo 通道： cargo search --limit 1 → 正常输出
⑨ go 通道：    go env GOPROXY → 非 direct
```

### 修复优先级

```
① 代理不通 → ⛔ 阻断，提示开代理，不继续
② curl 不通 → 检查代理配置（可能走系统代理）
③ git 不通  → git config http.proxy
④ Python 不通 → 检查 hermes .env（罕见）
⑤⑥  pip/uv 不通 → PyPI 镜像
⑦ npm 不通 → npmmirror
⑧ cargo 不通 → USTC 镜像
⑨ go 不通 → goproxy.cn
```

### 为什么去掉环境变量检测

原始的 v1.0 版本包含「② 环境变量检测」并会 `export HTTPS_PROXY`。实战中发现：

1. **代理退出后死循环**：HTTPS_PROXY 指向死端口 → Python/curl/npm 全死 → Agent API 调用也死 → 无法自救
2. **副作用太大**：一个环境变量影响所有子进程，无法精细控制
3. **git 不读环境变量**：设了也没用，git 需要自己的 `http.proxy` config

v2.0+ 版本去掉了环境变量相关的检测和修复，改为各工具独立配置。

---

## 功能特性

- ✅ **9 项通道检测**：代理 → curl → git → Python → pip → uv → npm → cargo → go
- ✅ **意图感知**：根据用户任务自动锁定检测范围
- ✅ **三态诊断报告**：✅ 通 / ⚠️ 不通（可修复） / ❌ 阻断
- ✅ **交互式修复**：逐项告知问题 + 修复命令 + 等待确认
- ✅ **记忆持久化**：「以后都自动」偏好写入 Hermes memory
- ✅ **修复后复验**：修复项重新检测确认
- ✅ **完整知识库**：国内已验证的镜像方案、失效镜像列表、错误签名速查表

---

## 检测矩阵

| # | 检测点 | 命令 | 不通的修复 |
|---|--------|------|-----------|
| ① | 代理存活 | `curl --proxy` GitHub | ⛔ 请开启代理 |
| ② | curl → GitHub | `curl -sI` GitHub | 检查代理配置 |
| ③ | git → GitHub | `git ls-remote` | `git config http.proxy` |
| ④ | Python → GitHub | `urllib` codeload | 检查 hermes .env |
| ⑤ | pip → PyPI | `pip install --dry-run` | 清华镜像 |
| ⑥ | uv → PyPI | `uv pip install --dry-run` | `UV_INDEX_URL` |
| ⑦ | npm → registry | `npm ping` | npmmirror |
| ⑧ | cargo → crates.io | `cargo search` | USTC 镜像 |
| ⑨ | go → proxy | `go env GOPROXY` | goproxy.cn |

---

## 使用流程

### Phase 0: 确定检测范围

Agent 加载 skill 后，根据上下文或用户回答确定需要检测的通道：

```
Agent: 这次需要用到哪些工具？git/npm/pip/...？
User: npm install
→ 锁定 ①②⑦（代理/curl/npm）
```

### Phase 1: 执行诊断

```
━━━ 网络诊断 ━━━
① 代理   127.0.0.1:7890   ✅ 通 (1.2s)
② curl   GitHub HTTPS      ✅ 通 (0.8s)
⑦ npm    registry          ⚠️ ENOTFOUND

结果: 2通 / 1不通 / 1项可修复
```

### Phase 2: 逐项修复

```
⑦ npm 无法连接 registry (npmjs.org DNS 被墙)。

执行以下命令：
  npm config set registry https://registry.npmmirror.com

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

### Phase 3: 复验

```
━━━ 复验 ━━━
⑦ npm    registry          ✅ 通 (1.5s)

全部修复成功 ✓
```

---

## 交互模式

每次修复提供四个选项：

| 选项 | 行为 | 持久化 |
|------|------|--------|
| **Y** (是) | 执行本次修复 | 不持久化 |
| **N** (否) | 跳过此项 | — |
| **A** (全部是) | 执行本次+后续所有项 | 不改变默认行为 |
| **F** (以后都自动) | 执行+写入 memory | `auto_fix: true` |

---

## 修复知识库

### GitHub 通道

#### 方案 A：git 代理（推荐）

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

> 仅影响 git，不影响其他工具。

#### 方案 B：Python urllib 下载

当 git clone/curl 都失败时，Python urllib 有时能穿透：

```python
import urllib.request
url = "https://codeload.github.com/NousResearch/hermes-agent/zip/refs/heads/main"
req = urllib.request.Request(url, headers={"User-Agent": "curl/7.0"})
with urllib.request.urlopen(req, timeout=60) as resp:
    data = resp.read()
```

#### 方案 C：jsDelivr CDN（单文件）

```bash
curl -sL "https://cdn.jsdelivr.net/gh/user/repo@branch/path"
```

#### 已失效镜像（勿用）

- `ghproxy.com` / `hub.fastgit.xyz` / `gitclone.com` / `kgithub.com` 等均不可用

### npm 镜像

| 镜像 | URL |
|------|-----|
| 阿里云 npmmirror | `https://registry.npmmirror.com` |
| 华为云 | `https://mirrors.huaweicloud.com/repository/npm/` |
| 腾讯云 | `https://mirrors.cloud.tencent.com/npm/` |

### Electron / Playwright 二进制镜像

```bash
# 安装期间临时设置
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export PLAYWRIGHT_DOWNLOAD_HOST="https://npmmirror.com/mirrors/playwright/"

# 安装完立即清理
unset ELECTRON_MIRROR PLAYWRIGHT_DOWNLOAD_HOST
```

### PyPI 镜像

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### Cargo 镜像

`~/.cargo/config.toml`:
```toml
[source.crates-io]
replace-with = 'ustc'
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
```

### Go 镜像

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

---

## 自动修复开关

通过 Hermes memory 持久化：

```
memory key: network-diagnostics.auto_fix
值: true / false (默认)
```

用户随时可以说「以后自动修复网络问题」或「关闭网络自动修复」。

> ⚠️ 即使 `auto_fix: true`，git proxy 设置仍需确认。镜像类操作可自动执行。

---

## 安装方法

### 通过 Hermes Skills Hub（即将支持）

```bash
hermes skills install network-diagnostics
```

### 手动安装

```bash
# 克隆仓库
git clone git@github.com:marvinsui/network-diagnostics-in-GFW.git

# 复制到 Hermes skills 目录
mkdir -p ~/AppData/Local/hermes/skills/devops/network-diagnostics
cp network-diagnostics-in-GFW/SKILL.md ~/AppData/Local/hermes/skills/devops/network-diagnostics/
cp -r network-diagnostics-in-GFW/references/ ~/AppData/Local/hermes/skills/devops/network-diagnostics/
```

### 验证

```bash
hermes skills list | grep network-diagnostics
```

---

## 陷阱与注意事项

1. **代理不通是阻断性错误** — 代理挂了所有走代理的通道全死，必须先修
2. **Windows curl 可能走系统代理** — curl 通了不代表所有工具都通
3. **npm ping 显示当前 registry** — 已切 npmmirror 的用户看到的是镜像状态
4. **`pip` 可能未安装** — 部分环境只用 `uv`，自动跳过 pip 检测
5. **curl 可能在 git-bash 中缺失** — 自动降级为 Python urllib 检测
6. **安装脚本修补** — 当脚本内置了 `git fetch` 且无法跳过时，需手动下载源码 + 初始化本地 git 仓库 + 修补脚本

---

## 文件结构

```
network-diagnostics-in-GFW/
├── SKILL.md                       # Hermes 技能定义文件
├── README.md                      # 英文说明（本文件）
├── README_CN.md                   # 中文详细说明
├── LICENSE                        # MIT 许可证
└── references/
    └── error-signatures.md        # GFW 错误签名速查表
```

## 许可证

MIT
