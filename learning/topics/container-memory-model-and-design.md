# Storage Container Memory Model And Design

## 原始问题

```text
详细介绍 container 的各个函数，以及内存模型，写成笔记，解释设计原理
```

## 核心结论

NeuG 的 storage container 层把底层存储统一抽象成：

```text
void* data_ + size_t size_
```

上层的 `Column`、`CSR`、`Indexer` 不直接关心数据来自文件、匿名 mmap、shared mmap 还是 hugepage。它们只拿：

```cpp
container->GetData()
container->GetDataSize()
container->Resize(bytes)
container->Dump(path)
```

container 层负责：

- 打开 checkpoint 文件。
- 在 `kSyncToFile` 模式下复制 checkpoint 到 `runtime/tmp` 并 mmap 工作副本。
- 在 `kInMemory` 模式下用 private mmap 或 anonymous mmap。
- 在 `kHugePagePreferred` 模式下优先用 hugepage anonymous mmap。
- 统一处理 file header、payload 指针、resize、dump、close、dirty check。

类层级：

```text
IDataContainer
  └── MMapContainer
        ├── FilePrivateMMap
        ├── FileSharedMMap
        ├── AnonMMap
        └── AnonHugeMMap

factory:
  OpenContainer(snapshot_file, tmp_file, memory_level)
    └── OpenDataContainer(memory_level, file_name)
```

## 文件和源码位置

```text
include/neug/storages/container/i_container.h
include/neug/storages/container/mmap_container.h
include/neug/storages/container/file_mmap_container.h
include/neug/storages/container/anon_mmap_container.h
include/neug/storages/container/container_utils.h
include/neug/storages/container/file_header.h

src/storages/container/mmap_container.cc
src/storages/container/file_mmap_container.cc
src/storages/container/anon_mmap_container.cc
src/storages/container/container_utils.cc
```

## MemoryLevel

`MemoryLevel` 定义在生成配置头的模板 `include/neug/config.h.in`：

```cpp
enum class MemoryLevel : uint8_t {
  kUnSet = 0,
  kInMemory = 1,
  kSyncToFile = 2,
  kHugePagePreferred = 3
};
```

container 层只处理后三种：

```text
kInMemory
  - 如果 snapshot 文件存在，用 FilePrivateMMap 打开。
  - 如果没有文件名，用 AnonMMap。
  - 修改不写回原文件。
  - Dump 时重新写一个新文件。

kSyncToFile
  - 必须有 tmp_file。
  - open 前 copy snapshot_file -> tmp_file。
  - tmp_file 用 FileSharedMMap 打开。
  - 写入会进入 tmp file 的 MAP_SHARED 映射。
  - Dump 时 Sync，然后 rename/copy 到 checkpoint。

kHugePagePreferred
  - 用 AnonHugeMMap。
  - 如果 snapshot 文件存在，把文件内容读进 hugepage/anonymous memory。
  - resize 时优先 hugepage，失败 fallback 到普通 anonymous mmap。
```

## FileHeader

所有持久化 container 文件前面都有一个 `FileHeader`：

```cpp
struct FileHeader {
  unsigned char data_md5[MD5_DIGEST_LENGTH];
};
```

文件格式：

```text
file bytes:
  [FileHeader: MD5(payload)][payload bytes...]

mmap_data_:
  指向文件映射起点，也就是 header 起点

data_:
  指向 payload 起点，也就是 mmap_data_ + sizeof(FileHeader)

size_:
  payload bytes 数量
```

上层永远不应该看到 header。`GetData()` 返回的是 payload 起点，不是文件起点。

匿名 mmap 的布局不同：

```text
anonymous mmap:
  [payload bytes...]

mmap_data_ == data_
mmap_size_ == size_
path_ == ""
```

所以 container 有两种内存形态：

```text
file-backed:
  mmap_data_ -> [header][payload]
  data_      ->          [payload]

anonymous:
  mmap_data_ -> [payload]
  data_      -> [payload]
```

## IDataContainer 接口

`IDataContainer` 是所有上层 storage 代码依赖的接口。

### 状态字段

```cpp
void* data_;
size_t size_;
```

含义：

- `data_`: 上层可读写的数据区起点。
- `size_`: 上层可读写的数据区大小，单位是 bytes。

