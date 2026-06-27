# Copilot CLI 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Copilot CLI 上，这些解析为下面的工具。

| Action skills request | Copilot CLI equivalent |
|----------------------|----------------------|
| Read a file | `view` |
| Create / edit / delete a file | `apply_patch` (Copilot CLI has no separate create/edit/write tools) |
| Run a shell command | `bash` |
| Search file contents | `rg` (ripgrep; Copilot CLI does not expose a `grep` tool) |
| Find files by name | `glob` |
| Fetch a URL | `web_fetch` |
| Search the web | `web_search` |
| Invoke a skill | `skill` |
| Dispatch a subagent (`Subagent (general-purpose):` template) | `task` with `agent_type: "general-purpose"` (other accepted types: `explore`, `task`, `code-review`, `research`, `configure-copilot`) |
| Multiple parallel dispatches | Multiple `task` calls in one response |
| Subagent status/output/control | `read_agent`, `list_agents`, `write_agent` |
| Task tracking ("create a todo", "mark complete") | `update_todo` |
| Enter / exit plan mode | No equivalent — stay in the main session |

## 指令文件

当 skill 提到"你的指令文件"时，在 Copilot CLI 上是仓库根目录的 **`AGENTS.md`**。如果 `AGENTS.md` 和 `.github/copilot-instructions.md` 都存在，Copilot 会读取两者。

## 个人 skills 目录

用户级 skills 位于 **`~/.copilot/skills/`**。Copilot CLI 也识别跨运行时别名 **`~/.agents/skills/`**，与 Codex 和 Gemini CLI 共享。每个 skill 是一个包含 `SKILL.md`（带有 `name` 和 `description` 前置元数据）的子目录。

## 异步 Shell 会话

Copilot CLI 支持持久的异步 shell 会话：

| Tool | Purpose |
|------|---------|
| `bash` with `mode: "async"` (and optionally `detach: true`) | Start a long-running command in the background; returns a `shellId` |
| `write_bash` | Send input to a running async session |
| `read_bash` | Read output from an async session |
| `stop_bash` | Terminate an async session |
| `list_bash` | List all active shell sessions |

## 额外的 Copilot CLI 工具

| Tool | Purpose |
|------|---------|
| `store_memory` | Persist facts about the codebase for future sessions |
| `report_intent` | Update the UI status line with current intent |
| `sql` | Query the session's SQLite database (todos, metadata) |
| `fetch_copilot_cli_documentation` | Look up Copilot CLI documentation |
| GitHub MCP tools (`github-mcp-server-*`) | Native GitHub API access (issues, PRs, code search) |
