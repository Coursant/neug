# Schema Deserialize And OutArchive

## 原始问题

```text
调用learning下工作流，回答问题Schema的Deserialize OutArchive都是什么工作原理，写成笔记
```

## 我的理解

- 这里的 `Schema` 指 storage graph schema：`include/neug/storages/graph/schema.h` 中的 `neug::Schema`。
- 这次问题容易被名字误导：`OutArchive` 不是“输出 archive”，在当前代码里它主要承担“从已有 bytes 中读出对象”的角色。
- 对应地，`InArchive` 在当前代码里反而是“把对象写入内部 byte buffer”的角色。

## 代码事实

- `include/neug/storages/file_names.h`: `schema_path(work_dir)` 返回 `${work_dir}/schema`，这是二进制 schema 文件路径。
- `src/storages/graph/property_graph.cc`: `PropertyGraph::Open(work_dir, memory_level)` 如果发现 schema 文件存在，会调用 `loadSchema(schema_file)`。
- `src/storages/graph/property_graph.cc`: `loadSchema` 打开 `std::ifstream` 后调用 `schema_.Deserialize(in)`。
- `src/storages/graph/property_graph.cc`: `DumpSchema()` 打开 `std::ofstream` 后调用 `schema_.Serialize(out)`，同时还会 dump 一份 YAML。
- `src/storages/graph/schema.cc`: `Schema::Serialize(std::ostream&)` 先写 `vlabel_indexer_` 和 `elabel_indexer_`，再用 `InArchive` 打包 vertex schema、edge schema、description/name/id，最后写 tombstone bitsets。
- `src/storages/graph/schema.cc`: `Schema::Deserialize(std::istream&)` 按完全相同顺序读回：两个 indexer、一个 archive blob、三个 bitset。
- `include/neug/utils/serialization/in_archive.h`: `InArchive` 内部持有 `std::vector<char> buffer_`，`operator<<` 负责 append bytes。
- `include/neug/utils/serialization/out_archive.h`: `OutArchive` 持有 `begin_` / `end_` 指针，`operator>>` 负责从 `begin_` 当前位置取 bytes，并推进 `begin_`。
- `src/storages/graph/schema.cc`: `DataType` 序列化时先写 `DataTypeId`，再写一个 `char has_extra_type_info`，然后按 `List` / `Struct` / `Varchar` 写额外信息。
- `src/storages/graph/schema.cc`: `VertexSchema` / `EdgeSchema` 序列化时，default values 不是直接写 `execution::Value`，而是通过 `get_default_properties()` 转成 `Property` 写入；反序列化后再用 `execution::property_to_value` 转回 `execution::Value`。
- `src/utils/bitset.cc`: `Bitset::Deserialize` 也用同一套 `OutArchive` 读 metadata，然后直接从 stream 读 bitset data bytes。
- `include/neug/utils/id_indexer.h`: `IdIndexer::Deserialize` 先读 key buffer，再用 `OutArchive` 读 hash metadata，最后直接读 indices/distances arrays。

## 工作原理

### 1. Schema 文件的大结构

`Schema::Serialize` 写出的二进制格式可以按顺序理解成：

```text
schema file
  ├── vlabel_indexer_ serialized bytes
  ├── elabel_indexer_ serialized bytes
  ├── size_t archive_size
  ├── archive bytes
  │     ├── uint32_t vertex schema count
  │     ├── VertexSchema[0]
  │     ├── VertexSchema[1]
  │     ├── ...
  │     ├── uint32_t edge schema count
  │     ├── (uint32_t edge_triplet_key, EdgeSchema)[0]
  │     ├── ...
  │     ├── description_
  │     ├── name_
  │     └── id_
  ├── vlabel_tomb_ serialized bytes
  ├── elabel_tomb_ serialized bytes
  └── elabel_triplet_tomb_ serialized bytes
```

所以 `Deserialize` 不是解析 YAML，也不是靠字段名查找；它是严格依赖写入顺序的 binary replay。读写顺序一旦不一致，后面的字段都会错位。

### 2. Schema::Serialize 逐步详解

`Schema::Serialize(std::ostream& os)` 分三段写文件：

```cpp
vlabel_indexer_.Serialize(os);
elabel_indexer_.Serialize(os);
```

第一段直接把 vertex label name -> label id、edge label name -> label id 这两个 indexer 写入 stream。这里不经过 `Schema` 自己的 `InArchive arc`，而是交给 `IdIndexer::Serialize` 自己写。

然后创建一个 `InArchive arc`，把 schema 主体写进内存 buffer：

