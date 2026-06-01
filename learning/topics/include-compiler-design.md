# include/neug/compiler Design Notes

## 原始问题

```text
详解include compiler 的设计思路，做成笔记
```

## 我的理解

- 这里的 `include compiler` 指 `include/neug/compiler/` 这组公开头文件。
- 目标不是逐个解释所有 400+ 个 header，而是理解它们为什么这样分层，以及 Cypher query 如何从字符串逐步变成 logical/physical plan。

## 代码事实

- `include/neug/compiler/parser/parser.h`: `Parser::parseQuery(std::string_view)` 是 parser 对外入口，返回 `std::vector<std::shared_ptr<Statement>>`。
- `src/compiler/parser/parser.cpp`: `Parser::parseQuery()` 先 trim query，再用 ANTLR `CypherLexer`、`KuzuCypherParser` 解析，最后交给 `Transformer` 把 ANTLR parse tree 转成 NeuG 自己的 parser AST。
- `include/neug/compiler/parser/transformer.h`: `Transformer` 负责把语法节点转换为 `Statement`、query part、reading/updating clause、projection、graph pattern、expression、DDL、COPY 等 parser 层对象。
- `include/neug/compiler/parser/statement.h`: `parser::Statement` 只记录 `StatementType`、parsing time、internal flag，并提供 `requireTransaction()`；它还不是语义绑定后的对象。
- `src/compiler/main/client_context.cpp`: `ClientContext::prepare()` 的核心链路是 `parseQuery()` -> `prepareNoLock()`；`prepareNoLock()` 内部执行 read/write 分析、`Binder::bind()`、`Planner::getBestPlan()`、`Optimizer::optimize()`，最终写入 `PreparedStatement::logicalPlan`。
- `include/neug/compiler/binder/binder.h`: `Binder` 是 semantic binding 的总入口，覆盖 DDL、COPY、query、standalone call、transaction、extension、reading clause、updating clause、projection、graph pattern、table entries 和 validation。
- `src/compiler/binder/binder.cpp`: `Binder::bind()` 按 `StatementType` dispatch 到不同 `bind*` 函数，然后统一调用 `BoundStatementRewriter::rewrite()`。
- `include/neug/compiler/binder/bound_statement.h`: `BoundStatement` 是 bound IR 的统一基类，保存 `StatementType` 和 `BoundStatementResult`。
- `include/neug/compiler/binder/expression/expression.h`: `binder::Expression` 是 bound expression 基类，保存 `ExpressionType`、`LogicalType`、`uniqueName`、alias 和 children；hash/equality 基于 `uniqueName`。
- `include/neug/compiler/binder/expression_binder.h`: `ExpressionBinder` 专门负责 expression binding、函数绑定、参数绑定、literal/variable/property/subquery/case、implicit/force cast 和 constant folding。
- `include/neug/compiler/binder/binder_scope.h`: `BinderScope` 维护当前 scope 中的表达式、大小写不敏感的 name -> expression 映射、节点 table entry 记忆和 node replacement。
- `include/neug/compiler/binder/query/query_graph.h`: `QueryGraph` 表示 `MATCH` 中一个连通 graph pattern；`QueryGraphCollection` 表示多个连通分量；`SubqueryGraph` 用 bitset 支持 join-order 枚举。
- `include/neug/compiler/binder/query/bound_regular_query.h`: `BoundRegularQuery` 保存 normalized single queries、union flags、pre/post query parts 和 pre-query expressions。
- `include/neug/compiler/binder/query/normalized_single_query.h`: `NormalizedSingleQuery` 是 query part 的顺序容器，并保存 statement result。
- `include/neug/compiler/binder/query/normalized_query_part.h`: `NormalizedQueryPart` 把 reading clauses、updating clauses、projection body 和 projection predicate 统一成 planner 更好消费的结构。
- `include/neug/compiler/binder/query/reading_clause/bound_match_clause.h`: `BoundMatchClause` 保存 `QueryGraphCollection`、match type 和 optional join hint。
- `include/neug/compiler/binder/query/return_with_clause/bound_projection_body.h`: `BoundProjectionBody` 保存 projection、group by、aggregate、order by、skip、limit、distinct。
- `include/neug/compiler/catalog/catalog.h`: `Catalog` 是 binder/planner 查 schema 的来源，提供 table、rel group、sequence、type、index、function 等 catalog entry 的查改接口。
- `include/neug/compiler/function/function.h`: function 层定义 `Function`、`FunctionBindData`、`ScalarOrAggregateFunction` 等，供 expression binding 判断签名、返回类型和 bind data。
- `include/neug/compiler/planner/planner.h`: `Planner` 把 `BoundStatement` 转成 `LogicalPlan`；它按 simple statement、COPY、query、read、update、projection、subquery、query graph、scan、extend、join、accumulate、filter/table function 等分组组织接口。
- `include/neug/compiler/planner/operator/logical_operator.h`: `LogicalOperator` 是 logical operator tree 节点基类，包含 operator type、children、factorized schema、cardinality、copy、printing。
- `include/neug/compiler/planner/operator/logical_plan.h`: `LogicalPlan` 只保存最后一个 logical operator root 和 cost；通过 root 反向代表整棵 plan tree。
- `include/neug/compiler/planner/operator/schema.h`: planner schema 用 `FactorizationGroup` 表示 factorized execution 中哪些 expression 同组、是否 flat/single-state，以及当前 scope 中有哪些 expression。
- `include/neug/compiler/planner/graph_planner.h`: `IGraphPlanner` 是面向上层的编译接口，输入 Cypher string，输出 protobuf `physical::PhysicalPlan` 和 result schema string。
- `include/neug/compiler/planner/gopt_planner.h`: `GOptPlanner` 实现 `IGraphPlanner`，内部持有 `MetadataManager` 和 `ClientContext`，通过 `compilePlan()` 编译 query。
- `src/compiler/planner/gopt_planner.cc`: `GOptPlanner::compilePlan()` 调用 `ctx->prepare(query)` 得到 logical plan，然后用 `GAliasManager`、`GPhysicalConvertor` 转成 protobuf physical plan，并用 `GResultSchema::infer()` 生成结果 schema YAML。
- `include/neug/compiler/gopt/g_physical_convertor.h`: `GPhysicalConvertor` 先用 `GPhysicalAnalyzer` 推断 execution flag，再调用 `GQueryConvertor` 把 logical plan 转为 protobuf physical plan。
- `include/neug/compiler/gopt/g_query_converter.h`: `GQueryConvertor` 按 logical operator 类型转换 scan、extend、filter、project、aggregate、order、limit、join、copy、insert、merge、set、delete、union、unwind 等物理 protobuf 节点。
- `include/neug/compiler/gopt/g_expr_converter.h`: `GExprConverter` 把 `binder::Expression` 转为 protobuf expression，并处理 variable/property/literal/parameter/function/cast/case/aggregate 等表达式形态。

