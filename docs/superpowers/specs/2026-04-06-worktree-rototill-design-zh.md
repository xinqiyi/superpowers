# Worktree 重构：检测与委托

**日期：** 2026-04-06
**状态：** 草稿
**Ticket：** PRI-974
**包含：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对工作树管理有强烈意见——特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。与此同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供具有自己路径、生命周期管理和清理的原生工作树支持。

这产生了三种失败模式：

1. **重复**——在 Claude Code 上，skill 做了 `EnterWorktree`/`ExitWorktree` 已有的工作
2. **冲突**——在 Codex App 上，skill 试图在已管理的工作树内部创建工作树
3. **幻影状态**——skill 创建的位于 `.worktrees/` 的工作树对 harness 不可见；harness 创建的位于 `.claude/worktrees/` 的工作树对 skill 不可见

对于没有原生支持的 harness（Codex CLI、OpenCode、Copilot standalone），superpowers 填补了真正的空白。Skill 不应消失——它应在原生支持存在时让路。

## 目标

1. 在原生 harness 工作树系统存在时优先使用它们
2. 继续为缺乏原生支持的 harness 提供工作树支持
3. 修复 finishing-a-development-branch 中的三个已知 bug（#940、#999、#238）
4. 使工作树创建为 opt-in 而非强制（#991）
5. 将硬编码的 `CLAUDE.md` 引用替换为平台中立的语言（#1049）

## 非目标

- 每工作树环境约定（`.worktree-env.sh`、端口偏移）——第四阶段
- PreToolUse hooks 用于路径强制——第四阶段
- 多仓库工作树文档——第四阶段
- Brainstorming 检查清单关于工作树的更改——第四阶段
- `.superpowers-session.json` 元数据追踪（有趣的 PR #997 想法，v1 不需要）
- Hooks 符号链接到工作树（PR #965 想法，独立关注点）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来确定"我是否已在工作树中？"而非嗅探环境变量以识别 harness。这是一个稳定的 git 原语（自 git 2.5，2015），跨所有 harness 通用工作，且在新 harness 出现时无需维护。

### 声明性意图，规定性回退

Skill 描述目标（"确保工作在隔离工作区中进行"）并在原生工具可用时优先使用它们。它仅作为对没有原生工作树支持的 harness 的回退，规定特定的 git 命令。步骤 1a 居前并显式命名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；步骤 1b 紧随其后，作为 git 回退。原始 spec 保持步骤 1a 抽象（"你知道你自己的工具包"），但 TDD 证明当步骤 1a 过于模糊时，agent 会锚定在步骤 1b 的具体命令上。需要显式的工具命名和同意授权桥接才能使偏好变得可靠。

### 基于来源的所有权

谁创建工作树，谁拥有其清理权。如果 harness 创建了它，superpowers 就不碰它。如果 superpowers 创建了它（通过 git 回退），superpowers 清理它。启发式规则：如果工作树位于 `.worktrees/` 或 `worktrees/` 下，superpowers 拥有它。其他任何位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/` 或旧的用户级 Superpowers 路径）都属于 harness 或用户，保持不变。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

Skill 在创建之前获得三个新步骤，并简化创建流程。

#### 步骤 0：检测现有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 操作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 正常仓库检出 | 继续到步骤 0.5 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 已在链接工作树中 | 跳到步骤 3（项目设置）。报告："Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 外部管理工作树（例如 Codex App 沙箱） | 跳到步骤 3。报告："Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

步骤 0 不关心谁创建了工作树或哪个 harness 在运行。工作树就是工作树，无论来源如何。

**子模块保护：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在断定"已在工作树中"之前，检查我们是否不在子模块中：

```bash
# If this returns a path, we're in a submodule, not a worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在子模块中，视为 `GIT_DIR == GIT_COMMON`（继续到步骤 0.5）。

#### 步骤 0.5：同意

当步骤 0 未发现现有隔离（`GIT_DIR == GIT_COMMON`）时，在创建前询问：

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

如果同意，继续到步骤 1。如果拒绝，在原地工作——跳到步骤 3，不创建工作树。

当步骤 0 检测到现有隔离时，此步骤被完全跳过（无需询问关于已存在的内容）。

#### 步骤 1a：原生工具（首选）

