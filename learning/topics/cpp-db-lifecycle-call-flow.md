# C++ DB Lifecycle Call Flow

## 原始问题

```text
删除这篇回答，我要求从cpp层面的该数据库的创建执行的完整函数调用并愿读源码
```

## 我的理解

- 需要从 C++ 源码层面读 NeuG 的主流程，而不是泛泛介绍如何用工具抓调用栈。
- 这里的“创建执行”拆成两条主线：
  - database creation/open lifecycle：`NeugDB` 如何打开、初始化 storage/planner/query processor。
  - query execution lifecycle：C++ `Connection::Query()` 如何编译/获取 pipeline 并执行 operators。

## 代码事实

- `src/main/neug_db.cc`: `NeugDB::Open(const std::string&, ...)` 构造 `NeugDBConfig` 后调用 `Open(config)`。
- `src/main/neug_db.cc`: `NeugDB::Open(const NeugDBConfig&)` 依次执行 `preprocessConfig()`, `PlanParser::get().init()`, `initAllocators()`, `openGraphAndIngestWals()`, `initPlannerAndQueryProcessor()`。
- `src/main/neug_db.cc`: `openGraphAndIngestWals()` 调用 `graph_.Open(...)`，初始化 WAL parser，并调用 `ingestWals(...)`。
- `src/storages/graph/property_graph.cc`: `PropertyGraph::Open(...)` 加载 schema 或创建 empty graph，然后打开 vertex tables 和 edge tables。
- `src/main/neug_db.cc`: `initPlannerAndQueryProcessor()` 创建 `GOptPlanner`，更新 schema/statistics，创建 `GlobalQueryCache`、`QueryProcessor` 和 `ConnectionManager`。
- `src/main/connection_manager.cc`: `ConnectionManager::CreateConnection()` 创建 `Connection(graph_, query_processor_)`。
- `src/main/connection.cc`: `Connection::Query()` 调用 `query_processor_->execute(...)`。
- `src/main/query_processor.cc`: `QueryProcessor::execute()` 调用 `check_and_retrieve_pipeline()`，按 access mode 加锁，再调用 `execute_internal()`。
- `include/neug/execution/execute/query_cache.h`: `GlobalQueryCache::Get()` cache miss 时调用 `planner_->compilePlan(query)`，再调用 `PlanParser::get().parse_execute_pipeline(...)`。
- `src/compiler/planner/gopt_planner.cc`: `GOptPlanner::compilePlan()` 调用 `ctx->prepare(query)` 生成 logical plan，再用 `GPhysicalConvertor::convert(...)` 生成 physical plan。
- `src/execution/execute/plan_parser.cc`: `PlanParser::init()` 注册 operator builders；`parse_execute_pipeline_with_meta()` 根据 physical plan 构造 `std::vector<std::unique_ptr<IOperator>>`。
- `src/main/query_processor.cc`: `execute_internal()` 调用 `cache_value->pipeline.Execute(...)`，然后 `Sink::sink_results(...)`，最后封装 `QueryResult`。
- `src/execution/execute/pipeline.cc`: `Pipeline::Execute()` 顺序调用每个 `operators_[i]->Eval(...)`。

## C++ 创建 / Open 主调用链

```text
用户代码
neug::NeugDB db;
db.Open(data_dir, ...)

-> NeugDB::Open(const std::string&, ...)
   src/main/neug_db.cc
   - 构造 NeugDBConfig
   - return Open(config)

-> NeugDB::Open(const NeugDBConfig&)
   - config_ = config
   - preprocessConfig()
   - 创建 data_dir
   - FileLock(work_dir_)
   - file_lock_->lock(...)
   - PlanParser::get().init()
   - initAllocators()
   - openGraphAndIngestWals()
   - initPlannerAndQueryProcessor()
   - closed_ = false

-> NeugDB::preprocessConfig()
   - 处理 thread_num
   - 处理 :memory / :memory: 临时目录
   - 设置 is_pure_memory_

-> PlanParser::get().init()
   - 注册 Scan / Edge / Vertex / Project / Join / DDL / Insert 等 operator builders

-> NeugDB::initAllocators()
   - 清理 allocator_dir(work_dir_)
   - 为每个 thread 创建 Allocator

-> NeugDB::openGraphAndIngestWals()
   - graph_.Open(work_dir_, memory_level)
   - WalParserFactory::Init()
   - CreateWalParser(wal_dir(work_dir_))
   - ingestWals(parser)

-> PropertyGraph::Open(work_dir, memory_level)
   - loadSchema(schema_file) 或创建 checkpoint dir
   - 创建 vertex_tables_
   - VertexTable::Open(...)
   - VertexTable::EnsureCapacity(...)
   - 创建 EdgeTable
   - EdgeTable::Open(...)
   - EdgeTable::EnsureCapacity(...)

-> NeugDB::ingestWals(parser)
   - 对 update WAL: UpdateTransaction::IngestWal(...)
   - 对 insert WAL range: InsertTransaction::IngestWal(...)
   - 对 compaction marker: graph_.Compact(...)
   - 更新 last_ts_

-> NeugDB::initPlannerAndQueryProcessor()
   - planner_ = make_shared<GOptPlanner>()
   - planner_->update_meta(schema().to_yaml().value())
   - planner_->update_statistics(graph().get_statistics_json())
   - global_query_cache_ = make_shared<GlobalQueryCache>(planner_)
   - query_processor_ = make_shared<QueryProcessor>(...)
   - connection_manager_ = make_unique<ConnectionManager>(...)
```

