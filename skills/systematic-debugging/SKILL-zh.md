---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes
---

# 系统性调试

## 概述

随机修复浪费时间并制造新 bug。快速补丁掩盖了潜在问题。

**核心原则：** 在尝试修复之前始终找到根本原因。症状修复就是失败。

**违反此流程的文字就是违反调试的精神。**

## 铁律

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

如果你还没有完成第 1 阶段，你不能提出修复方案。

## 何时使用

用于任何技术问题：
- 测试失败
- 生产环境中的 bug
- 意外行为
- 性能问题
- 构建失败
- 集成问题

**特别在以下情况下使用：**
- 时间紧迫时（紧急情况使猜测变得诱人）
- "只是一次快速修复"看起来很明显时
- 你已经尝试了多次修复
- 之前的修复没有效果
- 你还没有完全理解问题

**不要跳过的情况：**
- 问题看起来很简单（简单 bug 也有根本原因）
- 你很匆忙（冲 刺 保 证 返 工）
- 经理想立即修复（系统性的方法比乱试更快）

## 四个阶段

在进入下一步之前，你必须完成每个阶段。

### 第 1 阶段：根本原因调查

**在尝试任何修复之前：**

1. **仔细阅读错误消息**
   - 不要跳过错误或警告
   - 它们通常包含确切的解决方案
   - 完整阅读堆栈跟踪
   - 注意行号、文件路径、错误代码

2. **可靠地复现**
   - 你能稳定触发它吗？
   - 确切的步骤是什么？
   - 每次都会发生吗？
   - 如果不能复现 → 收集更多数据，不要猜测

3. **检查最近的更改**
   - 什么变化可能导致这个问题？
   - Git diff、最近的 commits
   - 新的依赖、配置变更
   - 环境差异

4. **在多组件系统中收集证据**

   **当系统有多个组件时（CI → 构建 → 签名、API → 服务 → 数据库）：**

   **在提出修复之前，添加诊断工具：**
   ```
   For EACH component boundary:
     - Log what data enters component
     - Log what data exits component
     - Verify environment/config propagation
     - Check state at each layer

   Run once to gather evidence showing WHERE it breaks
   THEN analyze evidence to identify failing component
   THEN investigate that specific component
   ```

   **示例（多层系统）：**
   ```bash
   # Layer 1: Workflow
   echo "=== Secrets available in workflow: ==="
   echo "IDENTITY: ${IDENTITY:+SET}${IDENTITY:-UNSET}"

   # Layer 2: Build script
   echo "=== Env vars in build script: ==="
   env | grep IDENTITY || echo "IDENTITY not in environment"

   # Layer 3: Signing script
   echo "=== Keychain state: ==="
   security list-keychains
   security find-identity -v

   # Layer 4: Actual signing
   codesign --sign "$IDENTITY" --verbose=4 "$APP"
   ```

   **这揭示：** 哪一层失败（secrets → workflow ✓, workflow → build ✗）

5. **追踪数据流**

   **当错误在调用栈深处时：**

   请参阅此目录中的 `root-cause-tracing.md`，了解完整的反向追踪技术。

   **快速版：**
   - 错误值从何而来？
   - 谁用错误值调用了这个？
   - 一直向上追踪直到找到源头
   - 在源头修复，而不是在症状处修复

### 第 2 阶段：模式分析

**在修复之前找到模式：**

1. **找到工作中的示例**
   - 在相同代码库中找到类似的工作代码
   - 有什么工作正常的代码与损坏的代码类似？

2. **对照参考进行比较**
   - 如果要实现模式，完整阅读参考实现
   - 不要浏览——阅读每一行
   - 在应用之前完全理解模式

3. **识别差异**
   - 工作代码和损坏代码之间有什么不同？
   - 列出每一个差异，无论多小
   - 不要假设"那个不重要"

4. **理解依赖关系**
   - 这需要哪些其他组件？
   - 需要什么设置、配置、环境？
   - 它做了哪些假设？

### 第 3 阶段：假设与测试

**科学方法：**

1. **形成单一假设**
   - 清晰陈述："我认为 X 是根本原因，因为 Y"
   - 写下来
   - 要具体，不要模糊

2. **最小化测试**
   - 做出可能的最小的更改来测试假设
   - 一次只改变一个变量
   - 不要同时修复多个问题

3. **在继续之前验证**
   - 有效吗？有效 → 第 4 阶段
   - 没效？形成新的假设
   - 不要在此基础上添加更多修复

