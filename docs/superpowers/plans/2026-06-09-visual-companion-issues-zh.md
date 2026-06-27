# 可视化脑暴伴侣——问题与变更目录

**日期：** 2026-06-09
**状态：** 分析 / 分类。我们将自行实施这些；引用的社区 PR 是证据和参考材料，**不是**我们打算合并的代码。

## 目的

一个单一位置，捕获涉及可视化脑暴伴侣（`skills/brainstorming/scripts/` 中的本地服务器）的每个未解决问题和 PR，提炼为根本问题和我们将要进行的更改。每个条目基于当前代码进行评估，而非 PR 作者的描述。

## 范围决策（Jesse，2026-06-09）

- **不引入 Alpine.js。** PR #1639（通过 vendored Alpine 构建实现交互式 mockup）**已放弃**。参见 E3。
- **E1（终端与 HTML 的硬关卡）是一个研讨项目。** 我们将一起设计；此处未进行 spec 设计。
- **E2（存储位置，#975/#977）暂时推迟**。
- **远程服务是一等场景。** Superpowers 是通用的；用户从远程连接（SSH 隧道、Tailscale、`--host 0.0.0.0`）。安全修复必须保护这些用户，而不仅仅是回环。**决策：使用每会话密钥**，而非 Host 允许列表。Host 允许列表仅防御回环浏览器混淆代理；直接远程客户端只发送预期的 `Host`，因此允许列表对于远程暴露只是做做样子。密钥是唯一在回环、隧道和直接远程之间统一认证客户端的机制，并且还能防御 DNS 重绑定。参见 A1。

## 组件映射

| 文件 | 角色 |
|------|------|
| `skills/brainstorming/scripts/server.cjs` | 零依赖 HTTP + WebSocket 服务器（RFC 6455 手写实现）。提供最新 screen，监视 `content/`，将事件记录到 `state/events`。 |
| `skills/brainstorming/scripts/helper.js` | 注入到每个页面。WebSocket 客户端、点击捕获、`window.brainstorm` API。 |
| `skills/brainstorming/scripts/frame-template.html` | 框架（头部、主题 CSS、状态点、指示器栏）包裹内容片段。 |
| `skills/brainstorming/scripts/start-server.sh` | 启动包装器。会话目录、host/url-host、拥有者 PID 解析、平台后台化。 |
| `skills/brainstorming/scripts/stop-server.sh` | 通过 PID 文件杀死服务器，清理 `/tmp` 会话。 |
| `skills/brainstorming/visual-companion.md` | agent 接受伴侣时读取的操作指南。 |
| `skills/brainstorming/SKILL.md` | 提供伴侣和每个问题决策的地方。 |

## 处理总结

| ID | 项目 | 来源 | 处理方式 |
|----|------|--------|-------------|
| A1 | 在 `/`、`/files/*` 和 WS 上使用每会话密钥（取代 Host 允许列表） | issues #1014, PRs #1110/#1553 | **执行**——所选方法 |
| A2 | Host 允许列表；浏览器 WS Origin 检查 | PRs #1110/#1553 | Host 允许列表已放弃；WS Origin 检查在认证后保留，用于浏览器混淆代理防御 |
| A3 | 在 `null` / 非对象 WS payload 上崩溃 | PR #1504 | 执行 |
| A4 | decodeFrame 中的帧长度限制 | issue #1446 | 已修复——验证/关闭 |
| B1 | 点文件 screen 作为内容提供（`._*.html`） | PR #950 | 执行 |
| B2 | `stop-server.sh` 杀死已重用/过期的 PID | PR #1703 | 执行 |
| B3 | WS 客户端重连回退 + 状态指示器 | PR #856 | 执行 |
| C1 | 空闲超时太短/不可配置；关闭时 WS 未关闭 | issue #1237 (PR #1689) | 执行 |
| C2 | 服务器死亡对用户/agent 不可见 | issue #1237（残留） | 执行 |
| D1 | 永久退出伴侣 | issue #892 | 推迟——不在 PR #1720 中 |
| D2 | 来自浏览器的自由文本反馈 | issue #957 | 推迟——不在 PR #1720 中 |
| D3 | 自动打开伴侣 URL | PR #759 (#755) | 在 PR #1720 中通过 `--open` 完成 |
| D4 | 框架中的亮/暗对比辅助类 | PR #1683 | 推迟——不在 PR #1720 中 |
| E1 | 每个问题的终端与 HTML 硬关卡 | PR #1037 | **研讨** |
| E2 | 将会话状态移出工作树 | issue #975 (PR #977) | **推迟** |
| E3 | 引入 Alpine.js 用于交互式 mockup | PR #1639 | **放弃** |
| E4 | 启动/停止脚本中的 Shell-lint 警告 | PR #1677 | 仅机会主义 |

