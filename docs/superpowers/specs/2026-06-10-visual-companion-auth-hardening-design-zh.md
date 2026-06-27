# 可视化伴侣认证加固设计

**日期：** 2026-06-10
**状态：** Drew 审查的草稿

## 目标

修复 PR #1720 脑暴可视化伴侣中发现的安全和可靠性差距，而不改变伴侣的核心工作流或添加运行时依赖。

修复必须是测试优先的，并必须为以下内容留下清晰的自动化证据：

- 跨源浏览器标签不能通过携带 cookie 来注入伴侣事件
- 重启重连工作，不仅依赖浏览器 cookie 行为
- Bearer 密钥在引导后不在可见 URL 中保留
- `/files/*` 不能提供内容目录之外的文件
- 未来的同源 vendored UI 库仍然工作

## 威胁模型

伴侣为单个脑暴会话提供 agent 生成的本地 UI。重要资产是：

- 从伴侣提供的 screen 内容
- 会话密钥
- `state/events`，agent 将其读取为用户反馈
- 伴侣会话目录下的本地文件

范围内的攻击者：

- 另一个 `localhost` 端口上的恶意浏览器标签
- 一个可以向伴侣发出请求但不应能作为伴侣 UI 认证的浏览器页面
- 当服务器绑定到非回环接口时的直接远程客户端
- 通过 URL 历史、引荐来源或承诺的本地状态的意外泄露
- 逃逸 `/files/*` 的内容目录符号链接或路径技巧

此修复范围外的：

- 恶意 agent 编写的 screen HTML
- 由伴侣 screen 加载的恶意同源 vendored JavaScript

此范围外边界是有意的。伴侣 screen 是 agent UI 表面的一部分。它们今天可以使用内联脚本，未来可能使用同源 vendored 库如 Alpine 或 Three.js。防范恶意 screen HTML 需要一个更大的沙箱化 iframe 架构，带有窄消息桥；那不是此 PR 加固轮次的范围。

## 当前失败

自动化和有头浏览器测试在 PR 分支中发现了这些失败：

1. 一个跨源 localhost 页面可以在真实伴侣页面设置 cookie 后打开 cookie 认证的 WebSocket，并将攻击者控制的选项写入 `state/events`。
2. `/files/*` 提供指向 `content/` 外部的符号链接，包括指向包含密钥 URL 的 `state/server-info` 的符号链接。
3. 会话密钥保留在实际 screen 页面的 URL 中，因此同源 screen JavaScript 和意外的引荐来源/历史可以看到它。
4. Helper 使用无密钥的 `ws://host` URL 重连。在有头 Chrome 中，在相同端口/相同 token 重启后，浏览器停止向重启的服务器提供 cookie，因此打开的标签页停留在墓碑页面上，直到手动重新加载。
5. Shell lint 和生命周期测试需要清理，以便测试通过可以在 Codex 中稳定。

## 设计

### 1. 带密钥的引导加载

`GET /?key=<token>` 变成引导响应，而非 screen 响应。

当密钥有效时，服务器：

1. 像今天一样设置 HttpOnly 会话 cookie
2. 返回一个小型 HTML 引导页面
3. 引导页面将密钥存储在标签范围的 `sessionStorage` 中
4. 引导页面使用 `location.replace('/')` 导航到 `/`

此后，可见的 screen URL 是裸 `/`，而非 `/?key=...`。

带有有效 cookie 的 `GET /` 提供当前 screen。无有效 cookie 的 `GET /` 仍然返回友好的 403 页面。`GET /?key=<wrong>` 返回 403。

为什么使用 `sessionStorage`：helper 需要一个在相同端口重启后存活且不仅依赖 cookie 行为的重连凭证。由于 screen HTML 是受信任的同源 UI，将密钥存储在标签范围的存储中对这个威胁模型是可接受的。它比将密钥留在地址栏、历史和引荐来源表面中要好得多。

### 2. WebSocket 同源强制

WebSocket 升级必须通过两个检查：

1. 有效的会话认证，通过查询密钥或 cookie
2. 如果存在 `Origin` 头部，它必须匹配请求目标来源

Origin 检查应比较：

```text
Origin === "http://" + req.headers.host
```