```cpp
arc << static_cast<uint32_t>(v_schemas_.size());
for (const auto& v_schema : v_schemas_) {
  arc << (*v_schema);
}

arc << static_cast<uint32_t>(e_schemas_.size());
for (const auto& e_pair : e_schemas_) {
  arc << (uint32_t) e_pair.first << (*e_pair.second);
}

arc << description_ << name_ << id_;
```

这里有两个细节：

- `v_schemas_` 是 vector，所以只需要写数量，然后按下标顺序写每个 `VertexSchema`。反序列化时同样按下标恢复到 `v_schemas_[i]`。
- `e_schemas_` 是 `unordered_map<uint32_t, shared_ptr<EdgeSchema>>`，所以每个 edge schema 前面必须额外写 `e_pair.first`。这个 key 是 `generate_edge_label(src, dst, edge)` 生成的 edge triplet key；恢复时靠它重新插入 `e_schemas_`。

最后把这个内存 archive 作为一个整体写入文件：

```cpp
size_t size = arc.GetSize();
os.write(reinterpret_cast<char*>(&size), sizeof(size));
os.write(arc.GetBuffer(), size);
```

也就是先写 `archive_size`，再写 `archive bytes`。这样 `Deserialize` 才知道接下来应该从 stream 读多少 bytes 给 `OutArchive`。

第三段写 tombstone bitsets：

```cpp
vlabel_tomb_.Serialize(os);
elabel_tomb_.Serialize(os);
elabel_triplet_tomb_.Serialize(os);
```

这些 bitset 用来记录 soft deleted vertex label、edge label、edge triplet。它们放在 schema 主体 archive 后面，由 `Bitset::Serialize` 自己负责写 metadata 和 raw bit data。

### 3. Schema::Deserialize 逐步详解

`Schema::Deserialize(std::istream& is)` 的顺序必须和 `Serialize` 完全一致。

第一段读回两个 label indexer：

```cpp
vlabel_indexer_.Deserialize(is);
elabel_indexer_.Deserialize(is);
```

这一步恢复的是 label name 到 label id 的映射。后续 `get_vertex_label_id("Person")`、`get_edge_label_id("KNOWS")` 这类查询依赖它。

第二段读取 schema 主体 archive：

```cpp
size_t arc_size;
is.read(reinterpret_cast<char*>(&arc_size), sizeof(arc_size));
arc.Allocate(arc_size);
is.read(arc.GetBuffer(), arc_size);
```

这里 `OutArchive::Allocate(arc_size)` 分配一块内部 buffer；`is.read` 把刚才 `Serialize` 写出的 archive bytes 整段读进去。之后所有字段都从这段 buffer 中用 `operator>>` 顺序消费。

然后恢复 vertex schemas：

```cpp
uint32_t v_schema_size;
arc >> v_schema_size;
v_schemas_.resize(v_schema_size);
for (uint32_t i = 0; i < v_schema_size; ++i) {
  v_schemas_[i] = std::make_shared<VertexSchema>();
  arc >> (*v_schemas_[i]);
}
```

这里的恢复方式说明：vertex label id 和 `v_schemas_` 下标绑定。第 `i` 个读出的 `VertexSchema` 被放回 `v_schemas_[i]`。

再恢复 edge schemas：

```cpp
uint32_t e_schema_size;
arc >> e_schema_size;
for (uint32_t i = 0; i < e_schema_size; ++i) {
  uint32_t key;
  auto e_schema = std::make_shared<EdgeSchema>();
  arc >> key >> (*e_schema);
  e_schemas_.emplace(key, e_schema);
}
```

edge schema 不靠 vector 下标，而靠序列化时写入的 `key` 回到 `e_schemas_`。这个 key 对应 `(src_label, dst_label, edge_label)` 三元组。

最后读回普通字符串字段和 tombstone bitsets：

```cpp
arc >> description_ >> name_ >> id_;
vlabel_tomb_.Deserialize(is);
elabel_tomb_.Deserialize(is);
elabel_triplet_tomb_.Deserialize(is);
```

注意：当前实现没有在 `arc >> description_ >> name_ >> id_` 后调用 `arc.Empty()`。所以如果 schema 主体 archive 后面还有多余 bytes，当前代码不会主动报错。

### 4. InArchive 负责写入内存 buffer

`InArchive` 的核心是：

```text
operator<<(value)
  -> AddBytes(&value, sizeof(value))
```

对 POD 类型，它直接把内存 bytes append 到 `buffer_`。对 `std::string` / `std::string_view`，先写 `size_t size`，再写字符串内容。对 `std::vector<T>`，先写长度，再写元素；POD vector 可以批量 `AddBytes`，非 POD vector 则逐个元素递归调用 `operator<<`。

