# NeuG 调用链、核心结构与内存布局笔记

本文基于 CodeGraph 对当前仓库的结构索引整理，重点回答三个问题：

- 一条 Cypher 查询从 Python/C++ API 到执行器、存储层的完整调用链。
- NeuG 运行时的核心结构如何分层。
- 点、边、属性、结果集在内存中如何组织。

## 1. 一条完整查询调用链

以 Python API `conn.execute("MATCH ... RETURN ...")` 为例，完整链路如下：

```text
tools/python_bind/neug/connection.py
  Connection.execute()****
    -> tools/python_bind/src/py_connection.cc
       PyConnection::execute()
         -> include/neug/main/connection.h / src/main/connection.cc
            Connection::Query()
              -> src/main/query_processor.cc
                 QueryProcessor::execute()
                   -> QueryProcessor::check_and_retrieve_pipeline()
                     -> GlobalQueryCache::Get(schema, query)
                       -> GOptPlanner::compilePlan(query)
                         -> ClientContext::prepare(query)
                           -> ClientContext::parseQuery()
                             -> Parser::parseQuery()
                           -> ClientContext::prepareNoLock()
                             -> Binder::bind()
                             -> Planner::getBestPlan()
                             -> optimizer::Optimizer::optimize()
                         -> GPhysicalConvertor::convert(logicalPlan)
                       -> PlanParser::parse_execute_pipeline()
                         -> PlanParser::parse_execute_pipeline_with_meta()
                           -> IOperatorBuilder::Build(...)
                   -> QueryProcessor::execute_internal()
                     -> Pipeline::Execute(...)
                       -> IOperator::Eval(...) for each operator
                         -> StorageReadInterface / StorageAPUpdateInterface
                           -> PropertyGraph / VertexTable / EdgeTable / CSR
                     -> Sink::sink_results(...)
                     -> QueryResult::From(serialized QueryResponse)
```

关键点：

- Python 层只做参数校验和异常包装。`Connection.execute()` 最终调用 `_py_connection.execute(query, access_mode, parameters)`。
- `PyConnection::execute()` 把 Python dict 序列化成 `rapidjson::Document`，然后调用 C++ `Connection::Query()`。
- `Connection::Query()` 只检查连接状态，然后把查询交给共享的 `QueryProcessor`。
- `QueryProcessor::execute()` 先取得 access mode 和已编译 pipeline，再按访问模式选择互斥锁或共享锁执行。
- `GlobalQueryCache::Get()` 是编译缓存入口：缓存未命中时调用 `planner_->compilePlan(query)`，再把 physical plan 解析成 `Pipeline`。
- `GOptPlanner::compilePlan()` 内部走 Kuzu 风格编译链：parse -> bind -> logical plan -> optimize -> gopt physical protobuf plan。
- `PlanParser` 把 `physical::PhysicalPlan` 的 protobuf operators 映射成执行期 `IOperator` 列表。
- `Pipeline::Execute()` 顺序执行 `operators_[i]->Eval(...)`，每个算子吃一个 `Context`，吐出下一个 `Context`。
- `Sink::sink_results()` 把最终 `Context` 里的列转换进 protobuf `QueryResponse`。
- `QueryProcessor::execute_internal()` 用 protobuf arena 临时构造 `QueryResponse`，序列化后再用 `QueryResult::From()` 反序列化成 `QueryResult` 持有的 `shared_ptr<QueryResponse>`。

## 2. 编译链核心结构

编译链的核心对象是：

```text
GOptPlanner
  owns MetadataManager
  owns ClientContext

ClientContext::prepare()
  parseQuery()
  prepareNoLock()

PreparedStatement
  parsedStatement
  statementResult
  parameterMap
  logicalPlan
  readOnly flags

Binder
  Statement -> BoundStatement

Planner
  BoundStatement -> LogicalPlan

Optimizer
  LogicalPlan rewrite / optimize

GPhysicalConvertor
  LogicalPlan -> physical::PhysicalPlan protobuf

PlanParser
  physical::PhysicalPlan -> Pipeline
```

