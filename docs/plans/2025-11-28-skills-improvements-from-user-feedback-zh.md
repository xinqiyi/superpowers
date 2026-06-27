# 基于用户反馈的 Skills 改进

**日期：** 2025-11-28
**状态：** 草稿
**来源：** 两个在真实开发场景中使用 superpowers 的 Claude 实例

---

## 概述

两个 Claude 实例从实际开发会话中提供了详细反馈。他们的反馈揭示了当前 skills 中的**系统性漏洞**，这些漏洞使得可预防的 bug 尽管遵循了 skills 仍然被发布。

**关键洞察：** 这些是问题报告，而不仅仅是解决方案提议。问题是真实的；解决方案需要仔细评估。

**核心主题：**
1. **验证漏洞**——我们验证操作成功，但不验证它们是否达到了预期结果
2. **进程卫生**——后台进程积累并在 subagent 之间相互干扰
3. **上下文优化**——Subagent 获得太多不相关的信息
4. **缺少自我反思**——没有提示要求代理在移交前审视自己的工作
5. **Mock 安全性**——Mock 可能在未被检测到的情况下与接口发生偏离
6. **Skill 激活**——Skills 存在但没有被读取/使用

---

## 发现的问题

### 问题 1：配置变更验证漏洞

**发生了什么：**
- Subagent 测试了"OpenAI 集成"
- 设置了 `OPENAI_API_KEY` 环境变量
- 收到了状态 200 的响应
- 报告"OpenAI 集成正常工作"
- **但是**响应中包含 `"model": "claude-sonnet-4-20250514"`——实际上使用的是 Anthropic

**根本原因：**
`verification-before-completion` 检查操作是否成功，但不检查结果是否反映了预期的配置变更。

**影响：** 高——集成测试中的虚假信心，bug 被发布到生产环境

**典型的失败模式：**
- 切换 LLM provider → 验证状态 200 但不检查模型名称
- 启用 feature flag → 验证无错误但不检查 feature 是否激活
- 更改环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程积累

**发生了什么：**
- 在会话期间调度了多个 subagent
- 每个 subagent 都启动了后台服务进程
- 进程不断积累（4 个以上服务器在运行）
- 过期进程仍然绑定在端口上
- 后续的 E2E 测试命中了带有错误配置的过期服务器
- 得到混乱/不正确的测试结果

**根本原因：**
Subagent 是无状态的——它们不知道之前 subagent 的进程。没有清理协议。

**影响：** 中高——测试命中错误的服务器，假性通过/失败，调试混乱

---

### 问题 3：Subagent 提示中的上下文臃肿

**发生了什么：**
- 标准方法：给 subagent 完整的计划文件阅读
- 实验方法：仅提供任务 + 模式 + 文件 + 验证命令
- 结果：更快、更专注、一次性完成更常见

**根本原因：**
Subagent 在不相关的计划章节上浪费 token 和注意力。

**影响：** 中——执行速度较慢，失败尝试增多

**有效的方法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：移交前缺乏自我反思

**发生了什么：**
- 添加了自我反思提示："Look at your work with fresh eyes - what could be better?"
- 任务 5 的实施者发现测试失败是由于实现 bug 而非测试 bug
- 追踪到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 创建了无效的 Docker 语法
- 如果没有自我反思，只会报告"测试失败"而不找到根本原因

**根本原因：**
实施者不会自然地退后一步反思自己的工作，然后才报告完成。

**影响：** 中——实施者本可以自己发现的 bug 被移交给了审查者

---

### 问题 5：Mock 与接口的偏差

**发生了什么：**
```typescript
// 接口定义了 close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// 代码（有 bug）调用了 cleanup()
await adapter.cleanup();

// Mock（匹配了 bug）定义了 cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // 错误！
  })),
}));
```
- 测试通过了
- 运行时崩溃："adapter.cleanup is not a function"

**根本原因：**
Mock 源自 bug 代码的调用方式，而非接口定义。TypeScript 无法捕捉带有错误方法名的内联 mock。

