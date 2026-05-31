# NeuG CMake Build System

## 原问题

解读这个项目的 CMakeLists，做成笔记。

## 总览

NeuG 的 CMake 不是“每个模块各自生成一个库再互相链接”的结构，而是：

1. 根目录 `CMakeLists.txt` 负责全局选项、编译参数、第三方依赖、安装导出、测试/示例/扩展入口。
2. `src/**/CMakeLists.txt` 多数生成 `OBJECT` library。
3. 各子模块把 `$<TARGET_OBJECTS:...>` 追加到 `ALL_OBJECT_FILES`。
4. `src/CMakeLists.txt` 最后用这些 object files 组装出核心共享库 `neug`。

核心产物可以理解为：

```text
third_party deps + generated proto + src object libraries
-> libneug.so / libneug.dylib
-> Python binding / executable / tests / extensions
```

## 根目录 CMakeLists.txt

路径：`CMakeLists.txt`

### 版本与生成头文件

- 从 `NEUG_VERSION` 读取完整版本号。
- 用正则提取 `<major>.<minor>.<patch>`，设置 `project(NeuG VERSION ...)`。
- 生成：
  - `${build}/include/neug/version.h`
  - `${build}/include/neug/config.h`
- 添加宏：
  - `NEUG_VERSION`
  - `NEUG_CMAKE_VERSION`

这说明版本信息不是只用于 package metadata，也会进入 C++ 编译产物。

### 主要构建开关

常用开关：

| 选项 | 默认值 | 作用 |
| --- | --- | --- |
| `BUILD_TEST` | `OFF` | 是否构建 C++ tests |
| `BUILD_DOC` | `OFF` | 是否生成 Doxygen 文档 |
| `BUILD_EXECUTABLES` | `OFF` | 是否构建 `bin/` 下 executable |
| `BUILD_HTTP_SERVER` | `OFF` | 是否构建 HTTP service 相关代码 |
| `WITH_MIMALLOC` | `ON` | 是否链接 mimalloc |
| `BUILD_PYTHON` | `ON` | 是否构建 pybind11 Python binding |
| `BUILD_COMPILER` | `ON` | 是否构建 compiler/query planning 相关模块 |
| `BUILD_EXAMPLES` | `ON` | 是否构建 examples |
| `BUILD_EXTENSIONS` | `""` | 分号分隔的 extension 列表，如 `parquet` |

调试/工具类开关：

| 选项 | 作用 |
| --- | --- |
| `ENABLE_WERROR` | warnings as errors |
| `ENABLE_ADDRESS_SANITIZER` | AddressSanitizer |
| `ENABLE_THREAD_SANITIZER` | ThreadSanitizer |
| `ENABLE_UBSAN` | UndefinedBehaviorSanitizer |
| `ENABLE_LTO` | Link-Time Optimization |
| `ENABLE_GCOV` | coverage |
| `ENABLE_BACKTRACES` | exception/segfault backtrace |
| `AUTO_UPDATE_GRAMMAR` | grammar 自动生成相关 |

配置型变量：

| 变量 | 默认值 | 用途 |
| --- | --- | --- |
| `NEUG_DEFAULT_REL_STORAGE_DIRECTION` | `BOTH` | relation table 默认存储方向 |
| `NEUG_PAGE_SIZE_LOG2` | `12` | page size log2 |
| `NEUG_VECTOR_CAPACITY_LOG2` | `11` | vector capacity log2 |
| `NEUG_NODE_GROUP_SIZE_LOG2` | `17` | node group size log2 |
| `ENABLE_PROTOCOLS` | `http` | 启用协议，当前识别 `http` 和 `all` |

注意：这些数值型配置在当前文件里用 `option()` 声明，语义上更像 cache variable。读代码时按“可由 `-D...=...` 覆盖的构建参数”理解即可。

### 全局编译参数

项目强制 C++20：

```cmake
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
```

同时还会检查 `-std=c++20` / `-std=c++2a` 并追加到 `CMAKE_CXX_FLAGS`。

Linux/GCC 下的关键行为：

- 默认 `Release`。
- 非 Apple 且非 Clang 时：
  - 加 `-Wl,-rpath,$ORIGIN`
  - 加 `-Werror`
  - 加 OpenMP：`-fopenmp`
- 默认加：
  - `-Wall`
  - `-fPIC`
  - `-Wno-psabi`
- `OPTIMIZE_FOR_HOST=ON` 且非 Apple 时加 `-march=native`。
- Debug: `-O0 -g`
- Release: `-O3`

Clang 相关：

