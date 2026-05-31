# Questions

这里只记录仍需后续查证的问题；已形成规则的问题不再保留为 open item。

## Open

- CodeGraph 如何在当前 NeuG C++ 项目中稳定生成可查询索引？
- `DataTypeId` 与 compiler 侧 `LogicalTypeID` 在 query pipeline 中如何互相转换？
- `StringTypeInfo::max_length` 在 storage column、Arrow、protobuf、YAML 序列化中是否始终被保留？
- 为什么运行时/存储侧和 compiler 侧存在两套相似类型系统，边界在哪里？
- 对 `CREATE NODE TABLE ...` 这类 schema query，具体会生成哪个 DDL operator？
- 对 `MATCH (n) RETURN n`，实际 physical plan 中 operators 的顺序是什么？
- WAL 目录中多线程 WAL 文件的 timestamp 是否全局单调，如何由 `VersionManager` 保证？
- `LocalWalParser::open()` 读取目录文件时没有显式排序，是否会影响 recovery 正确性？

## Resolved

- 学习笔记语言：个人笔记可中英混搭。
- GitHub 目标：`coursant` remote 的 `learning/neug-notes` 分支。
- 记录粒度：默认写入 `learning-log.md`，可复用内容再进入 `topics/*.md`。
