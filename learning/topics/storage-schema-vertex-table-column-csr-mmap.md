# Storage: Schema -> VertexTable / EdgeTable -> Table -> Column -> CSR -> mmap

## 原始问题

```text
详细解释storage层中schema-vertextable-table-column-csr的逐级存储方式，以及一层层的open，最终的内存布局，以及mmap内存管理的方式
```

## 核心结论

NeuG storage 层不是一条单线的 `Schema -> VertexTable -> Table -> Column -> CSR`。更准确的层级是：

```text
PropertyGraph
  ├── Schema
  │     ├── VertexSchema[label_id]
  │     └── EdgeSchema[(src_label, dst_label, edge_label)]
  │
  ├── vertex_tables_[label_id]
  │     └── VertexTable
  │           ├── IndexerType       // primary key / oid -> lid
  │           ├── Table             // non-primary-key vertex properties
  │           │     └── ColumnBase[]
  │           │           └── IDataContainer / mmap buffer
  │           └── VertexTimestamp   // vertex visibility / deletion state
  │
  └── edge_tables_[edge_triplet_key]
        └── EdgeTable
              ├── out_csr_          // src_lid -> outgoing neighbors
              ├── in_csr_           // dst_lid -> incoming neighbors
              └── Table             // only for unbundled edge properties
                    └── ColumnBase[]
                          └── IDataContainer / mmap buffer
```

所以：

- vertex 的普通属性存在 `Table -> Column`。
- vertex 的主键不在普通属性 `Table` 中，而在 `IndexerType` 的 key column 中。
- edge 的邻接关系存在 CSR。
- edge 属性分两种：bundled 属性直接放进 CSR 的 neighbor `data` 字段；unbundled 属性由 CSR 保存 row id，真实属性放进 `EdgeTable::table_` 的 `Table -> Column`。

## Schema 层

`Schema` 是图类型系统，不直接保存数据行。它保存两类元数据：

```text
Schema
  ├── v_schemas_: vector<shared_ptr<VertexSchema>>
  ├── e_schemas_: unordered_map<uint32_t, shared_ptr<EdgeSchema>>
  ├── vlabel_indexer_: vertex label name -> label id
  ├── elabel_indexer_: edge label name -> label id
  └── tombstone bitsets
```

`VertexSchema` 记录一个点 label 的：

- `label_name`
- `property_types`
- `property_names`
- `primary_keys`
- `default_property_values`
- `vprop_soft_deleted`

`EdgeSchema` 记录一个边三元组的：

- `src_label_name`, `dst_label_name`, `edge_label_name`
- `properties`, `property_names`, `default_property_values`
- `oe_mutable`, `ie_mutable`
- `oe_strategy`, `ie_strategy`
- `sort_key_for_nbr`
- `eprop_soft_deleted`

`PropertyGraph::Open()` 先读 schema，再按 schema 创建真实存储对象。也就是说 schema 决定应该有多少 `VertexTable`、多少 `EdgeTable`、每个 table 有哪些 columns、每个 edge CSR 用哪种策略。

## PropertyGraph::Open 链路

入口有两个：

```text
PropertyGraph::Open(schema, work_dir, memory_level)
  -> schema_ = schema
  -> PropertyGraph::Open(work_dir, memory_level)

PropertyGraph::Open(work_dir, memory_level)
```

核心步骤：

```text
1. memory_level_ = memory_level
2. work_dir_ = work_dir
3. schema_file = schema_path(work_dir)
4. if schema file exists:
     loadSchema(schema_file)
   else:
     create checkpoint dir and keep empty graph

5. vertex_label_total_count_ = schema_.vertex_label_frontier()
6. edge_label_total_count_ = schema_.edge_label_frontier()

7. for each valid vertex label:
     vertex_tables_.emplace_back(schema_.get_vertex_schema(i))

8. remove and recreate runtime/tmp

9. for each valid vertex label:
     vertex_tables_[i].Open(work_dir_, memory_level)
     vertex_tables_[i].EnsureCapacity(max(4096, size + 25%))
     remember vertex capacity

10. for each valid edge triplet:
      EdgeTable edge_table(schema_.get_edge_schema(src, dst, edge))
      edge_table.Open(work_dir_, memory_level)
      edge_table.EnsureCapacity(src_vertex_capacity,
                                dst_vertex_capacity,
                                max(4096, edge_prop_size + 20%))
      edge_tables_.emplace(edge_triplet_key, move(edge_table))

11. initialize vertex mutex array
```

