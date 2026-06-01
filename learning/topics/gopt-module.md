# NeuG gopt Module

## 原始问题

```text
解析gpot模块，写成笔记、
```

## 我的理解

- 这里的 `gpot` 应该是笔误，实际指 `gopt` 模块。
- `gopt` 模块主要位于 `include/neug/compiler/gopt/` 和 `src/compiler/gopt/`。
- 这个模块名字里有 `opt`，但从当前源码看，它更像 “logical plan -> protobuf physical plan” 的转换和 GraphScope/physical protocol 适配层，不是 NeuG logical optimizer 的主体。

## 代码事实

- `include/neug/compiler/planner/gopt_planner.h`: `GOptPlanner` 实现 `IGraphPlanner`，对外暴露 `compilePlan()`、`update_meta()`、`update_statistics()`、`analyzeMode()`。
- `src/compiler/planner/gopt_planner.cc`: `GOptPlanner::compilePlan()` 先调用 `ctx->prepare(query)` 生成 `PreparedStatement` 和 `LogicalPlan`，再创建 `GAliasManager`、`GPhysicalConvertor`，最后返回 `physical::PhysicalPlan` 和 `GResultSchema::infer()` 生成的 YAML result schema。
- `src/main/neug_db.cc`: `NeugDB::initPlannerAndQueryProcessor()` 中，当 `config_.planner_kind == "gopt"` 时创建 `GOptPlanner`，并用 `schema().to_yaml()` 和 `graph().get_statistics_json()` 更新 planner metadata。
- `src/compiler/main/metadata_manager.cpp`: `MetadataManager` 默认创建 `catalog::GCatalog`，`updateSchema()` 会转调 `GCatalog::updateSchema()`。
- `include/neug/compiler/gopt/g_catalog.h` / `src/compiler/gopt/g_catalog.cpp`: `GCatalog` 继承 `Catalog`，从 YAML schema 构建 node/rel catalog entry，并注册 built-in functions。
- `src/compiler/gopt/g_catalog.cpp`: `GCatalog::loadSchema()` 读取 `schema.vertex_types` 和 `schema.edge_types`；edge type 的每个 source/destination pair 会创建一个 `GRelTableCatalogEntry`，多 pair 时再创建 `RelGroupCatalogEntry`。
- `include/neug/compiler/gopt/g_rel_table_entry.h`: `GRelTableCatalogEntry` 继承 `RelTableCatalogEntry`，额外保存 `labelId`；`tableId` 可以是某个 source/destination pair 的内部 table id，而 `labelId` 表示用户级 edge label id。
- `include/neug/compiler/gopt/g_node_table.h` 和 `g_rel_table.h`: `GNodeTable`、`GRelTable` 是带统计 row count 的轻量 storage table 适配类，主要服务 planner/stats。
- `include/neug/compiler/gopt/g_alias_manager.h` / `src/compiler/gopt/g_alias_manager.cpp`: `GAliasManager` 遍历 logical operator tree，为 expression unique name 分配 physical alias id；无须输出的匿名 alias 可映射到 `DEFAULT_ALIAS_ID = -1`。
- `include/neug/compiler/gopt/g_alias_name.h`: `GAliasName` 同时保存系统生成的 `uniqueName` 和用户 query 中给出的 `queryName`。
- `include/neug/compiler/gopt/g_physical_analyzer.h`: `GPhysicalAnalyzer` 遍历 logical plan，推断 `ExecutionFlag`，包括 `read`、`insert`、`update`、`schema`、`batch`、`transaction`、`procedure_call`。
- `include/neug/compiler/gopt/g_physical_convertor.h`: `GPhysicalConvertor::convert()` 先通过 `GPhysicalAnalyzer` 生成 execution flag，再调用 `GQueryConvertor` 生成 plan protobuf，并把 flag 写入 `PhysicalPlan`。
- `include/neug/compiler/gopt/g_query_converter.h` / `src/compiler/gopt/g_query_converter.cpp`: `GQueryConvertor` 是 operator 级转换器，负责 scan、extend、recursive extend、getV、filter、project、aggregate、join、copy、insert、merge、set、delete、union、unwind、transaction、extension、DDL 等 logical operator 到 protobuf operator 的转换。
- `src/compiler/gopt/g_query_converter.cpp`: `convertOperator()` 对 `INTERSECT`、`CROSS_PRODUCT`、`HASH_JOIN`、`UNION_ALL` 做特殊递归处理；其他 operator 默认先转换 children，再转换自身。
- `src/compiler/gopt/g_query_converter.cpp`: `GQueryConvertor::convert()` 默认会在 plan 末尾添加 `Sink`；如果是 update 或 DDL 类 clause，`GPhysicalConvertor` 会设置 `skipSink`。
- `include/neug/compiler/gopt/g_expr_converter.h` / `src/compiler/gopt/g_expr_converter.cpp`: `GExprConverter` 是 expression 级转换器，把 `binder::Expression` 转为 protobuf `common::Expression`。
- `src/compiler/gopt/g_expr_converter.cpp`: `GExprConverter::convert()` 会先检查 expression 是否已经在 child schema alias 中出现；如果出现且不是 `PATTERN`，就转成 variable 引用，避免重复计算。
- `src/compiler/gopt/g_expr_converter.cpp`: expression conversion 支持 literal、property、variable、comparison、boolean、pattern、function、case、parameter 等；aggregate 通过 `convertAggFunc()` 转成 `physical::GroupBy_AggFunc`。
- `include/neug/compiler/gopt/g_scalar_type.h`: `GScalarType` 把 binder scalar function name 分类成 add/subtract/cast/date/label/path extract/properties/list/string 等 gopt 可识别的 scalar type。
- `include/neug/compiler/gopt/g_type_converter.h` / `src/compiler/gopt/g_type_converter.cpp`: `GPhysicalTypeConverter` 把 compiler `LogicalType`、`GNodeType`、`GRelType` 转成 protobuf `IrDataType`；`GLogicalTypeConverter` 做部分反向转换。
- `include/neug/compiler/gopt/g_graph_type.h`: `GNodeType` 和 `GRelType` 是 graph label type wrapper，可从 bound node/rel expression 中提取 catalog entries、label ids，并可输出 YAML。
- `include/neug/compiler/gopt/g_type_utils.h`: `GTypeUtils` 在 YAML type 和 compiler `LogicalType` 之间转换。
- `include/neug/compiler/gopt/g_type_registry.h` / `src/compiler/gopt/g_type_registration.cpp`: `LogicalTypeRegistry` 用 YAML dump 字符串注册基础类型到 `LogicalTypeID` 的映射。
- `include/neug/compiler/gopt/g_ddl_converter.h` / `src/compiler/gopt/g_ddl_converter.cpp`: `GDDLConverter` 把 `LogicalCreateTable`、`LogicalDrop`、`LogicalAlter` 转成 DDL physical protobuf。
- `include/neug/compiler/gopt/g_result_schema.h`: `GResultSchema::infer()` 从 logical plan schema 中取 expressions in scope，结合 `GAliasManager` 和 `GTypeUtils` 生成返回列 YAML；COPY/INSERT/MERGE/SET/DELETE/DDL/transaction/extension 等语句不从 expression 推断返回列。
- `tests/compiler/gopt_test.h`: gopt compiler tests 的 fixture 走 `MetadataManager -> ClientContext::prepare -> LogicalPlan -> GAliasManager -> GPhysicalConvertor`，并提供 physical JSON、logical string、result YAML 的验证工具。
- `tests/compiler/flag_test.cpp`: 专门验证 `GPhysicalAnalyzer` 对 read/insert/update/schema/batch/transaction/procedure_call flag 的判断。

