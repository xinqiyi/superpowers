# 文档审查系统设计

## 概述

向 superpowers 工作流添加两个新的审查阶段：

1. **Spec 文档审查**——在 brainstorming 之后、writing-plans 之前
2. **计划文档审查**——在 writing-plans 之后、实施之前

两者都遵循实施审查使用的迭代循环模式。

## Spec 文档审查者

**目的：** 验证 spec 完整、一致并可用于实施计划。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 检查内容 |
|----------|------------------|
| 完整性 | TODOs、占位符、"TBD"、不完整章节 |
| 覆盖度 | 缺失的错误处理、边缘情况、集成点 |
| 一致性 | 内部矛盾、冲突的需求 |
| 清晰度 | 模糊的需求 |
| YAGNI | 未请求的功能、过度设计 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**审查循环：** 发现问题 -> brainstorming agent 修复 -> 重新审查 -> 重复直至批准。

**调度机制：** 使用带有 `subagent_type: general-purpose` 的 Task 工具。审查者提示模板提供完整提示。Brainstorming skill 的控制者调度审查者。

## 计划文档审查者

**目的：** 验证计划完整、与 spec 匹配，并具有正确的任务分解。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 检查内容 |
|----------|------------------|
| 完整性 | TODOs、占位符、不完整的任务 |
| Spec 对齐 | 计划覆盖 spec 需求，无范围蔓延 |
| 任务分解 | 任务原子化，边界清晰 |
| 任务语法 | 任务和步骤上的复选框语法 |
| 块大小 | 每个块不超过 1000 行 |

**块定义：** 块是计划文档中逻辑分组的任务，由 `## Chunk N: <name>` 标题分隔。Writing-plans skill 根据逻辑阶段（例如"基础"、"核心功能"、"集成"）创建这些边界。每个块应足够自包含以独立审查。

**Spec 对齐验证：** 审查者接收两者：
1. 计划文档（或当前块）
2. 用于参考的 spec 文档路径

审查者阅读两者并比较需求覆盖度。

**输出格式：** 与 spec 审查者相同，但范围限定到当前块。

**审查过程（逐块）：**
1. Writing-plans 创建块 N
2. 控制者调度 plan-document-reviewer，提供块 N 内容和 spec 路径
3. 审查者读取块和 spec，返回判定
4. 如有问题：writing-plans agent 修复块 N，转至步骤 2
5. 如果批准：继续到块 N+1
6. 重复直至所有块被批准

**调度机制：** 与 spec 审查者相同——使用 `subagent_type: general-purpose` 的 Task 工具。

## 更新的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**Spec 审查循环：**
1. Spec 完成
2. 调度审查者
3. 如有问题：修复 -> 转至 2
4. 如果批准：继续

**计划审查循环：**
1. 块 N 完成
2. 为块 N 调度审查者
3. 如有问题：修复 -> 转至 2
4. 如果批准：下一个块或实施

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**审查循环终止：**
- 无硬迭代限制——循环继续至审查者批准
- 如果循环超过 5 次迭代，控制者应向人类上报以获取指导
- 人类可选择：继续迭代、带着已知问题批准或中止

**分歧处理：**
- 审查者是建议性的——他们标记问题但不阻止
- 如果 agent 认为审查者反馈不正确，应在修复中解释原因
- 如果同一问题的分歧持续超过 3 次迭代，上报给人类

**格式错误的审查者输出：**
- 控制者应验证审查者输出具有必填字段（Status，如果适用 Issues）
- 如果格式错误，重新调度审查者并附上关于预期格式的说明
- 在 2 次格式错误的响应后，上报给人类

## 要更改的文件

**新文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改的文件：**
- `skills/brainstorming/SKILL.md`——在 spec 编写后添加审查循环
- `skills/writing-plans/SKILL.md`——添加逐块审查循环，更新任务语法示例
