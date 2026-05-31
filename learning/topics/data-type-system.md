# NeuG DataType System

## 原始问题

```text
ExtraTypeInfoType DataTypeId DataType的逻辑关系
```

## 我的理解

- 这次问题没有先给出个人假设；目标是澄清 `ExtraTypeInfoType`、`DataTypeId`、`DataType` 三者在 NeuG 类型系统里的分工。

## 代码事实

- `include/neug/common/types.h`: `DataTypeId` 是运行时/存储侧的主类型枚举，包含 `kInt64`, `kVarchar`, `kList`, `kStruct`, `kVertex`, `kEdge`, `kPath` 等。
- `include/neug/common/types.h`: `DataType` 是完整类型对象，内部保存 `DataTypeId id_` 和可选的 `std::shared_ptr<ExtraTypeInfo> type_info_`。
- `include/neug/common/extra_type_info.h`: `ExtraTypeInfoType` 是 extra type info 自己的类别标签，当前包含 `STRING_TYPE_INFO`, `LIST_TYPE_INFO`, `STRUCT_TYPE_INFO` 等。
- `include/neug/common/extra_type_info.h`: `StringTypeInfo`, `ListTypeInfo`, `StructTypeInfo` 都继承自 `ExtraTypeInfo`。
- `src/common/types.cc`: `DataType::Varchar(size_t)` 创建 `DataTypeId::kVarchar + StringTypeInfo`。
- `src/common/types.cc`: `DataType::List(const DataType&)` 创建 `DataTypeId::kList + ListTypeInfo`。
- `src/common/types.cc`: `DataType::Struct(std::vector<DataType>)` 创建 `DataTypeId::kStruct + StructTypeInfo`。
- `src/common/types.cc`: `ListType::GetChildType` 和 `StructType::GetChildTypes` 会先检查主类型 `DataTypeId`，再把 `RawExtraTypeInfo()` cast 成对应的 info 子类。
- `src/common/extra_type_info.cc`: `ListTypeInfo::EqualsInternal` 比较 child type；`StructTypeInfo::EqualsInternal` 比较 child types。
- `include/neug/compiler/common/types/types.h`: compiler 侧另有一套相似模型：`LogicalTypeID` / `LogicalType` / `common::ExtraTypeInfo`，并额外区分 `PhysicalTypeID`。

## 修正后的结论

`DataTypeId` 决定类型的大类，`DataType` 把大类和可选附加信息组合成完整类型，`ExtraTypeInfoType` 只是附加信息对象内部的类别标签。

```text
DataType
  ├── DataTypeId              // 主类型: INT64 / VARCHAR / LIST / STRUCT / VERTEX ...
  └── ExtraTypeInfo(optional) // 附加信息: 字符串长度、list child type、struct child types ...
          └── ExtraTypeInfoType // 附加信息自己的类型标签
```

具体例子：

- `DataType(DataTypeId::kInt64)`: 普通标量类型，不需要 extra info。
- `DataType::Varchar(256)`: 主类型是 `kVarchar`，extra info 是 `StringTypeInfo{max_length=256}`。
- `DataType::List(DataType(DataTypeId::kInt64))`: 主类型是 `kList`，extra info 是 `ListTypeInfo{child_type=INT64}`。
- `DataType::Struct({DataType(DataTypeId::kInt64), DataType(DataTypeId::kVarchar)})`: 主类型是 `kStruct`，extra info 是 `StructTypeInfo{child_types=[INT64, VARCHAR]}`。

可以把它类比成：

```text
DataTypeId = type tag
ExtraTypeInfo = payload / metadata
ExtraTypeInfoType = payload tag
DataType = tagged type object
```

## 未验证假设

- 当前笔记只基于本地源码静态阅读，没有运行测试验证所有类型比较、序列化和 schema round-trip 行为。
- `DataType::operator==` 对 `StringTypeInfo` 的比较逻辑需要后续单独确认：`ExtraTypeInfo::Equals` 对 `STRING_TYPE_INFO` 当前看起来没有调用 `StringTypeInfo::EqualsInternal`。

## 后续问题

- `DataTypeId` 与 compiler 侧 `LogicalTypeID` 在 query pipeline 中如何互相转换？
- `StringTypeInfo::max_length` 在 storage column、Arrow、protobuf、YAML 序列化中是否始终被保留？
- 为什么运行时/存储侧和 compiler 侧存在两套相似类型系统，边界在哪里？
