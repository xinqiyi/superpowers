---
name: receiving-code-review
description: Use when receiving code review feedback, before implementing suggestions, especially if feedback seems unclear or technically questionable - requires technical rigor and verification, not performative agreement or blind implementation
---

# 接收代码审查

## 概述

代码审查需要技术评估，而不是情感表演。

**核心原则：** 先验证再实施。先询问再假设。技术正确性优于社交舒适感。

## 响应模式

```
WHEN receiving code review feedback:

1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

## 禁止的回应

**永远不要：**
- "你说得完全正确！"（明确违反指令文件）
- "好观点！" / "极好的反馈！"（表演性）
- "让我现在实施"（在验证之前）

**而是：**
- 重述技术需求
- 提出澄清性问题
- 如果对方错了，用技术推理进行反驳
- 直接开始工作（行动胜于言语）

## 处理不清晰的反馈

```
IF any item is unclear:
  STOP - do not implement anything yet
  ASK for clarification on unclear items

WHY: Items may be related. Partial understanding = wrong implementation.
```

**示例：**
```
your human partner: "Fix 1-6"
You understand 1,2,3,6. Unclear on 4,5.

❌ WRONG: Implement 1,2,3,6 now, ask about 4,5 later
✅ RIGHT: "I understand items 1,2,3,6. Need clarification on 4 and 5 before proceeding."
```

## 按来源处理

### 来自你的人类搭档
- **可信赖的** - 在理解后实施
- **如果范围不清楚仍然要问**
- **不要表演性同意**
- **直接跳到行动**或技术性确认

### 来自外部审阅者
```
BEFORE implementing:
  1. Check: Technically correct for THIS codebase?
  2. Check: Breaks existing functionality?
  3. Check: Reason for current implementation?
  4. Check: Works on all platforms/versions?
  5. Check: Does reviewer understand full context?

IF suggestion seems wrong:
  Push back with technical reasoning

IF can't easily verify:
  Say so: "I can't verify this without [X]. Should I [investigate/ask/proceed]?"

IF conflicts with your human partner's prior decisions:
  Stop and discuss with your human partner first
```

**你的人类搭档的规则：** "外部反馈——保持怀疑，但仔细检查"

## 对"专业"功能的 YAGNI 检查

```
IF reviewer suggests "implementing properly":
  grep codebase for actual usage

  IF unused: "This endpoint isn't called. Remove it (YAGNI)?"
  IF used: Then implement properly
```

**你的人类搭档的规则：** "你和审阅者都向我汇报。如果我们不需要这个功能，就不要添加它。"

## 实施顺序

```
FOR multi-item feedback:
  1. Clarify anything unclear FIRST
  2. Then implement in this order:
     - Blocking issues (breaks, security)
     - Simple fixes (typos, imports)
     - Complex fixes (refactoring, logic)
  3. Test each fix individually
  4. Verify no regressions
```

## 何时反驳

在以下情况下反驳：
- 建议破坏了现有功能
- 审阅者缺乏完整上下文
- 违反了 YAGNI（未使用的功能）
- 对于这个技术栈来说技术不正确
- 存在遗留/兼容性原因
- 与你的搭档的架构决策冲突

**如何反驳：**
- 使用技术推理，而不是防御性态度
- 提出具体问题
- 引用有效的测试/代码
- 如果涉及架构问题，邀请你的搭档参与

**如果你对公开反驳感到不舒服：** 说出这种矛盾，然后告诉你的搭档你发现的问题。他们会欣赏你的诚实。

## 确认正确的反馈

当反馈确实是正确的时候：
```
✅ "Fixed. [Brief description of what changed]"
✅ "Good catch - [specific issue]. Fixed in [location]."
✅ [Just fix it and show in the code]

❌ "You're absolutely right!"
❌ "Great point!"
❌ "Thanks for catching that!"
❌ "Thanks for [anything]"
❌ ANY gratitude expression
```

**为什么不需要感谢：** 行动胜于言语。直接修复。代码本身表明你听到了反馈。

**如果你发现自己正要写"谢谢"：** 删除它。改为陈述修复内容。

## 优雅地纠正你的反驳

如果你反驳了但错了：
```
✅ "You were right - I checked [X] and it does [Y]. Implementing now."
✅ "Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."

❌ Long apology
❌ Defending why you pushed back
❌ Over-explaining
```

陈述事实性纠正，然后继续。

## 常见错误

| Mistake | Fix |
|---------|-----|
| Performative agreement | State requirement or just act |
| Blind implementation | Verify against codebase first |
| Batch without testing | One at a time, test each |
| Assuming reviewer is right | Check if breaks things |
| Avoiding pushback | Technical correctness > comfort |
| Partial implementation | Clarify all items first |
| Can't verify, proceed anyway | State limitation, ask for direction |

## 真实示例

**表演性同意（糟糕）：**
```
Reviewer: "Remove legacy code"
❌ "You're absolutely right! Let me remove that..."
```

**技术验证（良好）：**
```
Reviewer: "Remove legacy code"
✅ "Checking... build target is 10.15+, this API needs 13+. Need legacy for backward compat. Current impl has wrong bundle ID - fix it or drop pre-13 support?"
```

**YAGNI（良好）：**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
✅ "Grepped codebase - nothing calls this endpoint. Remove it (YAGNI)? Or is there usage I'm missing?"
```

**不清晰项（良好）：**
```
your human partner: "Fix items 1-6"
You understand 1,2,3,6. Unclear on 4,5.
✅ "Understand 1,2,3,6. Need clarification on 4 and 5 before implementing."
```

## GitHub 线程回复

在回复 GitHub 上的内联审阅评论时，在评论线程中回复（`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`），而不是作为 PR 的顶级评论。

## 底线

**外部反馈 = 需要评估的建议，不是需要服从的命令。**

验证。质疑。然后实施。

不要表演性同意。始终坚持技术严谨。