注意：`size_` 不是元素个数。`Column<T>` 会自己用：

```text
element_count = container->GetDataSize() / sizeof(T)
```

### GetContainerType()

```cpp
virtual ContainerType GetContainerType() const = 0;
```

返回具体容器类型：

```text
kAnonMMap
kAnonHugeMMap
kFilePrivateMMap
kFileSharedMMap
```

这个函数主要用于测试、诊断或需要知道具体策略的地方。正常 storage 读写不需要依赖它。

### GetData()

```cpp
inline void* GetData() const { return data_; }
```

返回 payload 起点。上层通常会 reinterpret：

```cpp
auto* ptr = reinterpret_cast<T*>(container->GetData());
```

典型使用：

```text
TypedColumn<T>:
  reinterpret_cast<T*>(buffer_->GetData())[index]

ImmutableCsr:
  reinterpret_cast<nbr_t*>(nbr_list_buffer_->GetData())

Indexer:
  reinterpret_cast<INDEX_T*>(indices_->GetData())
```

### GetDataSize()

```cpp
inline size_t GetDataSize() const { return size_; }
```

返回 payload bytes，不包含 `FileHeader`。

### Resize(size)

```cpp
virtual void Resize(size_t size) = 0;
```

把 payload 调整到指定 bytes。不同实现语义不同：

- `MMapContainer::Resize`: 新建 anonymous mmap，复制旧 payload，旧映射 munmap。
- `FileSharedMMap::Resize`: ftruncate backing file，然后重新 MAP_SHARED。
- `AnonHugeMMap::Resize`: 分配 rounded hugepage size，复制旧 payload，旧映射 munmap。

### GetPath()

```cpp
virtual std::string GetPath() const = 0;
```

返回 backing file path。匿名 mmap 或 resize 后变成 anonymous 的容器通常返回空字符串。

### Open(path)

```cpp
virtual void Open(const std::string& path) = 0;
```

打开一个文件并建立映射。`MMapContainer` 实现通用流程，子类只实现 `mmapImpl()`。

### Sync()

```cpp
virtual void Sync() = 0;
```

把修改同步到底层持久化介质。当前只有 `FileSharedMMap::Sync()` 真正做事；普通 `MMapContainer::Sync()` 是空函数。

### Dump(path)

```cpp
virtual void Dump(const std::string& path) = 0;
```

把当前 payload 写成持久化文件，并关闭当前 container。

注意：注释里也写了 dump 会 close。调用 dump 后，上层不应继续使用旧指针。

### Close()

```cpp
virtual void Close() = 0;
```

释放 mmap 资源并重置状态。

### IsDirty()

```cpp
virtual bool IsDirty() = 0;
```

判断 payload 是否和 header 里的 MD5 不一致。匿名 mmap 没有 backing file，当前实现认为只要有映射就是 dirty。

## MMapContainer 基类

`MMapContainer` 实现了大多数共用逻辑。

### 状态字段

继承自 `IDataContainer`：

```text
data_
size_
```

自己新增：

```text
path_       // backing file path；匿名映射为空
mmap_data_  // mmap 返回的真实起点
mmap_size_  // mmap 区域大小
```

状态可以理解为：

```text
Closed:
  path_ = ""
  mmap_data_ = nullptr
  mmap_size_ = 0
  data_ = nullptr
  size_ = 0

File-backed:
  path_ = "/path/to/file"
  mmap_data_ = mmap(file, file_size)
  mmap_size_ = file_size
  data_ = mmap_data_ + sizeof(FileHeader)
  size_ = mmap_size_ - sizeof(FileHeader)

Anonymous:
  path_ = ""
  mmap_data_ = mmap(anonymous, size)
  mmap_size_ = size or hugepage-rounded size
  data_ = mmap_data_
  size_ = logical payload size
```

### MMapContainer()

```cpp
MMapContainer::MMapContainer()
    : IDataContainer(), mmap_data_(nullptr), mmap_size_(0) {}
```

构造时只初始化为空状态。

### GetPath()

```cpp
std::string MMapContainer::GetPath() const { return path_; }
```

返回当前 backing file path。匿名映射返回空字符串。

### Open(path)

流程：

