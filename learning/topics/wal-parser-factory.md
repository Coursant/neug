# WalParserFactory

## 原始问题

```text
WalParserFactory是什么
```

## 我的理解

- 这次问题没有先给出个人假设；目标是理解 `WalParserFactory` 在 NeuG C++ 数据库启动和 WAL recovery 中的作用。

## 代码事实

- `include/neug/transaction/wal/wal.h`: `IWalParser` 是 WAL parser 接口，提供 `open()`, `close()`, `last_ts()`, `get_insert_wal(ts)`, `get_update_wals()`。
- `include/neug/transaction/wal/wal.h`: `WalParserFactory` 保存 `wal_parser_type -> initializer` 的注册表，并通过 `CreateWalParser(wal_uri)` 创建具体 parser。
- `src/transaction/wal/wal.cc`: `get_wal_uri_scheme(uri)` 从 `xxx://...` 中取 scheme；如果没有 scheme，默认使用 `"file"`。
- `src/transaction/wal/wal.cc`: `WalParserFactory::CreateWalParser(wal_uri)` 根据 scheme 查找 parser initializer；找不到就抛 `NotSupportedException`。
- `src/transaction/wal/local_wal_parser.cc`: `LocalWalParser` 通过静态成员 `registered_` 注册到 `WalParserFactory`，注册 key 是 `"file"`。
- `src/main/neug_db.cc`: `NeugDB::openGraphAndIngestWals()` 在 `graph_.Open(...)` 后调用 `WalParserFactory::Init()` 和 `WalParserFactory::CreateWalParser(wal_dir(work_dir_))`，然后把 parser 交给 `ingestWals(...)`。
- `src/transaction/wal/local_wal_parser.cc`: `LocalWalParser::open()` 会遍历 WAL 目录，打开每个非空 WAL file，用 `mmap` 读入，然后按 `WalHeader` 顺序解析。
- `include/neug/transaction/wal/wal.h`: `WalHeader` 包含 `timestamp`, `type`, `length`。
- `src/transaction/insert_transaction.cc`: `InsertTransaction::Commit()` 写 WAL 时设置 `header->type = 0`。
- `src/transaction/update_transaction.cc`: `UpdateTransaction::Commit()` 写 WAL 时设置 `header->type = 1`。
- `src/transaction/compact_transaction.cc`: `CompactTransaction::Commit()` 写 compaction marker 时设置 `type = 1` 且 `length = 0`。
- `src/main/neug_db.cc`: `NeugDB::ingestWals()` 先按 update WAL 的 timestamp 分段 replay insert WAL，再 replay update WAL；`size == 0` 的 update WAL 表示 compaction marker，会触发 `graph_.Compact(...)`。

## 修正后的结论

`WalParserFactory` 是 WAL parser 的工厂和注册中心。它本身不解析 WAL，只负责根据 WAL URI 的 scheme 创建一个实现了 `IWalParser` 的具体 parser。

当前代码里真正的 parser 是：

```text
scheme: "file"
parser: LocalWalParser
registration: LocalWalParser::registered_
```

所以在默认路径下：

```text
WalParserFactory::CreateWalParser(wal_dir(work_dir_))
-> get_wal_uri_scheme(...)
   -> 没有 "://"，默认 scheme = "file"
-> 查 registry["file"]
-> LocalWalParser::Make(wal_uri)
-> new LocalWalParser(wal_uri)
-> LocalWalParser::open(wal_uri)
```

它在 database open 阶段的位置是：

```text
NeugDB::Open(...)
-> openGraphAndIngestWals()
   -> graph_.Open(...)
   -> WalParserFactory::Init()
   -> WalParserFactory::CreateWalParser(wal_dir(work_dir_))
   -> ingestWals(*wal_parser)
```

`LocalWalParser` 解析 WAL 后提供两类数据：