浏览器攻击者页面示例：

```text
Origin: http://localhost:9999
Host: localhost:58088
```

即使浏览器发送伴侣 cookie，这也必须被拒绝。

合法伴侣页面示例：

```text
Origin: http://localhost:58088
Host: localhost:58088
```

当密钥或 cookie 有效时，这应被接受。

直接非浏览器客户端可能省略 `Origin`；它们仍然需要会话密钥。

### 3. Helper 重连凭证

`helper.js` 应从 `sessionStorage` 读取标签范围的密钥并将其附加到 WebSocket URL：

```text
ws://<host>/?key=<stored-key>
```

如果不存在存储的密钥，helper 回退到当前仅 cookie 的 `ws://<host>` 行为。这保留了已加载页面的兼容性，这些页面确实有有效的 cookie 但没有存储条目。

### 4. `/files/*` 包含

文件服务器应继续拒绝空名称和点文件。它还必须确保文件是 `CONTENT_DIR` 内的真实常规文件。

使用 realpath 包含作为边界：

- 计算 `realContentDir = fs.realpathSync(CONTENT_DIR)`
- 计算 `realFilePath = fs.realpathSync(filePath)`
- 仅当 `realFilePath` 等于 `realContentDir` 的后代时提供服务
- 拒绝符号链接和内容目录之外的任何内容，返回 404

服务器应继续使用 `path.basename`，因此嵌套路径保持不支持。

### 5. 泄露减少头

添加不妨碍内联脚本或未来同源 vendored 库的保守头：

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

在此轮次中不要添加限制性的 `script-src` CSP。伴侣当前注入内联的 helper JavaScript，未来的 screen 可能加载同源 vendored 库。

### 6. Gitignore 持久会话状态

将 `.superpowers/` 添加到仓库根 `.gitignore`，以便使用 `--project-dir` 时不会意外提交持久化的伴侣状态和 `.last-token`。

### 7. 测试稳定性和 Lint

清理触及的启动/停止脚本中的 shell lint 警告。

更新调用 `start-server.sh --idle-timeout-minutes` 的生命周期测试，使其不会在 Codex 的 `CODEX_CI` 前台自动检测下挂起。测试应在期望脚本返回启动 JSON 时强制后台模式，使用 `--background`。

## 测试策略

所有行为更改应是 TDD：

1. 编写失败的聚焦测试
2. 运行并确认它因预期原因失败
3. 实施最小修复
4. 重新运行聚焦测试
5. 重新运行完整 brainstorm-server 套件

必需的聚焦回归测试：

- 带有效密钥的 `/` 返回引导页面，而非 screen 内容
- 引导页面在 `sessionStorage` 中存储密钥并剥离 URL
- 仅 cookie 的 `/` 仍然提供 screen 内容
- Helper 使用 `sessionStorage` 密钥用于 WebSocket URL
- 同源 cookie WebSocket 打开
- 跨源 cookie WebSocket 被拒绝且不写入事件
- 直接密钥 WebSocket 在没有 `Origin` 时仍然打开
- `content/` 下指向 `state/server-info` 的符号链接返回 404
- 安全头存在于正常 HTML、引导、403 和文件响应上
- 相同端口/token 重启可以使用存储的密钥认证重连
- Shell lint 对触及的 shell 脚本通过
- 生命周期套件在 Codex 下不挂起

## 验收标准

- `cd tests/brainstorm-server && npm test` 重复通过且不挂起。
- 之前从另一个 localhost 来源写入 `attacker-injected` 的安全探测现在无法打开 WebSocket，且 `state/events` 保持不变。
- 到 `server-info` 的符号链接探测返回 404。
- 有头或无头浏览器带密钥加载结束于裸 `/` URL，且状态胶囊达到 Connected。
- 相同端口/相同 token 重启自动重连，无需手动重新加载。
- `scripts/lint-shell.sh` 对触及的 shell 脚本通过。

## 推迟的工作

如果项目后来需要将 screen HTML 视为不受信任，设计一个单独的沙箱化 iframe 架构。那应在单独的来源或沙箱化框架上隔离生成的 screen，并仅暴露一个窄 `postMessage` 桥用于用户选择。不要将其捆绑到此修复中。
