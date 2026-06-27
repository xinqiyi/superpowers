---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# 完成开发分支

## 概述

通过展示清晰的选项并处理所选工作流，引导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 展示选项 → 执行选择 → 清理。

**开始时声明：** "我正在使用 finishing-a-development-branch skill 来完成这项工作。"

## 流程

### 第 1 步：验证测试

**在展示选项之前，验证测试通过：**

```bash
# 运行项目测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止。不要进入第 2 步。

**如果测试通过：** 继续到第 2 步。

### 第 2 步：检测环境

**在展示选项之前确定工作空间状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了显示哪个菜单以及如何清理：

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (normal repo) | Standard 4 options | No worktree to clean up |
| `GIT_DIR != GIT_COMMON`, named branch | Standard 4 options | Provenance-based (see Step 6) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | Reduced 3 options (no merge) | No cleanup (externally managed) |

### 第 3 步：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："这个分支是从 main 分出的——对吗？"

### 第 4 步：展示选项

**普通仓库和已命名分支的 worktree——精确展示这 4 个选项：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD——精确展示这 3 个选项：**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**不要添加解释**——保持选项简洁。

### 第 5 步：执行选择

#### 选项 1：本地合并

```bash
# 获取主仓库根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并——在移除任何内容之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 验证合并后的测试结果
<test command>

# 只有在合并成功后：清理 worktree（第 6 步），然后删除分支
```

然后：清理 worktree（第 6 步），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>
```

**不要清理 worktree**——用户需要它存活以便根据 PR 反馈进行迭代。

#### 选项 3：保持现状

报告："保持分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**首先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待准确的确认。

如果已确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（第 6 步），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### 第 6 步：清理工作空间

**仅对选项 1 和 4 运行。** 选项 2 和 3 始终保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无需清理 worktree。完成。

**如果 worktree 路径在 `.worktrees/` 或 `worktrees/` 下：** Superpowers 创建了此 worktree——我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**否则：** 主机环境（harness）拥有此工作空间。不要移除它。如果你的平台提供了工作空间退出工具，请使用它。否则，将工作空间保留在原位。

## 快速参考

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## 常见错误

**跳过测试验证**
- **问题：** 合并损坏的代码，创建失败的 PR
- **修复：** 在提供选项之前始终验证测试

**开放式问题**
- **问题：** "下一步我该做什么？" 模棱两可
- **修复：** 精确展示 4 个结构化选项（或 detached HEAD 为 3 个）

**为选项 2 清理 worktree**
- **问题：** 移除了用户用于 PR 迭代所需的 worktree
- **修复：** 仅对选项 1 和 4 进行清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍引用该分支
- **修复：** 先合并，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 位于正在移除的 worktree 内部时，命令静默失败
- **修复：** 在 `git worktree remove` 之前始终 `cd` 到主仓库根目录

**清理 harness 拥有的 worktrees**
- **问题：** 移除 harness 创建的 worktree 会导致幻影状态
- **修复：** 仅清理 `.worktrees/` 或 `worktrees/` 下的 worktree

**丢弃时没有确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 确认

## 红旗

**永远不要：**
- 在测试失败的情况下继续
- 在不验证合并结果测试的情况下合并
- 未经确认删除工作
- 未经明确请求强制推送
- 在确认合并成功之前移除 worktree
- 清理不是你创建的 worktree（来源检查）
- 从 worktree 内部运行 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在展示菜单之前检测环境
- 精确展示 4 个选项（或 detached HEAD 为 3 个）
- 对选项 4 获取输入确认
- 仅对选项 1 和 4 清理 worktree
- 在移除 worktree 之前 `cd` 到主仓库根目录
- 移除后运行 `git worktree prune`
