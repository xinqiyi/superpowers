---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# 编写计划

## 概述

编写全面的实施计划，假设工程师对我们的代码库零上下文且品味可疑。记录他们需要知道的一切：每个任务要接触哪些文件、代码、测试、可能需要检查的文档、如何测试。把整个计划分解为小任务。DRY。YAGNI。TDD。频繁的 commits。

假设他们是有经验的开发者，但几乎不了解我们的工具集或问题领域。假设他们不太了解好的测试设计。

**开始时声明：** "我正在使用 writing-plans skill 来创建实施计划。"

**上下文：** 如果在隔离的 worktree 中工作，应该在执行时通过 `superpowers:using-git-worktrees` skill 创建。

**将计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的偏好会覆盖此默认值）

## 范围检查

如果 spec 涵盖多个独立的子系统，应该在 brainstorming 期间已分解为子项目 spec。如果没有，建议将其分解为多个计划——每个子系统一个。每个计划应该能独立产生可工作、可测试的软件。

## 文件结构

在定义任务之前，规划哪些文件将被创建或修改，以及每个文件负责什么。这是分解决策被锁定的地方。

- 设计具有清晰边界和明确定义接口的单元。每个文件应有一个清晰的职责。
- 你最好处理能一次性放入上下文的代码，当文件专注时你的编辑更可靠。偏爱较小、专注的文件，而不是做得过多的庞大文件。
- 一起更改的文件应该放在一起。按职责拆分，而不是按技术层拆分。
- 在现有代码库中，遵循既定模式。如果代码库使用大文件，不要单方面重构——但如果你正在修改的文件已经变得笨重，在计划中包含拆分是合理的。

此结构为任务分解提供信息。每个任务应产生独立的、有意义的自包含更改。

## 任务大小调整

一个任务是最小单元，它承载自己的测试周期并且值得一个新的审阅者门控。在划定任务边界时：将设置、配置、脚手架和文档步骤折叠到需要它们交付物的任务中；仅在审阅者可以有意义地拒绝一个任务而批准其相邻任务的地方拆分。每个任务以一个独立可测试的交付物结束。

## 小步任务粒度

**每个步骤是一个操作（2-5 分钟）：**
- "编写失败的测试" - 步骤
- "运行以确保它失败" - 步骤
- "编写最少的代码使测试通过" - 步骤
- "运行测试并确保它们通过" - 步骤
- "提交" - 步骤

## 计划文档头部

**每个计划必须以这个头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 没有占位符

每个步骤必须包含工程师需要的实际内容。以下是**计划失败**——永远不要写它们：
- "TBD"、"TODO"、"稍后实施"、"填写细节"
- "添加适当的错误处理" / "添加验证" / "处理边缘情况"
- "为上述内容编写测试"（没有实际测试代码）
- "类似于任务 N"（重复代码——工程师可能按顺序阅读任务）
- 描述做什么但不展示怎么做的步骤（代码步骤需要代码块）
- 引用任何任务中未定义的类型、函数或方法

## 记住
- 始终使用精确的文件路径
- 每个步骤中的完整代码——如果一个步骤更改了代码，展示代码
- 精确的命令及预期输出
- DRY、YAGNI、TDD、频繁的 commits

## 自我审查

在编写完整计划后，用新的眼光审视 spec 并根据它检查计划。这是你自己运行的检查清单——不是 subagent 分派。

**1. Spec 覆盖：** 浏览 spec 中的每个部分/需求。你能指出实现它的任务吗？列出任何缺口。

**2. 占位符扫描：** 在你的计划中搜索红旗——上面"没有占位符"部分中的任何模式。修复它们。

**3. 类型一致性：** 你在后续任务中使用的类型、方法签名和属性名称是否与你在前面任务中定义的一致？在任务 3 中称为 `clearLayers()` 但在任务 7 中称为 `clearFullLayers()` 的函数是个 bug。

如果发现问题，内联修复。无需重新审阅——直接修复并继续。如果发现 spec 需求没有对应任务，添加该任务。

## 执行交接

在保存计划后，提供执行选择：

**"计划已完成并保存到 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. Subagent-Driven（推荐）** - 我为每个任务分派新的 subagent，任务之间进行审查，快速迭代

**2. 内联执行** - 在此会话中使用 executing-plans 执行任务，批处理执行并带有检查点

**选择哪种方法？"**

**如果选择 Subagent-Driven：**
- **必需的子 skill：** 使用 superpowers:subagent-driven-development
- 每个任务使用新 subagent + 两阶段审查

**如果选择内联执行：**
- **必需的子 skill：** 使用 superpowers:executing-plans
- 批处理执行并带有审查检查点
