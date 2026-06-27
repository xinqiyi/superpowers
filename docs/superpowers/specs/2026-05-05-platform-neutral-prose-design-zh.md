# 平台中立散文——A 阶段设计

## 背景

Superpowers 交付到多个 agent 运行时（Claude Code、Codex、Cursor、OpenCode、Copilot CLI、Gemini CLI）。Skill 内容和支持文档首先为 Claude Code 编写，并在任何运行时的 agent 都适用的地方使用"Claude"。OpenAI 的 vendored fork（openai/plugins#217）尝试了全面重写，但在某些地方存在明显错误——重写了历史归属路径、模型名称和平台特定的安装说明——我们希望避免那个错误，同时仍然在"Claude"确实只是附带提到的地方移除以平台为中心的散文。

完整工作按引用类别分为多个阶段。**此 spec 仅涵盖 A 阶段：** 在非平台特定语境中提及"Claude"的通用第三人称散文。后续阶段（配置文件引用、营销文案、工具名称引用）不在此范围内，将有自己的 specs。

## 范围内

通用散文在以下位置提及"Claude"：

- `skills/*/SKILL.md` 和活跃 skill 目录中的支持 `.md` 文件
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md`（仅当提及是通用散文，而非平台营销）

加上一个创造的术语重命名：**Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)** 在 `skills/writing-skills/SKILL.md` 中。

## 范围外

- **平台/运行时声明** —— "In Claude Code："、安装说明、工具映射引用。（D 阶段候选。）
- **配置文件引用** —— CLAUDE.md、AGENTS.md、GEMINI.md 优先级列表和"where to put project conventions"提示。（B 阶段。）
- **工具名称引用** —— `Skill`、`Bash`、`Read`、`Task`、`TodoWrite`。Skills 使用 Claude Code 的工具词汇编写；现有的 `references/{codex,copilot,gemini}-tools.md` 文件映射它们。（在此 spec 编写时，计划是推迟或跳过这些。E 阶段最终做了它们——将工具名称替换为跨活跃 skills 的动作语言，并围绕相同词汇统一平台工具引用。）
- **README 中的营销文案** —— "Superpowers for Claude Code"、以平台命名的安装部分。（C 阶段。）
- **历史产物** —— `docs/plans/*.md`、`docs/superpowers/specs/*.md`、`CREATION-LOG.md`。这些是带日期、时间点的文档；重写它们就是重写历史。
- **模型标识符** —— Claude Haiku / Sonnet / Opus。这些是真实的产品名称。
- **文件名 / URL 引用** —— `CLAUDE.md`、`claude.com`、`claude-plugin/`、`~/.claude/` 下的路径。
- **`anthropic-best-practices.md` 文件名** —— 即使我们重写其中的散文，文件仍以其来源命名。

## 替换风格

使用在英语中读起来自然的组合：

- **第二人称——"your agent"** 当针对 skill 作者关于*他们的*运行时
  - "your agent reads the description"
- **第三人称——"the agent" / "agents" / "an agent"** 当一般性描述系统行为时
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

选择适合周围句子的形式；不要以牺牲别扭措辞为代价强制一致性。在自然时使用复数（"future agents"、"agents read"）而非总说"the agent"。

### 保留为"Claude"的例外

- 模型名称：Claude Haiku、Claude Sonnet、Claude Opus
- 文件名和 URL：`CLAUDE.md`、`claude.com`、`~/.claude/`
- 品牌平台名称"Claude Code"无论其指代运行时（在后续阶段处理）

### 创造的术语重命名

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - 出现在 `skills/writing-skills/SKILL.md` 中作为章节标题和附近散文中。重命名标题、缩写和任何文件内交叉引用。

## 受影响的文件

基于过滤排除例外后的 `grep` 的大致计数：

| 文件 | 通用散文提及 |
|------|------------------------|
| `skills/writing-skills/SKILL.md` | ~12（包括 CSO 标题 + 正文） |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1——文件名保留（它是 CLAUDE.md 测试产物）；"Variant C: Claude.AI Emphatic Style"标题也保留（它是命名特定风格的标签） |
| `README.md` | ~1 |

在实施期间通过重新运行过滤后的 grep 确认最终列表。

## 提交计划

按顺序的四个原子提交：

1. **在 `skills/writing-skills/SKILL.md` 中重命名 CSO → SDO。** 机械，隔离，如果我们对该术语改变主意则易于恢复。
2. **活跃 skills 散文** —— 跨 `skills/*/SKILL.md` 和支持 `.md` 的通用"Claude" → "agent"形式，排除 `anthropic-best-practices.md`。
3. **`anthropic-best-practices.md` 散文** —— 相同的替换规则。单独的提交，因为该文件是外部文档的 vendored 改编；隔离更改使未来与上游的协调更易于阅读。
4. **README.md 散文** *（仅当过滤后仍有通用散文提及时）。如果为空则跳过。*

每个提交消息命名阶段（"Phase A"）和切片（"rename CSO to SDO"、"agent prose in active skills"等），使系列自文档化。

## 验证

每个提交后：

- `grep -rn "Claude" <touched-paths>` —— 每个剩余匹配必须属于已记录的例外（模型名称、文件名、URL、"Claude Code"平台名称、历史产物）。
- 从头到尾阅读修改的文件——替换不应破坏句子流畅性、代词一致或列表并列。
- 无需运行测试；这只是散文。

最终提交后：

- 在实时会话中浏览每个修改的 skill 以确认没有内容读起来别扭。

## 非目标

- 不要更改行为、结构、标题（CSO→SDO 除外）、示例、代码块或 YAML frontmatter。
- 不要引入新章节、提示或兼容性说明。
- 在编辑时不要"改进"超出替换范围的散文。
