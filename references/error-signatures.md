# GFW 错误签名速查

记录已被 GFW 的网络行为产生的特征性错误信息，用于快速诊断。

## GitHub git clone

### SSH
```
Cloning into '...'...
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```
或直接挂住不输出任何内容（`ConnectTimeout` 对数据传输阶段无效）。

### HTTPS
```
fatal: unable to access 'https://github.com/.../': Recv failure: Connection was reset
```

### GitHub API / codeload (curl)
```
curl: (35) Recv failure: Connection was reset
```

## npm registry

```
npm error code ENOTFOUND
npm error syscall getaddrinfo
npm error errno ENOTFOUND
npm error network request to https://registry.npmjs.org/... failed, reason: getaddrinfo ENOTFOUND registry.npmjs.org
```

## Electron postinstall

```
npm error path .../node_modules/electron
npm error command failed
npm error command ... node install.js
npm error RequestError: read ECONNRESET
```

## Playwright 浏览器下载

进程挂起，`__dirlock` 文件出现在 `%LOCALAPPDATA%/ms-playwright/` 但浏览器文件始终不出现。

## DNS 特征

- `nslookup github.com` 能解析到 IP（DNS 未被污染）
- 但 TCP 连接被 RST 或超时
- 说明是 IP 层 / DPI 封锁，非 DNS 污染

## 镜像故障特征

| 症状 | 含义 |
|------|------|
| `Could not resolve host` | DNS 失败，镜像站已下线 |
| `Connection timed out` | IP 被封或服务已停 |
| `HTTP 502` | 上游 GitHub 不可达 |
| `SEC_E_ILLEGAL_MESSAGE` (Windows schannel) | SSL/TLS 握手失败，可能是中间人干扰 |
| `Recv failure: Connection was reset` | TCP RST，典型的 DPI 封锁 |
