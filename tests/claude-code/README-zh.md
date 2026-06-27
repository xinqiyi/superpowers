# Claude Code Skills 测试

使用 Claude Code CLI 对 superpowers skills 进行自动化测试。

## 概述

本测试套件验证 skills 是否正确加载，以及 Claude 是否按预期遵循它们。测试以无头模式（`claude -p`）调用 Claude Code 并验证行为。

## 要求

- 已安装 Claude Code CLI 且在 PATH 中（`claude --version` 应可工作）
- 已安装本地 superpowers plugin（安装方法见主 README）

## 运行测试

### 运行所有快速测试（推荐）：
```bash
./run-skill-tests.sh
```

### 运行集成测试（慢速，10-30 分钟）：
```bash
./run-skill-tests.sh --integration
```

### 运行特定测试：
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 使用详细输出运行：
```bash
./run-skill-tests.sh --verbose
```

### 设置自定义超时时间：
```bash
./run-skill-tests.sh --timeout 1800  # 集成测试 30 分钟
```

## 测试结构

### test-helpers.sh
Skills 测试的公共函数：
- `run_claude "prompt" [timeout]` - 使用提示运行 Claude
- `assert_contains output pattern name` - 验证模式存在
- `assert_not_contains output pattern name` - 验证模式不存在
- `assert_count output pattern count name` - 验证精确计数
- `assert_order output pattern_a pattern_b name` - 验证顺序
- `create_test_project` - 创建临时测试目录
- `create_test_plan project_dir` - 创建示例计划文件

### 测试文件

每个测试文件：
1. 引用 `test-helpers.sh`
2. 使用特定提示运行 Claude Code
3. 使用断言验证预期行为
4. 成功返回 0，失败返回非零值

## 测试示例

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# 询问 Claude 关于该 skill 的信息
output=$(run_claude "What does the my-skill skill do?" 30)

# 验证回复
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 当前测试

### 快速测试（默认运行）

#### test-subagent-driven-development.sh
测试 skill 内容和要求（约 2 分钟）：
- Skill 加载和可访问性
- 工作流顺序（spec 合规性先于代码质量）
- 自审要求已记录
- 计划读取效率已记录
- Spec 合规性审查者的怀疑态度已记录
- 审查循环已记录
- 任务上下文提供已记录

### 集成测试（使用 --integration 标志）

#### test-subagent-driven-development-integration.sh
完整工作流执行测试（约 10-30 分钟）：
- 创建真实测试项目，包含 Node.js 设置
- 创建包含 2 个任务的实施计划
- 使用 subagent-driven-development 执行计划
- 验证实际行为：
  - 计划仅在开始时读取一次（而非每个任务读取一次）
  - 完整任务文本在子 agent 提示中提供
  - 子 agent 在报告前执行自审
  - Spec 合规性审查在代码质量审查之前进行
  - Spec 审查者独立阅读代码
  - 生成可工作的实现
  - 测试通过
  - 创建正确的 git 提交

**测试内容：**
- 工作流端到端确实有效
- 我们的改进确实被应用
- 子 agent 正确遵循 skill
- 最终代码功能完整且经过测试

#### test-worktree-native-preference.sh
针对 using-git-worktrees skill 的 RED-GREEN-REFACTOR 验证（约 5 分钟）：
- RED：没有步骤 1a 的 skill——agent 应使用 `git worktree add`
- GREEN：包含步骤 1a 的 skill——agent 应使用原生的 EnterWorktree 工具
- PRESSURE：在紧急框架下且存在预先存在的 `.worktrees/` 目录时，与 GREEN 相同
- Drill 场景 `worktree-creation-under-pressure.yaml` 仅覆盖 PRESSURE 阶段

## 添加新测试

1. 创建新的测试文件：`test-<skill-name>.sh`
2. 引用 test-helpers.sh
3. 使用 `run_claude` 和断言编写测试
4. 添加到 `run-skill-tests.sh` 的测试列表中
5. 赋予执行权限：`chmod +x test-<skill-name>.sh`

## 超时注意事项

- 默认超时：每个测试 5 分钟
- Claude Code 可能需要时间响应
- 如有需要，使用 `--timeout` 调整
- 测试应聚焦，避免长时间运行

## 调试失败的测试

使用 `--verbose`，你将看到完整的 Claude 输出：
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

不带 verbose 时，仅失败时显示输出。

## CI/CD 集成

要在 CI 中运行：
```bash
# 为 CI 环境使用显式超时
./run-skill-tests.sh --timeout 900

# 退出码 0 = 成功，非零 = 失败
```

## 备注

- 测试验证的是 skill *指令*，而非完整执行
- 完整工作流测试会非常慢
- 重点关注验证关键的 skill 要求
- 测试应具有确定性
- 避免测试实现细节
