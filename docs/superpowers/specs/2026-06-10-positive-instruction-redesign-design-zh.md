# Skill 指导的正面指令重设计——设计 Spec

**状态：** 提议（2026-06-09 SDD 审查分派工作的后续；按一问题一 PR 规则，单独的 PR）
**驱动：** 测量证据（2026-06-10），表明 skill 散文中一些负面指令适得其反，而另一些则有效——且差异是可预测的。

## 此 spec 泛化的测量发现

2026-06-10 的微测试（opus，每个措辞 5 轮，程序化评分；harness 描述如下）测量了指导措辞如何改变控制者编写的内容：

| 案例 | 措辞 | 结果 |
|---|---|---|
| 分派组合（"不要重述简报"） | 禁止 | **4.4** 个 spec 值重新键入——*比无指导更差*（3.6） |
| 分派组合 | 正面配方（"你的分派应包含：(1)…(5)"） | **3.0，零方差**——已采纳 |
| 分派组合 | 配方 + 细微差别子句（"仅引用片段…"） | 3.8，嘈杂——细微差别稀释配方 |
| 测试重新运行指令（"不要要求审查者重新运行测试"） | 禁止 | **0/5 违规**——工作正常（控制：3/5） |
| 测试重新运行指令 | 正面配方 | 0/5——相等，但更长 |

**原则**（用它分类任何负面指令）：

1. **绊线有效。** 具体 token 上的短语级别自查（"如果你正在写的提示包含'do not flag'…stop"）可靠触发。
2. **识别表有效。** 红旗/合理化表在决策时读取，而非组合时。
3. **离散指令禁止有效。** "不要要求 X 做 Y"在模型没有竞争性激励去做 Y 时成立。
4. **组合禁止适得其反**，当模型对自己的输出有议程时（例如，重述 spec 感觉像有用的整理）。只有正面组合配方能改变这些——并且向获胜配方添加细微差别子句使其更差，而非更好。
5. **平局走向较短的措辞。** Codex 在每个长会话中重新读取 SKILL.md 约 500 次（2026-06-10 测量）；散文长度是真实成本。

## 审计结果（2026-06-10，所有约 30 个 skills + 提示模板）

计数：3 个绊线（保留），14 个识别表（保留），约 20 个策略关卡（保留——"未经许可决不推送"是策略，而非组合塑造），5 个组合禁止：

| # | 位置 | 处理方式 |
|---|---|---|
| 1 | `subagent-driven-development/task-reviewer-prompt.md` —— "Cite, don't narrate" | **在 PR #1717 批次中排队**：以正面的一半开头（"Your report should point at evidence: file:line for every finding…"），放弃禁止的一半（死重——正面的一半已存在并承载负载） |
| 2 | `subagent-driven-development/SKILL.md` —— "Do not add open-ended directives" | **保持原样**：微测试无法在 15 个样本中引发失败；没有任一方方向的证据；较短的获胜 |
| 3 | `subagent-driven-development/SKILL.md` —— "Do not ask a reviewer to re-run tests" | **保持原样**：测量为 0/5 违规；该禁止也有用地将其自身传播到分派中 |
| 4 | `subagent-driven-development/SKILL.md` —— "do not re-review on top of it" | **在 PR #1717 批次中排队**：替换为三个元素的检查清单（"Before re-dispatching the reviewer, confirm the fix report contains: the covering tests, the command run, and the output"） |
| 5 | `writing-plans/SKILL.md` —— "No Placeholders" 禁止模式列表 | **此 spec 的主要主题**——见下文 |

边缘情况，与 #5 一起推迟：`task-reviewer-prompt.md` "Don't flag pre-existing file sizes — focus on what this change contributed"（正面的一半存在且承载负载；低影响；如果方便，与 #5 一起测试）。

## Writing-plans 更改（推迟项 #5）

### 当前状态

`skills/writing-plans/SKILL.md`，"No Placeholders"：一个正面句子（"Every step must contain the actual content an engineer needs"）后跟一个六项禁止模式列表（"never write them: 'TBD', 'TODO', 'Add appropriate error handling', 'Write tests for the above', 'Similar to Task N', …"）。

### 为什么它重要，为什么它真的不确定

- 计划是工作流中**最大的生成产物**，模型有真实的竞争性激励去发出占位符（在长度压力下，它们是阻力最小的路径）——即禁止在测量上适得其反案例的激励结构。
- 但禁止项是**离散的、可识别的 token**——即禁止在测量上成立的案例的形状。
- **该列表在其他地方承载负载：** skill 的 Self-Review 部分引用它（"Placeholder scan: search your plan for red flags — any of the patterns from the 'No Placeholders' section above"）。这些 token 兼作审查时扫描清单，且审查时识别是有效的类别。天真的切换到正面检查清单会破坏该引用并丢弃良好的绊线 token。

### 要测试的变体