```text
MMapContainer::Open(path)
  1. 如果当前已经打开，先 Close()
  2. 检查文件存在；不存在则抛 IO exception
  3. path_ = path
  4. mmap_size_ = file_size(path)
  5. 如果文件大小为 0：
       清空 path_
       warning
       return
  6. mmap_data_ = mmapImpl(path, mmap_size_)
  7. 如果 mmap failed，抛异常
  8. 如果 mmap_size_ < sizeof(FileHeader)：
       munmap
       清空状态
       抛异常
  9. data_ = mmap_data_ + sizeof(FileHeader)
  10. size_ = mmap_size_ - sizeof(FileHeader)
  11. 不校验 checksum
```

关键点：

- `Open()` 不直接调用 `mmap()`，而是调用子类的 `mmapImpl()`。
- file-backed open 后，`data_` 跳过 header。
- 当前代码注释明确写着 skip checksum verification，也就是 open 时不验证 MD5，只在 dirty check / sync / dump 时使用 MD5。

### Close()

流程：

```text
if mmap_data_ && mmap_size_ > 0:
  munmapImpl(mmap_data_, mmap_size_)

path_.clear()
mmap_data_ = nullptr
mmap_size_ = 0
data_ = nullptr
size_ = 0
```

这里也通过子类的 `munmapImpl()` 释放，因为 hugepage 需要用 rounded size。

### Sync()

```cpp
void MMapContainer::Sync() {}
```

默认空实现。private mmap / anonymous mmap 没有直接同步回原文件的语义。

### Resize(size)

基类 resize 的语义是“变成匿名 mmap”。

流程：

```text
MMapContainer::Resize(size)
  1. 如果 size == size_，直接 return
  2. path_.clear()
  3. 如果 size == 0：
       munmap old mapping
       清空状态
       return
  4. 保存旧状态：
       old_mmap_data
       old_mmap_size
       old_data
       old_size
  5. mmap anonymous private region，大小为新 payload size
  6. 如果旧 payload 存在：
       memcpy(new, old, min(old_size, size))
  7. munmap old mapping
  8. mmap_data_ = new_mmap_data
  9. mmap_size_ = size
  10. data_ = mmap_data_
  11. size_ = size
```

重要设计含义：

- 对 `FilePrivateMMap` 或 `AnonMMap` 来说，resize 后数据在匿名内存里，不再关联原文件。
- 因为 resize 后 `path_` 被清空，所以后续 `Dump(path)` 会走 fwrite 方式写出完整新文件。
- 基类 resize 的新区域没有 `FileHeader`，所以 `data_ == mmap_data_`。

### Dump(path)

基类 dump 使用 fwrite 写完整文件。

流程：

```text
MMapContainer::Dump(path)
  1. MD5(data_, size_) -> header.data_md5
  2. fopen(path, "wb")
  3. fwrite(header)
  4. if size_ > 0:
       fwrite(payload)
  5. Close()
```

结果文件格式：

```text
[FileHeader][payload]
```

### IsDirty()

流程：

```text
MMapContainer::IsDirty()
  1. if mmap_data_ == nullptr:
       return false

  2. if path_.empty():
       return true

  3. if size_ == 0:
       return false

  4. md5 = MD5(data_, size_)
  5. return md5 != FileHeader(mmap_data_).data_md5
```

设计含义：

- 没有映射就不是 dirty。
- 匿名映射只要存在就认为 dirty，因为没有 header 可以对比。
- header-only 文件没有 payload，认为不 dirty。
- file-backed 映射用 header 中保存的 MD5 判断 payload 是否改变。

## FilePrivateMMap

`FilePrivateMMap` 用于 `kInMemory` 打开已有 snapshot 文件。

### 设计语义

```text
file + MAP_PRIVATE
```

特点：

- 从文件加载初始内容。
- 写入映射时触发 OS copy-on-write。
- 修改不会写回原文件。
- dump 时需要重新写新文件。

### 构造和析构

```cpp
FilePrivateMMap::FilePrivateMMap() : MMapContainer() {}
FilePrivateMMap::~FilePrivateMMap() { Close(); }
```

析构时释放 mmap。

### GetContainerType()

```cpp
return ContainerType::kFilePrivateMMap;
```

### OpenAnonymous(size)