---

## A. 服务器安全加固（`server.cjs`）

### A1——每会话密钥（所选方法）

**威胁模型。** 两个资产：提供 screen（`/`）和文件（`/files/*`）的机密性，以及 `state/events` 的完整性——一个具有真值 `choice` 的 WebSocket 客户端在那里写入（`server.cjs:243-246`），agent 在下一轮将其读取为用户的选择，即 **注入到具有完全工具访问权限的实时会话中的提示**。攻击者：使用默认 `127.0.0.1` 绑定时，用户浏览器中的恶意页面（混淆代理——运行攻击者 JS *并且*可以到达回环）；使用远程绑定（`--host 0.0.0.0`、tailnet/LAN）时，任何可以路由到该端口的主机，直接，无同源策略阻挡。当前 `handleUpgrade`（`server.cjs:176`）仅检查 `Sec-WebSocket-Key`，而 `handleRequest`（`server.cjs:138`）什么也不检查——两者都完全开放。

**为什么使用密钥，而非 Host 允许列表。** Host 允许列表仅防御回环浏览器代理。直接远程客户端只发送预期的 `Host` 并伪造/省略 `Origin`，因此允许列表对于我们必须保护的远程情况只是做做样子。每会话密钥在回环、SSH 隧道和直接远程之间统一认证客户端，并且还能消除 DNS 重绑定（重绑定的页面既不知道密钥也不接收主机范围的 cookie）。因此密钥**取代**了 A1/A2 的 Host 允许列表——完全不需要 `BRAINSTORM_ALLOWED_HOSTS`。

**设计。** 随机 token（`crypto.randomBytes(32)` hex），在 `server.cjs` 启动时生成（可通过 `BRAINSTORM_TOKEN` 覆盖，用于确定性测试）：

1. **URL 携带它**作为 `?key=<token>`。服务器已经在 `server-started` JSON（`server.cjs:351`）中构建 `url` 并写入 `state/server-info`——在那里追加 `?key=` 意味着 `start-server.sh`（grep 并打印该 JSON）和 skill（将 URL 交给用户）**无需更改**。
2. **Cookie 引导。** `/` 上的有效 `?key` 设置 `brainstorm-key-<port>=<token>; HttpOnly; SameSite=Strict; Path=/`。浏览器然后自动将其附加到同源子资源（`/files/*`）和 WebSocket 握手，因此 agent 可以编写任何 URL 样式且都能工作，且 `helper.js` 无需更改。Cookie 名称**按端口**以避免 Jupyter 多服务器冲突（cookie 不按端口范围划分）。`SameSite=Strict` 对于 CDN/Unsplash 内容是安全的——该 cookie 是主机范围的，因此出站 CDN 请求从不携带它；SameSite 仅管理回我们源的请求，这些请求都是同站的。
3. **认证关卡** = 有效的 `?key` **或** 有效的 cookie（使用 `crypto.timingSafeEqual` 比较）应用于 `/`、`/files/*` 和 WS 升级。缺失/错误的 key → 友好的 **403 HTML 页面**（"this page needs the full URL your coding agent gave you, including `?key=…`"——通用的"coding agent"，而非"Claude"，因为这也用于 Codex/Gemini/Copilot）。WS 升级 → 销毁 socket。

查询 token 是真实来源；cookie 是一种便利，从不承担初始认证负载。

**影响范围。** `server.cjs`（所有逻辑）。`helper.js` 可选的一行（从 `location.search` 将 `?key=` 追加到 WS URL 作为被 cookie 阻止的回退）。`start-server.sh` 无。`visual-companion.md` 文档说明（URL 现在有 `?key=`；不要剥离它）。测试更新以传递 token。