> 用户已请求隔离工作区（步骤 0 同意）。检查你的可用工具——你有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志吗？如果有：用户同意创建工作树就是你的使用授权。现在使用它并跳到步骤 3。

使用原生工具后，跳到步骤 3（项目设置）。

**设计说明——TDD 修订：** 原始 spec 使用了一个故意简短、抽象的步骤 1a（"你知道你自己的工具包——skill 不需要命名特定工具"）。TDD 验证否定了这一点：agent 锚定在步骤 1b 的具体 git 命令上，忽略了抽象指导（2/6 通过率）。三个更改修复了它（GREEN 和 PRESSURE 测试中 50/50 通过率）：

1. **显式工具命名**——列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree` 的名字，将决策从解释（"我有原生工具吗？"）转变为事实查找（"`EnterWorktree` 在我的工具列表中吗？"）。在没有这些工具的平台上，agent 简单检查，找不到任何东西，然后落入步骤 1b。未观察到误报。
2. **同意桥接**——"用户同意创建工作树就是你的使用授权"直接解决了 `EnterWorktree` 的工具级护栏（"仅当用户明确要求时"）。工具描述覆盖 skill 指令（Claude Code #29950），因此 skill 必须将用户同意框定为工具所需的授权。
3. **红旗条目**——在红旗部分命名特定的反模式（"当你拥有原生工作树工具时使用 `git worktree add`——这是 #1 错误"）。

文件拆分（将步骤 1b 放在单独的 skill 中）经过测试证明不必要。锚定问题通过步骤 1a 文本的质量解决，而非通过 git 命令的物理分离。使用完整 240 行 skill（所有 git 命令可见）的控制测试以 20/20 通过。

#### 步骤 1b：Git Worktree 回退

当没有原生工具可用时，手动创建工作树。

**目录选择**（优先级顺序）：
1. 检查项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）中是否有工作树目录偏好。
2. 检查是否存在 `.worktrees/` 或 `worktrees/` 目录——如果找到，使用它。如果两者都存在，`.worktrees/` 获胜。
3. 默认使用 `.worktrees/`。

无交互式目录选择提示。旧的用户级 Superpowers 工作树路径不被检测或提供；新的手动工作树是项目本地的，除非用户明确指定其他位置。

**安全检查**（仅项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被忽略，在继续前添加到 `.gitignore` 并提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Hooks 感知：** Git 工作树不继承父仓库的 hooks 目录。通过 1b 创建工作树后，如果存在 hooks 目录，从主仓库符号链接：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这防止了 pre-commit 检查、linters 和其他 hooks 在工作移到工作树时静默停止。（来自 PR #965 的想法。）

**沙箱回退：** 如果 `git worktree add` 因权限错误失败，视为受限环境。跳过创建，在当前目录工作，继续到步骤 3。

**步骤编号说明：** 当前 skill 将步骤 1-4 作为扁平列表。此重新设计使用 0、0.5、1a、1b、3、4。没有步骤 2——它是旧的单体"创建隔离工作区"，现已拆分为 1a/1b 结构。实现应干净地重新编号（例如，0 → "Step 0: Detect"，0.5 → 在步骤 0 流程内，1a/1b → "Step 1"，3 → "Step 2"，4 → "Step 3"）或保持当前编号并附带说明。由实现者选择。

#### 步骤 3-4：项目设置和基准测试（不变）

无论哪个路径创建了工作区（步骤 0 检测到现有、步骤 1a 原生工具、步骤 1b git 回退，或根本没有工作树），执行汇聚：

- **步骤 3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **步骤 4：** 运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

完成 skill 获得环境检测并修复三个 bug。

#### 步骤 1：验证测试（不变）

运行项目的测试套件。如果测试失败，停止。不提供完成选项。

#### 步骤 1.5：检测环境（新增）

重新运行与创建步骤 0 中相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（正常仓库） | 标准 4 选项 | 无工作树需要清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（参见步骤 5） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 精简菜单：推为新分支 + PR、保持原样、丢弃 | 无合并选项（无法从 detached HEAD 合并） |

#### 步骤 2：确定基础分支（不变）

#### 步骤 3：提供选项

**正常仓库和命名分支工作树：**

1. 合并回 `<base-branch>` 本地
2. 推送并创建 Pull Request
3. 保持分支原样（我之后处理）
4. 丢弃此工作

**Detached HEAD：**

1. 推为新分支并创建 Pull Request
2. 保持原样（我之后处理）
3. 丢弃此工作

#### 步骤 4：执行选择

**选项 1（本地合并）：**

```bash
# Get main repo root for CWD safety (Bug #238 fix)
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first, verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# Only after merge succeeds: remove worktree, then delete branch (Bug #999 fix)
git worktree remove "$WORKTREE_PATH"  # only if superpowers owns it
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除工作树 → 删除分支。旧的 skill 在移除工作树之前删除分支（会失败，因为工作树仍然引用分支）。先移除工作树的朴素修复也是错误的——如果合并随后失败，工作目录就消失了，更改丢失。