在 `Schema::Serialize` 中，`InArchive arc` 只是中间打包器。真正落盘前会先写：

```text
size_t size = arc.GetSize()
os.write(&size, sizeof(size))
os.write(arc.GetBuffer(), size)
```

这个 `size` 让反序列化端知道接下来要读多少 bytes 到 `OutArchive`。

### 5. OutArchive 负责从内存 buffer 顺序读

`OutArchive` 的核心状态是两个指针：

```text
begin_ -> current read cursor
end_   -> buffer end
```

`GetBytes(size)` 返回当前 `begin_` 指向的位置，然后执行：

```text
begin_ += size
```

因此每一次 `archive >> field` 都会消费一段 bytes。`Schema::Deserialize` 的关键步骤是：

```text
read archive_size from stream
arc.Allocate(archive_size)
read archive bytes into arc.GetBuffer()
arc >> v_schema_size
arc >> VertexSchema...
arc >> e_schema_size
arc >> EdgeSchema...
arc >> description_ >> name_ >> id_
```

`OutArchive::Allocate(size)` 会自己持有一段 `std::vector<char>`；`SetSlice(buffer, size)` 则是不拥有外部 buffer，只把 `begin_` / `end_` 指向传入内存。`Schema::Deserialize` 用的是 `Allocate`，WAL ingest 和一些测试常用 `SetSlice`。

### 6. DataType serialize / deserialize 对照

`DataType` 的写入格式是：

```text
DataType
  ├── DataTypeId id
  ├── char has_extra_type_info
  └── extra payload, optional
```

`DataTypeId` 本身不是直接按 enum object 写，而是在 `src/utils/property/types.cc` 中转成 `int32_t`：

```text
DataTypeId -> int32_t -> bytes
bytes -> int32_t -> DataTypeId
```

extra payload 的规则如下：

| DataTypeId | Serialize 写入 | Deserialize 恢复 |
|---|---|---|
| 无 extra info 的类型 | `char 0` | `DataType(id)` |
| `kList` | `char 1` + child `DataType` | `DataType::List(child_type)` |
| `kStruct` | `char 1` + child count + child `DataType` 列表 | `DataType::Struct(child_types)` |
| `kVarchar` | `char 1` + `StringTypeInfo::max_length` | `DataType::Varchar(max_length)` |

所以 `VARCHAR` 的 `max_length` 不是 YAML 专属信息；在 binary schema 中也会通过 `DataType` 的 extra type info 保存。

### 7. VertexSchema serialize / deserialize 对照

`VertexSchema` 的写入顺序是：

```text
label_name
property_types
property_names
primary_keys
default_properties
description
max_num
vprop_soft_deleted
```

对应代码是：

```cpp
archive << v_schema.label_name << v_schema.property_types
        << v_schema.property_names << v_schema.primary_keys
        << v_schema.get_default_properties() << v_schema.description
        << v_schema.max_num << v_schema.vprop_soft_deleted;
```

反序列化按同样顺序读：

```text
label_name
property_types
property_names
primary_keys
deserialized_defaults
description
max_num
vprop_soft_deleted
```

然后把 `std::vector<Property> deserialized_defaults` 转回 `std::vector<execution::Value>`：

```text
Property -> execution::property_to_value(...) -> execution::Value
```

这里说明一个设计边界：schema 文件里保存 default values 时复用 storage 侧 `Property` 的序列化格式；恢复到运行时 schema 对象时再变回 execution 侧 `Value`。

### 8. EdgeSchema serialize / deserialize 对照

`EdgeSchema` 的写入顺序是：

```text
src_label_name
dst_label_name
edge_label_name
description
ie_mutable
oe_mutable
ie_strategy
oe_strategy
properties
property_names
default_properties
eprop_soft_deleted
has_sort_key_for_nbr
sort_key_for_nbr, optional
```

前面字段直接顺序写入。`sort_key_for_nbr` 是 `std::optional<std::string>`，所以先写一个 `uint8_t`：

```text
1 -> 后面继续写 sort_key_for_nbr 字符串
0 -> 后面没有 sort key 字符串
```

反序列化时先读 `has_sort_key_for_nbr`。如果是 1，就再读一个 `std::string` 并赋给 `e_schema.sort_key_for_nbr`；否则设成 `std::nullopt`。

### 9. DataType / VertexSchema / EdgeSchema 怎么恢复

`DataType` 的恢复逻辑是先读主类型：

```text
DataTypeId id
char has_extra_type_info
```