这里有两个容量概念：

- vertex table capacity 来自 indexer/table/timestamp 的容量，最少 4096。
- edge table capacity 对 CSR 和 unbundled edge property table 分别生效；CSR 的 vertex-side capacity 要和 source/destination vertex capacity 对齐。

## VertexTable 层

`VertexTable` 的构造函数只创建对象壳：

```text
VertexTable
  ├── indexer_       = make_shared<IndexerType>()
  ├── table_         = make_unique<Table>()
  ├── vertex_schema_ = schema pointer
  ├── v_ts_          = make_shared<VertexTimestamp>()
  ├── pk_type_       = primary key type
  └── memory_level_  = kInMemory by default
```

真正打开底层文件和 mmap 在 `VertexTable::Open()`：

```text
VertexTable::Open(work_dir, memory_level)
  ├── checkpoint_dir_path = checkpoint_dir(work_dir)
  ├── tmp_dir_path        = tmp_dir(work_dir)
  ├── label_name          = vertex_schema_->label_name
  ├── vertex_tracker_file = checkpoint/vertex_tracker_{label}
  ├── indexer_filename    = {indexer_prefix}_vertex_map_{label}
  │
  ├── if kSyncToFile:
  │     indexer_->open(indexer_filename, checkpoint_dir, work_dir)
  │     table_->open(vertex_table_{label}, work_dir, property_names, types)
  │
  ├── if kInMemory:
  │     indexer_->open_in_memory(checkpoint_dir/indexer_filename)
  │     table_->open_in_memory(vertex_table_{label}, work_dir, ...)
  │
  ├── if kHugePagePreferred:
  │     indexer_->open_with_hugepages(checkpoint_dir/indexer_filename)
  │     table_->open_with_hugepages(vertex_table_{label}, work_dir, ...)
  │
  └── v_ts_->Open(checkpoint/vertex_tracker_{label})
```

插入 vertex 时：

```text
VertexTable::AddVertex(id, props, vid, ts, insert_safe)
  ├── vid = insert_vertex_pk(id, ts, insert_safe)
  │     ├── indexer_->get_index(id)
  │     ├── if not exists: indexer_->insert(id)
  │     └── v_ts_->InsertVertex(vid, ts)
  │
  └── table_->insert(vid, props, insert_safe)
```

因此 `vid/lid` 是所有属性列的行号：

```text
vid = 10
  indexer keys[10]         = original id / primary key
  table.columns_[0][10]    = property 0
  table.columns_[1][10]    = property 1
  vertex_timestamp[10]     = visibility state
```

`VertexTable::EnsureCapacity(capacity)` 会同步扩容：

```text
indexer_->reserve(capacity)
table_->resize(capacity, default_properties)
v_ts_->Reserve(capacity)
```

## Table 层

`Table` 是列式表，不是行式表。

```text
Table
  ├── col_id_map_: column name -> column id
  ├── col_names_:  column id -> column name
  └── columns_:    vector<shared_ptr<ColumnBase>>
```

`Table::open()` 的逻辑：

```text
Table::open(name, work_dir, col_names, property_types)
  ├── name_ = name
  ├── work_dir_ = work_dir
  ├── snapshot_dir_ = checkpoint_dir(work_dir)
  ├── initColumns(col_names, property_types)
  │     └── columns_[i] = CreateColumn(property_types[i])
  └── for each column i:
        columns_[i]->open(name + ".col_" + i,
                          snapshot_dir_,
                          tmp_dir(work_dir))
```