**选项 2（创建 PR）：**

推送分支，创建 PR。不要清理工作树——用户需要它用于 PR 迭代。（Bug #940 修复：移除矛盾的"Then: Cleanup worktree"文字。）

**选项 3（保持原样）：** 不操作。

**选项 4（丢弃）：** 需要键入"discard"确认。然后移除工作树（如果 superpowers 拥有它），强制删除分支。

#### 步骤 5：清理（更新）

```
if GIT_DIR == GIT_COMMON:
    # Normal repo, no worktree to clean up
    done

if worktree path is under .worktrees/ or worktrees/:
    # Superpowers created it — we own cleanup
    cd to main repo root       # Bug #238 fix
    git worktree remove <path>

else:
    # Harness created it — hands off
    # If platform provides a workspace-exit tool, use it
    # Otherwise, leave the worktree in place
```

清理仅对选项 1 和 4 运行。选项 2 和 3 始终保留工作树。（Bug #940 修复。）

**过期工作树修剪：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自我修复步骤。工作树目录可能被带外删除（例如，由 harness 清理、手动 `rm` 或 `.claude/` 清理），留下导致混淆错误的过期注册。一行，防止静默腐烂。（来自 PR #1072 的想法。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者目前在其集成部分将 `using-git-worktrees` 列为 REQUIRED。改为：

> `using-git-worktrees` — Ensures isolated workspace (creates one or verifies existing)

Skill 本身现在处理同意（步骤 0.5）和检测（步骤 0），因此调用 skills 不需要把关或提示。

#### `writing-plans`

移除过时的声明"should be run in a dedicated worktree (created by brainstorming skill)"。Brainstorming 是一个设计 skill，不创建工作树。工作树提示在执行时通过 `using-git-worktrees` 进行。

### 4. 平台中立指令文件引用

工作树相关 skill 中所有硬编码的 `CLAUDE.md` 实例被替换为：

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

这适用于步骤 1b 中的目录偏好检查。

## 捆绑的 Bug 修复

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | 选项 2 文字说"Then: Cleanup worktree (Step 5)"但快速参考说保留它。步骤 5 说"For Options 1, 2, 4"但常见错误说"Options 1 and 4 only." | 从选项 2 移除清理。步骤 5 仅适用于选项 1 和 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除工作树之前删除分支。由于工作树仍然引用分支，`git branch -d` 可能失败。 | 重新排序为：合并 → 验证测试 → 移除工作树 → 删除分支。在移除任何内容之前，合并必须成功。 | finishing SKILL.md |
| #238 | 如果 CWD 位于正在移除的工作树内部，`git worktree remove` 静默失败。 | 添加 CWD 保护：在 `git worktree remove` 之前 `cd` 到主仓库根目录。 | finishing SKILL.md |

## 已解决问题的状态

| Issue | 解决方案 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | 步骤 0.5 中的 opt-in 同意 |
| #918 | 步骤 0 检测 + 步骤 1.5 完成检测 |
| #1009 | 由步骤 1a 解决——agent 使用原生工具（例如 `EnterWorktree`），其在 harness 原生路径创建。取决于步骤 1a 的工作情况；参见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立指令文件引用 |
| #279 | 通过检测与委托解决——原生路径被尊重，因为我们不覆盖它们 |
| #574 | **已推迟。** 此 spec 中没有涉及 bug 所在的 brainstorming skill。完整修复（向 brainstorming 的检查清单添加工作树步骤）是第四阶段。 |

## 风险

### 步骤 1a 是承载假设——已解决