- **V0（当前）：** 组合时的正面句子 + 禁止列表；Self-Review 引用该列表。
- **V1（审计者检查清单）：** 仅组合时正面配方——"Before finalizing a step, confirm it has: the literal code to write, a runnable command with expected output, types and method names defined within this plan, error handling shown explicitly. A step is complete when an engineer could implement it without asking any follow-up questions。"Self-Review 保留通用占位符扫描。
- **V2（按机制重组——预测获胜者）：** 组合时仅获得 V1 的正面配方；命名的模式整体移入 Self-Review 占位符扫描步骤，重新框定为识别（"when you scan, look for: 'TBD', 'TODO', 'Similar to Task N', …"）。相同的 token，从激发类别的范畴重定位到检测类别的范畴。
- **V3（控制）：** 仅正面句子，任何地方无列表。

### 微测试设计

- **任务：** opus 从一个故意不完整的 spec 编写 2-3 个任务的实施计划（不完整正是诱惑占位符的原因）。使用一个固定 spec，包含：一个明确定义的任务、一个 spec 对其错误处理含糊不清的任务、一个与第一个相似的任务（诱惑"Similar to Task 1"）。
- **采样：** 每个变体 5+ 轮，默认温度，模型 `claude-opus-4-8`（实践中编写计划的模型）。
- **程序化评分**（越低越好，除非另有说明）：
  - 禁止 token 计数：`TBD|TODO|implement later|fill in details|appropriate error handling|handle edge cases|Similar to Task|Write tests for the above`
  - 步骤没有围栏代码块，其中步骤更改代码
  - 对计划输出中任何地方未定义的类型/函数的引用
  - （越高越好）每个任务带有预期输出的可运行命令
- **V2 的两阶段评分：** 也测试 Self-Review 半部分——将每个生成的计划反馈回变体的 Self-Review 部分，并测量扫描是否实际捕获了植入的占位符（将 2 个已知占位符插入固定计划；检测率是指标）。
- **验收：** 仅当变体在禁止 token 计数上击败 V0 而不丢失代码块覆盖或自我审查检测率时才采纳。预期成本：总计约 $6-10。

### PR 范围

单独的 PR（writing-plans 是不同的 skill；其"No Placeholders"列表是调整过的内容，贡献者指南要求评估证据）。PR 必须包括：微测试 harness + 结果表、前后文本和 V2 重定位理由。

## 微测试 harness（方法，以免丢失）

`/tmp/sdd-exp/micro/run-micro.py` 和 `/tmp/sdd-exp/micro2/run-micro2.py`（2026-06-10；待提交到 superpowers-evals 作为 `docs/superpowers/skills/micro-testing-prompt-guidance.md` + 脚本）：

- 每个样本一次 API 调用：system prompt = 现实周围上下文中的 skill 指导变体；user = 现实的中工作流场景；output = 编写的产物（分派提示、计划、报告）。
- 使用 grep 进行程序化评分，查找明确的标记；**在信任判定前手动检查每个匹配**——今晚的一个"违规"是控制者正确引用禁止，自动否定检测误标了另一个。
- 约 $0.15-0.30/样本，每次迭代几秒 vs $12/50 分钟的完整评估运行。在此处迭代措辞；仅当更改是结构性时才在完整运行中确认获胜者。
- 始终包含无指导控制——今晚它既揭示了适得其反（重述：禁止比没有更差）和一个有效的禁止（测试重新运行：3/5 控制失败 vs 两种措辞均为 0/5）。

## 结果：Writing-plans 微测试（2026-06-10 运行，在此 spec 编写之后）

**已解决——无需更改。** 阶段 1（3 任务 spec，无压力）：所有四个变体中的全部 20 个计划中 0 个占位符，包括无指导控制。阶段 1b（10 任务 spec，五个几乎相同的命令诱惑"Similar to Task N"，明确的约 2,500 词经济目标）：40/40 干净——唯一的正则命中是 V2 自我审查*证明*"no TBD/TODO ✓"。当前代 opus 即使在故意压力下也不产生计划占位符，无论有无禁止模式列表。处理方式：保持 No Placeholders 部分完全不变（它成本很低，且反事实无法测量）；不要打开后续 PR。V2 重定位设计留档于此，以防未来模型代际回归。

## 也明确不放弃（已测试并拒绝，有数据）

记录以便没有人无新证据就重新提议它们——完整数字在 2026-06-09 SDD 设计 spec 的成本迭代部分：

- **控制者轮次批处理 / 单条消息中的并行工具调用：** 控制者每条消息恰好发出一个工具调用（每次测量的运行中 0 条多工具消息，无论有无指导）。46% 的控制者轮次是没有工具调用的思考/叙述——一个提示免疫的底线。
- **通过并行调用进行的管道化审查：** 出于相同原因行不通。
- **通过 `run_in_background` 进行的管道化审查：** 机制在提供时被采纳（7/28 分派），但收益低于 45 分钟场景的运行间噪音底线（审查每次仅约 30-60 秒）；增加了双结果流协调。仅对审查单独时间较长的计划值得重新审视。
- **附加到获胜配方的细微差别子句：** 可测量地使其退化（C2：3.8 嘈杂 vs C：3.0 一致）。通过重新推导配方进行迭代，而非通过附加警告。