不同 memory level 对应：

```text
kSyncToFile:
  Table::open()
  Column::open(snapshot_dir/name.col_i, tmp_dir/name.col_i)

kInMemory:
  Table::open_in_memory()
  Column::open_in_memory(checkpoint_dir/name.col_i)

kHugePagePreferred:
  Table::open_with_hugepages()
  Column::open_with_hugepages(checkpoint_dir/name.col_i)
```

写入一行时，`Table` 只是把 row 拆到各列：

```text
Table::insert(row_id, values)
  for i in columns:
    columns_[i]->set_any(row_id, values[i], insert_safe)
```

读取一行时再临时组装：

```text
Table::get_row(row_id)
  return [columns_[0]->get_prop(row_id),
          columns_[1]->get_prop(row_id),
          ...]
```

## Column 层

`ColumnBase` 是统一接口。实际列主要是 `TypedColumn<T>`：

```text
TypedColumn<T>
  ├── buffer_: unique_ptr<IDataContainer>
  └── size_:   element count
```

定长列最终就是一段连续数组：

```text
buffer_->GetData()
  -> reinterpret_cast<T*>(data)

memory:
  [T row0][T row1][T row2] ... [T rowN]

byte size:
  size_ * sizeof(T)
```

定长列写入：

```text
set_value(index, val)
  reinterpret_cast<T*>(buffer_->GetData())[index] = val
```

定长列扩容：

```text
resize(row_count)
  buffer_->Resize(row_count * sizeof(T))
```

### StringColumn 布局

`StringColumn` 是 `TypedColumn<std::string_view>`，不是 `std::string[]`。

它有两个 mmap buffer 和一个 pos：

```text
StringColumn
  ├── items_buffer_: string_item[row_count]
  ├── data_buffer_:  char[data_capacity]
  ├── pos_:          append cursor
  └── width_:        max string width / default estimate
```

`string_item` 是：

```text
offset: 48 bits
length: 16 bits
```

布局：

```text
items_buffer_
  row0 -> {offset=0,  length=5}
  row1 -> {offset=5,  length=3}
  row2 -> {offset=8,  length=4}

data_buffer_
  bytes[0..5)   = row0 string
  bytes[5..8)   = row1 string
  bytes[8..12)  = row2 string
```

写字符串时总是 append：

```text
offset = pos_.fetch_add(value.size())
items[row_id] = {offset, value.size()}
memcpy(data + offset, value.data(), value.size())
```

如果更新同一行，旧字符串 bytes 不会立刻回收。dump/compact 相关逻辑会在落盘时重新整理数据。

## EdgeTable 和 CSR 层

`EdgeTable` 按一个 edge triplet 创建：

```text
(src_label, dst_label, edge_label) -> EdgeTable
```

内部结构：

```text
EdgeTable
  ├── meta_: EdgeSchema
  ├── out_csr_: CsrBase
  ├── in_csr_:  CsrBase
  ├── table_:   Table
  ├── table_idx_
  └── capacity_
```

`EdgeTable` 构造时根据 schema 选择 CSR 类型：

```text
if meta_->is_bundled():
  property_type = first edge property type, or Empty
  out_csr_ = create_csr(oe_mutable, oe_strategy, property_type)
  in_csr_  = create_csr(ie_mutable, ie_strategy, property_type)
else:
  out_csr_ = create_csr(oe_mutable, oe_strategy, UInt64)
  in_csr_  = create_csr(ie_mutable, ie_strategy, UInt64)
```

unbundled 时 CSR 的 `data` 字段是 `uint64_t row_id`，指向 `table_` 中的边属性行。

`EdgeTable::Open()`：

```text
EdgeTable::Open(work_dir, memory_level)
  ├── ckp_dir = checkpoint_dir(work_dir)
  ├── ie_prefix = ie_{src}_{edge}_{dst}
  ├── oe_prefix = oe_{src}_{edge}_{dst}
  ├── edata_prefix = e_{src}_{edge}_{dst}_data
  │
  ├── open in_csr_ and out_csr_ according to memory_level
  │
  └── if !bundled:
        open table_ according to memory_level
        load statistics file into capacity_ and table_idx_
```

