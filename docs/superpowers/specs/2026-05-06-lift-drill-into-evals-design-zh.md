# 将 drill 提升到 superpowers 作为 `evals/`——设计

## 背景

Drill 是一个 Python skill 合规基准，位于自己的仓库 `obra/drill` 中。它驱动真实的 tmux 会话，运行 LLM actor 作为模拟用户，在结果记录上运行 LLM 验证器，并报告每个场景的通过/失败。它支持 Claude Code、Codex、Gemini CLI 以及（根据最近的提交）OpenCode 和 Copilot CLI。

Drill 已经是 superpowers 的*事实上的*评估框架。Drill 仓库中的 PRI-1397 提交系列将约 22 个 superpowers bash 测试提升为 drill 场景，而最近的 superpowers 提交（`a2292c5`）明确移除了一个冗余的 bash 测试，消息为*"replaced by drill behavioral coverage"*。迁移势头存在；此 spec 完成它。

此工作将 drill 移入 superpowers 下的 `evals/`，在逐文件验证 drill 场景覆盖率后删除冗余的 bash 测试，并更新文档以便贡献者找到新结构。

## 目标

1. `evals/` 是 superpowers 中的规范评估框架——完整的 drill 源代码、场景、固定数据、提示、后端配置和测试。
2. 在 `superpowers/tests/` 中已被逐个验证为 100% 被 drill 场景覆盖的 bash 测试被删除；其余保留。
3. `tests/`（plugin 基础设施：bash + node + python 集成测试）和 `evals/`（使用 actor + 验证器的 LLM 行为测试）之间的区分是有意义的并被记录。
4. 顶级文档（`README.md`、`CLAUDE.md`、`docs/testing.md`）将贡献者指向正确的位置。
5. 独立的 `obra/drill` 仓库继续存在（此 PR 不触及它），并在此 PR 合并后作为单独的步骤归档。

## 非目标

- **CI 集成。** 此处仅手动。自然的后续是"分级"：每个 PR 上的快速子集，每晚 + 按需的全量扫描。这需要 API 预算决策、GitHub Actions secrets 以及安装了 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLI 的运行器映像。范围外。
- **场景与 skills 的共置。** 场景保持集中在 `evals/scenarios/`。如果我们后来决定每个 skill 应拥有自己的场景，那是路径查找和重命名操作；YAML 格式不变。
- **重命名内部 Python 包**（`drill` → `evals`）。目录是 `evals/`（对用户而言）；Python 包保留其 `drill` 名称以保持差异小。在 `evals/README.md` 中用简短说明解释。
- **Drill 仓库归档。** 此 PR 不触及 `obra/drill`。合并后，drill 仓库被手动归档（GitHub 上只读，README 指针指向 `obra/superpowers/evals/`）。
- **将 `tests/claude-code/analyze-token-usage.py` 提升到 `evals/bin/`。** 有用的工具，非测试代码。可稍后移动；此 PR 不要求。

## 分支策略

从 `dev` 创建分支 `f/evals-lift`。此工作独立于开放的 `f/cross-platform` PR——除了可能是 `README.md` 之外无共享文件更改，而 `README.md` 足够小，可在合并时解决冲突（如果有）。

## 移动后的架构

```
superpowers/
  evals/                              ← 新增（完整 drill 副本）
    pyproject.toml                    （Python 3.11，uv 管理）
    uv.lock
    .gitignore                        （drill 自己的；results/、.venv/、.env）
    README.md                         （原 drill 的 README；安装说明已更新）
    CLAUDE.md                         （原 drill 的 CLAUDE.md；路径已更新）
    docs/
      design.md                       （drill 的设计——逐字保留，从此 spec 交叉链接）
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            （Python 包；名称保留；cli、engine、actor、verifier 等）
    backends/                         （claude-*.yaml、codex.yaml、gemini.yaml）
    scenarios/                        （32+ 个 YAML 场景）
    setup_helpers/                    （15 个 Python 辅助函数；create_base_repo、sdd_*、spec_*、worktree 等）
    fixtures/                         （template-repo、sdd-go-fractals、sdd-svelte-todo）
    prompts/                          （actor.md、verifier.md）
    bin/                              （断言辅助脚本：tool-called、tool-count 等）
    tests/                            （drill 自己的 pytest 套件）

  tests/                              ← bash 测试默认保留
    brainstorm-server/                ← 保留（brainstorm-server JS 代码的 node 测试）
    opencode/                         ← 保留（plugin 加载测试）
    codex-plugin-sync/                ← 保留（同步验证）
    claude-code/                      ← 大部分保留——参见删除关卡
    explicit-skill-requests/          ← 保留，除非验证为已替换
    skill-triggering/                 ← 保留，除非验证为已替换
    subagent-driven-dev/              ← 保留，除非验证为已替换

  docs/
    testing.md                        ← 已更新（拆分为"Plugin tests" + "Skill behavior evals"）
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 本 spec

  README.md                           ← 小型 Contributing 部分指针指向 evals/
  CLAUDE.md                           ← 一行"Eval harness lives at evals/"指针
```

