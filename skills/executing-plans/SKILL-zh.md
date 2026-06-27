---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# 执行计划

## 概述

加载计划，批判性审阅，执行所有任务，完成后报告。

**开始时声明：** "我正在使用 executing-plans skill 来实施这个计划。"

**注意：** 告诉你的人类搭档，Superpowers 在能够访问 subagent 时效果会好得多。如果在支持 subagent 的平台上运行（Claude Code、Codex CLI、Codex App、Copilot CLI 和 Gemini CLI 均符合条件；请参阅 `../using-superpowers/references/` 中各平台的工具参考），其工作质量会显著提高。如果 subagent 可用，请使用 superpowers:subagent-driven-development 代替本 skill。

## 流程

### 第 1 步：加载并审阅计划
1. 读取计划文件
2. 批判性审阅——识别关于计划的任何问题或疑虑
3. 如果有疑虑：在开始之前向你的人类搭档提出
4. 如果没有疑虑：为计划项创建任务列表并继续

### 第 2 步：执行任务

对于每个任务：
1. 标记为进行中
2. 精确遵循每个步骤（计划包含小步操作）
3. 按指定运行验证
4. 标记为已完成

### 第 3 步：完成开发

在所有任务完成并验证后：
- 声明："我正在使用 finishing-a-development-branch skill 来完成这项工作。"
- **必需的子 skill：** 使用 superpowers:finishing-a-development-branch
- 遵循该 skill 来验证测试、展示选项、执行选择

## 何时停下来寻求帮助

**在以下情况下立即停止执行：**
- 遇到障碍（缺少依赖、测试失败、说明不清）
- 计划存在阻止开始的严重缺口
- 不理解某条指令
- 验证反复失败

**询问澄清而不是猜测。**

## 何时重新审视前面的步骤

**在以下情况下返回审阅（第 1 步）：**
- 搭档根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行通过障碍**——停下来询问。

## 记住
- 首先批判性审阅计划
- 精确遵循计划步骤
- 不要跳过验证
- 在计划要求时引用相关 skill
- 遇到阻碍时停下来，不要猜测
- 在未经用户明确同意的情况下，永远不要在 main/master 分支上开始实施

## 集成

**必需的工作流 skills：**
- **superpowers:using-git-worktrees** - 确保隔离的工作空间（创建一个或验证已有）
- **superpowers:writing-plans** - 创建本 skill 执行的计划
- **superpowers:finishing-a-development-branch** - 在所有任务完成后完成开发
