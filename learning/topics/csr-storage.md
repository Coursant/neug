# NeuG CSR Mechanism And Low-Level Storage

## 原始问题

```text
详细探索csr机制和底层存储
```

## 我的理解

- 这次问题没有先给出个人假设；目标是从源码层面理解 NeuG 的 CSR 类族如何保存边、如何遍历、如何持久化，以及它和 `EdgeTable`、`Table`、`IDataContainer` 的边界。
- 这里的 CSR 指 `include/neug/storages/csr/` 下的图邻接存储，不是 query planner 或 compiler 里的表结构。

## 代码事实

- `include/neug/storages/csr/csr_base.h`: `CsrBase` 是统一接口，提供 `get_generic_view`, `open`, `dump`, `resize`, `batch_put_edges`, `delete_edge`, `compact`, `edge_num` 等操作。
- `include/neug/storages/csr/csr_base.h`: `CsrType` 包含 `kImmutable`, `kMutable`, `kSingleMutable`, `kSingleImmutable`, `kEmpty`。
- `include/neug/storages/csr/nbr.h`: `ImmutableNbr<EDATA_T>` 包含 `neighbor` 和 `data`，没有 timestamp。
- `include/neug/storages/csr/nbr.h`: `MutableNbr<EDATA_T>` 包含 `neighbor`, `std::atomic<timestamp_t> timestamp`, `data`。
- `include/neug/storages/csr/mutable_csr.h`: `MutableCsr` 保存 `adj_list_buffer_`, `degree_list_`, `cap_list_`, `nbr_list_`, `locks_`, `unsorted_since_`, `edge_num_`。
- `src/storages/csr/mutable_csr.cc`: `MutableCsr::open_internal` 从 `.nbr`, `.deg`, `.cap` 重建每个 vertex 的 adjacency list pointer array。
- `src/storages/csr/mutable_csr.cc`: `MutableCsr::put_edge` 使用 per-source `SpinLock`，必要时通过 `Allocator` 给单个 source 的邻接表扩容。
- `src/storages/csr/mutable_csr.cc`: `MutableCsr::batch_put_edges` 会整体重排并重建 `nbr_list_`，适合批量导入。
- `include/neug/storages/csr/immutable_csr.h`: `ImmutableCsr` 保存 `adj_list_buffer_`, `degree_list_buffer_`, `nbr_list_buffer_`, `unsorted_since_`, `edge_num_`。
- `src/storages/csr/immutable_csr.cc`: `ImmutableCsr::open_internal` 从连续 `nbr_list_buffer_` 和 `degree_list_buffer_` 重建 `adj_list_buffer_` pointer array。
- `src/storages/csr/immutable_csr.cc`: `ImmutableCsr::batch_put_edges` 通过调整 degree、扩展连续 `nbr_list_buffer_`、倒序 `memmove` 来插入边。
- `include/neug/storages/csr/generic_view.h`: `GenericView` 是轻量只读视图，记录 `adjlists_`, `degrees_`, `NbrIterConfig`, `timestamp_`, `unsorted_since_`。
- `include/neug/storages/csr/generic_view.h`: `NbrIterator` 根据 `cfg.stride`, `cfg.ts_offset`, `cfg.data_offset` 解释不同 CSR 布局，并按 timestamp 跳过不可见边。
- `src/storages/graph/edge_table.cc`: `EdgeTable` 持有 `out_csr_` 和 `in_csr_`，分别服务 outgoing 和 incoming 遍历。
- `src/storages/graph/edge_table.cc`: `EdgeTable` 根据 `EdgeSchema` 的 `oe_mutable`, `ie_mutable`, `oe_strategy`, `ie_strategy`, property type 创建具体 CSR 实现。
- `src/storages/graph/edge_table.cc`: bundled edge property 直接存在 CSR neighbor entry 的 `data` 字段；unbundled edge property 在 CSR 中保存 `uint64_t row_id`，真实属性存在 `table_`。
- `include/neug/storages/container/i_container.h`: CSR 底层数据统一通过 `IDataContainer` 暴露 `GetData`, `Resize`, `Dump`, `Close`。
- `src/storages/container/container_utils.cc`: `OpenContainer` 根据 `MemoryLevel` 选择 anonymous mmap、file private mmap、file shared mmap 或 hugepage preferred mmap。