## 修正后的结论

`gopt` 模块处在 NeuG query pipeline 的后半段。它不负责 parse/bind/logical planning，而是复用 compiler 前半段生成的 `LogicalPlan`，再把它转换成执行端可消费的 protobuf `physical::PhysicalPlan`。

```text
Cypher query
  -> ClientContext::prepare()
       -> Parser
       -> Binder
       -> Planner
       -> Optimizer
       -> planner::LogicalPlan
  -> GAliasManager
  -> GPhysicalConvertor
       -> GPhysicalAnalyzer
       -> GQueryConvertor
            -> GExprConverter
            -> GPhysicalTypeConverter
            -> GDDLConverter
  -> physical::PhysicalPlan
  -> GResultSchema::infer()
  -> result schema YAML
```

从上层看：

```text
NeugDB::initPlannerAndQueryProcessor()
  -> GOptPlanner()
  -> update_meta(schema YAML)
  -> update_statistics(stats JSON)
  -> GlobalQueryCache(planner)
  -> QueryProcessor(graph, planner, cache, ...)

Connection::Query()
  -> QueryProcessor
  -> GlobalQueryCache
  -> GOptPlanner::compilePlan(query)
  -> physical::PhysicalPlan
```

## 子模块职责

### 1. GOptPlanner：模块入口

