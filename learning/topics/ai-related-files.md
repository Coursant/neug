# NeuG AI-Related Files And Usage

## 原始问题

```text
详细分析这个主文件夹下全部和ai有关的文件，以及如何使用，写入笔记
```

## 我的理解

- 这里的“主文件夹”理解为仓库根目录 `/home/lcc/cpp/neug`。
- “AI 有关的文件”不是指文件名里偶然包含 `ai` 的文件，例如 `main`、`container`、`metadata`，而是指给 LLM/coding agent/AI-assisted workflow/CodeGraph/Spec-Kit 使用的文件。
- 本笔记只基于本地文件静态阅读，没有实际调用 GitHub CLI、CodeGraph CLI 或 agent slash commands。

## 文件总览

NeuG 根目录下 AI 相关内容大致分四组：

```text
1. Agent 协作规则
   AGENTS.md
   .cursor/rules/codegraph.mdc
   .agents/                 // 当前为空
   .codex/                  // 当前为空

2. AI-assisted development workflow
   README.md
   CONTRIBUTING.md
   doc/source/development/ai_coding.md
   scripts/init_skills.sh
   .cursor/skills/**

3. Spec-Driven workflow support
   .specify/**
   specs/**

4. CodeGraph / CodeScope
   skills/codegraph/**
   doc/source/tutorials/codegraph-openclaw-example.md
   .codegraph/**
```

另外，`doc/README.md` 里提到 `qwen-translator`，用于文档同步和 Qwen3 AI 翻译。

## 1. Agent 协作规则

### `AGENTS.md`

用途：给 LLM/coding agent 的仓库级说明文件。

它告诉 agent：

- 项目是 C++20 图数据库 NeuG。
- 推荐构建和测试命令。
- 主要目录结构。
- 查询 pipeline：`Cypher -> ANTLR Parser -> Binder -> Logical Plan -> gopt Converter -> Physical Plan -> Execution`。
- C++/Python 代码风格入口。

怎么用：

- Codex、Claude Code、Cursor 等 agent 通常会自动读取。
- 人也可以把它当作“让 AI 正确进入仓库”的最小上下文。
- 如果新增重要构建命令或架构变化，应同步更新这里。

注意：

- 它是面向 agent 的，不是用户文档。
- 当前内容是高层概览，不包含 CodeGraph 规则；CodeGraph 规则在 `.cursor/rules/codegraph.mdc` 和本会话的 AGENTS 注入内容里。

### `.cursor/rules/codegraph.mdc`

用途：Cursor 规则文件，告诉 AI agent 什么时候应该用 CodeGraph MCP，而不是 grep/read。

核心规则：

- 结构性问题用 CodeGraph：定义位置、调用关系、影响面、签名、flow trace。
- 字符串内容、日志文本、注释内容等 literal query 才用 native grep/read。
- 推荐工具映射：
  - `codegraph_search`: 找 symbol。
  - `codegraph_callers`: 谁调用。
  - `codegraph_callees`: 调了谁。
  - `codegraph_trace`: 从 X 到 Y 的调用路径。
  - `codegraph_impact`: 改某 symbol 的影响。
  - `codegraph_context`: task/area 级上下文。
  - `codegraph_explore`: 一次看多个相关 symbol 源码。
  - `codegraph_status`: 检查索引状态。

怎么用：

- 在 Cursor 中应自动生效，因为 frontmatter 有 `alwaysApply: true`。
- 对当前 Codex 会话，等价规则也被注入到了 `AGENTS.md instructions` 中。

### `.agents/` 与 `.codex/`

当前状态：

- `.agents/` 是空目录。
- `.codex/` 是空目录。

含义：

- 这些目录是给不同 agent 生态预留的输出目录或配置目录。
- 如果运行 `./scripts/init_skills.sh codex`，会把 `.cursor/skills` 复制到 `.codex/skills`。

## 2. AI-Assisted Development Workflow

### `README.md`

AI 相关部分在 `AI-Assisted Workflow` 小节。

它说明 NeuG 使用 AI-assisted Spec-Driven workflow，并给出两个主要入口：

```text
/create-issue
/create-pr
```

怎么用：

- 在支持 slash commands 的 IDE/agent 中输入 `/create-issue ...` 或 `/create-pr ...`。
- 更详细的说明跳转到 `doc/source/development/ai_coding.md`。

