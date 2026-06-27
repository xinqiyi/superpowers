# Codex App 兼容性：工作树和完成 Skill 适配

使 superpowers skills 在 Codex App 的沙箱工作树环境中正常工作，而不破坏现有的 Claude Code 或 Codex CLI 行为。

**Ticket：** PRI-823

## 动机

Codex App 在它管理的 git 工作树内运行 agent——detached HEAD，位于 `$CODEX_HOME/worktrees/` 下，带有阻止 `git checkout -b`、`git push` 和网络访问的 Seatbelt 沙箱。三个 superpowers skills 假设不受限制的 git 访问：`using-git-worktrees` 使用命名分支创建手动工作树，`finishing-a-development-branch` 按分支名称合并/推送/PR，而 `subagent-driven-development` 需要两者。

Codex CLI（开源终端工具）**没有**此冲突——它没有内置的工作树管理。我们的手动工作树方法在那里填补了隔离空白。问题特指 Codex App。

## 实证发现

于 2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙箱 | 完全访问沙箱 |
|---|---|---|
| `git add` | 有效 | 有效 |
| `git commit` | 有效 | 有效 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 有效 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 有效 |
| `gh pr create` | **被阻止**（网络） | 有效 |
| `git status/diff/log` | 有效 | 有效 |

额外发现：
- `spawn_agent` subagent **共享**父线程的文件系统（通过标记文件测试确认）
- 无论工作树从哪个分支启动，"Create branch"按钮都会出现在 App 头部
- App 的原生完成流程：Create branch → Commit modal → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上静默损坏（issue #10390）

## 设计：只读环境检测

三个只读 git 命令无副作用地检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

得出的两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON`——agent 在由其他方创建的工作树中（Codex App、Claude Code Agent tool、之前的 skill 运行或用户）
- **ON_DETACHED_HEAD：** `BRANCH` 为空——不存在命名分支

为什么使用 `git-dir != git-common-dir` 而非检查 `show-toplevel`：
- 在正常仓库中，两者都解析到同一个 `.git` 目录
- 在链接的工作树中，`git-dir` 是 `.git/worktrees/<name>` 而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等——避免了 `show-toplevel` 可能产生的误报
- 通过 `cd && pwd -P` 解析处理相对路径问题（`git-common-dir` 在正常仓库中返回相对 `.git`，但在工作树中返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接工作树？ | Detached HEAD？ | 环境 | 操作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整的 skill 行为（不变） |
| 是 | 是 | Codex App 工作树（workspace-write） | 跳过工作树创建；在完成时发送交接负载 |
| 是 | 否 | Codex App（完全访问）或手动工作树 | 跳过工作树创建；完整的完成流程 |
| 否 | 是 | 不常见（手动 detached HEAD） | 正常创建工作树；在完成时发出警告 |

## 更改

### 1. `using-git-worktrees/SKILL.md`——添加步骤 0（约 12 行）

在"Overview"和"Directory Selection Process"之间的新部分：

**Step 0: Check if Already in an Isolated Workspace**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过工作树创建。而是：
1. 跳到 Creation Steps 下的"Run Project Setup"子部分——`npm install` 等是幂等的，为安全起见值得运行
2. 然后"Verify Clean Baseline"——运行测试
3. 报告分支状态：
   - 在分支上："Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD："Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

如果 `GIT_DIR == GIT_COMMON`，继续完整的工作树创建流程（不变）。

当步骤 0 触发时跳过安全检查（.gitignore 检查）——对外部创建的工作树不相关。

更新集成部分的"Called by"条目。将每个条目的描述从特定上下文文本改为："Ensures isolated workspace (creates one or verifies existing)"。例如，`subagent-driven-development` 条目从"REQUIRED: Set up isolated workspace before starting"改为"REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙箱回退：** 如果 `GIT_DIR == GIT_COMMON` 且 skill 进入创建步骤，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙箱拒绝），则将其视为检测到受限环境。回退到步骤 0 的"已在工作区"行为——跳过创建，在当前目录中运行设置和基准测试，相应报告。

在步骤 0 报告后，停止。不要继续到目录选择或创建步骤。

**其他所有内容不变：** 目录选择、安全检查、创建步骤、项目设置、基准测试、快速参考、常见错误、红旗。

### 2. `finishing-a-development-branch/SKILL.md`——添加步骤 1.5 + 清理保护（约 20 行）

**Step 1.5: Detect Environment**（在步骤 1 "Verify Tests"之后，步骤 2 "Determine Base Branch"之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过步骤 2 和 3（不需要基础分支或选项）。
- **路径 B 和 C** 正常通过步骤 2（Determine Base Branch）和步骤 3（Present Options）。

**路径 A——外部管理工作树 + detached HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的完成控制只对已提交的工作操作。

然后向用户呈现此信息（不要显示 4 选项菜单）：

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名称推导：如果可用则使用 ticket ID（例如 `pri-823/codex-compat`），否则使用计划标题的前 5 个词的 slug，否则省略建议。避免在分支名称中包含敏感内容（漏洞描述、客户名称）。

跳到步骤 5（对于外部管理工作树，清理为空操作）。

**路径 B——外部管理工作树 + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常显示 4 选项菜单。（步骤 5 的清理保护将独立重新检测外部管理状态。）

**路径 C——正常环境**（`GIT_DIR == GIT_COMMON`）：

像今天一样显示 4 选项菜单（不变）。

**步骤 5 清理保护：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 检测（不要依赖更早的 skill 输出——完成 skill 可能在不同会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove`——主机环境拥有此工作区。

