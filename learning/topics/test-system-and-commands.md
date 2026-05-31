# NeuG Test System And Commands

## 原始问题

```text
为我详细解释这个项目的测试
写成笔记，测试指令要详细
```

## 总览

NeuG 的测试分成三套：

```text
1. C++ / GTest / CTest
   tests/
     ├── compiler
     ├── execution
     ├── storage
     ├── transaction
     ├── unittest
     ├── utils
     └── main

2. Python binding / pytest
   tools/python_bind/tests/

3. E2E query tests / pytest
   tests/e2e/
     ├── queries/**/*.test
     ├── test_neug.py
     ├── test_cypher.py
     └── test_kuzu.py
```

C++ 测试主要验证模块内部行为和编译期 plan 结构；Python binding 测试验证用户 API、pybind 生命周期、错误码和服务接口；E2E 测试从 `.test` 查询文件动态生成测试，验证真实 Cypher workload 的执行结果。

## C++ 测试入口

根入口是 `tests/CMakeLists.txt`：

```text
add_subdirectory(compiler)
add_subdirectory(execution)
add_subdirectory(storage)
add_subdirectory(transaction)
add_subdirectory(unittest)
add_subdirectory(utils)
add_subdirectory(main)
```

每个测试 target 都通过根 `CMakeLists.txt` 里的 `add_neug_test(TEST_NAME ...)` 创建：

```text
add_executable(TEST_NAME ...)
target_link_libraries(TEST_NAME neug::GTest neug ...)
add_test(NAME TEST_NAME COMMAND TEST_NAME)
gtest_add_tests(TARGET TEST_NAME TEST_LIST ...)
```

所以一个 C++ test target 通常既可以直接执行：

```bash
./build/tests/storage/storage_test
```

也可以通过 CTest 执行：

```bash
ctest -R storage_test
```

## C++ 测试完整构建和运行

### 从零构建并运行全部 C++ 测试

```bash
cd /home/lcc/cpp/neug
rm -rf build
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=DEBUG -DBUILD_TEST=ON
make -j$(nproc)
ctest --output-on-failure
```

### Release 构建并运行测试

```bash
cd /home/lcc/cpp/neug
rm -rf build
cmake -S . -B build -DCMAKE_BUILD_TYPE=RELEASE -DBUILD_TEST=ON
cmake --build build -j$(nproc)
ctest --test-dir build --output-on-failure
```

### 打开 HTTP server 相关测试

部分测试只有 `BUILD_HTTP_SERVER=ON` 才会加入，例如：

- `tests/storage/test_recovery.cc`
- `tests/storage/test_checkpoint.cc`
- `tests/transaction/*`
- `tests/unittest/test_http_server.cc`
- `tests/unittest/test_db_svc.cc`

构建命令：

```bash
cd /home/lcc/cpp/neug
rm -rf build
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=DEBUG \
  -DBUILD_TEST=ON \
  -DBUILD_HTTP_SERVER=ON \
  -DBUILD_EXECUTABLES=ON
cmake --build build -j$(nproc)
ctest --test-dir build --output-on-failure
```

### 只编译某个 C++ 测试 target

```bash
cd /home/lcc/cpp/neug
cmake --build build --target storage_test -j$(nproc)
cmake --build build --target csr_test -j$(nproc)
cmake --build build --target gopt_test -j$(nproc)
cmake --build build --target utils_test -j$(nproc)
cmake --build build --target execution_test -j$(nproc)
cmake --build build --target test_vertex_table -j$(nproc)
cmake --build build --target edge_table_test -j$(nproc)
```

### 只跑某个 CTest target

```bash
cd /home/lcc/cpp/neug/build
ctest -R storage_test --output-on-failure
ctest -R csr_test --output-on-failure
ctest -R edge_table_test --output-on-failure
ctest -R test_vertex_table --output-on-failure
ctest -R gopt_test --output-on-failure
ctest -R utils_test --output-on-failure
ctest -R execution_test --output-on-failure
```

