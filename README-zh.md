# Superpowers

Superpowers 是一套完整的软件开发方法论，适用于你的 coding agent。它构建在一组可组合的 skill 之上，配合一些初始指令确保你的 agent 会使用它们。

## 我们在招聘！

我们正在招聘一名全职人员，协助 Superpowers 的社区和代码工作。
你可以在 https://primeradiant.com/jobs/superpowers-community-engineer/ 阅读职位详情。
如果你认识合适的人选，请一定推荐给我们。

## 快速开始

为你的 agent 赋予 Superpowers：[Claude Code](#claude-code)、[Antigravity](#antigravity)、[Codex App](#codex-app)、[Codex CLI](#codex-cli)、[Cursor](#cursor)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[GitHub Copilot CLI](#github-copilot-cli)、[Kimi Code](#kimi-code)、[OpenCode](#opencode)、[Pi](#pi)。

## 工作原理

从你启动 coding agent 的那一刻开始，它看到你在构建某个东西时，*不会*直接开始写代码。相反，它会退一步，先问问你真正想要做什么。

一旦它从对话中提炼出 spec，会以足够短小、便于阅读和消化的分段展示给你。

当你确认设计之后，你的 agent 会制定一个实现计划——这个计划清晰到即使是一个热情但品味不佳、缺乏判断力、没有项目背景、且讨厌测试的初级工程师也能照做。它强调真正的 RED/GREEN TDD、YAGNI（You Aren't Gonna Need It）和 DRY。

接下来，一旦你说"开始"，它会启动 *subagent-driven-development* 流程，让多个 agent 逐个处理每个工程任务，检查和审查它们的工作，持续推进。你的 agent 自主工作连续几个小时而不偏离已制定的计划，这种情况并不少见。

还有更多内容，但以上是系统的核心。由于 skill 会自动触发，你不需要做任何特殊操作——你的 coding agent 自然就拥有了 Superpowers。

## 商业服务

如果你在企业中使用 Superpowers，并且需要商业支持、额外工具或托管式支出管理，请随时联系我们：sales@primeradiant.com。

## 安装

安装方式因 harness 而异。如果你使用多个 harness，请为每个 harness 分别安装 Superpowers。

### Claude Code

Superpowers 可通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取。

#### 官方 Marketplace

- 从 Anthropic 的官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers Marketplace

Superpowers marketplace 提供 Superpowers 及其他一些 Claude Code 相关插件。

- 注册 marketplace：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从该市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

从本仓库将 Superpowers 作为插件安装：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 会在会话启动时运行插件的 session-start hook，因此 Superpowers 从第一条消息起就处于激活状态。使用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 在 Codex 应用中，点击侧边栏的 Plugins。
- 你会在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁的 `+` 并按照提示操作。

### Codex CLI

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或在插件市场中搜索 "superpowers"。

### Factory Droid

- 注册 marketplace：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装 extension：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 后续更新：

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- 注册 marketplace：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Kimi Code

Superpowers 可在 Kimi Code 的插件市场中获取。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 进入 `Marketplace` > `Superpowers` 并安装。

- 或直接从本仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他 harness 中使用 Superpowers，也需要单独安装。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

将 Superpowers 作为 Pi package 从本仓库安装：

```bash
pi install git:github.com/obra/superpowers
```

本地开发时，使用已签出的此仓库作为临时 package 运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi package 会加载 Superpowers 的 skill 和一个小型 extension，该 extension 在 session 启动时以及 compaction 后注入 `using-superpowers` bootstrap。Pi 原生支持 skill，因此不需要兼容的 `Skill` 工具。subagent 和 task-list 工具仍为可选的 Pi companion package。

## 基本工作流

1. **brainstorming** - 在写代码之前激活。通过提问来完善粗糙的想法，探索替代方案，分段展示设计以供验证。保存设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离的 workspace，运行项目初始化，验证干净的测试 baseline。

3. **writing-plans** - 设计批准后激活。将工作拆分为 2-5 分钟的小任务。每个任务都有确切的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划后激活。为每个任务派发全新的 subagent，进行两阶段审查（spec 合规性，然后是代码质量），或以批量方式执行并设置人工 checkpoint。

5. **test-driven-development** - 实现过程中激活。强制执行 RED-GREEN-REFACTOR 循环：编写失败的测试，观察其失败，编写最简代码，观察其通过，然后 commit。会删除在测试之前编写的代码。

6. **requesting-code-review** - 任务之间激活。对照计划进行审查，按严重程度报告问题。关键问题会阻止进度。

7. **finishing-a-development-branch** - 任务完成时激活。验证测试，展示选项（merge/PR/保留/丢弃），清理 worktree。

**agent 在任何任务之前都会检查是否有相关的 skill。** 这是强制性的工作流，而非建议。

## 内部构成

### Skills Library

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含 testing anti-patterns 参考）

**调试**
- **systematic-debugging** - 4 阶段根因分析流程（包含 root-cause-tracing、defense-in-depth、condition-based-waiting 技术）
- **verification-before-completion** - 确保问题真正修复

**协作**
- **brainstorming** - Socratic 式的设计精炼
- **writing-plans** - 详细的实现计划
- **executing-plans** - 带有 checkpoint 的批量执行
- **dispatching-parallel-agents** - 并发的 subagent 工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - Merge/PR 决策工作流
- **subagent-driven-development** - 快速迭代，两阶段审查（spec 合规性，然后代码质量）

**元**
- **writing-skills** - 按照最佳实践创建新 skill（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## 理念

- **Test-Driven Development** - 始终先写测试
- **系统性优于临时性** - 流程优于猜测
- **降低复杂性** - 以简洁为首要目标
- **证据胜于断言** - 在宣布成功之前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

以下是 Superpowers 的一般贡献流程。请注意，我们通常不接受新 skill 的贡献，且对 skill 的任何更新都必须在我们支持的所有 coding agent 上都能正常运行。

1. Fork 本仓库
2. 切换到 'dev' 分支
3. 为你的工作创建一个分支
4. 在创建和测试新 skill 或修改现有 skill 时，遵循 `writing-skills` skill
5. 提交 PR，务必填写 pull request template

Skill 行为测试使用 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 中的 drill eval harness，克隆到 `evals/` 目录下——查看 `evals/README.md` 了解设置方式。Plugin 基础设施测试位于 `tests/` 目录，通过相应的 `run-*.sh` 或 `npm test` 运行。

完整指南请参阅 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新方式因 coding agent 而异，但通常会自动进行。

## 许可证

MIT 许可证——详见 LICENSE 文件

## Visual companion 遥测

由于 skill 和 plugin 不向创建者提供任何反馈，我们无从知晓有多少人在使用 Superpowers。默认情况下，brainstorming 的可选 visual companion 功能中显示的 Prime Radiant 标志从我们的网站加载，其中包含当前使用的 Superpowers 版本。它不包含关于你的项目、prompt 或 coding agent 的任何详细信息。我们看不到你的点击或你正在构建的任何内容。这帮助我们大致了解有多少人在使用 Superpowers 以及他们使用的版本。这是完全可选的。要禁用它，将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设置为任意真值即可。Superpowers 也遵循 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出设置。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz)获取社区支持、提问和分享你用 Superpowers 构建的项目
- **Issues**：https://github.com/obra/superpowers/issues
- **发布通知**：[注册](https://primeradiant.com/superpowers/)以获取新版本通知