## 修正后的结论

`include/neug/compiler/` 的设计核心是多层 IR pipeline，而不是一个单体 compiler。

```text
Cypher string
  -> parser AST
  -> bound semantic IR
  -> logical plan
  -> optimized logical plan
  -> protobuf physical plan
```

更具体地说：

```text
ClientContext::prepare()
  -> Parser::parseQuery()
       -> ANTLR lexer/parser
       -> Transformer
       -> parser::Statement / ParsedExpression / QueryPart / Clause / Pattern
  -> StatementReadWriteAnalyzer
  -> Binder::bind()
       -> BoundStatement
       -> BoundRegularQuery / BoundCopy / BoundDDL / ...
       -> binder::Expression
       -> QueryGraphCollection
       -> NormalizedSingleQuery / NormalizedQueryPart
  -> Planner::getBestPlan()
       -> LogicalPlan
       -> LogicalOperator tree
       -> factorized planner Schema
  -> Optimizer::optimize()
  -> PreparedStatement::logicalPlan

GOptPlanner::compilePlan()
  -> ClientContext::prepare()
  -> GAliasManager
  -> GPhysicalConvertor
       -> GPhysicalAnalyzer
       -> GQueryConvertor
       -> GExprConverter
  -> physical::PhysicalPlan + result schema YAML
```

### 1. parser 层：只做语法结构化