虽然类名是 `FilePrivateMMap`，但它也提供 anonymous allocation：

```text
OpenAnonymous(size)
  1. 如果已有映射，Close()
  2. path_.clear()
  3. mmap_size_ = size
  4. mmap(PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS)
  5. data_ = mmap_data_
  6. size_ = mmap_size_
```

这个函数在测试里会用来构造临时 payload，然后 `Dump()` 成带 header 的文件。

### mmapImpl(path, mmap_size)

```text
1. open(path, O_RDONLY)
2. mmap(PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0)
3. close(fd)
4. return mmap_data
```

虽然 fd 是 `O_RDONLY`，但 `MAP_PRIVATE + PROT_WRITE` 是允许的：写入修改的是私有 COW 页，不写回文件。

### munmapImpl(data, size)

```cpp
munmap(mmap_data, mmap_size);
```

## FileSharedMMap

`FileSharedMMap` 用于 `kSyncToFile` 模式。

### 设计语义

```text
tmp file + MAP_SHARED
```

特点：

- open 时通常打开的是 `runtime/tmp` 里的工作副本，不是 checkpoint 原文件。
- 写入映射会修改 tmp file 的 page cache。
- `Sync()` 更新 header MD5 并 `msync()`。
- `Dump()` 后把 tmp file rename/copy 到 checkpoint 位置。

### 构造和析构

```cpp
FileSharedMMap::FileSharedMMap() : MMapContainer() {}
FileSharedMMap::~FileSharedMMap() { Close(); }
```

### GetContainerType()

```cpp
return ContainerType::kFileSharedMMap;
```

### Resize(size)

这是 `FileSharedMMap` 相对基类最关键的 override。

流程：

```text
FileSharedMMap::Resize(size)
  1. 如果 size == size_，直接 return
  2. real_size = size + sizeof(FileHeader)
  3. 如果已有映射：
       munmap old mapping
  4. 清空 mmap_data_/data_/size_
  5. open(path_, O_RDWR)
  6. ftruncate(fd, real_size)
  7. close(fd)
  8. mmap_size_ = file_size(path_)
  9. mmap_data_ = mmapImpl(path_, mmap_size_)
  10. data_ = mmap_data_ + sizeof(FileHeader)
  11. size_ = mmap_size_ - sizeof(FileHeader)
```

和基类 resize 的区别：

```text
MMapContainer::Resize:
  allocate anonymous memory
  path_ is cleared

FileSharedMMap::Resize:
  resize backing tmp file
  keep path_
  re-map as MAP_SHARED
```

这保证 `kSyncToFile` 模式下扩容后的内容仍然落在 tmp file 中。

### mmapImpl(path, mmap_size)

```text
1. open(path, O_RDWR)
2. mmap(PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0)
3. close(fd)
4. return mmap_data
```

### munmapImpl(data, size)

```cpp
munmap(mmap_data, mmap_size);
```

### Sync()

流程：

```text
FileSharedMMap::Sync()
  1. 如果没有映射、没有 data、或 size_ == 0，return
  2. md5 = MD5(data_, size_)
  3. 如果 md5 != FileHeader(mmap_data_).data_md5:
       copy md5 into header
       msync(mmap_data_, mmap_size_, MS_SYNC)
```

关键点：

- `msync()` 范围从 `mmap_data_` 开始，包括 header 和 payload。
- 只有 MD5 变化时才 msync。
- header-only 文件不会 sync，因为 `size_ == 0` 直接 return。

### Dump(path)

`FileSharedMMap::Dump()` 不走 fwrite 复制 payload，而是尽量利用 tmp file。

流程：

```text
FileSharedMMap::Dump(path)
  1. 如果 path_ 为空：
       MMapContainer::Dump(path)
       Close()
       return

  2. Sync()

  3. 如果目标 path == 当前 path_：
       Close()
       return

  4. src_path = move(path_)
  5. Close()

  6. 尝试 rename(src_path, path)
       成功则 return

  7. 如果 errno != EXDEV：
       抛 IO exception

  8. 跨文件系统 fallback：
       copy_file(src_path, path, overwrite=true)
       unlink(src_path)
```

设计含义：