### A2——Host 允许列表已放弃；浏览器 WS Origin 保留

被 A1 取代。密钥在一个机制中关闭了 WS 注入向量（#1014）、HTTP/WS DNS 重绑定读取向量（PR #1553）和跨源 WS 向量（PR #1110），并且与允许列表不同，它实际上保护了远程绑定情况。无 `BRAINSTORM_ALLOWED_HOSTS`，无 Host 允许列表。最终实现仍在会话认证后检查浏览器 WebSocket `Origin`，以便跨源 localhost 标签不能使用伴侣 cookie。

### A3——服务器在 `null` / 原始 WS payload 上崩溃

**问题。** `handleMessage`（`server.cjs:233`）执行 `JSON.parse(text)` 然后在 `server.cjs:243` 执行 `if (event.choice)`。发送 4 字节文本帧 `null` 的客户端导致 `event === null`，且 `null.choice` 抛出。该抛出**未被**捕获——`handleMessage` 从 `socket.on('data')` 处理器（`server.cjs:207`）调用，该处理器位于 `try/catch` 之外，仅包装 `decodeFrame`。结果是未捕获的异常和进程退出。任何本地客户端都可以杀死服务器。

**更改。** 保护访问：`if (event && event.choice)`。最小且精确——`JSON.parse` 不能产生 `undefined`，且原始类型返回 `undefined` 用于 `.choice` 而不抛出，因此只有 `null` 是实际危险。（避免更广泛的修复——顶级的 `try/catch` 或 `process.on('uncaughtException')` 会掩盖其他 bug。）

### A4——decodeFrame 中的帧长度限制（邻近）

被 PR #1504 引用为 #1446。当前代码**已经**限制了扩展帧长度：`MAX_FRAME_PAYLOAD_BYTES = 10MB`（`server.cjs:10`）在 `server.cjs:58-67` 中的任何 `Buffer.alloc` 之前执行。措施：对照当前 `dev` 验证 #1446，如果已解决则关闭，而非重新实现。

---

## B. 服务器健壮性 / 正确性

### B1——macOS 资源分支点文件作为 screen 内容提供

**问题。** 最新 screen 选择器仅按 `f.endsWith('.html')` 过滤（`server.cjs:127-128`）。在 macOS/ExFAT 上，`._screen.html` 资源分支文件通过该过滤器，并且与真实文件一起写入时，可能排序为最新——因此浏览器获得二进制元数据而非 mockup。四个读取站点共享弱过滤器：`getNewestScreen`（`server.cjs:127`）、`knownFiles` 初始化（`server.cjs:279`）、`fs.watch` 处理器（`server.cjs:286`）和 `/files/` 端点（`server.cjs:154-156`）。

**更改。** 在所有四个站点拒绝点文件（`!f.startsWith('.')`）。涵盖 `._*`、`.DS_Store` 等。

### B2——`stop-server.sh` 可以杀死已重用的 PID

**问题。** `stop-server.sh` 从 `state/server.pid`（`stop-server.sh:20`）读取 PID 并 `kill` 它（`:23`，在 `:35` 升级到 `-9`），而不确认该 PID 仍然属于我们的服务器。在重启或 PID 回绕后，文件可能指向一个不相关的进程，然后我们就会 SIGKILL 它。

**更改。** 在发信号之前，验证所有权——该 PID 的命令是运行我们的 `server.cjs` 的 `node`，理想情况下匹配此会话。如果无法证明所有权，则安全失败（报告 `stale_pid`，不杀死）。保留现有的 `stopped` / `not_running` 输出用于真实情况。

### B3——WebSocket 客户端：静默重连，过时的"Connected"

**问题。** `helper.js` 在固定的 1 秒定时器上重连（`helper.js:21-23`），没有 `onerror` 处理器，在关闭时从未将 `ws` 置为 null，也从未清除待处理的重连定时器。框架的状态元素硬编码为"Connected"，点固定为 `var(--success)`（`frame-template.html:77,200`）。当笔记本电脑休眠或服务器重启时，页面在死连接上显示"Connected"并排队事件而无反馈。

