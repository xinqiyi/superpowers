# 跨平台多语言 Hooks for Claude Code

Claude Code 插件需要能在 Windows、macOS 和 Linux 上正常工作的 hooks。本文档描述了 `hooks/run-hook.cmd` 中使用的单一通用分配器模式。

> **权威来源：** `hooks/run-hook.cmd` 是规范的实现。当本文档与代码不一致时，请相信代码。

## 问题

Claude Code 通过系统的默认 shell 运行 hook 命令：
- **Windows**: CMD.exe
- **macOS/Linux**: bash 或 sh

这带来了几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中无效
4. **`.sh` 自动前置**：Claude Code 在 Windows 上会自动在任何包含 `.sh` 的命令前添加 `bash`——如果脚本有扩展名，这会干扰分配器

## 解决方案：无扩展名脚本 + 单一通用分配器

本仓库为所有 hooks 使用一个通用的 `run-hook.cmd` 分配器。Hook 脚本**没有扩展名**（`session-start`，而非 `session-start.sh`）。这是有意为之：它防止 Claude Code 的 Windows 自动检测在分配器命令前添加 `bash` 从而破坏它。

### 文件结构

```
hooks/
├── hooks.json          # 指向 run-hook.cmd，包含无扩展名的脚本名称
├── run-hook.cmd        # 跨平台分配器（多语言包装器）
└── session-start       # 实际的 hook 逻辑——无扩展名的 bash 脚本
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

路径使用引号，因为 `${CLAUDE_PLUGIN_ROOT}` 可能包含空格。

## `run-hook.cmd` 的高层工作原理

`run-hook.cmd` 是一个多语言脚本：Windows 将第一个块视为 batch 命令，而 Unix shell 将该块视为空操作 heredoc 并在其后继续执行。

当修改分配器时，不要从本文档复制实现。请直接阅读 `hooks/run-hook.cmd`，并在修改后运行 `tests/hooks/test-session-start.sh`。

### 在 Windows（CMD.exe）上如何工作

1. Batch 部分验证脚本名称，并从分配器自身位置解析 hook 目录。
2. 它在三个位置尝试查找 bash：
   - `C:\Program Files\Git\bin\bash.exe`
   - `C:\Program Files (x86)\Git\bin\bash.exe`
   - `PATH` 上的 `bash`（MSYS2、Cygwin 或非默认 Git 安装）
3. 如果找到 bash，它从 hooks 目录运行指定的无扩展名 hook 脚本。
4. 如果未找到 bash，分配器静默退出（返回 `0`）——插件继续正常工作，只是跳过了 hook。
5. `exit /b` 在到达 Unix 部分之前停止 CMD。

### 在 Unix（bash/sh）上如何工作

1. `: << 'CMDBLOCK'` 在空操作命令上打开一个 heredoc。
2. 整个 CMD batch 块被 heredoc 消耗并忽略。
3. 在 `CMDBLOCK` 之后，bash 解析脚本目录并直接 `exec` 指定的无扩展名脚本。

### 关键设计决策

| Decision | Why |
|----------|-----|
| 无扩展名脚本 | 防止 Claude Code 的 Windows `.sh` 自动前置干扰分配器命令 |
| 不使用 `-l`（登录 shell） | 不需要；hook 脚本应自包含，不依赖登录 shell 的 PATH 设置 |
| 不使用 `cygpath` | Bash 直接接收 Windows 路径并正确处理；`cygpath` 是旧的 `-c "..."` 调用模式所需的，而非直接 exec |
| 无 bash 时静默退出 | 避免破坏未安装 Git for Windows 用户的插件；hook 上下文注入被优雅跳过 |

## 编写跨平台 Hook 脚本

你的 hook 逻辑放在无扩展名的脚本文件中。一些可移植的模式：

### 请做
- 尽可能使用纯 bash 内置命令
- 使用 `$(command)` 代替反引号
- 引用所有变量展开：`"$VAR"`

### 避免
- 依赖 PATH 上的工具而没有备用方案（hook 在无 `-l` 模式下运行，因此登录 shell 的 PATH 未设置）
- 给脚本加上 `.sh` 扩展名——这会触发 Claude Code 的 Windows 自动前置

### 示例：无需外部工具的 JSON 转义

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 故障排除

### "bash is not recognized"

CMD 在分配器尝试的三个位置均未找到 bash。分配器静默退出（返回 0）而非报错，因此 hook 被跳过。请将 Git for Windows 安装到标准路径，或确保 `bash` 在 `PATH` 上。

### Hook 在 Unix 上可运行，但在 Windows 上无效果

检查 `hooks.json` 中的脚本文件名是否为**无扩展名**。像 `run-hook.cmd session-start.sh` 这样的命令可能触发 Claude Code 的 `.sh` 自动检测，绕过预期的 CMD 分配器路径，或只是尝试运行不存在的 `session-start.sh` 脚本。

### Hook 完全不触发

验证 `hooks.json` 中的 `matcher` 是否与你 harness 发出的事件类型匹配。Claude Code 使用 `startup|clear|compact`；Codex 使用 `startup|resume|clear`。查看 `hooks-codex.json` 了解 Codex 的变体。

## 相关问题

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) — `.sh` 脚本在 Windows 上以编辑器打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) — Hooks 在 Windows 上不工作