**影响：** 高——测试给出虚假信心，运行时崩溃

**为什么 testing-anti-patterns 没有阻止此问题：**
该 skill 涵盖了测试 mock 行为以及在不理解的情况下进行 mock，但没有覆盖"从接口而非实现派生 mock"的具体模式。

---

### 问题 6：代码审查者的文件访问

**发生了什么：**
- 代码审查 subagent 被调度
- 找不到测试文件："The file doesn't appear to exist in the repository"
- 文件实际上存在
- 审查者不知道应该先显式读取文件

**根本原因：**
审查者提示中没有包含显式的文件读取指令。

**影响：** 低中——审查失败或不完整

---

### 问题 7：修复工作流的延迟

**发生了什么：**
- 实施者在自我反思中发现 bug
- 实施者知道如何修复
- 当前工作流：报告 → 我调度修复者 → 修复者修复 → 我验证
- 额外的往返增加了延迟而没有增加价值

**根本原因：**
当实施者已经诊断出问题时，实施者和修复者角色之间的严格分离。

**影响：** 低——延迟，但无正确性问题

---

### 问题 8：Skills 未被读取

**发生了什么：**
- `testing-anti-patterns` skill 存在
- 人类和 subagent 在编写测试前都没有读取它
- 本可以预防一些问题（尽管不是全部——见问题 5）

**根本原因：**
没有强制 subagent 读取相关 skills。没有提示包含 skill 读取。

**影响：** 中——不使用时 skill 投资被浪费

---

## 提议的改进

### 1. verification-before-completion：添加配置变更验证

**添加新章节：**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**为什么这样做有效：**
强制验证**意图**，而不仅仅是操作成功。

---

### 2. subagent-driven-development：为 E2E 测试添加进程卫生

**添加新章节：**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**权衡分析：**
- 增加了提示的样板内容
- 但预防了非常令人困惑的调试过程
- 对于 E2E 测试 subagent 来说是值得的

---

### 3. subagent-driven-development：添加精简上下文选项

**修改第 2 步：使用 Subagent 执行任务**

**修改前：**
```
Read that task carefully from [plan-file].
```

**修改后：**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**示例：**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**为什么这样做有效：**
减少 token 使用，提高专注度，在适当时加快完成速度。

---

### 4. subagent-driven-development：添加自我反思步骤

**修改第 2 步：使用 Subagent 执行任务**

**添加到提示模板：**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**为什么这样做有效：**
在移交前捕捉实施者自己可以发现的 bug。记录的案例：通过自我反思发现了 entrypoint bug。

**权衡：**
每个任务增加约 30 秒，但在审查前就能发现问题。

---

### 5. requesting-code-review：添加显式的文件读取

**修改 code-reviewer 模板：**

**在开头添加：**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**为什么这样做有效：**
显式指令防止"找不到文件"的问题。

---

### 6. testing-anti-patterns：添加 Mock-接口偏差反模式

**添加新的反模式 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

When you see runtime error "X is not a function" and tests pass:
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**为什么这样做有效：**
直接针对反馈中发现的失败模式。

---

### 7. subagent-driven-development：要求测试 Subagent 读取 Skills

**当任务涉及测试时，添加到提示模板：**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**为什么这样做有效：**
确保 skills 被实际使用，而不仅仅是存在。

**权衡：**
每个任务增加时间，但预防了整类 bug。

---

### 8. subagent-driven-development：允许实施者修复自己发现的问题

**修改第 2 步：**

**当前：**
```
Subagent reports back with summary of work.
```

**提议：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为什么这样做有效：**
当实施者已经知道如何修复时减少延迟。记录的案例：本可以为 entrypoint bug 节省一次往返。

**权衡：**
提示略微复杂，但端到端速度更快。

---

## 实施计划

### 第一阶段：高影响、低风险（优先执行）

1. **verification-before-completion：配置变更验证**
   - 清晰的补充，不改变现有内容
   - 解决高影响问题（测试中的虚假信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock-接口偏差**
   - 添加新的反模式，不修改现有内容
   - 解决高影响问题（运行时崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 简单的模板补充
   - 修复具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### 第二阶段：中等程度的变更（仔细测试）