CTest 的 `-R` 是正则匹配。想看有哪些测试：

```bash
cd /home/lcc/cpp/neug/build
ctest -N
ctest -N -R storage
```

### 直接运行 GTest executable

直接跑 executable 更适合本地调试，因为可以使用 GTest filter。

```bash
cd /home/lcc/cpp/neug/build
./tests/storage/storage_test
./tests/storage/csr_test
./tests/storage/test_vertex_table
./tests/storage/edge_table_test
./tests/compiler/gopt_test
./tests/utils/utils_test
./tests/execution/execution_test
```

列出某个 executable 里的所有 gtest case：

```bash
./tests/storage/test_vertex_table --gtest_list_tests
./tests/storage/csr_test --gtest_list_tests
./tests/compiler/gopt_test --gtest_list_tests
```

只跑某个 test suite：

```bash
./tests/storage/test_vertex_table --gtest_filter='VertexTableTest.*'
```

只跑某个 case：

```bash
./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.VertexTableBasicOps'
```

跑多个 case：

```bash
./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.VertexTableBasicOps:VertexTableTest.VertexTableDumpAndReload'
```

排除某些 case：

```bash
./tests/storage/csr_test \
  --gtest_filter='*:-*Benchmark*'
```

失败时重复跑：

```bash
./tests/storage/edge_table_test \
  --gtest_filter='EdgeTableTest.SomeCase' \
  --gtest_repeat=100 \
  --gtest_break_on_failure
```

## C++ 调试命令

### 打开 glog verbose

```bash
cd /home/lcc/cpp/neug/build
GLOG_v=10 ./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.VertexTableBasicOps'
```

### gdb 调试 C++ 测试

```bash
cd /home/lcc/cpp/neug/build
GLOG_v=10 gdb --args ./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.VertexTableBasicOps'
```

进入 gdb 后：

```text
run
bt
frame 0
info locals
```

### lldb 调试 C++ 测试

```bash
cd /home/lcc/cpp/neug/build
GLOG_v=10 lldb -- ./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.VertexTableBasicOps'
```

进入 lldb 后：

```text
run
bt
frame select 0
frame variable
```

### 使用 CTest verbose 输出

```bash
cd /home/lcc/cpp/neug/build
ctest -R test_vertex_table -V
ctest -R test_vertex_table --output-on-failure
```

## C++ 测试目录说明

### tests/compiler

`tests/compiler` 主要是 planner/optimizer/gopt 的 golden tests。

它读取：

```text
tests/compiler/resources/**/queries
tests/compiler/resources/**/*_logical
tests/compiler/resources/**/*_physical
tests/compiler/resources/**/*_result
tests/compiler/resources/schema/*.yaml
```

核心 fixture 是 `tests/compiler/gopt_test.h`：

```text
GOptTest
  ├── updateSchema()
  ├── updateStats()
  ├── ctx->prepare(query)
  ├── planLogical()
  ├── applyRules()
  ├── planPhysical()
  └── VerifyFactory compares actual plan with golden file
```

这类测试不执行真实查询，而是验证：

- Cypher 到 logical plan 是否稳定。
- optimizer rewrite 是否正确。
- logical plan 到 physical plan JSON 是否符合预期。
- result schema 推导是否符合预期。

常用命令：

```bash
cd /home/lcc/cpp/neug/build
./tests/compiler/gopt_test --gtest_list_tests
./tests/compiler/gopt_test
./tests/compiler/gopt_test --gtest_filter='*Agg*'
./tests/compiler/gopt_test --gtest_filter='*DML*'
```

如果资源路径不在默认位置，可以设置：

```bash
cd /home/lcc/cpp/neug/build
TEST_RESOURCE=/home/lcc/cpp/neug/tests/compiler \
  ./tests/compiler/gopt_test
```

### tests/storage

`tests/storage` 覆盖 storage 主路径：

