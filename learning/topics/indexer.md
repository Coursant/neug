# NeuG Indexer

## 原始问题

```text
indexer详解
写成笔记放入learning中
```

## 我的理解

- 这次问题没有先给出个人假设；目标是把 NeuG 中 `indexer` 的源码结构、职责、数据流和使用位置整理成可复习笔记。
- 重点不是数据库里的二级索引，而是 NeuG 当前源码中的 id mapping：把外部 key 映射成内部连续 id。

## 代码事实

- `include/neug/utils/indexers.h`: `IndexerType = LFIndexer<vid_t>`，这是图存储里使用的顶点主键索引类型。
- `include/neug/utils/indexers.h`: `IndexerBuilderType<KEY_T> = IdIndexer<KEY_T, vid_t>`，是一个基于模板 key 类型的轻量 id indexer 别名。
- `include/neug/utils/id_indexer.h`: `LFIndexer<INDEX_T>` 保存 `keys_`、`indices_`、`num_elements_`、`num_slots_minus_one_`、`hash_policy_` 和 `GHash<Property>`。
- `include/neug/utils/id_indexer.h`: `LFIndexer::init` 按 `DataType` 创建底层 key column；当前支持 `int64`, `int32`, `uint64`, `uint32`, `varchar`。
- `include/neug/utils/id_indexer.h`: `LFIndexer::insert` 分配新的 `lid`，写入 `keys_`，再通过 hash + linear probing 把 `lid` 放入 `indices_` 空槽。
- `include/neug/utils/id_indexer.h`: `LFIndexer::get_index` 用 hash + linear probing 查询 key；槽位中保存的是 `lid`，再用 `keys_->get_prop(lid) == oid` 验证命中。
- `include/neug/utils/id_indexer.h`: `LFIndexer::rehash` 根据 `max_load_factor = 0.5` 扩容并重建哈希槽。
- `include/neug/utils/id_indexer.h`: `LFIndexer::open`, `open_in_memory`, `open_with_hugepages`, `dump`, `load_meta`, `dump_meta` 负责 `.keys`, `.indices`, `.meta` 三类文件的持久化和加载。
- `include/neug/storages/graph/vertex_table.h`: `VertexTable` 构造时创建 `std::shared_ptr<IndexerType>`，并用顶点主键类型初始化 indexer。
- `src/storages/graph/vertex_table.cc`: `VertexTable::AddVertex` 通过 `insert_vertex_pk` 写入主键索引，再把属性写入 `table_`。
- `src/storages/graph/vertex_table.cc`: `VertexTable::get_index` 先查 `indexer_`，再用 `VertexTimestamp` 判断该 `lid` 在指定时间戳是否有效。
- `src/storages/graph/vertex_table.cc`: `VertexTable::DeleteVertex` 不会从 indexer 删除 key，而是通过 `v_ts_` 做 tombstone。
- `include/neug/storages/graph/schema.h`: `Schema` 中的 `vlabel_indexer_` 和 `elabel_indexer_` 是 `IdIndexer<std::string, label_t>`，用于 label name 到 label id 的映射。
- `src/storages/graph/schema.cc`: `contains_vertex_label`, `get_vertex_label_id`, `get_vertex_label_name`, `Compact` 等函数通过 `IdIndexer` 查询或反查 label。

## 修正后的结论

NeuG 里当前最核心的 `indexer` 是 id mapping，不是通用查询优化意义上的 property index。

它的核心任务是：

```text
external id / primary key / label name -> internal continuous id
```

在顶点表中：

```text
vertex primary key / OID -> vid_t / lid
```

在 schema 中：

```text
vertex label string -> label_t
edge label string   -> label_t
```

### `LFIndexer`: 运行时顶点主键索引

`LFIndexer<vid_t>` 是 `VertexTable` 使用的索引。它的数据组织可以理解为：

```text
keys_[lid] = original key
indices_[hash slot] = lid
```

查询时不是在哈希槽中直接保存 key，而是保存 `lid`：

```text
hash(oid)
  -> slot
  -> indices_[slot] gives lid
  -> keys_[lid] == oid confirms hit
```