## 修正后的结论

NeuG 的边存储不是单一 CSR，而是一组按“是否可变”和“每个 source 是否最多一条边”组合出的 CSR 实现：

```text
CsrBase
  ├── ImmutableCsr<T>        // multiple edges per source, no timestamp in nbr
  ├── MutableCsr<T>          // multiple edges per source, nbr has timestamp
  ├── SingleImmutableCsr<T>  // one slot per source, no timestamp in nbr
  ├── SingleMutableCsr<T>    // one slot per source, nbr has timestamp
  └── EmptyCsr<T>
```

`EdgeTable` 同时维护两个 CSR：

```text
out_csr_: src_lid -> dst_lid list
in_csr_:  dst_lid -> src_lid list
```

所以一条逻辑边会写两份结构：一份 outgoing，一份 incoming。这样从源点扩展和从终点反查都可以直接走邻接表。

## 邻接项结构

### immutable neighbor

`ImmutableNbr<T>`:

```text
neighbor: vid_t
data:     T
```

删除 immutable 边时没有 timestamp 可改，所以用 `neighbor == max(vid_t)` 标记删除。

### mutable neighbor

`MutableNbr<T>`:

```text
neighbor:  vid_t
timestamp: atomic<timestamp_t>
data:      T
```

删除 mutable 边时把 `timestamp` 写成 `INVALID_TIMESTAMP` / `max(timestamp_t)`。读取时 `NbrIterator` 会跳过 `timestamp > read_ts` 的边。

## Multiple CSR: 多邻接表

### ImmutableCsr

`ImmutableCsr<T>` 的核心布局：

```text
degree_list_buffer_: int[vnum]
  degree_list_buffer_[src] = degree(src)

nbr_list_buffer_: ImmutableNbr<T>[edge_num]
  所有 source 的邻接表连续拼在一起

adj_list_buffer_: ImmutableNbr<T>*[vnum]
  adj_list_buffer_[src] = &nbr_list_buffer_[offset(src)]
```

`adj_list_buffer_` 是运行时重建出来的 pointer array，方便快速定位某个 source 的邻接表；真正持久化的是 `.deg` 和 `.nbr`。

打开时：

```text
load .meta -> unsorted_since, edge_num
open .deg
open .nbr
allocate .adj pointer array
for each src:
  adj[src] = current nbr pointer
  current += degree[src]
```

文件大致是：

```text
oe_A_E_B.meta
oe_A_E_B.deg
oe_A_E_B.nbr
```

`.adj` 只是 runtime/tmp 里的 pointer array，不作为 snapshot 的核心数据。

### MutableCsr

`MutableCsr<T>` 多了 capacity 和 per-source list allocation：

```text
degree_list_: int[vnum]       // 当前有效/占用 degree
cap_list_:    int[vnum]       // 每个 source 的邻接表容量
nbr_list_:    MutableNbr<T>[] // dump/open 时的连续存储
adj_list_buffer_: MutableNbr<T>*[vnum]
locks_: SpinLock[vnum]
```

打开 snapshot 时，`MutableCsr::open_internal` 用 `.deg` 和 `.cap` 从连续 `.nbr` 里切出每个 source 的邻接表：

```text
ptr = nbr_list_.data
for src in vertices:
  adj[src] = ptr
  ptr += cap[src]
```

注意这里移动的是 `cap[src]`，不是 `degree[src]`，因为 `.nbr` 中保存了每个 source 的 reserved capacity 区间。

`put_edge` 单条插入时：

```text
lock(src)
if degree[src] == cap[src]:
  new_cap = max(cap + cap/2, 8)
  new_buffer = allocator.allocate(new_cap * sizeof(nbr))
  memcpy old neighbors
  adj[src] = new_buffer
  cap[src] = new_cap
offset = degree[src]++
adj[src][offset] = {dst, timestamp, data}
edge_num++
unlock(src)
```

所以 mutable CSR 单条插入可以对单个 source 局部扩容，不必整体重排。

`batch_put_edges` 则会为所有 source 重新计算新 capacity，并把邻接表整理到一个新的连续 `nbr_list_` 中，适合 bulk load。

