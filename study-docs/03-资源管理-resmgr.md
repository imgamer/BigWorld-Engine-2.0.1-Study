# BigWorld Engine 2.0.1 资源管理模块（resmgr）

> 源码目录：`/workspace/src/lib/resmgr/`
> 模块定位：所有 BigWorld 资源（`*.xml` / `*.chunk` / `*.visual` / `*.primitive` / `*.mfm` / `*.tex` / `*.terrain` / `*.model` / `*.sound` …）的**统一访问层**。
> 模块边界：上层（chunk / terrain / moo / model / entitydef 等）一律通过 `BWResource::openSection()` 取得 `DataSectionPtr`；下层落盘与文件枚举由 `MultiFileSystem` + `IFileSystem` 实现栈完成。
> 编号：03（接 `00-架构总览.md`，与 `04-场景分块-chunk.md`、`05-地形系统-terrain.md` 并列）。

---

## 1. 模块定位

`resmgr` 是 BigWorld 引擎最底层的库之一，它对上层屏蔽了“资源从哪里来”这一细节。它解决三件事：

1. **资源寻址**：把一个相对资源 ID（如 `entities.xml`、`spaces/abc/chunks/oooo00o.chunk`）解析到具体的字节流，可以来自磁盘、ZIP 包、网络缓存目录，甚至多种来源叠加。
2. **结构化读写**：把任意二进制 / XML / 目录树统一抽象为 `DataSection` 树，让上层代码使用同一套 `readString` / `openSection` / `writeVector3` 接口读写任意格式资源。
3. **缓存与监控**：通过 `DataSectionCache`（LRU）、`ResourceCache`（按对象指针注册）和 `AccessMonitor`（文件访问统计）来减少重复加载与定位热点资源。

模块整体对外接口极简，绝大多数代码只接触 `BWResource` 静态方法和 `DataSectionPtr` 这两个抽象。

### 1.1 入口与 singleton

