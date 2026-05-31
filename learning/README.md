# NeuG Learning Notes

这个目录用于沉淀学习 NeuG 时产生的问题、理解、纠偏和结论。它不是 NeuG 官方文档的一部分，而是个人学习知识库，目标是让每次提问都能逐步积累成可检索、可复习、可上传到 GitHub 的 Markdown 材料。

## 目录

- [AGENTS.md](AGENTS.md): 给 assistant/agent 读取的 NeuG 学习协作流程。
- [workflow.md](workflow.md): 每次学习对话如何记录、整理和提交。
- [learning-log.md](learning-log.md): 按时间顺序记录每次学习问题和产出。
- [questions.md](questions.md): 待解决问题池。
- [dialogues/](dialogues/README.md): 已发生对话的结构化整理。
- [topics/project-map.md](topics/project-map.md): NeuG 第一版项目地图。
- [topics/cpp-db-lifecycle-call-flow.md](topics/cpp-db-lifecycle-call-flow.md): C++ 层数据库创建与 query 执行主调用链。
- [topics/db-initialization-memory-layout.md](topics/db-initialization-memory-layout.md): 从 `Database` / `NeugDB::Open` 开始的对象所有权与内存布局。
- [topics/graph-vertex-table-initialization.md](topics/graph-vertex-table-initialization.md): `NeugDB::graph_`、`PropertyGraph` 与 `VertexTable` 的默认初始化方式。
- [topics/data-type-system.md](topics/data-type-system.md): `DataTypeId`、`DataType`、`ExtraTypeInfoType` 的关系。
- [topics/indexer.md](topics/indexer.md): `LFIndexer`、`IdIndexer` 与 NeuG 中 OID/label 到内部 id 的映射。
- [topics/table-column-storage.md](topics/table-column-storage.md): `Table`、`ColumnBase`、定长列和字符串列的列式存储布局。
- [topics/csr-storage.md](topics/csr-storage.md): CSR 类族、邻接项、`GenericView`、`EdgeTable` 双向边存储与底层 container。
- [topics/wal-parser-factory.md](topics/wal-parser-factory.md): `WalParserFactory`、`LocalWalParser` 与 WAL recovery。
- [topics/neug-kuzu-comparison.md](topics/neug-kuzu-comparison.md): NeuG 与 Kuzu 的相同部分、差异边界和阅读路线。
- [topics/source-reading-path.md](topics/source-reading-path.md): 按原创度和重要性排序的 NeuG 源码阅读路径。
- [topics/codegraph-skill-intro.md](topics/codegraph-skill-intro.md): CodeGraph/CodeScope skill 的能力、schema、工作流和 NeuG 学习用途。
- [topics/ai-related-files.md](topics/ai-related-files.md): 根目录下 AI/agent/spec-kit/CodeGraph 相关文件与使用方式总览。
- [templates/question-note.md](templates/question-note.md): 单个问题笔记模板。
- [github-upload.md](github-upload.md): 上传到 GitHub 的推荐流程。

## 如何阅读

1. 想看学习时间线，先读 `learning-log.md`。
2. 想复习一个主题，读 `topics/*.md`。
3. 想找还没解决的问题，读 `questions.md`。
4. 想回顾某次对话，读 `dialogues/<date>/`。
5. 想了解 assistant 应如何维护这些笔记，读 `AGENTS.md`。