`GOptPlanner::compilePlan()` 返回的是：

```cpp
result<std::pair<physical::PhysicalPlan, std::string>>
```

其中：

- `physical::PhysicalPlan` 是执行计划 protobuf。
- `std::string` 是结果 schema 的 YAML dump，之后被 `parse_result_schema_column_names()` 转成 `MetaDatas`。

编译缓存结构在 `include/neug/execution/execute/query_cache.h`：

```text
GlobalQueryCache
  planner_
  cache_: query string -> shared_ptr<CacheValue>

CacheValue
  Pipeline pipeline
  ParamsMetaMap params_type
  MetaDatas result_schema
  physical::ExecutionFlag flags
```

也就是说，重复查询不再重新 parse/bind/plan，而是直接复用 `Pipeline` 和结果 schema。

## 3. 执行链核心结构

执行器核心抽象在 `include/neug/execution/execute/operator.h`：

```cpp
class IOperator {
  virtual result<Context> Eval(
      IStorageInterface& graph,
      const ParamsMap& params,
      Context&& ctx,
      OprTimer* timer) = 0;
};

class IOperatorBuilder {
  virtual result<OpBuildResultT> Build(
      const Schema& schema,
      const ContextMeta& ctx_meta,
      const physical::PhysicalPlan& plan,
      int op_idx) = 0;
};
```

`PlanParser::parse_execute_pipeline_with_meta()` 按 physical operator 类型查找 builder，调用 `Build()` 生成 `unique_ptr<IOperator>`，最终构造：

```text
Pipeline
  vector<unique_ptr<IOperator>> operators_
```

`Pipeline::Execute()` 的模型很直接：

```text
ctx0
  -> operator[0].Eval(...)
ctx1
  -> operator[1].Eval(...)
ctx2
  -> ...
ctxN
```

以 scan 为例：

```text
ScanOprBuilder::Build()
  physical::Scan -> ScanParams + predicate
  -> ScanWithGPredOpr / ScanWithSPredOpr / FilterOidsGPredOpr

ScanWithGPredOpr::Eval()
  -> Scan::scan_vertex(...)
    -> dynamic_cast<const StorageReadInterface&>(graph)
    -> graph.GetVertexSet(label)
    -> MSVertexColumnBuilder
    -> ctx.set(alias, builder.finish())
```

执行期 `Context` 是列式中间结果。点列不是直接存完整对象，而是存 `(label, vid)`：

```text
SLVertexColumn
  vector<vid_t> vertices_
  label_t label_

MSVertexColumn
  vector<pair<label_t, vector<vid_t>>> vertices_
  set<label_t> labels_
```

这意味着查询执行阶段传递的是轻量引用，真正的属性值在需要投影、过滤或 sink 时再通过 storage interface 读取。

## 4. 存储层核心结构

顶层图存储是 `PropertyGraph`：

```text
PropertyGraph
  string work_dir_
  Schema schema_
  vector<shared_ptr<mutex>> v_mutex_
  vector<VertexTable> vertex_tables_
  unordered_map<uint32_t, EdgeTable> edge_tables_
  size_t vertex_label_total_count_
  size_t edge_label_total_count_
  MemoryLevel memory_level_
```

访问方式：

- 点表按 `label_t` 直接索引：`vertex_tables_[vertex_label]`。
- 边表按三元组编码索引：`schema_.generate_edge_label(src_label, dst_label, edge_label)`。
- 出边视图：`GetGenericOutgoingGraphView(src, dst, edge, ts)`。
- 入边视图：`GetGenericIncomingGraphView(dst, src, edge, ts)`。
- 点属性列：`GetVertexPropertyColumn(label, prop)`。
- 边属性访问器：`GetEdgeDataAccessor(src, dst, edge, prop)`。