4. **subagent-driven-development：进程卫生**
   - 添加新章节，不改变工作流
   - 解决中高影响（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自我反思**
   - 更改提示模板（风险较高）
   - 但有记录表明能发现 bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：要求读取 Skills**
   - 增加提示开销
   - 但确保 skills 被实际使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### 第三阶段：优化（先验证）

7. **subagent-driven-development：精简上下文选项**
   - 增加复杂性（两种方法）
   - 需要验证不会引起混淆
   - 文件：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允许实施者修复**
   - 更改工作流（风险较高）
   - 优化，而非 bug 修复
   - 文件：`skills/subagent-driven-development/SKILL.md`

---

## 待解决的问题

1. **精简上下文方法：**
   - 是否应该将其设为基于模式的任务的默认方式？
   - 如何决定使用哪种方法？
   - 过于精简而遗漏重要上下文的风险？

2. **自我反思：**
   - 是否会显著拖慢简单任务？
   - 是否只应适用于复杂任务？
   - 如何防止"反思疲劳"（变成例行公事）？

3. **进程卫生：**
   - 应该放在 subagent-driven-development 还是单独的 skill 中？
   - 是否适用于 E2E 测试之外的其他工作流？
   - 如何处理进程应该持久存在的情况（开发服务器）？

4. **强制读取 Skills：**
   - 是否应该要求所有 subagent 读取相关 skills？
   - 如何防止提示变得过长？
   - 过度记录而失去重点的风险？

---

## 成功指标

如何知道这些改进有效？

1. **配置验证：**
   - 零次"测试通过但使用了错误配置"的实例
   - Jesse 不会说"那实际上并没有测试你想象的内容"

2. **进程卫生：**
   - 零次"测试命中了错误的服务器"的实例
   - E2E 测试运行期间无端口冲突错误

3. **Mock-接口偏差：**
   - 零次"测试通过但运行时因缺少方法崩溃"的实例
   - Mock 和接口之间无方法名称不匹配

4. **自我反思：**
   - 可测量：实施者报告中是否包含自我反思发现？
   - 定性：更少的 bug 进入代码审查阶段？

5. **读取 Skills：**
   - Subagent 报告引用了 skill gate 函数
   - 代码审查中更少的反模式违规

---

## 风险和缓解措施

### 风险：提示臃肿
**问题：** 添加所有这些要求使提示变得过于繁重
**缓解：**
- 分阶段实施（不要一次性添加所有内容）
- 使某些添加具有条件性（仅 E2E 测试的 E2E 卫生）
- 考虑为不同任务类型使用模板

### 风险：分析瘫痪
**问题：** 过多的反思/验证拖慢了执行
**缓解：**
- 保持 gate 函数快速（秒级，而非分钟级）
- 初始阶段使精简上下文为 opt-in
- 监控任务完成时间

### 风险：虚假安全感
**问题：** 遵循检查清单并不能保证正确性
**缓解：**
- 强调 gate 函数是最低要求，而非最高要求
- 在 skills 中保留"运用判断力"的语言
- 记录 skills 能发现常见失败，而非所有失败

### 风险：Skill 分歧
**问题：** 不同的 skills 给出矛盾的建议
**缓解：**
- 审查所有 skills 的更改以保持一致性
- 记录 skills 如何交互（集成章节）
- 在部署前使用真实场景进行测试

---

## 建议

**立即执行第一阶段：**
- verification-before-completion：配置变更验证
- testing-anti-patterns：Mock-接口偏差
- requesting-code-review：显式文件读取

**在最终确定前与 Jesse 测试第二阶段：**
- 获取关于自我反思影响的反馈
- 验证进程卫生方法
- 确认要求读取 Skills 的开销值得

**推迟第三阶段，等待验证：**
- 精简上下文需要真实世界的测试
- 实施者修复工作流更改需要仔细评估

这些更改解决了用户记录的真实问题，同时最小化了使 skills 变得更糟的风险。