- 非 Apple/Clang executable linker flags 会加 `-lopen-pal`。
- Clang 统一加 `-Wno-uninitialized`、`-fclang-abi-compat=14` 等兼容参数。

### 平台、RPATH 与符号导出

项目设置：

```cmake
set(CMAKE_ENABLE_EXPORTS TRUE)
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
```

原因是 extension 可能通过 `dlopen` 加载，需要 executable/shared library 的公共符号可见。

符号可见性集中在：

- `cmake/neug_symbol_visibility.cmake`
- `neug_apply_symbol_visibility(neug)`
- `neug_apply_symbol_visibility(neug_py_bind)`

核心目的：隐藏 Arrow/protobuf/abseil 等 third-party 符号，降低和 `pyarrow` 等 Python 生态库的符号冲突风险。

### 第三方依赖入口

根 CMake 管理大部分 third-party：

| 依赖 | 触发方式 |
| --- | --- |
| `mimalloc` | `WITH_MIMALLOC=ON` 时 `add_subdirectory(third_party/mimalloc)` |
| `fast_float` | 总是加入 |
| `abseil` | `BuildAbseilThirdParty.cmake` |
| `gtest/gmock` | `BUILD_TEST=ON` 时加入 |
| `cpptrace` | `ENABLE_BACKTRACES=ON` 时 `find_package` 或 `FetchContent` |
| `OpenSSL` | 必需 |
| `pybind11` | `BUILD_PYTHON=ON` 时加入 |
| `gflags` | `BuildGflagsAsThirdParty.cmake` |
| `glog` | `BuildGlogAsThirdParty.cmake` |
| `yaml-cpp` | `BuildYamlCppAsThirdParty.cmake` |
| `protobuf` | `BuildProtobufThirdParty.cmake` |
| `leveldb` | `BUILD_HTTP_SERVER=ON` 时加入 |
| `brpc` | `BUILD_HTTP_SERVER=ON` 时加入 |
| `Arrow` | 系统 Arrow 或 bundled Arrow |

Arrow 的策略比较特殊：

- 默认 `NEUG_USE_SYSTEM_ARROW=OFF`，通过 `cmake/BuildArrowAsThirdParty.cmake` 构建 bundled Arrow。
- 如果环境变量 `CI=ON`，改用系统安装的 Arrow。
- Arrow JSON 总是开启。
- 如果 `BUILD_EXTENSIONS` 包含 `parquet`，会开启 Arrow Parquet，并解析 `Parquet::parquet*` 目标或 fallback 到 `libparquet` 文件。

### 子目录展开顺序

根 CMake 的主要展开顺序：

```text
third_party / dependency setup
-> add_subdirectory(src)
-> add_subdirectory(tools)
-> optionally add_subdirectory(bin)
-> optionally add_subdirectory(tests)
-> optionally add_subdirectory(examples)
-> compiler third_party targets
-> install/export config
-> format/lint/coverage targets
```

这里有一个重要特点：`src/CMakeLists.txt` 里链接了一些 compiler third-party target 名称，但这些 target 的 `add_subdirectory(third_party/...)` 在后面才出现。CMake 允许先在 link interface 里写 target/library 名称，后续再创建目标。

## src/CMakeLists.txt

路径：`src/CMakeLists.txt`

这是核心库 `neug` 的组装点。

子模块顺序：

```cmake
add_subdirectory(utils)
add_subdirectory(storages)
add_subdirectory(transaction)
add_subdirectory(common)

if (BUILD_COMPILER)
    add_subdirectory(compiler)
    add_subdirectory(main)
endif()

add_subdirectory(server)
add_subdirectory(execution)
```

最后：

```cmake
add_library(neug SHARED ${ALL_OBJECT_FILES})
```

所以 `libneug` 的源码来自各子目录收集的 object files。

`neug` 主要链接：

- `OpenSSL::SSL`
- `OpenSSL::Crypto`
- `gflags`
- `yaml-cpp`
- `COMPILER_LIBRARIES`
- `Arrow`
- `glog`
- `protobuf`
- `brpc/leveldb`，仅 HTTP server 开启时
- `mimalloc`，仅 `WITH_MIMALLOC=ON` 时
- `cpptrace`，仅 backtrace 开启时

`protobuf` 被设为 `PRIVATE`，注释里说明原因：避免下游 shared library，例如 `neug_py_bind.so`，重复初始化 protobuf global statics。

## src 子模块模式

大多数模块都是 `OBJECT` library。

例子：