## Single CSR: 一个 source 一个 slot

`SingleImmutableCsr<T>` 和 `SingleMutableCsr<T>` 用于 `EdgeStrategy::kSingle`，也就是每个 source 最多一个 neighbor。

布局很简单：

```text
nbr_list_buffer_: Nbr<T>[vnum]
nbr_list_buffer_[src] = the single outgoing/incoming edge slot
```

没有 `degree_list`，也没有 `adj_list_buffer`。`GenericView::get_edges(v)` 对 single CSR 直接返回：

```text
start = adjlists + v * stride
end   = start + stride
```

未使用或删除的 slot 用：

- immutable: `neighbor == max(vid_t)`
- mutable: `timestamp == max(timestamp_t)`

## GenericView 如何统一遍历

不同 CSR 的物理布局不同，但都能返回 `GenericView`。关键是 `NbrIterConfig`：

```text
stride      // 每个 neighbor entry 的字节大小
ts_offset   // timestamp 字段偏移；immutable 为 0
data_offset // data 字段偏移
```

multiple view:

```text
GenericView(adjlists pointer array, degrees array, cfg, ts, unsorted_since)
```

single view:

```text
GenericView(nbr array, cfg, ts, unsorted_since)
```

`NbrIterator` 不知道具体类型，只按字节偏移解释：

```text
vertex = *(vid_t*)cur
timestamp = *(timestamp_t*)(cur + ts_offset)
data_ptr = cur + data_offset
cur += stride
```

这就是 NeuG 用一个 `GenericView` 覆盖 mutable/immutable、single/multiple 的关键。

一个明显的实现细节是：immutable CSR 的 `ts_offset = 0`，但 `NbrIterator::get_timestamp()` 会从 entry 起始位置读 timestamp，也就是读到 `neighbor` 字段。实际调用中 immutable view 的 read timestamp 被设置为 `max(timestamp_t) - 1`，普通 `vid_t` neighbor 不会大于它，因此不会被过滤掉。删除的 immutable edge 用 `neighbor == max(vid_t)`，会大于 read timestamp，从而被跳过。这个设计把“删除标记”和“timestamp visibility”复用了同一个过滤逻辑。

## Edge property: bundled vs unbundled

`EdgeTable` 根据 schema 决定边属性怎么存。

### bundled

如果 edge schema 是 bundled，CSR 的 `data` 字段直接存边属性：

```text
MutableNbr<T>
  neighbor
  timestamp
  data = actual edge property
```

`EdgeDataAccessor` 中 `data_column_ == nullptr`，读取时直接从 `it.get_data_ptr()` 解释出属性值。

bundled 适合：

- 0 个属性，也就是 `EmptyType`
- 1 个非字符串属性

源码里 `create_csr` 不支持 `DataTypeId::kVarchar` 作为 bundled CSR data。

### unbundled

如果 edge schema 不是 bundled，CSR 的 `data` 字段存的是 `uint64_t row_id`：

```text
CSR edge data = row_id
EdgeTable::table_[row_id] = real edge properties
```

插入时：

```text
row_id = table_idx_++
csr data = row_id
table_->insert(row_id, edge_data)
```

读取时 `EdgeDataAccessor` 通过 row id 到 `table_` 的列存储中取真实属性：

```text
idx = *(size_t*)it.get_data_ptr()
property = data_column_->get_prop(idx)
```

这和前一篇 `Table` 笔记接上：边的多属性/字符串属性放在 `Table` 的列存储里，CSR 只保存 row id。

## 删除和 compaction

Mutable CSR 删除：

```text
timestamp = max(timestamp_t)
edge_num--
```

Immutable CSR 删除：

```text
neighbor = max(vid_t)
edge_num--
```

`compact()` 才真正移除删除项：

- `ImmutableCsr::compact` 会把未删除 neighbor 前移，缩小 `nbr_list_buffer_`，重建 pointer array。
- `MutableCsr::compact` 会在每个 source 的邻接表内部前移未删除项，但不缩小每个 source 的 capacity。
- `Single*::compact` 基本为空，因为每个 source 固定一个 slot。

## sort 和 unsorted_since

`batch_sort_by_edge_data(ts)` 会按 edge data 对每个邻接表排序，并设置：

