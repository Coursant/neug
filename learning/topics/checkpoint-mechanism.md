# NeuG Checkpoint Mechanism

## 原始问题

```text
详解 checkpoint 机制，写成笔记
```

## 核心结论

NeuG 的 checkpoint 机制负责把当前内存中的 graph 状态落成可重新打开的持久化快照。

它主要落两类内容：

```text
1. schema metadata
   {work_dir}/schema
   {work_dir}/graph.yaml

2. storage data files
   {work_dir}/checkpoint/
     ├── vertex_map_*.meta / .keys / .indices
     ├── vertex_table_*.col_*
     ├── vertex_tracker_*.ts
     ├── oe_*.meta / .deg / .nbr / .cap / .snbr
     ├── ie_*.meta / .deg / .nbr / .cap / .snbr
     ├── e_*_data.col_*
     └── statistics_*
```

checkpoint 完成后会清理：

```text
{work_dir}/runtime/tmp/
{work_dir}/wal/
```

所以 checkpoint 的语义可以理解为：

```text
当前内存 graph
  -> 可重新 Open 的 checkpoint 文件集
  -> WAL 被截断/清理，因为 checkpoint 已经包含这些变更
```

## 相关源码

```text
src/main/neug_db.cc
include/neug/main/neug_db.h

src/storages/graph/property_graph.cc
include/neug/storages/graph/property_graph.h

src/storages/graph/vertex_table.cc
src/storages/graph/edge_table.cc
src/storages/graph/vertex_timestamp.cc

src/utils/property/table.cc
src/storages/csr/immutable_csr.cc
src/storages/csr/mutable_csr.cc
src/storages/container/mmap_container.cc
src/storages/container/file_mmap_container.cc

src/execution/execute/ops/admin/checkpoint.cc
include/neug/storages/file_names.h
```

测试：

```text
tests/storage/test_checkpoint.cc
tests/storage/test_recovery.cc
tools/python_bind/tests/test_db_query.py
tools/python_bind/tests/test_db_transaction.py
```

## 目录模型

`include/neug/storages/file_names.h` 定义了关键路径：

```text
schema_path(work_dir)          -> {work_dir}/schema
checkpoint_dir(work_dir)       -> {work_dir}/checkpoint/
wal_dir(work_dir)              -> {work_dir}/wal/
runtime_dir(work_dir)          -> {work_dir}/runtime/
tmp_dir(work_dir)              -> {work_dir}/runtime/tmp/
temp_checkpoint_dir(work_dir)  -> {work_dir}/runtime/temp_checkpoint_dir/
allocator_dir(work_dir)        -> {work_dir}/runtime/allocator/
```

注意：schema 文件不在 `checkpoint/` 里，而是在 work dir 根目录：

```text
{work_dir}/schema
{work_dir}/graph.yaml
```

数据文件才在：

```text
{work_dir}/checkpoint/
```

打开数据库时：

```text
PropertyGraph::Open()
  -> 先从 {work_dir}/schema 读 schema
  -> 再按 schema 从 {work_dir}/checkpoint/ 打开 vertex/edge 数据
```

## 入口一：Close 时自动 checkpoint

`NeugDB::Close()` 中：

```text
if config_.checkpoint_on_close && config_.mode == DBMode::READ_WRITE:
  createCheckpoint(false, false)
```

含义：

- 只有 read-write 模式会在 close 时 checkpoint。
- read-only 模式不会 checkpoint。
- `checkpoint_on_close=false` 时 close 不落盘当前变更。
- `reopen=false`，因为 close 后不需要重新打开 graph。

调用链：

```text
NeugDB::Close()
  -> close connections/query_processor/planner
  -> createCheckpoint(force_compaction=false, reopen=false)
  -> graph_.Clear()
  -> unlock file lock
```

测试里 `tools/python_bind/tests/test_db_transaction.py` 覆盖了三个场景：

```text
test_auto_enable_checkpoint
  默认 checkpoint_on_close=True，close 后 reopen 能读到数据

test_manual_enable_checkpoint
  显式 checkpoint_on_close=True，close 后 reopen 能读到数据

test_manual_disable_checkpoint
  checkpoint_on_close=False，close 后 reopen 读不到未 checkpoint 的数据
```