- `insert_wal_list_`: 按 timestamp 索引的 insert WAL payload，来自 `header->type == 0`。
- `update_wal_list_`: 排序后的 update WAL units，来自 `header->type == 1`。

`NeugDB::ingestWals()` 的 replay 思路是：

```text
from_ts = 1
for each update_wal in sorted update_wals:
  replay [from_ts, update_wal.timestamp) 的 insert WAL
  if update_wal.size == 0:
    graph_.Compact(...)
  else:
    UpdateTransaction::IngestWal(...)
  from_ts = update_wal.timestamp + 1

replay 剩余 insert WAL 到 parser.last_ts()
```

这样做的原因是 insert WAL 可以按 timestamp 直接索引，而 update WAL 需要排序并作为分段边界。compact WAL 是一种特殊 update WAL：`type=1, length=0`。

## WAL 在 WalParserFactory 前后的完整作用

WAL 是 Write-Ahead Log。它的作用是：把一次事务要做的修改先写成 redo log，再让数据库状态进入 committed / recoverable 状态。数据库崩溃后，NeuG 重新打开时可以从 checkpoint + WAL 把图恢复到最近一次已提交状态。

在 NeuG 里，WAL 分两半看：

```text
写入端: Transaction -> IWalWriter -> WAL file
恢复端: WalParserFactory -> IWalParser -> ingestWals -> PropertyGraph
```

`WalParserFactory` 只属于恢复端。

### WAL 写入端

每个事务会把 redo 操作序列化到 `arc_`，前面预留一个 `WalHeader`：

```cpp
struct WalHeader {
  uint32_t timestamp;
  uint8_t type : 1;
  int32_t length : 31;
};
```

含义：

- `timestamp`: 事务时间戳。
- `type`: WAL 类型；`0` 表示 insert WAL，`1` 表示 update WAL 或 compact marker。
- `length`: 后面 redo payload 的长度。

Insert commit 时：

```cpp
header->length = arc_.GetSize() - sizeof(WalHeader);
header->type = 0;
header->timestamp = timestamp_;
logger_.append(...);
```

Update commit 时：

```cpp
header->length = arc_.GetSize() - sizeof(WalHeader);
header->type = 1;
header->timestamp = timestamp_;
logger_.append(...);
```

Compact commit 是特殊 update WAL：

```cpp
header->length = 0;
header->timestamp = timestamp_;
header->type = 1;
logger_.append(...);
```

所以 WAL file 的基本布局是：

```text
[WalHeader][redo payload][WalHeader][redo payload]...
```

写文件的具体实现是 `LocalWalWriter`。它在 `append()` 后会做 `fdatasync`，确保 WAL 落盘。

### WalParserFactory 插入的位置

数据库打开时，流程是：

```text
NeugDB::Open(...)
-> openGraphAndIngestWals()
   -> graph_.Open(...)
   -> WalParserFactory::Init()
   -> WalParserFactory::CreateWalParser(wal_dir(work_dir_))
   -> ingestWals(*wal_parser)
```

也就是说，`graph_.Open(...)` 先打开 checkpoint / snapshot 里的图数据，然后 `WalParserFactory` 创建 parser 读取 WAL，最后 `ingestWals()` 把 WAL 重放到 `graph_` 上。

### LocalWalParser 解析过程

`LocalWalParser::open()` 做这些事：

```text
1. 取 WAL 目录路径。
2. 遍历目录里的 WAL files。
3. 对每个非空文件 open + mmap。
4. 从 mmap 内存里循环读取 WalHeader。
5. timestamp == 0 时停止。
6. 根据 header->type 分流:
   - type == 0 -> insert_wal_list_[timestamp]
   - type == 1 -> update_wal_list_
7. update_wal_list_ 按 timestamp 排序。
```

这里 `timestamp == 0` 是停止条件，因为 `LocalWalWriter` 会预分配/扩展文件，未写入区域是空的。

解析结果有两类：