### CSR 类型

CSR 抽象是 `CsrBase`，常见实现：

```text
CsrBase
  ├── ImmutableCsr<T>        // multiple neighbors per source
  ├── MutableCsr<T>          // multiple neighbors per source, with timestamp
  ├── SingleImmutableCsr<T>  // one neighbor slot per source
  ├── SingleMutableCsr<T>    // one neighbor slot per source, with timestamp
  └── EmptyCsr<T>
```

neighbor entry：

```text
ImmutableNbr<T>
  ├── vid_t neighbor
  └── T data

MutableNbr<T>
  ├── vid_t neighbor
  ├── atomic<timestamp_t> timestamp
  └── T data
```

### ImmutableCsr 内存布局

`ImmutableCsr<T>`：

```text
degree_list_buffer_: int[vnum]
  degree[src] = number of neighbors

nbr_list_buffer_: ImmutableNbr<T>[edge_count]
  all adjacency lists are stored contiguously

adj_list_buffer_: ImmutableNbr<T>*[vnum]
  runtime pointer array, rebuilt from degree + nbr list
```

打开时：

```text
ImmutableCsr::open_internal(snapshot_prefix, tmp_prefix, mem_level)
  ├── load_meta(snapshot_prefix)
  ├── degree_list_buffer_ = OpenContainer(prefix + ".deg")
  ├── nbr_list_buffer_    = OpenContainer(prefix + ".nbr")
  ├── adj_list_buffer_    = OpenContainer("", tmp_prefix + ".adj")
  ├── adj_list_buffer_->Resize(vnum * sizeof(nbr_t*))
  └── rebuild pointers:
        cur = nbr_list
        for src in [0, vnum):
          if degree[src] != 0:
            adj[src] = cur
          else:
            adj[src] = NULL
          cur += degree[src]
```

最终内存形态：

```text
degree:
  src0: 2
  src1: 0
  src2: 3

nbr_list:
  [src0_nbr0][src0_nbr1][src2_nbr0][src2_nbr1][src2_nbr2]

adj:
  adj[0] -> &nbr_list[0]
  adj[1] -> NULL
  adj[2] -> &nbr_list[2]
```

`.deg` 和 `.nbr` 是核心持久化文件；`.adj` 是运行期 pointer array，重新 open 时可以重建。

### MutableCsr 内存布局

`MutableCsr<T>` 多一个 per-source capacity：

```text
degree_list_: int[vnum]
cap_list_:    int[vnum]
nbr_list_:    MutableNbr<T>[sum(cap)]
adj_list_buffer_: MutableNbr<T>*[vnum]
locks_: SpinLock[vnum]
```

打开时按 `cap[src]` 切分连续 `nbr_list_`：

```text
cur = nbr_list
for src in [0, vnum):
  adj[src] = cur
  edge_count += degree[src]
  cur += cap[src]
```

单条插入时：

```text
put_edge(src, dst, data, ts)
  lock(src)
  if degree[src] == cap[src]:
    new_cap = max(cap + cap/2, 8)
    new_buffer = allocator.allocate(new_cap * sizeof(nbr_t))
    memcpy old neighbors
    adj[src] = new_buffer
    cap[src] = new_cap

  offset = degree[src]++
  adj[src][offset] = {dst, ts, data}
  unlock(src)
```

因此 mutable CSR 允许单个 source 局部扩容，不需要每次都重排完整 `nbr_list_`。

## 文件命名和目录布局

关键目录：

```text
{work_dir}/schema
{work_dir}/checkpoint/
{work_dir}/runtime/tmp/
{work_dir}/runtime/allocator/
{work_dir}/wal/
```

vertex 相关：

