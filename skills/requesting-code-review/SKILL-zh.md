---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# 请求代码审查

分派一个代码审阅者 subagent，在问题级联之前捕获它们。审阅者获得精确构建的评估上下文——永远不会获得你的会话历史。这使审阅者专注于工作产品，而不是你的思考过程，并为继续工作保留你自己的上下文。

**核心原则：** 及早审查，经常审查。

## 何时请求审查

**必须：**
- 在 subagent-driven development 中的每个任务之后
- 在完成主要功能后
- 在合并到 main 之前

**可选但有价值：**
- 卡住时（新的视角）
- 在重构之前（基线检查）
- 在修复复杂 bug 之后

## 如何请求

**1. 获取 git SHAs：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 分派代码审阅者 subagent：**

分派一个 `general-purpose` subagent，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你构建内容的简要摘要
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始 commit
- `{HEAD_SHA}` - 结束 commit

**3. 根据反馈采取行动：**
- 立即修复关键问题
- 在继续之前修复重要问题
- 记录小问题待后续处理
- 如果审阅者错了则反驳（附上理由）

## 示例

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch code reviewer subagent]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## 与工作流的集成

**Subagent-Driven Development：**
- 在每项任务之后进行审查
- 在问题叠加之前捕获它们
- 在进入下一个任务之前修复

**执行计划：**
- 在每个任务之后或自然的检查点进行审查
- 获取反馈，应用，继续

**临时开发：**
- 在合并之前审查
- 卡住时审查

## 红旗

**永远不要：**
- 因为"太简单"而跳过审查
- 忽略关键问题
- 在未修复重要问题的情况下继续
- 与有效的技术反馈争论

**如果审阅者错了：**
- 用技术推理反驳
- 展示证明它有效的代码/测试
- 请求澄清

参见模板：[code-reviewer.md](code-reviewer.md)
