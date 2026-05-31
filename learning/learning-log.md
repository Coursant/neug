# Learning Log

按时间顺序记录 NeuG 学习问题、处理结果和产出文件。详细内容进入 `topics/` 或 `dialogues/`。

## 2026-05-22 - 建立 NeuG 学习笔记体系

- 问题：希望把 NeuG 学习过程中的问题、理解和回答沉淀成 Markdown，并上传 GitHub。
- 结果：创建 `learning/` 目录和基础笔记结构。
- 产出：`README.md`, `workflow.md`, `learning-log.md`, `questions.md`, `topics/`, `dialogues/`, `templates/`。

## 2026-05-22 - 整理既有对话为 Markdown

- 问题：把早期关于学习体系、目录设计和上传方式的对话整理出来。
- 结果：创建 `dialogues/2026-05-22/`。
- 产出：`dialogues/2026-05-22/*.md`。

## 2026-05-23 - 整理 NeuG 与 Kuzu 对比

- 问题：提取之前关于 NeuG 与 Kuzu 相同部分的对比。
- 结果：确认 NeuG compiler/common 中存在大量 Kuzu-derived code；原创重点在 storage、execution、transaction、server、gopt 接入。
- 产出：`topics/neug-kuzu-comparison.md`。

## 2026-05-23 - 更新默认记录规则

- 问题：用户要求以后每个回答都要有条理地放入 `learning/`。
- 结果：默认记录 NeuG 学习/源码阅读问题；提交/upload 仍需明确要求。
- 产出：`workflow.md`。

## 2026-05-23 - 补录源码阅读路径

- 问题：用户追问“刚才关于源码阅读路径的问题呢？”
- 结果：按原创度和重要程度给出阅读路线。
- 产出：`topics/source-reading-path.md`。

## 2026-05-23 - 分析 CodeGraph Skill

- 问题：介绍 skills codegraph。
- 结果：总结其 schema、call graph、semantic search、bug/PR analysis 能力。
- 产出：`topics/codegraph-skill-intro.md`。

## 2026-05-23 - 尝试实际使用 CodeGraph

- 问题：使用 codegraph 分析 NeuG。
- 结果：当前 `.codegraph/graph.db` schema 为空；embedding model 未缓存；toolchain 对 NeuG `.cc/.cpp` 支持不完整。
- 产出：`topics/codegraph-skill-intro.md` 的环境状态记录。

## 2026-05-23 - 调整 VSCode 文件打开行为

- 问题：新文件打开时不要替换当前文件。
- 结果：新增 workspace setting，关闭 preview tab。
- 产出：`.vscode/settings.json`。

## 2026-05-23 - 安装 andrej-karpathy-skills

- 问题：安装 `andrej-karpathy-skills`。
- 结果：安装到 `/home/lcc/.codex/skills/andrej-karpathy-skills`；实际 skill name 是 `karpathy-guidelines`。
- 备注：需要 restart Codex 才能自动加载新 skill。

## 2026-05-23 - 设置默认使用 Karpathy Guidelines

- 问题：如何写代码时默认使用刚安装的 skill。
- 结果：全局偏好中记录 coding/review/refactor/bug fix 默认采用 `karpathy-guidelines` 原则。
- 备注：显式触发名是 `$karpathy-guidelines`。

## 2026-05-23 - 迁移项目特定偏好到 learning README

- 问题：`user-preferences.md` 不应包含具体 NeuG/C++ 项目规则。
- 结果：全局 memory 只保留通用偏好；项目规则迁入仓库内 learning 文档。
- 产出：`README.md`。

## 2026-05-23 - 拆分 README 与 AGENTS 工作流程

- 问题：README 给人读，agent 工作流程应单独给助手读。
- 结果：新增 `AGENTS.md`，精简 `README.md`。
- 产出：`README.md`, `AGENTS.md`。

## 2026-05-23 - 精简 learning 辅助文件

- 问题：用户要求其他文件如 `workflow.md` 也进行精简。
- 结果：压缩流程、上传、问题池、模板、对话摘要、topic notes 和 learning log。
- 产出：`workflow.md`, `github-upload.md`, `questions.md`, `templates/question-note.md`, `dialogues/`, `topics/`, `learning-log.md`。

## 2026-05-24 - 梳理 NeuG DataType 类型系统

- 问题：`ExtraTypeInfoType`、`DataTypeId`、`DataType` 的逻辑关系。
- 结果：确认 `DataTypeId` 是主类型标签，`DataType` 是完整类型对象，`ExtraTypeInfoType` 是附加类型信息的类别标签；同时记录 compiler 侧 `LogicalTypeID` / `LogicalType` 的相似设计。
- 产出：`topics/data-type-system.md`。

## 2026-05-24 - C++ 层数据库创建与执行调用链

- 问题：删除前一篇泛泛回答，改为从 C++ 源码层面整理 NeuG 数据库创建与 query 执行的完整主调用链。
- 结果：删除 `topics/execution-flow-tracing.md`；新增 `cpp-db-lifecycle-call-flow.md`，按 `NeugDB::Open`、`PropertyGraph::Open`、`Connection::Query`、`QueryProcessor::execute`、`GlobalQueryCache::Get`、`GOptPlanner::compilePlan`、`PlanParser`、`Pipeline::Execute` 串起源码阅读路线。
- 产出：`topics/cpp-db-lifecycle-call-flow.md`。

