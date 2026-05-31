# NeuG 与 Kuzu 对比笔记

## 原始问题

```text
帮我比较neug与kuzu之间相同的部分
```

## 代码事实

- `README.md` 说明 NeuG 的 C++ Cypher compiler is adapted from Kuzu。
- `THIRD_PARTY_LICENSES.md` 说明 NeuG incorporates source code from Kuzu under MIT License。
- Kuzu-derived 范围包括 `antlr4/`, `binder/`, `parser/`, `catalog/`, `common/`, `function/`, `main/`, `graph/`, `planner`, `optimizer`。
- 多个文件头部保留 “originally from Kuzu” 和 “Modified ... to support Neug-specific features”。

## 主要相同部分

NeuG 与 Kuzu 最相同的是 C++ Cypher compiler stack 和 common data structures：

- `parser/`, `antlr4/`: OpenCypher grammar, generated parser, AST transformer。
- `binder/`: variable / label / property / function binding。
- `common/`: `ValueVector`, `DataChunk`, `SelectionVector`, `NullMask`, serializer, type utilities。
- `function/`: scalar / aggregate / list / path / string / date function framework。
- `catalog/`: metadata abstractions，NeuG 做了适配。
- `optimizer/`, `planner`: optimizer framework 和部分 rules。
- `compiler/graph`: compiler-side graph abstraction。

## 量化观察

上一轮粗粒度比较使用 Kuzu `HEAD=89f0263cc7a1fd9c396d2c4953747a013556a7f9`。

```text
带 Kuzu 来源声明的 C/C++ 文件: 621
能映射到当前 Kuzu 同路径文件: 590
当前 Kuzu 中同路径不存在: 31
归一化后近似一致 >=99.5%: 227
高度相似 90%-99.5%: 217
较高相似 75%-90%: 79
```

这些数字只用于源码阅读路线判断，不作为法律或版权结论。

## 主要不同部分

一句话：

```text
NeuG 最像 Kuzu 的是 frontend compiler + common vector/data structures；
NeuG 最不像 Kuzu 的是 storage, execution, transaction, service mode, and graph optimizer integration.
```

NeuG 更值得优先读的原创区域：

- `src/storages/`, `include/neug/storages/`
- `src/execution/`, `include/neug/execution/`
- `src/transaction/`, `include/neug/transaction/`
- `src/server/`, `include/neug/server/`
- `src/main/`, `include/neug/main/`
- `src/compiler/gopt/`, `include/neug/compiler/gopt/`
- `src/compiler/planner/gopt_planner.cc`

## 阅读建议

先读 NeuG 原创 backend，再读 Kuzu-derived frontend：

```text
main -> storages -> transaction -> execution -> gopt/server/python_bind -> compiler frontend
```