## initPlannerAndQueryProcessor 解析

源码位置：`src/main/neug_db.cc`。

```cpp
void NeugDB::initPlannerAndQueryProcessor() {
  if (config_.planner_kind == "gopt") {
    planner_ = std::make_shared<GOptPlanner>();
  } else {
    THROW_INVALID_ARGUMENT_EXCEPTION("Invalid planner kind: " +
                                     config_.planner_kind);
  }
  planner_->update_meta(schema().to_yaml().value());
  planner_->update_statistics(graph().get_statistics_json());

  global_query_cache_ = std::make_shared<execution::GlobalQueryCache>(planner_);

  query_processor_ = std::make_shared<QueryProcessor>(
      graph_, planner_, global_query_cache_, *allocators_[0], thread_num_,
      config_.mode == DBMode::READ_ONLY);

  connection_manager_ = std::make_unique<ConnectionManager>(
      graph_, planner_, query_processor_, config_);
}
```

它做的事不是打开 storage；storage 已经在前一步 `openGraphAndIngestWals()` 中通过 `graph_.Open(...)` 打开并 replay WAL。这个函数负责把已经可用的 `graph_` 接到 query 编译和执行层。

分成四步：

### 1. 创建 planner

```cpp
planner_ = std::make_shared<GOptPlanner>();
```

当前只支持 `config_.planner_kind == "gopt"`。如果不是 `"gopt"`，直接抛 `InvalidArgumentException`。

`GOptPlanner` 是 `IGraphPlanner` 的实现。它的构造函数里创建 compiler 侧的 metadata 和 client context：

```cpp
database = std::make_unique<neug::main::MetadataManager>();
ctx = std::make_unique<neug::main::ClientContext>(database.get());
neug::main::MetadataRegistry::registerMetadata(database.get());
```

所以这里创建的不是 storage graph，而是 query compiler/planner 需要的 metadata manager 和编译上下文。

### 2. 把当前 graph 的 schema/statistics 同步给 planner

```cpp
planner_->update_meta(schema().to_yaml().value());
planner_->update_statistics(graph().get_statistics_json());
```

`schema()` 实际返回 `graph_.schema()`；`graph().get_statistics_json()` 从当前 `PropertyGraph` 取统计信息。

这一步的意义是：planner 编译 Cypher query 时需要知道当前图有哪些 vertex label、edge label、properties、primary key、统计信息等。没有这一步，`GOptPlanner::compilePlan()` 里的 catalog/metadata 就不知道当前数据库结构。

### 3. 创建 GlobalQueryCache

```cpp
global_query_cache_ =
    std::make_shared<execution::GlobalQueryCache>(planner_);
```

`GlobalQueryCache` 持有同一个 `planner_`。执行 query 时，`GlobalQueryCache::Get(schema, query)` 会：

```text
cache hit:
  return CacheValue

cache miss:
  planner_->compilePlan(query)
  PlanParser::get().parse_execute_pipeline(...)
  parse result schema
  parse params type
  cache.emplace(query, CacheValue(...))
```

所以 `GlobalQueryCache` 缓存的不是 SQL 字符串本身，而是已经编译好的执行相关对象：

```text
CacheValue
  - Pipeline
  - ParamsMetaMap
  - result_schema
  - physical::ExecutionFlag
```

### 4. 创建 QueryProcessor

```cpp
query_processor_ = std::make_shared<QueryProcessor>(
    graph_, planner_, global_query_cache_, *allocators_[0], thread_num_,
    config_.mode == DBMode::READ_ONLY);
```

`QueryProcessor` 是 C++ query 执行入口背后的核心对象。它保存：

```text
PropertyGraph& g_
shared_ptr<IGraphPlanner> planner_
shared_ptr<GlobalQueryCache> global_query_cache_
Allocator& allocator_
int32_t max_num_threads_
bool is_read_only_
```

后面 `Connection::Query()` 会直接调用：

```cpp
query_processor_->execute(query_string, access_mode, parameters);
```

`QueryProcessor` 再负责：

```text
1. 判断/解析 access mode
2. 从 GlobalQueryCache 获取 compiled pipeline
3. 根据 read/write 加 shared_lock 或 unique_lock
4. 调用 Pipeline::Execute(...)
5. Sink::sink_results(...)
6. 必要时刷新 planner metadata/statistics
```

### 5. 创建 ConnectionManager

```cpp
connection_manager_ = std::make_unique<ConnectionManager>(
    graph_, planner_, query_processor_, config_);
```

`ConnectionManager` 管理用户拿到的 `Connection`。它内部保存同一份：

```text
PropertyGraph& graph_
shared_ptr<IGraphPlanner> planner_
shared_ptr<QueryProcessor> query_processor_
const NeugDBConfig& config_
```

