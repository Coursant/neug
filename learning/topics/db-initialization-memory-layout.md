# DB Initialization Memory Layout

## 原始问题

```text
从db初始化开始的内存布局是什么样
```

## 我的理解

- 这里的“DB 初始化”从 Python `Database(...)` / C++ `NeugDB::Open(...)` 开始。
- 重点不是查询执行链，而是打开数据库后哪些对象被创建、谁拥有谁、图数据在内存里如何展开。
- 需要把 `NeugDB`、`PropertyGraph`、点表、边表、CSR、列存和 allocator 串成一张心智图。

## 代码事实

- `tools/python_bind/src/py_database.h`: `PyDatabase` 持有 `std::unique_ptr<NeugDB> database`。
- `tools/python_bind/src/py_database.h`: `PyDatabase` 构造函数创建 `NeugDBConfig`，设置 mode/planner/memory level，然后调用 `database->Open(config)`。
- `src/main/neug_db.cc`: `NeugDB::Open(config)` 依次调用 `preprocessConfig()`、`PlanParser::get().init()`、`initAllocators()`、`openGraphAndIngestWals()`、`initPlannerAndQueryProcessor()`。
- `include/neug/main/neug_db.h`: `NeugDB` 内嵌 `PropertyGraph graph_`，并持有 planner、query processor、connection manager、query cache 和 allocators。
- `src/storages/graph/property_graph.cc`: `PropertyGraph::Open()` 加载 schema，创建并打开 `VertexTable` / `EdgeTable`，最后创建 vertex-level mutex。
- `src/storages/graph/vertex_table.cc`: `VertexTable::Open()` 根据 `MemoryLevel` 打开 indexer、属性 `Table` 和 vertex timestamp tracker。
- `src/storages/graph/edge_table.cc`: `EdgeTable::Open()` 打开 incoming/outgoing CSR；非 bundled 边属性还会打开属性 `Table`。
- `include/neug/storages/allocators.h`: `ArenaAllocator` 按 batch 管理 `IDataContainer`，用于 WAL ingest 和 mutable CSR 动态邻接数组分配。

## 修正后的结论

从 DB 初始化开始，内存布局可以按“外壳对象 -> DB 核心对象 -> 图存储 -> 表/CSR/列存”看。

### 1. 初始化链路

```text
Python Database
  -> PyDatabase
     -> unique_ptr<NeugDB>
        -> NeugDB::Open(config)
           -> PlanParser::get().init()
           -> initAllocators()
           -> graph_.Open(...)
           -> ingestWals(...)
           -> initPlannerAndQueryProcessor()
```

关键入口：

- `tools/python_bind/neug/database.py`: Python `Database.__init__()` 做参数校验并创建 `neug_py_bind.PyDatabase`。
- `tools/python_bind/src/py_database.h`: `PyDatabase` 构造 C++ `NeugDB` 并调用 `Open(config)`。
- `src/main/neug_db.cc`: `NeugDB::Open(const NeugDBConfig&)` 是 C++ 初始化主入口。

### 2. Python/C++ 包装层

Python 的 `Database(...)` 自身不持有图数据；真正数据在 `PyDatabase::database -> NeugDB` 里。

```text
Database Python object
  _database: PyDatabase

PyDatabase
  recursive_mutex mtx_
  string db_dir_
  unique_ptr<NeugDB> database
  unique_ptr<NeugDBService> service_  // only when server enabled
```

`PyDatabase` 是 Python binding 层的外壳。它负责把 Python 参数转成 `NeugDBConfig`，比如：

- `database_path`
- `max_thread_num`
- `mode`
- `planner`
- `checkpoint_on_close`
- `buffer_strategy`

然后把这些交给 `NeugDB::Open(config)`。

### 3. NeugDB 主体布局

`NeugDB` 是 DB 生命周期的根对象。核心字段：

```text
NeugDB
  timestamp_t last_compaction_ts_
  timestamp_t last_ts_
  atomic<bool> closed_
  bool is_pure_memory_
  int thread_num_
  NeugDBConfig config_
  string work_dir_
  unique_ptr<FileLock> file_lock_

  PropertyGraph graph_                     // 直接内嵌，不是指针
  shared_ptr<IGraphPlanner> planner_
  shared_ptr<QueryProcessor> query_processor_
  unique_ptr<ConnectionManager> connection_manager_
  shared_ptr<GlobalQueryCache> global_query_cache_

  mutex mutex_
  vector<shared_ptr<Allocator>> allocators_
```