```text
storage_test
  ├── test_ddl.cc
  ├── test_export.cc
  ├── test_open_graph.cc
  ├── alter_property_test.cc
  ├── test_set_property.cc
  ├── test_memory_level.cc
  └── test_property_graph.cc

csr_test
  ├── test_csr_stream_ops.cc
  ├── test_csr_batch_ops.cc
  ├── test_mutable_csr.cc
  └── test_immutable_csr.cc

test_vertex_table
  └── test_vertex_table.cc

edge_table_test
  └── test_edge_table.cc
```

重点覆盖：

- `Schema` 驱动的 graph open。
- `PropertyGraph` DDL/DML。
- `VertexTable` 的 OID/LID、timestamp、dump/reload。
- `EdgeTable` 的 outgoing/incoming CSR。
- mutable/immutable/single CSR。
- `MemoryLevel::kSyncToFile / kInMemory / kHugePagePreferred` 的持久化行为。

常用命令：

```bash
cd /home/lcc/cpp/neug/build
./tests/storage/storage_test
./tests/storage/csr_test
./tests/storage/test_vertex_table
./tests/storage/edge_table_test
```

只跑 memory level 参数化测试：

```bash
./tests/storage/storage_test \
  --gtest_filter='AllMemoryLevels/MemoryLevelPersistenceTest.DDLAndDMLPersistence/*'
```

只跑 VertexTable：

```bash
./tests/storage/test_vertex_table \
  --gtest_filter='VertexTableTest.*'
```

只跑 CSR：

```bash
./tests/storage/csr_test \
  --gtest_filter='*Immutable*:*Mutable*'
```

### tests/transaction

事务测试只有 `BUILD_HTTP_SERVER=ON` 时构建。

覆盖：

- insert transaction
- update transaction
- add/delete vertex
- add/delete edge
- vertex/edge property update
- DDL commit/abort
- WAL replay
- checkpoint
- ACID anomaly tests：G0、G1A、G1B、G1C、IMP、PMP、OTV、FR、LU、WS

构建：

```bash
cd /home/lcc/cpp/neug
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=DEBUG \
  -DBUILD_TEST=ON \
  -DBUILD_HTTP_SERVER=ON
cmake --build build -j$(nproc)
```

运行：

```bash
cd /home/lcc/cpp/neug/build
./tests/transaction/transaction_test
./tests/transaction/transaction_test --gtest_list_tests
./tests/transaction/transaction_test --gtest_filter='UpdateTransactionTest.*'
./tests/transaction/transaction_test --gtest_filter='NeugDBACIDTest.G1C'
```

### tests/execution

`tests/execution` 主要测 execution value 和 runtime column。

```bash
cd /home/lcc/cpp/neug/build
./tests/execution/execution_test
./tests/execution/execution_test --gtest_list_tests
```

### tests/utils

`tests/utils` 覆盖：

- property table
- type conversion
- CSV/Arrow reader
- exception
- sniffer
- JSON
- bitset
- Arrow/YAML/PB utils
- string view vector
- encoder/decoder

运行：

```bash
cd /home/lcc/cpp/neug/build
./tests/utils/utils_test
./tests/utils/utils_test --gtest_list_tests
./tests/utils/utils_test --gtest_filter='TableTest.*'
./tests/utils/utils_test --gtest_filter='ReaderTest.*'
./tests/utils/utils_test --gtest_filter='ArrowUtilsTest.*'
```

### tests/unittest

`tests/unittest` 是一些分散基础设施测试：

```text
test_extension
schema_test
logical_delete_test
test_indexer
test_mmap_container
test_connection
test_http_server      // BUILD_HTTP_SERVER=ON
test_db_svc           // BUILD_HTTP_SERVER=ON
```

运行：

```bash
cd /home/lcc/cpp/neug/build
./tests/unittest/schema_test
./tests/unittest/logical_delete_test
./tests/unittest/test_indexer
./tests/unittest/test_mmap_container
./tests/unittest/test_connection
```