入口类为 `BWResource`，声明于 [bwresource.hpp](file:///workspace/src/lib/resmgr/bwresource.hpp)。它继承自 `Singleton<BWResource>`，整个进程只有一份实例：

```cpp
class BWResource : public Singleton< BWResource >
{
public:
    BWResource();
    ~BWResource();

    static bool                 init( int argc, const char * argv[] );
    static bool                 init( const std::string& fullPath, bool addAsPath = true );
    static void                 fini();

    void                        refreshPaths();

    /// Returns the root section
    DataSectionPtr              rootSection();
    MultiFileSystemPtr          fileSystem();

    /// Remove a resource from the cache.
    void                        purge( const std::string& resourceID,
                                        bool recurse = false,
                                        DataSectionPtr spSection = NULL );
    void                        purgeAll();

    /// Saves a section.
    bool                        save( const std::string & resourceID );

    /// Shortcut method for opening a section
    static DataSectionPtr       openSection( const std::string & resourceID,
                                    bool makeNewSection = false,
                                    DataSectionCreator* creator = NULL );
    // ...
};
```

要点：

- `BWResource::init()` 解析 `BW_RES_PATH` 环境变量（路径分隔符在 Windows 上为 `;`，Linux/Unix 为 `:`，参见宏 `BW_RES_PATH_SEPARATOR`），逐个挂载到内部 `MultiFileSystem`。
- `BWResource::instance()` 由 `Singleton<>` 模板提供；同时 `BWResource` 允许多实例并存，便于“多线程各自一份资源上下文”的高级用法（说明文档原文）。
- `BWResource::openSection()` 是**唯一**的对外读入口，绝大多数上层代码以此为唯一接触点。
- `BWResolver` 是历史遗留的 helper（前 `FilenameResolver`），仅在 `EDITOR_ENABLED` 下暴露 `findFile` / `resolveFilename` 等绝对路径 API；新代码不应直接使用。

### 1.2 与 `BWResolver` 的关系

[bwresource.hpp](file:///workspace/src/lib/resmgr/bwresource.hpp) 文件末尾保留了 `BWResolver` 类：

```cpp
class BWResolver
{
public:
    static const std::string    findFile( const std::string& file );
    static const std::string    removeDrive( const std::string& file );
    static void                 defaultDrive( const std::string& drive );
    static const std::string    dissolveFilename( const std::string& file );
    static const std::string    resolveFilename( const std::string& file );
};
```

注释明确指出：新代码不应直接使用 `BWResolver`，因为它破坏了“相对路径 / 文件系统抽象”的封装，仅在编辑器 / 工具场景下使用。运行时一律走 `BWResource`。

---

## 2. 模块依赖关系

```text
                    ┌──────────────────────────────────────────────┐
                    │                上层模块                       │
                    │  chunk  terrain  moo  model  entitydef  ...  │
                    └───────────────────┬──────────────────────────┘
                                        │ 仅依赖 BWResource + DataSectionPtr
                                        ▼
                ┌───────────────────────────────────────────────────┐
                │                   resmgr 模块                    │
                │                                                   │
                │   BWResource (singleton, 入口)                   │
                │      │                                            │
                │      ├──► DataSection  (抽象接口 + DataSectionPtr)│
                │      │       ▲                                      │
                │      │       ├── XMLSection     (XML 文本)        │
                │      │       ├── BinSection      (二进制包装)     │
                │      │       ├── PackedSection   (自有 packed 格式)│
                │      │       ├── ZipSection     (zip 条目)        │
                │      │       └── DirSection     (目录树)          │
                │      │                                            │
                │      ├──► MultiFileSystem (文件系统叠加栈)        │
                │      │       ▲                                      │
                │      │       ├── WinFileSystem / UnixFileSystem   │
                │      │       ├── ZipFileSystem                    │
                │      │       └── CacheFileSystem                 │
                │      │                                            │
                │      ├──► DataSectionCache (LRU)                  │
                │      ├──► ResourceCache    (按指针注册)          │
                │      ├──► AccessMonitor    (访问统计)             │
                │      └──► DataSectionCensus (命名 section 普查)   │
                │                                                   │
                │   二进制层:  BinaryBlock (引用计数内存块)        │
                │              PrimitiveFile (.primitive 解析)      │
                │   工具层:    bdiff / bundiff / quick_file_writer  │
                │              sanitise_helper / xml_special_chars │
                │              filename_case_checker                │
                │   配置层:    auto_config / string_provider        │
                └───────────────────────────────────────────────────┘
                                        │
                                        ▼
                            cstdmf (SmartPointer, Singleton, Concurrency)
                            math (Vector/Matrix)
```

依赖关系：

- `resmgr` 是引擎最底层库之一，仅依赖 `cstdmf`（智能指针 / 单例 / 互斥锁 / 调试）、`math`（Vector/Matrix 用于 DataSection 类型转换）。
- 上层一切模块都通过 `BWResource::openSection()` 间接使用 resmgr。

---

## 3. `DataSection` 抽象层

### 3.1 抽象接口

[datasection.hpp](file:///workspace/src/lib/resmgr/datasection.hpp) 定义了 `DataSection` 抽象基类与 `DataSectionPtr`：

```cpp
class DataSection;
typedef SmartPointer<DataSection> DataSectionPtr;
```

`DataSection` 继承自 `SafeReferenceCount`（线程安全的引用计数），不可拷贝。它定义了所有子类必须实现的纯虚接口：

| 接口分组 | 关键方法 | 说明 |
| --- | --- | --- |
| 子节点访问 | `countChildren()` / `openChild(int)` / `childSectionName(int)` | 按索引遍历直接子节点 |
| 子节点编辑 | `newSection(tag, creator)` / `findChild(tag, creator)` / `delChild(...)` / `delChildren()` | 增删查 |
| 元信息 | `sectionName()` / `bytes()` / `save()` / `setParent()` | 名字 / 占用 / 持久化 / 反向引用 |
| 路径访问 | `openSection(tagPath, makeNewSection, creator)` 等 | 支持 `A/B/C` 这种斜杠路径，自动按层下钻 |
| 类型化读取 | `asBool/asInt/asFloat/asString/asVector3/asMatrix34/asBinary/asBlob` … | 自带默认值 |
| 类型化写入 | `setBool/setInt/.../setBinary` | 对应 set 系列 |
| 路径类型读写 | `readBool/readInt/.../readVector3(tagPath, defaultVal)` | `openSection + as*` 的快捷组合 |
| 数组读写 | `readBools/readInts/readVector3s/readMatrix34s` | 同名子节点的批量读取 |
| 迭代器 | `begin()` / `end()` / `SearchIterator` | STL 风格遍历 |

接口的几个值得注意的设计点：

1. **类型自动转换模板**：`as<C>()` 通过 `DataSectionTypeConverter<C>` 调度到 `asBool / asInt / asFloat / asString / asVector3 / asMatrix34 / asBinary` 等具体方法；内置类型用 `BUILTIN_TEMPLATES` 宏批量特化：

   ```cpp
   BUILTIN_TEMPLATES( bool,    asBool,    setBool )
   BUILTIN_TEMPLATES( int,     asInt,     setInt )
   BUILTIN_TEMPLATES( float,   asFloat,   setFloat )
   BUILTIN_TEMPLATES( std::string, asString, setString )
   BUILTIN_TEMPLATES( Vector3, asVector3, setVector3 )
   BUILTIN_TEMPLATES( Matrix, asMatrix34, setMatrix34 )
   // ...
   ```

2. **`DataSectionCreator`** 工厂接口允许在 `openSection` 时指定具体生成哪种 Section，例如把 XML 文件以 `BinSection` 形式打开：

   ```cpp
   DataSectionPtr pDS = BWResource::openSection( "existing_file.xml", false,
                                                 BinSection::creator() );
   ```

3. **`SERIALISE` / `SERIALISE_ENUM` / `SERIALISE_TO_FUNCTION`** 宏配合 `read*` / `write*`，用于把成员变量与 DataSection 双向同步（详见 [datasection.hpp](file:///workspace/src/lib/resmgr/datasection.hpp) 文件末尾）。

### 3.2 五种 Section 实现

| 实现 | 文件 | 文件类型 | 用途 |
| --- | --- | --- | --- |
| `XMLSection` | [xml_section.hpp](file:///workspace/src/lib/resmgr/xml_section.hpp) | `.xml` / `.chunk`（源数据） / `.def` / `.model` / `.sound` 等文本格式 | 编辑器与工具的首选；运行时也可读 |
| `BinSection` | [bin_section.hpp](file:///workspace/src/lib/resmgr/bin_section.hpp) | 任意二进制（包装 `BinaryPtr`） | 把一段 `BinaryBlock` 当作 DataSection 使用；延迟 introspect 子节点 |
| `PackedSection` | [packed_section.hpp](file:///workspace/src/lib/resmgr/packed_section.hpp) | `.chunk` / `.visual` / `.terrain` / `.primitive` 等 BigWorld 自有二进制 | 运行时主力格式，`res_packer` 把 XML 转换成 packed |
| `ZipSection` | [zip_section.hpp](file:///workspace/src/lib/resmgr/zip_section.hpp) | `.zip` 内条目 | 把整个 zip 文件作为一个“目录树”型 Section 暴露 |
| `DirSection` | [dir_section.hpp](file:///workspace/src/lib/resmgr/dir_section.hpp) | 真实目录 | 把磁盘目录映射为 DataSection 树 |

#### 3.2.1 XMLSection

[xml_section.hpp](file:///workspace/src/lib/resmgr/xml_section.hpp) 实现：

```cpp
class XMLSection : public DataSection
{
public:
    XMLSection( const std::string & tag );
    XMLSection( const char * tag, bool cstrToken );

    static XMLSectionPtr createFromFile( const char * filename,
            bool returnEmptySection = true );
    static XMLSectionPtr createFromStream(
        const std::string & tag, std::istream& astream );
    static XMLSectionPtr createFromBinary(
        const std::string & tag, BinaryPtr pBinary );
    // ...
private:
    typedef std::vector< XMLSectionPtr > Children;

    const char      * ctag_;
    const char      * cval_;
    std::string     tag_;
    std::string     value_;

    Children        children_;
    DataSectionPtr  parent_;
    BinaryPtr       block_;
};
```

特点：

- 子节点用 `std::vector<XMLSectionPtr>` 存储，所有 `XMLSection` 通过 `XMLResource` 共享底层 XML 节点，互相联动。
- 提供 `sanitise` / `unsanitise` 处理 XML 不合法字符；提供 `decodeWideString` / `encodeWideString` 处理宽字符串。
- 支持从文件 / 流 / 二进制块三种方式构造。

#### 3.2.2 BinSection

[bin_section.hpp](file:///workspace/src/lib/resmgr/bin_section.hpp) 包装一个 `BinaryPtr`：

```cpp
class BinSection : public DataSection
{
public:
    BinSection( const std::string& tag, BinaryPtr pBinary );
    // ...
    BinaryPtr asBinary();
    bool setBinary(BinaryPtr pBinary);
    virtual bool canPack() const    { return false; }
    virtual DataSectionPtr convertToZip(...);
private:
    std::string     tag_;
    BinaryPtr        binaryData_;
    typedef std::vector< DataSectionPtr > Children;
    bool            introspected_;
    Children        children_;
    DataSectionPtr  parent_;
};
```

关键设计是**惰性 introspect**：构造时不解析内容，只有访问子节点时才把二进制按 PackedSection / XMLSection 等规则展开（`introspect()` 私有方法）。`canPack()` 返回 false，表示无法被 `res_packer` 进一步压缩。

#### 3.2.3 PackedSection（重点）

[packed_section.hpp](file:///workspace/src/lib/resmgr/packed_section.hpp) 是运行时最常用的二进制 Section 实现。它定义了 BigWorld 自有的“紧凑二进制”格式：

```cpp
namespace PackedSectionData
{
    typedef uint8 VersionType;
    typedef int16 NumChildrenType;
    typedef int32 DataPosType;
    typedef int16 KeyPosType;
}

enum SectionType
{
    DATA_POS_MASK       = 0x0fffffff,
    TYPE_MASK           = ~DATA_POS_MASK,

    TYPE_DATA_SECTION   = 0x00000000,
    TYPE_STRING         = 0x10000000,
    TYPE_INT            = 0x20000000,
    TYPE_FLOAT          = 0x30000000, // Vector[234] 和 Matrix 也用此类型
    TYPE_BOOL           = 0x40000000,
    TYPE_BLOB           = 0x50000000,
    TYPE_ENCRYPTED_BLOB = 0x60000000,
    TYPE_ERROR          = 0x70000000,
};
```

格式要点：

1. **数据 + 索引 + 字符串表**三段式布局：一个 packed 文件由 `PackedSectionFile` 持有，包含 `StringTable`（共享 tag 名）+ 嵌套 `ChildRecord` 数组 + 子节点数据块。
2. `ChildRecord` 是 `#pragma pack(push, 1)` 的紧凑结构，包含 `dataPos_`（高 4 位是类型 `SectionType`，低 28 位是数据偏移）和 `keyPos_`（指向字符串表）：

   ```cpp
   class ChildRecord
   {
   public:
       DataPosType startPos() const       { return dataPos_ & DATA_POS_MASK; }
       DataPosType endPos() const         { return (this + 1)->startPos(); }
       SectionType type() const
           { return SectionType((this + 1)->dataPos_ & TYPE_MASK); }
   private:
       DataPosType dataPos_;
       KeyPosType  keyPos_;
   };
   ```

3. 支持的类型：`TYPE_DATA_SECTION`（子节点）/ `TYPE_STRING` / `TYPE_INT` / `TYPE_FLOAT`（同时承载 Vector2/3/4 和 Matrix34）/ `TYPE_BOOL` / `TYPE_BLOB` / `TYPE_ENCRYPTED_BLOB`（加密 blob）。
4. `PackedSection::convert()` 把 XML 文件转换为 packed，供 `res_packer` 工具使用；可以指定 `shouldEncrypt` 与 `stripRecursionLevel`。

`PackedSection` 在 `asFloat` / `asVector3` 等读取方法里直接根据 `ownDataType_` 把 `pOwnData_` 指向的字节重新解释为对应类型，零拷贝、零解析开销。

#### 3.2.4 ZipSection

[zip_section.hpp](file:///workspace/src/lib/resmgr/zip_section.hpp)：

```cpp
class ZipSection : public DataSection
{
public:
    ZipSection( const std::string& tag, FileSystemPtr parentSystem,
        std::vector<DataSectionPtr>* pChildren = NULL);
    ZipSection( ZipFileSystemPtr parent, const std::string& zipPath,
        const std::string& tag );
    // ...
private:
    std::string         tag_;
    std::string         fullPath_;
    bool                introspected_;
    mutable Children    children_;
    DataSectionPtr      parent_;
    ZipFileSystemPtr    pFileSystem_;
};
```

把整个 `.zip` 文件当作一个 DataSection 树暴露：zip 内每个条目变成一个子节点；条目内字节流再走 `BinSection` / `PackedSection` 二级 introspect。`canPack()` 同样返回 false（zip 已经压缩过）。

#### 3.2.5 DirSection

[dir_section.hpp](file:///workspace/src/lib/resmgr/dir_section.hpp)：

```cpp
class DirSection : public DataSection
{
public:
    DirSection(const std::string& path, FileSystemPtr pFileSystem);
    // ...
private:
    std::string                 fullPath_;
    std::string                 tag_;
    std::vector<std::string>    children_;
    FileSystemPtr               pFileSystem_;
    void                        addChildren();
};
```

把磁盘目录映射为 DataSection，子节点 = 目录条目。`newSection` / `delChild` 在 DirSection 中是 no-op（只读视图）。它是 `BWResource::rootSection()` 在“裸目录”挂载点上的典型实现。

### 3.3 `DataSection` 的工厂：`DataSectionCreator`

[datasection.hpp](file:///workspace/src/lib/resmgr/datasection.hpp) 还定义了 `DataSectionCreator` 接口，所有具体 Section 实现都提供一个静态 `creator()`：

```cpp
class DataSectionCreator
{
public:
    virtual DataSectionPtr create(DataSectionPtr pSection, const std::string& tag) = 0;
    virtual DataSectionPtr load(DataSectionPtr pSection, const std::string& tag,
                                BinaryPtr pBinary = NULL) = 0;
};
```

`BWResource::openSection` 的第三个参数允许调用方覆盖默认创建逻辑，例如强制以 `ZipSection` 创建新文件：

```cpp
DataSectionPtr pDS = BWResource::openSection( "new_file.zip", true,
                                             ZipSection::creator() );
```

### 3.4 `createAppropriateSection` 自动嗅探

[datasection.hpp](file:///workspace/src/lib/resmgr/datasection.hpp) 还提供静态方法 `DataSection::createAppropriateSection(tag, pData, allowModifyData, creator)`，根据二进制内容自动选择最合适的实现（如以 `<?xml` 开头 → XMLSection，否则 → PackedSection）。

---

## 4. `MultiFileSystem` 文件系统栈

### 4.1 `IFileSystem` 抽象接口

[file_system.hpp](file:///workspace/src/lib/resmgr/file_system.hpp) 定义了所有文件系统实现的统一接口：

```cpp
class IFileSystem : public SafeReferenceCount
{
public:
    typedef std::vector<std::string> Directory;
    enum FileType { FT_NOT_FOUND, FT_DIRECTORY, FT_FILE, FT_ARCHIVE };

    struct FileInfo
    {
        uint64 size; uint64 created; uint64 modified; uint64 accessed;
    };

    virtual FileType    getFileType( const std::string & path, FileInfo * pFI = NULL ) const = 0;
    virtual Directory   readDirectory( const std::string & path ) = 0;
    virtual BinaryPtr   readFile( const std::string & path ) = 0;
    virtual bool        makeDirectory( const std::string & path ) = 0;
    virtual bool        writeFile( const std::string & path, BinaryPtr pData, bool binary ) = 0;
    virtual bool        moveFileOrDirectory( const std::string & oldPath, const std::string & newPath ) = 0;
    virtual bool        eraseFileOrDirectory( const std::string & path ) = 0;
    virtual FILE *      posixFileOpen( const std::string & path, const char * mode );
    virtual std::string getAbsolutePath( const std::string& path ) const = 0;
    virtual std::string correctCaseOfPath(const std::string &path) const;
    virtual FileSystemPtr clone() = 0;
    // ...
};
typedef SmartPointer<IFileSystem> FileSystemPtr;
```

文件类型枚举：`FT_NOT_FOUND` / `FT_DIRECTORY` / `FT_FILE` / `FT_ARCHIVE`（zip 条目）。`ZIP_MAGIC = 0x4b50` 用于嗅探 zip 文件。

### 4.2 `MultiFileSystem` 叠加栈

[multi_file_system.hpp](file:///workspace/src/lib/resmgr/multi_file_system.hpp) 是叠加文件系统的核心：

```cpp
class MultiFileSystem : public IFileSystem
{
public:
    void                addBaseFileSystem( FileSystemPtr pFileSystem, int index = -1 );
    void                delBaseFileSystem( int index );

    virtual FileType    getFileType( const std::string& path, FileInfo * pFI = NULL ) const;
    virtual Directory   readDirectory(const std::string& path);
    virtual BinaryPtr   readFile(const std::string& path);
    virtual void        collateFiles(const std::string& path, std::vector<BinaryPtr>& ret);
    virtual bool        writeFile(const std::string& path, BinaryPtr pData, bool binary);
    // ...
private:
    typedef std::vector<FileSystemPtr> FileSystemVector;
    FileSystemVector    baseFileSystems_;
};
```

工作原理：

- `addBaseFileSystem` 按“后加优先级更高”的顺序入栈（除非 `index` 显式指定）。
- `readFile(path)`：从栈顶往栈底找，**第一个**返回非 NULL `BinaryPtr` 的栈层胜出。
- `readDirectory(path)` / `collateFiles(path)`：**合并**所有栈层结果（去重）。
- `writeFile(path, data, binary)`：写到栈顶可写的层。
- `getFileType(path)`：从栈顶往栈底找，第一个不是 `FT_NOT_FOUND` 的结果胜出。

`BWResource` 内部持有一个 `MultiFileSystem`，通过 `BWResource::fileSystem()` 暴露；`BW_RES_PATH` 中的每个路径都被 `addBaseFileSystem` 依次压入。

### 4.3 文件系统实现

| 实现 | 文件 | 平台 / 用途 |
| --- | --- | --- |
| `NativeFileSystem::create(path)` | [file_system.hpp](file:///workspace/src/lib/resmgr/file_system.hpp) | 工厂方法，根据编译目标自动选 `WinFileSystem` 或 `UnixFileSystem` |
| `WinFileSystem` | [win_file_system.hpp](file:///workspace/src/lib/resmgr/win_file_system.hpp) | Windows 平台 native FS，带 `FilenameCaseChecker` 处理大小写 |
| `UnixFileSystem` | [unix_file_system.hpp](file:///workspace/src/lib/resmgr/unix_file_system.hpp) | Linux/Unix 平台 native FS |
| `ZipFileSystem` | [zip_file_system.hpp](file:///workspace/src/lib/resmgr/zip_file_system.hpp) | 读取 zip 包作为只读文件系统（最大支持 2GB，常量 `MAX_ZIP_FILE_KBYTES`） |
| `CacheFileSystem` | [cache_file_system.hpp](file:///workspace/src/lib/resmgr/cache_file_system.hpp) | 把读到的文件缓存到目标目录，配合 `ignoredPrefix/Suffix` / `replaces` 配置 |

#### 4.3.1 ZipFileSystem

[zip_file_system.hpp](file:///workspace/src/lib/resmgr/zip_file_system.hpp) 内嵌完整的 zip 解析逻辑，包含两个 `#pragma pack(1)` 紧凑结构：

```cpp
struct LocalHeader {
    uint32 signature PACKED;
    uint16 extractorVersion PACKED;
    uint16 mask PACKED;
    uint16 compressionMethod PACKED;
    uint16 modifiedTime PACKED;
    uint16 modifiedDate PACKED;
    uint32 crc32 PACKED;
    uint32 compressedSize PACKED;
    uint32 uncompressedSize PACKED;
    uint16 filenameLength PACKED;
    uint16 extraFieldLength PACKED;
};

struct DirEntry {
    uint32 signature PACKED;
    // ... 完整的 zip 中心目录条目 ...
    int32 localHeaderOffset PACKED;
};
```

它维护 `FileMap fileMap_`（路径 → 索引对）和 `DirMap dirMap_`（目录 → 文件列表），实现 O(log n) 的路径查找。`FileHandleHolder` 内部类做 RAII 关闭，编辑器模式下每次都关闭以避免占用 `bwlockd` 的 cdata 文件。

#### 4.3.2 CacheFileSystem

[cache_file_system.hpp](file:///workspace/src/lib/resmgr/cache_file_system.hpp) 用于“读取时自动落到本地缓存目录”的场景：

```cpp
class CacheFileSystem : public IFileSystem
{
private:
    MultiFileSystemPtr          fileSystem_;
    std::string                 targetPath_;
    typedef std::vector<std::string> IgnoredSections;
    typedef std::map<std::string, IgnoredSections> IgnoredItems;
    typedef std::set<std::string> IncludedItems;
    typedef std::map<std::string, std::string> Replaces;

    IgnoredItems    ignoredPrefix_;
    IgnoredItems    ignoredSuffix_;
    IncludedItems   includedItems_;
    Replaces        replaces_;
};
```

支持 `ignoredPrefix` / `ignoredSuffix` / `replaces` 三种过滤规则，便于把“关键资源缓存到本地、其他资源直读”。`postResouceInitialised()` 时自动调用 `cacheResourcesXML()` 把所有 XML 资源预热到缓存目录。

#### 4.3.3 WinFileSystem 的大小写处理

[win_file_system.hpp](file:///workspace/src/lib/resmgr/win_file_system.hpp) 集成了 `FilenameCaseChecker`：

```cpp
class WinFileSystem : public IFileSystem
{
private:
    std::string                 basePath_;
    mutable FilenameCaseChecker caseChecker_;
#if ENABLE_FILE_CASE_CHECKING
    bool                        checkCase_;
    SimpleMutex                 pathCacheMutex_;
    std::set<std::string>       validPathCache_;
    std::set<std::string>       invalidPathCache_;
#endif
    static long callsTo_getFileType_;
    static long callsTo_readDirectory_;
    // ...
};
```

Windows 文件系统大小写不敏感但保留大小写，跨平台构建时极易出现“打包时小写、引用时大写”的错误。`FilenameCaseChecker` 维护两个 `std::set` 缓存（valid / invalid），避免重复扫描。

---

## 5. 资源缓存

### 5.1 `DataSectionCache`（LRU）

[data_section_cache.hpp](file:///workspace/src/lib/resmgr/data_section_cache.hpp) 是按“资源 ID → DataSectionPtr”的 LRU 缓存：

```cpp
class DataSectionCache
{
public:
    static DataSectionCache* instance();
    DataSectionCache* setSize( int maxBytes );

    void                add( const std::string & name, DataSectionPtr dataSection );
    DataSectionPtr       find( const std::string & name );
    void                remove( const std::string & name );
    void                clear();
    void                dumpCacheState();
private:
    struct CacheNode
    {
        std::string     path_;
        DataSectionPtr  dataSection_;
        int             bytes_;
        CacheNode*      prev_;
        CacheNode*      next_;
    };
    typedef std::map<std::string, CacheNode*> DataSectionMap;
    DataSectionMap    map_;
    static int         s_maxBytes_;
    static int         s_currentBytes_;
    CacheNode*         cacheHead_;
    CacheNode*         cacheTail_;
    static int         s_hits_;
    static int         s_misses_;
    SimpleMutex        accessControl_;
    // ...
};
```

特点：

- 双向链表 + map 实现 LRU；命中后 `moveToHead(pNode)`，超容量时 `purgeLRU()`。
- `SimpleMutex accessControl_` 保证线程安全（可被后台加载线程与主线程并发访问）。
- `BWResource::purge(resourceID)` 和 `BWResource::purgeAll()` 走的就是这个缓存。

### 5.2 `ResourceCache`（按对象指针注册）

[resource_cache.hpp](file:///workspace/src/lib/resmgr/resource_cache.hpp) 是另一种缓存机制，按“对象指针”注册：

```cpp
class CachedResource
{
    void* key_;
public:
    CachedResource( void* key = NULL );
    virtual void init(){};
    virtual void fini(){};
    virtual ~CachedResource();
};

template<typename SP>
class SmartPointerCache : public CachedResource
{
    SP sp_;
public:
    SmartPointerCache( SP sp ) : CachedResource( sp.getObject() ), sp_( sp ) {}
    virtual void fini() { delete this; }
};

class ResourceCache
{
    std::map<void*,CachedResourcePtr> resources_;
    bool inited_;
public:
    static ResourceCache& instance();
    void registerResource( void* key, CachedResourcePtr resource );
    void unregisterResource( void* key );
    template<typename SP> void addResource( SP sp );
    void init();
    void fini();
};
```

特点：

- 任何 `SmartPointer<T>` 在构造时会通过 `CachedResource` 构造函数自动注册到 `ResourceCache::instance()`，析构时自动注销。
- `init()` / `fini()` 控制所有缓存资源的统一生命周期（例如 D3D 设备丢失时全部 fini，恢复时全部 init），这是 BigWorld 处理 D3D 设备丢失的关键机制。
- `SmartPointerCache<SP>` 模板包装智能指针，方便把任意 `SmartPointer<T>` 注册进去。

### 5.3 `AccessMonitor`（访问统计）

[access_monitor.hpp](file:///workspace/src/lib/resmgr/access_monitor.hpp)：

```cpp
class AccessMonitor
{
public:
    void record( const std::string &fileName );
    void active( bool flag );
    static AccessMonitor &instance();
private:
    bool active_;
};
```

- 默认 `active_ = false`，不产生开销。
- 启用后每次文件访问都通过 `record(fileName)` 记录，便于离线脚本统计“哪些资源被加载、加载顺序如何”。
- 是单元测试 / 性能分析工具的常用入口。

### 5.4 `DataSectionCensus`（命名 section 普查）

[data_section_census.hpp](file:///workspace/src/lib/resmgr/data_section_census.hpp)：

```cpp
namespace DataSectionCensus
{
    void store( void ** word );
    DataSectionPtr find( const std::string & id );
    DataSectionPtr add( const std::string & id, DataSectionPtr pSect );
    void del( DataSection * pSect );
    void clear();
    void fini();
}
```

- 维护所有“有名字”的 DataSection 的弱引用普查表，便于按名字快速查找。
- 不持有强引用，section 被释放时自动从普查中移除。
- 主要用于编辑器中“按名字定位资源”的功能。

---

## 6. 二进制数据层

### 6.1 `BinaryBlock`

[binary_block.hpp](file:///workspace/src/lib/resmgr/binary_block.hpp) 是引用计数的二进制内存块：

```cpp
class BinaryBlock : public SafeReferenceCount
{
public:
    BinaryBlock( const void* data, int len, const char * allocator, BinaryPtr pOwner = 0 );
    BinaryBlock( std::istream& stream, std::streamsize len, const char * allocator );

    const void *    data() const        { return data_; }
    int             len() const         { return (int)len_; }
    BinaryPtr       pOwner()            { return pOwner_; }

    static const int RAW_COMPRESSION        =  0;
    static const int DEFAULT_COMPRESSION    =  6;
    static const int BEST_COMPRESSION       = 10;
    BinaryPtr compress(int level = DEFAULT_COMPRESSION) const;
    BinaryPtr decompress() const;
    bool isCompressed() const;
    // ...
private:
    void *          data_;
    std::streamsize len_;
    BinaryPtr       pOwner_;
    bool canZip_;
    static bool s_memoryCritical_;
};
```

要点：

- 持有 `data_` / `len_`，外加可选 `pOwner_`（指向另一 BinaryBlock，用于“视图型”BinaryBlock，零拷贝）。
- 内置 zlib 压缩 / 解压接口（`compress` / `decompress` / `isCompressed`），与 `zip/` 库配合。
- `s_memoryCritical_` 静态标志：内存吃紧时通过 `memoryCritical(true)` 通知，便于上层主动释放。
- `BinaryInputBuffer` 继承 `std::streambuf`，把 `BinaryPtr` 包装成 STL 流，便于第三方库（如 libpng、libxml）读取。

### 6.2 `PrimitiveFile`

[primitive_file.hpp](file:///workspace/src/lib/resmgr/primitive_file.hpp) 是 `.primitive` 文件的旧式解析入口：

```cpp
class PrimitiveFile : public ReferenceCount
{
public:
    static DataSectionPtr get( const std::string & resourceID );
    // ...
};
```

注释明确表示“将来会把 primitive 数据集成到资源管理器中，目前仅服务于 primitive 数据”。`get()` 接口返回 `DataSectionPtr`，即把 `.primitive` 文件包装成 `BinSection`。`#if 0` 块中保留了旧的 `readBinary` / `updateBinary` 接口（功能已迁移到 `BinSection`）。

`splitOldPrimitiveName` 和 `fetchOldPrimitivePart` 是过渡期的辅助函数，处理旧的“file.part”型 primitive 引用。

---

## 7. 工具与配置

### 7.1 工具类

| 文件 | 类 | 用途 |
| --- | --- | --- |
| [bdiff.hpp](file:///workspace/src/lib/resmgr/bdiff.hpp) / .cpp | `BDiff` | 资源二进制 diff |
| [bundiff.hpp](file:///workspace/src/lib/resmgr/bundiff.hpp) / .cpp | `BunDiff` | 资源 diff 应用 / 反向 |
| [quick_file_writer.hpp](file:///workspace/src/lib/resmgr/quick_file_writer.hpp) | `QuickFileWriter` | 栈上缓冲的快速文件写 |
| [sanitise_helper.hpp](file:///workspace/src/lib/resmgr/sanitise_helper.hpp) / .cpp | `SanitiseHelper` | XML 名字 / 字符串的合法化处理 |
| [filename_case_checker.hpp](file:///workspace/src/lib/resmgr/filename_case_checker.hpp) / .cpp | `FilenameCaseChecker` | 路径大小写检查（WinFileSystem 使用） |
| [xml_special_chars.hpp](file:///workspace/src/lib/resmgr/xml_special_chars.hpp) / .cpp | `XMLSpecialChars` | XML 特殊字符转义/反转义 |

### 7.2 配置层

| 文件 | 类 | 用途 |
| --- | --- | --- |
| [auto_config.hpp](file:///workspace/src/lib/resmgr/auto_config.hpp) / .cpp | `AutoConfig` / `AutoConfigString` | 资源路径的“配置文件自动解析”，把 `BW_RES_PATH` 中的占位符替换为真实路径 |
| [string_provider.hpp](file:///workspace/src/lib/resmgr/string_provider.hpp) / .cpp | `StringProvider` | 字符串资源接口（i18n 多语言支持） |

### 7.3 `dataresource.hpp`

[dataresource.hpp](file:///workspace/src/lib/resmgr/dataresource.hpp) 提供 `DataResource` 类——把“资源 ID + 文件系统”组合成一个可独立管理的资源句柄，`XMLSection` 等通过它访问底层文件。它与 `XMLSection` 配合实现 XML 节点级别的共享（同一文件被多个 XMLSection 引用时，只解析一次）。

---

## 8. 典型调用流程

### 8.1 读取 `entities.xml`

```text
上层: BWResource::openSection("entities.xml")
      │
      ▼
BWResource::openSection (静态方法)
      │
      ├──► DataSectionCache::find("entities.xml")    # 先查 LRU
      │       │
      │       └──► 命中? 直接返回 DataSectionPtr
      │
      └──► 未命中:
              │
              ▼
          BWResource::rootSection()->openSection("entities.xml", false, NULL)
              │
              ▼
          DataSection::openSection (基类方法)
              │
              ├──► splitTagPath: 拆 "A/B/C" 路径，逐层 openChild
              │
              └──► findChild("entities")  ──► DirSection / PackedSection 的具体实现
                      │
                      ▼
                  DataSection::createAppropriateSection(tag, BinaryPtr)
                      │
                      ├──► 嗅探: '<?xml' 开头 → XMLSection::creator()
                      │    否则               → PackedSection::creator()
                      │
                      ▼
                  MultiFileSystem::readFile("entities.xml")
                      │
                      ├──► WinFileSystem::readFile  (栈顶优先)
                      ├──► ZipFileSystem::readFile  (zip 内查找)
                      └──► ... 返回 BinaryPtr
                      │
                      ▼
                  XMLSection::createFromBinary(tag, pBinary)
                      │
                      ▼
              DataSectionCache::add("entities.xml", pSection)  # 入 LRU
              AccessMonitor::record("entities.xml")             # 可选统计
                      │
                      ▼
                  返回 DataSectionPtr
```

### 8.2 子节点遍历

```cpp
DataSectionPtr pEntities = BWResource::openSection( "entities.xml" );
if (pEntities)
{
    DataSection::iterator it  = pEntities->begin();
    DataSection::iterator end = pEntities->end();
    for (; it != end; ++it)
    {
        DataSectionPtr pEntity = *it;
        std::string    type    = pEntity->readString( "type" );
        Vector3        pos     = pEntity->readVector3( "position" );
        // ...
    }

    // 也可用 SearchIterator 按 tag 过滤
    for (DataSection::SearchIterator sit = pEntities->beginSearch("Properties");
         sit != DataSection::endOfSearch(); ++sit)
    {
        DataSectionPtr pProp = *sit;
        // ...
    }
}
```

### 8.3 类型化写入并保存

```cpp
DataSectionPtr pConfig = BWResource::openSection( "config.xml", true /*makeNewSection*/ );
pConfig->writeString ( "name",     "demo"      );
pConfig->writeInt    ( "maxPlayers", 1000       );
pConfig->writeVector3( "spawnPoint", Vector3(1,2,3) );
pConfig->writeBool   ( "enablePVP", true        );
pConfig->save();     // 落盘
BWResource::purge( "config.xml" );   // 让下次读取看到最新内容
```

### 8.4 跨文件系统叠加查找

```text
BW_RES_PATH = "patches.zip;data.zip;data/"

读取 "spaces/abc/chunks/oooo00o.chunk":
  MultiFileSystem::readFile
   ├── (栈顶) CacheFileSystem  → 未命中本地缓存
   ├── ZipFileSystem(patches.zip) → 命中,返回 BinaryPtr   ✅
   ├── ZipFileSystem(data.zip)    → (不会到达)
   └── WinFileSystem("data/")     → (不会到达)
```

### 8.5 PackedSection 的零拷贝读取

```cpp
DataSectionPtr pChunk = BWResource::openSection( "spaces/abc/chunks/oooo00o.chunk" );
// pChunk 实际类型是 PackedSection
float x = pChunk->readFloat( "transform/row0/x" );  // 直接 reinterpret_cast(pOwnData_)
Vector3 v = pChunk->readVector3( "boundingBox/min" ); // 12 字节直读
```

`PackedSection` 在 `asFloat` / `asVector3` / `asMatrix34` 中**不**做任何解析或转换，直接把 `pOwnData_` 指向的字节按对应类型重新解释。这是 BigWorld 在运行时大量使用 packed 格式的主要原因。

---

## 9. 单元测试

[unit_test/](file:///workspace/src/lib/resmgr/unit_test/) 目录下包含完整的单元测试，覆盖关键功能：

| 测试文件 | 覆盖 |
| --- | --- |
| [test_bwresource.cpp](file:///workspace/src/lib/resmgr/unit_test/test_bwresource.cpp) | `BWResource::init` / `openSection` / 路径解析 |
| [test_data_resource.cpp](file:///workspace/src/lib/resmgr/unit_test/test_data_resource.cpp) | `DataResource` 的资源 ID 解析 |
| [test_file_system.cpp](file:///workspace/src/lib/resmgr/unit_test/test_file_system.cpp) | `MultiFileSystem` / `WinFileSystem` / `ZipFileSystem` |
| [test_packed_section.cpp](file:///workspace/src/lib/resmgr/unit_test/test_packed_section.cpp) | `PackedSection` 的 round-trip（XML → packed → XML） |
| [test_zip_section.cpp](file:///workspace/src/lib/resmgr/unit_test/test_zip_section.cpp) | `ZipSection` 的 zip 内读写 |
| [test_harness.hpp](file:///workspace/src/lib/resmgr/unit_test/test_harness.hpp) | 测试公用脚手架 |

入口在 [main.cpp](file:///workspace/src/lib/resmgr/unit_test/main.cpp)，用 CppUnit 风格的宏组织。工程文件包括 `resmgr_unit_test2005.vcproj` 和 `resmgr_unit_test2008.vcproj`，Linux 下用 Makefile。

---

## 10. 与其他模块的协作

| 上游模块 | 协作方式 |
| --- | --- |
| `chunk` | 所有 `.chunk` / `.chunkb` 文件经 `BWResource::openSection` 加载，运行时落到 `PackedSection`；详见 [04-场景分块-chunk.md](file:///workspace/study-docs/04-场景分块-chunk.md) |
| `terrain` | `.terrain` / `.cdata` / 高度图 / 法线图等同样经 `BWResource` 加载；详见 [05-地形系统-terrain.md](file:///workspace/study-docs/05-地形系统-terrain.md) |
| `moo` | `.visual` / `.primitive` / `.mfm` / `.tex` 等 GPU 资源加载，依赖 `PackedSection` 与 `PrimitiveFile` |
| `model` | `.model` / `.mfo` 文件由 `BWResource` 拿到 `DataSectionPtr`，再传给 `SuperModel` 解析 |
| `entitydef` | `entities.xml` / `*.def` 走 `XMLSection`，编辑器写入也用同一接口 |
| `cstdmf/bgtask_manager` | 后台线程加载 chunk 时，所有 `BWResource::openSection` 都通过 `DataSectionCache::accessControl_` 串行化关键路径 |
| `auto_config` | 启动时解析 `paths.xml` 等配置文件，把 `BW_RES_PATH` 中的占位符（如 `{USER_DIR}`）替换成实际路径 |

---

## 11. 设计要点小结

1. **统一抽象**：`DataSection` + `DataSectionPtr` 把 XML、二进制、zip、目录四种存储形式统一成同一套接口，上层代码不感知格式差异。
2. **零拷贝 packed**：`PackedSection` 通过 `SectionType` + `dataPos_` 高 4 位标识类型，运行时直接 reinterpret_cast，避免解析开销。
3. **栈式文件系统**：`MultiFileSystem` 让“补丁包 + 主资源包 + 本地目录”可叠加，热补丁、A/B test、用户 mod 都可热插拔。
4. **多级缓存**：`DataSectionCache`（LRU）+ `ResourceCache`（按对象指针 + init/fini 生命周期）+ `AccessMonitor`（统计）+ `DataSectionCensus`（命名普查），覆盖热路径、设备丢失、统计、查找四种需求。
5. **跨平台 / 大小写友好**：`WinFileSystem` 集成 `FilenameCaseChecker`，让 Windows 平台开发的内容能安全部署到 Linux 服务器。
6. **编辑器扩展**：`BWResolver` / `BWResource::findFile` 等绝对路径 API 仅在 `EDITOR_ENABLED` 下暴露，避免运行时代码意外破坏文件系统抽象。
7. **资源 diff 支持**：`bdiff` / `bundiff` 提供资源 diff 工具，便于增量更新与版本对比。
