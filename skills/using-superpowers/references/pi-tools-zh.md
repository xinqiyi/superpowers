# Pi 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Pi 上，这些解析为下面的工具。

| Action skills request | Pi equivalent |
| --- | --- |
| Invoke a skill | Pi native skills: load the relevant `SKILL.md` with `read`, or let the human use `/skill:name` |
| Read a file | `read` |
| Create a file | `write` |
| Edit a file | `edit` |
| Run a shell command | `bash` |
| Search file contents | `grep` when active; otherwise `bash` with `rg`/`grep` |
| Find files by name | `find` or `bash` with shell globs |
| List files and subdirectories | `ls` when active; otherwise `bash` with `ls` |
| Dispatch a subagent (`Subagent (general-purpose):` template) | Use an installed subagent tool such as `subagent` from `pi-subagents` if available |
| Task tracking ("create a todo", "mark complete") | Use an installed todo/task tool if available, otherwise track tasks in the plan or `TODO.md` |

## Skills

Pi 从配置的 skill 目录和已安装的 Pi 包中发现 skills。Superpowers Pi 包应通过其 `pi.skills` 清单条目暴露 `skills/`。Pi 不暴露 Claude Code 的 `Skill` 工具，但 agent 仍应遵循 Superpowers 规则：当 skill 适用时，在回应之前加载并遵循它。

## Subagents

Pi 核心不提供标准的 subagent 工具。`pi-subagents` 包是一个强可选的伴侣，提供带有单 agent、链、并行、异步、分支上下文和恢复/状态工作流的 `subagent` 工具。如果没有 subagent 工具可用，不要编造 `Task` 调用；在当前会话中串行执行，或解释可选的 subagent 功能未安装。

## 任务列表

Pi 核心不提供标准的任务列表工具。如果已安装 todo/task 扩展，使用其文档化的工具。否则使用 Superpowers 计划文件、Markdown 中的检查清单或仓库本地的 `TODO.md` 进行任务跟踪。较旧的 Superpowers 文档可能引用 `TodoWrite`；将其视为上述的任务跟踪操作。