`parser/` 的职责是把文本 query 变成 NeuG 自己的 AST，而不是做 schema 校验。

- `Parser` 是窄入口，隐藏 ANTLR 细节。
- `Transformer` 是语法树适配层，把 ANTLR context 转成项目自己的 `Statement`、`ParsedExpression`、`PatternElement`、`ReadingClause` 等。
- `parser::Statement` 只有 statement type、内部语句标记、解析耗时和 transaction 需求判断。

这个设计的好处是：ANTLR 生成代码只泄漏到 parser 边界内，后续 binder/planner 不需要依赖 ANTLR context。

### 2. binder 层：把语法变成有语义的 IR

`binder/` 是 compiler 的语义中心。它做的事情包括：

- 根据 `StatementType` dispatch 到 DDL、COPY、query、call、transaction、extension 等 binder。
- 访问 `Catalog` 校验 table、label、property、type、function 是否存在。
- 维护 `BinderScope`，解决变量名、alias、WITH/RETURN scope、大小写不敏感 lookup。
- 用 `ExpressionBinder` 给 expression 绑定类型、函数签名、参数、property、cast 和 aggregate。
- 把 graph pattern 绑定成 `QueryGraphCollection`，给 planner 做图查询规划。
- 把 query 规范化为 `NormalizedSingleQuery` / `NormalizedQueryPart`，让 planner 不直接面对原始 Cypher 语法形态。

可以把 parser AST 理解成“用户写了什么”，把 bound IR 理解成“系统确认这句话在当前 schema 下是什么意思”。

### 3. expression 设计：统一、可 hash、可改写

`binder::Expression` 的几个字段很关键：

- `expressionType`: 表达式类别，如 variable、literal、property、function、aggregate、case、subquery。
- `dataType`: binder 后确定的 compiler logical type。
- `uniqueName`: 表达式身份。hash/equality 都基于它。
- `alias`: 用户可见名字。
- `children`: expression tree 子节点。

这使 expression 可以同时服务 binder、planner schema、operator、gopt protobuf 转换。缺点是 `uniqueName` 的正确性很重要；如果重写表达式时名字管理错误，会影响 scope、schema lookup、hash map 和 planner 可求值判断。

### 4. QueryGraph 设计：把 MATCH 从语法问题变成图规划问题

Cypher 的 `MATCH (a)-[e]->(b)` 不适合直接按文本顺序规划。`QueryGraph` 把它抽象成：

- query nodes: `NodeExpression`
- query rels: `RelExpression`
- name -> position map
- connected component

`QueryGraphCollection` 表示一个 `MATCH` 里可能有多个 disconnected components。`SubqueryGraph` 用 node/rel bitset 表示子图，服务 join-order enumeration。这个设计让 planner 可以枚举 scan、extend、intersect、hash join、cross product 等方案，而不是被原始 pattern 顺序锁死。

### 5. planner 层：从 bound IR 到 logical operator tree

`planner/` 的目标不是直接生成最终执行 protobuf，而是生成可优化、可打印、可估算 cardinality 的 `LogicalPlan`。

- `Planner::getBestPlan()` 是主入口。
- `LogicalPlan` 只存 root operator，即 `lastOperator`。
- `LogicalOperator` 子类覆盖 DDL、scan、extend、recursive extend、filter、projection、aggregate、join、update、copy、table function 等。
- `Schema`/`FactorizationGroup` 表示 factorized intermediate result，planner 在 append filter/projection/join/flatten/accumulate 时依赖它判断 expression 是否在 scope、是否需要 flatten。
- `CardinalityEstimator`、`CostModel`、`JoinOrderEnumeratorContext` 支持 query graph 规划和 best plan 选择。

这里体现了 NeuG/Kuzu 风格图数据库 planner 的重点：不是简单 relational row schema，而是 factorized schema + 图模式 join-order。

### 6. gopt 层：把 logical plan 适配成服务端执行协议

`gopt/` 更像 logical plan 到执行 protobuf 的 adapter。