```text
src/main/CMakeLists.txt
-> add_library(neug_main OBJECT ...)
-> set(ALL_OBJECT_FILES ... PARENT_SCOPE)

src/execution/CMakeLists.txt
-> add_library(neug_execution OBJECT ...)
-> set(ALL_OBJECT_FILES ... PARENT_SCOPE)

src/common/CMakeLists.txt
-> add_library(neug_common OBJECT ...)
-> set(ALL_OBJECT_FILES ... PARENT_SCOPE)
```

`src/storages/CMakeLists.txt` 继续拆成：

```text
graph
csr
loader
container
```

然后把这些 storage object targets 合并进 `ALL_OBJECT_FILES`。

这种设计的含义：

- 子模块不直接生成多个 installable libraries。
- 最终 ABI/API 边界集中在 `neug` 这个 shared library。
- 子模块之间如果有循环依赖，可以通过统一 object 聚合绕开一部分 link ordering 问题。
- 缺点是模块边界在 CMake target 层不够强，依赖关系不如 modern CMake target graph 清晰。

## compiler 模块

路径：`src/compiler/CMakeLists.txt`

compiler 模块做了几件事：

- 关闭 warning：`add_compile_options(-w)`。
- 加入 parser/runtime 相关 include：
  - `third_party/re2/include`
  - `third_party/antlr4_cypher/include`
  - `third_party/antlr4_runtime/src`
  - `third_party/utf8proc/include`
- 生成 `system_config.h`：
  - 输入：`cmake/templates/system_config.h.in`
  - 输出：`${build}/src/compiler/neug/compiler/common/system_config.h`
- 添加宏：`ANTLR4CPP_STATIC`
- 展开 binder/catalog/common/function/graph/main/optimizer/parser/planner/transaction/extension/gopt 等子目录。

compiler 模块最后把自己的 `ALL_OBJECT_FILES` 继续 `PARENT_SCOPE` 给上层 `src/CMakeLists.txt`。

## Proto 生成

路径：`src/utils/CMakeLists.txt`

这里定义了 `compile_proto(...)` function，负责调用 `protoc` 生成 `.pb.h` 和 `.pb.cc`。

生成内容分三类：

1. `response.proto`
2. `http_svc.proto`，仅 `BUILD_HTTP_SERVER=ON`
3. `proto/*.proto` 中除 response/http service 外的 physical plan protos

然后创建：

```cmake
add_library(neug_proto OBJECT ${ALL_PROTO_SRCS})
```

`neug_utils` 依赖 `neug_proto`，最终二者都进入 `libneug`。

## Python binding

路径：`tools/python_bind/CMakeLists.txt`

入口条件：

- 根目录 `BUILD_PYTHON=ON`
- `tools/CMakeLists.txt` 中要求 `TARGET neug_main` 存在，否则跳过 Python binding

核心目标：

```cmake
pybind11_add_module(neug_py_bind MODULE ${SOURCE_CPP})
```

重要细节：

- 明确 `set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)`，避免 pybind11 默认 LTO 触发 GCC-13 ICE。
- `neug_py_bind` 依赖 `neug_proto`。
- 链接：
  - `neug`
  - `glog`
  - `protobuf`
- Python binding 自己也应用 `neug_apply_symbol_visibility`，避免 protobuf/abseil/arrow 符号泄露到 Python 进程。

## HTTP server / executables

HTTP server 由 `BUILD_HTTP_SERVER` 控制。

涉及路径：

- `src/server/CMakeLists.txt`
- `bin/CMakeLists.txt`

`src/server`：

- 仅在 `BUILD_HTTP_SERVER=ON` 时收集 server `.cc`。
- 创建 `neug_execution_service OBJECT`。
- 依赖 `neug_proto` 和 `neug_utils`。
- object files 进入 `libneug`。

`bin`：

- 仅在 `BUILD_HTTP_SERVER=ON` 时构建：
  - `rt_server`
  - `benchmark`
- 两者都链接 `neug`、`glog`、`protobuf`、`Arrow` 等。

所以 `BUILD_EXECUTABLES=ON` 只是允许进入 `bin/`，真正的 server executable 还需要 `BUILD_HTTP_SERVER=ON`。

## Tests

路径：`tests/CMakeLists.txt`

根 CMake 在 `BUILD_TEST=ON` 时：

- `enable_testing()`
- 使用 bundled `third_party/gtest`
- 创建 imported interface target：`neug::GTest`
- 定义 helper function：`add_neug_test(TEST_NAME ...)`

`add_neug_test` 会：

- 创建 executable。
- 链接：
  - `neug::GTest`
  - `neug`
  - `protobuf`
  - `glog`
  - `Arrow`
  - `OpenSSL`
  - `yaml-cpp`
- 调用 `add_test(...)` 和 `gtest_add_tests(...)`。