**更改。**
- `helper.js`：指数退避（500ms → ×2 → 上限 30s，打开时重置）；`onerror` 委托给 `onclose`；关闭时 `ws = null`；重连前 `clearTimeout`。
- `frame-template.html`：从 `--status-color` 自定义属性驱动状态点，以便 JS 可以切换 Connected（绿色）/ Reconnecting（黄色）/ Disconnected（红色）。

---

## C. 生命周期 / 超时（issue #1237）

### C1——空闲超时太短，不可配置，WS 保持进程存活

**问题。** `IDLE_TIMEOUT_MS` 硬编码为 30 分钟（`server.cjs:258`），由 60 秒生命周期检查（`server.cjs:329-332`）强制执行。单个脑暴问题可能因为用户思考或离开而超过 30 分钟，因此服务器会在会话中途死亡。另外，`shutdown()`（`server.cjs:310-321`）调用 `server.close()` 但从未关闭 `clients`（`server.cjs:174`）中已升级的 socket，因此打开的浏览器连接可以在关闭后保持 Node 进程存活。

**更改。**
- 将默认值提高到 4 小时并使其可配置：`start-server.sh` 中的 `--idle-timeout-minutes` → 环境变量 → `IDLE_TIMEOUT_MS`，带 Node 定时器溢出验证。
- 在启动 JSON / `state/server-info` 中暴露有效的超时值。
- 在 `shutdown()` 中，关闭 `clients` 中的每个 socket，以便进程实际退出。

### C2——服务器死亡不可见

**问题。** 当服务器退出时，写入 `state/server-stopped` 并移除 `state/server-info`（`server.cjs:312-317`），并且 skill *被告知*检查这些文件（`visual-companion.md:108`）——但这是模型跳过的软指导，浏览器只显示通用的"can't be reached"。用户手动诊断；agent 继续引用死 URL。

**更改（两部分，独立于 C1）：**
- **浏览器面对墓碑。** 在最后提供的 URL 处保留一些内容，显示"this companion expired — ask Claude to restart it"而不是连接错误。需要权衡的选项：`helper.js` 在 socket 在退避后保持断开时渲染横幅（仅在页面加载时有效），vs. 更复杂的方法，保持最小响应者存活以提供墓碑页面。
- **更严格的 skill 检查。** 收紧 `visual-companion.md` / `SKILL.md`，使"在引用 URL 或推送 screen 之前检查 `server-info`/`server-stopped`"成为必需步骤，而非说明。保持轻量——可能是一个 agent 始终运行的单行辅助程序。

---

## D. 功能

### D1——永久退出可视化伴侣（issue #892）

**问题。** 伴侣在每个会话中作为自己的消息提供（`SKILL.md:25,151-152`）。从不想要它的用户每次都要付出往返——以及 HTML 生成——的代价。没有说"永不提供"的方法。

**更改。** 在提供步骤之前，skill 检查用户级别的设置，并在设置 opt-out 时完全跳过提供。

**设计选择未定。** 机制未确定：
- 环境变量（例如 `SUPERPOWERS_VISUAL_COMPANION=off`），skill 被告知读取——最简单，符合 issue 的要求，存在于 `.zshrc` 中。
- Plugin 设置文件（`.claude/superpowers.local.md` frontmatter）——更结构化，可针对项目，但更重且限于项目范围。
- issue 中的可靠性警示：单独的"无伴侣"skill 在触发词上竞争且不可靠——已拒绝。

选择机制，然后是小的 `SKILL.md` 更改加上文档化的开关。

### D2——来自浏览器的自由文本反馈（issue #957）

**问题。** 客户端仅捕获 `[data-choice]` 上的点击（`helper.js:36-62`）。想要注释 mockup（"wrong shade of blue"）的用户必须切换到终端，打破了视觉流程。

**更改。** 添加反馈 `<textarea>`，其提交通过现有的 `window.brainstorm.send` 路径（`helper.js:82-85`）发出 `{"type":"feedback","text":...,"timestamp":...}`。

