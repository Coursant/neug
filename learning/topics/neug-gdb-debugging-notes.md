# NeuG GDB 调试笔记

本文记录在 NeuG 项目中使用 GDB 调试 C++ 测试、Python 绑定、服务进程和 core dump 的常用流程。命令默认从仓库根目录 `/home/lcc/cpp/neug` 执行。

## 1. 先构建 Debug 版本

GDB 最怕两件事：没有调试符号、优化级别太高。NeuG 根 `CMakeLists.txt` 在 `CMAKE_BUILD_TYPE=DEBUG` 时会追加 `-O0 -g`，所以调试时优先使用单独的 Debug 构建目录。

```bash
cd /home/lcc/cpp/neug
cmake -S . -B build-debug \
  -DCMAKE_BUILD_TYPE=DEBUG \
  -DBUILD_TEST=ON \
  -DBUILD_EXECUTABLES=ON \
  -DBUILD_HTTP_SERVER=ON \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DOPTIMIZE_FOR_HOST=OFF
cmake --build build-debug -j"$(nproc)"
```

如果只调 Python 绑定，走项目推荐路径：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
make requirements
BUILD_TYPE=DEBUG BUILD_TEST=ON DEBUG=ON make build
```

几个变量的作用：

- `BUILD_TYPE=DEBUG`：让 Python 绑定构建 CMake Debug 版本。
- `DEBUG=ON`：运行 Python 绑定时打开 C++ glog 到 stderr。
- `GLOG_v=10`：打开更详细的 `VLOG(n)`。
- `BUILD_TEST=ON`：让 Python 绑定构建时也生成 C++ 测试目标。
- `BUILD_HTTP_SERVER=ON`：需要调 `rt_server` 或 HTTP service 时必须打开。

## 2. GDB 基础配置

进入 GDB 后建议先输入：

```gdb
set pagination off
set print pretty on
set breakpoint pending on
handle SIGPIPE nostop noprint pass
```

常用命令：

```gdb
run                         # 启动程序
bt                          # 当前线程调用栈
bt full                     # 调用栈 + 局部变量
info threads                # 查看线程
thread apply all bt         # 所有线程调用栈
frame 3                     # 切到第 3 层栈帧
list                        # 看当前源码附近
print query_string          # 打印变量
ptype query_result          # 查看类型
next                        # 单步越过函数
step                        # 单步进入函数
finish                      # 跑完当前函数并返回
continue                    # 继续运行到下一个断点
watch variable              # 写监视点
catch throw                 # 捕获 C++ 异常抛出点
```

## 3. 调试 C++ gtest

先列出测试：

```bash
ctest --test-dir build-debug -N
```

直接用 GDB 启动某个测试二进制。例如 `test_connection` 在 `tests/unittest/CMakeLists.txt` 中由 `add_neug_test(test_connection test_connection.cc)` 生成：

```bash
cd /home/lcc/cpp/neug/build-debug
gdb --args ./tests/unittest/test_connection --gtest_filter='*Connection*'
```

在 GDB 中常设的查询链路断点：

```gdb
b neug::Connection::Query
b neug::QueryProcessor::execute
b neug::QueryProcessor::execute_internal
run
```

如果调存储相关测试：

```bash
cd /home/lcc/cpp/neug/build-debug
gdb --args ./tests/storage/storage_test --gtest_filter='*DDL*'
```

常见存储/事务断点：

```gdb
b neug::NeugDB::Open
b neug::ConnectionManager::CreateConnection
b neug::InsertTransaction::AddVertex
b neug::InsertTransaction::AddEdge
b neug::UpdateTransaction::Commit
b neug::ReadTransaction::Commit
```

断在 `Connection::Query` 后，建议先看：

```gdb
print query_string
print access_mode
bt
```

这可以确认查询字符串从测试进入引擎时是否已经正确。

## 4. 调试 Python / pybind11 路径

Python API 的调用大致是：

```text
tools/python_bind/neug/connection.py:Connection.execute()
  -> neug::PyConnection::execute()
  -> neug::Connection::Query()
  -> neug::QueryProcessor::execute()
```

先确认 Python 实际加载的是刚构建的 `.so`：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 -c "import neug_py_bind; print(neug_py_bind.__file__)"
```