- 同一文件系统上，dump 几乎是 O(1) 的 rename。
- 跨文件系统时才 copy。
- `copy_file()` 底层会尽量用 reflink / copy_file_range / read-write fallback。
- dump 后 container 已关闭，旧指针失效。

## AnonMMap

`AnonMMap` 是普通 anonymous mmap 容器。

### 设计语义

```text
MAP_PRIVATE | MAP_ANONYMOUS
```

用于没有文件输入的纯内存数据，也用于 `kInMemory` 中文件不存在的情况。

### 构造和析构

```cpp
AnonMMap::AnonMMap() : MMapContainer() {}
AnonMMap::~AnonMMap() { Close(); }
```

### GetContainerType()

```cpp
return ContainerType::kAnonMMap;
```

### OpenAnonymous(size)

流程和 `FilePrivateMMap::OpenAnonymous()` 基本一致：

```text
1. 如果已有映射，Close()
2. path_.clear()
3. mmap_size_ = size
4. mmap(PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS)
5. data_ = mmap_data_
6. size_ = mmap_size_
```

### mmapImpl(path, mmap_size)

```text
1. open(path, O_RDONLY)
2. mmap(PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0)
3. close(fd)
4. return mmap_data
```

这个和 `FilePrivateMMap::mmapImpl()` 几乎一样。区别主要在类型标识和语义命名上。

### munmapImpl(data, size)

```cpp
munmap(mmap_data, mmap_size);
```

## AnonHugeMMap

`AnonHugeMMap` 用于 `MemoryLevel::kHugePagePreferred`。

### 设计语义

目标是给大块 column / CSR buffer 使用更大的页，减少 TLB miss。

Linux 下优先：

```text
MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB
```

如果 hugepage 分配失败，部分路径 fallback 到普通 anonymous mmap。

### hugepage round up

文件里定义了 2MB hugepage：

```text
HUGEPAGE_SIZE = 2 * 1024 * 1024
HUGEPAGE_MASK = 2 * 1024 * 1024 - 1
ROUND_UP(size) = align up to 2MB
```

辅助函数：

```cpp
void* allocate_hugepages(size_t size) {
  return mmap(ADDR, ROUND_UP(size), PROTECTION, FLAGS, -1, 0);
}

size_t hugepage_round_up(size_t size) { return ROUND_UP(size); }
```

### 构造和析构

```cpp
AnonHugeMMap::AnonHugeMMap() : MMapContainer() {}
AnonHugeMMap::~AnonHugeMMap() { Close(); }
```

### GetContainerType()

```cpp
return ContainerType::kAnonHugeMMap;
```

### OpenAnonymous(size)

流程：

```text
1. 如果已有映射，Close()
2. path_.clear()
3. mmap_size_ = size
4. hugepage_size = round_up(size, 2MB)
5. mmap_data_ = allocate_hugepages(hugepage_size)
6. 如果失败，直接抛异常
7. data_ = mmap_data_
8. size_ = size
```

注意：这里失败会直接抛异常，没有 fallback。fallback 在 `try_allocate_hugepages()`，被 `Resize()` 和 `mmapImpl()` 使用。

还有一个细节：当前 `OpenAnonymous()` 设置 `mmap_size_ = size`，而 `munmapImpl()` 会按 `hugepage_round_up(mmap_size_)` munmap，所以释放时仍会释放 rounded size。

### try_allocate_hugepages(size)

流程：

```text
1. allocate_hugepages(size)
2. 如果失败：
     warning
     mmap regular MAP_PRIVATE | MAP_ANONYMOUS
3. 如果 regular mmap 也失败：
     抛异常
4. return mmap_data
```

这是 `kHugePagePreferred` 里的 preferred：优先 hugepage，但允许 fallback。

### Resize(size)

流程：

```text
AnonHugeMMap::Resize(size)
  1. 如果 size == size_，return
  2. path_.clear()
  3. 如果 size == 0：
       munmap old rounded region
       清空状态
       return
  4. hugepage_size = round_up(size)
  5. new_mmap_data = try_allocate_hugepages(hugepage_size)
  6. 如果旧映射存在：
       memcpy(new, old, min(old_size, size))
       munmap old rounded region
  7. mmap_data_ = new_mmap_data
  8. mmap_size_ = hugepage_size
  9. data_ = mmap_data_
  10. size_ = logical size
```