## 5. 点表内存布局

`VertexTable` 负责一个 vertex label 的所有点：

```text
VertexTable
  shared_ptr<IndexerType> indexer_
  unique_ptr<Table> table_
  DataType pk_type_
  shared_ptr<const VertexSchema> vertex_schema_
  shared_ptr<VertexTimestamp> v_ts_
  MemoryLevel memory_level_
  string work_dir_
```

点表分成三块：

1. `indexer_`
   - 负责 primary key/OID 到 local id/LID 的映射。
   - `get_index(oid, lid, ts)` 查询外部主键对应的内部 `vid_t`。

2. `table_`
   - 属性列存。
   - 只存非主键属性；主键属性通过 `indexer_->get_keys()` 暴露为引用列。

3. `v_ts_`
   - 点级时间戳/有效性信息。
   - `GetVertexSet(ts)` 返回 `VertexSet(LidNum(), *v_ts_, ts)`，执行器扫描时据此过滤可见点。

属性列布局来自 `Table` 和 `TypedColumn<T>`：

```text
Table
  unordered_map<string, int> col_id_map_
  vector<string> col_names_
  vector<shared_ptr<ColumnBase>> columns_
  vector<bool> col_deleted_

TypedColumn<T>
  unique_ptr<IDataContainer> buffer_
  size_t size_
```

`TypedColumn<T>` 的实际数据是连续数组：

```cpp
reinterpret_cast<T*>(buffer_->GetData())[index] = val;
```

因此点属性是按列连续存储，不是每个点一个 struct。

## 6. 边表与 CSR 内存布局

`EdgeTable` 负责一个 `(src_label, dst_label, edge_label)` 三元组：

```text
EdgeTable
  shared_ptr<const EdgeSchema> meta_
  string work_dir_
  MemoryLevel memory_level_
  atomic<int32_t> csr_alter_version_
  unique_ptr<CsrBase> out_csr_
  unique_ptr<CsrBase> in_csr_
  unique_ptr<Table> table_
  atomic<uint64_t> table_idx_
  atomic<uint64_t> capacity_
```

边表同时维护：

- `out_csr_`：按源点查出边。
- `in_csr_`：按目标点查入边。
- `table_`：边属性列。当边属性不能 bundled inline 时，CSR 的 edge data 里存的是属性表 row index。

CSR 抽象：

```text
CsrBase
  get_generic_view(ts)
  edge_num()
  put_generic_edge(...)
  batch_delete_edges(...)
  compact()
```

### Immutable CSR

`ImmutableCsr<EDATA_T>`：

```text
adj_list_buffer_      contiguous ImmutableNbr<EDATA_T> entries
degree_list_buffer_   int degree per vertex
nbr_list_buffer_      neighbor storage backing
unsorted_since_
edge_num_
```

`ImmutableNbr<EDATA_T>`：

```text
vid_t neighbor
EDATA_T data
```

`ImmutableNbr<EmptyType>` 用 union 复用 `neighbor/data`，空属性边只需要邻居 id。

### Mutable CSR

`MutableCsr<EDATA_T>`：

```text
locks_                SpinLock per source vertex
adj_list_buffer_      array of MutableNbr<EDATA_T>* per source vertex
degree_list_          int degree per source vertex
cap_list_             int capacity per source vertex
nbr_list_             backing storage / persisted neighbor area
unsorted_since_
edge_num_
```

`MutableNbr<EDATA_T>`：

```text
vid_t neighbor
atomic<timestamp_t> timestamp
EDATA_T data
```

插入单条边时，`MutableCsr::put_edge()` 的内存行为是：

```text
buffers = reinterpret_cast<nbr_t**>(adj_list_buffer_->GetData())
sizes   = reinterpret_cast<int*>(degree_list_->GetData())
caps    = reinterpret_cast<int*>(cap_list_->GetData())

lock source vertex
  if sizes[src] == caps[src]:
    allocate new neighbor array with ArenaAllocator
    memcpy old neighbors
    update buffers[src]
  write buffers[src][sizes[src]]:
    neighbor = dst
    data = edge data
    timestamp = ts
  sizes[src]++
unlock
```