## 入口二：恢复 WAL 后自动 checkpoint

`NeugDB::Open()` 中：

```text
openGraphAndIngestWals()
initPlannerAndQueryProcessor()

if last_ts_ > 0 && config.checkpoint_after_recovery:
  createCheckpoint(true)
```

含义：

- `openGraphAndIngestWals()` 先打开 checkpoint，再回放 WAL。
- 如果确实回放了 WAL，即 `last_ts_ > 0`，并且配置允许，会立即 checkpoint。
- 这里 `force_compaction=true`，目的是把 WAL recovery 后的 tombstone / 增量状态整理进新的 checkpoint。
- 默认 `reopen=true`，所以 checkpoint 后会重新打开 graph，保证后续服务继续可用。

调用链：

```text
NeugDB::Open()
  -> graph_.Open(work_dir, memory_level)
  -> WalParserFactory::CreateWalParser(wal_dir)
  -> ingestWals(parser)
  -> if recovered and checkpoint_after_recovery:
       createCheckpoint(force_compaction=true, reopen=true)
```

`tests/storage/test_recovery.cc` 验证了：

- `checkpoint_on_close=true` 时，重启后数据可见。
- `checkpoint_on_close=false` 时，依赖 WAL recovery 重启后仍可见。
- 后续 read-write open + close 生成 checkpoint 后，再次启动仍可见。

## 入口三：手动 CHECKPOINT 命令

Cypher 支持：

```cypher
CHECKPOINT;
```

编译链路：

```text
Parser transform_transaction
  CHECKPOINT -> TransactionAction::CHECKPOINT

GQueryConvertor::convertCheckpoint()
  -> physical::Checkpoint operator

PlanParser
  kCheckpoint -> "checkpoint"

CheckpointOpr::Eval()
  -> dynamic_cast<StorageUpdateInterface&>(graph_interface)
  -> graph.CreateCheckpoint()

StorageAPUpdateInterface::CreateCheckpoint()
  -> graph_.Dump()
```

这里要注意：手动 `CHECKPOINT;` 走的是 storage interface 的 `CreateCheckpoint()`，当前 AP update 实现直接调用：

```cpp
graph_.Dump();
```

也就是：

```text
手动 CHECKPOINT
  -> PropertyGraph::Dump(reopen=true)
```

它不会经过 `NeugDB::createCheckpoint()`，因此不会执行 `NeugDB::createCheckpoint()` 里的：

```text
mutex lock
compact_on_close / force_compaction 判断
graph_.Compact(...)
```

除非调用路径外层另有并发控制，否则手动 checkpoint 的 compaction 语义和 close/recovery checkpoint 不完全一样。

Python 测试 `test_manual_checkpoint_command` 验证：

```text
checkpoint_on_close=False
写数据
执行 CHECKPOINT;
close
reopen
数据仍然存在
```

## NeugDB::createCheckpoint

核心实现：

```text
NeugDB::createCheckpoint(force_compaction, reopen)
  1. lock mutex_
  2. if config_.compact_on_close || force_compaction:
       graph_.Compact(config_.compact_csr,
                      config_.csr_reserve_ratio,
                      MAX_TIMESTAMP)
  3. graph_.Dump(reopen)
```

参数：

```text
force_compaction
  true  -> 即使 compact_on_close=false，也强制 compact
  false -> 只在 config_.compact_on_close=true 时 compact

reopen
  true  -> Dump 后重新 Open graph
  false -> Dump 后保持 graph cleared，适合 Close()
```

设计含义：

- `mutex_` 保护 checkpoint 过程，避免同一个 `NeugDB` 内部并发 checkpoint。
- compact 和 checkpoint 分离：checkpoint 负责落盘，compact 负责清理 tombstone/删除状态和 CSR。
- close 场景不 reopen，在线 checkpoint / recovery checkpoint 需要 reopen。

## PropertyGraph::Dump 主流程

`PropertyGraph::Dump(bool reopen)` 是 checkpoint 的核心。

伪代码：