步骤 1a——agent 优先使用原生工作树工具而非 git 回退——是整个设计所依赖的基础。如果 agent 忽略步骤 1a 并在有原生支持的 harness 上落入步骤 1b，检测与委托就完全失败。

**状态：** 此风险在实施期间已显现。原始的抽象步骤 1a（"你知道你自己的工具包"）在 Claude Code 上以 2/6 失败。TDD 关卡按设计工作——它在任何 skill 文件被修改之前就捕获了失败，防止了损坏的发布。三次 REFACTOR 迭代确定了根本原因（agent 锚定在具体命令上、工具描述护栏覆盖 skill 指令）并产生了在 GREEN 和 PRESSURE 测试中以 50/50 验证的修复。详情见上述步骤 1a 设计说明。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一具有 agent 可调用的会话中工作树工具（`EnterWorktree`）的 harness。其他所有工具要么在 agent 启动前创建工作树（Codex App、Gemini CLI、Cursor），要么没有原生工作树支持（Codex CLI、OpenCode）。步骤 1a 是向前兼容的：当其他 harness 添加 agent 可调用的工作树工具时，agent 会将它们与命名的示例匹配并使用它们，无需更改 skill。

| Harness | 当前工作树模型 | Skill 机制 | 已测试 |
|---------|----------------------|-----------------|--------|
| Claude Code | Agent 可调用 `EnterWorktree` | 步骤 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具（仅 shell） | 步骤 1b git 回退 | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` 标志，无 agent 工具 | 如果带标志启动则步骤 0，否则步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`gemini -p`） |
| Cursor Agent | 用户面对 `/worktree`，无 agent 工具 | 如果用户激活则步骤 0，否则步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`cursor-agent -p`） |
| Codex App | 平台管理，detached HEAD，无 agent 工具 | 步骤 0 检测现有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无 agent 工具 | 步骤 1b git 回退 | 未测试（无 CLI 访问） |

**残余风险：**
1. 如果 Anthropic 更改 `EnterWorktree` 的工具描述使其更加严格（例如，"不要基于 skill 指令使用"），同意桥接就会被破坏。值得提交 issue 请求工具描述适应 skill 驱动的调用。
2. 当其他 harness 添加 agent 可调用的工作树工具时，它们可能使用不在步骤 1a 列表中的名称。随着新工具的出现，列表应更新。通用措辞（"a worktree or workspace-isolation tool"）提供了一些前瞻覆盖。

### 基于来源的启发式

`.worktrees/` 或 `worktrees/` = 我们的，其他任何 = 不碰`启发式对当前所有 harness 都有效。如果未来某个 harness 采用这些项目本地目录之一作为其约定，我们将遇到误报（superpowers 试图清理 harness 拥有的工作树）。类似地，如果用户在没有 superpowers 的情况下手动运行 `git worktree add .worktrees/experiment`，我们会错误地声称所有权。两者的风险都很低——每个 harness 都使用品牌化路径，手动 `.worktrees/` 创建可能性不大——但值得记录。

### Detached HEAD 完成

为 detached HEAD 工作树提供的精简菜单（无合并选项）对于 Codex App 的沙箱模型是正确的。如果用户因其他原因处于 detached HEAD，精简菜单仍然合理——不从 detached HEAD 合并就无法合并，而不先创建分支。

## 实施说明

两个 skill 文件都包含超出核心步骤的部分，在实施期间需要更新：

- **Frontmatter**（`name`、`description`）：更新以反映检测与委托行为
- **Quick Reference 表**：重写以匹配新的步骤结构和 bug 修复
- **Common Mistakes 部分**：更新或移除引用旧行为的条目（例如，"Skip CLAUDE.md check"现在是错误的）
- **Red Flags 部分**：更新以反映新的优先级（例如，"当步骤 0 检测到现有隔离时，永远不要创建工作树"）
- **Integration 部分**：更新 skills 之间的交叉引用

Spec 描述了*什么变了*；实施计划将指定对这些次要部分的确切编辑。

## 未来工作（不在此 spec 中）

- **第三阶段剩余：** `$TMPDIR` 目录选项（#666）、缓存的设置文档和环境继承（#299）
- **第四阶段：** PreToolUse hooks 用于路径强制（#1040）、每工作树环境约定（#597）、brainstorming 检查清单工作树步骤（#574）、多仓库文档（#710）
