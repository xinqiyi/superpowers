# 平台中立 README 排序——C 阶段设计

## 背景

A 和 B 阶段（参见 `2026-05-05-platform-neutral-prose-design.md` 和 `2026-05-05-platform-neutral-config-refs-design.md`）已经在 README 中中和了通用的 Claude 散文和配置文件引用。剩余的平台倾向信号是布局：README 的两个平台列表将 Claude Code 放在首位，且其他地方并非严格按字母顺序排列。

此阶段修复排序。无散文更改。

## 范围内

1. **快速入门平台列表**（`README.md:7`）——支持的 harnesses 的内联链接列表
2. **安装部分排序**（`README.md:35-152`）——每个 harness 的安装子部分

## 范围外

- 散文、市场名称、plugin ID、URL——所有事实正确，不变。
- Claude Code 部分的视觉权重（有两个子部分——官方 Anthropic 市场和 Superpowers 市场）。两者都是真实的安装路径；折叠它们会隐藏准确信息。
- 每个安装块内的章节标题和内容——仅块排序更改。

## 替换

两个列表重新排序为严格字母顺序：

| 旧顺序 | 新顺序 |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

三次移动：Codex App 与 Codex CLI 交换；Cursor 上移两位；GitHub Copilot CLI 上移一位。

Claude Code 仍因字母顺序保留第一位（`Cl…` 在 `Co…` 之前）。

## 提交计划

一个原子提交涵盖两个列表，因为更改一个而不更改另一个会在快速入门和安装部分之间造成不一致。

## 验证

- 快速入门锚点（`#claude-code`、`#codex-app` 等）仍然解析到现有的 `### …` 标题——无标题重命名。
- 每个安装子部分的正文在前后字节一致；仅位置更改。
- `git diff README.md` 仅显示部分移动，无内容编辑。