```text
PropertyGraph::Dump(reopen)
  1. target_dir = runtime/temp_checkpoint_dir/

  2. 如果 target_dir 存在:
       remove_directory(target_dir)
     否则:
       create_directories(target_dir)

  3. create_directories(target_dir)

  4. dump vertex tables:
       for each vertex label:
         if table not dropped:
           vertex_num = LidNum()
           EnsureCapacity(label, max(4096, vertex_num + vertex_num / 4))
           vertex_capacity[label] = Capacity()
           vertex_table.Dump(target_dir)

  5. dump edge tables:
       for each valid edge triplet:
         e_size = edge_table.PropTableSize()
         new_cap = max(4096, e_size + (e_size + 4) / 5)
         EnsureCapacity(src, dst, edge,
                        vertex_capacity[src],
                        vertex_capacity[dst],
                        new_cap)
         edge_table.Dump(target_dir)

  6. DumpSchema()

  7. copy_directory(target_dir, checkpoint_dir(work_dir), overwrite=true)

  8. remove_directory(target_dir)
  9. remove_directory(runtime/tmp)
  10. remove_directory(wal)

  11. Clear()

  12. if reopen:
        Open(work_dir, memory_level)
```

### 为什么先写 temp_checkpoint_dir

数据不会直接写到 `checkpoint/`，而是先写：

```text
{work_dir}/runtime/temp_checkpoint_dir/
```

然后：

```text
copy_directory(temp_checkpoint_dir, checkpoint_dir, overwrite=true, recursive=true)
```

`copy_directory()` 当前实现：

```text
1. 如果 dst 存在且 overwrite=true:
     remove_all(dst)
2. create_directory(dst)
3. 遍历 src:
     对每个 regular file 创建 hard link 到 dst
```

所以它不是严格的单个目录 rename 原子切换；它会先删除旧 checkpoint 目录，再创建新目录和 hard links。但它避免了“直接在旧 checkpoint 文件上边写边改”的问题：所有新文件先在 temp 目录完整生成。

## DumpSchema

`PropertyGraph::DumpSchema()` 写两个文件：

```text
{work_dir}/schema
{work_dir}/graph.yaml
```

流程：

```text
1. std::ofstream out(schema_path(work_dir))
2. schema_.Serialize(out)
3. schema_.to_yaml()
4. write_yaml_file(..., get_schema_yaml_path())
```

其中：

```text
schema_path(work_dir)      -> {work_dir}/schema
get_schema_yaml_path()     -> {work_dir}/graph.yaml
```

`schema` 是二进制格式，给 `PropertyGraph::Open()` 使用。  
`graph.yaml` 是可读 YAML，偏向调试/导出查看。

## VertexTable Dump

`VertexTable::Dump(target_dir)`：

```text
indexer_->dump(indexer_prefix + "_vertex_map_{label}", target_dir)
table_->dump("vertex_table_{label}", target_dir)
v_ts_->Dump(target_dir + "/vertex_tracker_{label}")
```

落盘文件大致是：

```text
vertex_map_PERSON.meta
vertex_map_PERSON.keys
vertex_map_PERSON.indices

vertex_table_PERSON.col_0
vertex_table_PERSON.col_1.items
vertex_table_PERSON.col_1.data
vertex_table_PERSON.col_1.pos

vertex_tracker_PERSON.ts
```

### VertexTimestamp::Dump

`VertexTimestamp::Dump(prefix)` 会：

```text
1. 对 inserted_vertices_:
     如果不是 DELETED_TIMESTAMP，则 timestamp 重置为 0
2. Compact()
3. dump_ts(prefix + ".ts")
```

设计含义：

- checkpoint 之后，已经持久化的 insert timestamp 可以归零。
- deleted vertices 会通过 timestamp/tracker 状态保留下来，或在 compaction 中被整理。

## Table / Column Dump

`Table::dump(name, snapshot_dir)`：

```text
for each column i:
  column->dump(snapshot_dir + "/" + name + ".col_" + i)
columns_.clear()
```

定长列：

```text
TypedColumn<T>::dump(filename)
  -> buffer_->Dump(filename)
```

普通 container dump：

