# Claude Code 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Claude Code 上，这些解析为下面的工具。

## 工具

| Action skills request | Claude Code tool |
|----------------------|------------------|
| Read a file | `Read` |
| Create a new file | `Write` |
| Edit a file | `Edit` |
| Run a shell command | `Bash` |
| Search file contents | `Grep` |
| Find files by name | `Glob` |
| Fetch a URL | `WebFetch` |
| Search the web | `WebSearch` |
| Invoke a skill | `Skill` |
| Dispatch a subagent (`Subagent (general-purpose):` template) | `Agent` (older releases named this `Task`) |
| Multiple parallel dispatches | Multiple `Agent` calls in one response |
| Task tracking ("create a todo", "mark complete") | `TaskCreate`, `TaskUpdate`, `TaskList`, `TaskGet`; `TodoWrite` in `claude -p` / Agent SDK unless `CLAUDE_CODE_ENABLE_TASKS=1` is set |
| Background-process / subagent lifecycle (read output, cancel) | `TaskOutput`, `TaskStop` — these are distinct from the todo tools above and apply to running shells, agents, and remote sessions |

## 指令文件

当 skill 提到"你的指令文件"时，在 Claude Code 上是 **`CLAUDE.md`**。Claude Code 从当前工作目录向上遍历目录树，并连接沿途找到的每个 `CLAUDE.md` 和 `CLAUDE.local.md`。标准位置：

| Scope | Location |
|-------|----------|
| Project (team-shared) | `./CLAUDE.md` or `./.claude/CLAUDE.md` |
| User global | `~/.claude/CLAUDE.md` |
| Local-private (gitignored) | `./CLAUDE.local.md` |
| Managed policy (org-wide) | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS), `/etc/claude-code/CLAUDE.md` (Linux/WSL), `C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) |

CLAUDE.md 文件可以通过 `@path/to/file` 导入引入额外内容（相对或绝对路径，最多五层深度）。子目录中的 `CLAUDE.md` 文件也会被自动发现，并在 Claude Code 读取这些子目录中的文件时按需加载。

Claude Code **不会**直接读取 `AGENTS.md`。如果项目已经为其他 agent 维护了 `AGENTS.md`，从 `CLAUDE.md` 中导入它，以便两个运行时共享相同的指令：

```markdown
@AGENTS.md

## Claude Code

(Claude-Code-specific instructions go here.)
```

有关路径范围规则和更大项目组织的信息，请参见 `.claude/rules/`（规则可以通过 `paths` 前置元数据限定到特定文件，并按需加载）。

## 个人 skills 目录

用户级 skills 位于 **`~/.claude/skills/`**。每个 skill 是一个包含 `SKILL.md`（带有 `name` 和 `description` 前置元数据）以及任何支持文件的子目录。Claude Code 目前不识别 Codex、Copilot CLI 和 Gemini CLI 读取的跨运行时 `~/.agents/skills/` 路径；如果你将来依赖跨运行时支持，请验证 [官方 skills 文档](https://code.claude.com/docs/en/skills)。