HTTP server 相关：

```bash
cd /home/lcc/cpp/neug/build
./tests/unittest/test_http_server
./tests/unittest/test_db_svc
```

### tests/main

`tests/main` 目前主要有 query request 序列化 round-trip。

```bash
cd /home/lcc/cpp/neug/build
./tests/main/test_request
```

## Python Binding 测试

Python binding 在 `tools/python_bind` 下。先构建：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
make requirements
make build
```

等价的手动命令：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pip install -r requirements.txt
python3 -m pip install -r requirements_dev.txt
python3 setup.py build_ext
```

构建 wheel：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 setup.py bdist_wheel
python3 -m pip install dist/*.whl
```

### 运行全部 Python binding 测试

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -s tests
```

更详细输出：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -sv tests
```

### 运行单个 Python 测试文件

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -sv tests/test_db_init.py
python3 -m pytest -sv tests/test_db_query.py
python3 -m pytest -sv tests/test_ddl.py
python3 -m pytest -sv tests/test_merge.py
python3 -m pytest -sv tests/test_db_transaction.py
python3 -m pytest -sv tests/test_tp_service.py
```

### 运行单个 pytest case

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -sv tests/test_db_query.py::test_create_schema_basic_types
python3 -m pytest -sv tests/test_db_init.py::test_rw_mode_exclusive
```

### 用关键字过滤 pytest

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -sv tests -k 'schema'
python3 -m pytest -sv tests -k 'transaction'
python3 -m pytest -sv tests -k 'not java'
python3 -m pytest -sv tests/test_db_query.py -k 'type_check'
```

### 失败即停 / 只显示失败

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -m pytest -q tests
python3 -m pytest -x -sv tests/test_db_query.py
python3 -m pytest --maxfail=3 -sv tests
```

### Python 测试调试

打开 C++ glog：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
GLOG_v=10 python3 -m pytest -sv tests/test_db_query.py::test_create_schema_basic_types
```

gdb：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
GLOG_v=10 gdb --args python3 -m pytest -sv tests/test_db_query.py::test_create_schema_basic_types
```

lldb：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
GLOG_v=10 lldb -- python3 -m pytest -sv tests/test_db_query.py::test_create_schema_basic_types
```

## Python Binding 测试覆盖点

`tools/python_bind/tests/test_db_init.py`：

- memory mode open/close
- local DB open
- read-only / read-write mode
- exclusive lock
- invalid path
- config 参数
- permission / disk / corruption / version 类错误

`tools/python_bind/tests/test_db_query.py`：

- DDL
- basic types
- insert type check
- Cypher query
- error code
- service session path

其他文件：

```text
test_ddl.py
test_alter_property.py
test_batch_loading.py
test_db_connection.py
test_db_transaction.py
test_db_import_export.py
test_export.py
test_load.py
test_merge.py
test_to_arrow.py
test_tp_service.py
test_ngcli_basics.py
test_ngcli_commands.py
```

这套测试比 C++ compiler golden tests 更接近用户路径，因为它通过 Python API 调真实 DB。

## E2E 测试

E2E 在 `tests/e2e`。它有 pytest 文件，但真正的 query case 来自：

```text
tests/e2e/queries/**/*.test
tests/e2e/queries/**/*.benchmark
tests/e2e/queries/**/*.cypher
```

`.test` 文件格式：

```text
-DATASET CSV tinysnb

-CASE BasicTests

-LOG Basic1
-STATEMENT MATCH (a:person) RETURN COUNT(a)
---- 1
8

-LOG OnlyCheckSuccess
-STATEMENT MATCH (a:person) RETURN COUNT(a)
---- ok