用 GDB 跑单个 pytest：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
DEBUG=ON GLOG_v=10 gdb --args python3 -m pytest -s tests/test_db_query.py -k 'test_name'
```

GDB 里设置 pending breakpoint，因为 `neug_py_bind` 是 Python import 后才动态加载：

```gdb
set breakpoint pending on
b neug::PyConnection::execute
b neug::Connection::Query
b neug::QueryProcessor::execute
run
```

如果断点没绑定，先让 Python 停在入口，再检查动态库：

```gdb
start
info sharedlibrary neug_py_bind
sharedlibrary neug_py_bind
continue
```

进入 `neug::PyConnection::execute` 后，重点看：

```gdb
print query_string
print access_mode
bt
```

如果参数序列化可疑，单步看 `PyParameterSerializer::SerializeParameter`；如果结果转换可疑，转到 `tools/python_bind/src/py_query_result.cc` 的 `PyQueryResult` 相关函数。

## 5. 调试服务模式 rt_server

服务端二进制在打开 `BUILD_HTTP_SERVER=ON` 和 `BUILD_EXECUTABLES=ON` 后生成：

```bash
cd /home/lcc/cpp/neug/build-debug
gdb --args ./bin/rt_server -d /tmp/tinysnb -p 10000 -s 1 -m 1
```

服务入口在 `bin/rt_server.cc`：

- 解析参数。
- 创建 `neug::NeugDB db`。
- `db.Open(config)` 加载图。
- 创建 `neug::NeugDBService service(db, service_config)`。
- `service.run_and_wait_for_exit()` 开始监听。

常用断点：

```gdb
b main
b neug::NeugDB::Open
b neug::NeugDBService::run_and_wait_for_exit
b neug::NeugDBSession::Eval
run
```

服务启动后，在另一个终端用 Python Session 发查询：

```bash
cd /home/lcc/cpp/neug/tools/python_bind
python3 - <<'PY'
from neug import Session
s = Session("http://localhost:10000")
print(s.execute("MATCH (n) RETURN count(n)", "read").get_all())
PY
```

HTTP 查询最终会进入 `src/server/neug_db_session.cc` 的 `neug::NeugDBSession::Eval`。如果服务端没有停在断点，先确认端口、数据目录和 `/service_status` 是否可访问。

## 6. 调试崩溃和 core dump

先打开 core dump：

```bash
ulimit -c unlimited
```

运行测试或服务崩溃后，用对应二进制打开 core：

```bash
gdb ./build-debug/tests/unittest/test_connection core
gdb ./build-debug/bin/rt_server core
```

进入后直接看：

```gdb
bt full
info threads
thread apply all bt full
```

如果是 Python 进程崩溃：

```bash
gdb "$(which python3)" core
```

然后在 GDB 中：

```gdb
info sharedlibrary neug_py_bind
bt full
thread apply all bt
```

## 7. 断点选择策略

不要一开始就在底层容器或第三方库里下太多断点。推荐按层推进：

1. API 层：`neug::PyConnection::execute`、`neug::Connection::Query`、`neug::NeugDBSession::Eval`。
2. 查询处理层：`neug::QueryProcessor::execute`、`neug::QueryProcessor::execute_internal`。
3. 规划/执行层：根据栈里出现的 planner、physical operator、executor 名称继续下断点。
4. 存储/事务层：`InsertTransaction`、`UpdateTransaction`、CSR 或 property column 相关函数。

排查查询错误时，最小闭环是：

```gdb
b neug::Connection::Query
b neug::QueryProcessor::execute
catch throw
run
print query_string
print access_mode
bt full
```

排查并发或服务问题时，重点加：

```gdb
info threads
thread apply all bt
b neug::NeugDBSession::Eval
b neug::SessionPool::Acquire
```

## 8. 常见问题

### 断点显示 pending 或 never hit

可能原因：

- Python 扩展还没有被 import。保留 `set breakpoint pending on`，继续运行。
- 运行的不是 Debug 构建产物。用 `python3 -c "import neug_py_bind; print(neug_py_bind.__file__)"` 或 `info files` 确认路径。
- 符号被优化掉。确认 `CMAKE_BUILD_TYPE=DEBUG` 或 `BUILD_TYPE=DEBUG`。
- 函数被重载。用 `rbreak QueryProcessor::execute` 或按文件行号下断点。

### GDB 进入大量 third_party 代码

用 `next` 代替 `step`，或者跳过目录：

```gdb
skip -gfi third_party/*
```

### 需要更多日志

Python 绑定：

```bash
DEBUG=ON GLOG_v=10 python3 -m pytest -s tests/test_db_query.py -k 'test_name'
```

C++ 二进制：

```bash
GLOG_logtostderr=1 GLOG_v=10 ./tests/unittest/test_connection
```

### 怀疑内存问题

GDB 适合看崩溃现场；越界、use-after-free、double-free 更适合先用 ASan 缩小范围，再回到 GDB 看具体栈。可以临时追加：

```bash
cmake -S . -B build-asan \
  -DCMAKE_BUILD_TYPE=DEBUG \
  -DBUILD_TEST=ON \
  -DCMAKE_CXX_FLAGS='-fsanitize=address -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=address'
cmake --build build-asan -j"$(nproc)"
```

## 9. 推荐的日常调试模板

### C++ 单测

```bash
cd /home/lcc/cpp/neug/build-debug
gdb --args ./tests/unittest/test_connection --gtest_filter='*'
```

```gdb
set pagination off
set print pretty on
set breakpoint pending on
b neug::Connection::Query
b neug::QueryProcessor::execute
catch throw
run
```

### Python 单测

```bash
cd /home/lcc/cpp/neug/tools/python_bind
DEBUG=ON GLOG_v=10 gdb --args python3 -m pytest -s tests/test_db_query.py -k 'test_name'
```

```gdb
set pagination off
set print pretty on
set breakpoint pending on
b neug::PyConnection::execute
b neug::Connection::Query
b neug::QueryProcessor::execute
run
```

### 服务进程

```bash
cd /home/lcc/cpp/neug/build-debug
gdb --args ./bin/rt_server -d /tmp/tinysnb -p 10000 -s 1 -m 1
```

```gdb
set pagination off
set print pretty on
b main
b neug::NeugDB::Open
b neug::NeugDBSession::Eval
run
```
