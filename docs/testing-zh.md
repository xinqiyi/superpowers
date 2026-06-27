# 测试 Superpowers

Superpowers 有两种不同类型的测试，各自位于独立目录中：

- **`tests/`** — 插件的非 LLM 代码是否正常工作？Bash + Node + Python 集成测试，涵盖 brainstorm-server JS、OpenCode plugin 加载、codex-plugin 同步以及分析工具。
- **`evals/`** — agent 在真实的 LLM 会话中行为是否正确？Python harness，使用真实的 tmux 会话驱动 Claude Code / Codex / Gemini CLI，配合 LLM actor 和 verifier 评判技能合规性。

## Plugin 测试

位于 `tests/` 目录下。当前包含：

- `tests/brainstorm-server/` — brainstorm server JS 代码的 Node 测试套件。
- `tests/opencode/` — OpenCode plugin 加载、bootstrap 缓存和工具注册的 Bash 测试。
- `tests/codex-plugin-sync/` — Bash 同步验证。
- `tests/kimi/` — Kimi plugin 清单连接的 Bash/Python 检查。
- `tests/claude-code/test-helpers.sh`、`analyze-token-usage.py` — 其余 Bash 测试使用的工具。
- `tests/claude-code/test-subagent-driven-development.sh` — agent 能否描述 SDD 的测试（无 drill 对应版本；测试的是描述回忆，而非行为）。
- `tests/claude-code/test-subagent-driven-development-integration.sh` — 扩展的 SDD 集成测试，包含 token 分析（drill 覆盖 YAGNI 子集；bash 额外添加提交计数、Claude Code 任务跟踪和 token 遥测断言）。
- `tests/claude-code/test-worktree-native-preference.sh` — worktree skill 的 RED-GREEN-REFACTOR 验证（drill 覆盖 PRESSURE 阶段；bash 也覆盖 RED/GREEN 基线）。
- `tests/explicit-skill-requests/` — Haiku 特定、多轮次和按 skill 名称提示的测试，drill 未覆盖。

通过相关目录下的 `run-*.sh` 或 `npm test` 运行 plugin 测试。

## 技能行为评估

位于 `evals/` 目录下。Drill 是 harness；场景位于 `evals/scenarios/*.yaml`。设置方法参见 `evals/README.md`。快速开始：

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill 场景运行速度较慢（每个 3-30 分钟以上），运行真实的 LLM 会话。它们目前不属于 CI 的一部分；自然的后续方案是分层模型（PR 上运行快速子集，每晚 + 按需运行全量扫描）。