```text
unsorted_since_ = ts
```

`GenericView` 和 typed view 可以用 `unsorted_since_` 判断某些边是否处于“排序后追加”的区域。当前 typed optimization 主要在 `TypedView<T, kMultipleMutable>` 里看到，用于 `foreach_nbr_gt/lt`。

## 底层 container 和文件

CSR 不直接管理 mmap 细节。它通过 `IDataContainer` 存字节块：

```text
IDataContainer
  ├── GetData()
  ├── GetDataSize()
  ├── Resize(bytes)
  ├── Dump(path)
  └── Close()
```

`OpenContainer` 根据 `MemoryLevel` 选择：

```text
kSyncToFile
  snapshot file -> copy/create tmp file -> FileSharedMMap

kInMemory
  existing snapshot file -> FilePrivateMMap
  empty file name        -> AnonMMap

kHugePagePreferred
  AnonHugeMMap, optionally opened from snapshot
```

典型文件名由 `file_names.h` 生成：

```text
oe_{src_label}_{edge_label}_{dst_label}  // outgoing edge csr
ie_{src_label}_{edge_label}_{dst_label}  // incoming edge csr
e_{src_label}_{edge_label}_{dst_label}_data // unbundled edge data table
```

Multiple CSR 文件：

```text
*.meta  // unsorted_since + edge_num, or only edge_num for single csr
*.deg   // degree array
*.cap   // mutable multiple only, capacity array
*.nbr   // neighbor entries
```

Single CSR 文件：

```text
*.meta
*.snbr
```

## 读写路径总结

### 批量加载边

```text
EdgeTable::BatchAddEdges
  -> endpoint OID 通过 IndexerType 转成 lid
  -> out_csr_->resize(src vertex count)
  -> in_csr_->resize(dst vertex count)
  -> bundled:
       batch_add_bundled_edges_impl
       -> csr.batch_put_edges(src_lid, dst_lid, data)
     unbundled:
       table_idx 分配 row id
       csr.batch_put_edges(src_lid, dst_lid, row_id)
       table_->insert(row_id, edge properties)
```

### 单条插入边

```text
EdgeTable::AddEdge
  -> bundled:
       in_csr_->put_generic_edge(dst, src, property)
       out_csr_->put_generic_edge(src, dst, property)
     unbundled:
       row_id = table_idx_++
       in/out csr data = row_id
       table_->insert(row_id, properties)
```

### 遍历边

```text
EdgeTable::get_outgoing_view(ts)
  -> out_csr_->get_generic_view(ts)
  -> view.get_edges(src_lid)
  -> NbrIterator filters invisible/deleted entries
```

incoming 同理，只是从 `in_csr_` 取。

## 易错点

- CSR 不是只存 outgoing，`EdgeTable` 同时存 outgoing 和 incoming 两份。
- `ImmutableCsr` 仍然有 delete/put 实现，只是 neighbor entry 没有 timestamp；删除靠 `neighbor=max`。
- `MutableCsr` 的 `.nbr` dump 时按 `cap[src]` 写，不是按 `degree[src]` 写，因为它要保留每个 source 的预留空间。
- `adj_list_buffer_` 是 pointer array，通常是 runtime 可重建数据，不是核心 snapshot。
- bundled edge data 在 CSR 内；unbundled edge data 在 `Table` 里，CSR 只存 row id。
- `GenericView` 是只读视图，不拥有数据；底层 CSR/container 生命周期必须覆盖 view 使用期。
- single CSR 中 `size()` 是 vertex slot 数，不是 edge 数；`edge_num()` 才是有效边数。

## 后续问题

- `EdgeSchema::is_bundled()` 的具体判定规则值得单独整理，它决定 CSR data 是真实属性还是 row id。
- `search_other_offset_with_cur_offset` 如何在 out/in CSR 之间找对应边 offset，可以继续作为删除和事务语义的专题。
- `Allocator` 在 transaction path 和 WAL ingest 中如何管理生命周期，需要和 update transaction 一起看。
- immutable view 复用 `neighbor=max` 作为 timestamp filtering 的细节，需要结合 `vid_t` 和 `timestamp_t` 类型宽度确认边界是否始终安全。
