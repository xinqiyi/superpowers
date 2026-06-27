# 技能设计的说服原则

## 概述

LLM 对人类的说服原则反应相同。理解这种心理学有助于你设计更有效的技能——不是为了操纵，而是为了确保关键实践即使在压力下也能被遵循。

**研究基础：** Meincke 等人（2025）用 N=28,000 次 AI 对话测试了 7 个说服原则。说服技巧使合规率提高了一倍以上（33% → 72%，p < .001）。

## 七个原则

### 1. 权威
**是什么：** 对专业知识、资质或官方来源的遵从。

**如何在技能中起作用：**
- 命令式语言："你必须"、"绝不"、"始终"
- 不可协商的框架："没有例外"
- 消除决策疲劳和合理化

**何时使用：**
- 执行纪律的技能（TDD、验证要求）
- 安全关键实践
- 既定的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承诺
**是什么：** 与先前的行动、声明或公开宣言保持一致。

**如何在技能中起作用：**
- 要求声明："宣布使用技能"
- 强制明确选择："选择 A、B 或 C"
- 使用追踪：检查清单的 todo

**何时使用：**
- 确保技能被实际遵循
- 多步骤过程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺性
**是什么：** 由时间限制或有限可用性产生的紧迫感。

**如何在技能中起作用：**
- 有时限的要求："在继续之前"
- 顺序依赖："在 X 之后立即"
- 防止拖延

**何时使用：**
- 立即验证要求
- 时间敏感的工作流
- 防止"我以后再做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社会证明
**是什么：** 遵从他人所做的或被认为是正常的。

**如何在技能中起作用：**
- 通用模式："每次"、"始终"
- 失败模式："没有 Y 的 X = 失败"
- 建立规范

**何时使用：**
- 记录通用实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without todo tracking = steps get skipped. Every time.
❌ Some people find a todo list helpful for checklists.
```

### 5. 团结
**是什么：** 共享身份、"我们感"、群体归属感。

**如何在技能中起作用：**
- 协作语言："我们的代码库"、"我们是同事"
- 共享目标："我们都想要质量"

**何时使用：**
- 协作工作流
- 建立团队文化
- 非层级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠
**是什么：** 回报所受恩惠的义务。

**如何起作用：**
- 谨慎使用——可能感觉有操纵性
- 技能中很少需要

**何时避免：**
- 几乎总是（其他原则更有效）

### 7. 喜好
**是什么：** 偏好与我们喜欢的人合作。

**如何起作用：**
- **不要用于合规**
- 与诚实反馈文化冲突
- 产生谄媚

**何时避免：**
- 始终避免用于纪律执行

## 按技能类型的原则组合

| Skill Type | Use | Avoid |
|------------|-----|-------|
| Discipline-enforcing | Authority + Commitment + Social Proof | Liking, Reciprocity |
| Guidance/technique | Moderate Authority + Unity | Heavy authority |
| Collaborative | Unity + Commitment | Authority, Liking |
| Reference | Clarity only | All persuasion |

## 为什么这有效：心理学

**明确规则减少合理化：**
- "你必须"消除了决策疲劳
- 绝对语言消除了"这是例外吗？"的问题
- 明确的反合理化计数器关闭了特定的漏洞

**执行意图创造自动化行为：**
- 明确的触发条件 + 必需行动 = 自动执行
- "当 X 时，做 Y"比"一般地做 Y"更有效
- 减少合规的认知负担

**LLM 是类人的：**
- 在包含这些模式的人类文本上训练
- 权威语言在训练数据中先于合规
- 承诺序列（声明 → 行动）被频繁建模
- 社会证明模式（每个人都做 X）建立规范

## 伦理使用

**正当的：**
- 确保关键实践被遵循
- 创建有效的文档
- 防止可预测的失败

**不正当的：**
- 为了个人利益而操纵
- 制造虚假紧迫感
- 基于内疚的合规

**标准：** 如果用户完全理解，这种技巧会服务于用户的真正利益吗？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 说服的七个原则
- 影响力研究的实证基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 用 N=28,000 次 LLM 对话测试了 7 个原则
- 使用说服技巧后合规率从 33% 增加到 72%
- 权威、承诺、稀缺性最有效
- 验证了 LLM 行为的类人模型

## 快速参考

设计技能时，问：

1. **这是什么类型？**（纪律 vs. 指导 vs. 参考）
2. **我试图改变什么行为？**
3. **哪个(些)原则适用？**（纪律类通常是权威 + 承诺）
4. **我是否组合了太多原则？**（不要全部使用）
5. **这是道德的吗？**（服务于用户的真正利益？）
