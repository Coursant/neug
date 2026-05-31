# NeugDB graph_ 与 VertexTable 初始化

## 原始问题

```text
neugdb graph_ vertextable 的初始化方式，以及默认初始化是如何进行的
```

## 结论

`NeugDB::graph_` 是 `NeugDB` 的值成员，不是指针。创建 `NeugDB db;` 时，`graph_` 会自动调用 `PropertyGraph` 默认构造函数，形成一个空图对象。

但 `VertexTable` 不会在 `NeugDB` 构造时默认创建。只有两种情况会创建 `VertexTable`：

1. 打开已有数据库时，`PropertyGraph::Open()` 从 schema 里读出 vertex label，并为每个 label 创建对应的 `VertexTable`。
2. 空库运行 `CreateVertexType` 创建点类型时，新增 schema label 后创建对应的 `VertexTable`。

默认情况下，一个新建空库没有 schema，所以 `vertex_tables_` 是空的。第一个 vertex table 要等创建第一个 vertex label 后才出现。

## 代码事实

### 1. NeugDB 内嵌 PropertyGraph

`NeugDB` 的核心字段在 `include/neug/main/neug_db.h`：

```cpp
PropertyGraph graph_;
```

这说明 `graph_` 的生命周期跟随 `NeugDB`，不是动态分配的 `unique_ptr` / `shared_ptr`。

### 2. NeugDB 构造函数没有显式初始化 graph_

`src/main/neug_db.cc`:

```cpp
NeugDB::NeugDB()
    : last_compaction_ts_(0),
      last_ts_(0),
      closed_(true),
      is_pure_memory_(false),
      thread_num_(1) {}
```

这里没有写 `graph_(...)`，所以 C++ 会对成员 `graph_` 做默认构造，也就是调用 `PropertyGraph::PropertyGraph()`。

### 3. PropertyGraph 默认构造为空图

`src/storages/graph/property_graph.cc`:

```cpp
PropertyGraph::PropertyGraph()
    : vertex_label_total_count_(0),
      edge_label_total_count_(0),
      memory_level_(MemoryLevel::kInMemory) {}
```

默认状态：

```text
vertex_label_total_count_ = 0
edge_label_total_count_ = 0
memory_level_ = MemoryLevel::kInMemory
vertex_tables_ = empty
edge_tables_ = empty
```

也就是说，`NeugDB` 构造完成后已经有 `graph_`，但还没有任何点表或边表。

## Open 时的初始化链路

调用 `NeugDB::Open(...)` 后，主链路是：

```text
NeugDB::Open(...)
  -> preprocessConfig()
  -> initAllocators()
  -> openGraphAndIngestWals()
      -> graph_.Open(work_dir_, config_.memory_level)
      -> ingestWals(...)
  -> initPlannerAndQueryProcessor()
```

其中 `graph_.Open(...)` 是图存储初始化的关键。

`PropertyGraph::Open()` 的核心逻辑：

```text
if schema file exists:
  loadSchema(schema_file)
else:
  create checkpoint dir
  build empty graph

vertex_label_total_count_ = schema_.vertex_label_frontier()
edge_label_total_count_ = schema_.edge_label_frontier()

for each vertex label:
  vertex_tables_.emplace_back(schema_.get_vertex_schema(i))
  vertex_tables_[i].Open(work_dir_, memory_level)
  vertex_tables_[i].EnsureCapacity(...)
```

如果是全新的空库，没有 schema 文件：

```text
schema_.vertex_label_frontier() == 0
```

所以 `for each vertex label` 不会执行，`vertex_tables_` 仍然为空。

## CreateVertexType 时如何创建 VertexTable

空库里第一个 `VertexTable` 通常来自 `PropertyGraph::CreateVertexType()`。

核心步骤：

```text
1. 检查 label 是否已存在。
2. 从 config.GetProperties() 收集 property name/type/default value。
3. 检查 primary key：
   - 必须至少一个 primary key
   - 不支持多个 primary key
   - primary key 必须存在于 properties
   - primary key 类型只能是 int64/int32/uint64/uint32/varchar
4. schema_.AddVertexLabel(...)
5. vertex_tables_.emplace_back(schema_.get_vertex_schema(vertex_label_id))
6. vtable.Open(work_dir_, memory_level_)
7. vtable.EnsureCapacity(4096)
8. 更新 vertex_label_total_count_ 和 v_mutex_
```

