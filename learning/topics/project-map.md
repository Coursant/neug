# NeuG Project Map

这是学习 NeuG 的第一版项目地图。后续阅读源码时，需要不断用具体类、函数和调用链补充它。

## 项目定位

NeuG 是一个 C++20 图数据库，面向 HTAP 工作负载，支持 Cypher 查询。它有两种使用模式：

- Embedded Mode：偏分析场景，例如批量加载、复杂模式匹配和图分析。
- Service Mode：偏事务和在线服务场景，例如并发访问和实时应用。

## 顶层结构

```text
include/neug/       Public C++ headers
src/                C++ implementation
tools/python_bind/  Python binding and Python API
doc/                Official documentation source
tests/              C++ tests and resources
examples/           Benchmark and example workloads
third_party/        Vendored dependencies
```

## 核心源码模块

### compiler

路径：

```text
src/compiler/
```

职责：

- 解析 Cypher。
- 绑定名称、标签、属性和表达式。
- 生成 logical plan。
- 通过 optimizer/planner 相关组件转换到 physical plan。

学习重点：

- parser、binder、logical operator、physical operator 的边界。
- AST/statement/expression 如何流转。
- 错误信息如何从编译阶段返回给上层。

### execution

路径：

```text
src/execution/
```

职责：

- 执行 physical plan。
- 实现 scan、filter、project、join、aggregation 等物理算子。
- 产生查询结果。

学习重点：

- iterator/vectorized execution 模型是否存在。
- 算子之间如何传递数据。
- runtime value 和 result set 的表示。

### storages

路径：

```text
src/storages/
include/neug/storages/
```

职责：

- 管理图数据存储。
- 处理 schema、property column、CSR 类图结构。
- 为查询执行提供点边扫描能力。

学习重点：

- 点、边、label、property 的物理布局。
- CSR 在图数据库中的优势和限制。
- schema 和数据文件之间的映射。

### main

路径：

```text
src/main/
```

职责：

- 实现数据库核心入口。
- 管理 database、connection、query processor 等对象。
- 串联加载、连接、查询执行等用户可见流程。

学习重点：

- public API 到内部查询管线的调用链。
- Embedded Mode 如何启动和使用。
- 生命周期、资源管理和错误返回。

### server

路径：

```text
src/server/
```

职责：

- 提供 Service Mode 的 HTTP 服务。
- 将远程请求转换为数据库操作。

学习重点：

- 服务模式和嵌入模式共享了哪些内部组件。
- 并发、连接、会话和错误响应如何处理。

### Python binding

路径：

```text
tools/python_bind/
```

职责：

- 通过 pybind11 暴露 C++ 能力。
- 提供 Python 层的 `Database`、`Connection`、`Session` 等 API。
- 承载 Python 测试和开发入口。

学习重点：

- pybind11 如何映射 C++ 类和生命周期。
- Python API 与 C++ API 的差异。
- Python 测试如何覆盖核心查询流程。

## 查询管线

第一版抽象：

```text
Cypher
  -> ANTLR Parser
  -> Binder
  -> Logical Plan
  -> gopt Converter / Optimizer
  -> Physical Plan
  -> Execution
  -> QueryResult
```

后续需要补充：

- 每一步对应的入口类和关键函数。
- 每一步的数据结构。
- 错误在管线中的传播方式。
- Python `conn.execute(...)` 如何进入这条管线。

## 推荐学习顺序

1. 从 `tools/python_bind/tests/` 找最小查询用例。
2. 追踪 Python `Connection.execute` 到 C++ connection/query processor。
3. 阅读 query processor 如何调用 compiler。
4. 阅读 logical plan 和 physical plan 的主要节点。
5. 阅读 execution operator 如何产出结果。
6. 回头看 storage/schema 如何支撑 scan。
7. 最后看 server mode 如何复用核心能力。

## 当前待补充

- `Database` / `Connection` / `QueryResult` 的 C++ 类图。
- Python binding 到 C++ 的完整调用链。
- 一个 `MATCH ... RETURN ...` 查询的端到端源码路径。
- CSR storage 的文件格式和内存布局。
