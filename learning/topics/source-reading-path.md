# NeuG 源码阅读路径

## 原始问题

```text
为我列出一份源码阅读路径，重要程度最高的优先，原创的代码优先
```

## 排序原则

- 原创优先：先读 NeuG self-developed storage/execution/transaction/server/gopt。
- 重要优先：先读能串起主流程的入口，再读局部工具。
- Kuzu-derived compiler frontend 最后读。

## 总路线

```text
NeugDB / Connection / QueryProcessor
-> PropertyGraph / Schema / VertexTable / EdgeTable
-> CSR / mmap container
-> Transaction / WAL / VersionManager
-> Execution operators
-> GOptPlanner / gopt converters
-> Server / Python binding
-> Kuzu-derived parser / binder / common / function / catalog
```

## 1. Core Entry

重点文件：

- `include/neug/main/neug_db.h`
- `src/main/neug_db.cc`
- `include/neug/main/connection.h`
- `src/main/connection.cc`
- `include/neug/main/query_processor.h`
- `src/main/query_processor.cc`

目标：看 query 如何从 public API 进入 compiler、execution、storage。

## 2. Storage 原创核心

重点文件：

- `include/neug/storages/graph/property_graph.h`
- `src/storages/graph/property_graph.cc`
- `include/neug/storages/graph/schema.h`
- `src/storages/graph/schema.cc`
- `include/neug/storages/graph/vertex_table.h`
- `src/storages/graph/vertex_table.cc`
- `include/neug/storages/graph/edge_table.h`
- `src/storages/graph/edge_table.cc`

推荐顺序：

```text
schema -> property_graph -> vertex_table -> edge_table
```

## 3. CSR / Container

重点文件：

- `include/neug/storages/csr/mutable_csr.h`
- `src/storages/csr/mutable_csr.cc`
- `include/neug/storages/csr/immutable_csr.h`
- `src/storages/csr/immutable_csr.cc`
- `include/neug/storages/container/i_container.h`
- `src/storages/container/file_mmap_container.cc`
- `src/storages/container/mmap_container.cc`

目标：理解 edge adjacency、mutable/immutable CSR、mmap file layout。

## 4. Transaction / WAL

重点文件：

- `src/transaction/`
- `include/neug/transaction/`

目标：理解 write path、version、recovery 与 storage 的关系。

## 5. Execution

重点文件：

- `src/execution/`
- `include/neug/execution/`

优先读 scan、filter、project、join、aggregation。目标是看 physical operator 如何消费 plan 并访问 storage。

## 6. GOpt / Planner

重点文件：

- `src/compiler/gopt/`
- `include/neug/compiler/gopt/`
- `src/compiler/planner/gopt_planner.cc`

目标：理解 NeuG 如何把 compiler logical plan 接到自己的 optimizer/planner。

## 7. 外部入口

- `src/server/`: Service Mode。
- `tools/python_bind/`: Python API 和测试入口。

## 8. Kuzu-Derived Frontend

最后读：

- `src/compiler/parser/`
- `src/compiler/binder/`
- `src/compiler/common/`
- `src/compiler/function/`
- `src/compiler/catalog/`
- `src/compiler/optimizer/`

目标：理解 Cypher frontend，但不要把这里误认为 NeuG 最原创的核心。

## 第一条线

建议先从这一组开始：

```text
Connection -> QueryProcessor -> PropertyGraph -> EdgeTable -> execution scan operator
```