`GOptPlanner` 是 gopt 对外的 facade。

- 构造时创建 `MetadataManager` 和 `ClientContext`。
- `update_meta()` 把外部 YAML schema 写入 `MetadataManager`。
- `update_statistics()` 把 graph statistics JSON 写入 `StatsManager`。
- `compilePlan()` 先让 `ClientContext::prepare()` 生成 optimized logical plan，再调用 gopt converter。
- `analyzeMode()` 用 query string token 粗略判断 `AccessMode`，当前只做静态 token 扫描。

这里的关键设计是复用已有 compiler pipeline。gopt 没有重新实现 parser、binder、logical planner。

线程安全注释的含义：

- `compilePlan()` 是读操作：它读取当前 catalog/schema、statistics、function registry，并基于这些 metadata 编译 query。
- `update_meta()` 是写操作：它会重建 `GCatalog` 内部的 `tables` 和 `relGroups`。
- `update_statistics()` 是写操作：它会替换 `MetadataManager` 中的 `StatsManager`。
- `GOptPlanner` 本身没有在这三个 public 方法外层加统一读写锁，所以调用方必须保证并发同步。

典型 race 是：线程 A 正在 `compilePlan("MATCH (p:person) RETURN p")`，binder/planner 正在从 catalog 查 `person` label；线程 B 同时调用 `update_meta()`，`GCatalog::updateSchema()` 把旧的 `tables` 和 `relGroups` 替换掉并重新 `loadSchema()`。这时线程 A 可能读到半更新的 catalog，轻则编译失败，重则使用了已经失效的 catalog entry 指针。

另一个一致性问题是 schema 和 statistics 通常是一组 metadata。若线程 A 在 `update_meta(newSchema)` 和 `update_statistics(newStats)` 中间插进来 `compilePlan()`，它可能用新 schema 配旧 stats 做 cardinality/cost 估计，得到不一致的 plan。

### 2. GCatalog：YAML schema 到 compiler catalog 的适配

`GCatalog` 是 gopt mode 下的 catalog 实现。

它读取的 schema 形态大致是：

```text
schema:
  vertex_types:
    - type_name
      type_id
      primary_keys
      properties
  edge_types:
    - type_name
      type_id
      vertex_type_pair_relations
      properties
```

加载逻辑：

- vertex type -> `NodeTableCatalogEntry`
- edge type 的每个 source/destination pair -> `GRelTableCatalogEntry`
- 一个 edge label 对应多个 source/destination pair 时 -> `RelGroupCatalogEntry`
- node 默认补 `ID`、`LABEL` 内部字段
- rel 默认补 `ID`、`SRC`、`DST`、`LABEL` 内部字段
- properties 通过 `GTypeUtils::createLogicalType()` 转成 compiler `LogicalType`

`GRelTableCatalogEntry` 中 `tableId` 和 `labelId` 分离是一个重要点：执行 protobuf 往往需要用户级 edge label id，但 planner/catalog 内部还要区分不同 source/destination pair 的 rel table。

### 3. GAliasManager：logical expression 到 physical alias id

logical plan 里的 expression 用 `uniqueName` 做身份，但 physical protobuf 使用 numeric alias id。

`GAliasManager` 做三件事：

- 遍历 logical operator tree，收集 operator 输出的 `GAliasName`。
- 给需要在 physical plan 中保留的 unique name 分配 alias id。
- 对不需要显式输出、且没有用户 query alias 的名字使用 `DEFAULT_ALIAS_ID = -1`。

它不是简单给所有 expression 编号。代码里对 scan、extend、recursive extend、getV、unwind、insert、merge、projection、aggregate、distinct 等 operator 都有不同提取逻辑。

一个细节是 `GAliasName` 同时保存：

- `uniqueName`: 系统内部稳定名字，用于查找。
- `queryName`: 用户 query 中给出的 alias，用于 result schema column name。