### `CONTRIBUTING.md`

AI 相关部分是 `Generative AI Policy`。

核心要求：

- 不希望提交由 AI/LLM 生成的 PR。
- 原因是维护者审核这类 PR 的负担较重。

理解方式：

- 仓库允许 AI-assisted workflow 辅助 issue/PR/spec/task 管理。
- 但贡献代码本身不能简单变成“LLM 生成后直接提交”。开发者仍需负责设计、验证和代码质量。

### `doc/source/development/ai_coding.md`

用途：NeuG 官方 AI-assisted development guide。

它说明：

- 怎么初始化 agent skills。
- 怎么用 slash commands。
- 每个 skill 的用途。
- 支持哪些 coding agent。

快速使用：

```bash
./scripts/init_skills.sh <agent>
```

示例：

```bash
./scripts/init_skills.sh qoder
./scripts/init_skills.sh qwen
./scripts/init_skills.sh codex
./scripts/init_skills.sh --output=.qwen/skills
```

Cursor 是默认 agent，不需要初始化，因为 `.cursor/skills/` 已存在。

文档中列出的主要 slash commands：

```text
/create-issue
/create-pr
/update-with-comments
/speckit.specify
/speckit.plan
/speckit.tasks
/sync-modules
/sync-tasks
/generate_testcase
```

### `scripts/init_skills.sh`

用途：把 `.cursor/skills` 复制到其他 agent 的 skills 目录。

支持快捷方式：

```text
claude    -> .claude/skills
codebuddy -> .codebuddy/skills
codex     -> .codex/skills
gemini    -> .gemini/skills
kilocode  -> .kilocode/skills
opencode  -> .opencode/skills
qoder     -> .qoder/skills
qwen      -> .qwen/skills
roo       -> .roo/skills
windsurf  -> .windsurf/skills
```

常用命令：

```bash
./scripts/init_skills.sh codex
./scripts/init_skills.sh qwen
./scripts/init_skills.sh qoder
./scripts/init_skills.sh --output=.my-agent/skills
```

注意：

- 脚本只做复制，不做格式转换。
- 源目录固定是 `.cursor/skills`。
- 如果目标 agent 的 skill 标准和 Cursor 不完全一致，可能还需要人工调整 frontmatter 或目录格式。

## 3. `.cursor/skills`: Slash Command Skills

所有 skill 都有：

```yaml
disable-model-invocation: true
```

含义：

- 默认不让模型自动触发。
- 需要用户显式通过 slash command 调用。
- 这可以减少误触发。

### `/create-issue`

文件：

```text
.cursor/skills/create-issue/SKILL.md
.cursor/skills/create-issue/templates/bug-issue.md
.cursor/skills/create-issue/templates/feature-issue.md
.cursor/skills/create-issue/scripts/gh-update.sh
```

用途：

- 根据用户输入创建 GitHub issue。
- 自动判断 Bug Report 或 Feature Request。
- 套用模板。
- 让用户 review 临时文件。
- 最后用 `gh issue create` 提交。

用法示例：

```text
/create-issue
Type: Bug
Assignee: @me
Parent Issue: #42

运行 MATCH 查询时 executor segfault，终端日志如下...
```

模板字段：

- bug issue 包含 issue title、assignee、labels、project、parent issue、execution logs、expected behavior、error message。
- feature issue 包含 problem、solution、alternatives、additional context。

注意：

- 依赖 GitHub CLI `gh`。
- 如果 GitHub Projects API 报 classic project deprecated，说明 `gh` 版本可能太旧，skill 提供 `scripts/gh-update.sh` 作为更新辅助脚本。

### `/create-pr`

文件：

```text
.cursor/skills/create-pr/SKILL.md
.cursor/skills/create-pr/templates/pull-request.md
```

用途：

- 从当前分支创建 PR。
- 如果当前在 `main`，要求切到新分支。
- 自动生成 PR title/body。
- 让用户 review。
- 用 `gh pr create` 提交。

用法示例：

```text
/create-pr
Fixes: #42
Reviewers: @reviewer

修复 query execution 中的空指针访问。
```

PR title prefix 候选：

```text
feat, fix, refactor, docs, test, ci
```

