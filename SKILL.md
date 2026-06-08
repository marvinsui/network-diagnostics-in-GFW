---
name: network-diagnostics
description: "网络连通性诊断与修复 — 检测 git/pip/npm/curl/Python/uv/cargo/go 等开发工具的外网连通性，按需逐步修复，含国内镜像方案。"
version: 2.2.0
author: agent
license: MIT
metadata:
  hermes:
    tags: [network, diagnostics, gfw, proxy, git, npm, pip, china, repair, mirror]
---

# 网络连通性诊断与修复

在中国大陆 GFW 网络环境下，开发工具经常出现部分通道不通、部分通道通的情况。本 skill 提供：
1. **按需诊断**：先明确检测目标，不浪费时间测无关通道
2. **逐步修复**：逐项告知用户问题 + 修复方案，确认后执行
3. **可记忆偏好**：用户可选择"以后都自动"，通过 memory 持久化
4. **修复知识库**：国内已验证的镜像/绕过方案

> **设计原则**：不设置全局环境变量（如 `HTTPS_PROXY`），避免代理退出后所有工具瘫痪。各工具独立配置自己的代理或镜像。

---

## Phase 0: 确定检测范围（激活 skill 后第一步）

根据用户意图锁定检测项，**不跑全矩阵**：

| 用户意图 | 锁定检测项 |
|---------|-----------|
| `git clone/pull/fetch/push` | ①代理 ②curl ③git |
| `hermes update` | ①代理 ②curl ③git ④Python |
| `pip install / uv pip install` | ①代理 ②curl ⑤pip ⑥uv |
| `npm install / npx` | ①代理 ②curl ⑦npm |
| `cargo build/install` | ①代理 ②curl ⑧cargo |
| `go get/install` | ①代理 ②curl ⑨go |
| `curl / wget 下载外网文件` | ①代理 ②curl |
| 不确定 / 全面体检 | 全矩阵 ①-⑨ |
| 修复后复验 | 仅修复过的那几项 |

**执行方式**：先问用户"这次需要用到哪些工具？"或根据上下文推断。如用户说不确定，走全矩阵。

---

## 检测矩阵（9 项）

每项检测耗时 3-5 秒，超时统一设 10s。

### ① 代理存活检测

```bash
curl -sI --max-time 5 --proxy http://127.0.0.1:7890 https://github.com -o /dev/null -w "%{http_code}"
```

- ✅ 返回 200/301/302 → 通
- ❌ 超时/000/拒绝连接 → 不通，**所有后续检测依赖此项**。提示用户开启代理。

### ② curl → GitHub HTTPS

```bash
curl -sI --max-time 5 https://github.com -o /dev/null -w "%{http_code}"
```

- ✅ 200/301 → 通（可能走了系统代理或各工具自身的代理配置）
- ❌ 超时/000 → 不通

> Windows 注意：curl 在 Windows 上可能通过系统代理（Internet Options）自动走代理。

### ③ git → GitHub HTTPS

```bash
git ls-remote https://github.com/NousResearch/hermes-agent.git HEAD 2>&1
```

- ✅ 返回 commit hash → 通
- ❌ `Connection refused`/`reset`/`Could not resolve` → 不通

### ④ Python urllib → GitHub codeload

```bash
python -c "
import urllib.request, sys
req = urllib.request.Request('https://codeload.github.com/NousResearch/hermes-agent/zip/refs/heads/main', headers={'User-Agent': 'curl/7.0'})
try:
    resp = urllib.request.urlopen(req, timeout=8)
    print(f'OK HTTP {resp.status}')
except Exception as e:
    print(f'FAIL: {e}')
    sys.exit(1)
"
```

> Windows 注意：`python3` 可能不存在，优先用 `python`。

### ⑤ pip → PyPI

```bash
pip install --dry-run six 2>&1 | tail -5
```

- ✅ 包含 `Would install` → 通（PyPI 在国内通常直连OK）
- ❌ 超时 → 不通

### ⑥ uv → PyPI

```bash
uv pip install --dry-run six 2>&1 | tail -5
```

- ✅ 正常输出 → 通
- ❌ 超时 → 不通

### ⑦ npm → registry

```bash
npm ping 2>&1
```

- ✅ `Ping success` → 通
- ❌ `ENOTFOUND`/超时 → 不通（npmjs.org DNS 被墙）

> 注意：如果用户已切换到 npmmirror，`npm ping` 会 ping 镜像而非 npmjs.org。若想检测 npmjs.org 连通性，用 `npm ping --registry https://registry.npmjs.org/`。

### ⑧ cargo → crates.io

```bash
cargo search hello --limit 1 2>&1
```

- ✅ 正常输出 → 通
- ❌ 超时/网络错误 → 不通

### ⑨ go → proxy

```bash
go env GOPROXY 2>&1
```

- ⚠️ 返回 `direct` → 直连模式，可能被墙
- ✅ 返回 `https://goproxy.cn,...` → 已设国内镜像

---

## Phase 1: 执行诊断

按锁定的检测项逐项运行，输出三态报告。遇到错误的签名可对照 `references/error-signatures.md` 快速定位。