```text
MMapContainer::Dump(path)
  1. MD5(data_, size_) -> FileHeader
  2. fwrite(FileHeader)
  3. fwrite(payload)
  4. Close()
```

`kSyncToFile` 的 shared mmap dump：

```text
FileSharedMMap::Dump(path)
  1. Sync()
  2. 如果 path == backing path:
       Close()
  3. 否则:
       Close()
       rename(tmp_file, path)
       如果跨文件系统 EXDEV:
         copy_file(tmp_file, path)
         unlink(tmp_file)
```

字符串列会写：

```text
{col}.items
{col}.data
{col}.pos
```

并在 dump 时重新计算 `.items` 和 `.data` 的 MD5 header。

## EdgeTable Dump

`EdgeTable::Dump(checkpoint_dir)`：

```text
in_csr_->dump(ie_{src}_{edge}_{dst}, checkpoint_dir)
out_csr_->dump(oe_{src}_{edge}_{dst}, checkpoint_dir)

if edge properties are unbundled:
  table_->dump(e_{src}_{edge}_{dst}_data, checkpoint_dir)
  write statistics_{src}_{edge}_{dst}
```

bundled edge property：

```text
edge property exists in CSR neighbor.data
no extra edge property Table dump
```

unbundled edge property：

```text
CSR neighbor.data = row_id
actual edge properties in EdgeTable::table_
checkpoint includes e_*_data.col_* and statistics_*
```

## CSR Dump

### ImmutableCsr

`ImmutableCsr<T>::dump(name, checkpoint_dir)`：

```text
dump_meta(checkpoint_dir/name)
degree_list_buffer_->Dump(checkpoint_dir/name + ".deg")
nbr_list_buffer_->Dump(checkpoint_dir/name + ".nbr")
```

文件：

```text
oe_PERSON_KNOWS_PERSON.meta
oe_PERSON_KNOWS_PERSON.deg
oe_PERSON_KNOWS_PERSON.nbr
```

`.meta` 内容：

```text
timestamp_t unsorted_since_
uint64_t edge_num
```

### MutableCsr

`MutableCsr<T>::dump(name, checkpoint_dir)`：

```text
dump_meta(checkpoint_dir/name)
degree_list_->Dump(checkpoint_dir/name + ".deg")
manual write checkpoint_dir/name + ".nbr"
cap_list_->Dump(checkpoint_dir/name + ".cap")
```

`.nbr` 不能简单 dump `nbr_list_`，因为 mutable CSR 的真实 adjacency list 可能已经分散到 allocator 分配的新 buffer 中。它会按每个 vertex 的 `cap[src]` 从 `adj_list_buffer_` 指针数组逐段写出：

```text
for src in vertices:
  data = adj_list_buffer_[src]
  len = cap[src] * sizeof(nbr_t)
  write data to .nbr
  update md5
```

文件：

```text
oe_PERSON_KNOWS_PERSON.meta
oe_PERSON_KNOWS_PERSON.deg
oe_PERSON_KNOWS_PERSON.nbr
oe_PERSON_KNOWS_PERSON.cap
```

### Single CSR

Single CSR 文件更简单：

```text
SingleImmutableCsr:
  {name}.meta
  {name}.snbr

SingleMutableCsr:
  {name}.meta
  {name}.snbr
```

`snbr` 是 single-neighbor array。

## Compact 和 Checkpoint 的关系

checkpoint 不一定等于 compaction。

`NeugDB::createCheckpoint()` 中：

```text
if config_.compact_on_close || force_compaction:
  graph_.Compact(...)
graph_.Dump(...)
```

`PropertyGraph::Compact()` 会：

```text
1. compact_schema()
   - 移除 soft deleted labels/properties
   - 重建 compacted schema/table mapping

2. for each valid vertex table:
     vertex_table.Compact(ts)

3. for each valid edge table:
     edge_table.Compact(compact_csr, sort_key_for_nbr, ts)
```

当前 `VertexTable::Compact()` 主要 compact timestamp tracker，TODO 里还写着未支持 compact unused lid in indexer/table。

`EdgeTable::Compact()` 会视配置 compact CSR、reset timestamp、可能按 sort key 排序等。

几个关键场景：