所以 `VertexTable` 的创建依赖 `VertexSchema`。没有 vertex schema，就没有 vertex table。

## VertexTable 构造函数做了什么

`include/neug/storages/graph/vertex_table.h`:

```cpp
VertexTable(std::shared_ptr<const VertexSchema> vertex_schema)
    : indexer_(std::make_shared<IndexerType>()),
      table_(std::make_unique<Table>()),
      vertex_schema_(vertex_schema),
      v_ts_(std::make_shared<VertexTimestamp>()),
      memory_level_(MemoryLevel::kInMemory),
      work_dir_("") {
  assert(vertex_schema->primary_keys.size() == 1);
  pk_type_ = std::get<0>(vertex_schema->primary_keys[0]);
  indexer_->init(pk_type_.id());
}
```

构造后得到：

```text
VertexTable
  indexer_        primary key -> local vertex id 的索引
  table_          非主键属性列存表
  vertex_schema_  当前点 label 的 schema
  v_ts_           vertex timestamp / 删除状态 tracker
  pk_type_        primary key 的 DataType
  memory_level_   默认 kInMemory
  work_dir_       默认空字符串
```

注意：构造函数只创建对象壳和 indexer 类型，不真正打开底层数据文件或 container。真正打开在 `VertexTable::Open()`。

## VertexTable::Open 做了什么

`src/storages/graph/vertex_table.cc`:

```cpp
void VertexTable::Open(const std::string& work_dir, MemoryLevel memory_level) {
  memory_level_ = memory_level;
  work_dir_ = work_dir;

  const auto& label_name = vertex_schema_->label_name;
  ...
  if (memory_level_ == MemoryLevel::kSyncToFile) {
    indexer_->open(...);
    table_->open(...);
  } else if (memory_level_ == MemoryLevel::kInMemory) {
    indexer_->open_in_memory(...);
    table_->open_in_memory(...);
  } else if (memory_level_ == MemoryLevel::kHugePagePreferred) {
    indexer_->open_with_hugepages(...);
    table_->open_with_hugepages(...);
  }
  v_ts_->Open(vertex_tracker_filename);
}
```

默认 `NeugDBConfig` 里：

```cpp
memory_level(MemoryLevel::kInMemory)
```

所以默认路径是：

```text
indexer_->open_in_memory(...)
table_->open_in_memory(...)
v_ts_->Open(...)
```

`v_ts_->Open(...)` 如果没有已有 `.ts` 文件，会执行：

```cpp
Init(0, 4096);
```

这给 vertex timestamp tracker 初始化默认容量。

## 默认容量如何初始化

新建 vertex type 时，`CreateVertexType()` 会显式调用：

```cpp
vtable.EnsureCapacity(4096);
```

`VertexTable::EnsureCapacity()`：

```cpp
capacity = std::max(capacity, 4096UL);
indexer_->reserve(capacity);
table_->resize(capacity, vertex_schema_->get_default_properties());
v_ts_->Reserve(capacity);
```

因此默认点表容量至少是 4096。它会同时扩：

- primary key indexer
- 属性列存 table
- vertex timestamp tracker

打开已有 checkpoint 时，`PropertyGraph::Open()` 会根据已有 size 再决定容量：

```text
v_size < 4096 ? 4096 : v_size + v_size / 4
```

所以：

- 空表默认 4096。
- 已有数据表默认保留约 25% 额外空间。

## 心智模型

可以把初始化分成三层：

```text
NeugDB db;
  graph_ exists
  but graph_ is empty

db.Open(...)
  graph_.Open(...)
    if schema has labels:
      create VertexTable for each label
    else:
      no VertexTable yet

CREATE VERTEX TYPE / CreateVertexType
  schema_.AddVertexLabel(...)
  vertex_tables_.emplace_back(VertexTable(schema))
  VertexTable::Open(...)
  VertexTable::EnsureCapacity(4096)
```

## 一句话总结

`NeugDB::graph_` 默认随 `NeugDB` 构造为一个空的 `PropertyGraph`；`VertexTable` 不会默认存在，而是由 schema 驱动创建。打开已有库时按 schema 创建，空库创建 vertex type 时创建。创建后默认走 `MemoryLevel::kInMemory`，打开 indexer、属性 table、timestamp tracker，并预留至少 4096 容量。