这里 `mmap_size_` 是实际分配大小，`size_` 是上层可用 payload 逻辑大小。

### mmapImpl(path, mmap_size)

`AnonHugeMMap` 打开已有文件时不使用 file-backed mmap，而是：

```text
1. hugepage_size = round_up(mmap_size)
2. allocate hugepage or fallback regular anonymous mmap
3. fopen(path, "rb")
4. fread whole file into allocated memory
5. return allocated memory
```

之后 `MMapContainer::Open()` 会继续：

```text
data_ = mmap_data_ + sizeof(FileHeader)
size_ = mmap_size_ - sizeof(FileHeader)
```

所以 hugepage 模式是“把文件内容读进匿名内存”，不是直接 mmap 文件。

### munmapImpl(data, mmap_size)

```text
rounded_size = hugepage_round_up(mmap_size)
munmap(data, rounded_size)
```

## container_utils

`container_utils` 是上层真正使用的入口。`Column` / `CSR` / `Indexer` 通常不会直接 new 某个 container，而是调用 `OpenContainer()`。

### prepare_container_file(snapshot_file, tmp_file)

只服务 `kSyncToFile`。

流程：

```text
1. 确保 tmp_file 的 parent dir 存在
2. 如果 snapshot_file 存在：
     copy_file(snapshot_file, tmp_file, overwrite=true)
   否则：
     create_file(tmp_file, sizeof(FileHeader))
```

设计含义：

- sync-to-file 模式不直接修改 checkpoint。
- 每次 open 都先生成 tmp 工作副本。
- 新文件至少有 header，后续 `MMapContainer::Open()` 才能建立 header/payload 布局。

### OpenDataContainer(strategy, file_name)

这是策略到具体类的分发。

#### kInMemory

```text
if file_name.empty():
  return make_unique<AnonMMap>()
else:
  ret = make_unique<FilePrivateMMap>()
  if file exists:
    ret->Open(file_name)
  return ret
```

含义：

- 有 snapshot 文件就 private map 进内存，初始内容来自文件。
- 没有文件时返回一个空的 `FilePrivateMMap` 或 `AnonMMap`，等待后续 `Resize()`。
- private mmap 修改不污染原 checkpoint。

#### kHugePagePreferred

```text
ret = make_unique<AnonHugeMMap>()
if file exists:
  ret->Open(file_name)
return ret
```

含义：

- 始终返回 hugepage 容器。
- 有 snapshot 则读入 hugepage/anonymous memory。
- 没有 snapshot 则等待后续 `Resize()` 分配。

#### kSyncToFile

```text
if file_name.empty():
  throw invalid argument

ret = make_unique<FileSharedMMap>()
if file missing:
  create_file(file_name, sizeof(FileHeader))
ret->Open(file_name)
return ret
```

含义：

- 必须有真实文件，因为 MAP_SHARED 要有 backing file。
- 文件不存在时创建 header-only 文件。

### OpenContainer(snapshot_file, tmp_file, memory_level)

这是最常用入口。

```text
if memory_level == kSyncToFile:
  if tmp_file.empty():
    throw invalid argument
  prepare_container_file(snapshot_file, tmp_file)
  return OpenDataContainer(kSyncToFile, tmp_file)
else:
  return OpenDataContainer(memory_level, snapshot_file)
```

三种模式对比：

```text
kSyncToFile:
  snapshot_file -> copy -> tmp_file -> FileSharedMMap(tmp_file)

kInMemory:
  snapshot_file -> FilePrivateMMap(snapshot_file)
  or empty      -> AnonMMap

kHugePagePreferred:
  snapshot_file -> fread into AnonHugeMMap
  or empty      -> later allocate hugepage on Resize
```

## 完整状态流：kSyncToFile

假设一个 column 文件：

```text
checkpoint/vertex_table_PERSON.col_0
runtime/tmp/vertex_table_PERSON.col_0
```

open：

```text
OpenContainer(checkpoint_file, tmp_file, kSyncToFile)
  -> prepare_container_file()
       if checkpoint exists:
         copy checkpoint -> tmp
       else:
         create tmp with FileHeader only
  -> OpenDataContainer(kSyncToFile, tmp)
       -> FileSharedMMap::Open(tmp)
            -> mmap tmp as MAP_SHARED
            -> data_ skips FileHeader
```