所以 mutable CSR 是“每个源点一个动态邻接数组”，并且按源点加锁；immutable CSR 更偏向紧凑连续布局。

## 7. GenericView 如何隐藏 CSR 差异

执行器不直接关心 mutable/immutable 或 single/multiple CSR，而是用 `GenericView` / `NbrList` / `NbrIterator`。

关键布局描述是：

```cpp
struct NbrIterConfig {
  int stride : 16;
  int ts_offset : 8;
  int data_offset : 8;
};
```

`get_generic_view(ts)` 会根据实际 CSR 类型填：

- `stride = sizeof(nbr_t)`
- `ts_offset = offsetof(nbr_t, timestamp)`，immutable CSR 为 `0`
- `data_offset = offsetof(nbr_t, data)`

`NbrIterator` 内部只保存裸指针和 layout 配置：

```text
NbrIterator
  const void* cur
  const void* end
  NbrIterConfig cfg
  timestamp_t timestamp
```

读取邻居：

```cpp
*reinterpret_cast<const vid_t*>(cur)
```

读取边属性指针：

```cpp
static_cast<const char*>(cur) + cfg.data_offset
```

MVCC 可见性过滤：

```text
while cur != end and get_timestamp() > read_timestamp:
  cur += stride
```

注意：immutable CSR 的 `ts_offset == 0`，但构造 view 时传入的 timestamp 是 `max - 1`，配合 entry 首字段 `neighbor` 作为读取值，可以让不可变边默认可见。这是一个依赖布局的低层技巧，读这段代码时要特别注意。

## 8. 边属性的两种布局

边属性通过 `EdgeDataAccessor` 统一读取：

```text
EdgeDataAccessor
  DataTypeId data_type_
  ColumnBase* data_column_
```

两种模式：

1. Bundled / inline
   - `data_column_ == nullptr`
   - `Nbr.data` 里直接存属性值。
   - 读取时按 `data_offset` 直接 reinterpret。

2. Column-based / unbundled
   - `data_column_ != nullptr`
   - `Nbr.data` 里存 `size_t` 类型的属性表 row index。
   - 读取时先取 index，再到 `TypedColumn<T>` 里读真实值。

因此，边的邻接关系永远在 CSR；边属性可能在 CSR entry 内，也可能在独立列存表中。

## 9. 结果集内存布局

执行完成后：

```text
Context
  tag_ids
  alias -> IContextColumn

Sink::sink_results()
  Context columns -> QueryResponse protobuf arrays

QueryResult
  shared_ptr<QueryResponse> response_
```

`QueryResult` 是 protobuf response 的轻量包装：

```text
QueryResult
  shared_ptr<neug::QueryResponse> response_
```

迭代器 `QueryResult::const_iterator` 只保存：

```text
const QueryResponse* response_
size_t row_index_
```

所以 API 返回的结果不是流式游标，而是完整 `QueryResponse` 已经物化在内存中的结果集。

## 10. 最短心智模型

可以把 NeuG 当前实现理解成四层：

```text
API 层
  Python Connection / C++ Connection

编译层
  Parser -> Binder -> Planner -> Optimizer -> GPhysicalConvertor

执行层
  PhysicalPlan protobuf -> PlanParser -> Pipeline -> IOperator::Eval -> Context

存储层
  PropertyGraph
    VertexTable: PK index + columnar properties + vertex timestamps
    EdgeTable: outgoing CSR + incoming CSR + optional edge property table
```

查询执行时，`Context` 里流动的是列式中间结果和 `(label, vid)` 引用；真正图数据在 `PropertyGraph` 的点表、边表、CSR 和属性列中。
