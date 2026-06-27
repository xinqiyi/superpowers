# 可视化伴侣最终加固修复设计

**日期：** 2026-06-11
**状态：** Drew 审查的草稿

## 目标

完成 PR #1720 可视化伴侣加固轮次，使分支准备好供 Jesse 审查，具有干净的安全行为、确定性测试和仅包含伴侣工作的 PR 差异。

这是在现有认证加固设计之上的修复。不应重新设计伴侣或扩展功能表面。

## 背景

之前的加固轮次添加了带密钥的会话、同源 WebSocket 检查、URL 密钥剥离、`/files/*` 包含、泄露减少头、IPv6 URL 格式、Windows 生命周期覆盖和 PR 证据更新。

最终审查发现了五个剩余问题：

1. 根 `GET /` screen 选择路径仍然可以提供 `content/` 下指向内容目录外部的符号链接或硬链接。
2. 当首选端口被占用时，回退服务器可以重用持久化的 `.last-token`，创建两个具有相同 bearer 密钥的实时同项目伴侣服务器。
3. 当无法提供强有力的所有权证明时，`stop-server.sh` 可以向不相关的 `node server.cjs` 进程发信号。
4. 一些测试可以针对错误的回退进程通过，在失败时泄漏后台进程，或假设类似 Windows 的主机支持符号链接。
5. PR 当前存在冲突，因为分支包含一个已单独处理的旧 `evals` 子模块更新。

## 非目标

- 此轮次不添加 HTTPS 隧道或 `wss://` 来源语义。
- 不实现退出、自由文本或对比辅助伴侣功能。
- 不引入 Alpine、Three.js 或任何其他 JavaScript 库。
- 不尝试沙箱化恶意 agent 编写的 screen HTML。
- 不添加对过期 stop-server PID 文件的向后兼容性，除非 Drew 明确批准该权衡。

## 继承的安全不变量

此修复保留已设计并实施的认证加固：

- `.last-token` 和 `state/server-info` 仍然是敏感的拥有者仅状态。
- 回退 token 可能出现在启动 JSON 和 `state/server-info` 中，但不得写入 `.last-token`。
- Cookie 保持按端口命名、`HttpOnly`、`SameSite=Strict` 且范围限定到 `/`。
- WebSocket 升级仍然需要有效的密钥或 cookie。
- 当浏览器提供 `Origin` 头部时，WebSocket `Origin` 检查仍然强制执行。
- 直接的无 `Origin` 客户端仅当它们携带会话密钥时被允许。
- 生成的同源 screen JavaScript 和未来的同源 vendored 库是受信任的。沙箱化恶意的 screen HTML 仍然推迟。

## 设计

### 1. 重新基于当前 `dev`

在实施工作之前将 `brainstorming-companion` 重新基于当前的 `origin/dev`。通过采用 `dev` 解决 `evals` 子模块冲突。

重新基础后：

- `evals` 不得出现在 PR 差异中。
- PR #1720 仍然可以提及在别处运行的评估证据，但它必须包括确切的外部证据：评估仓库提交、场景路径、命令、结果产出路径或 ID，以及 RED/GREEN 结果。
- PR 正文不得暗示 evals 子模块更新是此 PR 的一部分。
- 任何暗示子模块更新包含在内的早期 PR 正文文本或评论必须被最终的 PR 正文证据取代。

### 2. 根 Screen 包含

根 screen 路由必须使用与 `/files/*` 相同的包含边界。

`getNewestScreen()` 应忽略任何未通过常规文件在内容目录内保护检查的 `.html` 候选。该保护必须解析真实路径并确保提供的文件在 `CONTENT_DIR` 内部。它还必须通过当平台报告链接计数时拒绝链接计数不是一的文件来保留现有的硬链接保护。

预期行为：

- `content/` 下指向 `content/` 外部的符号链接被忽略。
- `content/` 下到 `state/server-info` 的硬链接在 `fs.linkSync` 成功且 `lstat.nlink > 1` 时被忽略。
- 如果没有安全的 screen 文件保留，则提供等待页面。
- 现有的 `/files/*` 包含行为保持不变：空名称、点文件、符号链接、硬链接和目录仍然返回 404。

