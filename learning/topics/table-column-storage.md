# NeuG Table Column Storage

## 原始问题

```text
Table的列存储是怎么样的
```

## 我的理解

- 这次问题没有先给出个人假设；目标是理解 NeuG 的 `Table` 是否是真正的行存储，还是由多个 column 组成的列式存储。
- 这里的 `Table` 指 `include/neug/utils/property/table.h` 中的 `neug::Table`，不是 compiler/catalog 里的 table 概念。

## 代码事实

- `include/neug/utils/property/table.h`: `Table` 内部保存 `col_id_map_`, `col_names_`, `columns_`, `col_deleted_`, `name_`, `work_dir_`, `snapshot_dir_`。
- `include/neug/utils/property/table.h`: `columns_` 是 `std::vector<std::shared_ptr<ColumnBase>>`，每个元素是一列。
- `src/utils/property/table.cc`: `Table::initColumns` 根据 schema 中的列名和 `DataType` 调用 `CreateColumn`，为每列创建一个 `ColumnBase` 子类实例。
- `src/utils/property/column.cc`: `CreateColumn` 对非字符串标量类型创建 `TypedColumn<T>`，对 `DataTypeId::kVarchar` 创建 `StringColumn`，也就是 `TypedColumn<std::string_view>`。
- `include/neug/utils/property/column.h`: `ColumnBase` 是列接口，定义 `open`, `open_in_memory`, `open_with_hugepages`, `dump`, `resize`, `set_any`, `get_prop`, `ingest` 等方法。
- `include/neug/utils/property/column.h`: 普通 `TypedColumn<T>` 只有一个 `std::unique_ptr<IDataContainer> buffer_` 和 `size_`。
- `include/neug/utils/property/column.h`: 普通 `TypedColumn<T>::resize(size)` 把底层 buffer 调整到 `size * sizeof(T)`。
- `include/neug/utils/property/column.h`: 普通 `TypedColumn<T>::set_value(index, val)` 通过 `reinterpret_cast<T*>(buffer_->GetData())[index] = val` 写入。
- `include/neug/utils/property/column.h`: 普通 `TypedColumn<T>::get_view(index)` 通过同样的数组视图读取第 `index` 个值。
- `include/neug/utils/property/column.h`: `StringColumn` 使用两个 buffer：`items_buffer_` 保存每行字符串的 `{offset, length}`，`data_buffer_` 保存连续字符串内容。
- `include/neug/utils/property/column.h`: `string_item` 用 bit field 表示，`offset` 48 bit，`length` 16 bit。
- `include/neug/utils/property/column.h`: `StringColumn::set_value` 总是把新字符串 append 到 `data_buffer_` 的尾部，然后更新对应行的 `string_item`；旧字符串空间不会立即回收。
- `src/utils/property/table.cc`: `Table::insert(index, values, insert_safe)` 会逐列调用 `columns_[i]->set_any(index, values[i], insert_safe)`。
- `src/utils/property/table.cc`: `Table::get_row(row_id)` 是从每一列取 `get_prop(row_id)` 临时组装成一行。
- `src/utils/property/table.cc`: `Table::open` 以 `{table_name}.col_{i}` 为每列打开单独的 column 文件。
- `include/neug/storages/graph/vertex_table.h`: `VertexTable::GetPropertyColumn` 对主键列特殊处理：主键来自 `indexer_->get_keys()`，普通属性才来自 `table_->get_column(prop)`。
- `src/storages/graph/vertex_table.cc`: `VertexTable::AddVertex` 先通过 indexer 写入主键，再调用 `table_->insert(vid, props, insert_safe)` 写入普通属性列。

## 修正后的结论

NeuG 的 `Table` 是列式组织，不是把每一行打包成一个 row object 存储。

整体结构可以理解为：

```text
Table
  ├── col_names_              // column id -> column name
  ├── col_id_map_             // column name -> column id
  └── columns_
        ├── ColumnBase col_0  // property column 0
        ├── ColumnBase col_1  // property column 1
        └── ColumnBase col_2  // property column 2
```

每一列自己管理底层数据 buffer。同一个 `row_id` / `vid` 在所有列中对应同一个下标。

```text
row_id = vid = 10

columns_[0][10] = property 0 of vertex 10
columns_[1][10] = property 1 of vertex 10
columns_[2][10] = property 2 of vertex 10
```

### 定长列

对 `int32`, `int64`, `uint64`, `float`, `double`, `Date`, `DateTime`, `Interval` 等定长类型，`TypedColumn<T>` 的物理布局接近一个连续数组：

```text
buffer_
  ├── T[0]
  ├── T[1]
  ├── T[2]
  └── ...
```

源码上的关键关系是：