写入：

```text
TypedColumn<T>::set_value()
  -> reinterpret_cast<T*>(container->GetData())[idx] = value
  -> writes to MAP_SHARED tmp mapping
```

扩容：

```text
TypedColumn<T>::resize(n)
  -> container->Resize(n * sizeof(T))
  -> FileSharedMMap::Resize()
       ftruncate(tmp, new_size + header)
       remap tmp
```

dump：

```text
container->Dump(checkpoint_file)
  -> FileSharedMMap::Dump()
       Sync()
       Close()
       rename(tmp, checkpoint)
```

## 完整状态流：kInMemory

open：

```text
OpenContainer(checkpoint_file, tmp_file, kInMemory)
  -> OpenDataContainer(kInMemory, checkpoint_file)
       if checkpoint exists:
         FilePrivateMMap::Open(checkpoint_file)
       else:
         return empty FilePrivateMMap / AnonMMap
```

写入：

```text
MAP_PRIVATE page is changed by COW
checkpoint file is not modified
```

扩容：

```text
container->Resize(bytes)
  -> MMapContainer::Resize()
       allocate anonymous mmap
       copy old payload
       clear path_
```

dump：

```text
container->Dump(checkpoint_file)
  -> MMapContainer::Dump()
       fwrite header
       fwrite payload
       Close()
```

## 完整状态流：kHugePagePreferred

open：

```text
OpenContainer(checkpoint_file, tmp_file, kHugePagePreferred)
  -> OpenDataContainer(kHugePagePreferred, checkpoint_file)
       -> AnonHugeMMap
       -> if checkpoint exists:
            allocate hugepage or fallback anonymous mmap
            fread whole file into memory
            data_ skips FileHeader
```

resize：

```text
container->Resize(bytes)
  -> round bytes up to 2MB
  -> allocate hugepage if possible
  -> fallback regular anonymous mmap if hugepage unavailable
  -> copy old payload
```

dump：

```text
container->Dump(checkpoint_file)
  -> base MMapContainer::Dump()
       fwrite header
       fwrite logical payload bytes
```

## 内存模型图

### 文件持久化格式

```text
offset 0
  |
  v
+----------------------+-------------------------------+
| FileHeader           | payload                       |
| data_md5[16]         | raw bytes used by Column/CSR  |
+----------------------+-------------------------------+
                       ^
                       |
                     data_

mmap_data_ points to file offset 0
data_ points to file offset sizeof(FileHeader)
size_ = file_size - sizeof(FileHeader)
```

### 定长 Column 使用 container

```text
TypedColumn<int32_t>
  buffer_ -> IDataContainer

container payload:
  [int32 row0][int32 row1][int32 row2] ...

set_value(i, v):
  reinterpret_cast<int32_t*>(GetData())[i] = v
```

### CSR 使用 container

```text
ImmutableCsr
  degree_list_buffer_ -> int[vnum]
  nbr_list_buffer_    -> Nbr[edge_num]
  adj_list_buffer_    -> Nbr*[vnum]

all buffers are IDataContainer
```

container 不知道里面是 int、Nbr、pointer array 还是 string bytes。它只管理 bytes。

## 设计原理

### 1. 上层只依赖连续内存

Column / CSR / Indexer 都是性能敏感结构。它们希望直接把 `void*` 当数组用：

```text
Column<T> wants T[]
CSR wants int[] / Nbr[] / Nbr*[]
Indexer wants index slot array
```

container 层把复杂的文件、mmap、hugepage、dump、resize 隐藏掉，上层只处理 typed pointer。

### 2. 读写策略由 MemoryLevel 控制

同一套 Column / CSR 代码可以在三种模式下工作：

```text
kInMemory          快，修改不直接落盘
kSyncToFile        修改在 tmp file 中，通过 dump/checkpoint 持久化
kHugePagePreferred 大块内存访问更友好，降低 TLB 压力
```

上层打开时只传 `MemoryLevel`，不需要分散判断。

### 3. checkpoint 和运行期工作副本隔离

`kSyncToFile` 不直接 mmap checkpoint，而是：