```text
checkpoint/vertex_map_{Label}.meta
checkpoint/vertex_map_{Label}.keys
checkpoint/vertex_map_{Label}.indices
checkpoint/vertex_table_{Label}.col_0
checkpoint/vertex_table_{Label}.col_1.items
checkpoint/vertex_table_{Label}.col_1.data
checkpoint/vertex_table_{Label}.col_1.pos
checkpoint/vertex_tracker_{Label}
```

edge 相关：

```text
checkpoint/oe_{Src}_{Edge}_{Dst}.meta
checkpoint/oe_{Src}_{Edge}_{Dst}.deg
checkpoint/oe_{Src}_{Edge}_{Dst}.nbr

checkpoint/ie_{Src}_{Edge}_{Dst}.meta
checkpoint/ie_{Src}_{Edge}_{Dst}.deg
checkpoint/ie_{Src}_{Edge}_{Dst}.nbr

checkpoint/e_{Src}_{Edge}_{Dst}_data.col_0
checkpoint/e_{Src}_{Edge}_{Dst}_data.col_1.items
checkpoint/e_{Src}_{Edge}_{Dst}_data.col_1.data
checkpoint/statistics_{Src}_{Edge}_{Dst}
```

`runtime/tmp/` 是 sync-to-file 模式的工作副本位置。open 时从 checkpoint copy 到 tmp，然后 mmap tmp。dump 时再把 tmp 或内存数据写回新的 checkpoint。

## mmap / IDataContainer 层

所有 Column、CSR、Indexer 的大块数据最终都通过 `IDataContainer` 统一管理：

```text
IDataContainer
  ├── GetData()
  ├── GetDataSize()
  ├── Resize(bytes)
  ├── Open(path)
  ├── Sync()
  ├── Dump(path)
  ├── Close()
  └── IsDirty()
```

实现类：

```text
MMapContainer
  ├── FilePrivateMMap  // file + MAP_PRIVATE
  ├── FileSharedMMap   // file + MAP_SHARED
  ├── AnonMMap         // anonymous mmap
  └── AnonHugeMMap     // hugepage preferred anonymous mmap
```

`OpenContainer(snapshot_file, tmp_file, memory_level)` 是入口：

```text
if kSyncToFile:
  prepare_container_file(snapshot_file, tmp_file)
    if snapshot exists:
      copy snapshot -> tmp
    else:
      create tmp file with FileHeader only
  return OpenDataContainer(kSyncToFile, tmp_file)

else:
  return OpenDataContainer(memory_level, snapshot_file)
```

`OpenDataContainer()` 的选择：

```text
kInMemory:
  if file_name empty:
    AnonMMap
  else:
    FilePrivateMMap
    if file exists: Open(file)

kHugePagePreferred:
  AnonHugeMMap
  if file exists: Open(file)

kSyncToFile:
  FileSharedMMap
  if file missing: create file with FileHeader
  Open(file)
```

### MMapContainer::Open

文件容器 open 后的内存布局总是跳过 `FileHeader`：

```text
file bytes:
  [FileHeader: md5][payload bytes...]

mmap_data_ -> beginning of file mapping
data_      -> mmap_data_ + sizeof(FileHeader)
size_      -> mmap_size_ - sizeof(FileHeader)
```

因此上层看到的 `GetData()` 永远是 payload，不包含 header。

### Resize 行为

普通 `MMapContainer::Resize(size)`：

```text
1. allocate anonymous MAP_PRIVATE | MAP_ANONYMOUS region
2. memcpy old payload to new region
3. munmap old region
4. data_ = new region
5. size_ = size
```

`FileSharedMMap::Resize(size)` 不一样：

```text
1. real_size = size + sizeof(FileHeader)
2. munmap old mapping
3. open backing file O_RDWR
4. ftruncate(fd, real_size)
5. mmap file again with MAP_SHARED
6. data_ = mmap_data_ + sizeof(FileHeader)
7. size_ = payload size
```

所以 sync-to-file 模式的 resize 会改变 tmp 文件大小，并重新建立 shared mapping。

### Sync / Dump

`FileSharedMMap::Sync()`：

