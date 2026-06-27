# 平台中立配置文件引用——B 阶段设计

## 背景

A 阶段（参见 `2026-05-05-platform-neutral-prose-design.md`）将通用的第三人称"Claude"散文替换为 agent 中立的形式。此阶段处理下一个类别：skills 内部对每平台指令文件（CLAUDE.md、AGENTS.md、GEMINI.md）的引用。

该 Plugin 在多个 harness 上运行，每个 harness 读取自己的指令文件。当某个 skill 将 CLAUDE.md 称为唯一文件时，这是一个以 Claude Code 为中心的假设，在 Codex / Gemini CLI / OpenCode 上不成立。

## 范围内

活跃 skills 中的两行具体内容：

1. **`skills/writing-skills/SKILL.md:58`** —— `Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`** —— `"You're absolutely right!" (explicit CLAUDE.md violation)`

## 范围外

- **`skills/using-superpowers/SKILL.md:22, 26`** —— 指令优先级列表。该列表已经包容性地命名了所有三个（CLAUDE.md、GEMINI.md、AGENTS.md），这是正确的：该部分正在做一个真实的声明，关于*什么算作用户指令*在一个多平台 plugin 上。无需更改。
- **历史 / 示例产物**：
  - `skills/systematic-debugging/CREATION-LOG.md` —— 归属路径（`~/.claude/CLAUDE.md`）是历史事实。
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` —— 整个文件是一个测试 CLAUDE.md 内容变体的工作示例。文件名、正文和来自 `testing-skills-with-subagents.md` 的引用都保留；标准化它们会破坏示例。
- **平台工​​具引用** —— D 阶段候选：
  - `skills/using-superpowers/SKILL.md:40`（Gemini CLI 工具映射说明关于 GEMINI.md）
  - `skills/using-superpowers/references/gemini-tools.md`（`save_memory` 持久化到 GEMINI.md）

## 替换规则

每个范围内一行两个不同的调用。

### 规则 1："哪里放项目特定约定"

`writing-skills/SKILL.md:58`：

- **之前：** `Project-specific conventions (put in CLAUDE.md)`
- **之后：** `Project-specific conventions (put in your instructions file)`

使用通用短语而非选择某个文件名。不同的 harness 读取不同的文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），skill 不应假设其中一个。平台工具参考文档（`references/{codex,copilot,gemini}-tools.md`）是命名每个平台首选文件的合适位置。

### 规则 2："（explicit CLAUDE.md violation）"括号

`receiving-code-review/SKILL.md:30`：

- **之前：** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **之后：** `"You're absolutely right!" (explicit instruction-file violation)`

括号在做实际工作——它表明这个短语不仅在风格上不好，而且积极地违反了用户放入其指令文件中的规则。"Instruction file"是涵盖 AGENTS.md / CLAUDE.md / GEMINI.md 集合的自然跨平台术语，保留了原始信号而不选择某个文件名或弱化为"common"。

## 提交计划

按顺序的原子提交：

1. **`writing-skills/SKILL.md`** —— "where to put project conventions"行中 CLAUDE.md → "your instructions file"
2. **`receiving-code-review/SKILL.md`** —— 违规括号中 CLAUDE.md → instruction-file
3. **平台工具参考文档** —— 将每个平台的首选指令文件名（CLAUDE.md、AGENTS.md、GEMINI.md 等）添加到每个 `references/{codex,copilot,gemini}-tools.md`，以便读者可以将"your instructions file"解析为真实文件名。

每个提交消息命名"Phase B"和切片。

## 验证

每个提交后：

- 阅读周围段落以确认语法和含义仍然通顺。
- `grep -n "CLAUDE\.md" <touched-file>` —— 活跃散文中没有剩余匹配（已记录的例外已记录）。

两个提交后：

- `grep -rn "CLAUDE\.md" skills/` 应仅返回已记录的例外（CREATION-LOG、CLAUDE_MD_TESTING 及其入站引用、using-superpowers 中的优先级列表）。

## 非目标

- 不要触碰 `using-superpowers/SKILL.md` 中的优先级列表排序。重新排序 CLAUDE.md / GEMINI.md / AGENTS.md 是美观更改，而非替换，且超出此处范围。
- 不要重命名 `examples/CLAUDE_MD_TESTING.md` 或更改其内容。
- 不要修改 Gemini CLI 特定的工具引用（D 阶段候选）。

## 实施说明

此处编写的 B 阶段涵盖了三个提交和三个非 Claude Code 平台工具引用。实施更进一步：在提交 `8505703` 中为对称性添加了第四个引用 `references/claude-code-tools.md`，因此 Claude Code 的指令文件约定和工具名称列表与其他平台并列，而非隐含在周围的 skill 散文中。该添加未在此 spec 中预料到，但与其意图一致。