最关键的一点：`PropertyGraph graph_` 是 `NeugDB` 的值成员，不是 `unique_ptr`。因此图存储主体跟随 `NeugDB` 对象生命周期存在。

### 4. NeugDB::Open 期间创建什么

`NeugDB::Open(config)` 的初始化顺序：

```text
config_ = config
preprocessConfig()
work_dir_ = config_.data_dir
create data_dir if not exists
file_lock_ = make_unique<FileLock>(work_dir_)
file_lock_->lock(...)
PlanParser::get().init()
initAllocators()
openGraphAndIngestWals()
initPlannerAndQueryProcessor()
closed_ = false
```

其中 `preprocessConfig()` 会处理内存库：

```text
if data_dir == "" or ":memory" or ":memory:":
  create /tmp/neug_db_<timestamp>
  is_pure_memory_ = true
  config_.data_dir = temp dir
else:
  is_pure_memory_ = false
```

所以 NeuG 的 “in-memory database” 仍然会有一个临时 `work_dir_`，只是默认用内存级别的 container 承载数据。

### 5. Allocator 布局

`initAllocators()` 会先清理 allocator 目录，然后按 `config_.thread_num` 创建 allocator：

```text
allocators_: vector<shared_ptr<Allocator>>
  allocator[0]
  allocator[1]
  ...
```

`Allocator` 当前是 `ArenaAllocator`：

```text
ArenaAllocator
  MemoryLevel strategy_
  string prefix_
  vector<unique_ptr<IDataContainer>> mmap_buffers_
  void* cur_buffer_
  size_t cur_loc_
  size_t cur_size_
  size_t allocated_memory_
  size_t allocated_batches_
```

它按 16MB batch 增长。典型用途：

- WAL ingest 时给恢复出的结构分配内存。
- mutable CSR 插边时，源点邻接数组扩容使用 allocator 分配新 buffer。

### 6. PropertyGraph 布局

`openGraphAndIngestWals()` 先调用：

```text
graph_.Open(work_dir_, config_.memory_level)
```

`PropertyGraph` 是图存储顶层对象：

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

打开流程：

```text
load graph.yaml -> schema_
if schema missing:
  create checkpoint dir and build empty graph

vertex_label_total_count_ = schema_.vertex_label_frontier()
edge_label_total_count_ = schema_.edge_label_frontier()

for each valid vertex label:
  vertex_tables_.emplace_back(VertexTable(schema))
  vertex_tables_[i].Open(...)
  EnsureCapacity(max(4096, size + size / 4))

for each valid (src_label, dst_label, edge_label):
  EdgeTable(schema edge triplet)
  edge_table.Open(...)
  edge_table.EnsureCapacity(src_vertex_capacity, dst_vertex_capacity, edge_capacity)
  edge_tables_[encoded_triplet] = edge_table

v_mutex_.resize(vertex_label_total_count_)
```

点表按 label 直接放在 `vector<VertexTable>` 中。边表按三元组编码放在 `unordered_map<uint32_t, EdgeTable>` 中。

### 7. VertexTable 布局

每个 vertex label 对应一个 `VertexTable`：

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

它分成三块：

```text
indexer_
  primary key / oid -> local vid_t

table_
  非主键属性列

v_ts_
  点的时间戳 / 有效性信息
```

主键不在普通属性表里重复存一份，而是通过 `indexer_->get_keys()` 暴露成引用列。

属性表是列存：

```text
Table
  unordered_map<string, int> col_id_map_
  vector<string> col_names_
  vector<shared_ptr<ColumnBase>> columns_
  vector<bool> col_deleted_

TypedColumn<T>
  unique_ptr<IDataContainer> buffer_
  size_t size_

buffer layout:
  T[0], T[1], T[2], ...
```

所以点属性不是“每个点一个 struct”，而是“每个属性一条连续列”。

### 8. EdgeTable / CSR 布局

每个 `(src_label, dst_label, edge_label)` 对应一个 `EdgeTable`：

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

边表同时维护两个 CSR：

```text
out_csr_: src -> dst
in_csr_:  dst -> src
```

这样出边遍历和入边遍历都可以直接走邻接结构。