-LOG ShouldFail
-STATEMENT MATCH (a:person RETURN COUNT(a)
---- error
```

含义：

```text
---- 1      expect exact 1 result row
---- ok     query should succeed, no result validation
---- error  query should throw
-CHECK_ORDER  expected result order matters
-SKIP          skip unless include_skip_tests is enabled
-UNSUPPORTED   always skip
```

### E2E pytest 参数

`tests/e2e/conftest.py` 支持：

```text
--service_uri          service URI, default bolt://localhost:7687
--db_dir               database directory
--read_only            open db read-only
--query_dir            query root, default queries
--dataset              run only one dataset
--test_names           comma-separated query names
--include_skip_tests   include skipped tests
--iterations           benchmark iterations
--rounds               benchmark rounds
--warmup_rounds        benchmark warmup rounds
```

pytest markers：

```text
neug_ap_test       NeuG embedded AP mode
neug_tp_test       NeuG embedded TP/session mode
neug_benchmark     NeuG benchmark
cypher_test        remote Cypher client
cypher_benchmark   remote Cypher benchmark
kuzu_test          Kuzu embedded
kuzu_benchmark     Kuzu benchmark
```

### 直接运行 E2E NeuG AP 测试

前提：`db_dir` 中已经有对应 dataset 的 NeuG 数据库。

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_ap_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only
```

运行某一类 query：

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_ap_test \
  --query_dir=queries/match \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only
```

运行单个 query name：

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_ap_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only \
  --test_names=BasicTests_Basic1
```

包含 skip 测试：

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_ap_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only \
  --include_skip_tests
```

### 运行 E2E NeuG TP/session 测试

`neug_tp_test` 会通过 fixture 启动 embedded DB 的 service endpoint，然后用 `Session.open()` 执行。

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_tp_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only
```

### 运行 E2E benchmark

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_benchmark \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only \
  --iterations=1 \
  --rounds=5 \
  --warmup_rounds=1
```

### 通过脚本运行 E2E

脚本：

```text
tests/e2e/scripts/run_e2e_test.sh
```

参数：

```text
run_e2e_test.sh <test_mode> <dataset_name> [db_dir] [subquery_dirs] [rw_flag]
```

例子：

```bash
cd /home/lcc/cpp/neug/tests/e2e
./scripts/run_e2e_test.sh neug_ap_test tinysnb /tmp/tinysnb basic_test false
```

跑多个 query 子目录：

```bash
cd /home/lcc/cpp/neug/tests/e2e
./scripts/run_e2e_test.sh neug_ap_test tinysnb /tmp/tinysnb basic_test,projection,filter false
```

排除某些 query 子目录：

```bash
cd /home/lcc/cpp/neug/tests/e2e
./scripts/run_e2e_test.sh neug_ap_test tinysnb /tmp/tinysnb '!ldbc,!lsqb' false
```

读写模式运行。脚本会复制一份 `${DB_DIR}_rw`，避免破坏原库：

```bash
cd /home/lcc/cpp/neug/tests/e2e
./scripts/run_e2e_test.sh neug_ap_test tinysnb /tmp/tinysnb ddl true
```

TP/session 模式：

```bash
cd /home/lcc/cpp/neug/tests/e2e
./scripts/run_e2e_test.sh neug_tp_test tinysnb /tmp/tinysnb basic_test false
```

### 远程 Cypher client E2E

如果已有服务监听在 `bolt://localhost:7687`：

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m cypher_test \
  --service_uri=bolt://localhost:7687 \
  --query_dir=queries/basic_test \
  --dataset=tinysnb
```

### Kuzu 对照测试

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m kuzu_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/kuzu_tinysnb \
  --read_only
```

## 一次性本地检查

格式检查：

```bash
cd /home/lcc/cpp/neug
make format-check
```

完整检查，包括构建和测试：

```bash
cd /home/lcc/cpp/neug
make full-check
```

如果只想看脚本选项：

```bash
cd /home/lcc/cpp/neug
./scripts/pre_commit_check.sh --help
```

## 推荐测试策略

### 改 storage / csr / mmap

```bash
cd /home/lcc/cpp/neug
cmake --build build --target storage_test csr_test test_vertex_table edge_table_test -j$(nproc)

cd build
./tests/storage/storage_test
./tests/storage/csr_test
./tests/storage/test_vertex_table
./tests/storage/edge_table_test
```

如果改了 memory level / dump / checkpoint：

```bash
cd /home/lcc/cpp/neug/build
./tests/storage/storage_test \
  --gtest_filter='AllMemoryLevels/MemoryLevelPersistenceTest.DDLAndDMLPersistence/*'
```

### 改 compiler / planner / optimizer

```bash
cd /home/lcc/cpp/neug
cmake --build build --target gopt_test -j$(nproc)

cd build
TEST_RESOURCE=/home/lcc/cpp/neug/tests/compiler \
  ./tests/compiler/gopt_test
```

再补 E2E query：

```bash
cd /home/lcc/cpp/neug/tests/e2e
python3 -m pytest -sv \
  -m neug_ap_test \
  --query_dir=queries/basic_test \
  --dataset=tinysnb \
  --db_dir=/tmp/tinysnb \
  --read_only
```

### 改 Python API / pybind

```bash
cd /home/lcc/cpp/neug/tools/python_bind
make build
python3 -m pytest -sv tests/test_db_init.py
python3 -m pytest -sv tests/test_db_query.py
python3 -m pytest -sv tests/test_db_connection.py
```

### 改 transaction / WAL / service

```bash
cd /home/lcc/cpp/neug
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=DEBUG \
  -DBUILD_TEST=ON \
  -DBUILD_HTTP_SERVER=ON
cmake --build build --target transaction_test -j$(nproc)

cd build
./tests/transaction/transaction_test
```

再跑 Python service 相关：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
make build
python3 -m pytest -sv tests/test_tp_service.py
```

## 常见问题

### 1. `transaction_test` 不存在

原因通常是没有打开 `BUILD_HTTP_SERVER`。

重新配置：

```bash
cd /home/lcc/cpp/neug
cmake -S . -B build -DBUILD_TEST=ON -DBUILD_HTTP_SERVER=ON
cmake --build build -j$(nproc)
```

### 2. Python pytest import 不到 `neug`

先构建 Python binding：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
make requirements
make build
```

如果仍然 import 失败，可以安装 wheel：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 setup.py bdist_wheel
python3 -m pip install --force-reinstall dist/*.whl
```

### 3. E2E 没有 case 被执行

检查三件事：

```bash
cd /home/lcc/cpp/neug/tests/e2e
find queries/basic_test -type f
python3 -m pytest -sv -m neug_ap_test --query_dir=queries/basic_test --dataset=tinysnb --db_dir=/tmp/tinysnb --read_only
```

如果 `.test` 文件的 `-DATASET` 不是 `tinysnb`，`--dataset=tinysnb` 会过滤掉。

### 4. CTest 看不到新测试

重新配置 CMake：

```bash
cd /home/lcc/cpp/neug
cmake -S . -B build -DBUILD_TEST=ON
cmake --build build -j$(nproc)
cd build
ctest -N
```

### 5. 想看失败时更多日志

C++：

```bash
GLOG_v=10 ./tests/storage/storage_test --gtest_filter='SomeSuite.SomeCase'
```

Python：

```bash
GLOG_v=10 python3 -m pytest -sv tests/test_db_query.py::some_test
```

CTest：

```bash
ctest -R storage_test -V
```

## 测试体系的边界

- compiler tests 多数是 golden plan 对比，不验证真实执行结果。
- storage tests 多数直接操作 C++ 对象，覆盖底层行为，但不一定覆盖 Python API。
- Python binding tests 走真实用户 API，但 case 通常比 E2E 更手写、更局部。
- E2E query tests 最接近 workload，但依赖测试数据库准备、dataset 名和 query file 标记。
- transaction tests 默认可能不构建，因为它们依赖 `BUILD_HTTP_SERVER=ON`。