此 PR 后，`tests/` 和 `evals/` 目录服务于明显不同的角色：

- **`tests/`** —— plugin 的非 LLM 代码是否工作？brainstorm-server JS 代码、OpenCode plugin 加载、codex-plugin-sync 同步验证的单元和集成测试。Bash + node + python。
- **`evals/`** —— agent 在真实 LLM 会话上行为是否正确？带有 actor + 验证器的 drill 场景。仅 Python，运行真实 tmux 会话。

## 删除关卡（每个 bash 测试）

bash 测试*仅当* drill 场景可验证地覆盖其所做的每个断言时才会被删除。实施计划记录了每文件的此验证：读取 bash 测试，列出其检查，找到 drill 场景，确认每个检查都有匹配的 `verify.assertions` 或 `verify.criteria` 条目。如果即使一个检查缺失，选项是扩展 drill 场景或保留 bash 测试。默认为保留。

**暂定覆盖率映射**（基于提交消息；在任何删除前需要逐文件验证）：

| Bash 测试 | 声称的 drill 替代 | 覆盖状态 |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*`（6 个提示文件） | `triggering-*.yaml`（6 个场景） | 候选——删除前逐提示验证 |
| `tests/skill-triggering/run-test.sh`、`run-all.sh` | n/a（运行器，非测试） | **保留**——运行器脚本 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 需要验证——drill 尚无明显对应 | 可能**保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 需要验证——drill 尚无对应 | 可能**保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 部分 → `mid-conversation-skill-invocation.yaml` | 候选——逐脚本验证 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | 无 drill 场景覆盖 Haiku 特定行为 | **保留** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 无 drill 场景覆盖多轮构建 | **保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-test.sh`、`run-all.sh` | n/a（运行器） | **保留** |
| `tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`、`sdd-svelte-todo.yaml` | 候选——删除前验证（这些包含关于测试套件通过的真实断言） |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml`（YAGNI 子集） | **部分**——bash 测试还断言 ≥3 次提交 / `npm test` 通过 / 运行 `analyze-token-usage.py`。Drill 场景断言禁止的导出 + 审查者作为关卡。大多不重叠——几乎肯定**保留 + 扩展 drill 场景**。 |
| `tests/claude-code/test-subagent-driven-development.sh` | 元/文档测试（要求 agent *描述* SDD）；无 drill 场景覆盖描述测试 | **保留**，除非添加 drill 场景 |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 候选——删除前验证 |
| `tests/claude-code/test-helpers.sh`、`run-skill-tests.sh`、`analyze-token-usage.py` | n/a（工具，非测试） | **保留**——库/工具 |

## 验证协议（subagent 把关）

实施计划中的每个更改在提交前由独立的 subagent 交叉检查。

| 更改类别 | Subagent 验证 |
|----------------|----------------------|
| 每个 bash 测试删除 | 调度一个 subagent，带：(a) bash 测试文件内容，(b) 候选 drill 场景 YAML，(c) 提示：*"列出 bash 测试所做的每个断言。列出 drill 场景中的每个验证条目。对于每个 bash 断言，找到匹配的 drill 检查或报告为不匹配。输出逐断言表。"* Subagent 的输出是关卡——仅当每个 bash 断言都有匹配时才删除。 |
| 初始 `evals/` 副本 | Subagent 验证：(a) 被复制的 drill SHA 记录在提升提交消息中，以便来源可审计；(b) 每个文件的 **SHA-256 校验和** 与 drill 仓库匹配（不仅仅是文件计数）；(c) 排除的路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、任何 `.private-journal/`）不存在于 `evals/` 中；(d) 所有后端 YAML 引用的路径在移动后存在；(e) `pyproject.toml`、`uv.lock`、`.gitignore` 完整。 |
| Drill 自己的 pytest 套件 | Subagent 在路径默认更改后运行 `cd evals && uv run pytest`。Drill 在 `evals/tests/` 中自带 pytest 套件，包括测试 `SUPERPOWERS_ROOT` 环境变量行为的 `test_backend.py`——这些测试必须更新以匹配辅助函数并继续通过。 |
| 删除后的引用清理 | Subagent grep 整个 superpowers 树（排除 `node_modules/`、`.venv/` 和 `evals/`）以查找对已删除 bash 测试路径的引用。搜索目标：`docs/`、`docs/superpowers/plans/`、`RELEASE-NOTES.md`、`CLAUDE.md`、`GEMINI.md`、`AGENTS.md`、`README.md`、`.github/`、`scripts/`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`lefthook.yml`。任何命中要么被更新，要么暴露缺失的依赖。 |
| 路径默认更改（`SUPERPOWERS_ROOT` 默认） | Subagent 在路径更改后运行至少一个便宜的 drill 场景（例如 `triggering-test-driven-development`）并确认它仍然通过。真实验证，而不仅仅是代码审查。 |
| 最终 PR 前对抗性审查 | 两个并行 subagent，采用"5 分给找到最多合法问题的人"框架——与跨平台 PR 使用的相同协议。验证源代码和行为。 |

每个 subagent 任务在实施计划中获得自己的要点，带有明确的输入和通过标准。Subagent 的输出在相关提交消息中总结（"Subagent verification: …"），以便跟踪可审计。

## 具体路径/配置编辑

**在编写此 spec 前已验证。** `drill/cli.py` 定义 `PROJECT_ROOT = Path(__file__).parent.parent`。移动后，`cli.py` 位于 `evals/drill/cli.py`，因此 `PROJECT_ROOT` 解析为 `evals/`，且 `PROJECT_ROOT.parent` 解析为 superpowers 仓库根目录。那是 `SUPERPOWERS_ROOT` 默认应取的值。

**YAML 替换审计。** 只有四个 `claude*.yaml` 后端配置将 `${SUPERPOWERS_ROOT}` 插值到 `args` 中（用于 `--plugin-dir` 标志）；`codex.yaml` 和 `gemini.yaml` 仅在 `required_env` 中列出 `SUPERPOWERS_ROOT`（被 `engine.py:233` / `setup.py:25` 在 pre/post-run hooks 中的 `os.environ["SUPERPOWERS_ROOT"]` 查找消耗）。辅助函数的 `os.environ` 变异覆盖两个代码路径。

| 文件 | 当前 | 之后 |
|------|---------|-------|
| `drill/cli.py` | 在模块导入时 `load_dotenv(PROJECT_ROOT / ".env")`；与 `SUPERPOWERS_ROOT` 无关 | 在 `load_dotenv` 之后，调用新辅助函数 `_set_superpowers_root_default()`，当且仅当尚未设置时设置 `os.environ["SUPERPOWERS_ROOT"]` 为 `str(PROJECT_ROOT.parent)`。顺序：`load_dotenv` → 设置默认 → click 组定义。 |
| `drill/engine.py:233`、`drill/setup.py:25` | 直接 `os.environ["SUPERPOWERS_ROOT"]` 访问（如果未设置则 KeyError） | 不变。CLI 启动钩子保证在 engine/setup 执行时环境变量已设置。 |
| `backends/claude*.yaml`（5 个文件） | `${SUPERPOWERS_ROOT}` 在 `args` 中替换为 `--plugin-dir` | 不变。YAML 替换在后端加载时读取 `os.environ`，这发生在 CLI 启动之后。 |
| `backends/codex.yaml`、`backends/gemini.yaml` | `SUPERPOWERS_ROOT` 仅在 `required_env` 中 | 从 `required_env` 中删除（辅助函数提供它）。`claude*.yaml` 保留 `required_env` 以实现向后兼容（环境变量作为覆盖仍有效）。 |
| `evals/tests/test_backend.py` | 测试断言 `SUPERPOWERS_ROOT` 在 `required_env` 列表中，加上路径解析测试 | 更新测试以匹配新契约：辅助函数提供的默认值，环境变量覆盖仍然有效，`required_env` 对 codex/gemini 不再是必需的。 |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | 删除 export 行；说明环境变量自动默认为 `evals/` 的父目录；提及唯一需要的设置是 `ANTHROPIC_API_KEY`（或 `OPENAI_API_KEY` / Gemini 认证）。 |
| `evals/CLAUDE.md` | 相同 | 相同 |
| `evals/.gitignore` | drill 现有的模式（`results/`、`.venv/`、`__pycache__/`、`.env`、`*.pyc`、`*.egg-info/`、`dist/`、`build/`、`.claude/`） | 逐字复制。模式相对于文件位置，因此它们在 `evals/` 下正确应用。 |
| `evals/lefthook.yml` | drill 自带 `lefthook.yml`，定义 `pre-commit: uv run ruff check && uv run ty check` | 移动到 `evals/lefthook.yml`。要么 (a) 在 superpowers 根目录安装 lefthook 并使其联合到 `evals/lefthook.yml`，或 (b) 记录贡献者手动运行 `cd evals && lefthook run pre-commit`。**实施中的决定：为简单起见选择选项 (b)**——superpowers 的顶级工作流不变。 |

`.env` 放置：保留 `evals/.env`（已 gitignore）。贡献者从那里加载它或在 shell 环境中设置 `ANTHROPIC_API_KEY`。

**需要小型添加的顶级 superpowers 文件：**

- `superpowers/.gitignore`：添加 `evals/results/`、`evals/.venv/`、`evals/.env`（双重保障；evals/.gitignore 已经在本地覆盖了这些）。
- `superpowers/CLAUDE.md`：添加一行指针"Eval harness lives at `evals/` — see `evals/README.md`"，以便 agent 发现它。
- `superpowers/docs/testing.md`：拆分为"## Plugin tests"（现有的 tests/ 内容，修剪已删除测试的引用）和"## Skill behavior evals"（一段总结 + 指向 `evals/` 的指针）。
- `superpowers/README.md`：在 Contributing 部分添加一行指向 `evals/` 用于 skill 行为测试。

## 迁移顺序

每个步骤是一个单独的提交（或小组提交）。步骤 2 是最大的单个提交（逐字的 drill 副本）；后续步骤很小且原子化。

```
1. 从 `dev` 创建分支（f/evals-lift）