```text
Close with compact_on_close=true:
  compact + checkpoint

Close with compact_on_close=false:
  checkpoint only

Recovery checkpoint:
  force_compaction=true，因此一定 compact + checkpoint

Manual CHECKPOINT:
  StorageAPUpdateInterface::CreateCheckpoint() -> graph_.Dump()
  通常是 checkpoint only，不经过 NeugDB::createCheckpoint()
```

## WAL 和 Checkpoint 的关系

启动时：

```text
NeugDB::openGraphAndIngestWals()
  -> graph_.Open(checkpoint)
  -> WalParserFactory::CreateWalParser(wal_dir)
  -> ingestWals(parser)
```

WAL parser 会扫描：

```text
{work_dir}/wal/
```

并按 timestamp 回放：

```text
insert WAL
update WAL
```

`ingestWals()` 大致逻辑：

```text
from_ts = 1
for each update_wal sorted by timestamp:
  replay insert WALs before update timestamp
  if update_wal.size == 0:
    graph_.Compact(...)
  else:
    UpdateTransaction::IngestWal(...)
  from_ts = update_ts + 1

replay remaining insert WALs
last_ts_ = parser.last_ts()
```

checkpoint 成功后：

```text
PropertyGraph::Dump()
  -> remove_directory(wal_dir(work_dir_))
```

这表示：

- checkpoint 前：持久状态 = checkpoint + wal。
- checkpoint 后：持久状态 = new checkpoint，wal 可以删除。

如果 `checkpoint_on_close=false`，close 时不会生成新 checkpoint，WAL 是否保留取决于 transaction/service 写入路径；recovery 测试证明服务重启能通过 WAL 恢复未 checkpoint 的变更。

## Open 如何使用 checkpoint

`PropertyGraph::Open(work_dir, memory_level)`：

```text
1. schema_file = {work_dir}/schema
2. checkpoint_dir_path = {work_dir}/checkpoint/

3. if schema exists:
     loadSchema(schema_file)
   else:
     create checkpoint dir

4. 按 schema 创建 vertex_tables_

5. 删除并重建 runtime/tmp

6. 每个 VertexTable.Open()
     从 checkpoint 打开 indexer/table/timestamp
     EnsureCapacity(...)

7. 每个 EdgeTable.Open()
     从 checkpoint 打开 in_csr/out_csr
     如 unbundled，打开 edge property table 和 statistics
     EnsureCapacity(...)
```

所以 checkpoint 必须和 schema 匹配。schema 决定要打开哪些文件；checkpoint 目录提供这些文件的实际数据。

## 手动 CHECKPOINT 的执行效果

Python 测试 `test_checkpoint` 中：

```text
1. 创建 Person 点表
2. 创建 Knows / Likes / Visits 边表
3. 插入点和边
4. 查询确认可见
5. conn.execute("CHECKPOINT;")
6. 再查询确认仍可见
```

`test_manual_checkpoint_command` 中：

```text
checkpoint_on_close=False
写数据
CHECKPOINT;
close
reopen
数据存在
```

这说明手动 checkpoint 可以替代 close checkpoint，把当前状态持久化。

## Checkpoint 测试覆盖范围

`tests/storage/test_checkpoint.cc` 覆盖了 modern graph 的多种 checkpoint 后 reopen 场景：

- basic vertex/edge 读取。
- add vertex property 后 checkpoint/reopen。
- delete vertex property 后 checkpoint/reopen。
- delete vertex 后 checkpoint/reopen。
- add/delete edge property。
- schema 改动和数据改动组合。

`tests/storage/test_recovery.cc` 覆盖：

- service 写入后停止/重启。
- `checkpoint_on_close=true/false` 两种模式。
- WAL recovery 后再 read-write open + checkpoint。

Python binding 测试覆盖：

- 默认 close checkpoint。
- 显式启用/禁用 checkpoint_on_close。
- 手动 `CHECKPOINT;`。
- checkpoint 后点边查询结果。

## 设计原理

### 1. checkpoint 是“全量稳定基线”

WAL 是增量日志。checkpoint 是全量基线。

```text
启动:
  load checkpoint
  replay wal

checkpoint:
  dump full graph
  remove wal
```