如果没有 extra type info，就构造 `DataType(id)`。如果有：

- `kList`: 递归读 child `DataType`，构造 `DataType::List(child_type)`。
- `kStruct`: 读 child type 数量，再逐个读 child `DataType`。
- `kVarchar`: 读 `max_length`，构造 `DataType::Varchar(max_length)`。

`VertexSchema` 恢复字段包括 label name、property types、property names、primary keys、default properties、description、max vertex num、property tombstone flags。default properties 先读成 `std::vector<Property>`，再转成 `std::vector<execution::Value>`。

`EdgeSchema` 类似，额外包含 src/dst/edge label names、edge strategies、mutable flags、edge property tombstone flags，以及一个 `uint8_t` 标记判断有没有 `sort_key_for_nbr`。

## 修正后的结论

`Schema::Deserialize` 的本质是：从 `${work_dir}/schema` 这个二进制文件里，按 `Schema::Serialize` 写入的顺序，把 indexer、schema 主体、tombstone bitset 逐段读回内存结构。

`OutArchive` 的本质是一个“顺序读 cursor”。它不理解 `Schema` 语义，只提供通用的 `operator>>`：从一段 bytes 里按类型消费数据。`Schema` 的语义由这些重载组合出来：

```text
OutArchive
  -> DataType
  -> VertexSchema / EdgeSchema
  -> Schema::Deserialize
```

命名上要特别注意：

```text
InArchive  = write values into archive buffer
OutArchive = read values out from archive buffer
```

这和直觉中的 input/output 容易反过来。更准确地说，`InArchive` 是“对象进入 archive”，`OutArchive` 是“对象从 archive 出来”。

## 易错点

- `Schema::Deserialize` 不会自动识别字段名；它只按 byte layout 和顺序读取。
- `OutArchive` 没有边界检查；如果 `arc_size` 或字段顺序错了，可能读错或越界。
- 当前 binary 格式没有 schema version 字段；修改序列化字段顺序时需要同时修改 Deserialize，并考虑兼容旧文件。
- `std::string_view` 从 `OutArchive` 读出时指向 archive buffer 内部内存；不能假设它长期独立拥有字符串内容。
- `DataType::Varchar(max_length)` 的 `max_length` 会通过 `DataType` 的 extra type info 序列化保留。

## operator>> 的分派规则

`OutArchive` 不是把所有类型都当作 `tuple` 读取。`arc >> value` 调用哪个函数，由 C++ 的重载决议决定：

```text
POD 类型              -> POD 模板重载，直接读 sizeof(T) 字节
EmptyType             -> 空重载，不消费任何字节
std::string           -> 先读 size_t 长度，再读字符串 bytes
std::string_view      -> 先读 size_t 长度，再让 view 指向 archive 内部 bytes
std::vector<POD>      -> 先读 size_t 元素数量，再批量 memcpy
std::vector<非 POD>   -> 先读 size_t 元素数量，再逐个元素递归 operator>>
std::tuple<...>       -> 按 tuple 字段顺序逐个递归 operator>>
自定义类型            -> 必须有对应的非成员 operator>> 重载
```

所以“任意类型的读取是按照 tuple 吗？”答案是否定的。

只有目标类型本身是 `std::tuple<Args...>` 时，才会使用 tuple 重载：

```cpp
std::tuple<DataType, std::string, size_t> pk;
arc >> pk;
```

这会等价于：

```cpp
arc >> std::get<0>(pk);
arc >> std::get<1>(pk);
arc >> std::get<2>(pk);
```

如果目标类型是 `VertexSchema`、`EdgeSchema`、`DataType` 这类自定义类型，就不会自动拆成 tuple。它们必须自己定义对应重载，例如：

```cpp
OutArchive& operator>>(OutArchive& archive, VertexSchema& v_schema);
OutArchive& operator>>(OutArchive& archive, EdgeSchema& e_schema);
OutArchive& operator>>(OutArchive& out_archive, DataType& type);
```

这些自定义重载内部再按它们自己的字段顺序调用 `archive >> field`。如果字段里有 tuple，例如 `VertexSchema::primary_keys` 是 `std::vector<std::tuple<DataType, std::string, size_t>>`，那么读到具体 tuple 元素时才会进入 tuple 重载。

## 后续问题

- 当前 schema binary format 是否需要显式 version / magic number / checksum，以支持后续兼容性检查？
- `Schema::Deserialize` 是否应在读完 archive 主体后检查 `arc.Empty()`，像 `VertexTimestamp::load_ts` 那样确认没有剩余 bytes？
