---
name: verification-before-completion
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always
---

# 完成前验证

## 概述

在未经验证的情况下声称工作已完成是欺骗，而不是高效。

**核心原则：** 始终先有证据，再有声明。

**违反此规则的文字就是违反此规则的精神。**

## 铁律

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

如果你没有在此消息中运行验证命令，你就不能声称它通过了。

## 门控函数

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## 常见失败

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check, extrapolation |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red-green cycle verified | Test passes once |
| Agent completed | VCS diff shows changes | Agent reports "success" |
| Requirements met | Line-by-line checklist | Tests passing |

## 红旗——停止

- 使用"应该"、"可能"、"似乎"
- 在验证之前表达满意（"太棒了！"、"完美！"、"完成了！"等）
- 即将 commit/push/PR 而未经验证
- 信任 agent 的成功报告
- 依赖部分验证
- 认为"就这一次"
- 累了想让工作结束
- **任何暗示成功但未运行验证的措辞**

## 合理化预防

| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | Verify independently |
| "I'm tired" | Exhaustion ≠ excuse |
| "Partial check is enough" | Partial proves nothing |
| "Different words so rule doesn't apply" | Spirit over letter |

## 关键模式

**测试：**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**回归测试（TDD 红-绿）：**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**构建：**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**需求：**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent 委派：**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## 为什么这很重要

来自 24 个失败记忆：
- 你的人类搭档说"我不相信你"——信任破裂
- 未定义的函数发布——会崩溃
- 缺少的需求发布——功能不完整
- 时间浪费在虚假完成上 → 重定向 → 返工
- 违反："诚实是核心价值观。如果你撒谎，你将被替换。"

## 何时应用

**始终在以下情况之前：**
- 任何形式的成功/完成声明
- 任何形式的满意表达
- 任何关于工作状态的正面陈述
- 提交、创建 PR、任务完成
- 移动到下一个任务
- 委派给 agent

**规则适用于：**
- 精确短语
- 释义和同义词
- 暗示成功
- 任何暗示完成/正确的沟通

## 底线

**验证没有捷径。**

运行命令。读取输出。然后声明结果。

这是不可谈判的。