### 3. 回退 Token 隔离

端口回退不得重用从持久化的 `.last-token` 加载的 token。

Token 来源应在代码中明确：

- 来自环境的 `BRAINSTORM_TOKEN` 是有意的操作员/测试覆盖。如果当设置了明确的环境 token 时首选端口被占用，服务器必须安全失败而非回退，因为被占用的服务器可能正在使用相同的明确 token。
- `.last-token` 是为相同端口重连便利性而持久化的状态。如果服务器因首选端口被占用而回退，丢弃该加载的 token 并为回退进程生成一个新鲜的未持久化 token。
- 未从 `.last-token` 加载的新生成的 token 可以在同一进程中重用，因为没有其他存活进程已知拥有它。

回退服务器必须继续避免覆盖 `.last-port` 和 `.last-token`。

### 4. Stop-Server 所有权证明

`start-server.sh` 应创建一个每次启动的服务器实例 ID，并将其作为惰性命令行参数传递给 Node，例如：

```text
node server.cjs --brainstorm-server-id=<id>
```

该 ID 不是认证凭证。它只是本地生命周期脚本的进程所有权证据。`server.cjs` 可以忽略该参数。

ID 必须使用 shell/MSYS 安全的字母表，例如 `^[A-Za-z0-9_-]{32,64}$`。将其存储在 `state/server-instance-id` 中，拥有者仅权限。

`stop-server.sh` 应从状态读取预期的 ID，并且仅当目标进程 argv 包含确切的参数 `--brainstorm-server-id=<id>` 作为完整 argv token（而非宽松的子串）时才向该 PID 发信号。当可用时优先使用 `/proc/<pid>/cmdline`，然后回退到宽 `ps` 输出。匹配的实例 ID 是足够的证据，即使 `server-info` 缺失或 `lsof` 不可用。现有的端口到 PID 检查可以作为额外证据保留。

当无法证明所有权时安全失败：

- 缺失 PID 文件
- 缺失或格式错误的服务器 ID
- 目标命令行不可用
- 目标命令行不包含预期的 ID
- 没有新 ID 的旧/过期会话元数据

这有意偏好保留过期进程运行而不是杀死不相关进程。

操作者可观察的结果应是明确的：

- 缺失 PID 文件返回 `not_running`
- 缺失或格式错误的服务器 ID 返回 `stale_pid`
- 不可用的命令行返回 `stale_pid`
- 错误或缺失的 argv ID 返回 `stale_pid`
- 成功停止返回 `stopped`

在 `stale_pid` 和 `stopped` 结果上，移除 `server.pid` 和 `server-instance-id`，以便未来的停止尝试不会继续定位到相同的歧义进程。不移除持久化的会话内容。

### 5. 测试加固

测试通过应在 macOS 和用于验证的 Windows Git Bash 主机上是确定性的。

必需的更改：

- 固定端口套件必须要么在服务器报告回退端口时快速失败，要么从报告的启动端口驱动所有客户端。
- `stop-server.test.sh` 需要在任何后台进程启动之前有一个顶级清理 trap。
- 符号链接特定的断言应探测符号链接能力，并且在主机无法创建可用的测试符号链接时仅跳过该断言。
- 创建冒名顶替者进程的测试必须断言当生命周期元数据缺失或不足时冒名顶替者存活。
- Windows/MSYS start-server 测试必须断言类似 Windows 的检测仍然清除 `BRAINSTORM_OWNER_PID`、仍然在适当时自动前台化、并且仍然精确传递实例 ID argv。

### 6. 文档和 PR 一致性

在 Jesse 审查之前，协调审查者可看到的文档和 PR 元数据：

