# Gemini CLI 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Gemini CLI 上，这些解析为下面的工具。

| Action skills request | Gemini CLI equivalent |
|----------------------|----------------------|
| Read a file | `read_file` |
| Read multiple files at once | `read_many_files` |
| Create a new file | `write_file` |
| Edit a file | `replace` |
| Run a shell command | `run_shell_command` |
| Search file contents | `grep_search` |
| Find files by name | `glob` |
| List files and subdirectories | `list_directory` |
| Fetch a URL | `web_fetch` |
| Search the web | `google_web_search` |
| Invoke a skill | `activate_skill` |
| Dispatch a subagent (`Subagent (general-purpose):` template) | `invoke_agent` with `agent_name: "generalist"` (invocable via `@generalist` chat syntax — see [Subagent support](#subagent-support)) |
| Multiple parallel dispatches | Multiple `invoke_agent` calls in the same response |
| Task tracking ("create a todo", "mark complete") | `write_todos` (statuses: pending, in_progress, completed, cancelled, blocked) |

## 指令文件

当 skill 提到"你的指令文件"时，在 Gemini CLI 上是 **`GEMINI.md`**。Gemini CLI 分层加载 `GEMINI.md`：全局在 `~/.gemini/GEMINI.md`，项目级在工作空间目录及其祖先目录中，以及当工具访问这些目录中的文件时子目录的 `GEMINI.md` 文件。

## 个人 skills 目录

用户级 skills 位于 **`~/.gemini/skills/`**，**`~/.agents/skills/`** 作为跨运行时别名（与 Codex 和 Copilot CLI 共享）。当两个目录在同一作用域中存在时，`.agents/skills/` 优先。每个 skill 是一个包含 `SKILL.md`（带有 `name` 和 `description` 前置元数据）的子目录。

## Subagent 支持

Gemini CLI 通过 `invoke_agent` 工具分派 subagent，该工具接受 `agent_name` 和 `prompt` 参数。相同的分派也作为聊天语法快捷方式呈现：输入 `@generalist <prompt>` 等同于调用 `agent_name: "generalist"` 的 `invoke_agent`。内置 agent 名称包括 `generalist`、`cli_help`、`codebase_investigator` 和（启用浏览器工具时）`browser_agent`。

Skills 使用 `Subagent (general-purpose):` 分派，并引用提示模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`）或提供内联提示。在 Gemini CLI 上：

| Skill dispatch form | Gemini CLI equivalent |
|---------------------|----------------------|
| References a `*-prompt.md` template (implementer, task-reviewer, code-reviewer, etc.) | Fill the template, then `invoke_agent` with `agent_name: "generalist"` and the filled prompt |
| References `superpowers:requesting-code-review`'s `./code-reviewer.md` | `invoke_agent` with `agent_name: "generalist"` and the filled review template |
| Inline prompt (no template referenced) | `invoke_agent` with `agent_name: "generalist"` and your inline prompt |

### 提示填充

Skills 提供带有占位符的提示模板，如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。在将完整提示传递给 `invoke_agent` 之前，填充所有占位符。提示模板本身包含 agent 的角色、审查标准和预期的输出格式——subagent 将遵循它。

### 并行分派

Gemini CLI 支持并行 subagent 分派。在同一个响应中发出多个 `invoke_agent` 调用（或在一个提示中多次 `@generalist` 调用）以并行运行独立的 subagent 工作。保持有依赖关系的任务为串行，但不要仅仅为了保持更简单的历史而串行化独立的 subagent 任务。

## 额外的 Gemini CLI 工具

这些工具是 Gemini CLI 独有的：

| Tool | Purpose |
|------|---------|
| `save_memory` (legacy) | Persist facts across sessions when `experimental.memoryV2 = false` |
| `get_internal_docs` | Look up Gemini CLI's bundled documentation |
| `ask_user` | Pose structured questions to the user (text / single-select / multi-select) |
| `enter_plan_mode` / `exit_plan_mode` | Switch into and out of read-only plan mode |
| `update_topic` | Update the current conversation's topic / strategic-intent metadata |
| `complete_task` | Signal that a Gemini subagent has completed and return its result to the parent agent |
| `tracker_create_task`, `tracker_update_task`, `tracker_get_task`, `tracker_list_tasks`, `tracker_add_dependency`, `tracker_visualize` | Rich task tracker with dependency and visualization support |
| `read_mcp_resource`, `list_mcp_resources` | MCP resource access |
