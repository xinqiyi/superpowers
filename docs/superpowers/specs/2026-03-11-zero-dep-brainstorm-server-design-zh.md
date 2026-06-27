# 零依赖脑暴服务器

将脑暴伴侣服务器的 vendored node_modules（express、ws、chokidar——714 个跟踪文件）替换为仅使用 Node.js 内置模块的单个零依赖 `server.js`。

## 动机

将 node_modules 引入 git 仓库会带来供应链风险：冻结的依赖不会获得安全补丁，714 个第三方代码文件在未经审计的情况下提交，且对 vendored 代码的修改看起来像普通提交。虽然实际风险很低（仅 localhost 的开发服务器），但消除它是直接的。

## 架构

单个 `server.js` 文件（约 250-300 行）使用 `http`、`crypto`、`fs` 和 `path`。该文件扮演两个角色：

- **直接运行**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require**（`require('./server.js')`）：导出用于单元测试的 WebSocket 协议函数

### WebSocket 协议

实现仅用于文本帧的 RFC 6455：

**握手：** 使用 SHA-1 + RFC 6455 魔术 GUID 从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种掩码长度编码：
- 小型：payload < 126 字节
- 中型：126-65535 字节（16 位扩展）
- 大型：> 65535 字节（64 位扩展）

使用 4 字节掩码密钥进行 XOR 解掩码 payload。返回 `{ opcode, payload, bytesConsumed }` 或 `null`（缓冲区不完整时）。拒绝未掩码的帧。

**帧编码（服务器到客户端）：** 具有相同三种长度编码的未掩码帧。

**处理的 Opcodes：** TEXT（0x01）、CLOSE（0x08）、PING（0x09）、PONG（0x0A）。无法识别的 opcode 获得状态 1003（不支持的数据）的关闭帧。

**有意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。对于 localhost 客户端之间的小型 JSON 文本消息来说，这些是不必要的。扩展和子协议在握手中协商——通过不宣传它们，它们永远不会激活。

**缓冲区累积：** 每个连接维护一个缓冲区。在 `data` 上，追加并循环 `decodeFrame` 直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`**——按 mtime 提供 screen 目录中最新的 `.html`。检测完整文档与片段，将片段包裹在框架模板中，注入 helper.js。返回 `text/html`。当没有 `.html` 文件存在时，提供硬编码的等待页面（"Waiting for Claude to push a screen..."），注入 helper.js。
2. **`GET /files/*`**——从 screen 目录提供静态文件，从硬编码的扩展名映射（html、css、js、png、jpg、gif、svg、json）中查找 MIME 类型。如果未找到则返回 404。
3. **其他所有**——404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理器分开。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT`——绑定端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST`——绑定接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST`——启动 JSON 中 URL 的主机名（默认：当 host 为 `127.0.0.1` 时为 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR`——screen 目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 如果 `SCREEN_DIR` 不存在则创建（`mkdirSync` 递归）
2. 从 `__dirname` 加载框架模板和 helper.js
3. 在配置的主机/端口上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 监听成功后，将 `server-started` JSON 记录到 stdout：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便当 stdout 被隐藏（后台执行）时，agent 可以找到连接详情

### 应用级 WebSocket 消息

当从客户端到达 TEXT 帧时：

1. 解析为 JSON。如果解析失败，记录到 stderr 并继续。
2. 作为 `{ source: 'user-event', ...event }` 记录到 stdout。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每事件一行）。

### 文件监视

`fs.watch(SCREEN_DIR)` 替换 chokidar。在 HTML 文件事件上：

- 新文件（存在的文件的 `rename` 事件）：删除 `.events` 文件（如果存在），将 `screen-added` 作为 JSON 记录到 stdout
- 文件修改（`change` 事件）：将 `screen-updated` 作为 JSON 记录到 stdout（不清除 `.events`）
- 两个事件：向所有连接的 WebSocket 客户端发送 `{ type: 'reload' }`

对每个文件名进行约 100ms 的防抖处理，以防止重复事件（macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端发来的格式错误的 JSON：记录到 stderr，继续
- 未处理的 opcode：用状态 1003 关闭
- 客户端断开：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 无优雅关闭逻辑——shell 脚本通过 SIGTERM 处理进程生命周期

## 更改内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单个文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从 screen 目录提供 |

## 保持不变

- `helper.js`——无更改
- `frame-template.html`——无更改
- `start-server.sh`——一行更新：`index.js` → `server.js`
- `stop-server.sh`——无更改
- `visual-companion.md`——无更改
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上对单个扁平目录可靠
- Shell 脚本需要 bash（Windows 上的 Git Bash，Claude Code 需要）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 的导出直接测试 WebSocket 帧编码/解码、握手计算和协议边缘情况。

**集成测试**（`server.test.js`）：测试完整的服务器行为——HTTP 服务、WebSocket 通信、文件监视、脑暴工作流。使用 `ws` npm 包作为仅测试的客户端依赖（不交付给最终用户）。