```text
1. MD5(payload)
2. compare with FileHeader.data_md5
3. if changed:
     update header md5
     msync(mmap_data_, mmap_size_, MS_SYNC)
```

`FileSharedMMap::Dump(path)`：

```text
1. Sync()
2. if target path == current path:
     Close()
3. else:
     Close()
     try rename(tmp_file, target_path)
     if cross-device: copy_file then unlink tmp
```

普通 `MMapContainer::Dump(path)`：

```text
1. compute MD5(payload)
2. fwrite FileHeader
3. fwrite payload
4. Close()
```

## 三种 MemoryLevel 的语义

```text
kInMemory
  - snapshot file exists: MAP_PRIVATE 读入文件映射
  - file missing / empty name: anonymous mmap
  - 修改不直接写回 snapshot
  - dump 时通过 fwrite 写 checkpoint

kSyncToFile
  - open 前 copy checkpoint -> runtime/tmp
  - tmp file 使用 MAP_SHARED
  - resize 通过 ftruncate 扩 tmp file
  - sync/dump 时 msync + rename/copy 到 checkpoint

kHugePagePreferred
  - 使用 AnonHugeMMap
  - 优先 MAP_HUGETLB anonymous allocation
  - 失败时 fallback 到 regular anonymous mmap
  - 如果 snapshot 存在，会读文件内容到 hugepage/anonymous memory
```

## 一个完整例子

假设 schema 中有：

```text
Vertex Person(id UINT64 primary key, name VARCHAR, age INT32)
Edge Person -[KNOWS {since INT32}]-> Person
```

打开后大致是：

```text
PropertyGraph
  ├── schema_
  │     ├── VertexSchema(Person)
  │     └── EdgeSchema(Person, Person, KNOWS)
  │
  ├── vertex_tables_[person_label]
  │     ├── indexer_
  │     │     ├── vertex_map_PERSON.keys       // id values
  │     │     └── vertex_map_PERSON.indices    // hash table oid -> lid
  │     │
  │     ├── table_
  │     │     ├── vertex_table_PERSON.col_0.items/data/pos  // name
  │     │     └── vertex_table_PERSON.col_1                 // age int32[]
  │     │
  │     └── vertex_tracker_PERSON
  │
  └── edge_tables_[person, person, knows]
        ├── out_csr_
        │     ├── oe_PERSON_KNOWS_PERSON.deg
        │     └── oe_PERSON_KNOWS_PERSON.nbr
        │
        ├── in_csr_
        │     ├── ie_PERSON_KNOWS_PERSON.deg
        │     └── ie_PERSON_KNOWS_PERSON.nbr
        │
        └── table_
              └── absent if edge property is bundled into CSR
```

如果 `since` 是 bundled，`since` 存在 `Nbr<int32_t>::data` 中。

如果是 unbundled，CSR 中存的是 `uint64_t row_id`：

```text
out_csr_.nbr[src_offset].data = row_id
in_csr_.nbr[dst_offset].data  = row_id
edge_table.table_.columns_[0][row_id] = since
```

## 易错点

- `Schema` 是 metadata，不是数据容器。
- `VertexTable` 不是只有 `Table`，它还有主键 indexer 和 timestamp tracker。
- vertex primary key 不在 `table_` 普通属性列中。
- `Table` 是列式存储，`get_row()` 只是临时组装。
- CSR 属于 `EdgeTable`，不是 `VertexTable` 的子层。
- CSR 的 `.adj` / `.buf` 是运行期 pointer array，可由 `.deg/.cap/.nbr` 重建。
- `kSyncToFile` 不是直接 mmap checkpoint 文件，而是先 copy 到 `runtime/tmp`，再 mmap tmp。
- `MMapContainer::GetData()` 指向 payload，已经跳过 `FileHeader`。
- `FileSharedMMap::Resize()` 会 `ftruncate` 文件并重新 mmap。
- `StringColumn` 更新时 append 新 bytes，旧 bytes 不立即回收。