否则，像今天一样检查和移除。注意：现有的步骤 5 文本说"For Options 1, 2, 4"但 Quick Reference 表和 Common Mistakes 部分说"Options 1 & 4 only。"新的保护在此现有逻辑之前添加，不更改哪些选项触发清理。

**其他所有内容不变：** 选项 1-4 逻辑、快速参考、常见错误、红旗。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md`——每文件一行编辑

两个 skill 都有相同的集成部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其他所有内容不变：** 调度/审查循环、提示模板、模型选择、状态处理、红旗。

### 4. `codex-tools.md`——添加环境检测文档（约 15 行）

末尾的两个新部分：

**环境检测：**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App 完成：**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 不变的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md`——subagent 提示不变
- `executing-plans/SKILL.md`——仅一行集成描述更改（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md`——无工作树或完成操作
- `.codex/INSTALL.md`——安装过程不变
- 4 选项完成菜单——为 Claude Code 和 Codex CLI 完全保留
- 完整的工作树创建流程——为非工作树环境完全保留
- Subagent 调度/审查/迭代循环——不变（文件系统共享已确认）

## 范围总结

| 文件 | 更改 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（步骤 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（步骤 1.5 + 清理保护） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件中添加/更改约 50 行。零个新文件。零个破坏性更改。

## 未来考虑

如果第三个 skill 需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件中（方法 B）。现在不需要——仅 2 个 skill 使用它。

## 测试计划

### 自动化（实施后在 Claude Code 中运行）

1. 正常仓库检测——断言 IN_LINKED_WORKTREE=false
2. 链接工作树检测——`git worktree add` 测试工作树，断言 IN_LINKED_WORKTREE=true
3. Detached HEAD 检测——`git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 完成 skill 交接输出——验证受限环境中的交接消息（非 4 选项菜单）
5. **步骤 5 清理保护**——创建链接工作树（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入其中，运行步骤 5 清理检测（`GIT_DIR` vs `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同的检测，断言它会调用 `git worktree remove`。之后清理测试工作树。

### 手动 Codex App 测试（5 个测试）

1. 工作树线程（workspace-write）中的检测——验证 GIT_DIR != GIT_COMMON，空分支
2. 工作树线程（完全访问）中的检测——相同的检测，不同的沙箱行为
3. 完成 skill 交接格式——验证 agent 发出交接负载，而非 4 选项菜单
4. 完整生命周期——检测 → 提交 → 完成检测 → 正确行为 → 清理
5. **本地线程中的沙箱回退**——启动 Codex App **本地线程**（workspace-write 沙箱）。提示："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change."预检查：`git checkout -b test-sandbox-check` 应因 `Operation not permitted` 失败。预期：skill 检测 `GIT_DIR == GIT_COMMON`（正常仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到步骤 0"已在工作区"行为——运行设置、基准测试，从当前目录报告就绪。通过：agent 优雅恢复，无神秘错误消息。失败：agent 打印原始 Seatbelt 错误、重试，或以混乱输出放弃。

### 回归

- 现有的 Claude Code skill 触发测试仍然通过
- 现有的 subagent-driven-development 集成测试仍然通过
- 正常的 Claude Code 会话：完整的工作树创建 + 4 选项完成仍然工作