### 4. GPhysicalAnalyzer：推断执行 flag

`GPhysicalAnalyzer` 不生成 operator，只分析 logical plan 对图和系统状态的影响，输出：

```cpp
struct ExecutionFlag {
  bool read;
  bool insert;
  bool update;
  bool schema;
  bool batch;
  bool create_temp_table;
  bool transaction;
  bool procedure_call;
};
```

核心判断：

- `SCAN_NODE_TABLE`、`EXTEND`、`GET_V`、`RECURSIVE_EXTEND` 通常意味着 read。
- `INSERT` 根据插入 node/rel 以及是否依赖已有 matched nodes，判断 insert 或 update。
- `SET_PROPERTY`、`DELETE`、`MERGE` 是 update。
- `COPY_FROM`、`COPY_TO`、file table function 是 batch。
- `CREATE_TABLE`、`ALTER`、`DROP` 是 schema。
- `TRANSACTION` 是 transaction。
- extension 和非 file table function 是 procedure call。

`tests/compiler/flag_test.cpp` 就是围绕这一层写的。

### 5. GPhysicalConvertor：转换总控

`GPhysicalConvertor` 是轻量总控：

```text
convert(LogicalPlan)
  -> GPhysicalAnalyzer::analyze()
  -> convertExecutionFlag()
  -> decide skipSink
  -> GQueryConvertor::convert()
  -> attach ExecutionFlag to PhysicalPlan
```

它会对 update 和 DDL plan 设置 `skipSink`，因为这类 plan 不需要普通 query sink。

### 6. GQueryConvertor：operator 级转换主力

`GQueryConvertor` 是 gopt 最大的类。它把 logical operator tree 展平成 protobuf physical operator list。

主要转换范围：

- graph read: `LogicalScanNodeTable` -> `physical::Scan`
- traversal: `LogicalExtend` -> `physical::EdgeExpand`
- recursive path: `LogicalRecursiveExtend` -> `physical::PathExpand`
- get vertex: `LogicalGetV` -> `physical::GetV`
- relational operators: filter、project、aggregate、distinct、order、limit、join、cross product、intersect、union
- data IO: table function、copy from、copy to、data source、data export
- DML: insert、merge、set property、delete
- DDL: delegate to `GDDLConverter`
- admin/procedure: checkpoint、extension、procedure call

转换策略：

- 对普通 unary operator，先递归转换 child，再添加当前 protobuf operator。
- 对 `INTERSECT`、`CROSS_PRODUCT`、`HASH_JOIN`、`UNION_ALL`，有特殊转换逻辑，因为它们需要处理多个 child plan 或 sub plan。
- 默认末尾追加 `Sink`，但 update/DDL 跳过。
- 如果最后是 `Sink` 且倒数第二个是 `DataExport`，会交换顺序，保证 export 语义正确。

### 7. GExprConverter：expression 级转换

`GExprConverter` 把 `binder::Expression` 转成 protobuf `common::Expression`。

支持的表达式大类：

- literal -> `common::Value`
- parameter -> `DynamicParam`
- variable -> `Variable`
- property -> property variable/extract
- pattern node/rel -> graph variable
- comparison/boolean/null check -> expression operators
- scalar function -> arithmetic、cast、temporal、label、path extract、list/string function、extension function 等
- aggregate function -> `physical::GroupBy_AggFunc`
- case expression -> protobuf case expression

一个重要逻辑：

```text
如果 expression 的 uniqueName 已经在 child schema aliases 中，
且 expression 不是 PATTERN，
则直接转成 variable 引用，而不是重新展开计算。
```

这说明 gopt conversion 依赖 logical plan schema 来判断某个 expression 是“已产出的列”还是“需要生成计算表达式”。

### 8. GPhysicalTypeConverter / GTypeUtils：类型转换

gopt 需要在三套类型之间转换：

- NeuG compiler `common::LogicalType`
- graph label wrapper: `GNodeType` / `GRelType`
- protobuf `common::IrDataType` / YAML type

`GPhysicalTypeConverter` 负责：

- `NODE` -> protobuf graph vertex type
- `REL` -> protobuf graph edge type
- `RECURSIVE_REL` -> protobuf path type
- `LIST` / `ARRAY` -> protobuf list/array type
- `STRUCT` -> protobuf tuple type
- basic scalar -> protobuf primitive/string/temporal type

