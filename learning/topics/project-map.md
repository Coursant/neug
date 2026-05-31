# NeuG Project Map

## 定位

NeuG 是 C++20 graph database，面向 HTAP workload，支持 Cypher。主要模式：

- Embedded Mode：本地分析、批量加载、复杂查询。
- Service Mode：HTTP service、在线访问、事务场景。

## 顶层目录

```text
include/neug/       Public C++ headers
src/                C++ implementation
tools/python_bind/  Python binding and Python API
doc/                Documentation
tests/              C++ tests/resources
examples/           Example workloads
third_party/        Vendored dependencies
```

## 核心模块

| 模块 | 路径 | 重点 |
| --- | --- | --- |
| compiler | `src/compiler/`, `include/neug/compiler/` | Cypher parser, binder, logical/physical plan；大量 Kuzu-derived |
| execution | `src/execution/`, `include/neug/execution/` | physical operators, runtime pipeline |
| storages | `src/storages/`, `include/neug/storages/` | schema, vertex/edge table, CSR, property columns；NeuG 原创核心 |
| transaction | `src/transaction/`, `include/neug/transaction/` | WAL, version, transaction lifecycle |
| main | `src/main/`, `include/neug/main/` | `NeugDB`, `Connection`, `QueryProcessor` |
| server | `src/server/`, `include/neug/server/` | Service Mode HTTP entry |
| python_bind | `tools/python_bind/` | pybind11 binding and Python tests |

## Query Pipeline

```text
Cypher
-> parser / transformer
-> binder
-> logical plan
-> gopt planner/converter
-> physical plan
-> execution operators
-> storages
```

## 推荐学习顺序

1. `src/main`: 先找 public API 到 query processor 的入口。
2. `src/storages`: 读 schema、property graph、vertex/edge table、CSR。
3. `src/execution`: 读 scan/filter/project/join/aggregation 等算子。
4. `src/compiler/gopt`: 读 NeuG 自己的 planner/optimizer 接入。
5. `src/server` 和 `tools/python_bind`: 看外部使用入口。
6. `src/compiler` 其他部分：最后读 Kuzu-derived compiler frontend。