2. 将 drill 仓库复制到 evals/（单个提交，易于恢复）
   ├─ 在复制时记录 drill SHA → 提交消息
   ├─ 使用 `rsync -a --exclude=.git --exclude=.venv --exclude=results
   │  --exclude=.env --exclude=__pycache__ --exclude='*.egg-info'
   │  --exclude=.private-journal /path/to/drill/ evals/`
   │  （选择 rsync 而非 `cp -r` 是因为显式排除；通过
   │  `find evals -name '.git' -type d` 返回无结果验证）
   ├─ Subagent 关卡：每个非排除文件的逐文件 SHA-256 校验和与 drill 仓库匹配；排除的路径不存在于 evals/
   └─ 冒烟检查：`cd evals && uv sync` 成功（仅证明安装；非行为测试）

3. 更新路径默认值
   ├─ 向 drill/cli.py 添加 _set_superpowers_root_default() 辅助函数
   ├─ 在 load_dotenv 之后、click 组定义之前连接它
   ├─ 更新 evals/README.md 和 evals/CLAUDE.md（删除 SUPERPOWERS_ROOT 安装步骤）
   ├─ 从 codex.yaml/gemini.yaml 的 required_env 中删除 SUPERPOWERS_ROOT
   │  （保留在 claude*.yaml 中作为覆盖）
   └─ 更新 evals/tests/test_backend.py 以匹配新契约