```cpp
buffer size = row_count * sizeof(T)
value[index] = reinterpret_cast<T*>(buffer_->GetData())[index]
```

所以定长列按列连续存储，适合按列扫描，也方便 mmap 或 hugepage 管理。

### 字符串列

字符串列不是把 `std::string` 对象数组直接放进 buffer。`StringColumn = TypedColumn<std::string_view>`，内部拆成：

```text
items_buffer_:
  row 0 -> {offset, length}
  row 1 -> {offset, length}
  row 2 -> {offset, length}

data_buffer_:
  raw bytes of string values
```

读取第 `idx` 行时：

```text
item = items_buffer_[idx]
return string_view(data_buffer_ + item.offset, item.length)
```

写入字符串时：

```text
offset = pos_.fetch_add(string_length)
items_buffer_[idx] = {offset, string_length}
memcpy(data_buffer_ + offset, string bytes)
```

如果同一个 idx 被更新，新的字符串会 append 到尾部，旧内容不立即回收。源码注释也说明 previous value should be handled by garbage collection or compaction。

### 文件布局

`Table::open` 给每列使用独立文件名：

```text
{table_name}.col_0
{table_name}.col_1
{table_name}.col_2
```

普通定长列就是一个 column 文件。字符串列会进一步拆成：

```text
{table_name}.col_i.items
{table_name}.col_i.data
{table_name}.col_i.pos
```

其中：

- `.items`: 每行的 `{offset, length}`。
- `.data`: 连续字符串字节区。
- `.pos`: 当前 append 到 data buffer 的位置。

### open / memory level

`Table` 根据 memory level 打开每一列：

```text
kSyncToFile
  -> column.open(snapshot_dir/name.col_i, tmp_dir/name.col_i)

kInMemory
  -> column.open_in_memory(snapshot_dir/name.col_i)

kHugePagePreferred
  -> column.open_with_hugepages(snapshot_dir/name.col_i)
```

底层由 `OpenContainer` 创建 `IDataContainer`。因此 `Table` 不直接关心 mmap、匿名内存、hugepage 等细节；这些由 column 的 container 处理。

### 插入和扩容

`Table::insert(index, values, insert_safe)` 不追加 row object，而是逐列写入：

```text
for each column i:
  columns_[i]->set_any(index, values[i], insert_safe)
```

`Table::resize(row_num)` 也是逐列扩容：

```text
for each column:
  column->resize(row_num)
```

定长列扩容就是调整 `size * sizeof(T)`；字符串列扩容时还要估算或预留 `data_buffer_` 的字节空间。

### 顶点主键不在普通属性 Table 中

在 `VertexTable` 中，主键列和普通属性列分开：

```text
primary key
  -> indexer_->get_keys()

ordinary properties
  -> table_->columns_
```

所以读取某个 vertex property 时，如果 property name 是 primary key，`GetPropertyColumn` 返回的是 indexer 的 key column；否则才返回 `table_` 里的普通属性列。

这和上一篇 indexer 笔记能接上：

```text
OID / primary key -> IndexerType / LFIndexer
ordinary properties -> Table / ColumnBase
```

## 对比总结

| 层级 | 职责 | 关键结构 |
|---|---|---|
| `Table` | 管理列名、列 id、列对象数组 | `col_id_map_`, `col_names_`, `columns_` |
| `ColumnBase` | 统一列接口 | `open`, `resize`, `set_any`, `get_prop`, `dump` |
| `TypedColumn<T>` | 定长类型列 | single `IDataContainer` as `T[]` |
| `StringColumn` | varchar 列 | `items_buffer_` + `data_buffer_` + `pos_` |
| `VertexTable` | 把 vertex id 对齐到属性列下标 | `indexer_` for pk, `table_` for properties |

## 易错点

- `Table::get_row` 会组装一行返回，但底层不是行存储。
- `Table::size()` 直接用第一列的 size；隐含要求所有列行数一致。
- 字符串列返回的是 `std::string_view`，视图指向 column 的 `data_buffer_`。
- 更新字符串不会原地覆盖旧字符串，而是 append 新内容并更新 `{offset, length}`。
- 顶点主键不是普通 `table_` 属性列；主键存在 `LFIndexer` 的 `keys_` 中。
- `insert_safe` 对定长列基本被忽略；对字符串列可能触发 `data_buffer_` resize。

## 后续问题

- `StringColumn::dump` 会压缩重复 item 并重写 data/items 文件；它和后续 compaction 的边界可以单独梳理。
- `Table::delete_column` 当前只 close/reset 内存结构，是否删除底层 column 文件需要继续确认。
- `EdgeTable` 中边属性是否也复用同一套 `Table`/`ColumnBase` 存储路径，可以接着沿 `EdgeTable::AddEdge` 和 batch ingest 阅读。
