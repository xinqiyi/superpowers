# 为 OpenCode 安装 Superpowers

## 前置条件

- 已安装 [OpenCode.ai](https://opencode.ai)

## 安装

将 superpowers 添加到 `opencode.json`（全局或项目级别）的 `plugin` 数组中：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。该 plugin 将通过 OpenCode 的 plugin 管理器安装并注册所有 skills。

通过提问来验证："Tell me about your superpowers"

OpenCode 使用自己的 plugin 安装机制。如果你同时使用 Claude Code、Codex 或其他 harness，请为每个 harness 分别安装 Superpowers。

## 从旧的 symlink 安装方式迁移

如果你之前通过 `git clone` 和 symlink 方式安装了 superpowers，请移除旧配置：

```bash
# 移除旧的 symlink
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 可选：移除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 如果为 superpowers 添加了 skills.paths，从 opencode.json 中移除
```

然后按照上述安装步骤操作。

## 使用方法

使用 OpenCode 原生的 `skill` 工具：

```
use skill tool to list skills
use skill tool to load brainstorming
```

## 更新

OpenCode 通过 git 支持的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会将解析后的 git 依赖锁定在 lockfile 或 cache 中，因此重启后可能无法获取最新的 Superpowers 提交。如果更新未生效，请清除 OpenCode 的包缓存或重新安装 plugin。

要锁定特定版本：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 故障排除

### Plugin 未加载

1. 检查日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证 `opencode.json` 中的 plugin 配置行
3. 确保你运行的是较新版本的 OpenCode

### Windows 安装问题

某些 Windows 版本的 OpenCode 在 git 支持的包规范方面存在安装问题，包括 `git+https` URL 的缓存路径以及 Bun 无法找到 `git.exe`（即使在普通终端中可正常工作）。如果 OpenCode 无法安装 plugin，请尝试使用系统 npm 安装并将 OpenCode 指向本地包：

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

1. 使用 `skill` 工具查看已发现的 skills
2. 检查 plugin 是否正常加载（见上文）

### 工具映射

Skills 使用操作描述（"create a todo"、"dispatch a subagent"、"read a file"）。在 OpenCode 上，这些操作对应：

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` template → `task` 工具，使用 `subagent_type: "general"`（代码库探索使用 `"explore"`）
- "Invoke a skill" → OpenCode 原生的 `skill` 工具
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`、`glob`
- "Fetch a URL" → `webfetch`

## 获取帮助

- 提交问题：https://github.com/obra/superpowers/issues
- 完整文档：https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
