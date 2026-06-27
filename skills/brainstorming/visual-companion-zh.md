# 可视化伴侣指南

基于浏览器的可视化 brainstorm 伴侣，用于展示 mockups、图表和选项。

## 何时使用

按问题而不是按会话决定。标准：**用户通过观看比通过阅读能更好地理解吗？**

**使用浏览器** 当内容本身是视觉性的：

- **UI mockups** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉比较** — 比较两种布局、两种配色方案、两种设计方向
- **设计打磨** — 当问题涉及外观与感觉、间距、视觉层次时
- **空间关系** — 状态机、流程图、实体关系以图表形式呈现

**使用终端** 当内容是文本或表格时：

- **需求和范围问题** — "X 是什么意思？"，"哪些功能在范围内？"
- **概念性 A/B/C 选择** — 在用文字描述的方案之间进行选择
- **权衡列表** — 优缺点、比较表
- **技术决策** — API 设计、数据建模、架构方案选择
- **澄清性问题** — 任何答案是用文字表达，而不是视觉偏好的内容

一个关于 UI 主题的问题不自动成为视觉问题。"你想要哪种向导？"是概念性的——使用终端。"这些向导布局中哪个感觉合适？"是视觉性的——使用浏览器。

## 工作原理

服务器监视目录中的 HTML 文件，并将最新的文件提供给浏览器。你向 `screen_dir` 写入 HTML 内容，用户在浏览器中查看并可点击选择选项。选择内容记录到 `state_dir/events`，你可以在下一轮对话中读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（仅注入辅助脚本）。否则，服务器会自动将你的内容包装在框架模板中——添加标题、CSS 主题、连接状态和所有交互基础设施。**默认编写内容片段。** 仅在需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 在用户批准伴侣后启动。--open 在第一个画面时自动打开浏览器；
# --project-dir 持久化 mockup 并支持同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open`，当你推送第一个画面时浏览器会自动打开——你不需要让用户打开它，但还是要分享 URL 作为备用（无头/远程设置不会自动打开）。

**URL 包含会话密钥（`?key=...`）。** 服务器拒绝任何没有密钥的请求，所以始终给用户 `url` 字段中的**完整** URL——永远不要去掉查询字符串，也永远不要给出裸的 `http://host:port`。该密钥控制 HTTP 和 WebSocket 访问，因此其他浏览器标签页或网络上的其他机器无法读取画面或注入事件。首次加载后，浏览器通过 cookie 记住密钥，因此刷新和 `/files/*` 资源无需重复输入。

**查找连接信息：** 服务器将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动服务器且没有捕获标准输出，读取该文件以获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 下的会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传递，以便 mockup 持久化在 `.superpowers/brainstorm/` 中并能在服务器重启后存活。不传则文件进入 `/tmp` 并被清理。提醒用户如果还没有，将 `.superpowers/` 添加到 `.gitignore`。

**按平台启动服务器：**

**Claude Code：**
```bash
# 默认模式即可——脚本会自动在后台运行服务器。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本会自动检测并切换到前台模式（会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，以便服务器在对话轮次之间存活，然后在下一轮读取 `$STATE_DIR/server-info` 以获取 URL 和端口。

**Codex：**
```bash
# Codex 会回收后台进程。脚本自动检测 CODEX_CI 并
# 切换到前台模式。正常运行——无需额外标志。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Gemini CLI：**
```bash
# 使用 --foreground 并在 shell 工具调用上设置 is_background: true
# 以便进程在轮次之间存活
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**Copilot CLI：**
```bash
# 使用 --foreground 并通过 bash 工具以 mode: "async" 启动服务器
# 以便进程在轮次之间存活。捕获返回的 shellId，
# 以便后续需要时使用 read_bash / stop_bash 与之交互。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话轮次之间在后台持续运行。如果你的环境会回收分离的进程，使用 `--foreground` 并通过平台的背景执行机制启动命令。

如果从你的浏览器无法访问 URL（在远程/容器化设置中常见），绑定一个非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器是否存活**，然后在 `screen_dir` 中写入一个新的 HTML 文件：
   - **必须：在引用 URL 或推送画面之前确认服务器已启动。** 检查 `$STATE_DIR/server-info` 是否存在且 `$STATE_DIR/server-stopped` 不存在。如果已关闭，使用相同的 `--project-dir` 用 `start-server.sh` 重启——它会重用同一端口，因此用户打开的标签页会自动重新连接（服务器关闭时显示"已暂停"覆盖层），你无需发送新 URL。服务器在空闲 4 小时后自动退出（可用 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要重复使用文件名**——每个画面使用新文件
   - 使用你的文件创建工具——**永远不要使用 cat/heredoc**（会在终端中产生噪音）

2. **告诉用户预期内容并结束你的回合：**
   - 提醒他们 URL（每一步都要提醒，不仅是第一次）
   - 给一个简短的文字摘要说明画面上显示的内容（例如，"显示主页的 3 种布局选项"）
   - 让用户在终端中回应："看一看，告诉我你的想法。如果需要，可以点击选择某个选项。"

3. **在你的下一轮**——在用户在终端中回应后：
   - 如果存在，读取 `$STATE_DIR/events`——这包含用户的浏览器交互（点击、选择），以 JSON 行的形式记录
   - 与用户的终端文本合并以获取完整情况
   - 终端消息是主要反馈；`state_dir/events` 提供结构化交互数据

4. **迭代或推进**——如果反馈改变了当前画面，写入新文件（例如，`layout-v2.html`）。只有当当前步骤经过验证后，才进入下一个问题。

5. **返回终端时卸载**——当下一步不需要浏览器时（例如，澄清性问题、权衡讨论），推送一个等待画面以清除过期内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这可以防止用户在对话已经继续时，还在盯着一个已经解决的选择。当下一个视觉问题出现时，像往常一样推送新的内容文件。

6. 重复直到完成。

## 编写内容片段

只需编写页面内部的内容。服务器会自动将其包裹在框架模板中（标题、主题 CSS、连接状态和所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、CSS、`<script>` 标签。服务器会提供所有这些。

## 可用的 CSS 类

框架模板提供以下 CSS 类供你的内容使用：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 在容器上添加 `data-multiselect` 以允许用户选择多个选项。每次点击切换项目的选中样式。

```html
<div class="options" data-multiselect>
  <!-- same option markup — users can select/deselect multiple -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 分割视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### 模拟元素（线框图构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版和章节

- `h2` — 页面标题
- `h3` — 章节标题
- `.subtitle` — 标题下的副文本
- `.section` — 带底部边距的内容块
- `.label` — 小号大写标签文本

## 浏览器事件格式

当用户在浏览器中点击选项时，交互记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新画面时，文件会自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流显示用户的探索路径——他们可能在确定选择之前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击的模式可以揭示值得询问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有与浏览器交互——只使用他们的终端文本。

## 设计技巧

- **按问题缩放保真度** — 布局问题用线框图，打磨问题用精致设计
- **在每个页面解释问题** — "哪种布局感觉更专业？"而不是仅仅"选一个"
- **在推进之前迭代** — 如果反馈改变了当前画面，编写新版本
- **每个画面最多 2-4 个选项**
- **在必要时使用真实内容** — 对于摄影作品集，使用实际图片（Unsplash）。占位符内容会掩盖设计问题。
- **保持 mockup 简单** — 专注于布局和结构，而不是像素完美设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要重用文件名——每个画面必须是新文件
- 迭代时：附加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，mockup 文件会保留在 `.superpowers/brainstorm/` 中供以后参考。只有 `/tmp` 会话会在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
