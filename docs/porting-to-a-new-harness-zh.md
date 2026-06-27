# 将 Superpowers 移植到新 Harness

本指南说明如何为新的 harness（IDE、CLI 或非 Claude Code 的 agent 运行环境）添加支持，使 Superpowers skills 能够像在原生环境中一样自动触发。

本文分为两个层次。**第 1-3 部分**解释系统的工作原理以及如何判断某个 harness 是否可被支持；请先阅读这些部分再进行任何操作。**第 4-8 部分**为 agent（由人类伙伴监督）提供从移植到分发的端到端可执行流程。附录列出了当前的参考集成，供你复制最接近的实现。

不同 harness 的集成机制各不相同，并且会持续变化。本指南有意教授**不变的部分**——无论采用哪种机制都必须成立的条件——并引导你参考实时的参考实现。当本指南与代码不一致时，以代码为准；请修复本指南。

## 开始之前

添加 harness 是本仓库中风险最高的贡献类型。在编写任何代码之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md`——贡献者规则和新 harness 的 PR 要求不可省略。
- 搜索**已开放和已关闭**的 PR，查看是否曾有人尝试过该 harness。如果存在，请在开始之前了解其停滞原因。

---

## 第 1 部分 — Superpowers 在各 harness 中的工作原理

Superpowers 在所有平台上内容相同。每个 harness 变化的是那一层薄薄的交付层——它负责将内容传递给模型，并将其指令翻译为 harness 的原生工具。共三个组件：

1. **Skills（harness 无关）。** `skills/` 中的所有内容都是对所有 harness 共享的权威来源，无需修改。Skills 描述的是**行为**——"调用 skill"、"读取文件"、"派发子 agent"、"创建 todo"——从不指定具体工具名称。这使得同一份 skill 内容可以在 Claude Code、Codex、Gemini、pi 等其他环境中无需修改即可运行。

2. **工具映射（每个 harness 独立）。** 每个 harness 需要将行为词汇翻译为实际工具名称。该翻译存在于 `skills/using-superpowers/references/<harness>-tools.md` 和/或 harness 的 bootstrap 注入器中（见第 5 部分）。例如，"*派发子 agent* → 调用 `task` 并设置 `subagent_type`"。

3. **Bootstrap（每个 harness 独立）。** 在每个会话开始时，完整的 `skills/using-superpowers/SKILL.md` 会被注入到模型的上下文中，包裹在 `<EXTREMELY_IMPORTANT>` 标签内，并附加上工具映射。这个被注入的 skill 教会模型 skills 的存在，以及它必须在行动之前检查是否有相关的 skill。**Bootstrap 是整个集成的核心。** 没有它，skill 文件就是一堆死文件——存在于磁盘上，但永远不会被调用。

### 使其生效的两条规则

**1. Skills 命名行为，而非工具。** 切勿**修改 skill 内容以适应你的 harness。移植时只需添加工具映射引用和 bootstrap 注入器；永远不要进入 `skills/*/SKILL.md` 中替换工具名称。（项目的贡献者指南将 skill 内容视为精心调优的行为塑造代码；为"合规"而改写会被直接拒绝。）

**2. 所有内容都通过 harness 自身的安装机制分发。永远不要编辑用户的文件。** Bootstrap、skills 和工具映射都应作为 harness 安装内容的一部分进行交付——plugin、extension、商城条目、extension 自带的上下文文件。移植**不得**侵入用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、手动编辑的 `~/.bashrc` 等）来注入任何内容。Harness 负责加载什么，你的安装产物就是你能写入的唯一内容。如果安装机制确实无法携带 bootstrap，这是需要明确指出（第 6 部分）的限制——绝不是手动编辑用户配置的借口。（Shape C *并非*例外：Gemini 的上下文文件之所以可行，是因为它随安装的 extension *一起分发*，并由清单的 `contextFileName` 声明——harness 加载的是 extension 自身的文件，而不是你在用户 home 目录下编辑的文件。）

---

## 第 2 部分 — 该 Harness 是否可被支持？

一个 harness 只有在能够满足以下所有条件时才能支持 Superpowers。在编写代码之前请检查这些条件——如果第一个条件不满足，请停止。

### 硬性要求：自动的会话启动注入

Harness 必须允许你在**每个会话启动时自动将文本注入模型的上下文，无需人类伙伴每次手动选择加入。** 这是不可妥协的能力。它可以采用任何形式：

- **hook/事件系统**，在会话启动时运行 shell 命令并读取其 stdout（Claude Code、Codex、Cursor、Copilot CLI），或
- **进程内 plugin/extension**，具有可修改消息数组的会话启动或消息生命周期回调（OpenCode、pi），或
- **指令文件**约定，harness 加载一个由你安装的 extension 分发并声明的上下文文件（例如 Gemini 的 `contextFileName` 指向 extension 自身的 `GEMINI.md`）——而不是你在用户 home 目录下编辑的文件。

如果让 Superpowers 出现在模型面前的唯一方式是让人类伙伴在每个会话中选择加入（粘贴提示、运行命令、启用某种模式），则该 harness **无法**被正确支持。第 3 部分中的验收测试将失败，PR 将被关闭。这是"移植"实际上并非真正移植的最常见原因。

### 其余能力清单

| Capability | Why it's needed | If absent |
|---|---|---|
| **Skill 发现 + 调用** | 模型必须能够按需加载 skill 的完整内容 | 如果没有原生的 skill 工具，官方认可的降级方案是直接 `read` 相关的 `SKILL.md`——见第 5 部分。既没有 skill 工具也不能读取文件的 harness 无法工作。 |
| **文件读取 / 写入 / 编辑** | 几乎所有 skill 都需要操作文件 | 必需。无替代方案。 |
| **运行 shell 命令** | TDD、验证、git 工作流 | 必需。 |
| **子 agent / 任务派发** | `dispatching-parallel-agents`、`subagent-driven-development` | 可降级：如果不可用，这些特定 skill 会指示模型内联完成工作或报告能力缺失——*绝不*虚构 `Task` 调用。某些 harness 将此功能隐藏在配置开关后（例如 Codex 需要启用 multi-agent）。 |
| **Todo / 任务跟踪** | 多个 skill 中的进度跟踪 | 可降级：回退到计划文件或 `TODO.md`。 |
| **网络获取 / 搜索** | 少数几个 skill | 可降级。 |
| **Shell 或多语言脚本执行（Windows）** | 仅适用于 shell-hook 形态，仅当你需要 Windows 支持时 | 见第 7 部分。进程内 plugin harness 完全避免了这个问题。 |

"可降级"意味着：该 skill 已经包含针对缺失工具的备用措辞。你在工具映射中的工作是：当真实工具存在时指向它，当不存在时复用那些备用措辞。

### 你可能根本不需要新建目录

有些"新 harness"实际上只是现有集成使用了不同的安装方式。例如 Factory 的 Droid 通过自身的 `plugin install` 命令使用 Claude Code plugin，不需要在此添加新文件。在开始之前，请检查该 harness 是否可以直接加载现有清单。如果移植只需在 README 中添加一个段落，那也是完全可以接受的结果。

---

## 第 3 部分 — 完成标准

当**所有**以下条件都满足时，移植才算完成：

1. `using-superpowers` bootstrap 在每次会话启动时自动加载，无需每次选择加入。
2. 存在针对该 harness 的工具映射（在 `references/<harness>-tools.md`、bootstrap 内联中，或两者皆有——见第 5 部分）。
3. Skills 确实可以被调用——原生方式，或通过文档化的 read-`SKILL.md` 降级方式——并且模型会遵循它们。
4. **验收测试通过。** 在全新会话中，用户消息：

   > Let's make a react todo list

   会自动触发 `brainstorming` skill，*在编写任何代码之前*。捕获完整的对话记录——PR 需要它。
5. 测试覆盖该集成（第 5 部分）并通过。
6. 真实用户可以通过 harness 自身的机制（而非手动复制文件）安装它，并且版本在 `.version-bump.json` 中记录（如适用，第 6 部分）。注意某些安装程序会在安装时重写或精简清单（有一个会把清单精简为只剩 `{"name": …}`），因此"已安装的文件报告仓库版本"并不总是可行——在源清单中记录版本，不要将重写后的已安装清单视为失败。

在完整验收测试之前进行快速烟雾测试：启动一个会话，要求模型描述它的 superpowers。如果 bootstrap 已注入，模型就知道自己有 superpowers。（OpenCode 的安装文档使用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过不同的机制达到同样的目的——日志搜索而非询问模型；`2>&1` 很重要，因为日志输出到 stderr。找到你的 harness 的等效方法。）

---

## 第 4 部分 — 选择你的集成形态

共有三种结构形态，以*bootstrap 如何呈现在模型面前*来区分。选择与你的 harness 暴露的能力相匹配的形态，然后复制相应的参考实现。形态决定了第 5 部分中的几乎所有内容——下面的步骤将根据形态分支。

### 如何判断你属于哪种形态

在确定路径之前，先了解 harness 的*实际*机制——不要假设它有完善的文档，也不要假设它与其 fork 来源的 harness 行为相同。

**寻找表面层：**

- **搜索网络查找 harness 的文档**（extension / plugin / hook / skill / MCP / "context file" / "rules file"）。厂商工具变化很快；搜索比依赖训练数据更可靠。
- **找到并阅读一个现有的第三方 extension/plugin 该 harness。** 真实的工作示例胜过文档——它能展示清单形态、安装命令以及 harness 实际加载了哪些组件。
- 检查 harness 在启动时加载了什么：一个设置文件？一个 extensions 目录？一个项目级或全局的指令文件（`AGENTS.md`、`<NAME>.md`）？

**如果文档不足，通过实证进行逆向工程**（真实的移植者曾做过以下所有事情）：

- 使用 `strings` 检查二进制文件 / 在安装树中搜索 hook 事件名称、配置路径以及它读取的指令文件。
- **让正在运行的模型枚举自己的工具名称**——例如，"列出你可以调用的每个工具的确切机器名称，每行一个"。这是获取工具名称的权威方式，无需自己编造（见步骤 4）。
- 用**唯一标记测试**证明每个假设：通过你认为可行的机制注入一个无意义的标记，启动全新会话，确认该标记确实到达了模型。

**一个 fork 不会继承其父级的行为。** 从其他项目派生出来的 harness（例如基于 Gemini 的 CLI）可能暴露父级的清单字段和 `@`-include 语法，但*却不会以相同的方式处理它们*。请用标记进行验证；永远不要假设父级的方案可以直接套用。

然后确定形态：

- 会话启动时运行 shell 命令并读取其 stdout → **形状 A**。
- 具有生命周期回调的 plugin/extension 模块，可以在其中运行代码 → **形状 B**。
- 仅有始终加载的指令文件，没有 hook 也没有代码 plugin → **形状 C**。

**形状可以组合——它们并非互斥。** *Skill 发现*机制和 *bootstrap* 机制不必是同一形态——但**两者都必须通过安装机制来交付**（规则 2）。分别决定两个问题：*skills 从哪里被发现？* 以及 *bootstrap 如何每次会话到达模型？* 一个 harness 可能通过 plugin 安装 skills，却需要通过另一种安装交付的方式（extension 声明的上下文文件，或——见下文——通过 harness 在会话启动时展示已安装的 `using-superpowers` skill 自身的描述）来传递 bootstrap。如果有多个安装机制表面自动注入，选择最可靠的那个。你**不得**通过编辑用户的全局配置来弥补缺口。

### 形状 A — Shell-hook

Harness 具有 hook 系统，可在会话启动时运行 shell 命令并读取其 stdout。配置的命令运行 `run-hook.cmd`（一个多语言包装器），该包装器找到 bash 并分派指定的脚本；脚本（`hooks/session-start`，或 harness 特定的变体如 `hooks/session-start-codex`）读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套结构因 harness 而异**。

- 参考：`hooks/session-start`（及 `hooks/session-start-codex`）、`hooks/run-hook.cmd` 以及每个 harness 的 hook 配置 `hooks/hooks.json`（Claude Code）、`hooks/hooks-codex.json`（Codex）、`hooks/hooks-cursor.json`（Cursor）。
- 清单：`.codex-plugin/plugin.json`、`.cursor-plugin/plugin.json` 将 harness 指向 `./skills/` 和对应的 `hooks-*.json`。（Claude Code 的 `.claude-plugin/plugin.json` 不设置这两个字段——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。）

> **Hook *系统* 并非会话启动 *事件*。** 一个 harness 可以有 `hooks.json` 机制——甚至二进制文件中包含字面字符串 `SessionStart`——却可能没有在会话启动时触发并能注入上下文的 hook 事件。（一个真实的 harness 只暴露了工具前/后和停止事件；其中的 `SessionStart` 字符串只是遥测数据。）在确认你需要的*特定事件*存在并能够写入模型上下文之前，不要贸然选择形状 A。如果不行，bootstrap 应归入指令文件（形状 C）。

### 形状 B — 进程内 plugin/extension

Harness 加载一个暴露生命周期回调的 JS/TS 模块。你通过 harness 的 API 注册 skills 目录，并通过在代码中修改消息数组来注入 bootstrap。

- 参考：`.opencode/plugins/superpowers.js`（JavaScript）和 `.pi/extensions/superpowers.ts`（TypeScript）。对于**没有原生 skill 工具**的 harness，pi 是最接近的参考。

### 形状 C — 指令文件

Harness 既没有 shell hook 也没有代码 plugin——它的会话启动表面是一个由你安装的 extension 分发并由清单声明的上下文文件（例如 Gemini 的 `contextFileName` → extension 自身的 `GEMINI.md`）。你不能运行代码或修改消息；extension 的上下文文件指向 bootstrap。没有注入器来组装字符串或去除 frontmatter——harness 直接加载所引用的内容。**这之所以可行，仅仅因为该文件是已安装 extension 的一部分**——绝不能将"编辑用户的全局 `GEMINI.md`/`AGENTS.md`"作为分发你自己的文件的替代方案（规则 2）。

- 参考：`gemini-extension.json`（清单，包含 `contextFileName`）、`GEMINI.md`（两个 `@`-include——bootstrap skill 和工具映射引用）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@`-include 是 Gemini 的功能。如果你的 harness 加载指令文件但没有 include 语法，你必须将 bootstrap 内容内联到文件中。
- **不要相信 `@`-include 一定会被展开——请证明它。** 基于 Gemini *衍生*的 harness 可能接受 `@./path` 语法，但将其视为*模型可以选择读取的提示*（它会发出文件读取工具调用）而非有保证的内联展开。这就是 bootstrap 可靠地存在于每个会话中与模型可能读取它之间的区别。运行唯一标记测试：如果标记在*没有*工具调用的情况下不在上下文中，**请内联内容**而非使用 `@`-include。

### 路由表

| If the harness… | Use shape | Copy from |
|---|---|---|
| 在会话启动时运行 shell 命令并读取其 stdout | A（shell-hook） | Codex（`hooks/session-start-codex` + `hooks/hooks-codex.json` + `.codex-plugin/`） |
| 是具有会话/消息生命周期回调的 JS/TS plugin 宿主 | B（进程内） | OpenCode（`.opencode/`）——如果没有原生 skill 工具，则参考 pi（`.pi/`） |
| 分发一个始终加载的、由 extension 声明的上下文文件 | C（指令文件） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`） |
| 具有 plugin 安装命令和安装程序保留的清单 `contextFileName`（或等效字段） | C 通过 plugin 安装程序 | Antigravity（`.antigravity-plugin/`——`agy plugin install` 分发一个生成的上下文文件；验证安装程序是否保留它——第 6 部分） |

大多数真实 harness 能干净地匹配其中一行；最后一行是混合情况（规则 2 仍然适用——bootstrap 通过安装机制交付，绝不通过编辑用户配置）。

---

## 第 5 部分 — 移植流程

### 步骤 1 — 研究最接近的参考实现

打开第 4 部分中针对你的形态命名的文件，从头到尾阅读。下面的模式是总结；代码才是规范。

### 步骤 2 — 创建清单 / 入口点

创建 harness 用于识别 plugin 的文件。在精神上匹配现有的文件：

- **形状 A：** `*-plugin/plugin.json`（参考 `.codex-plugin/plugin.json`），包含 `name`、`version`、`description`、author/license/keywords、`"skills": "./skills/"` 和 `"hooks": "./hooks/hooks-<harness>.json"`。再加上 `hooks-<harness>.json` 本身，注册一个会话启动 hook，其命令调用 `run-hook.cmd`。
- **形状 B：** harness 加载的模块（例如 `.<harness>/plugins/*.js`）以及被发现所需的任何包元数据。已提交的包元数据是**仓库根目录的 `package.json`**：`main` 指向 OpenCode plugin，`pi` 字段（`pi.extensions`、`pi.skills`）加上 `pi-package` 关键字声明了 pi extension。每个 harness 的本地清单和 lockfile 应排除在 git 之外——`.opencode/.gitignore` 排除了 `node_modules`、`package.json` 和 lockfile。对你的 harness 的*本地*安装产物也做同样处理，以免污染仓库——但永远不要忽略仓库根目录的 `package.json`，它是被跟踪的权威来源。
  - **构建/依赖检查。** 确定你的 harness 如何加载模块：是直接运行源码（pi 的 `.ts` 直接从 `package.json` 中引用；OpenCode 提供纯 `.js`），还是需要转译/构建步骤？Superpowers 是零运行时依赖的。pi 的 `import type { ExtensionAPI }` 之所以能工作，是因为 harness 直接运行 `.ts`，在加载时提供该类型，并且仓库在 CI 中从不检查该文件的类型——该导入甚至未声明为依赖。如果你的 *harness* 确实进行类型检查或打包 plugin，那就会出问题：未声明的类型导入会失败，而 PR 规则只为新 harness 划出了*运行时*依赖的例外，不包括 dev/type 包。如果遇到此情况，请向维护者确认方案，而不是悄悄添加依赖。构建产物应排除在 git 之外，并记录构建命令。
- **形状 C（指令文件）：** 一个小型清单（参考 `gemini-extension.json`：`name`、`description`、`version`、`contextFileName`）加上上下文文件本身（`GEMINI.md` 仅包含两个 `@`-include：bootstrap skill 和工具映射引用）。Gemini 清单没有 `skills` 字段——Gemini 会自动发现已安装 extension 中捆绑的 `skills/` 目录。如果你的 harness 有原生 skill 工具但没有注册目录的清单字段，你必须找到其发现约定（阅读其 extension 文档），然后通过实证验证：连接后，询问模型列出其可用的 skills——如果捆绑的 skills 没有出现，则发现机制尚未生效。

### 步骤 3 — 连接 Bootstrap 注入

这是移植的核心。共同目标：在会话启动时，将 `using-superpowers` skill 内容（包裹在 `<EXTREMELY_IMPORTANT>` 标签中）加上 harness 的工具映射呈现在模型面前，并附带一条说明该 skill 已经激活，模型无需再次加载。*如何*实现——以及你组装什么与 harness 原生加载什么——完全取决于你的形态。**不要**将一种形态的方案套用到另一种形态上。

**形状 A —— 脚本读取 `SKILL.md` 并打印 harness 的 JSON。** 被分派的脚本（`hooks/session-start`）`cat` 整个 `SKILL.md`（包括 frontmatter——这没问题；它被原样输出），用"你有 superpowers……对于其他所有 skills，请使用 Skill 工具"的前言包裹它，进行转义，然后打印 harness 的 JSON 形状。形状 A 的工具映射**不**在此内联——它存在于 `references/<harness>-tools.md` 中（步骤 4）。确保 JSON 输出形状完全正确。`hooks/session-start` 从环境变量检测 harness 并打印*三种形态之一*：

- Cursor（设置了 `CURSOR_PLUGIN_ROOT`）：`{ "additional_context": "…" }`
- Claude Code（设置了 `CLAUDE_PLUGIN_ROOT`，未设置 `COPILOT_CLI`）：`{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }`
- Copilot CLI / SDK 标准（其他情况）：`{ "additionalContext": "…" }`

这是一个陷阱。输出错误的字段或多余的字段，意味着 bootstrap 要么永远不会注入，要么会注入两次（Claude Code 同时读取 `additional_context` 和 `hookSpecificOutput` 而不去重，因此同时输出会导致双重注入）。找到你的 harness 期望的确切字段、嵌套结构和事件匹配器值。然后决定：向 `hooks/session-start` 添加第四个分支，或者——如果 harness 需要不同的 bootstrap 消息或环境变量约定——添加一个专用的 `hooks/session-start-<harness>` 脚本，就像 Codex 那样。如果你添加了分支，而你的 harness *也*设置了先前分支所依赖的环境变量（某些 harness 也会设置 `CLAUDE_PLUGIN_ROOT`），请将你的分支排在可能覆盖它的分支之前。匹配 harness 自身的事件匹配器字符串（Claude Code 使用 `startup|clear|compact`，Codex 使用 `startup|resume|clear`，Cursor 使用 `sessionStart`）；错误的匹配器意味着 hook 静默地永远不会触发。

**Hook 配置模式本身也因 harness 而异**——不要假设 Claude/Codex 的形态是通用的。比较 `hooks/hooks.json`、`hooks/hooks-codex.json` 和 `hooks/hooks-cursor.json`：Cursor 使用 `"version": 1`、小写的 `sessionStart` 键、相对路径的 `./hooks/run-hook.cmd` 命令，并省略了其他文件使用的 `matcher`/`type`/`async` 字段。将你的 `hooks-<harness>.json` 匹配到最接近的现有文件，而不是某个单一的规范模板。

Hook **命令字符串引用 harness 提供的 plugin-root 变量**，其名称因 harness 而异：`hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}`，`hooks-codex.json` 使用 `${PLUGIN_ROOT}`，Cursor 使用相对路径。使用你的 harness 导出的任何变量。（`session-start` 脚本通过 `dirname` 自行推导根目录，因此脚本体不依赖于此——但清单中的命令依赖。）

**发现 harness 的约定。** 以上三个事实——环境变量、JSON 字段/嵌套、匹配器字符串——是 harness 的约定，而非 Superpowers 的，因此你需要找到它们。阅读 harness 的 hook 文档，或通过实证发现：注册一个临时会话启动 hook，转储其环境并发出标记，然后观察哪个环境变量标识了该 harness，以及 harness 是否/如何获取你的 stdout。在编写真正的分支之前先确定这些。

**形状 B —— 在代码中组装字符串，然后作为用户消息注入。** 在这里你自行构建 bootstrap：读取 `SKILL.md`，去除其 YAML frontmatter，然后组装 `<EXTREMELY_IMPORTANT>` + 简短前言（说明该 skill 已加载，不应再次调用）+ 去除 frontmatter 后的正文 + 内联工具映射 + `</EXTREMELY_IMPORTANT>`。参考实现之间存在一个微妙差异：OpenCode 的前言说"不要使用 skill 工具……"（假定存在 `skill` 工具），而 pi 的只说"不要尝试再次加载 using-superpowers"。如果你的 harness 没有 skill 工具，请使用 pi 的措辞，而不是 OpenCode 的。

将结果作为**用户角色的消息而非系统消息**注入——系统消息在每次轮次中都会增加 token 负担（#750），且多个系统消息会破坏某些模型（#894）。你需要复制的三件事：

- **去重守卫。** 生命周期回调可能重复触发（OpenCode 的 transform 在*每次 agent 步骤*运行；pi 的 `context` 每轮触发）。在注入之前，检查 bootstrap 标记是否已存在，如果存在则跳过。（参考实现选择了不同的标记——pi 使用自定义字符串，OpenCode 使用 `EXTREMELY_IMPORTANT` 标签；匹配标签更健壮，因为它不需要 harness 特定的常量。）在模块级别缓存 bootstrap 内容，避免每次调用都重新读取和解析 `SKILL.md`（#1202）。
- **压缩（compaction）。** 如果 harness 压缩/总结历史记录，在压缩后重新注入。pi 在 `session_start` 和 `session_compact` 时设置 `injectBootstrap` 标志，在 `agent_end` 时清除，并在任何前置的压缩总结消息*之后*插入消息。OpenCode 依赖其每步重新注入加上去重守卫。
- **消息对象形状因 harness 而异——请发现你的，不要复制字面量。** 两个参考实现使用*不兼容*的形状：pi 构建 `{ role, content: [{ type, text }], timestamp }`；OpenCode 操作 `message.info.role` 和 `message.parts[]`。从 harness 的 API 中找到它的消息形状；逐字复制参考实现的对象字面量会静默失败。

**形状 C —— 将 extension 的上下文文件指向 bootstrap；无需组装任何内容。** 没有注入器，因此你*不*需要去除 frontmatter 或构建包裹字符串。你的 extension 分发的上下文文件（由清单声明——*不是*用户自己的全局文件）引入两部分内容：`using-superpowers` skill 和 harness 的工具映射引用。`GEMINI.md` 通过两个 `@`-include 实现（`@./skills/using-superpowers/SKILL.md` 和 `@./skills/using-superpowers/references/<harness>-tools.md`）；harness 直接加载它们，包括 frontmatter 和所有内容，而 `SKILL.md` 内部已经包含了自己的 `<EXTREMELY-IMPORTANT>` 块。如果你的 harness 没有 include 语法，将内容内联到指令文件中。Gemini **不**附带"已加载，不要重新调用"的前言——对于使用 `@`-include 的 harness，该内容就是活跃的指令集，而非模型会重新加载的 skill。如果你发现你的 harness 确实尝试重新调用，请将该说明作为字面行添加到指令文件中（你没有代码可以以其他方式添加）。

### 步骤 4 — 编写工具映射

将行为词汇翻译为 harness 的实际工具。覆盖以下每个行为（仅省略确实不适用的）：

- 读取文件
- 创建 / 编辑 / 删除文件（一个 `apply_patch` 风格的工具，还是分别的 write/edit？）
- 运行 shell 命令
- 搜索文件内容 / 按名称查找文件（grep, glob）
- 获取 URL / 网络搜索
- **派发子 agent**，包括如何传递 agent 类型——以及启用它所需的任何配置开关
- **创建 / 更新 todo**（将较旧的 `TodoWrite` 引用视为此行为）
- **调用 skill**——见步骤 5

**从 harness 获取真实的工具名称；永远不要自己编造。** 如果文档没有列出，权威来源是 harness 本身：在活动的会话中，要求模型"列出你可以调用的每个工具的确切机器名称，每行一个"，并使用它报告的内容。

**Harness 如何找到 `skills/` 目录本身也是每个 harness 各异的**——请确认，不要假设。可能的方式：清单中的 `skills` 路径字段（Codex 的 `"skills": "./skills/"`）；harness 自动扫描的*同位置* `skills/`（此时路径字段被**忽略**——一个真实 harness 只扫描 `plugin.json` 旁边的 `skills/`）；API/注册调用（OpenCode、pi）；或者你创建一个安装目录，其中包含清单和指向仓库 `skills/` 的**符号链接**，然后将安装程序指向该暂存目录（确认安装程序*解引用*符号链接并复制真实文件——在依赖此方式前，用 `agy plugin validate`/`install` 或等效方式确认）。`skills` 路径字段并*不可移植*。

映射的位置取决于形态：

- **形状 A：** 放在 `skills/using-superpowers/references/<harness>-tools.md`。Agent 通过 bootstrap 访问它——`SKILL.md` 的"Platform Adaptation"章节链接了各 harness 的参考文件。（形状 A harness 没有指令文件；映射*不*内联到 hook 输出中。）
- **形状 B：** 映射通常内联在你注入的 bootstrap 字符串中（参见 `superpowers.js` 中的 `toolMapping` 常量）。pi 将其放在*两个*地方——`piToolMapping()` 内联**和** `references/pi-tools.md`。如果你在两个地方维护，请同时更新，否则移植只完成了一半。
- **形状 C：** 放在 `references/<harness>-tools.md` 中，并拉入始终加载的指令文件（例如 `GEMINI.md` `@`-include `gemini-tools.md`）。

你还可以在 `SKILL.md` 的"Platform Adaptation"章节中添加一行指向你的 harness 的指针，以便阅读 bootstrap 的 agent 知道其映射在哪。这是移植可能对 `SKILL.md` 做的唯一编辑——且仅因为该章节是一个指针列表，而非行为塑造内容。这不违反"不修改 skill 正文"的规则（第 1 部分）；不要触碰任何 skill 中的其他内容。（该列表是一个方便指针，而非详尽注册表——并非所有 harness 都列在其中。）

### 步骤 5 — 处理没有原生 skill 工具的 Harness

`using-superpowers/SKILL.md` 告诉模型*永远不要用文件工具手动读取 skill 文件——始终使用平台的 skill 加载机制。* 其要点是"不要绕过机制"，而不是"永远不要使用文件读取"。什么是"你平台的机制"取决于 harness——对于没有 skill 工具的 harness，有文档记录的机制*就是*读取 `SKILL.md`。因此在那种情况下读取它，是遵守规则而非破坏规则。区分三种情况：

1. **原生 `Skill` 风格的工具**（Claude Code、Copilot CLI、Gemini 的 `activate_skill`）：将映射指向该工具。
2. **原生 skill *发现*但没有 `Skill` 工具**（pi、Antigravity）：harness 可以找到并列出 skills，但模型无法调用工具来加载一个。将 skills 安装到 harness 扫描的位置（pi 通过 `resources_discover` → `skillPaths` 注册；OpenCode 通过其 `config` hook；`agy plugin install` 复制它们），并告诉模型当某个 skill 适用时，通过**使用文件读取工具读取其 `SKILD.md`** 来加载该 skill——这是这里经过认可的方法，正如 `references/pi-tools.md` 所述。

   **对于 bootstrap 本身，优先使用声明的上下文文件（第 6 部分）。** 如果 harness 具有 `contextFileName` 风格的清单字段——就像 Antigravity 那样——通过安装程序分发生成的上下文文件：它保证被加载，并同时携带 `using-superpowers` 内容和工具映射。这是更强、更优的路径。

   **降级方案——暴露的技能索引。** 如果没有上下文文件字段，但 harness 在会话启动时暴露每个已安装 skill 的名称 + 描述，你*既*不需要构建索引也不需要运行时列出指令——harness 本身就是索引，而 `using-superpowers` 自身暴露的描述可以触发模型加载它。这比声明的上下文文件更弱；与上下文文件 / hook / 进程内注入器相比，它**不**提供以下两件事——请确保两者都考虑：
   - **它引导的是*触发*，而非*工具映射*。** 注入器会在每次会话时将 `<harness>-tools.md` 与 `using-superpowers` 一并前置。而在此处，没有内容注入映射——模型只看到 skill *描述*，并且必须*读取*你的 `references/<harness>-tools.md` 当它需要工具名称时。这样做是可行的，因为 skills 命名的是行为（模型在行动时读取映射），但它比注入更弱。确保映射可以从模型加载的内容中访问——例如在 `SKILL.md` 的 Platform Adaptation 章节中链接，并与 skills 一起安装——而不仅仅是放在仓库中。
   - **没有结构性的触发保证。** 没有 `<EXTREMELY_IMPORTANT>` 包裹，没有去重，没有压缩后重新注入——触发取决于模型是否选择对索引中看到的描述采取行动。这正是验收测试在此处必不可少的原因：它是*唯一*的保证，因此请在你用户实际会使用的模型上运行它，而不仅仅是最强的那个。
3. **没有任何 skill 系统：** 没有需要注册的内容，*唯一*的机制是模型按需读取 `SKILL.md`。但模型无法读取它找不到的内容：`using-superpowers/SKILL.md` **不**枚举可用的 skills，因此仅凭自身，模型不会知道哪些 skills 存在或其触发条件。你必须提供一个发现路径。两个选项，它们在持久性方面不同：(a) 生成一个 skill 索引（每个 `skills/*/SKILL.md` 的 `name` + `description` frontmatter）并将其放置在 `<EXTREMELY_IMPORTANT>` 包裹*内部*，与工具映射一起（上述形状 B 方案），使其被去重守卫覆盖——但构建时的索引会随着 skills 的增加而过时；或 (b) 指示模型在运行时列出 `skills/*/SKILL.md` 并读取其 frontmatter 以找到匹配项——较慢但永不过时。除非有理由不这样做，否则优先选择 (b)。如果没有这两者中的任何一个，无 skill 系统的移植会加载 bootstrap，但静默地永远不会触发任何其他 skill。

在情况 2 和 3 中，在你的工具映射中明确说明读取 `SKILL.md` 是被认可的路径，这样模型就不会认为它违反了"永远不要读取 skill 文件"的规则。不要在没有任何 skill 系统的 harness 中寻找 `skillPaths` 风格的注册 API——情况 3 压根没有。

### 步骤 6 — 添加测试

匹配现有的每个 harness 的测试风格：

- **形状 A：** 断言 hook 的 stdout 具有你的 harness 所消费的确切 JSON 形状，并且包含 bootstrap。参见 `tests/hooks/test-session-start.sh`，它验证了每个 harness 的输出形状。
- **形状 B：** 单元测试，模拟 harness 的 plugin API，并断言生命周期处理器已注册、bootstrap 只注入一次、去重守卫正常工作，以及（如果相关）压缩后重新注入正常工作。参见 `tests/pi/test-pi-extension.mjs`。添加一个 `tests/opencode/` 风格的独立安装集成检查。
- 如果 bootstrap 被缓存，测试文件缺失时缓存的行为（见 OpenCode 的缓存测试）。

这些自动化测试覆盖了连接部分；步骤 7 中的实时 tmux 运行才能证明集成确实触发了 skills。

### 步骤 7 — 本地安装，然后驱动真实实例进行验证

你无法通过阅读代码来确认移植是否有效。你必须使用加载了进行中移植的 harness 运行，并观察真实会话——这也是生成 PR 所需记录的方式。

**本地安装。** 将 harness 的*本地*实例指向你的工作树，而非已发布的构建：

- **形状 A / C：** 从本仓库的本地路径安装 plugin/extension（或将其目录符号链接到 harness 查找的位置）。在 harness 的文档中找到"从本地目录/git 检出安装"的路径。
- **形状 B：** 注册本地模块——例如 `opencode.json` 的 `plugin` 条目指向本地路径，或 pi 从仓库解析 `package.json` 字段。

每次修改后重新安装并重启 harness，因为 bootstrap 在启动时加载。

**使用 tmux 驱动。** 大多数 harness 是交互式 REPL/TUI，无法通过管道 stdin 驱动，因此请在分离的 tmux 会话中运行 harness，并使用 `send-keys` / `capture-pane` 控制。某些 harness 可能支持非交互式的"运行单个提示"模式（例如 `opencode run "..."`）——可以尝试用于快速烟雾测试，但**不要依赖它**：这些模式通常不稳定、需要认证或信任授权（一个真实 harness 的 `--print` 模式挂起并超时，无任何输出）。准备好*所有*操作，包括烟雾测试，都通过 tmux 进行。

**先清除门禁，否则 tmux 会静默挂起。** 许多 harness 在首次运行时会阻塞于入门引导、"你信任此文件夹吗？"提示、沙盒模式或权限门禁——而分离的 tmux 会话会在等待时静默挂起，没有任何错误。在运行之前，预信任你的临时目录（在 harness 的设置/配置中），或准备通过 `send-keys` 回答这些提示，并在首次 `sleep` 中考虑 harness 的启动时间。

```bash
# 1. 在分离模式中启动 harness，位于临时项目目录中
mkdir -p /tmp/port-smoke
tmux new-session -d -s port-test -c /tmp/port-smoke '<harness-launch-command>'

# 2. 让它初始化——真正的 TUI 启动时间比你想象的要长（含模型握手 10 秒以上）；调整此值。
#    然后，在输入提示之前，捕获并清除任何阻塞性模态对话框：首次运行的引导和"信任此文件夹？"
#    是模态的，因此在它们期间发送的按键会选择菜单项，而不是输入你的提示。
sleep 12
tmux capture-pane -t port-test -p          # 引导/信任提示？先通过 send-keys 回答
# （例如 tmux send-keys -t port-test Enter   # 接受信任提示——在假设之前先检查）

# 3. 烟雾测试：模型知道它有 superpowers 吗？
#    将文本和 Enter 作为 SEPARATE send-keys 发送，中间隔一段时间——
#    一起发送会在某些 TUI 上产生竞态（Enter 在文本到达之前就被接收）。
tmux send-keys -t port-test 'What are your superpowers?'; sleep 0.4; tmux send-keys -t port-test Enter
sleep 5
tmux capture-pane -t port-test -p          # 回复应显示它知道自己的 skills

# 4. 验收测试：确切的提示（注意转义的单引号），全新会话
tmux send-keys -t port-test 'Let'\''s make a react todo list'; sleep 0.4; tmux send-keys -t port-test Enter
# 轮询直到该轮次结束——每几秒重新捕获一次，不要只捕获一次
sleep 8
tmux capture-pane -t port-test -p          # 通过 = brainstorming 在编写任何代码之前触发

# 5. 保存 PR 所需的记录，然后清理
tmux capture-pane -t port-test -p > /tmp/port-smoke/transcript.txt
tmux kill-session -t port-test
```

需要留意的 tmux 陷阱：启动后等待再进行首次捕获；将提示文本和 `Enter` 作为*分开*的 `send-keys` 调用发送，中间用较短的 `sleep` 分隔（一起发送会在某些 TUI 上产生竞态），`Enter` 是键名而非 `\n`；agent 的轮次需要时间，因此请在循环中**轮询 `capture-pane`** 而非仅捕获一次；`capture-pane` 只显示可见窗格，因此对于较长的对话，请使用 harness 自身的记录/日志文件作为真实记录；完成后始终 `kill-session`。

如果烟雾测试显示模型*不*知道它有 superpowers，则 bootstrap 未加载——在进行验收测试之前先修复此问题。

---

## 第 6 部分 — 分发与发布

本仓库中一个可用的集成在真实用户可以安装之前是没有意义的。不同 harness 生态的分发方式各不相同——请找到你的：

| Channel | Example | What you do |
|---|---|---|
| 原生 plugin 商城 | Claude Code | 在 `.claude-plugin/marketplace.json` 中注册；用户通过 `/plugin install` 安装。外部的 `superpowers-marketplace` 仓库是用户安装的权威来源——参见 `CLAUDE.md` 中的发布步骤。 |
| 外部商城 fork，通过脚本同步 | Codex | `scripts/sync-to-codex-plugin.sh` 将被跟踪的 plugin 文件 rsync 到单独的 fork 仓库并打开 PR。阅读其 include/exclude 列表，以便分发正确的目录树（它有意排除了仓库内部目录和其他 harness 的 dotdir）。 |
| Git-URL extension 安装 | Gemini、Kimi Code、OpenCode | 用户从 git URL 安装（`gemini extensions install …`；Kimi Code 的 `/plugins install …`；`opencode.json` 的 `plugin` 数组条目）。记录确切的命令。 |
| 包清单字段 | pi | 通过仓库根目录 `package.json` 中的字段声明；用户通过 harness 的包命令安装。 |
| 本地安装程序（plugin install） | Antigravity（`agy`） | 一个小型 `install.sh`，运行 harness 自身的 `agy plugin install` 命令，指向包含清单、skills 和生成的 `contextFileName` 上下文文件（bootstrap）的暂存目录。所有内容都通过安装机制交付——*不*通过编辑用户配置（见下文）。 |

然后：

- **Plugin 安装程序可能静默地剥离*未声明*的文件——因此请使 bootstrap 成为安装程序*可识别*的文件，永远不要编辑用户配置。** `plugin install` 通常只复制它知道的组件（skills/agents/commands/mcp/hooks/context），并丢弃其他内容，因此清单未声明的上下文文件会从安装中消失。解决方法是**不要**放弃并写入用户配置（**规则 2**）——而是将 bootstrap 声明为可识别的组件。按升级顺序：
  - **分发清单声明的上下文文件。** 如果 harness 有 `contextFileName` 风格的字段（它会在每次会话中加载的 extension 声明文件），这就是最强的干净 bootstrap：声明它，安装程序会保留它*并且* harness 会加载它。在安装时从实时的 `using-superpowers/SKILL.md` + 工具映射（包裹在 `<EXTREMELY_IMPORTANT>` 中）生成它，这样已安装的 bootstrap 永远不会漂移。这就是 `.antigravity-plugin/install.sh` 所做的——`agy plugin install` 报告 `✔ context : ANTIGRAVITY.md`，全新会话读取 `using-superpowers` 的 SKILL.md，加载 `brainstorming`，并在任何代码之前进入 brainstorming 流程。**用标记验证**安装程序保留了该文件且 harness 加载了它：一位移植者错误地认为它不行，因为他们分发了文件*而没*声明 `contextFileName`，结果文件被当作无法识别的内容剥离了。
  - **否则，依赖已安装的 `using-superpowers` skill 本身。** 如果 harness 在会话启动时暴露每个已安装 skill 的名称 + 描述，`using-superpowers` 的描述（"Use when starting any conversation…"）可以提示模型加载它——安装 skill *就是* bootstrap。更弱（没有保证的包裹；它承载触发但不承载工具映射——见步骤 5），因此当可用时优先使用声明的上下文文件。
  - 如果两者都不行，该 harness 尚不能干净地被支持——**请说明**并提出来，而不是手动编辑用户配置。

- **编写安装文档。** 一个 `docs/README.<harness>.md` 和/或 `.<harness>/INSTALL.md`（参见 `docs/README.opencode.md` 和 `.opencode/INSTALL.md`），再加上顶层 `README.md` 中的安装章节。唯一受支持的安装操作是**运行 harness 自身的安装命令**（`agy plugin install`、`gemini extensions install`、`/plugin install` 等）。手动复制 skill 文件和编辑用户的全局/个人配置都是*禁止的*（规则 2 / PR 规则）。如果 harness 压根没有安装命令——其唯一表面是用户拥有的配置文件——那么它未能满足"通过安装机制交付"的规则，你应该提出来，而不是发布一个编辑用户文件的安装程序。
- **注册版本。** 如果你的 harness 引入了*新的*带版本清单，将其路径和版本字段添加到 `.version-bump.json`，使 `scripts/bump-version.sh` 保持同步（阅读该文件以查看当前跟踪的内容）。未在此注册的新清单会分发过时的版本。如果你的 harness 使用的是已跟踪的文件——pi 在仓库根目录的 `package.json` 中声明自身，该文件已列出——则无需添加新内容。
- **如果没有现有渠道匹配，你正在建立一个新的渠道。** 以上四行可能都不匹配你的 harness。如果它需要 Codex 风格的外部 fork 同步，`scripts/sync-to-codex-plugin.sh` 是模板（注意其固定的 include/exclude 列表和 PR 自动化）。每当你添加新的每个 harness 目录时，将其添加到*其他* harness 的同步排除列表中（例如 `sync-to-codex-plugin.sh` 中的 EXCLUDES 列表），以使你的 dotdir 不会泄漏到它们的分发中。

---

## 第 7 部分 — 跨平台 / Windows

仅与 shell-hook 形态相关。`hooks/run-hook.cmd` 是一个多语言文件：同一个文件既可作为 Windows batch 脚本有效，也可作为 Unix shell 脚本有效。在 Windows 上，`cmd.exe` 运行 batch 部分，该部分定位 `bash`（先是 Git for Windows，然后是 PATH 上的 `bash`）并运行指定的 hook 脚本；如果未找到 bash，它会干净地退出，因此 harness 仍然可以工作，只是没有注入。在 Unix 上，开头的 `:` 使 batch 块成为空操作，shell 直接运行该脚本。

这强制执行两条规则，你必须遵守：

- **Hook 脚本没有扩展名**（`session-start`，而非 `session-start.sh`）。Claude Code 的 Windows 处理会在任何包含 `.sh` 的命令前添加 `bash`，这会导致双重调用。命名你的 hook 脚本时不带扩展名。
- 不要编写每个操作系统的 hook 变体。一个无扩展名的 bash 脚本加上多语言包装器覆盖所有三个平台。

`hooks/run-hook.cmd` 本身是权威实现——请阅读它。关于分配器模式的背景和原理，请参见 `docs/windows/polyglot-hooks.md`。

---

## 第 8 部分 — 提交 PR

- 目标为 **`dev`** 分支。每个 PR 只包含一个 harness。
- 填写 PR 模板的 **"New harness support"** 章节，并粘贴完整的验收测试记录（展示 `brainstorming` 自动触发的"Let's make a react todo list"会话）。没有此证明的 PR 将被关闭。
- Superpowers 是零依赖 plugin。不要添加第三方运行时依赖。添加新 harness 是贡献者规则允许的唯一例外，即便如此，也要保持仅限于集成严格所需的内容——仅类型导入（编译时消失）是可以的，运行时包则不行。
- 不要修改 skill 正文（第 1 部分）。如果你发现自己为了移植而编辑了 `SKILL.md`，正确的修复应在你的工具映射中。

---

## 附录 A — 参考集成（当前）

将此作为实时索引；有疑问时，阅读文件而非此表。

| Harness | Entry point | Bootstrap mechanism | Tool mapping | Tests | Distribution |
|---|---|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` + `hooks/hooks.json` | shell hook → `hooks/session-start`（`hookSpecificOutput.additionalContext`） | 原生 `Skill` 工具；`references/claude-code-tools.md` | `tests/hooks/` | marketplace |
| Codex | `.codex-plugin/plugin.json` + `hooks/hooks-codex.json` | shell hook → `hooks/session-start-codex` | `references/codex-tools.md` | `tests/codex-plugin-sync/`、`tests/hooks/` | fork 同步（`scripts/sync-to-codex-plugin.sh`） |
| Cursor | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json` | shell hook → `hooks/session-start`（`additional_context`） | `references/claude-code-tools.md` | `tests/hooks/` | 手动编写 |
| Copilot CLI | （共享 Claude Code hook 路径；`COPILOT_CLI` 环境变量） | shell hook → `hooks/session-start`（`additionalContext`） | `references/copilot-tools.md` | `tests/hooks/` | — |
| Gemini CLI | `gemini-extension.json` + `GEMINI.md` | 指令文件 `@`-include bootstrap + 映射 | `references/gemini-tools.md` | — | `gemini extensions install` |
| Kimi Code | `.kimi-plugin/plugin.json` | 清单 `sessionStart.skill` 加载 `using-superpowers` | 清单中的内联 `skillInstructions` | `tests/kimi/` | marketplace 或 `/plugins install` GitHub URL |
| OpenCode | `.opencode/plugins/superpowers.js`（通过根 `package.json` `main` 声明） | 进程内：`config` hook 注册 skills 目录；`experimental.chat.messages.transform` 注入用户消息 | `superpowers.js` 内联 | `tests/opencode/` | `opencode.json` plugin git URL |
| pi | `.pi/extensions/superpowers.ts` | 进程内：`resources_discover` 注册 skills；`context` 事件注入用户消息；生命周期标志 + 压缩感知 | `piToolMapping()` 内联**并** `references/pi-tools.md` | `tests/pi/` | 仓库根 `package.json` 字段 |

## 附录 B — 曾让移植者陷入困境的陷阱

- **选择加入不是移植。** 如果你的人类伙伴每次会话都需要做点什么才能获得 Superpowers，验收测试就会失败。重新阅读第 2 部分。
- **错误的 JSON 字段 → 静默失败或双重注入。** 仅形状 A。确认确切的字段/嵌套；Claude Code 读取两个字段且不去重。
- **Hook 配置模式因 harness 而异。** 形状 A。Cursor 的 `hooks-cursor.json` 看起来与 Claude/Codex 的完全不同（`version`、小写 `sessionStart`、相对路径命令、没有 `matcher`/`type`/`async`）。匹配最接近的现有文件。
- **Plugin-root 环境变量因 harness 而异。** 形状 A。Hook 命令使用 `${CLAUDE_PLUGIN_ROOT}`（Claude）、`${PLUGIN_ROOT}`（Codex）或相对路径（Cursor）。使用你的 harness 导出的变量；脚本自行重新推导根目录。
- **系统消息注入。** 形状 B 是有意注入*用户*消息的（#750、#894）。不要"修复"为系统消息。
- **每步 vs 每轮回调。** OpenCode 每步触发（每次调用去重守卫）；pi 每轮触发（生命周期标志 + `agent_end` 重置）。将一个 harness 的去重策略复制到另一个的回调频率上会破坏注入。
- **消息对象形状因 harness 而异。** 形状 B。pi 和 OpenCode 使用不兼容的形状；发现你的，不要复制参考实现的对象字面量。
- **寻找不存在的 skill 注册 API。** 没有 skill 系统的 harness（不仅仅是没有 `Skill` 工具）没有任何可注册的内容——模型按需读取 `SKILL.md`。不要假定存在 `skillPaths` 等效物。
- **映射在两个地方。** 对于进程内 plugin，映射可能同时存在于内联和 `references/` 文件中（pi）。同时更新两者。
- **"永远不要读取 skill 文件"这句话。** 它的意思是"不要绕过你平台的 skill 加载机制"，而不是"永远不要使用文件读取"。在没有 skill 工具的 harness 上，该机制*就是*读取 `SKILL.md`——在映射中明确说明（第 5 部分）。
- **Windows 上的 `.sh`。** 保持 hook 脚本无扩展名（第 7 部分）。
- **未注册的版本。** 未添加到 `.version-bump.json` 的新清单会分发过时版本（第 6 部分）。
- **编辑 skills 以适应 harness。** 绝不可以。修复应在工具映射中。