```cpp
std::vector<WalContentUnit> insert_wal_list_;
std::vector<UpdateWalUnit> update_wal_list_;
```

Insert WAL 用 timestamp 直接索引；Update WAL 收集后排序。

### ingestWals 如何重放 WAL

恢复逻辑是：

```text
from_ts = 1

for each update_wal in update_wals:
  replay [from_ts, update_wal.timestamp) 的 insert WAL

  if update_wal.size == 0:
    graph_.Compact(...)
  else:
    UpdateTransaction::IngestWal(...)

  from_ts = update_wal.timestamp + 1

replay 剩余 insert WAL 到 parser.last_ts()
```

为什么这样设计：

- insert WAL 存在 `insert_wal_list_[timestamp]`。
- update WAL 是一个排序后的列表。
- replay 时要保持 timestamp 顺序。
- update WAL 作为时间边界，把它之前的 insert WAL 先补上。

重放 insert 的逻辑在 `InsertTransaction::IngestWal(...)`：它从 redo payload 里读 `OpType`，然后调用 `graph.AddVertex(...)` 或 `graph.AddEdge(...)`。

重放 update 的逻辑在 `UpdateTransaction::IngestWal(...)`：它根据 `OpType` 执行 `CreateVertexType`, `CreateEdgeType`, `InsertVertex`, `UpdateVertexProp`, `RemoveEdge`, `DeleteVertexType` 等 redo 操作。

完整关系：

```text
正常运行:
Transaction
-> Serialize redo ops into arc_
-> Fill WalHeader
-> IWalWriter::append(...)
-> WAL file
-> apply / commit graph changes

重启恢复:
NeugDB::Open
-> graph_.Open(checkpoint)
-> WalParserFactory::CreateWalParser(wal_dir)
-> LocalWalParser mmap + parse WAL files
-> NeugDB::ingestWals(parser)
-> InsertTransaction::IngestWal(...)
-> UpdateTransaction::IngestWal(...)
-> graph_ restored
```

一句话总结：`WAL` 是 NeuG 的崩溃恢复日志；`WalParserFactory` 是恢复阶段的 parser 创建入口。它把 `wal_dir` 转换成一个具体的 `IWalParser`，当前就是 `LocalWalParser`，然后 `NeugDB::ingestWals()` 用这个 parser 提供的 insert/update WAL 内容把 `graph_` 从 checkpoint 之后恢复到最新提交状态。

## 两个澄清

### 初始化时只有恢复端吗

基本是。`NeugDB::Open()` 阶段走的是恢复端：

```text
WalParserFactory -> IWalParser -> ingestWals -> graph_
```

它负责读已有 WAL 并 replay。进入事务执行/commit 后才走写入端：

```text
Transaction -> IWalWriter -> WAL file
```

也就是说：

```text
DB Open: WalParserFactory 读 WAL 做 recovery
Transaction Commit: WalWriterFactory / IWalWriter 写 WAL 做 durability
```

### 第一次初始化会有已有 WAL 吗

通常没有。全新数据库第一次 `Open()` 仍然会创建 parser 并尝试 recovery，但 WAL 目录为空或没有非空 WAL file，因此：

```text
LocalWalParser 没有解析出 WAL
-> update_wal_list_ 为空
-> last_ts_ = 0
-> ingestWals() 基本不 replay
```

已有 WAL 通常来自之前运行过事务，并且 checkpoint 尚未覆盖或进程崩溃/退出后需要恢复。

## 未验证假设

- 当前笔记基于源码静态阅读，没有运行 recovery 测试。
- `Init()` / `Finalize()` 当前为空；注册依赖静态变量初始化。后续需要确认构建和链接方式是否保证 `LocalWalParser::registered_` 总会生效。

## 后续问题

- WAL 目录中多线程 WAL 文件的 timestamp 是否全局单调，如何由 `VersionManager` 保证？
- `LocalWalParser::open()` 读取目录文件时没有显式排序，是否会影响 recovery 正确性？
