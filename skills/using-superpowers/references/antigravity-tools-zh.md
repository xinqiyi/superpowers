# Antigravity CLI (`agy`) 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Antigravity CLI (`agy`) 上，这些解析为下面的工具。

| Action skills request | Antigravity CLI equivalent |
|----------------------|----------------------|
| Read a file | `view_file` |
| Create a new file | `write_to_file` |
| Edit a file | `replace_file_content` |
| Edit a file in several places at once | `multi_replace_file_content` |
| Run a shell command | `run_command` |
| Search file contents | `grep_search` |
| Find files by name / list a directory | `list_dir` (no dedicated glob tool — combine `list_dir` with `grep_search`) |
| Fetch a URL | `read_url_content` |
| Search the web | `search_web` |
| Pose a structured question to your human partner | `ask_question` |
| Dispatch a subagent (`Subagent (general-purpose):` template) | `invoke_subagent` with a built-in `TypeName` — `self` for full-capability work, `research` for read-only (see [Subagent support](#subagent-support)) |
| Multiple parallel dispatches | Multiple entries in one `invoke_subagent` call's `Subagents` array |
| Task tracking ("create a todo", "mark complete") | a **task artifact** — `write_to_file` with `IsArtifact: true` and `ArtifactType: "task"` (see [Task tracking](#task-tracking)). **Not** `manage_task`, which manages background processes. |

## 调用 skill — 读取其 `SKILL.md`

Antigravity 在每个会话开始时向你展示每个已安装 skill 的 `name` + `description`，但它**没有 `Skill`/`activate_skill` 工具**。要加载 skill，当 skill 适用时，**使用 `view_file` 读取其 `SKILL.md`，设置 `IsSkillFile: true`**——例如在 `.../plugins/superpowers/skills/<skill-name>/SKILL.md` 上使用 `view_file` 并设置 `IsSkillFile: true`。（`IsSkillFile` 是 agy 自己的信号，表示你正在读取文件以*执行其指令*，而不是编辑或预览它——每当你加载 skill 时都要设置它。）

这是此 harness 上受祝福的 skill 加载机制。通用规则"永远不要手动读取 skill 文件"意味着"不要绕过你平台的 skill 加载机制"——而在 Antigravity 上，读取 `SKILL.md` *就是*这个机制。读取它遵守了规则而不是打破了它。

你已经知道哪些 skills 存在以及它们的用途：它们的名称和描述在会话开始时就在你面前。当描述匹配你即将做的事情时，在行动之前读取该 skill 的 `SKILL.md`。

## Subagent 支持

Antigravity 使用 `invoke_subagent` 分派 subagent，在 `Subagents` 数组中为每个 subagent 传递一个 `TypeName`。两个 `TypeName` 是**内置的**——直接使用它们，无需 `define_subagent`：

- **`self`** — 你的完整克隆，拥有你拥有的每个工具（包括 `write_to_file`/`replace_file_content`/`run_command`）。通用工作的安全默认：实现、修复、任何编辑文件或运行命令的工作。
- **`research`** — 只读（文件读取、`grep_search`、web/URL 获取；没有写入或命令访问权限）。当你特别想要一个不能进行更改的 subagent 时使用——调查和只读审查。

仅当需要自定义系统提示或能力组合时才调用 `define_subagent`：设置 `enable_write_tools: true` 以授予文件编辑**和** `run_command`，`enable_subagent_tools` 用于嵌套分派，`enable_mcp_tools` 用于 MCP。然后使用你给它的名称来调用它。（`manage_subagents` 列出/杀死正在运行的 subagent。）

Skills 使用 `Subagent (general-purpose):` 分派，并引用提示模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`）或提供内联提示。在 Antigravity 上：

| Skill dispatch form | Antigravity equivalent |
|---------------------|----------------------|
| An implementer-style `*-prompt.md` template (writes code, runs tests) | Fill the template, then `invoke_subagent` with `TypeName: "self"` and the filled prompt |
| A read-only reviewer template (`task-reviewer`, `code-reviewer`, `requesting-code-review`'s `./code-reviewer.md`) | `invoke_subagent` with `TypeName: "research"` and the filled review template |
| Inline prompt (no template referenced) | `invoke_subagent` with `TypeName: "self"` (or `"research"` if the task only reads) and your inline prompt |

### 提示填充

Skills 提供带有占位符的提示模板，如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。在将完整提示传递给 `invoke_subagent` 之前，填充所有占位符。提示模板本身包含 agent 的角色、审查标准和预期的输出格式——subagent 将遵循它。

### 并行分派

在单个 `invoke_subagent` 调用的 `Subagents` 数组中放置多个条目，以并行运行独立的 subagent 工作。保持有依赖关系的任务为串行，但不要仅仅为了保持更简单的历史而串行化独立的 subagent 任务。

## 任务跟踪

Antigravity **没有 todo / `TodoWrite` 工具**（`manage_task` 管理后台进程——`list`/`kill`/`status`/`send_input`——它*不是*检查清单）。当 skill 说要创建 todo 列表或跟踪任务时，维护一个**任务工件**：使用 `write_to_file`（`IsArtifact: true`、`ArtifactMetadata.ArtifactType: "task"`）保存的 markdown 检查清单，随着进展使用 `replace_file_content` / `multi_replace_file_content` 进行编辑。

在任何多步骤任务的开始，创建列出计划每个步骤的任务工件。当你完成每个步骤时，编辑工件以标记完成（`- [x]`）。如果计划发生变化，更新检查清单。保持它的最新状态——它是剩余工作的真相来源；一旦对话变长，在开始每个步骤之前重新读取它。