`tests/CMakeLists.txt` 展开：

```text
compiler
execution
storage
transaction
unittest
utils
main
```

例如 `tests/compiler/CMakeLists.txt` 通过 `add_neug_test(gopt_test ...)` 把多个 compiler/query planning 测试源文件合成一个测试 executable。

## Extensions

路径：

- `extension/CMakeLists.txt`
- `extension/parquet/CMakeLists.txt`

根 CMake 通过 `BUILD_EXTENSIONS` 控制扩展：

```bash
cmake -S . -B build -DBUILD_EXTENSIONS=parquet
```

`extension/CMakeLists.txt` 提供三个核心 function：

| function | 作用 |
| --- | --- |
| `set_extension_properties` | 设置 extension 输出目录、文件名、后缀、RPATH |
| `build_extension_lib` | 根据 `<EXT>_EXTENSION_OBJECT_FILES` 创建 shared extension |
| `add_extension_test` | 为 extension 创建测试 executable |

extension 产物后缀是：

```text
lib<name>.neug_extension
```

输出目录形如：

```text
${build_or_python_output}/extension/<extension_name>/
```

`parquet` extension：

- 收集 `extension/parquet/src/*.cc` 和 `*.cpp`。
- 创建 `neug_parquet_extension`。
- 链接 `neug`、Arrow base、Arrow dataset、Parquet。
- 注释明确说明 extension 不应用 symbol visibility，因为隐藏 Arrow 符号可能导致 `dlopen` undefined symbol。

## Install / Package Config

根 CMake 定义：

- `install_neug_target(target)`
- `install_without_export_neug_target(target)`

`neug` 会安装并导出到：

```text
${CMAKE_INSTALL_LIBDIR}/cmake/neug
```

生成并安装：

- `neug-config.cmake`
- `neug-config-version.cmake`
- `neug-targets.cmake`

安装后，下游项目理论上可以：

```cmake
find_package(neug CONFIG REQUIRED)
target_link_libraries(my_app PRIVATE neug::neug)
```

## 常用构建命令

默认构建核心库、Python binding、examples：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
```

构建 C++ tests：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TEST=ON
cmake --build build -j
ctest --test-dir build --output-on-failure
```

构建 HTTP server executable：

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DBUILD_EXECUTABLES=ON \
  -DBUILD_HTTP_SERVER=ON
cmake --build build -j
```

构建 parquet extension：

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DBUILD_EXTENSIONS=parquet
cmake --build build -j
```

开启 sanitizer：

```bash
cmake -S . -B build-asan \
  -DCMAKE_BUILD_TYPE=Debug \
  -DENABLE_ADDRESS_SANITIZER=ON
cmake --build build-asan -j
```

## 阅读 CMake 的路线

建议按这个顺序读：

1. `CMakeLists.txt`：先看 option、dependency setup、`add_subdirectory` 顺序。
2. `src/CMakeLists.txt`：理解 `libneug` 如何由 object files 拼起来。
3. `src/utils/CMakeLists.txt`：理解 protobuf generated code。
4. `src/compiler/CMakeLists.txt`：理解 parser/compiler 相关 third-party 和 generated config。
5. `tools/python_bind/CMakeLists.txt`：理解 Python binding 的链接和符号隐藏。
6. `extension/CMakeLists.txt`：理解插件式 extension 的产物布局和 RPATH。
7. `tests/CMakeLists.txt`：理解 `add_neug_test` 如何封装测试。

## 我的理解

这个项目的 CMake 重点不是“写法优雅”，而是服务于几个现实约束：

- C++ graph database 模块多，最终希望暴露一个核心 shared library：`neug`。
- Python binding 和 pyarrow/protobuf/abseil 可能共存，所以 symbol visibility 很重要。
- Arrow/Parquet/brpc/protobuf 等依赖复杂，需要大量 third-party glue CMake。
- extension 需要动态加载，所以 RPATH、符号导出、输出目录都被显式处理。
- object library 聚合降低了链接多个内部库的复杂度，但也让 CMake target dependency graph 变得不够清晰。

## 后续问题

- `NEUG_PAGE_SIZE_LOG2` 等数值配置是否应该从 `option()` 改成 `set(... CACHE STRING ...)`？
- `src/**` 是否值得逐步从 object 聚合迁移成更清晰的 internal static libraries？
- `BUILD_HTTP_SERVER` 同时影响 proto、server object、third-party brpc/leveldb、bin executable，是否需要单独画依赖图？
- `BUILD_PYTHON=ON` 是默认值，纯 C++ 用户是否应默认关闭以减少构建依赖？