## 2026-05-25 - 理解 WalParserFactory

- 问题：`WalParserFactory` 是什么。
- 结果：确认它是 WAL parser 工厂和注册中心；默认 scheme 是 `"file"`，会创建 `LocalWalParser`；`LocalWalParser` mmap WAL files 并按 `WalHeader::type` 拆成 insert/update WAL，供 `NeugDB::ingestWals()` recovery。
- 产出：`topics/wal-parser-factory.md`。

## 2026-05-25 - 解析 initPlannerAndQueryProcessor

- 问题：解析 `NeugDB::initPlannerAndQueryProcessor()`。
- 结果：补充说明该函数把已打开并恢复好的 `graph_` 接到 `GOptPlanner`、`GlobalQueryCache`、`QueryProcessor` 和 `ConnectionManager`，使 C++ `Connection::Query()` 可以编译/缓存/执行 query。
- 产出：`topics/cpp-db-lifecycle-call-flow.md`。

## 2026-05-25 - DB 初始化后的内存布局

- 问题：从 DB 初始化开始，NeuG 的内存布局是什么样。
- 结果：整理 `Database -> PyDatabase -> NeugDB -> PropertyGraph -> VertexTable/EdgeTable -> Table/CSR/TypedColumn` 的对象所有权、初始化顺序和内存组织；补充 allocator、planner/cache/query_processor/connection_manager 的引用关系。
- 产出：`topics/db-initialization-memory-layout.md`。

## 2026-05-25 - graph_ 与 VertexTable 初始化

- 问题：`NeugDB::graph_`、`PropertyGraph`、`VertexTable` 的初始化方式，以及默认初始化如何进行。
- 结果：确认 `graph_` 是 `NeugDB` 的值成员，默认构造为空 `PropertyGraph`；`VertexTable` 不随 `NeugDB` 默认创建，而是由 schema 驱动，在打开已有库或 `CreateVertexType` 时创建；默认 memory level 是 `kInMemory`，点表默认预留 4096 容量。
- 产出：`topics/graph-vertex-table-initialization.md`。

## 2026-05-26 - NeuG indexer 详解

- 问题：`indexer` 详解，并写成笔记放入 `learning`。
- 结果：整理 `IndexerType = LFIndexer<vid_t>`、`IndexerBuilderType = IdIndexer<KEY_T, vid_t>`、`LFIndexer` 的 key/slot/lid 数据结构、插入/查询/rehash/持久化流程，以及 `VertexTable` 和 `Schema` 中的使用边界。
- 产出：`topics/indexer.md`。

## 2026-05-26 - Table 列存储

- 问题：`Table` 的列存储是怎么样的。
- 结果：确认 `Table` 是列式组织，`columns_` 中每个 `ColumnBase` 管一列；定长列用单个 `IDataContainer` 作为 `T[]`，字符串列用 `items_buffer_ + data_buffer_ + pos_`，顶点主键列则来自 `LFIndexer::keys_` 而不是普通属性 `Table`。
- 产出：`topics/table-column-storage.md`。

## 2026-05-26 - CSR 机制和底层存储

- 问题：详细探索 CSR 机制和底层存储。
- 结果：整理 `CsrBase` 类族、`ImmutableNbr`/`MutableNbr`、multiple/single CSR 布局、`GenericView` 统一遍历、`EdgeTable` 的 outgoing/incoming 双 CSR、bundled/unbundled edge data，以及 `IDataContainer`/mmap 文件布局。
- 产出：`topics/csr-storage.md`。

## 2026-05-27 - AI 相关文件总览

- 问题：详细分析主文件夹下全部和 AI 有关的文件，以及如何使用，并写入笔记。
- 结果：筛除 `main`/`container`/`metadata` 等误报，整理 `AGENTS.md`、`.cursor/rules/codegraph.mdc`、`.cursor/skills/**`、`.specify/**`、`skills/codegraph/**`、`doc/source/development/ai_coding.md`、`scripts/init_skills.sh`、`.codegraph/**`、`doc/README.md` 中的 AI workflow/CodeGraph/Qwen translator 相关内容和使用方式。
- 产出：`topics/ai-related-files.md`。

## 2026-05-28 - 解读 NeuG CMake 构建系统

- 问题：解读这个项目的 CMakeLists，做成笔记。
- 结果：梳理根 `CMakeLists.txt`、`src` object library 聚合、protobuf 生成、Python binding、HTTP server、tests、extensions、install/export 和常用构建命令。
- 产出：`topics/cmake-build-system.md`。

## 2026-05-29 - Schema Deserialize 和 OutArchive

- 问题：`Schema::Deserialize` 和 `OutArchive` 都是什么工作原理。
- 结果：确认 storage `Schema` 的二进制持久化路径是 `PropertyGraph::DumpSchema()` -> `Schema::Serialize()` 写 `${work_dir}/schema`，`PropertyGraph::Open()` -> `loadSchema()` -> `Schema::Deserialize()` 读回；`InArchive` 负责把对象写入 byte buffer，`OutArchive` 负责从 byte buffer 顺序读出对象。
- 产出：`topics/schema-deserialize-outarchive.md`。
