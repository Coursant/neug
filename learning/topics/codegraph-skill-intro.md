# CodeGraph / CodeScope Skill

## 原始问题

```text
分析skills codegraph 向我介绍
使用codegraph分析的skill
```

## 一句话理解

CodeGraph 的目标是把源码索引成 code knowledge graph，再结合 semantic embeddings，用 graph query 和 semantic search 回答复杂源码问题。

```text
source code
-> File / Function / Class / Module graph
-> CALLS / IMPORTS / INHERITS / BELONGS_TO edges
-> optional Commit / PR evolution graph
-> function-level vector index
```

## 适合的问题

- 谁调用了某个函数，某个函数调用了谁。
- 哪些函数 fan-in/fan-out 高，是 architecture hotspot。
- 某个功能可能在哪些函数实现。
- 哪些文件或函数经常一起改动。
- 某个 bug report 可能关联哪些 root cause functions。
- 某个 PR 的 blast radius 和 review risk。

## 核心 schema

常见 node：

- `File`
- `Function`
- `Class`
- `Module`
- `Commit`
- `PR`
- `AUTHOR`

常见 edge：

- `CALLS`
- `DEFINES_FUNC`
- `DEFINES_CLASS`
- `HAS_METHOD`
- `IMPORTS`
- `BELONGS_TO`
- `INHERITS`
- `COMPOSES`
- `AGGREGATES`
- `USES`
- `MODIFIES`
- `TOUCHES`
- `CHANGES`
- `OPENS`

## 常用查询形态

Callers:

```cypher
MATCH (caller:Function)-[:CALLS]->(f:Function {name: 'func_name'})
WHERE f.is_historical = 0
RETURN caller.name, caller.file_path
LIMIT 30
```

Callees:

```cypher
MATCH (f:Function {name: 'func_name'})-[:CALLS]->(callee:Function)
WHERE f.is_historical = 0
RETURN callee.name, callee.file_path
LIMIT 50
```

Hotspots:

```cypher
MATCH (caller:Function)-[:CALLS]->(f:Function)-[:CALLS]->(callee:Function)
WHERE f.is_historical = 0
WITH f, count(DISTINCT caller) AS fi, count(DISTINCT callee) AS fo
RETURN f.name, f.file_path, fi, fo, fi * fo AS risk
ORDER BY risk DESC
LIMIT 20
```

## 与 NeuG 学习的关系

如果索引可用，它可以辅助：

- trace `QueryProcessor` 到 execution/storage 的调用链。
- 找 `EdgeTable`、CSR、scan operator 的 callers/callees。
- 识别 storage/execution/compiler 之间的 coupling。
- 分析 PR 或改动影响面。

但它不能替代源码阅读。结论仍需回到具体 source path 验证。

## 当前环境状态：2026-05-23

已确认：

- 工作区存在 `./.codegraph`。
- `codegraph` CLI 位于 `.venv/bin/codegraph`。
- `codegraph-ai 0.3.0` 已安装。
- 当前 `.codegraph/graph.db` schema 为空，`vertex_types` 和 `edge_types` 都为空。

尝试结果：

- `codegraph status --db .codegraph` 卡在 embedding model 加载。
- `all-MiniLM-L6-v2` 未缓存，HuggingFace / hf-mirror 下载失败。
- 尝试 graph-only monkeypatch 后，又遇到 `tree-sitter-language-pack` API 与 `codegraph-ai 0.3.0` adapter 不兼容。
- 当前 C adapter 只支持 `.c` / `.h`，不直接支持 NeuG 的 `.cc` / `.cpp`。

## 结论

当前环境不能可靠使用 CodeGraph 分析 NeuG C++ 源码。

## 后续修复方向

- 准备本地 embedding model。
- 固定或修复 `tree-sitter-language-pack` 兼容版本。
- 扩展 C/C++ adapter，使其支持 `.cc` / `.cpp`。
- 重新生成 `.codegraph/graph.db`，确认有 `Function` / `File` nodes 后再使用。