这样设计的好处是 `lid` 连续稳定，可以直接作为属性列、CSR 邻接表、timestamp table 的下标。

### 插入流程

`VertexTable::AddVertex` 的主路径是：

```text
AddVertex(id, props)
  -> insert_vertex_pk(id)
      -> indexer_->get_index(id, vid)
      -> indexer_->insert(id, insert_safe)
      -> v_ts_->InsertVertex(vid, ts)
  -> table_->insert(vid, props, insert_safe)
```

`LFIndexer::insert` 自身主要负责分配和写入，不负责业务层去重。顶点是否已存在由 `VertexTable::insert_vertex_pk` 先查 `get_index` 决定。

### 查询流程

`VertexTable::get_index` 比 `LFIndexer::get_index` 多一层语义：

```text
indexer_->get_index(oid, lid)
  -> 如果没找到: false
  -> 如果找到了但 v_ts_ 认为该 lid 在 ts 下无效: false
  -> 否则: true
```

所以 `indexer_` 只回答“这个 key 对应哪个 lid”，顶点是否仍然有效由 `VertexTimestamp` 回答。

### 删除语义

删除顶点不会把 key 从 `LFIndexer` 里移除：

```text
DeleteVertex(id)
  -> get_index(id, lid, ts)
  -> v_ts_->RemoveVertex(lid)
```

这意味着 `lid` 和 key mapping 会继续存在，只是 timestamp 层标记不可见。`VertexTable::Compact` 里也有 TODO：还没有 compact unused lid in indexer/table。

### 持久化格式

`LFIndexer` 对应三类文件：

```text
{name}.keys     // 按 lid 保存原始 key column
{name}.indices  // hash slot array, slot value is lid
{name}.meta     // type, num_elements, slot count, hash policy metadata
```

`VertexTable::Open` 会根据 `MemoryLevel` 选择 `open`, `open_in_memory`, `open_with_hugepages`。因此 indexer 同时服务 embedded/in-memory 和 mmap/file-backed 场景。

### `IdIndexer`: schema 元数据索引

`IdIndexer<KEY_T, INDEX_T>` 是另一套轻量哈希索引，主要用于 schema 元数据：

```cpp
IdIndexer<std::string, label_t> vlabel_indexer_;
IdIndexer<std::string, label_t> elabel_indexer_;
```

它使用 `keys_ + indices_ + distances_`，更接近 robin-hood hashing 的内存结构；支持 `Serialize/Deserialize`，适合 schema 这种小规模元数据映射。

## 对比总结

| 类型 | 主要用途 | key 类型 | value | 存储方式 | 典型位置 |
|---|---|---|---|---|---|
| `LFIndexer<vid_t>` | 顶点主键到内部 id | `Property` | `vid_t/lid` | column + mmap/container hash slots | `VertexTable::indexer_` |
| `IdIndexer<std::string, label_t>` | schema label 到 label id | typed key | `label_t` | in-memory vectors | `Schema::vlabel_indexer_`, `Schema::elabel_indexer_` |

## 易错点

- `IndexerType` 不是一个 abstract interface，而是 `LFIndexer<vid_t>` 的类型别名。
- `LFIndexer::insert` 不做完整业务去重；上层应先查 `get_index`。
- 删除顶点不删除 indexer entry，只修改 `VertexTimestamp` 可见性。
- `LFIndexer` 的 `.indices` 里存的是 `lid`，不是 key 本身。
- `get_key(lid)` 是反查主键，通常用于 `VertexTable::GetOid` 或测试输出。
- schema label 的 `IdIndexer` 和 vertex primary key 的 `LFIndexer` 是两套不同实现。

## 后续问题

- `LFIndexer::insert` 的 CAS 只保护槽位插入，`reserve/rehash` 与并发插入之间的调用约束需要结合上层写入流程继续确认。
- `LFIndexer` 在 WAL recovery、batch ingest 和 normal insert 三类路径中的 `insert_safe` 使用边界值得单独梳理。
- 当前没有二级 property index；后续如果出现 `WHERE n.prop = value` 的索引优化，需要和这里的 id indexer 区分。