注意：

- 仍然需要用户确认最终内容。
- 和 `CONTRIBUTING.md` 的 AI policy 一起看：AI 可以辅助整理 PR，但代码质量和真实性仍由提交者负责。

### `/update-with-comments`

文件：

```text
.cursor/skills/update-with-comments/SKILL.md
```

用途：

- 找到当前分支对应的 PR，或使用用户指定的 PR ID。
- 拉取 conversation comments 和 inline review comments。
- 判断 comment 意图：approve、question、request changes。
- 对 request changes 尝试修改代码。
- 汇总变更，最终 commit/push。

用法：

```text
/update-with-comments
/update-with-comments #123
```

注意：

- 会操作 git branch、stash、commit、push，实际使用前应确保工作区干净或明确知道当前改动。
- 需要 `gh` 权限。

### `/generate-testcase`

文件：

```text
.cursor/skills/generate-testcase/SKILL.md
```

用途：

- 分析当前分支新增 commits。
- 找新增文件和相关 spec。
- 生成相关测试用例。
- 构建并运行测试。
- 可结合 coverage 工具分析覆盖率。

使用场景：

```text
/generate-testcase
为当前分支新增功能生成测试
```

注意：

- 文档中提到 `fastcov`、`lcov`、`ENABLE_GCOV`、`BUILD_TEST`。
- 这个 skill 更偏“实现完成后的测试补齐”。

## 4. Spec-Driven Workflow Skills

这组 skill 用于较大功能的分阶段设计：

```text
/speckit.specify -> spec.md
/speckit.plan    -> plan.md
/speckit.tasks   -> tasks/*
/sync-modules    -> GitHub module issues
/sync-tasks      -> GitHub task issues
```

### `/speckit.specify`

文件：

```text
.cursor/skills/speckit.specify/SKILL.md
.cursor/skills/speckit.specify/templates/spec-template.md
.specify/templates/spec-template.md
```

用途：

- 把自然语言需求变成 feature specification。
- 创建 `specs/<NNN-short-name>/spec.md`。
- 创建 feature branch 和 GitHub feature issue。
- 限制最多 3 个 `[NEEDS CLARIFICATION]`。

用法：

```text
/speckit.specify
Add graph algorithm extension support for NeuG.
Users should be able to register custom algorithms and execute them via Cypher.
```

输出：

```text
specs/001-graph-algo-extension/spec.md
```

spec 模板结构：

- Functional Modules
- Module priority
- Key Components
- Functional Requirements
- Acceptance Scenarios
- Test Strategy
- Edge Cases
- Success Criteria

### `/speckit.plan`

文件：

```text
.cursor/skills/speckit.plan/SKILL.md
.cursor/skills/speckit.plan/templates/plan-template.md
.specify/templates/plan-template.md
```

用途：

- 从 `spec.md` 生成技术 plan。
- 规划项目结构、数据模型、算法模型。
- 输出 `specs/<feature>/plan.md`。

用法：

```text
/speckit.plan 001-graph-algo-extension
```

plan 模板结构：

- Summary
- Technical Context
- Project Structure
- Data Model
- Algorithm Model

注意：

- 它要求 feature directory 和 `spec.md` 已存在。
- 设计阶段要求少写实现代码，重点写结构、数据模型和算法说明。

### `/speckit.tasks`

文件：

```text
.cursor/skills/speckit.tasks/SKILL.md
.cursor/skills/speckit.tasks/templates/tasks-metadata-template.md
.cursor/skills/speckit.tasks/templates/tasks-module-template.md
.specify/templates/tasks-template.md
```

用途：

- 从 `spec.md` 和 `plan.md` 生成可执行任务。
- 输出：

```text
specs/<feature>/tasks/metadata.md
specs/<feature>/tasks/module_1.md
specs/<feature>/tasks/module_2.md
...
```

任务 ID 规则：

```text
Feature 001 -> F001
Module 1 tasks -> T101, T102...
Module 2 tasks -> T201, T202...
```

用法：

```text
/speckit.tasks 001-graph-algo-extension
```

### `/sync-modules`

文件：

```text
.cursor/skills/sync-modules/SKILL.md
```

用途：

- 把 `tasks/module_i.md` 同步成 GitHub module issues。
- 把 module issue 作为 feature issue 的 sub-issue。