4. 从新位置验证（两项检查）
   ├─ 运行 drill 自己的 pytest：`cd evals && uv run pytest` — 必须通过
   └─ 运行便宜的 drill 场景：`cd evals && uv run drill run
      triggering-test-driven-development -b claude` — 必须通过。
      真实的行为验证，而不仅仅是代码审查。

5. Bash 测试删除阶段——逐文件，带 subagent 关卡
   对于候选删除列表中的每个文件：
   a. Subagent 比较 bash 测试断言与 drill 场景验证块
   b. 通过标准：每个 bash 断言都有匹配的 drill 检查
   c. 如果通过 → 删除 bash 测试文件（每文件或每一致组一个提交）
   d. 如果失败 → 要么扩展 drill 场景（单独的提交 + 验证）或保留 bash 测试（不提交）

6. 过期引用清理
   ├─ Subagent grep superpowers 树（排除 node_modules/、.venv/、
   │  evals/）查找已删除文件路径
   ├─ 搜索目标：docs/、docs/superpowers/plans/、RELEASE-NOTES.md、
   │  CLAUDE.md、GEMINI.md、AGENTS.md、README.md、.github/、scripts/、
   │  .opencode/INSTALL.md、.codex-plugin/INSTALL.md、lefthook.yml
   ├─ 更新活跃引用（例如 docs/testing.md、README.md install）
   └─ docs/superpowers/plans/*.md 和 RELEASE-NOTES.md 中的历史引用
      被保留并附上简短注释
      ("(test removed; behavior covered by drill scenario X)") 而非
      重写——这些是带日期的人工产物，而非活文档。

7. 顶级文档
   ├─ docs/testing.md 拆分
   ├─ CLAUDE.md 指针
   └─ README.md Contributing 部分

8. 重新运行冒烟检查（回归关卡）
   ├─ `cd evals && uv run pytest`
   └─ `cd evals && uv run drill run triggering-test-driven-development -b claude`

9. 最终对抗性审查
   └─ 两个并行 subagent，完整差异，"5 分给找到最多合法问题的人"
      框架。在推送前处理发现。

10. 推送分支 + 针对 dev 打开 PR
    └─ PR 描述包括：复制时固定的 drill SHA、归档操作项
       （"合并后：归档 obra/drill，添加 README 指针指向
       obra/superpowers/evals/"）、每删除文件覆盖率收据。
```

## 验证（实施后）

实施计划必须显示：

- 步骤 2 后 `evals/` 中存在所有非排除的 drill 源文件（subagent **逐文件 SHA-256 校验和差异** 与 `obra/drill@<recorded-sha>`）。
- 排除的路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、`.private-journal/`）不存在于 `evals/` 中。
- 步骤 2 的提交消息记录了 drill 源 SHA。
- `cd evals && uv sync` 在未设置 `SUPERPOWERS_ROOT` 的情况下成功。
- `cd evals && uv run pytest` 通过（drill 自己的 pytest 套件）。
- `cd evals && uv run drill list` 返回与记录 SHA 处的独立 drill 仓库相同的场景计数。
- `cd evals && uv run drill run triggering-test-driven-development -b claude` 通过（证明路径默认值端到端工作）。
- 对于每个删除的 bash 测试：提交消息中的 subagent 验证表显示每个断言映射到 drill 检查。
- 搜索已删除文件路径在活跃 superpowers 文档中返回零命中（步骤 6 后）；`docs/superpowers/plans/*.md` 和 `RELEASE-NOTES.md` 中的历史引用被注释，而非重写。
- `docs/testing.md` 同时具有"Plugin tests"和"Skill behavior evals"部分。
- Drill 仓库的历史未触及；`obra/drill` 不受此 PR 影响。
- PR 描述命名了合并后归档 `obra/drill` 的操作项。

## 未决问题

无。所有澄清决定已做出：

| 问题 | 决定 |
|----------|----------|
| Drill 在 superpowers 中的位置？ | `evals/`（从 drill 重命名）；独立仓库作为单独步骤归档 |
| 冗余 bash 测试的命运？ | 逐文件删除，带 subagent 覆盖验证；默认保留 |
| 场景布局？ | 集中在 `evals/scenarios/` |
| Python 工具链放置？ | 自包含在 `evals/` |
| CI 集成？ | 此 PR 仅手动；记录了未来路径 |
| 迁移机制？ | 普通复制；drill 仓库的历史保留在归档仓库中，而非树内 |
| 内部 Python 包名称？ | 保留为 `drill`（目录是 `evals/`） |
| 分支策略？ | 独立于 `dev`（不堆叠在 `f/cross-platform` 上） |
