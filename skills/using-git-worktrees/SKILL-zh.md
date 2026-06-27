---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作空间中进行。优先使用平台的原生 worktree 工具。仅在没有原生工具可用时回退到手动 git worktree。

**核心原则：** 首先检测现有隔离。然后使用原生工具。然后回退到 git。永远不要与 harness 对抗。

**开始时声明：** "我正在使用 using-git-worktrees skill 来设置隔离的工作空间。"

## 第 0 步：检测现有隔离

**在创建任何内容之前，检查你是否已经在隔离的工作空间中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块防护：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在得出"已经在 worktree 中"的结论之前，验证你不是在子模块中：

```bash
# If this returns a path, you're in a submodule, not a worktree — treat as normal repo
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已经在链接的 worktree 中。跳到第 2 步（项目设置）。不要创建另一个 worktree。

报告分支状态：
- 在分支上："已经在 `<path>` 的隔离工作空间中，位于分支 `<name>`。"
- Detached HEAD："已经在隔离工作空间中位于 `<path>`（detached HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你在普通仓库 checkout 中。

你的指令中用户是否已经表明了他们的 worktree 偏好？如果没有，在创建 worktree 前请求同意：

> "需要我设置一个隔离的 worktree 吗？它可以保护你当前的分支不受更改影响。"

如果有已有的偏好声明，无需询问直接遵从其偏好。如果用户拒绝同意，在当前目录工作并跳到第 2 步。

## 第 1 步：创建隔离的工作空间

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（优先）

用户已请求隔离工作空间（第 0 步同意）。你是否已经有创建 worktree 的方式？可能是一个名为 `EnterWorktree`、`WorktreeCreate` 的工具，一个 `/worktree` 命令，或一个 `--worktree` 标志。如果有，使用它并跳到第 2 步。

原生工具自动处理目录放置、分支创建和清理。当你有原生工具时使用 `git worktree add` 会创建你的 harness 无法看到或管理的幻影状态。

仅在你没有原生 worktree 工具可用时才继续到第 1b 步。

### 1b. Git Worktree 回退

**仅当第 1a 步不适用时使用此方法**——你没有可用的原生 worktree 工具。使用 git 手动创建 worktree。

#### 目录选择

按此优先级顺序。用户显式偏好总是胜过观察到的文件系统状态。

1. **检查你的指令中是否有声明的 worktree 目录偏好。** 如果用户已经指定了，直接使用无需询问。

2. **检查是否已存在项目本地的 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 胜出。

3. **如果没有其他指导可用**，默认使用项目根目录下的 `.worktrees/`。

#### 安全检查（仅限项目本地目录）

**在创建 worktree 之前必须验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，commit 更改，然后继续。

**为什么关键：** 防止意外将 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 根据选择的位置确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误（沙箱拒绝）而失败，告诉用户沙箱阻止了 worktree 创建，你将在当前目录中工作。然后在原地运行设置和基线测试。

## 第 2 步：项目设置

自动检测并运行适当的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 第 3 步：验证干净基线

运行测试以确保工作空间以干净状态启动：

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考

| Situation | Action |
|-----------|--------|
| Already in linked worktree | Skip creation (Step 0) |
| In a submodule | Treat as normal repo (Step 0 guard) |
| Native worktree tool available | Use it (Step 1a) |
| No native tool | Git worktree fallback (Step 1b) |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check instruction file, then default `.worktrees/` |
| Directory not ignored | Add to .gitignore + commit |
| Permission error on create | Sandbox fallback, work in place |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## 常见错误

### 与 Harness 对抗

- **问题：** 在平台已经提供隔离时使用 `git worktree add`
- **修复：** 第 0 步检测现有隔离。第 1a 步使用原生工具。

### 跳过检测

- **问题：** 在现有 worktree 内创建嵌套 worktree
- **修复：** 在创建任何内容之前始终运行第 0 步

### 跳过忽略验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修复：** 在创建项目本地 worktree 之前始终使用 `git check-ignore`

### 假定目录位置

- **问题：** 造成不一致，违反项目约定
- **修复：** 遵循优先级：显式指令 > 现有项目本地目录 > 默认

### 在测试失败的情况下继续

- **问题：** 无法区分新 bug 和已有问题
- **修复：** 报告失败，获得显式许可后再继续

## 红旗

**永远不要：**
- 在第 0 步检测到现有隔离时创建 worktree
- 在有原生 worktree 工具（例如 `EnterWorktree`）时使用 `git worktree add`。这是 #1 错误——如果你有它，就使用它。
- 跳过第 1a 步直接跳到第 1b 步的 git 命令
- 未验证目录被忽略（项目本地）就创建 worktree
- 跳过基线测试验证
- 未经询问在测试失败的情况下继续

**始终：**
- 首先运行第 0 步检测
- 优先选择原生工具而不是 git 回退
- 遵循目录优先级：显式指令 > 现有项目本地目录 > 默认
- 验证项目本地目录被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线