```
━━━ 网络诊断 ━━━
① 代理   127.0.0.1:7890   ✅ 通 (1.2s)
② curl   GitHub HTTPS      ⚠️ 不通 (超时)
③ git    GitHub HTTPS      ⚠️ 不通 (Connection refused)
⑤ pip    PyPI              ✅ 通 (2.1s)

结果: 2通 / 2不通 / 2项可修复
```

---

## Phase 2: 逐项修复

对每项"不通且可修复"的，按优先级逐项处理。修复方案取自下方的[修复知识库](#修复知识库)。

### ① 代理不通

阻断性错误。提示用户：
```
① 代理 127.0.0.1:7890 不通。
请确认代理软件已开启，端口未变化。
修复后重新运行诊断。
```
不继续后续步骤。

### ③ git 不通

```
③ git 无法连接 GitHub HTTPS。

执行以下命令：
  git config --global http.proxy http://127.0.0.1:7890
  git config --global https.proxy http://127.0.0.1:7890

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

### ④ Python urllib 不通

少见。hermes 自身可能通过其他机制走代理。如不通，检查 hermes 的 `.env` 或 Windows 系统代理设置。

### ⑤ pip 不通

```
⑤ pip 无法连接 PyPI。

执行以下命令：
  pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

### ⑥ uv 不通

```
⑥ uv 无法连接 PyPI。

执行以下命令：
  export UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

> 注意：`UV_INDEX_URL` 仅影响 uv，不影响其他工具，副作用可控。

### ⑦ npm 不通

```
⑦ npm 无法连接 registry (npmjs.org DNS 被墙)。

执行以下命令：
  npm config set registry https://registry.npmmirror.com

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

### ⑧ cargo 不通

```
⑧ cargo 无法连接 crates.io。

在 ~/.cargo/config.toml 中写入镜像配置：
  [source.crates-io]
  replace-with = 'ustc'
  [source.ustc]
  registry = "https://mirrors.ustc.edu.cn/crates.io-index"

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

### ⑨ go 不通

```
⑨ go 未配置国内镜像（GOPROXY=direct）。

执行以下命令：
  go env -w GOPROXY=https://goproxy.cn,direct

是否执行？[是(Y)/否(N)/全部是(A)/以后都自动(F)]
```

**交互选项说明**：
- **Y**：执行本次
- **A**：本次+后续所有项全部执行（但不改默认行为）
- **F**：执行+存入 memory `network-diagnostics.auto_fix: true`，下次不再询问

---

## Phase 3: 复验

对修复过的项重新检测，确认已通：

```
━━━ 复验 ━━━
③ git    GitHub HTTPS      ✅ 通 (2.1s)
⑦ npm    registry          ✅ 通 (1.5s)

全部修复成功 ✓
```

---

## 自动修复开关

通过 memory 持久化用户偏好：

| memory key | 值 | 含义 |
|-----------|-----|------|
| `network-diagnostics.auto_fix` | `true` | 检测到问题后自动执行修复，不再逐项询问 |
| `network-diagnostics.auto_fix` | `false` (默认) | 逐项询问用户确认后才执行 |

用户可随时说"以后自动修复网络问题"或"关闭网络自动修复"来翻转。

> ⚠️ 即使 `auto_fix: true`，**git proxy 设置**仍需确认（改变全局 git 行为）。镜像类操作（⑤pip、⑥uv、⑦npm、⑧cargo、⑨go）可以自动执行。

---

## 修复知识库

以下为各通道在国内网络环境下的已验证修复方案。

### GitHub 直连方案

#### 方案 A：git 代理配置（推荐，不影响 Python/curl）

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

> 仅影响 git，不会在代理退出后拖死其他工具。

#### 方案 B：Python urllib 下载（代理不可用时）

当 git clone 和 curl 都失败时，Python 的 `urllib` 有时能穿透下载：

```python
import urllib.request
url = "https://codeload.github.com/NousResearch/hermes-agent/zip/refs/heads/main"
req = urllib.request.Request(url, headers={"User-Agent": "curl/7.0"})
with urllib.request.urlopen(req, timeout=60) as resp:
    data = resp.read()  # 55MB+ repos take ~60s
```

#### 方案 C：jsDelivr CDN（单个文件）

```bash
curl -sL "https://cdn.jsdelivr.net/gh/NousResearch/hermes-agent@main/scripts/install.sh"
```

jsDelivr 不支持整个仓库的 zip/tarball 下载，只提供单个文件。

#### 已失效的 GitHub 镜像（供参考，勿用）

以下镜像在本环境测试中不可用（DNS 无法解析或连接超时）：
- `raw.gitmirror.com` — DNS 不可达
- `ghproxy.com` — 连接超时
- `hub.fastgit.xyz` — 连接超时
- `gitclone.com` — 502
- `kgithub.com` — 连接超时
- `mirror.ghproxy.com` — 连接超时
- `download.fastgit.org` — DNS 不可达

#### 伪造 Git 仓库以绕过联网检查

当安装脚本强制要求 `git clone` 或 `git fetch` 时：

```bash
cd /path/to/install/dir
git init
git add -A
git -c user.name="local" -c user.email="local@local" commit -m "initial"
git branch -M main
git config core.autocrlf false
git remote add origin https://github.com/original/repo.git
```

### npm registry 镜像

| 镜像 | URL | 设置命令 |
|------|-----|---------|
| 阿里云 npmmirror | `https://registry.npmmirror.com` | `npm config set registry https://registry.npmmirror.com` |
| 华为云 | `https://mirrors.huaweicloud.com/repository/npm/` | 同上替换 URL |
| 腾讯云 | `https://mirrors.cloud.tencent.com/npm/` | 同上替换 URL |

### Electron / Playwright 二进制镜像

⚠️ `ELECTRON_MIRROR` 和 `PLAYWRIGHT_DOWNLOAD_HOST` 是环境变量，**仅在 npm install 的 postinstall 阶段需要**。安装完成后可 unset。不能用 `npm config set`。

```bash
# 安装期间临时设置
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export ELECTRON_BUILDER_BINARIES_MIRROR="https://npmmirror.com/mirrors/electron-builder-binaries/"
export PLAYWRIGHT_DOWNLOAD_HOST="https://npmmirror.com/mirrors/playwright/"

# npm install 完成后清理
unset ELECTRON_MIRROR ELECTRON_BUILDER_BINARIES_MIRROR PLAYWRIGHT_DOWNLOAD_HOST
```

### PyPI 镜像

PyPI 在国内通常可直接访问。如需镜像加速：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 或临时使用
uv pip install --index-url https://pypi.tuna.tsinghua.edu.cn/simple <package>
```

### Cargo 镜像

在 `~/.cargo/config.toml`：

```toml
[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
```

备选：`https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git`

### Go 镜像

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 已验证通道状态

| 资源 | 状态 | 方法 |
|------|------|------|
| PyPI (pypi.org) | ✅ 可访问 | pip/uv 直连 |
| GitHub codeload | ✅ Python urllib 可下载 | `urllib.request.urlopen` |
| GitHub HTTPS/SSH | ❌ 直连被墙 | 需代理或镜像 |
| jsDelivr CDN | ✅ 单个文件可访问 | curl |
| npmjs.org | ❌ DNS 不可达 | 需 npmmirror 镜像 |
| npmmirror.com | ✅ 可访问 | npm registry 镜像 |
| Electron 二进制 (GitHub) | ❌ ECONNRESET | 需 ELECTRON_MIRROR |
| Electron 二进制 (npmmirror) | ✅ 可访问 | 临时环境变量 |
| Playwright 浏览器 (GitHub) | ❌ 被墙 | 需 PLAYWRIGHT_DOWNLOAD_HOST |

---

## 完整使用示例

**场景：用户要 `npm install`，但发现不通**

```
用户: npm install 一直超时，帮我看看

Agent: [加载 network-diagnostics skill]
       → Phase 0: 意图识别 → 锁定 ①②⑦（代理/curl/npm）
       → Phase 1: 诊断
       
       ━━━ 网络诊断 ━━━
       ① 代理   127.0.0.1:7890   ✅ 通
       ② curl   GitHub HTTPS      ✅ 通
       ⑦ npm    registry          ⚠️ ENOTFOUND
       
       → Phase 2:
       ⑦ npm DNS 不通 → 询问 → 用户选 F(以后都自动)
       → 执行 npm config set registry npmmirror.com
       
       → Phase 3: 复验通过 ✓
```

---

## 陷阱

- **代理不通是阻断性错误**：如果①检测失败，后续所有项都无法修复。直接报错让用户开代理。
- **不设全局 HTTPS_PROXY**：设置全局环境变量会导致代理退出后所有工具（包括 hermes 自身 API 调用）瘫痪。各工具独立配置自己的代理/镜像。
- **Windows curl 可能走系统代理**：在 Windows + git-bash 中，curl 可能通过 Internet Options 系统代理设置自动走代理。②通但其他工具不通是正常现象。
- **`python` vs `python3`**：Windows 上通常只有 `python`。检测时先用 `which python` 确认。
- **`pip` 可能未安装**：部分环境只用 `uv`。如 `pip` 不存在，跳过⑤，用⑥（uv）代替。
- **curl 在某些 git-bash 版本中缺失**：如 `curl` 不可用，可用 Python urllib 替代所有 HTTP 检测。
- **npm ping 显示的是当前 registry**：如果用户已切换到 npmmirror，`npm ping` 会 ping 镜像而非 npmjs.org。
- **`npm config set electron_mirror ...` 不生效**：Electron 下载器读的是环境变量，不是 npm config。必须用 `export ELECTRON_MIRROR=...`，安装后立即 unset。
- **安装脚本修补**：当安装脚本内置了 `git clone`/`git fetch` 且无法跳过时，先手动下载源码（Python urllib），初始化本地 git 仓库，再修改安装脚本跳过网络操作。

---

## 参考文件

- `references/error-signatures.md` — GFW 错误签名速查表（各工具被墙时的特征性错误信息）
- `scripts/diagnose.sh` — 离线自救脚本。当代理退出、Agent 自身无法响应时，用户在终端直接运行：`bash diagnose.sh`