- `GOptPlanner` 是对外 `IGraphPlanner` 实现。
- `GPhysicalAnalyzer` 推断 plan 的 read/insert/update/schema/batch/procedure 等执行 flag。
- `GPhysicalConvertor` 负责整体转换。
- `GQueryConvertor` 负责 operator 级转换。
- `GExprConverter` 负责 expression 级转换。
- `GAliasManager` 维护 logical expression/operator 与 protobuf alias/column id 的对应。
- `GResultSchema` 从 logical plan 和 alias 推断结果 schema YAML。

所以普通 preparation pipeline 和 service-mode physical plan pipeline 是复用关系：先走 `ClientContext::prepare()` 得到 logical plan，再由 gopt 转成 protobuf physical plan。

### 7. catalog/function/common 是 compiler 的支撑层

`catalog/`、`function/`、`common/` 虽然放在 `compiler/` 下，但它们不是某个阶段的 AST，而是多个阶段共享的基础设施。

- `catalog/`: schema metadata source，binder 做 name/type validation，planner/gopt 做 table id、label、property 映射。
- `function/`: scalar/aggregate/table/export/import/cast 等函数定义和 bind data。
- `common/`: compiler 侧 logical type、value、vector、data chunk、serializer、enum、utility。
- `processor/`、`storage/`、`transaction/` 在这个 include tree 下也有接口，主要是为了编译器、函数、catalog 与执行/存储边界交互。

## 设计思路总结

1. 分层 IR：parser AST、bound IR、logical plan、physical protobuf 各自服务不同阶段，降低阶段耦合。
2. 语法和语义分离：parser 不查 catalog，binder 才解析名字、类型、函数和 schema。
3. graph pattern 结构化：`QueryGraphCollection` 让 `MATCH` 进入图规划模型，而不是保留成语法链表。
4. expression 贯穿全链路：同一套 `binder::Expression` 被 binder、planner、schema、gopt 共享，减少重复表达式模型。
5. factorized schema 前置：planner schema 不是普通列列表，而是 `FactorizationGroup`，为图查询中延迟 flatten、accumulate、join 提供基础。
6. public interface 窄入口：`Parser::parseQuery()`、`Binder::bind()`、`Planner::getBestPlan()`、`IGraphPlanner::compilePlan()` 都是清晰阶段入口。
7. service physical plan 是适配层：gopt 不重新 parse/bind/plan，而是复用 logical plan 后转换为 protobuf plan。

## 推荐源码阅读顺序

1. `src/compiler/main/client_context.cpp`: 从 `ClientContext::prepare()` 和 `prepareNoLock()` 看完整 pipeline。
2. `src/compiler/parser/parser.cpp` + `include/neug/compiler/parser/transformer.h`: 看 query text 如何进入 parser AST。
3. `src/compiler/binder/binder.cpp` + `include/neug/compiler/binder/binder.h`: 看 `StatementType` 如何 dispatch。
4. `include/neug/compiler/binder/expression/expression.h` + `include/neug/compiler/binder/expression_binder.h`: 看 expression 语义绑定模型。
5. `include/neug/compiler/binder/query/query_graph.h`: 看 Cypher graph pattern 如何建模。
6. `include/neug/compiler/planner/planner.h` + `include/neug/compiler/planner/operator/logical_operator.h`: 看 bound IR 如何变 logical operator tree。
7. `src/compiler/planner/gopt_planner.cc` + `include/neug/compiler/gopt/g_physical_convertor.h`: 看 logical plan 如何变 protobuf physical plan。

## 未验证假设

- 这次主要基于头文件和关键实现文件静态阅读，没有运行 compiler/planner 测试。
- 没有逐个展开所有 `Transformer::transform*`、`Binder::bind*`、`Planner::plan*` 的实现细节；本笔记关注公开接口体现的设计分层。
- `optimizer/` 的具体 rule 和 rewrite 顺序没有展开，后续可以单独做一篇 logical optimizer 笔记。

## 后续问题

- `Optimizer::optimize()` 具体有哪些规则，和 `Planner` 自带 join-order 枚举如何分工？
- `GQueryConvertor` 生成的 protobuf operator 与 `src/execution/execute/plan_parser.cc` 如何一一对应？
- `binder::Expression::uniqueName` 在复杂 rewrite、alias、subquery 中如何保证稳定和唯一？