真正创建 connection 时：

```cpp
std::make_shared<Connection>(graph_, query_processor_);
```

也就是说，`Connection` 本身很薄，主要只是把用户的 `Query()` 转发给共享的 `QueryProcessor`。

### 总结

`initPlannerAndQueryProcessor()` 是 storage 层和 query 层的接线函数：

```text
已打开并恢复好的 graph_
-> schema/statistics
-> GOptPlanner metadata
-> GlobalQueryCache
-> QueryProcessor
-> ConnectionManager
-> Connection::Query()
```

它完成之后，`NeugDB` 才具备接受 C++ connection 和执行 query 的能力。

## C++ Query 执行主调用链

```text
用户代码
auto conn = db.Connect();
conn->Query("MATCH ... RETURN ...", "read");

-> NeugDB::Connect()
   src/main/neug_db.cc
   - return connection_manager_->CreateConnection()

-> ConnectionManager::CreateConnection()
   src/main/connection_manager.cc
   - READ_ONLY: 创建多个 read-only Connection
   - READ_WRITE: 只允许一个 read-write Connection
   - make_shared<Connection>(graph_, query_processor_)

-> Connection::Query(...)
   src/main/connection.cc
   - 检查 IsClosed()
   - query_processor_->execute(query_string, access_mode, parameters)

-> QueryProcessor::execute(...)
   src/main/query_processor.cc
   - check_and_retrieve_pipeline(...)
   - need_exclusive_lock(access_mode)
   - read query: shared_lock
   - write/update/schema query: unique_lock
   - execute_internal(...)

-> QueryProcessor::check_and_retrieve_pipeline(...)
   - 处理 num_threads
   - access mode:
       user_access_mode empty -> planner_->analyzeMode(query_string)
       otherwise -> ParseAccessMode(user_access_mode)
   - global_query_cache_->Get(g_.schema(), query_string)
   - read-only mode 下拒绝 write/schema/batch/procedure 等 flags

-> GlobalQueryCache::Get(schema, query)
   include/neug/execution/execute/query_cache.h
   - cache hit: return CacheValue
   - cache miss:
       planner_->compilePlan(query)
       PlanParser::get().parse_execute_pipeline(schema, ctx_meta, physical_plan)
       parse_result_schema_column_names(...)
       PlanParser::parse_params_type(...)
       cache.emplace(query, CacheValue(...))

-> GOptPlanner::compilePlan(query)
   src/compiler/planner/gopt_planner.cc
   - ctx->prepare(query)
   - statement->logicalPlan
   - GAliasManager
   - GPhysicalConvertor::convert(logicalPlan)
   - GResultSchema::infer(...)
   - return physical::PhysicalPlan + result schema yaml

-> PlanParser::parse_execute_pipeline(...)
   src/execution/execute/plan_parser.cc
   - 遍历 physical plan 的 operators
   - 找匹配的 IOperatorBuilder
   - builder->Build(schema, cur_ctx_meta, plan, i)
   - operators.emplace_back(...)
   - return Pipeline(operators)

-> QueryProcessor::execute_internal(...)
   src/main/query_processor.cc
   - StorageAPUpdateInterface graph(g_, 0, allocator_)
   - cache_value->pipeline.Execute(graph, Context(), parameters, timer)
   - Sink::sink_results(ctx, graph, response)
   - response->schema = cache_value->result_schema
   - QueryResult::From(response->SerializeAsString())
   - update_compiler_meta_if_needed(...)

-> Pipeline::Execute(...)
   src/execution/execute/pipeline.cc
   - for each operator:
       operators_[i]->Eval(graph, params, std::move(ctx), timer)
   - 返回最终 Context
```

## 推荐源码阅读顺序

1. `src/main/neug_db.cc`: 读 `Open()`, `openGraphAndIngestWals()`, `initPlannerAndQueryProcessor()`。
2. `src/storages/graph/property_graph.cc`: 读 `PropertyGraph::Open()`。
3. `src/main/connection_manager.cc` 和 `src/main/connection.cc`: 读连接创建与 `Query()`。
4. `src/main/query_processor.cc`: 读 `execute()`, `check_and_retrieve_pipeline()`, `execute_internal()`。
5. `include/neug/execution/execute/query_cache.h`: 读 `GlobalQueryCache::Get()`。
6. `src/compiler/planner/gopt_planner.cc`: 读 `compilePlan()`。
7. `src/execution/execute/plan_parser.cc`: 读 `init()` 和 `parse_execute_pipeline_with_meta()`。
8. `src/execution/execute/pipeline.cc`: 读 `Pipeline::Execute()`。

## 未验证假设

- 这份调用链基于源码静态阅读，没有跑 debugger。
- `Pipeline::Execute()` 后具体进入哪些 `IOperator::Eval()` 取决于 query 对应的 physical plan；需要针对具体 query 再追 operator builder 和 operator implementation。

## 后续问题

- 对 `CREATE NODE TABLE ...` 这类 schema query，具体会生成哪个 DDL operator？
- 对 `MATCH (n) RETURN n`，实际 physical plan 中 operators 的顺序是什么？
