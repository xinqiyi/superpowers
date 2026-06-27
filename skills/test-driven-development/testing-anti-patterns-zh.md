# 测试反模式

**在以下情况下加载此参考：** 编写或更改测试、添加 mock 或受到向生产代码添加仅测试使用的方法的诱惑时。

## 概述

测试必须验证真实行为，而不是 mock 的行为。Mock 是一种隔离手段，而不是被测试的对象。

**核心原则：** 测试代码做什么，而不是 mock 做什么。

**遵循严格的 TDD 可以防止这些反模式。**

## 铁律

```
1. NEVER test mock behavior
2. NEVER add test-only methods to production classes
3. NEVER mock without understanding dependencies
```

## 反模式 1：测试 Mock 行为

**违规：**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错误的：**
- 你在验证 mock 有效，而不是组件有效
- 测试在 mock 存在时通过，不存在时失败
- 这没告诉你任何关于真实行为的信息

**你的人类搭档的纠正：** "我们是在测试一个 mock 的行为吗？"

**修复：**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### 门控函数

```
BEFORE asserting on any mock element:
  Ask: "Am I testing real component behavior or just mock existence?"

  IF testing mock existence:
    STOP - Delete the assertion or unmock the component

  Test real behavior instead
```

## 反模式 2：生产代码中的仅测试方法

**违规：**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**为什么这是错误的：**
- 生产类被仅测试的代码污染
- 如果在生产中被意外调用会很危险
- 违反了 YAGNI 和关注点分离
- 混淆了对象生命周期与实体生命周期

**修复：**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### 门控函数

```
BEFORE adding any method to production class:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

## 反模式 3：在没有理解的情况下 Mock

**违规：**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**为什么这是错误的：**
- Mock 的方法具有测试所依赖的副作用（写入配置）
- 为了"安全"而过度 mock 破坏了实际行为
- 测试因错误原因通过或神秘地失败

**修复：**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### 门控函数

```
BEFORE mocking any method:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## 反模式 4：不完整的 Mock

**违规：**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**为什么这是错误的：**
- **部分 mock 隐藏了结构假设** - 你只 mock 了你了解的字段
- **下游代码可能依赖你未包含的字段** - 静默失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假的自信** - 测试未证明任何关于真实行为的内容

**铁律：** Mock 现实中存在的完整数据结构，而不仅仅是你的即时测试使用的字段。

**修复：**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### 门控函数

```
BEFORE creating mock responses:
  Check: "What fields does the real API response contain?"

  Actions:
    1. Examine actual API response from docs/examples
    2. Include ALL fields system might consume downstream
    3. Verify mock matches real response schema completely

  Critical:
    If you're creating a mock, you must understand the ENTIRE structure
    Partial mocks fail silently when code depends on omitted fields

  If uncertain: Include all documented fields
```

## 反模式 5：集成测试作为事后想法

**违规：**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**为什么这是错误的：**
- 测试是实现的一部分，不是可选的后续工作
- TDD 本可以捕获这种情况
- 没有测试就不能声称完成

**修复：**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## 当 Mock 变得过于复杂时

**警告信号：**
- Mock 设置比测试逻辑还长
- 为了让测试通过 mock 所有东西
- Mock 缺少真实组件拥有的方法
- 测试在 mock 更改时失败

**你的人类搭档的问题：** "我们这里需要使用 mock 吗？"

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么
2. **看着它失败** → 确认测试测试真实行为，而不是 mock
3. **最小化实现** → 不会渗入仅测试方法
4. **真实依赖** → 你在 mock 之前看到测试实际需要什么

**如果你在测试 mock 行为，你就违反了 TDD**——你在没有先看着测试对真实代码失败的情况下添加了 mock。

## 快速参考

| Anti-Pattern | Fix |
|--------------|-----|
| Assert on mock elements | Test real component or unmock it |
| Test-only methods in production | Move to test utilities |
| Mock without understanding | Understand dependencies first, mock minimally |
| Incomplete mocks | Mirror real API completely |
| Tests as afterthought | TDD - tests first |
| Over-complex mocks | Consider integration tests |

## 红旗

- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中被调用
- Mock 设置占测试的 50% 以上
- 移除 mock 时测试失败
- 无法解释为什么需要 mock
- "只是为了安全"而 mock

## 底线

**Mock 是隔离的工具，不是被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，你走错了方向。

修复：测试真实行为，或质疑你为什么要 mock 任何东西。