这样可以避免 WAL 无限增长，也减少下次启动 recovery 成本。

### 2. schema 和 data 分开

schema 写到：

```text
{work_dir}/schema
{work_dir}/graph.yaml
```

data 写到：

```text
{work_dir}/checkpoint/
```

原因是 schema 是打开 graph 的前置 metadata。打开时先要知道有哪些 vertex/edge label，才能知道应该打开哪些 table/CSR 文件。

### 3. 先写 temp，再替换 checkpoint

数据先 dump 到：

```text
runtime/temp_checkpoint_dir/
```

然后替换：

```text
checkpoint/
```

这样不会在旧 checkpoint 文件上直接改写。当前实现用 `copy_directory()` 删除旧目录后创建 hard links，不是严格目录级原子 rename，但写文件阶段和替换阶段是分离的。

### 4. Dump 后 Clear，可选 Reopen

`PropertyGraph::Dump()` 最后一定：

```text
Clear()
if reopen:
  Open(work_dir_, memory_level_)
```

原因：

- 各个子结构 dump 时会 close/clear 自己的 columns/containers。
- dump 后旧 mmap 指针已经失效。
- 在线 checkpoint 需要 reopen，close checkpoint 不需要。

### 5. 容量预留写进 checkpoint

dump vertex 前：

```text
EnsureCapacity(max(4096, vertex_num + 25%))
```

dump edge 前：

```text
EnsureCapacity(src_vertex_capacity,
               dst_vertex_capacity,
               max(4096, edge_size + 20%))
```

这样 checkpoint 文件不是刚好等于当前数据量，而是带一定余量。下次 open 后可以继续插入，减少频繁 resize。

### 6. mutable CSR 需要特殊 dump

mutable CSR 的 adjacency list 可能已经由 allocator 分散分配；不能只 dump 原始 `nbr_list_`。因此 dump 时从 `adj_list_buffer_` 指针数组逐个 source 写出，并依赖 `cap_list_` 保留每个 source 的容量区间。

## 易错点

- `CHECKPOINT;` 手动命令走 `graph_.Dump()`，不等同于 `NeugDB::createCheckpoint()` 的 compact 路径。
- schema 不在 `checkpoint/` 目录里，而在 `{work_dir}/schema`。
- checkpoint 成功后会删除 `wal/`，因为 WAL 的变更已经合并进 checkpoint。
- `PropertyGraph::Dump()` 会 `Clear()`，如果 `reopen=false`，graph 不再可用，适合 close。
- `copy_directory()` 当前是 hard-link copy，且会先删除旧 checkpoint 目录；不是严格原子 rename。
- `Table::dump()` 会清空 `columns_`。
- `container->Dump()` 会 `Close()`，旧 `GetData()` 指针失效。
- `VertexTimestamp::Dump()` 会把非删除的 inserted timestamp 重置为 0。
- close 时只有 read-write 且 `checkpoint_on_close=true` 才 checkpoint。
- recovery 后如果 `checkpoint_after_recovery=true`，会强制 compact checkpoint。

## 快速调用链图

### Close checkpoint

```text
NeugDB::Close()
  -> createCheckpoint(false, false)
       -> optional graph_.Compact(...)
       -> graph_.Dump(reopen=false)
            -> dump vertex tables
            -> dump edge tables
            -> DumpSchema()
            -> replace checkpoint/
            -> remove tmp/
            -> remove wal/
            -> Clear()
```

### Recovery checkpoint

```text
NeugDB::Open()
  -> graph_.Open(checkpoint)
  -> ingestWals()
  -> if last_ts_ > 0 && checkpoint_after_recovery:
       createCheckpoint(true, true)
          -> graph_.Compact(...)
          -> graph_.Dump(reopen=true)
               -> Clear()
               -> Open(checkpoint)
```

### Manual checkpoint

```text
CHECKPOINT;
  -> TransactionAction::CHECKPOINT
  -> physical Checkpoint
  -> CheckpointOpr::Eval()
  -> StorageUpdateInterface::CreateCheckpoint()
  -> StorageAPUpdateInterface::CreateCheckpoint()
  -> graph_.Dump(reopen=true)
```