`GTypeUtils` 负责 YAML schema 中的 type 和 `LogicalType` 互转。基础类型通过 `LogicalTypeRegistry` 注册，复杂类型如 string、temporal、array 走手写解析。

### 9. GResultSchema：返回 schema 推断

`GResultSchema::infer()` 从 logical plan 的 `Schema::getExpressionsInScope()` 中拿返回表达式，然后：

- 用 `GAliasManager` 找 alias id 和 query alias。
- 优先使用用户 query alias 作为 column name。
- 对 node/rel expression 输出 graph type YAML。
- 对普通 scalar/list/struct 输出 `GTypeUtils::toYAML()`。
- 对 DML、DDL、COPY、transaction、extension 等语句不从 expression 推断返回列。

这解释了为什么 `GOptPlanner::compilePlan()` 返回的是：

```cpp
result<std::pair<physical::PhysicalPlan, std::string>>
```

第二个 `std::string` 就是 result schema YAML。

## 设计思路总结

1. gopt 是适配层，不是 parser/binder/planner 的替代品。
2. logical plan 是输入边界，protobuf physical plan 是输出边界。
3. `GCatalog` 把外部 YAML schema 适配成 compiler catalog，让 binder/planner 能复用已有逻辑。
4. `GAliasManager` 解决 expression unique name 与 physical alias id 的映射，是 expression/operator/result schema 转换的公共依赖。
5. `GPhysicalAnalyzer` 把 plan 的副作用抽成 execution flag，供执行层判断 read/write/schema/batch/procedure 行为。
6. `GQueryConvertor` 负责 operator 粒度，`GExprConverter` 负责 expression 粒度，`GPhysicalTypeConverter` 负责 type 粒度。
7. `GResultSchema` 和 `GQueryConvertor` 共用 alias/type 信息，保证 physical plan 列 id 和对外 result schema 对齐。

## 推荐阅读顺序

1. `src/compiler/planner/gopt_planner.cc`: 先看 `compilePlan()` 的总流程。
2. `src/main/neug_db.cc`: 看 `initPlannerAndQueryProcessor()` 如何接入 gopt。
3. `src/compiler/main/metadata_manager.cpp`: 看默认 catalog 为什么是 `GCatalog`。
4. `src/compiler/gopt/g_catalog.cpp`: 看 YAML schema 如何变成 catalog entries。
5. `src/compiler/gopt/g_alias_manager.cpp`: 看 alias id 如何从 logical operator tree 中推导出来。
6. `include/neug/compiler/gopt/g_physical_analyzer.h`: 看 execution flag 如何判断。
7. `include/neug/compiler/gopt/g_physical_convertor.h`: 看转换总控。
8. `src/compiler/gopt/g_query_converter.cpp`: 看 operator 到 protobuf physical operator 的主转换。
9. `src/compiler/gopt/g_expr_converter.cpp`: 看 expression 到 protobuf expression 的转换。
10. `src/compiler/gopt/g_type_converter.cpp`: 看 LogicalType 到 protobuf type 的转换。
11. `include/neug/compiler/gopt/g_result_schema.h`: 看 result schema YAML 如何推断。
12. `tests/compiler/gopt_test.h` 和 `tests/compiler/flag_test.cpp`: 看测试如何验证 logical/physical/result/flag。

## 未验证假设

- 这次没有运行 `tests/compiler`，结论基于本地源码静态阅读。
- `GQueryConvertor` 的每个 operator conversion 都没有逐行展开；本笔记关注模块边界和主数据流。
- `gopt` 是否完全对应外部 GraphScope physical plan protocol，需要结合 `proto/` 和 `src/execution/execute/plan_parser.cc` 另开一篇确认。

## 后续问题

- `physical::PhysicalPlan` 中每个 protobuf operator 最终如何映射到 `src/execution/execute/ops/` 的执行算子？
- `GAliasManager` 的 `DEFAULT_ALIAS_ID = -1` 对执行层有什么具体语义？
- `GPhysicalAnalyzer` 对 insert vs update 的判断是否覆盖所有 `MERGE`、`CREATE relationship from matched nodes` 的边界情况？
- `GTypeUtils` 的 YAML type grammar 与用户文档中的 schema YAML 是否完全一致？
