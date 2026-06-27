# 在 OpenCode 中使用 Superpowers

在 [OpenCode.ai](https://opencode.ai) 中使用 Superpowers 的完整指南。

## 安装

将 superpowers 添加到 `opencode.json`（全局或项目级别）的 `plugin` 数组中：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。该 plugin 会通过 OpenCode 的 plugin 管理器安装，并注册所有 skills。

通过询问以下内容来验证："Tell me about your superpowers"

OpenCode 使用自己的 plugin 安装机制。如果你同时使用 Claude Code、Codex 或其他 harness，请为每个 harness 单独安装 Superpowers。

### 从旧的基于符号链接的安装迁移

如果你之前使用 `git clone` 和符号链接安装了 superpowers，请移除旧配置：

```bash
# 移除旧的符号链接
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选：移除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 从 opencode.json 中移除为 superpowers 添加的 skills.paths 配置
```

然后按照上述安装步骤操作。

## 使用方法

### 查找 Skills

使用 OpenCode 原生的 `skill` 工具列出所有可用的 skills：

```
use skill tool to list skills
```

### 加载 Skill

```
use skill tool to load brainstorming
```

### 个人 Skills

在 `~/.config/opencode/skills/` 中创建你自己的 skills：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目 Skills

在项目内的 `.opencode/skills/` 中创建项目特定的 skills。

**Skill 优先级：** 项目 skills > 个人 skills > Superpowers skills

## 更新

OpenCode 通过基于 git 的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会将解析后的 git 依赖固定在 lockfile 或缓存中，因此重启可能不会获取最新的 Superpowers 提交。如果更新未生效，请清除 OpenCode 的包缓存或重新安装 plugin。

要固定特定版本，请使用分支或标签：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理

该 plugin 完成两件事：

1. 通过 `experimental.chat.messages.transform` hook 注入 bootstrap 上下文，使每次对话都具备 superpowers 感知能力。
2. 通过 `config` hook 注册 skills 目录，使 OpenCode 能够发现所有 superpowers skills，无需符号链接或手动配置。

### 工具映射

Skills 使用行为描述而非命名特定运行时的工具。在 OpenCode 上，这些行为对应以下工具：

- "创建 todo" / "在 todo 列表中标记完成" → `todowrite`
- `Subagent (general-purpose):` 模板 → OpenCode 的 `task` 工具，配合 `subagent_type: "general"`（代码库探索使用 `"explore"`）
- "调用 skill" → OpenCode 原生的 `skill` 工具
- "读取文件" → `read`
- "创建文件" / "编辑文件" / "删除文件" → `apply_patch`
- "运行 shell 命令" → `bash`
- "搜索文件内容" / "按名称查找文件" → `grep`, `glob`
- "获取 URL" → `webfetch`

（已根据已安装的 OpenCode CLI 的工具清单验证。）

## 故障排除

### Plugin 未加载

1. 检查 OpenCode 日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 确认 `opencode.json` 中的 plugin 行正确无误
3. 确保你运行的是较新版本的 OpenCode

### Windows 安装问题

某些 Windows OpenCode 构建版本在基于 git 的 plugin 规范方面存在上游安装程序问题，包括 `git+https` URL 的缓存路径以及 Bun 无法找到 `git.exe`（即使在正常终端中可正常工作）。如果 OpenCode 无法安装 plugin，请尝试使用系统 npm 安装并将 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装的包路径：

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### Skills 未找到

1. 使用 OpenCode 的 `skill` 工具列出可用的 skills
2. 检查 plugin 是否已加载（见上方说明）
3. 每个 skill 需要包含有效 YAML frontmatter 的 `SKILL.md` 文件

### Bootstrap 未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` hook
2. 配置更改后重启 OpenCode

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/
