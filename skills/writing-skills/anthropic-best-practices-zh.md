# 技能编写最佳实践

> 了解如何编写 agent 能够发现并成功使用的有效 Skills。

好的 Skills 简洁、结构良好，并经过真实使用测试。本指南提供实用的编写决策，帮助你编写 agent 能够发现并有效使用的 Skills。

有关 Skills 如何工作的概念背景，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows) 是一种公共资源。你的 Skill 与 agent 需要知道的所有其他内容共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他 Skills 的元数据
* 你的实际请求

并非 Skill 中的每个 token 都有即时成本。在启动时，只预加载所有 Skills 的元数据（名称和描述）。Agent 只在 Skill 变得相关时才读取 SKILL.md，并且只在需要时才读取其他文件。然而，在 SKILL.md 中保持简洁仍然很重要：一旦 agent 加载了它，每个 token 都与对话历史和其他上下文竞争。

**默认假设：** Agent 已经非常聪明

只添加 agent 还没有的上下文。质疑每条信息：

* "Agent 真的需要这个解释吗？"
* "我能假设 agent 知道这个吗？"
* "这段文字值得它的 token 成本吗？"

**好例子：简洁**（约 50 tokens）：

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏例子：过于冗长**（约 150 tokens）：

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

简洁版本假设 agent 知道什么是 PDF 以及库如何工作。

### 设置适当的自由度

将具体程度与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

在以下情况使用：

* 多种方法都有效
* 决策依赖于上下文
* 启发式方法指导方案

示例：

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**中自由度**（伪代码或带参数的脚本）：

在以下情况使用：

* 存在首选模式
* 允许一定的变化
* 配置影响行为

示例：

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**低自由度**（特定脚本，很少或没有参数）：

在以下情况使用：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**类比：** 把 agent 想象成一个探索路径的机器人：

* **两侧是悬崖的窄桥**：只有一条安全的前进道路。提供具体的护栏和精确指令（低自由度）。示例：必须按确切顺序运行的数据库迁移。
* **没有危险的开放田野**：许多路径通向成功。给出大致方向并相信 agent 能找到最佳路线（高自由度）。示例：上下文决定最佳方法的代码审查。

### 使用你计划使用的所有模型进行测试

Skills 作为模型的补充，因此有效性取决于底层模型。使用你计划使用的所有模型测试你的 Skill。

**按模型的测试考虑：**

* **Claude Haiku**（快速、经济）：Skill 是否提供了足够的指导？
* **Claude Sonnet**（平衡）：Skill 是否清晰高效？
* **Claude Opus**（强大推理）：Skill 是否避免了过度解释？

在 Opus 上完美有效的内容可能需要在 Haiku 上提供更多细节。如果你计划跨多个模型使用你的 Skill，请针对所有模型都有效的指令。

## Skill 结构

<Note>
  **YAML 前置元数据**：SKILL.md 的前置元数据需要两个字段：

  * `name` - Skill 的人类可读名称（最多 64 个字符）
  * `description` - 一行描述 Skill 做什么以及何时使用（最多 1024 个字符）

  有关完整的 Skill 结构详细信息，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式，使 Skills 更容易被引用和讨论。我们建议对 Skill 名称使用**动名词形式**（动词 + -ing），因为它清楚地描述了 Skill 提供的活动或能力。

**好的命名示例（动名词形式）：**

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代方案：**

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 行动导向："Process PDFs"、"Analyze Spreadsheets"

**避免：**

* 模糊的名称："Helper"、"Utils"、"Tools"
* 过于泛化："Documents"、"Data"、"Files"
* 你的 skill 集合中的不一致模式

一致的命名使得：

* 在文档和对话中引用 Skills 更容易
* 一目了然地理解 Skill 的作用
* 组织和搜索多个 Skills
* 维护专业、一致的 skill 库

### 编写有效的 description

`description` 字段支持 Skill 发现，应包括 Skill 做什么以及何时使用。

<Warning>
  **始终以第三人称编写**。Description 被注入到系统提示中，不一致的视角可能导致发现问题。

  * **好的：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**要具体并包含关键术语**。包括 Skill 做什么以及何时使用的特定触发条件/上下文。

每个 Skill 只有一个 description 字段。Description 对技能选择至关重要：agent 使用它从可能 100+ 个可用 Skills 中选择正确的 Skill。你的 description 必须提供足够的细节，让 agent 知道何时选择此 Skill，而 SKILL.md 的其余部分提供实施细节。

