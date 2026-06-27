# Codex 工具映射

Skills 以动作描述（"分派 subagent"、"创建 todo"、"读取文件"）。在 Codex 上，这些解析为下面的工具。

| Action skills request | Codex equivalent |
|----------------------|------------------|
| Read a file | `shell` (e.g., `cat`, `head`, `tail`) — Codex reads files via shell |
| Create / edit / delete a file | `apply_patch` (structured diff for create, update, delete) |
| Run a shell command | `shell` |
| Search file contents | `shell` (e.g., `grep`, `rg`) |
| Find files by name | `shell` (e.g., `find`, `ls`) |
| Fetch a URL | `shell` with `curl` / `wget` — Codex has no native fetch tool |
| Search the web | `web_search` (enabled by default; configurable in `config.toml` via the top-level `web_search` setting — `live`, `cached`, or `disabled`) |
| Invoke a skill | Skills load natively — just follow the instructions |
| Dispatch a subagent (`Subagent (general-purpose):` template) | `spawn_agent` (see [Subagent dispatch requires multi-agent support](#subagent-dispatch-requires-multi-agent-support)) |
| Multiple parallel dispatches | Multiple `spawn_agent` calls in one response |
| Wait for subagent result | `wait_agent` |
| Free up subagent slot when done | `close_agent` |
| Task tracking ("create a todo", "mark complete") | `update_plan` |

## 指令文件

当 skill 提到"你的指令文件"时，在 Codex 上是项目根目录的 **`AGENTS.md`**。Codex 也读取 `~/.codex/AGENTS.md` 获取全局上下文，如果存在 `AGENTS.override.md`（在项目树中或 `~/.codex/` 中），则优先使用。Codex 从项目根目录向下到当前工作目录，连接沿途找到的 `AGENTS.md` 文件，直到达到 `project_doc_max_bytes`（默认 32 KiB）。

## 个人 skills 目录

用户级 skills 位于 **`$CODEX_HOME/skills/`**（默认 `~/.codex/skills/`）。Codex 也读取跨运行时路径 **`~/.agents/skills/`**（与 Copilot CLI 和 Gemini CLI 共享）。当两个目录在同一作用域中存在时，Codex 将它们作为独立的 skill 目录加载——Codex 的文档目前不记录它们之间的优先级。每个 skill 是一个包含 `SKILL.md`（带有 `name` 和 `description` 前置元数据）的子目录。

## Subagent 分派需要多 agent 支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这会为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills 启用 `spawn_agent`、`wait_agent` 和 `close_agent`。

旧版说明：`rust-v0.115.0` 之前的 Codex 版本将生成的 agent 等待暴露为 `wait`。当前的 Codex 对生成的 agent 使用 `wait_agent`。`wait` 名称现在属于 code-mode `exec/wait`，它通过 `cell_id` 恢复一个已让出的 exec 单元；它不是生成的 agent 结果工具。

## 环境检测

创建 worktrees 或完成分支的 skills 应该在继续之前使用只读 git 命令检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙箱创建分支/推送/PR）

参见 `using-git-worktrees` 第 0 步和 `finishing-a-development-branch` 第 1 步，了解每个 skill 如何使用这些信号。

## Codex App 完成

当沙箱阻止分支/推送操作（在外部管理的 worktree 中处于 detached HEAD）时，agent 提交所有工作并通知用户使用 App 的原生控件：

- **"Create branch"** — 命名分支，然后通过 App UI 进行 commit/push/PR
- **"Hand off to local"** — 将工作转移到用户的本地 checkout

Agent 仍然可以运行测试、暂存文件，并输出建议的分支名称、commit 消息和 PR 描述供用户复制。