```text
checkpoint -> runtime/tmp -> MAP_SHARED
```

好处：

- 运行中的写不会破坏旧 checkpoint。
- dump 成功后再 rename/copy，接近原子切换。
- 如果运行中失败，旧 checkpoint 仍然可用。

### 4. Header 只服务 dirty check 和 dump/sync

`FileHeader` 保存 payload MD5：

```text
IsDirty:
  compare MD5(payload) with header MD5

Dump:
  write new header MD5 + payload

FileSharedMMap::Sync:
  update header MD5 and msync
```

注意 open 时当前实现不校验 MD5，所以它不是强校验机制，更像 dirty/sync 辅助元数据。

### 5. Resize 语义按持久化策略分裂

这是 container 设计里最重要的差异：

```text
private / anonymous:
  resize = allocate new anonymous memory

shared:
  resize = ftruncate backing file + remap

hugepage:
  resize = allocate rounded hugepage size + copy
```

这样每种模式都保持自己的核心语义：

- private/in-memory 不污染 snapshot。
- shared/sync-to-file 始终保持 tmp file backing。
- hugepage 保持 hugepage preferred。

### 6. Dump 会关闭 container

`Dump()` 后关闭是为了明确所有权和生命周期：

```text
container owns mapping
Dump writes or moves data
Close releases mapping
```

这避免 dump 后还拿旧 mapping 继续写，尤其是 `FileSharedMMap::Dump()` 可能已经 rename/unlink 了 tmp 文件。

## 易错点

- `GetDataSize()` 是 bytes，不是元素数。
- file-backed `GetData()` 已经跳过 `FileHeader`。
- anonymous mmap 没有 header，`data_ == mmap_data_`。
- `MMapContainer::Resize()` 会清空 `path_`，把容器变成 anonymous mapping。
- `FileSharedMMap::Resize()` 不走基类 resize，而是 `ftruncate + remap`。
- `FilePrivateMMap` 写入不会修改原文件，因为它是 `MAP_PRIVATE`。
- `FileSharedMMap` 写的是 tmp file，不应该直接 mmap checkpoint 文件。
- `Dump()` 会 `Close()`，dump 后旧指针失效。
- `AnonHugeMMap::OpenAnonymous()` 分配 hugepage 失败会抛异常；`Resize()` 和 file open 路径才会 fallback。
- `Open()` 不校验 header MD5，`IsDirty()` 才比较 MD5。
- header-only 文件 payload size 为 0，`IsDirty()` 返回 false。

## 函数速查表

| 函数 | 所属 | 作用 |
|---|---|---|
| `GetData()` | `IDataContainer` | 返回 payload 指针 |
| `GetDataSize()` | `IDataContainer` | 返回 payload bytes |
| `Resize(size)` | `IDataContainer` | 调整 payload bytes |
| `Open(path)` | `IDataContainer` | 打开持久化文件 |
| `Sync()` | `IDataContainer` | 同步修改；主要是 shared mmap |
| `Dump(path)` | `IDataContainer` | 写出持久化文件并 close |
| `Close()` | `IDataContainer` | 释放 mmap 并清空状态 |
| `IsDirty()` | `IDataContainer` | 判断 payload 是否不同于 header MD5 |
| `mmapImpl()` | `MMapContainer` 子类 | 子类提供具体 mmap/read 策略 |
| `munmapImpl()` | `MMapContainer` 子类 | 子类提供具体释放策略 |
| `OpenAnonymous()` | `FilePrivateMMap` / `AnonMMap` / `AnonHugeMMap` | 创建匿名映射 |
| `FileSharedMMap::Resize()` | `FileSharedMMap` | ftruncate backing file 并 remap |
| `FileSharedMMap::Sync()` | `FileSharedMMap` | 更新 header MD5 并 msync |
| `FileSharedMMap::Dump()` | `FileSharedMMap` | sync 后 rename/copy tmp file |
| `prepare_container_file()` | `container_utils` | copy checkpoint 到 tmp 或创建 header-only tmp |
| `OpenDataContainer()` | `container_utils` | 根据 MemoryLevel 创建具体容器 |
| `OpenContainer()` | `container_utils` | 对外工厂入口，处理 sync-to-file 的 tmp 准备 |