用法：

```text
/sync-modules 001-graph-algo-extension
```

依赖：

- `gh` CLI。
- feature issue 存在。
- `tasks/metadata.md` 和 `module_i.md` 存在。

### `/sync-tasks`

文件：

```text
.cursor/skills/sync-tasks/SKILL.md
```

用途：

- 把指定 task 同步成 GitHub task issue。
- task issue 作为 module issue 的 sub-issue。

用法：

```text
/sync-tasks F001-T101
```

注意：

- 用户必须明确提供 Task ID。
- 如果 module issue 不存在，需要先 `/sync-modules`。

## 5. `.specify`: Spec-Kit 支撑文件

### `.specify/templates/*`

文件：

```text
.specify/templates/agent-file-template.md
.specify/templates/spec-template.md
.specify/templates/plan-template.md
.specify/templates/tasks-template.md
```

用途：

- 提供 Spec-Driven workflow 的底层模板。
- `.cursor/skills/speckit.*` 里也带有类似模板；`.specify/templates` 更像通用 Spec-Kit 脚本使用的模板源。

`agent-file-template.md` 用于生成或更新不同 agent 的上下文文件，例如 `AGENTS.md`, `CLAUDE.md`, `QWEN.md` 等。

### `.specify/scripts/bash/*`

文件：

```text
.specify/scripts/bash/common.sh
.specify/scripts/bash/create-new-feature.sh
.specify/scripts/bash/setup-plan.sh
.specify/scripts/bash/check-prerequisites.sh
.specify/scripts/bash/update-agent-context.sh
```

用途：

- `common.sh`: 获取 repo root、当前 branch、feature dir 等公共函数。
- `create-new-feature.sh`: 根据 feature description 创建编号分支和 `specs/<feature>` 目录。
- `setup-plan.sh`: 复制 plan template 到当前 feature 目录。
- `check-prerequisites.sh`: 检查 feature/spec/plan/tasks 是否存在，支持 JSON 输出。
- `update-agent-context.sh`: 从 `plan.md` 提取技术栈、项目结构、命令等信息，更新 agent context 文件。

典型命令：

```bash
.specify/scripts/bash/create-new-feature.sh --short-name graph-algo "Add graph algorithm extension support"
.specify/scripts/bash/setup-plan.sh --json
.specify/scripts/bash/check-prerequisites.sh --json
.specify/scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
.specify/scripts/bash/update-agent-context.sh codex
```

注意：

- 这些脚本会读当前 git branch；feature branch 应形如 `001-feature-name`。
- `update-agent-context.sh` 支持很多 agent 类型：`claude`, `gemini`, `copilot`, `cursor-agent`, `qwen`, `opencode`, `codex`, `windsurf`, `kilocode`, `roo`, `qoder` 等。

### `.specify/memory/constitution.md`

用途：

- Spec-Kit 的 constitution 模板。
- 当前仍是占位内容，未填写 NeuG 真实原则。

使用建议：

- 如果 NeuG 真要长期使用 Spec-Driven workflow，应把这里补成真实项目原则，比如测试策略、兼容性要求、性能约束、C++ style gate、storage correctness gate 等。

### `specs/**`

当前已有三个 feature/spec 示例：

```text
specs/001-ap-temp-graph/
specs/002-elle-testing(#1357)/
specs/003-elle-insert(#1416)/
```

用途：

- 这是 Spec-Driven workflow 的产物目录。
- 每个 feature 下有 `spec.md`, `plan.md`, `tasks.md` 或 `tasks/`。

怎么读：

- 先读 `spec.md` 理解需求。
- 再读 `plan.md` 理解技术方案。
- 最后读 `tasks/metadata.md` 和 `tasks/module_i.md` 看可执行任务拆分。

## 6. CodeGraph / CodeScope 文件

### `skills/codegraph/SKILL.md`

用途：CodeGraph skill 主说明文件。

它定义 CodeGraph 的能力：

- call graph / callers / callees。
- dependency 和 impact analysis。
- dead code。
- hotspots。
- module coupling。
- architecture report。
- semantic search。
- GitHub issue bug root cause analysis。
- PR review / risk scoring / conflict detection / labeling。

安装：

```bash
pip install codegraph-ai
```