有效的示例：

**PDF Processing skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免像这样的模糊 description：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 作为一个概览，根据需要引导 agent 获取详细材料，就像入门指南中的目录一样。有关渐进式披露如何工作的解释，请参阅概述中的 [Skills 如何工作](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指导：**

* 保持 SKILL.md 正文在 500 行以下以获得最佳性能
* 当接近此限制时，将内容拆分为单独的文件
* 使用下面的模式有效地组织指令、代码和资源

#### 视觉概览：从简单到复杂

一个基本的 Skill 从一个 SKILL.md 文件开始，包含元数据和指令：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着你的 Skill 增长，你可以打包仅在需要时由 agent 加载的额外内容：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的 Skill 目录结构可能如下所示：

```
pdf/
├── SKILL.md              # Main instructions (loaded when triggered)
├── FORMS.md              # Form-filling guide (loaded as needed)
├── reference.md          # API reference (loaded as needed)
├── examples.md           # Usage examples (loaded as needed)
└── scripts/
    ├── analyze_form.py   # Utility script (executed, not loaded)
    ├── fill_form.py      # Form filling script
    └── validate.py       # Validation script
```

#### 模式 1：带参考的高级指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

Agent 只在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

对于具有多个领域的 Skills，按领域组织内容以避免加载不相关的上下文。当用户询问销售指标时，agent 只需要读取与销售相关的模式，而不是财务或市场数据。这保持了低 token 使用量和聚焦的上下文。

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing → See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts → See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption → See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email → See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件式细节

显示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

Agent 只在用户需要这些功能时读取 REDLINING.md 或 OOXML.md。

### 避免深层嵌套引用

Agent 可能会在从其他引用的文件引用时部分读取文件。当遇到嵌套引用时，agent 可能使用像 `head -100` 这样的命令来预览内容，而不是读取整个文件，导致信息不完整。

**保持从 SKILL.md 一层深的引用**。所有参考文件应该直接从 SKILL.md 链接，以确保 agent 在需要时读取完整文件。

**坏例子：太深**：

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**好例子：一层深**：

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 为较长的参考文件添加目录

对于超过 100 行的参考文件，在顶部包含一个目录。这确保了即使通过部分读取预览时，agent 也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

Agent 然后可以根据需要读取完整文件或跳转到特定部分。

有关这种基于文件系统的架构如何实现渐进式披露的详细信息，请参阅以下高级部分中的 [运行时环境](#runtime-environment) 部分。

## 工作流和反馈循环

### 对复杂任务使用工作流

将复杂操作分解为清晰、顺序的步骤。对于特别复杂的工作流，提供一个 agent 可以复制到其响应中并逐个勾选的检查清单。

**示例 1：研究综合工作流**（适用于没有代码的 Skills）：

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

此示例展示了工作流如何应用于不需要代码的分析任务。检查清单模式适用于任何复杂的多步骤过程。

**示例 2：PDF 表单填写工作流**（适用于有代码的 Skills）：

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

清晰的步骤防止 agent 跳过关键的验证。检查清单帮助你和 agent 跟踪多步骤工作流的进度。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

这种模式极大提高了输出质量。

**示例 1：风格指南合规**（适用于没有代码的 Skills）：

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

这展示了使用参考文档而不是脚本的验证循环模式。"验证器"是 STYLE_GUIDE.md，agent 通过阅读和比较来执行检查。

**示例 2：文档编辑过程**（适用于有代码的 Skills）：

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

验证循环及早捕获错误。

## 内容指南

### 避免时效性信息

不要包含会过时的信息：

**坏例子：有时效性**（会变成错误的）：

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好例子**（使用"旧模式"部分）：

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

旧模式部分提供历史上下文而不会杂乱主要内容。

### 使用一致的术语

选择一个术语并在整个 Skill 中使用：

**好的 - 一致**：

* 始终使用"API endpoint"
* 始终使用"field"
* 始终使用"extract"

**差的 - 不一致**：

* 混用"API endpoint"、"URL"、"API route"、"path"
* 混用"field"、"box"、"element"、"control"
* 混用"extract"、"pull"、"get"、"retrieve"

一致性帮助 agent 理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。根据你的需求匹配严格程度。

**对于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**对于灵活指导**（当适应有用时）：

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Taylor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### 示例模式

对于输出质量取决于看到示例的 Skills，提供输入/输出对，就像常规提示中一样：

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

示例帮助 agent 理解所需的风格和详细程度，比单独的描述更清晰。

### 条件工作流模式

引导 agent 通过决策点：

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  如果工作流变得庞大或复杂，考虑将它们推送到单独的文件中，并告诉 agent 根据手头的任务读取相应的文件。
</Tip>

## 评估和迭代

### 先构建评估

**在编写大量文档之前创建评估。** 这确保你的 Skill 解决实际问题，而不是记录想象的问题。

**评估驱动开发：**

1. **识别缺口**：在没有 Skill 的情况下让 agent 运行代表性任务。记录具体的失败或缺失的上下文
2. **创建评估**：构建测试这些缺口的三个场景
3. **建立基线**：测量没有 Skill 时 agent 的性能
4. **编写最小指令**：创建刚好足以解决缺口并通过评估的内容
5. **迭代**：执行评估，与基线比较，并优化

这种方法确保你解决的是实际问题，而不是预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  此示例演示了一个数据驱动的评估和简单的测试标准。我们目前不提供运行这些评估的内置方法。用户可以创建自己的评估系统。评估是你衡量 Skill 有效性的真相来源。
</Note>

### 与 agent 迭代开发 Skill

最有效的 Skill 开发过程涉及 agent 本身。与一个实例（"Agent A"）一起创建一个将被其他实例（"Agent B"）使用的 Skill。Agent A 帮助你设计和细化指令，而 Agent B 在实际任务中测试它们。这之所以有效，是因为底层模型既了解如何编写有效的 agent 指令，也了解 agent 需要哪些信息。

**创建新的 Skill：**

1. **在没有 Skill 的情况下完成任务**：与 Agent A 一起使用常规提示解决问题。在工作中，你会自然地提供上下文、解释偏好并分享过程知识。注意你反复提供的信息。

2. **识别可重用模式**：完成任务后，识别你提供的上下文中有哪些对类似的未来任务有用。

   **示例**：如果你完成了一个 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **让 Agent A 创建 Skill**："创建一个捕捉我们刚使用的这个 BigQuery 分析模式的 Skill。包括表模式、命名约定和关于过滤测试账户的规则。"

   <Tip>
    现代 agent 原生理解 Skill 格式和结构。你不需要特殊的系统提示或"编写技能"技能来获得创建 Skills 的帮助。只需让 agent 创建一个 Skill，它就会生成带有适当前置元数据和正文内容的正确结构的 SKILL.md 内容。
   </Tip>

4. **检查简洁性**：检查 Agent A 是否添加了不必要的解释。问："删除关于胜率含义的解释——agent 已经知道这个。"

5. **改进信息架构**：让 Agent A 更有效地组织内容。例如："组织这个内容，使得表模式在单独的参考文件中。我们以后可能添加更多表。"

6. **在类似任务上测试**：在相关用例上用 Agent B（一个加载了 Skill 的新实例）使用该 Skill。观察 Agent B 是否找到正确的信息、正确应用规则并成功完成任务。

7. **基于观察迭代**：如果 Agent B 挣扎或遗漏了某些东西，带着具体信息回到 Agent A："当 agent 使用这个 Skill 时，它忘了按日期过滤 Q4。我们应该添加一个关于日期过滤模式的部分吗？"

**迭代现有 Skills：**

改进 Skill 时也遵循相同的层级模式。你在以下之间交替：

* **与 Agent A 合作**（帮助细化 Skill 的专家）
* **用 Agent B 测试**（使用 Skill 执行实际工作的 agent）
* **观察 Agent B 的行为**并将见解带回给 Agent A

1. **在实际工作流中使用 Skill**：给 Agent B（加载了 Skill）实际任务，而不是测试场景

2. **观察 Agent B 的行为**：注意它在哪些方面挣扎、成功或做出意外选择

   **示例观察**："当我让 Agent B 做区域销售报告时，它编写了查询但忘了过滤测试账户，即使 Skill 提了这个规则。"

3. **回到 Agent A 进行改进**：分享当前的 SKILL.md 并描述你观察到的内容。问："我注意到当我要求区域报告时，Agent B 忘了过滤测试账户。Skill 提到了过滤，但可能不够突出？"

4. **审查 Agent A 的建议**：Agent A 可能建议重新组织以使规则更突出，使用更强的语言如"必须过滤"而不是"始终过滤"，或重构工作流部分。

5. **应用并测试更改**：用 Agent A 的改进更新 Skill，然后在类似请求上再次用 Agent B 测试

6. **根据使用情况重复**：随着你遇到新场景，继续这个观察-细化-测试的循环。每次迭代都基于真实 agent 行为而不仅仅是假设来改进 Skill。

**收集团队反馈：**

1. 与队友分享 Skills 并观察他们的使用
2. 问：Skill 是否按预期激活？指令是否清晰？缺失了什么？
3. 整合反馈以解决你自己使用模式中的盲点

**为什么这种方法有效**：Agent A 理解 agent 需求，你提供领域专业知识，Agent B 通过真实使用揭示缺口，迭代细化基于观察到的行为而不是假设来改进 Skills。

### 观察 agent 如何导航 Skills

当你迭代 Skills 时，注意 agent 在实践中如何实际使用它们。注意：

* **意外的探索路径**：Agent 是否以你未预料到的顺序读取文件？这可能表明你的结构不像你想象的那么直观
* **遗漏的连接**：Agent 是否未能跟随对重要文件的引用？你的链接可能需要更明确或更突出
* **过度依赖某些部分**：如果 agent 反复读取同一文件，考虑该内容是否应该放在主 SKILL.md 中
* **被忽略的内容**：如果 agent 从未访问打包文件，它可能是不必要的，或者在主指令中信号不充分

基于这些观察而不是假设进行迭代。Skill 元数据中的 'name' 和 'description' 尤其关键。Agent 在决定是否针对当前任务触发 Skill 时使用它们。确保它们清楚描述 Skill 做什么以及何时应该使用。

## 要避免的反模式

### 避免 Windows 风格路径

始终在文件路径中使用正斜杠，即使在 Windows 上：

* ✓ **好的**：`scripts/helper.py`，`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`，`reference\guide.md`

Unix 风格路径在所有平台上都有效，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供过多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**坏例子：太多选择**（令人困惑）：
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**好例子：提供默认**（带有逃生口）：
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 高级：带有可执行代码的 Skills

以下部分侧重于包含可执行脚本的 Skills。如果你的 Skill 只使用 markdown 指令，请跳到 [有效 Skills 的检查清单](#checklist-for-effective-skills)。

### 解决问题，而不是推卸责任

在编写 Skills 的脚本时，处理错误条件，而不是推卸给 agent。

**好例子：显式处理错误**：

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**坏例子：推卸给 agent**：

```python  theme={null}
def process_file(path):
    # Just fail and let the agent figure it out
    return open(path).read()
```

配置参数也应该被证明和记录，以避免"巫毒常量"（Ousterhout 定律）。如果你不知道正确的值，agent 如何确定它？

**好例子：自我文档化**：

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**坏例子：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 提供工具脚本

即使你的 agent 可以编写脚本，预制脚本也有优势：

**工具脚本的好处**：

* 比生成的代码更可靠
* 节省 token（无需在上下文中包含代码）
* 节省时间（无需代码生成）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上图展示了可执行脚本如何与指令文件一起工作。指令文件（forms.md）引用脚本，agent 可以在不将脚本内容加载到上下文的情况下执行它。

**重要区别**：在你的指令中明确 agent 应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 提取字段"
* **作为参考阅读**（用于复杂逻辑）："参见 `analyze_form.py` 了解字段提取算法"

对于大多数工具脚本，执行是首选，因为它更可靠和高效。请参阅下面的 [运行时环境](#runtime-environment) 部分，了解脚本执行如何工作。

**示例**：

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 agent 分析它们：

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. The agent can see field locations and types visually
````

<Note>
  在这个例子中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Agent 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 agent 执行复杂的、开放式的任务时，他们可能会犯错。"计划-验证-执行"模式通过让 agent 首先以结构化格式创建计划，然后用脚本验证计划，再执行它来及早捕获错误。

**示例**：想象让 agent 根据电子表格更新 PDF 中的 50 个表单字段。没有验证，它可能引用不存在的字段、创建冲突的值、错过必填字段或错误地应用更新。

**解决方案**：使用上面展示的工作流模式（PDF 表单填写），但添加一个中间 `changes.json` 文件，在应用更改之前进行验证。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**为什么这种模式有效：**

* **及早捕获错误**：验证在更改应用之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的计划**：Agent 可以在不触及原始文件的情况下迭代计划
* **清晰的调试**：错误消息指向具体问题

**何时使用**：批处理操作、破坏性更改、复杂的验证规则、高风险操作。

**实现技巧**：使验证脚本的详细错误消息具体，如"Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed"，以帮助 agent 修复问题。

### 打包依赖

Skills 在代码执行环境中运行，具有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，并从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问权限，也没有运行时包安装

在你的 SKILL.md 中列出所需的包，并在 [代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) 中验证它们是否可用。

### 运行时环境

Skills 在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。有关此架构的概念性解释，请参阅概述中的 [Skills 架构](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这如何影响你的编写：**

**Agent 如何访问 Skills：**

1. **元数据预加载**：启动时，所有 Skills 的 YAML 前置元数据中的名称和描述被加载到系统提示中
2. **文件按需读取**：Agent 在需要时使用它们的文件读取工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：工具脚本可以通过 bash 执行，无需将其完整内容加载到上下文中。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际读取之前不消耗上下文 token

* **文件路径很重要**：Agent 像文件系统一样导航你的 skill 目录。使用正斜杠（`reference/guide.md`），而不是反斜杠
* **描述性地命名文件**：使用指示内容的名称：`form_validation_rules.md`，而不是 `doc2.md`
* **为发现而组织**：按领域或功能组织目录
  * 好的：`reference/finance.md`，`reference/sales.md`
  * 差的：`docs/file1.md`，`docs/file2.md`
* **打包全面资源**：包括完整的 API 文档、广泛的示例、大数据集；访问前无上下文惩罚
* **对于确定性操作优先使用脚本**：编写 `validate_form.py` 而不是让 agent 生成验证代码
* **明确执行意图**：
  * "运行 `analyze_form.py` 提取字段"（执行）
  * "参见 `analyze_form.py` 了解提取算法"（作为参考阅读）
* **测试文件访问模式**：通过实际请求测试 agent 是否能导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md (overview, points to reference files)
└── reference/
    ├── finance.md (revenue metrics)
    ├── sales.md (pipeline data)
    └── product.md (usage analytics)
```

当用户询问收入时，agent 读取 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 只读取该文件。sales.md 和 product.md 文件保留在文件系统中，在需要之前消耗零上下文 token。这种基于文件系统的模型正是渐进式披露的实现方式。Agent 可以导航并选择性地加载每个任务所需的内容。

关于技术架构的完整详细信息，请参阅 Skills 概述中的 [Skills 如何工作](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的 Skill 使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名称以避免"工具未找到"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器内的工具名称

没有服务器前缀，agent 可能无法定位工具，尤其是当多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包是可用的：

````markdown  theme={null}
**坏例子：假设已安装**：
"Use the pdf library to process the file."

**好例子：明确依赖**：
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML 前置元数据要求

SKILL.md 前置元数据需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。参见 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure) 了解完整的结构细节。

### Token 预算

保持 SKILL.md 正文在 500 行以下以获得最佳性能。如果你的内容超过此限制，使用前面描述的渐进式披露模式将其拆分为单独的文件。有关架构细节，请参见 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效 Skills 的检查清单

在分享 Skill 之前，验证：

### 核心质量

* [ ] Description 是具体的并包含关键术语
* [ ] Description 包括 Skill 做什么以及何时使用
* [ ] SKILL.md 正文在 500 行以下
* [ ] 额外细节在单独的文件中（如果需要）
* [ ] 没有时效性信息（或在"旧模式"部分）
* [ ] 整个过程中术语一致
* [ ] 示例是具体的，而不是抽象的
* [ ] 文件引用是一个层级深
* [ ] 适当使用了渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而不是推卸给 agent
* [ ] 错误处理明确且有用
* [ ] 没有"巫毒常量"（所有值都有理由）
* [ ] 所需的包在指令中列出并验证可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格路径（全部使用正斜杠）
* [ ] 关键操作的验证/核查步骤
* [ ] 对质量关键的任务包含反馈循环

### 测试

* [ ] 至少创建了三个评估
* [ ] 用 Haiku、Sonnet 和 Opus 测试了
* [ ] 用真实使用场景测试了
* [ ] 纳入了团队反馈（如果适用）

## 下一步

<CardGroup cols={2}>
  <Card title="开始使用 Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    创建你的第一个 Skill
  </Card>

  <Card title="在 Claude Code 中使用 Skills" icon="terminal" href="https://code.claude.com/docs/en/skills">
    在 Claude Code 中创建和管理 Skills
  </Card>

  <Card title="通过 API 使用 Skills" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    以编程方式上传和使用 Skills
  </Card>
</CardGroup>
