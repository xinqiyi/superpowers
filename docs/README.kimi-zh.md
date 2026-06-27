# 在 Kimi Code 中使用 Superpowers

在 [Kimi Code](https://github.com/MoonshotAI/kimi-code) 中使用 Superpowers 的完整指南。

## 安装

Superpowers 可在 Kimi Code 的 plugin 商城中获取。

打开 plugin 管理器：

```text
/plugins
```

进入 `Marketplace` > `Superpowers` 并安装。

你也可以从此仓库安装：

```text
/plugins install https://github.com/obra/superpowers
```

若想针对 `dev` 分支验证未发布版本，请显式固定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code 会将 plugin 变更应用于新会话。在安装、更新、启用、禁用或重新加载 plugin 后，使用 `/new` 启动新会话。

## 工作原理

Kimi plugin 清单位于 `.kimi-plugin/plugin.json`。

该清单完成三件事：

1. 将 Kimi Code 指向现有的 `skills/` 目录。
2. 通过 `sessionStart.skill` 在会话启动时加载 `using-superpowers`。
3. 通过 `skillInstructions` 提供 Kimi 特定的工具映射。

Kimi Code 从此仓库读取 Superpowers 的 skills。没有复制的 skill 文件、符号链接、hooks 或额外的运行时依赖。

## 工具映射

Skills 描述的是行为，而非硬编码某个运行时的工具名称。在 Kimi Code 上，这些行为对应以下工具：

- "询问用户" / "提出澄清性问题" -> `AskUserQuestion`
- "创建 todo" / "在 todo 列表中标记完成" -> `TodoList`
- "派发子 agent" -> `Agent`
- "调用 skill" -> Kimi Code 原生的 `Skill` 工具
- "读取文件" / "写入文件" / "编辑文件" -> `Read`, `Write`, `Edit`
- "运行 shell 命令" -> `Bash`
- "搜索文件内容" -> `Grep`
- "按路径或模式查找文件" -> `Glob`
- "获取 URL" -> `FetchURL`
- "搜索网络" -> `WebSearch`

## 更新

使用 Kimi Code 的 plugin 管理器：

```text
/plugins
```

选择 Superpowers 并在其中进行更新。更新后使用 `/new` 启动新会话。

## 故障排除

### Plugin 未加载

1. 运行 `/plugins info superpowers` 并检查诊断信息。
2. 确保 plugin 已启用。
3. 安装或更新后使用 `/new` 启动新会话。

### 直接从 GitHub 安装时使用了旧版本

当裸仓库 URL 存在时，Kimi Code 会安装最新的 GitHub 发布版本。若要在下一个 Superpowers 版本发布前测试未发布的变更，请显式安装指定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### Skills 未触发

1. 确认 `/plugins info superpowers` 显示 plugin 已启用。
2. 使用 `/new` 启动新会话。
3. 尝试验收测试提示：`Let's make a react todo list`。正常工作时应能在编写代码前加载 `brainstorming`。