**交叉——需要服务器更改。** `handleMessage` 仅在 `event.choice` 为真时持久化事件（`server.cjs:243`）。`feedback` 事件没有 `choice`，因此今天它会被记录但**永远不会写入 `state/events`**，agent 不会看到它。持久化条件也必须接受 `feedback` 事件。在 `visual-companion.md`（Browser Events Format，`:247-259`）中记录新的事件形状。决定提交触发器（按钮 vs blur vs 两者）以及 textarea 渲染位置（框架级别 vs 每 screen opt-in）。

### D3——自动打开伴侣 URL（PR #759，issue #755）

**问题。** `start-server.sh` 仅打印 URL；用户手动打开。特别是在 WSL2 中，人们期望浏览器自动打开。

**更改。** 解析 `server-started` JSON 后的尽力打开：Windows/WSL → `rundll32.exe url.dll,FileProtocolHandler <url>`，macOS → `open`，Linux → 仅在设置了 `DISPLAY`/`WAYLAND_DISPLAY` 时使用 `xdg-open`。吞掉失败，从不阻塞启动，继续回显 URL。在 `visual-companion.md` 中记录。（考虑为无头/远程运行添加 opt-out，其中弹出浏览器是错误的——与 D1 的配置机制关联。）

### D4——亮/暗对比辅助类（PR #1683）

**问题。** 内容片段被包裹在 OS 感知的框架中（`frame-template.html`）。在暗模式下，快速 mockup 通常使用白色内联背景同时继承低对比度的框架文本，使卡片/面板难以阅读。

**更改。** 添加 `.light-surface` / `.dark-surface` 辅助类加上常用内联浅色背景的保守回退，并在 `visual-companion.md` 的 CSS 参考中记录它们。`frame-template.html` 中的纯 CSS。

---

## E. 研讨 / 推迟 / 放弃

### E1——每个问题的终端与 HTML 硬关卡（PR #1037）——研讨

软指导已经存在："decide per-question"，在 `SKILL.md:156-161` 和 `visual-companion.md:5-25` 中有浏览器与终端的测试。问题是模型为纯文本内容（A/B 列表、澄清问题）渲染 HTML，浪费 token 和一轮交互。PR #1037 将决策包装在 `<HARD-GATE>` 中。**根据 Jesse 的说法，我们将一起研讨措辞/机制**——这是行为塑造 skill 内容，未在此处进行 spec 设计。

### E2——将会话状态移出工作树（issue #975 / PR #977）——推迟

今天 `--project-dir` 将会话状态写入 `<project>/.superpowers/brainstorm/`（`start-server.sh:80-84`），skill 告诉用户将其加入 gitignore（`visual-companion.md:58`）。需求是 `--state-dir` / `SUPERPOWERS_STATE_DIR` 默认在仓库外部（XDG），保留 `--project-dir` 作为别名。**Jesse 暂时推迟。** 此处记录以免丢失。

### E3——引入 Alpine.js 用于交互式 mockup（PR #1639）——放弃

添加 vendored Alpine 构建以便 mockup 可以交互（标签页、手风琴、表单）而无需手写 JS。**Jesse 放弃**——我们不在伴侣运行时中引入 vendored 的第三方依赖。底层需求（交互式 mockup）不通过此途径追求。

### E4——Shell-lint 警告（PR #1677）——机会主义

`start-server.sh` / `stop-server.sh` 中的 SC2034（等）。琐碎；在 B2/C1/D3 中我们在编辑这些脚本时顺便处理，而非作为单独的更改。

---

## 建议的实施分组

这些聚类成几个连贯的轮次（每个可独立针对 `tests/brainstorm-server/` 测试）：

1. **安全轮次**（进行中，分支 `brainstorm-companion-session-key`）——A1 每会话密钥（取代 A2）+ A3 null 崩溃保护。验证/关闭 A4。*最高优先级。*
2. **生命周期轮次**——C1 + C2 一起（两者都触及 `shutdown()` 和服务器死亡故事）。
3. **健壮性轮次**——B1、B2、B3（独立，小）。
4. **推迟功能轮次**——D1、D2、D4 不是 PR #1720 的一部分。D3 已通过 `--open` 流程交付。

E1 是单独的研讨会议。E2/E3 在本轮范围之外。