如果 edge schema 是 bundled，边属性直接存在 CSR entry 里；如果不是 bundled，`EdgeTable::table_` 会打开一张属性表，CSR entry 里存属性表 row index。

### 9. Immutable CSR 和 Mutable CSR

Immutable CSR：

```text
ImmutableCsr<T>
  adj_list_buffer_       contiguous ImmutableNbr<T> entries
  degree_list_buffer_    int degree per vertex
  nbr_list_buffer_
  timestamp_t unsorted_since_
  atomic<uint64_t> edge_num_

ImmutableNbr<T>
  vid_t neighbor
  T data
```

Mutable CSR：

```text
MutableCsr<T>
  SpinLock* locks_       // per source vertex
  adj_list_buffer_       // MutableNbr<T>* array
  degree_list_           // int degree per source vertex
  cap_list_              // int capacity per source vertex
  nbr_list_
  timestamp_t unsorted_since_
  atomic<uint64_t> edge_num_

MutableNbr<T>
  vid_t neighbor
  atomic<timestamp_t> timestamp
  T data
```

mutable CSR 插入单条边时大概是：

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

因此：

- immutable CSR 更像紧凑连续数组，适合 checkpoint / compact 后读取。
- mutable CSR 是每个源点一个可扩容邻接数组，适合增量写入。

### 10. Planner / QueryProcessor 初始化后的引用关系

`graph_.Open(...)` 和 WAL recovery 结束后，`initPlannerAndQueryProcessor()` 创建查询相关对象：

```text
planner_ = make_shared<GOptPlanner>()
planner_->update_meta(schema().to_yaml().value())
planner_->update_statistics(graph().get_statistics_json())

global_query_cache_ = make_shared<GlobalQueryCache>(planner_)

query_processor_ = make_shared<QueryProcessor>(
  graph_,
  planner_,
  global_query_cache_,
  *allocators_[0],
  thread_num_,
  read_only
)

connection_manager_ = make_unique<ConnectionManager>(
  graph_,
  planner_,
  query_processor_,
  config_
)
```

这时对象关系是：

```text
NeugDB
  graph_  <--------------------+
  planner_                     |
  global_query_cache_ -> planner_
  query_processor_ ------------+
    references graph_
    shared_ptr planner_
    shared_ptr global_query_cache_
    Allocator& allocators_[0]
  connection_manager_
    references graph_
    shared_ptr planner_
    shared_ptr query_processor_
```

创建连接时：

```text
ConnectionManager::CreateConnection()
  -> make_shared<Connection>(graph_, query_processor_)

Connection
  PropertyGraph& graph_
  shared_ptr<QueryProcessor> query_processor_
  atomic<bool> is_closed_
```

所以 `Connection` 不是数据拥有者，只是拿着 `graph_` 引用和共享的 `QueryProcessor`。

## 最短心智模型

```text
PyDatabase
  owns NeugDB

NeugDB
  owns PropertyGraph directly
  owns planner/cache/query_processor/connection_manager
  owns per-thread ArenaAllocator list

PropertyGraph
  owns Schema
  owns VertexTable vector
  owns EdgeTable map

VertexTable
  owns PK indexer
  owns columnar property Table
  owns VertexTimestamp

EdgeTable
  owns outgoing CSR
  owns incoming CSR
  optionally owns edge property Table

Table
  owns TypedColumn<T> columns

TypedColumn<T>
  owns IDataContainer
  stores contiguous T array

CSR
  stores neighbor entries
  Mutable CSR: per-source dynamic adjacency arrays
  Immutable CSR: compact adjacency buffers
```

一句话总结：DB 初始化后，`NeugDB` 是根对象；它内嵌一个 `PropertyGraph`。`PropertyGraph` 里按 label 存 `VertexTable`，按边三元组存 `EdgeTable`。点属性走列存 `Table -> TypedColumn<T> -> IDataContainer`，边关系走双向 CSR，边属性要么 inline bundled 在 CSR entry 里，要么通过 CSR entry 里的 index 指向 `EdgeTable::table_` 的列存属性。

## 后续问题

- `MemoryLevel::kInMemory`、`kSyncToFile`、`kHugePagePreferred` 分别对应哪几种 `IDataContainer` 实现？
- `Schema::generate_edge_label(src, dst, edge)` 的编码规则是什么？
- `EdgeSchema::is_bundled()` 的判定条件是什么，哪些属性类型会导致 unbundled？