创建索引：

```bash
codegraph init --repo . --lang auto --commits 500
```

检查索引：

```bash
codegraph status --db .codegraph
```

生成报告：

```bash
codegraph analyze --db .codegraph --output report.md
```

Python API 示例：

```python
from codegraph.core import CodeScope

cs = CodeScope(".codegraph")
rows = list(cs.conn.execute("""
    MATCH (caller:Function)-[:CALLS]->(f:Function {name: "free_irq"})
    RETURN caller.name, caller.file_path
    LIMIT 10
"""))
cs.close()
```

### `skills/codegraph/schema.md`

用途：CodeScope 图数据库 schema 说明。

主要 node：

```text
File
Function
Class
Module
Commit
Metadata
PR
AUTHOR
```

主要 edge：

```text
CALLS
DEFINES_FUNC
DEFINES_CLASS
HAS_METHOD
IMPORTS
BELONGS_TO
INHERITS
COMPOSES
AGGREGATES
USES
MODIFIES
TOUCHES
CHANGES
OPENS
```

什么时候用：

- 想自己写 Cypher 查询时先看这个文件。
- 想知道 `Function` 上有哪些属性，例如 `name`, `qualified_name`, `file_path`, `start_line`, `end_line`。

### `skills/codegraph/patterns.md`

用途：常用 Cypher 查询模板。

覆盖：

- 定位函数。
- 查 caller/callee。
- transitive callers/callees。
- module membership。
- cross-module calls。
- fan-in/fan-out。
- evolution queries。
- class/method/inheritance/composition/aggregation。

使用方式：

```python
rows = list(cs.conn.execute("""
MATCH (caller:Function)-[:CALLS]->(f:Function {name: 'func_name'})
WHERE f.is_historical = 0
RETURN caller.name, caller.file_path
LIMIT 30
"""))
```

### `skills/codegraph/bug-analysis.md`

用途：把 GitHub issue / bug report 映射到可能的代码根因。

常用 API：

```python
result = cs.analyze_issue("owner", "repo", 1234)
print(result.format_report())

results = cs.analyze_top_bugs("owner", "repo", k=10, label="bug")
```

也支持手动 bug 描述：

```python
from codegraph.bug_locator import find_semantic_matches, trace_callers
matches = find_semantic_matches(cs, "gateway crashes when processing messages", topk=10)
```

使用前提：

- `.codegraph` 索引存在。
- 通常需要 GitHub issue 可访问。
- 可能需要 `gh auth login` 或 GitHub token。

### `skills/codegraph/pr-analysis.md`

用途：分析 open PR 的结构影响、风险、冲突和可合并性。

典型 CLI：

```bash
codegraph pr-review prepare --db .codegraph --output /tmp/pr_review
codegraph pr-review label --db .codegraph
codegraph pr-review label --db .codegraph --dry-run
```

典型 Python API：

```python
from codegraph.pr_api import PRReview

with PRReview(db=".codegraph") as pr:
    pr.prepare()
    pr.label(dry_run=True)
```

输出报告通常分：

1. auto-merge candidates。
2. independent review。
3. conflicting PR groups。

### `skills/codegraph/evals/evals.json`

用途：CodeGraph skill 的 eval examples。

它包含几个测试 prompt，例如：

- 给定 GitHub issue，应该调用 `cs.analyze_issue(...)`。
- 对 top bug issues 做聚合热点分析。

这不是运行时必需文件，更像 skill 质量评估样例。

### `doc/source/tutorials/codegraph-openclaw-example.md`

用途：CodeGraph 教程，以 OpenClaw 代码库为例。

内容包括：

- 安装 `codegraph-ai`。
- 设置 `CODESCOPE_DB_DIR`。
- 创建索引。
- 查询 call chain。
- 生成 architecture report。
- Python API 示例。