4. **当你不确定时**
   - 说"我不理解 X"
   - 不要假装知道
   - 寻求帮助
   - 进一步研究

### 第 4 阶段：实施

**修复根本原因，而不是症状：**

1. **创建失败测试用例**
   - 尽可能简单的复现
   - 如果可能，编写自动化测试
   - 如果没有框架，编写一次性测试脚本
   - 必须在修复之前拥有
   - 使用 `superpowers:test-driven-development` skill 来编写正确的失败测试

2. **实施单一修复**
   - 针对确定的根本原因
   - 一次一个更改
   - 没有"既然我在这里"的改进
   - 没有捆绑的重构

3. **验证修复**
   - 测试现在通过了吗？
   - 其他测试没有被破坏？
   - 问题确实解决了？

4. **如果修复不奏效**
   - 停止
   - 计数：你已经尝试了多少次修复？
   - 如果 < 3：返回第 1 阶段，用新信息重新分析
   - **如果 ≥ 3：停止并质疑架构（下面的第 5 步）**
   - 在没有架构讨论的情况下，不要尝试第 4 次修复

5. **如果 3+ 次修复失败：质疑架构**

   **表明架构问题的模式：**
   - 每次修复都在不同的地方揭示新的共享状态/耦合/问题
   - 修复需要"大规模重构"才能实施
   - 每次修复都在其他地方创造新症状

   **停下来质疑基本问题：**
   - 这个模式是否从根本上合理？
   - 我们是否"纯粹因为惯性而坚持它"？
   - 我们应该重构架构，还是继续修复症状？

   **在尝试更多修复之前与你的人类搭档讨论**

   这不是失败的假设——这是错误的架构。

## 红旗——停下来遵循流程

如果你发现自己有这些想法：
- "先快速修复，以后再调查"
- "试试改变 X 看看是否有效"
- "做多个更改，运行测试"
- "跳过测试，我手动验证"
- "可能是 X，让我修复它"
- "我不完全理解但这可能有效"
- "模式说 X 但我会不同地调整"
- "以下是主要问题：[列出修复而不调查]"
- "在追踪数据流之前提出解决方案"
- **"再试一次修复"（当已经尝试了 2+ 次时）**
- **"每次修复都在不同的地方揭示新问题"**

**所有这些意味着：停止。返回第 1 阶段。**

**如果 3+ 次修复失败：** 质疑架构（参见第 4.5 阶段）

## 你的人类搭档表示你做错了的信号

**注意这些纠正：**
- "是不是没有发生？" - 你未经验证就假设了
- "它会不会显示...？" - 你应该添加证据收集
- "别再猜了" - 你在没有理解的情况下提出修复
- "深度思考这个" - 质疑基本问题，而不仅仅是症状
- "我们卡住了？"（沮丧的） - 你的方法不奏效

**当你看到这些时：** 停止。返回第 1 阶段。

## 常见合理化

| Excuse | Reality |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too. Process is fast for simple bugs. |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check thrashing. |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from the start. |
| "I'll write test after confirming fix works" | Untested fixes don't stick. Test first proves it. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question pattern, don't fix again. |

## 快速参考

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes, gather evidence | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare | Identify differences |
| **3. Hypothesis** | Form theory, test minimally | Confirmed or new hypothesis |
| **4. Implementation** | Create test, fix, verify | Bug resolved, tests pass |

## 当流程揭示"没有根本原因"

如果系统性调查揭示问题确实是环境性的、依赖于时序的或外部的：

1. 你已经完成了流程
2. 记录你调查的内容
3. 实施适当的处理（重试、超时、错误消息）
4. 为将来的调查添加监控/日志记录

**但是：** 95% 的"没有根本原因"案例都是不完整的调查。

## 支持技术

这些技术是系统性调试的一部分，可在此目录中找到：

- **`root-cause-tracing.md`** - 通过调用栈反向追踪 bug 以找到原始触发点
- **`defense-in-depth.md`** - 在找到根本原因后在多个层添加验证
- **`condition-based-waiting.md`** - 用条件轮询替换任意超时

**相关 skills：**
- **superpowers:test-driven-development** - 用于创建失败测试用例（第 4 阶段，第 1 步）
- **superpowers:verification-before-completion** - 在声称成功之前验证修复有效

## 实际影响

来自调试会话：
- 系统性方法：15-30 分钟修复
- 随机修复方法：2-3 小时的乱试
- 首次修复成功率：95% vs 40%
- 引入新 bug：接近零 vs 常见