- 更新问题目录，使处理方式匹配此 PR 实际交付的内容。
- 保持自动打开文档与已实现的 `--open` 行为一致。
- 保持记录的默认空闲超时在所有地方为 4 小时。
- 重新基础后对照模板审查 PR 正文。
- 在 PR 正文中记录 macOS、Windows、浏览器/手动和外部评估证据，带有具体命令和结果。

## 测试策略

为每个行为更改使用 TDD：

1. 添加或收紧一个聚焦的回归测试。
2. 运行并确认它因预期原因失败。
3. 实施最小的修复。
4. 重新运行聚焦测试。
5. 重新运行完整 brainstorm-server 套件。

必需的聚焦回归测试：

| 行为 | 测试文件 | 聚焦命令 | 预期 RED | 预期 GREEN |
| --- | --- | --- | --- | --- |
| 根路由忽略符号链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 认证的 `GET /` 提供链接的外部内容 | 响应提供等待页面或安全 screen |
| 根路由忽略受支持硬链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 认证的 `GET /` 提供硬链接的 `server-info` | 当 `nlink > 1` 时硬链接候选被忽略 |
| `/files/*` 包含保持不变 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 现有包含测试回归 | 空、点文件、目录、符号链接、硬链接案例仍然为 404 |
| 持久化 token 回退轮换 token | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 URL 密钥等于持久化的首选端口密钥 | 回退 URL 密钥不同且未写入 `.last-token` |
| 明确 token 回退安全失败 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 服务器在 `BRAINSTORM_TOKEN` 设置时回退 | 进程以非零退出且不启动回退 |
| 回退密钥不能认证到原始服务器 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退密钥从原始端口收到 200 | 原始端口拒绝回退密钥 |
| 正确实例 ID 允许停止 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 真实的 start-server 启动的服务器存活 | 停止返回 `stopped` 且进程退出 |
| 错误、缺失、格式错误或过期 ID 安全 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 冒名顶替者被发信号 | 停止返回 `stale_pid` 且冒名顶替者存活 |
| 固定端口套件不能通过回退 | `tests/brainstorm-server/server.test.js`、`tests/brainstorm-server/auth.test.js` | 各自的 `node` 命令 | 测试静默与回退端口通信 | 测试明确失败或有意使用报告端口 |
| Shell 清理 trap 在失败时运行 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 失败留下子进程 | trap 回收后台子进程 |
| Windows/MSYS 启动行为保持生命周期不变量 | `tests/brainstorm-server/start-server.test.sh`、`tests/brainstorm-server/windows-lifecycle.test.sh` | macOS 和 `ballmer` 上的 `bash` 测试命令 | 拥有者 PID 或 argv 处理回归 | 拥有者 PID 被清除，前台检测保持，ID argv 存在 |

每个 RED/GREEN 循环应为 PR 正文留下简短的证据说明：聚焦命令、修复前失败的断言、修复后通过的断言以及证据是在 macOS 还是 Windows 上收集的。

## 验证

在调用修复完成之前，运行：

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 期间使用的相关聚焦测试命令
- `git diff --check`
- Node 语法检查触及的 JavaScript 文件
- Shell lint 检查触及的 shell 文件
- Windows 在 `ballmer` 上的验证：完整可运行的 brainstorm-server 套件加上独立的 Windows 生命周期探测

手动/浏览器测试仅在自动化通过为绿色后才进行。

## 验收标准

- PR #1720 干净地重新基于当前 `dev`。
- `evals` 不在 PR 差异中。
- 根 screen 服务不能通过符号链接或受支持的硬链接逃逸读取 `content/` 外部。
- `/files/*` 包含保护保持不变。
- 没有回退服务器运行着可能与被占用的首选端口服务器共享的 token。
- `stop-server.sh` 在所有权证明缺失或歧义时不向不相关进程发信号。
- `stop-server.sh` 仍然可以在 `server-info` 或 `lsof` 不可用时停止具有匹配实例 ID 的合法服务器。
- 为每个回归记录聚焦的 RED/GREEN 证据。
- macOS 和 Windows 验证证据记录在 PR 正文中。
- PR 正文准确描述了分支中的内容以及在外部收集了什么证据。
