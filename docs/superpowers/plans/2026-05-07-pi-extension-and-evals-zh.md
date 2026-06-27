# Pi 扩展和 Evals 实施计划

> **对于 agentic workers：** 必需的子 skill：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实施此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 为 Superpowers 添加一流的 Pi 包支持，并将 Pi 添加为 Drill 评估后端。

**架构：** Pi 包在根 `package.json` 中声明，并加载现有的 `skills/` 加上一个小的 Pi 扩展。扩展将 `using-superpowers` bootstrap 作为用户角色消息注入到 provider context 中，在会话启动和压缩后，并包含 Pi 特定的工具映射。Drill 获得一个 `pi` 后端、Pi 会话日志标准化和测试。

**技术栈：** Pi TypeScript 扩展 API、Node 内置测试运行器、Drill Python 评估框架、pytest。

---

### 任务 1：Pi 包清单和扩展测试

**文件：**
- 修改：`package.json`
- 创建：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：编写失败的包/扩展测试**

创建 `tests/pi/test-pi-extension.mjs`，包含导入 `extensions/superpowers.ts`、注册假 Pi 处理器并断言的测试：
- 根 `package.json` 有包含 `pi-package` 的 `keywords`
- 根 `package.json` 有 `pi.skills: ["./skills"]`
- 根 `package.json` 有 `pi.extensions: ["./extensions/superpowers.ts"]`
- 扩展注册了 `resources_discover`、`session_start`、`session_compact`、`context` 和 `agent_end`
- 启动 `context` 恰好注入一条用户角色 bootstrap 消息
- `agent_end` 清除启动注入
- `session_compact` 重新启用注入
- 扩展未注册 `session_before_compact`

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：失败，因为 `extensions/superpowers.ts` 不存在且 `package.json` 缺少 `pi` 清单。

- [ ] **步骤 3：实现清单字段**

更新 `package.json`，添加 `description`、`keywords`、`pi.extensions` 和 `pi.skills`，同时保留现有的 `name`、`version`、`type` 和 `main`。

- [ ] **步骤 4：实现 `extensions/superpowers.ts`**

创建零运行时依赖的扩展，能够：
- 从 `import.meta.url` 定位包根目录
- 读取 `skills/using-superpowers/SKILL.md`
- 去除 YAML frontmatter
- 追加 Pi 特定的工具映射
- 暴露带有 skills 路径的 `resources_discover`
- 在 `session_start` 和 `session_compact` 上标记待处理的 bootstrap
- 在 `context` 中注入一条用户角色 bootstrap 消息
- 在前导的 `compactionSummary` 消息之后插入压缩后 bootstrap
- 在 `agent_end` 上清除待处理的 bootstrap

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：通过。

### 任务 2：Pi 工具映射参考

**文件：**
- 创建：`skills/using-superpowers/references/pi-tools.md`
- 修改：`tests/pi/test-pi-extension.mjs`

- [ ] **步骤 1：为 Pi 参考文档编写失败的测试**

添加断言，确认 `skills/using-superpowers/references/pi-tools.md` 存在并记录了 `Skill`、`Task`、`TodoWrite` 和内置工具名称的映射。

- [ ] **步骤 2：运行测试并验证 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：失败，因为 `pi-tools.md` 不存在。

- [ ] **步骤 3：添加 Pi 参考文档**

创建 `skills/using-superpowers/references/pi-tools.md`，解释 Pi 原生 skills、可选的 `pi-subagents`、无规范的 todo/tasklist plugin 以及内置小写工具。

- [ ] **步骤 4：运行测试并验证 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：通过。

### 任务 3：Drill Pi 后端和会话日志标准化

**文件：**
- 创建：`evals/backends/pi.yaml`
- 修改：`evals/drill/backend.py`
- 修改：`evals/drill/engine.py`
- 修改：`evals/drill/normalizer.py`
- 修改：`evals/tests/test_backend.py`
- 修改：`evals/tests/test_normalizer.py`

- [ ] **步骤 1：编写失败的后端/标准化测试**

为以下内容添加 pytest 覆盖：
- `load_backend("pi")` 返回 `family == "pi"`
- Pi 后端命令以 `pi` 开头并包含 `-e ${SUPERPOWERS_ROOT}`
- Pi 的 `_resolve_log_dir()` 指向 `~/.pi/agent/sessions`
- `filter_pi_logs_by_cwd()` 仅保留头部 `cwd` 与场景工作目录匹配的会话文件
- `normalize_pi_logs()` 从 Pi 助手会话条目中提取 `toolCall` 块，并将内置小写工具映射到规范名称

- [ ] **步骤 2：运行测试并验证 RED**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：失败，因为 Pi 后端和标准化器不存在。

- [ ] **步骤 3：添加 `evals/backends/pi.yaml`**

配置后端以运行 `pi -e ${SUPERPOWERS_ROOT}`，使用宽松的 TUI 就绪检查、`/quit` 关闭和 Pi 会话日志位置。

- [ ] **步骤 4：实现 Pi 系列支持**

使用 Pi 日志过滤和标准化更新 `Backend.family`、`Engine._resolve_log_dir`、`Engine._collect_tool_calls` 和 `normalizer.py`。

- [ ] **步骤 5：运行测试并验证 GREEN**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：通过。

### 任务 4：文档和全面验证

**文件：**
- 修改：`README.md`
- 修改：`evals/README.md`

- [ ] **步骤 1：记录 Pi 安装和评估后端**

将 Pi 添加到 README 快速开始/安装列表，并将后端条目/用法添加到 `evals/README.md`。

- [ ] **步骤 2：运行验证**

运行：
```bash
node --experimental-strip-types --test tests/pi/test-pi-extension.mjs
uv run pytest evals/tests/test_backend.py evals/tests/test_setup.py evals/tests/test_normalizer.py -q
```

预期：所有测试通过。