使用方式：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install codegraph-ai
export CODESCOPE_DB_DIR="/path/to/project/.codegraph"
codegraph init --repo /path/to/project --lang auto --commits 100
codegraph status --db $CODESCOPE_DB_DIR
codegraph query "Who calls runHeartbeatOnce?" --db $CODESCOPE_DB_DIR
```

### `.codegraph/**`

当前本地存在：

```text
.codegraph/.gitignore
.codegraph/codegraph.db
.codegraph/codegraph.db-shm
.codegraph/codegraph.db-wal
```

用途：

- CodeGraph/CodeScope 的本地索引数据库。
- `codegraph.db` 是主体数据库。
- `db-shm` 和 `db-wal` 是 SQLite WAL 相关文件。

注意：

- 这是生成产物，不应当作为源码笔记手动编辑。
- 之前学习记录里曾记录过旧环境下 `.codegraph` schema 为空的问题；当前本会话使用的是 MCP 提供的 CodeGraph 工具，已经能查询 NeuG C++ 符号。
- 如需用 CLI 重新检查，可运行：

```bash
codegraph status --db .codegraph
```

## 7. 文档翻译里的 AI

### `doc/README.md`

AI 相关部分是 `qwen-translator`。

用途：

- 把 `doc/source/` 同步到网站仓库。
- 使用 Qwen3 AI model 做多语言翻译。
- 自动生成 Nextra `_meta.ts`。

使用文档中的命令：

```bash
git clone https://github.com/graphscope/neug-web
cd neug-web
qwen-translator sync
```

配置位置：

```text
website/translation.config.json
```

注意：

- 这个工具不在当前仓库源码中，只是在文档中描述。
- 真正使用需要 `neug-web` 仓库和 `qwen-translator` 环境。

## 推荐使用路线

### 如果只是日常贡献

```text
1. 读 AGENTS.md，了解 agent 对仓库的基础上下文。
2. 用 /create-issue 创建 bug/feature issue。
3. 小修复完成后用 /create-pr 辅助整理 PR。
4. PR review 后用 /update-with-comments 辅助处理评论。
```

### 如果是较大功能

```text
1. /speckit.specify 生成 specs/<feature>/spec.md
2. 人工 review spec
3. /speckit.plan 生成 plan.md
4. 人工 review plan
5. /speckit.tasks 生成 tasks/
6. /sync-modules 同步模块 issue
7. /sync-tasks Fxxx-Txxx 同步具体任务 issue
8. 按任务开发、测试、提交 PR
```

### 如果是源码理解或架构分析

```text
1. 优先使用 CodeGraph MCP 或 codegraph CLI。
2. 查 symbol/caller/callee/impact 用结构查询。
3. 查具体文本或日志再用 rg。
4. 需要跨仓库报告时用 codegraph analyze。
5. 需要 bug/PR 分析时用 skills/codegraph/bug-analysis.md 或 pr-analysis.md。
```

### 如果要给其他 agent 安装 skills

```bash
./scripts/init_skills.sh codex
./scripts/init_skills.sh qwen
./scripts/init_skills.sh qoder
```

如果目标 agent 不在 shortcut 里：

```bash
./scripts/init_skills.sh --output=.custom-agent/skills
```

## 易错点

- 文件名包含 `ai` 不等于 AI 相关；例如 `main`, `container`, `metadata` 都是误报。
- `.cursor/skills` 是源 skills；`.codex/skills`, `.qwen/skills` 等通常由 `init_skills.sh` 复制生成。
- `disable-model-invocation: true` 表示这些 skills 主要通过 slash command 显式触发。
- `create-issue`, `create-pr`, `sync-modules`, `sync-tasks`, `update-with-comments` 都依赖 GitHub CLI 和认证。
- `CONTRIBUTING.md` 不鼓励直接提交 AI 生成 PR；AI-assisted workflow 更适合辅助组织材料、计划、检查和总结。
- `.codegraph` 是生成索引，不是人工维护文档。
- CodeGraph CLI 和本会话里的 CodeGraph MCP 不是同一层接口：CLI 是项目工具，MCP 是 agent 可直接调用的结构化工具。

## 后续问题

- 当前 `.specify/memory/constitution.md` 还是模板，是否需要为 NeuG 填写真实 development constitution？
- `.cursor/skills` 复制到 Codex/Qwen/Qoder 后是否完全兼容各 agent 的 skill 标准，需要实际试运行确认。
- `create-issue`、`create-pr` 是否应和 `CONTRIBUTING.md` 的 Generative AI policy 做更明确的边界说明？
- 当前 `.codegraph` CLI index 的健康状态可以单独检查一次，和 MCP CodeGraph 的可用状态做对照。
