# BigWorld Engine 2.0.1 实体定义层（entitydef）

> 源码位置：`src/lib/entitydef/`
> 模块定位：BigWorld 的 **Entity 反射元数据系统**，是所有跨进程同步、序列化、方法调用的基石。
> 类比：它同时扮演了 **IDL（接口描述语言）+ ORM（对象关系映射）+ 序列化框架** 三重角色，是 MMO 引擎版的“类型系统 + RPC 元数据”。

---

## 1. 模块定位与核心职责

`entitydef` 是 BigWorld 引擎中最核心的子系统之一。在 BigWorld 的分布式架构里，一个 Entity（玩家、NPC、道具、投射物等）的实例可能同时存在于多个进程：

- **cellapp**：模拟实体在空间中的真实状态（位置、AOI、物理）
- **baseapp**：持有实体的持久代理与客户端连接
- **client**：仅持有“可见的”镜像
- **dbmgr**：持久化到数据库

要让这些进程对“同一个逻辑实体”有一致的认知，必须有一套**统一的、声明式的类型描述**。`entitydef` 就是这套描述系统：它在启动时解析 `scripts/entities.xml` 与每个实体类型对应的 `scripts/entity_defs/<TypeName>.def` 文件，生成一套内存中的反射元数据，供 baseapp / cellapp / dbmgr / client 共享使用。

### 1.1 三重角色

| 角色 | 说明 | 对应概念 |
| --- | --- | --- |
| IDL | 声明实体的属性（Property）与方法（Method），以及它们在哪个 Component（BASE/CELL/CLIENT）可见 | 类似 Protobuf / Thrift IDL |
| ORM | 描述属性是否持久化（`DATA_PERSISTENT`）、是否是数据库索引列（`DATA_ID`）、数据库字段长度 | 类似 Django Model / SQLAlchemy |
| 序列化框架 | 提供 `PyObject ↔ BinaryStream ↔ DataSection` 三者互转，支撑网络传输与持久化 | 类似 Boost.Serialization |

### 1.2 核心数据流

```
scripts/entities.xml  ──┐
scripts/entity_defs/   ──┼──► EntityDescriptionMap::parse() ──► EntityDescription[] ──► 各进程运行时使用
scripts/alias.xml      ──┘                                         （baseapp/cellapp/dbmgr/client）
```

---

## 2. 模块全景与文件清单

`src/lib/entitydef/` 共约 40 个源文件，按职责可分为七大组：

| 组 | 代表文件 | 职责 |
| --- | --- | --- |
| 顶层描述 | `entity_description.hpp/cpp/ipp`、`entity_description_map.hpp/cpp`、`entity_description_debug.hpp/cpp` | 单个实体类型描述 + 全局注册表 + 调试 |
| 属性描述 | `data_description.hpp/cpp/ipp`、`data_type.hpp/cpp`、`data_types.hpp/cpp` | 属性元数据 + 类型系统（内建/复合） |
| LOD | `data_lod_level.hpp/cpp/ipp` | 按细节层级切分属性可见性 |
| 属性变化 | `property_change.hpp/cpp`、`property_change_reader.hpp/cpp`、`property_event_stamps.hpp/cpp/ipp`、`property_owner.hpp/cpp` | 属性增量广播 + 时间戳 + 所有权链 |
| 方法描述 | `method_description.hpp/cpp/ipp`、`entity_method_descriptions.hpp/cpp`、`method_response.hpp/cpp`、`remote_entity_method.hpp/cpp`、`mailbox_base.hpp/cpp` | 方法元数据 + RPC 代理 + Mailbox |
| 位流 | `bit_reader.hpp/cpp`、`bit_writer.hpp/cpp` | 紧凑位级序列化 |
| Volatile / UDO / 元数据 | `volatile_info.hpp/cpp/ipp`、`py_volatile_info.hpp/cpp`、`*_user_data_object_description*.hpp/cpp`、`meta_data_type.hpp/cpp`、`entity_member_stats.hpp/cpp`、`constants.hpp` | 易失信息 / 用户数据对象 / 类型工厂 / 统计 |

### 模块依赖关系图

```
                ┌──────────────────────────────────────────────────┐
                │                 EntityDescriptionMap             │
                │   (解析 entities.xml，注册所有 EntityDescription) │
                └────────────────────────┬─────────────────────────┘
                                         │ 持有 vector<EntityDescription>
                                         ▼
        ┌────────────────────────────────────────────────────────────┐
        │                    EntityDescription                       │
        │   (继承 BaseUserDataObjectDescription)                     │
        │   属性: properties_[]  方法: cell_/base_/client_           │
        │   VolatileInfo  DataLoDLevels  压缩类型                    │
        └──────┬──────────────────┬───────────────────┬──────────────┘
               │                  │                   │
               ▼                  ▼                   ▼
        DataDescription    EntityMethodDescriptions   VolatileInfo
        (属性元数据)        (方法集合)                 (易失属性)
          │   │                │
          │   ▼                ▼
          │ DataType      MethodDescription[]
          │ (类型系统)     (方法元数据)
          │   │
          │   ├── StringDataType / UnicodeStringDataType
          │   ├── SequenceDataType (ARRAY/TUPLE)
          │   ├── ClassDataType / FixedDictDataType
          │   └── UserDataType (脚本自定义)
          ▼
        MetaDataType (类型工厂注册表)
```

---

## 3. 顶层结构

### 3.1 EntityDescription —— 单个实体类型的描述

定义于 [entity_description.hpp](file:///workspace/src/lib/entitydef/entity_description.hpp)，继承自 `BaseUserDataObjectDescription`。它是每个实体类型的“类对象”，描述该类型实体的全部属性、方法、可见域、LOD、Volatile 信息。

**关键成员**：

```cpp
class EntityDescription: public BaseUserDataObjectDescription
{
public:
    bool parse( const std::string & name, bool isFinal = true );
    void supersede( MethodDescription::Component component );

    // 数据可见域掩码，用于序列化时筛选属性
    enum DataDomain
    {
        BASE_DATA   = 0x1,
        CLIENT_DATA = 0x2,
        CELL_DATA   = 0x4,
        EXACT_MATCH = 0x8,
        ONLY_OTHER_CLIENT_DATA = 0x10,
        ONLY_PERSISTENT_DATA = 0x20
    };

    // 核心：属性与方法的“域内序列化”
    bool addSectionToStream( DataSectionPtr, BinaryOStream &, int dataDomains ) const;
    bool addDictionaryToStream( PyObject * pDict, BinaryOStream &, int dataDomains ) const;
    bool readStreamToDict( BinaryIStream &, int dataDomains, PyObject * pDest ) const;
    bool readStreamToSection( BinaryIStream &, int dataDomains, DataSectionPtr ) const;

    EntityTypeID index() const;             // 服务端类型索引
    EntityTypeID clientIndex() const;       // 客户端类型索引（可能因 ClientName 别名不同）

    bool hasCellScript() const;
    bool hasBaseScript() const;
    bool hasClientScript() const;
    bool isClientOnlyType() const;          // 既无 cell 也无 base 脚本
    bool isClientType() const;              // name_ == clientName_

    const VolatileInfo & volatileInfo() const;
    const DataLoDLevels & lodLevels() const;   // 仅 MF_SERVER

    // 方法集合（按 Component 分组）
    const EntityMethodDescriptions & cell() const;
    const EntityMethodDescriptions & base() const;
    const EntityMethodDescriptions & client() const;

    void addToMD5( MD5 & md5 ) const;          // 用于版本一致性校验
    void addPersistentPropertiesToMD5( MD5 & md5 ) const;

private:
    EntityTypeID index_;
    EntityTypeID clientIndex_;
    std::string  clientName_;
    bool hasCellScript_, hasBaseScript_, hasClientScript_;
    VolatileInfo volatileInfo_;
    BWCompressionType internalNetworkCompressionType_;   // 内网压缩
    BWCompressionType externalNetworkCompressionType_;   // 外网（到客户端）压缩
    PropertyIndices clientServerProperties_;              // 客户端-服务端属性的索引表
    EntityMethodDescriptions cell_, base_, client_;       // 三套方法集
    unsigned int numEventStampedProperties_;
    DataLoDLevels lodLevels_;                             // 仅 MF_SERVER
    std::set< std::string > tempProperties_;              // 临时属性名集合
};
```

**`isClientType()` 与 `clientIndex()` 的意义**：BigWorld 允许服务端实体名与客户端实体名不同（通过 `.def` 中的 `<ClientName>` 标签）。若二者相同，则该实体是“客户端类型”，会被分配一个独立的 `clientIndex_`；若不同，则服务端实体会别名到一个客户端类型的索引上，前提是两者的客户端可见属性数与方法数完全一致（在 `EntityDescriptionMap::parse` 中校验）。

**`SERVER_APP_HEADER` 宏的对应物**：`EntityDescription` 没有类似宏，但 `hasCellScript_/hasBaseScript_/hasClientScript_` 的判定逻辑值得关注——在服务端/编辑器下会真实检查 `.py` 文件是否存在；在客户端发行版下则**假设** cell/base 脚本存在（因为发行版不携带服务端脚本），见 [entity_description.cpp](file:///workspace/src/lib/entitydef/entity_description.cpp) 第 147–160 行。

### 3.2 EntityDescriptionMap —— 全局实体类型注册表

定义于 [entity_description_map.hpp](file:///workspace/src/lib/entitydef/entity_description_map.hpp)。启动时由它解析 `scripts/entities.xml`，建立“名字→索引”与“索引→描述”的双向映射。

```cpp
class EntityDescriptionMap
{
public:
    bool parse( DataSectionPtr pSection );                       // 解析 entities.xml
    bool nameToIndex( const std::string& name, EntityTypeID& index ) const;
    const EntityDescription& entityDescription( EntityTypeID index ) const;
    void addToMD5( MD5 & md5 ) const;                            // 仅 isClientType 参与
    void addPersistentPropertiesToMD5( MD5 & md5 ) const;
private:
    typedef std::vector<EntityDescription> DescriptionVector;
    DescriptionVector vector_;           // 索引 → 描述
    DescriptionMap map_;                 // 名字 → 索引
};
```

**解析流程**（见 [entity_description_map.cpp](file:///workspace/src/lib/entitydef/entity_description_map.cpp)）：

1. `entities.xml` 的每个子节点名即为实体类型名，调用 `desc.parse( typeName )` 打开对应的 `entity_defs/<TypeName>.def`。
2. 解析成功后 `desc.index( i )`，并写入 `map_[name] = index`。
3. 若该实体是客户端类型（`isClientType()`），分配递增的 `clientIndex`。
4. 第二轮遍历处理 `ClientName` 别名：服务端实体若声明了与自身不同的 `ClientName`，则查找该名字对应的客户端类型，校验属性数与方法数一致后复用其 `clientIndex`。
5. `checkCount()` 校验暴露给客户端的属性/方法数未超上限（高效阈值 61/62，硬上限 256 / 62×256）——这些上限来自位流编码宽度。

### 3.3 EntityDescriptionDebug —— 调试辅助

定义于 [entity_description_debug.hpp](file:///workspace/src/lib/entitydef/entity_description_debug.hpp)，仅提供两个 `dump` 重载，把单个或全部 `EntityDescription` 的属性/方法/LOD 信息打印到日志，便于排查 `.def` 解析问题。

---

## 4. 属性描述

### 4.1 DataDescription —— 单个属性的元数据

定义于 [data_description.hpp](file:///workspace/src/lib/entitydef/data_description.hpp)。每个属性（如玩家的 `hp`、`position`）都对应一个 `DataDescription`，记录其名字、类型、可见域标志、持久化、索引、LOD 层级、数据库长度等。

**核心标志枚举**（`EntityDataFlags`）：

```cpp
enum EntityDataFlags
{
    DATA_GHOSTED     = 0x01,  // 同步到 ghost（其它 cell 的镜像）
    DATA_OTHER_CLIENT= 0x02,  // 发送给其它客户端
    DATA_OWN_CLIENT  = 0x04,  // 发送给自己的客户端
    DATA_BASE        = 0x08,  // 发送给 base
    DATA_CLIENT_ONLY = 0x10,  // 仅客户端静态数据
    DATA_PERSISTENT  = 0x20,  // 持久化到数据库
    DATA_EDITOR_ONLY = 0x40,  // 仅编辑器读写
    DATA_ID          = 0x80   // 数据库索引列
};
```

这些标志在 `.def` 中通过 `<Flags>` 字符串声明，`data_description.cpp` 内的 `s_entityDataFlagMappings` 表完成字符串到标志位的映射：

| `.def` 中的 Flags 字符串 | 对应标志位 | 含义 |
| --- | --- | --- |
| `CELL_PRIVATE` | 0 | 仅本 cell 可见，不同步 |
| `CELL_PUBLIC` | `DATA_GHOSTED` | 同步到 ghost |
| `OTHER_CLIENTS` | `DATA_GHOSTED\|DATA_OTHER_CLIENT` | ghost + 广播给其它客户端 |
| `OWN_CLIENT` | `DATA_OWN_CLIENT` | 仅发给自己客户端 |
| `BASE` | `DATA_BASE` | 发送给 base |
| `BASE_AND_CLIENT` | `DATA_OWN_CLIENT\|DATA_BASE` | base + 自己客户端 |
| `CELL_PUBLIC_AND_OWN` | `DATA_GHOSTED\|DATA_OWN_CLIENT` | ghost + 自己客户端 |
| `ALL_CLIENTS` | `DATA_GHOSTED\|DATA_OTHER_CLIENT\|DATA_OWN_CLIENT` | ghost + 所有客户端 |
| `EDITOR_ONLY` | `DATA_EDITOR_ONLY` | 仅编辑器 |

**关键索引字段**：

```cpp
int index_;                 // 属性在 EntityDescription::properties_ 中的全局索引
int localIndex_;            // 在本地属性值向量中的索引（去除了不可见属性）
int eventStampIndex_;       // 在事件时间戳向量中的索引
int clientServerFullIndex_; // 客户端-服务端全量索引
int detailLevel_;           // LOD 层级
int databaseLength_;        // 数据库字段长度（默认 65535）
```

**序列化三件套**（委托给 `DataType`）：

```cpp
bool addToStream( PyObject * pNewValue, BinaryOStream & stream, bool isPersistentOnly ) const;
PyObjectPtr createFromStream( BinaryIStream & stream, bool isPersistentOnly ) const;
bool addToSection( PyObject * pNewValue, DataSectionPtr pSection );
PyObjectPtr createFromSection( DataSectionPtr pSection ) const;
```

这些方法的实现（见 [data_description.ipp](file:///workspace/src/lib/entitydef/data_description.ipp)）只是简单转发给 `pDataType_->` 对应方法，真正的序列化逻辑在 `DataType` 体系内。

### 4.2 DataType / DataTypes —— 数据类型系统

#### 4.2.1 DataType 抽象基类

定义于 [data_type.hpp](file:///workspace/src/lib/entitydef/data_type.hpp)，继承自 `ReferenceCount`（引用计数智能指针管理）。它是所有具体数据类型的抽象基类，定义了“PyObject ↔ Stream ↔ Section”三组纯虚方法：

```cpp
class DataType : public ReferenceCount
{
public:
    DataType( MetaDataType * pMetaDataType, bool isConst = true );

    virtual void setDefaultValue( DataSectionPtr pSection ) = 0;
    virtual PyObjectPtr pDefaultValue() const = 0;
    virtual bool isSameType( PyObject * pValue ) = 0;

    // PyObject -> BinaryStream
    virtual bool addToStream( PyObject * pValue, BinaryOStream & stream,
        bool isPersistentOnly ) const = 0;
    // BinaryStream -> PyObject（返回 new reference）
    virtual PyObjectPtr createFromStream( BinaryIStream & stream,
        bool isPersistentOnly ) const = 0;

    // PyObject -> DataSection
    virtual bool addToSection( PyObject * pValue, DataSectionPtr pSection ) const = 0;
    // DataSection -> PyObject
    virtual PyObjectPtr createFromSection( DataSectionPtr pSection ) const = 0;

    virtual void addToMD5( MD5 & md5 ) const = 0;

    // 类型对象池：相同结构（同名同字段）的类型只创建一份
    static DataTypePtr buildDataType( DataSectionPtr pSection );
    static DataTypePtr buildDataType( const std::string& typeName );

protected:
    MetaDataType * pMetaDataType_;   // 所属的元类型（工厂）
    bool isConst_;                   // 是否不可变（影响赋值优化）
};
```

`DataType` 维护一个 `SingletonMap`（按 `operator<` 排序的指针集合），保证结构相同的类型实例全局唯一，减少内存与比较开销。还支持别名（`s_aliases_`），通过 `alias.xml` 定义类型别名。

#### 4.2.2 MetaDataType —— 类型工厂

定义于 [meta_data_type.hpp](file:///workspace/src/lib/entitydef/meta_data_type.hpp)。`MetaDataType` 是“类型的类型”，负责根据 `.def` 中的 `<Type>` 节点创建对应的 `DataType` 实例：

```cpp
class MetaDataType
{
public:
    static MetaDataType * find( const std::string & name );   // 按名查找已注册的元类型
    virtual const char * name() const = 0;
    virtual DataTypePtr getType( DataSectionPtr pSection ) = 0;
    static void addAlias( const std::string& orig, const std::string& alias );
protected:
    static void addMetaType( MetaDataType * pMetaType );
    static void delMetaType( MetaDataType * pMetaType );
private:
    static MetaDataTypes * s_metaDataTypes_;   // name -> MetaDataType* 注册表
};
```

`data_types.hpp` 提供了 `SimpleMetaDataType<DATATYPE>` 模板与 `SIMPLE_DATA_TYPE` 宏，让无参数的内建类型一行注册：

```cpp
#define SIMPLE_DATA_TYPE( TYPE, NAME )					\
	SimpleMetaDataType< TYPE > s_##NAME##_metaDataType( #NAME );
```

#### 4.2.3 内建类型与复合类型

定义于 [data_types.hpp](file:///workspace/src/lib/entitydef/data_types.hpp)，实现于 `data_types.cpp`。主要类型层级：

```
DataType (抽象)
├── StringDataType            // UTF-8 字符串
├── UnicodeStringDataType     // Unicode 字符串
├── SequenceDataType (抽象)   // 序列基类（含元素类型、容量、dbLen）
│   ├── ArrayDataType         // 变长数组（对应 Python list）
│   └── TupleDataType         // 定长元组（对应 Python tuple）
├── ClassDataType             // 命名字段集合（类似 struct）
├── FixedDictDataType         // 固定键字典（ClassDataType + 可选自定义类）
└── UserDataType              // 脚本自定义类型（Python 模块实现序列化）
```

**内建简单类型**（在 `data_types.cpp` 中通过 `SIMPLE_DATA_TYPE` 注册，未在头文件逐一声明）包括：`INT8/16/32/64`、`UINT8/16/32/64`、`FLOAT`、`DOUBLE`、`BOOL`、`STRING`、`UNICODE_STRING`、`VECTOR2/3/4`、`MATRIX`、`MAILBOX`、`ENTITY_TYPE` 等。这些类型的 `addToStream` / `createFromStream` 直接做二进制读写。

**SequenceDataType** 是数组与元组的共同基类，持有一个 `elementTypePtr_`（元素类型）和 `size_`（容量上限）。它的 `addToStreamInternal` 先写元素个数，再逐个写元素；`createFromStreamInternal` 反向操作。

**ClassDataType** 表示一组命名字段（`Fields`），每个字段有名字、类型、`dbLen_`、`isPersistent_`。它本身就是一个“迷你 EntityDescription”，支持 `fieldIndexFromName` 查找，并可作为 `PropertyOwner` 嵌套（`attach`/`detach`/`asOwner`），使字段级变更能向上传播。

**FixedDictDataType** 是 `ClassDataType` 的增强版——固定键字典，但允许用自定义 Python 类替换默认的 dict 实现（通过 `setCustomClassImplementor` 指定 `moduleName`/`instanceName`）。自定义类需实现 `getDictFromObj`、`createObjFromDict`、`isSameType`，可选实现 `addToStream` / `createFromStream`。其 C++ 侧的实例类型是 `PyFixedDictDataInstance`。

**UserDataType** 完全把序列化交给 Python 脚本：通过 `moduleName_` / `instanceName_` 定位一个 Python 模块里的实例对象，调用其 `addToStream` / `createFromStream` / `addToSection` / `createFromSection` / `defaultValue` 等方法。这给了游戏逻辑极大的灵活性（例如用 `cPickle` 序列化任意结构）。

### 4.3 DataLODLevel —— 按 LOD 切分属性可见性

定义于 [data_lod_level.hpp](file:///workspace/src/lib/entitydef/data_lod_level.hpp)，仅在 `MF_SERVER` 下编译。LOD（Level of Detail）这里不是渲染 LOD，而是**属性同步的细节层级**——离观察者越远的实体，同步的属性越少。

```cpp
class DataLoDLevel
{
public:
    float low() const;     // 优先级低于此值 → 切到更详细层级
    float high() const;    // 优先级高于此值 → 切到更粗略层级
    float start() const;   // 起始优先级
    float hyst() const;    // 滞回值（防止频繁切换）
    const std::string & label() const;
    void finalise( DataLoDLevel * pPrev, bool isLast );
    enum { OUTER_LEVEL = -2, NO_LEVEL = -1 };
};

class DataLoDLevels
{
public:
    bool addLevels( DataSectionPtr pSection );
    bool needsMoreDetail( int level, float priority ) const;
    bool needsLessDetail( int level, float priority ) const;
private:
    DataLoDLevel level_[ MAX_DATA_LOD_LEVELS + 1 ];
    int size_;
};
```

每个 `DataDescription` 有一个 `detailLevel_`，标记它属于哪个 LOD 层级。`EntityDescription::parse` 末尾会把 LOD 层级定义的乱序索引统一转换为实际层级序号（见 `entity_description.cpp` 第 169–202 行）。`OUTER_LEVEL` 表示最外层（最粗略），`NO_LEVEL` 表示不参与 LOD（总是同步）。

### 4.4 PropertyChange —— 属性变化广播

定义于 [property_change.hpp](file:///workspace/src/lib/entitydef/property_change.hpp)。当实体的某个属性（或嵌套属性）变化时，不是把整个实体重新序列化，而是产生一个轻量的 `PropertyChange`，只描述“哪条路径上的值变了”。

```cpp
class PropertyChange
{
public:
    virtual uint8 addToStream( BinaryOStream & stream,
            const PropertyOwnerBase * pOwner, int messageID ) const = 0;
    virtual PropertyChangeType type() const = 0;
    void addToPath( int index );   // 从叶子到根的路径
protected:
    typedef std::vector< int32 > ChangePath;
    ChangePath path_;   // 例如 [3,4,6] 表示 entity.myList[4][3]
};

class SinglePropertyChange : public PropertyChange;   // 单值变更
class SlicePropertyChange : public PropertyChange;     // 数组切片变更
```

**路径编码**：`path_` 是从叶子到根的索引序列。例如 `entity.myList[4][3]` 的路径是 `[3, 4, 6]`（6 是 `myList` 在 entity 中的属性索引）。`rootIndex()` 返回最靠近根的索引（即 `path_` 末尾，若无 path 则返回 `leafIndex_`）。

**变化类型**：`PROPERTY_CHANGE_TYPE_SINGLE`（单值，ID ≤ 60 可用简短编码）、`PROPERTY_CHANGE_TYPE_SLICE`（切片，ID = 61/62 是特殊标记位）。`MAX_SIMPLE_PROPERTY_CHANGE_ID = 60` 是为了用单字节高效编码“根属性索引”+“变化类型”。

`addToStream` 先写路径（`addPathToStream`），再用 `BitWriter` 写额外位（`addExtraBits`），实现紧凑编码。

### 4.5 PropertyChangeReader —— 属性变化应用

定义于 [property_change_reader.hpp](file:///workspace/src/lib/entitydef/property_change_reader.hpp)，是 `PropertyChange` 的逆操作——从流中读取一个属性变更并应用到 `PropertyOwnerBase` 树上：

```cpp
class PropertyChangeReader
{
public:
    int readAndApply( BinaryIStream & stream, PropertyOwnerBase * pOwner,
            int clientServerID,
            PyObjectPtr * ppOldValue = NULL,
            PyObjectPtr * ppChangePath = NULL );
protected:
    virtual PyObjectPtr apply( BinaryIStream & stream,
            PropertyOwnerBase * pOwner, PyObjectPtr pChangePath ) = 0;
    virtual int readExtraBits( BinaryIStream & stream ) = 0;
    virtual int readExtraBits( BitReader & reader, int leafSize ) = 0;
};

class SinglePropertyChangeReader : public PropertyChangeReader;
class SlicePropertyChangeReader : public PropertyChangeReader;
```

### 4.6 PropertyEventStamps —— 事件时间戳

定义于 [property_event_stamps.hpp](file:///workspace/src/lib/entitydef/property_event_stamps.hpp)。对于 `OTHER_CLIENT` 属性（广播给其它客户端的属性），每个属性记录最后一次变化的事件号（`EventNumber`），用于 ghost / 客户端做增量同步的“水位线”判定：

```cpp
class PropertyEventStamps
{
public:
    void init( const EntityDescription & entityDescription );
    void set( const DataDescription & dataDescription, EventNumber eventNumber );
    EventNumber get( const DataDescription & dataDescription ) const;
    void addToStream( BinaryOStream & stream ) const;
    void removeFromStream( BinaryIStream & stream );
private:
    typedef std::vector< EventNumber > Stamps;
    Stamps eventStamps_;
};
```

`EntityDescription::numEventStampedProperties()` 返回需要打时间戳的属性数，`DataDescription::eventStampIndex()` 是该属性在 `eventStamps_` 向量中的下标。

### 4.7 PropertyOwner —— 属性所有权链

定义于 [property_owner.hpp](file:///workspace/src/lib/entitydef/property_owner.hpp)。这是一个**组合模式**，让嵌套属性（数组元素、字典字段）的变更能沿着所有权链向上冒泡，直到顶层（Entity）。

```cpp
class PropertyOwnerBase
{
public:
    virtual void onOwnedPropertyChanged( PropertyChange & change );  // 子通知父
    virtual bool getTopLevelOwner( PropertyChange & change,
            PropertyOwnerBase *& rpTopLevelOwner );                  // 找到顶层
    virtual int getNumOwnedProperties() const = 0;
    virtual PropertyOwnerBase * getChildPropertyOwner( int childIndex ) const = 0;
    virtual PyObjectPtr setOwnedProperty( int childIndex, BinaryIStream & data ) = 0;
    virtual PyObjectPtr setOwnedSlice( int startIndex, int endIndex, BinaryIStream & data );
    virtual PyObjectPtr getPyIndex( int index ) const = 0;
};

class PropertyOwner : public PyObjectPlus, public PropertyOwnerBase;       // 普通 owner
class TopLevelPropertyOwner : public PropertyOwnerBase;                    // 顶层 owner（Entity）
template <class C> class PropertyOwnerLink : public TopLevelPropertyOwner; // 无虚表类的链接桥
```

`TopLevelPropertyOwner` 提供 `setPropertyFromInternalStream` / `setPropertyFromExternalStream`，分别处理来自服务端内部与来自客户端的属性变更流。`PropertyOwnerLink<C>` 是一个模板适配器，让不想用虚函数的类（通过引用 `self_`）也能接入所有权链。

---

## 5. 方法描述

### 5.1 MethodDescription —— 单个方法的元数据

定义于 [method_description.hpp](file:///workspace/src/lib/entitydef/method_description.hpp)。每个方法（如 `cell.public.remoteMethod`、`base.clientExposed.attack`）对应一个 `MethodDescription`，记录其名字、所属 Component、参数类型列表、返回值、是否暴露给客户端、优先级等。

```cpp
class MethodDescription
{
public:
    enum Component { CLIENT, CELL, BASE };

    bool parse( DataSectionPtr pSection, Component component );
    bool parseReturnValues( DataSectionPtr pSection );

    bool areValidArgs( bool exposedExplicit, PyObject * args, bool generateException ) const;
    bool addToStream( bool isFromServer, PyObject * args, BinaryOStream & stream ) const;
    bool callMethod( PyObject * self, BinaryIStream & data,
        EntityID sourceID = 0, int replyID = -1,
        const Mercury::Address* pReplyAddr = NULL,
        Mercury::NetworkInterface * pInterface = NULL ) const;

    bool isExposed() const;        // 是否可被客户端调用
    Component component() const;   // 属于哪个 Component
    MethodIndex internalIndex() const;   // 服务端内部索引
    MethodIndex exposedIndex() const;    // 客户端-服务端暴露索引
    float priority() const;
    uint returnValues() const;
    DataTypePtr returnValueType( uint index ) const;

private:
    std::string name_;
    uint8 flags_;                 // 低 2 位是 Component，第 3 位是 IS_EXPOSED
    typedef std::vector< DataTypePtr > Args;
    Args args_;                   // 参数类型列表
    typedef std::pair< std::string, DataTypePtr > ReturnValue;
    std::vector< ReturnValue > returnValues_;
    int internalIndex_;           // 服务端内部用
    int exposedIndex_;            // 客户端-服务端用
    int exposedSubIndex_;         // 扩展地址空间（sub-slot）
    float priority_;
};
```

**两套索引**：`internalIndex_` 用于服务端内部按名字查找方法；`exposedIndex_` + `exposedSubIndex_` 用于客户端-服务端协议——客户端只能调用“暴露”的方法，协议里用紧凑的整数索引而非字符串名。`exposedSubIndex_` 用于当暴露方法数超过单字节上限（62）时扩展地址空间。

**`callMethod`** 是 RPC 的核心：从 `BinaryIStream` 反序列化参数，构造 Python tuple，调用 `self.<name>(args)`。若方法有返回值且带 `replyID`，还会通过 `MethodResponse` 异步回传结果。

### 5.2 EntityMethodDescriptions —— 方法集合

定义于 [entity_method_descriptions.hpp](file:///workspace/src/lib/entity_method_descriptions.hpp)。每个 `EntityDescription` 持有三个 `EntityMethodDescriptions`（cell/base/client），分别管理各自 Component 的方法。

```cpp
class EntityMethodDescriptions
{
public:
    bool init( DataSectionPtr pMethods, MethodDescription::Component component,
        const char * interfaceName );
    void checkExposedForSubSlots();        // 检查暴露方法是否需要 sub-slot
    void checkExposedForPythonArgs( const char * interfaceName );
    void supersede();                      // 父类方法被覆盖时的处理

    unsigned int size() const;
    unsigned int exposedSize() const;
    MethodDescription * internalMethod( unsigned int index ) const;
    MethodDescription * exposedMethod( uint8 topIndex, BinaryIStream & data ) const;
    MethodDescription * find( const std::string & name ) const;
private:
    typedef std::map< std::string, uint32 > Map;
    Map map_;                              // 名字 -> internalMethods_ 索引
    MethodDescriptionList internalMethods_;
    std::vector< unsigned int > exposedMethods_;   // 暴露方法的索引表
};
```

`exposedMethod( topIndex, data )` 是客户端 RPC 的入口：`topIndex` 是协议里的暴露索引，`data` 流中可能还携带 `subIndex`（当暴露方法过多时）。`checkExposedForSubSlots` 会把超过 62 个的暴露方法分组到 sub-slot。

### 5.3 MethodResponse —— 异步返回值载体

定义于 [method_response.hpp](file:///workspace/src/lib/entitydef/method_response.hpp)。当一个暴露方法需要返回值时，服务端创建一个 `MethodResponse` 对象，它是一个 Python 对象（继承 `PyObjectPlus`），脚本通过设置其属性来填充返回值，最后调用 `py_done` 触发回传。

```cpp
class MethodResponse: public PyObjectPlus
{
public:
    MethodResponse( int replyID, const Mercury::Address & replyAddr,
        Mercury::NetworkInterface & networkInterface,
        const MethodDescription & methodDesc );
    PY_METHOD_DECLARE( py_done )   // 脚本调用 response.done() 触发回传
    virtual PyObject * pyGetAttribute( const char * attr );  // 读取返回值属性
    virtual int pySetAttribute( const char * attr, PyObject* value ); // 设置返回值
private:
    Mercury::ReplyID replyID_;
    Mercury::Address replyAddr_;
    Mercury::NetworkInterface & interface_;
    const MethodDescription & methodDesc_;
    typedef std::map< std::string, PyObjectPtr > ReturnValueData;
    ReturnValueData returnValueData_;
};
```

### 5.4 RemoteEntityMethod —— 跨进程方法调用代理

定义于 [remote_entity_method.hpp](file:///workspace/src/lib/entitydef/remote_entity_method.hpp)。这是一个轻量的 Python 类型，代表“某个远程实体上的某个方法”。当脚本写 `entityMailBox.someMethod` 时，`PyEntityMailBox::pyGetAttribute` 会返回一个 `RemoteEntityMethod`，调用它即触发跨进程 RPC。

```cpp
class RemoteEntityMethod : public PyObjectPlus
{
public:
    RemoteEntityMethod( PyEntityMailBox * pMailBox,
            const MethodDescription * pMethodDescription, ... );
    PY_METHOD_DECLARE( pyCall )   // 调用时：校验参数 -> mailbox.getStream() -> addToStream -> sendStream()
private:
    SmartPointer< PyEntityMailBox > pMailBox_;
    const MethodDescription * pMethodDescription_;
};
```

### 5.5 MailBoxBase —— Mailbox，指向远程 entity 的引用

定义于 [mailbox_base.hpp](file:///workspace/src/lib/entitydef/mailbox_base.hpp)。Mailbox 是 BigWorld 的核心抽象之一——它是一个 Python 对象，**指向另一个进程中的 entity**。脚本持有 Mailbox 后，就可以像调用本地方法一样调用远程 entity 的方法。

```cpp
class PyEntityMailBox: public PyObjectPlus
{
public:
    virtual const MethodDescription * findMethod( const char * attr ) const = 0;
    virtual BinaryOStream * getStream( const MethodDescription & methodDesc,
            Mercury::ReplyMessageHandler * pHandler = NULL ) = 0;
    virtual void sendStream() = 0;

    static PyObject * constructFromRef( const EntityMailBoxRef & ref );
    static bool reducibleToRef( PyObject * pObject );
    static EntityMailBoxRef reduceToRef( PyObject * pObject );

    virtual EntityID id() const = 0;
    virtual void address( const Mercury::Address & addr ) = 0;
    virtual const Mercury::Address address() const = 0;
    virtual void migrate() {}

    // 注册不同 Component（BASE/CELL/CLIENT）的 Mailbox 工厂
    static void registerMailBoxComponentFactory(
        EntityMailBoxRef::Component c, FactoryFn fn, PyTypeObject * pType );
    static void registerMailBoxRefEquivalent( CheckFn cf, ExtractFn ef );

    PY_PICKLING_METHOD_DECLARE( MailBox )   // 支持 pickle
    static void visit( PyEntityMailBoxVisitor & visitor );
private:
    static Population s_population_;   // 全局所有 Mailbox 的列表（用于遍历）
};
```

**`PyEntityMailBox` 是抽象基类**，其 `findMethod` / `getStream` / `sendStream` / `id` / `address` 由各进程的具体子类实现（baseapp 的 `BaseMailBox`、cellapp 的 `CellMailBox`、客户端的 `ClientMailBox` 等，定义在各自的 server 目录中）。

**`EntityMailBoxRef`** 是 Mailbox 的“序列化形态”——包含 Component 类型、entity ID、目标地址，可安全地写入流或 pickle。`reduceToRef` / `constructFromRef` 用于在属性变更流中传递 Mailbox 引用。

**`PyEntityMailBoxVisitor`** 是访问者模式的接口，`PyEntityMailBox::visit` 会遍历全局所有 Mailbox，常用于备份切换（见 `BaseBackupSwitchMailBoxVisitor`）或迁移（见 `MigrateMailBoxVisitor`）。

---

## 6. 位流（BitReader / BitWriter）

### 6.1 BitReader

定义于 [bit_reader.hpp](file:///workspace/src/lib/entitydef/bit_reader.hpp)。从 `BinaryIStream` 中按位读取，用于解析 `PropertyChange` 等紧凑编码的数据。

```cpp
class BitReader
{
public:
    BitReader( BinaryIStream & data );
    int get( int nbits );   // 读取 nbits 位，返回 int
private:
    BinaryIStream & data_;
    int bitsLeft_;          // 当前字节剩余未读位数
    uint8 byte_;            // 当前正在读取的字节
};
```

### 6.2 BitWriter

定义于 [bit_writer.hpp](file:///workspace/src/lib/entitydef/bit_writer.hpp)。向内部缓冲区按位写入。

```cpp
class BitWriter
{
public:
    BitWriter();
    void add( int numBits, int bits );      // 写入 numBits 位
    int usedBytes() const;                  // 已使用字节数
    const void * bytes() const;             // 缓冲区首地址
private:
    int byteCount_;
    int bitsLeft_;                          // 当前字节剩余可写位数
    uint8 bytes_[224];                      // 固定 224 字节缓冲区
};
```

位流主要用于 `PropertyChange` 的路径与类型标记编码——例如“根属性索引 ≤ 60”时可用 6 位编码，超过则用特殊标记 + 完整字节。这种紧凑编码在 MMO 场景下能显著降低属性广播的带宽。

---

## 7. Volatile 信息

### 7.1 VolatileInfo

定义于 [volatile_info.hpp](file:///workspace/src/lib/entitydef/volatile_info.hpp)。Volatile 属性是**频繁变化但不持久化**的属性，主要是位置与朝向——它们每 tick 都可能变，需要高频同步给 ghost 与客户端，但绝不写数据库。

```cpp
class VolatileInfo
{
public:
    VolatileInfo( float positionPriority = -1.f,
        float yawPriority = -1.f,
        float pitchPriority = -1.f,
        float rollPriority = -1.f );

    bool parse( DataSectionPtr pSection );
    bool shouldSendPosition() const;     // positionPriority_ > 0
    int dirType( float priority ) const; // 返回需要同步的朝向分量
    bool isLessVolatileThan( const VolatileInfo & info ) const;
    bool hasVolatile( float priority ) const;

    static const float ALWAYS;

    float positionPriority() const;
    float yawPriority() const;
    float pitchPriority() const;
    float rollPriority() const;
};
```

`.def` 中的 `<Volatile>` 节点配置各分量的优先级：`<position/>`、`<yaw/>`、`<pitch/>`、`<roll/>`。优先级为正表示该分量需要同步；`ALWAYS` 常量代表最高优先级。`isLessVolatileThan` 用于比较两个实体类型的 volatile 程度（派生类不能比父类更 volatile）。

### 7.2 PyVolatileInfo

定义于 [py_volatile_info.hpp](file:///workspace/src/lib/entitydef/py_volatile_info.hpp)。提供 `VolatileInfo` 与 Python 之间的转换：`Script::setData` 从 Python 对象构造 `VolatileInfo`，`Script::getData` 反向转换；`PyVolatileInfo::priorityFromPyObject` / `pyObjectFromPriority` 处理优先级值的 Python 表示。

---

## 8. UserDataObject（UDO）

UserDataObject 是 BigWorld 中与 chunk（场景分块）关联的“用户数据对象”——设计师在场景里放置的、带类型化属性的占位对象，用于触发器、标记点、配置点等。它的描述系统与 Entity 平行但更简单（只有属性，没有方法/客户端同步）。

### 8.1 类层级

```
BaseUserDataObjectDescription           (基类：属性解析、name、properties_、propertyMap_)
├── UserDataObjectDescription           (UDO：增加 domain_ 字段)
└── EntityDescription                   (Entity：在 .ipp 中也继承自此基类)
```

### 8.2 BaseUserDataObjectDescription

定义于 [base_user_data_object_description.hpp](file:///workspace/src/lib/entitydef/base_user_data_object_description.hpp)。是 `EntityDescription` 与 `UserDataObjectDescription` 的共同基类，提供属性解析的公共逻辑：

```cpp
class BaseUserDataObjectDescription
{
public:
    virtual bool parse( const std::string & name, DataSectionPtr pSection = NULL, bool isFinal = true );
    const std::string& name() const;
    unsigned int propertyCount() const;
    DataDescription* property( unsigned int n ) const;
    virtual DataDescription* findProperty( const std::string& name ) const;
protected:
    virtual bool parseProperties( DataSectionPtr pProperties ) = 0;   // 纯虚
    virtual bool parseInterface( DataSectionPtr pSection, const char * interfaceName );
    virtual bool parseImplements( DataSectionPtr pInterfaces );       // 接口继承
    std::string name_;
    typedef std::vector< DataDescription > Properties;
    Properties properties_;
    typedef std::map< std::string, unsigned int > PropertyMap;
    PropertyMap propertyMap_;
};
```

`parseImplements` 支持接口继承——`.def` 中的 `<Implements>` 节点可声明实现的接口名，系统会去 `entity_defs/interfaces/<interfaceName>.def` 加载接口定义的属性与方法（见 `base_user_data_object_description.cpp` 第 151 行）。

### 8.3 UserDataObjectDescription

定义于 [user_data_object_description.hpp](file:///workspace/src/lib/entitydef/user_data_object_description.hpp)。增加 `domain_` 字段标记 UDO 只存活在哪个域：

```cpp
class UserDataObjectDescription: public BaseUserDataObjectDescription
{
public:
    enum UserDataObjectDomain {
        NONE = 0x0, BASE = 0x1, CELL = 0x2, CLIENT = 0x4
    };
    const UserDataObjectDomain domain() const;
private:
    UserDataObjectDomain domain_;
};
```

`domain_` 由 `.def` 中的 `<Domain>` 节点指定，chunk 加载时据此决定是否实例化该 UDO。

### 8.4 UserDataObjectDescriptionMap

定义于 [user_data_object_description_map.hpp](file:///workspace/src/lib/entitydef/user_data_object_description_map.hpp)。解析 `scripts/user_data_objects.xml`，建立 UDO 类型注册表（结构与 `EntityDescriptionMap` 平行，但用 `map<string, Description>` 而非 vector+map 双索引，因为 UDO 不需要紧凑的整数索引）。

---

## 9. 元数据与统计

### 9.1 MetaDataType

见 [4.2.2 节](#422-metadatatype--类型工厂)。`MetaDataType` 是类型工厂的注册中心，所有内建类型与自定义类型在进程启动时通过 `addMetaType` 注册自己，`DataType::buildDataType` 通过 `MetaDataType::find` 查名后委托 `getType` 创建实例。

### 9.2 EntityMemberStats

定义于 [entity_member_stats.hpp](file:///workspace/src/lib/entitydef/entity_member_stats.hpp)。`DataDescription` 与 `MethodDescription` 各持有一个 `EntityMemberStats` 实例，用于统计该属性/方法的流量分布：

```cpp
class EntityMemberStats
{
#if ENABLE_WATCHERS
    class Stat
    {
        IntrusiveStatWithRatesOfChange< unsigned int > messages_;
        IntrusiveStatWithRatesOfChange< unsigned int > bytes_;
    };
    Stat sentToOwnClient_;        // 发往自己客户端
    Stat sentToOtherClients_;     // 发往其它客户端
    Stat addedToHistoryQueue_;    // 进入历史队列（断线重连用）
    Stat sentToGhosts_;           // 发往 ghost
    Stat sentToBase_;             // 发往 base
    Stat received_;               // 接收
#endif
};
```

这些统计通过 Watcher 系统暴露，运维可实时观察“哪个属性的同步占了最多带宽”，是 MMO 调优的关键工具。`Stat` 内部用 `IntrusiveStatWithRatesOfChange` 同时记录累计值与变化率。

---

## 10. 与其它模块的协作

### 10.1 与 network 模块

`entitydef` 不直接依赖 Mercury 的传输层，但通过 `BinaryOStream` / `BinaryIStream`（来自 `cstdmf/binary_stream.hpp`）与网络层衔接：

- 属性序列化：`DataDescription::addToStream` → `DataType::addToStream` → 写入 `BinaryOStream`
- 方法调用：`MethodDescription::addToStream` 把参数写入流，`callMethod` 从流读参数
- Mailbox：`PyEntityMailBox::getStream` 返回一个 `BinaryOStream*`（实际是 Mercury Bundle），脚本写完参数后 `sendStream()` 发出

`EntityDescription` 还引用 `BWCompressionType`（来自 `network/compression_type.hpp`），为该实体类型的大消息指定内网/外网压缩算法。

### 10.2 与 resmgr 模块

`.def` 与 `entities.xml` 都是 XML 文件，通过 `resmgr` 的 `DataSection` / `BWResource` 读取：

```cpp
// entity_description.cpp
std::string filename = this->getDefsDir() + "/" + name + ".def";
DataSectionPtr pSection = BWResource::openSection( filename );
```

`constants.hpp`（见 [entitydef/constants.hpp](file:///workspace/src/lib/entitydef/constants.hpp)）集中定义了所有资源路径常量：

| 常量方法 | 路径 | 用途 |
| --- | --- | --- |
| `entitiesFile()` | `scripts/entities.xml` | 实体类型清单 |
| `entitiesDefsPath()` | `scripts/entity_defs` | `.def` 文件目录 |
| `entitiesClientPath()` | `scripts/client` | 客户端脚本 |
| `entitiesCellPath()` | `scripts/cell` | cell 脚本 |
| `entitiesBasePath()` | `scripts/base` | base 脚本 |
| `aliasesFile()` | `scripts/entity_defs/alias.xml` | 类型别名 |
| `userDataObjectsFile()` | `scripts/user_data_objects.xml` | UDO 清单 |
| `userDataObjectsDefsPath()` | `scripts/user_data_object_defs` | UDO `.def` 目录 |

### 10.3 与 pyscript 模块

`entitydef` 大量使用 `pyscript`（`PyObjectPlus`、`Script`、`PY_METHOD_DECLARE` 等）把 C++ 对象暴露给 Python：

- `PyEntityMailBox`、`RemoteEntityMethod`、`MethodResponse`、`SharedData` 都继承 `PyObjectPlus`
- `Script::setData` / `getData` 提供自定义类型与 Python 的互转（如 `VolatileInfo`、`EntityMailBoxRef`、`AutoBackupAndArchive::Policy`）
- `DataType` 体系的核心就是“PyObject ↔ Stream/Section”互转
- `PY_PICKLING_METHOD_DECLARE` 让 Mailbox 支持 pickle

### 10.4 与 server 模块

`EntityDescription` 是 baseapp / cellapp / dbmgr 实体实例的依赖：

- **baseapp**：用 `EntityDescription` 序列化 base 属性到数据库、构建客户端属性同步流、处理客户端 RPC
- **cellapp**：用 `EntityDescription` 管理 cell 属性、ghost 同步、LOD 切换、volatile 属性广播
- **dbmgr**：用 `EntityDescription::addPersistentPropertiesToMD5` 生成数据库 schema 指纹，用 `addSectionToStream` / `readStreamToSection` 在 DB 与进程间搬运持久数据
- **client**：用 `clientIndex()` / `clientMethod()` / `clientServerProperty()` 处理与服务端的协议

---

## 11. `.def` 文件解析与示例

### 11.1 解析入口

`EntityDescription::parse`（[entity_description.cpp](file:///workspace/src/lib/entitydef/entity_description.cpp) 第 111 行）是单个实体类型的解析入口：

1. 打开 `entity_defs/<name>.def`
2. 读取 `<Parent>` —— 若存在，递归调用 `parse( parentName, isFinal=false )` 先解析父类（实现继承）
3. 读取 `<ClientName>` —— 客户端类名（可不同于服务端名）
4. 检查 cell/base/client 脚本文件是否存在（`.py/.pyc/.pyo/.pyd`），设置 `hasCellScript_` 等
5. 调用 `parseInterface` 解析主体：
   - `LoDLevels`（仅服务端）
   - `Properties`（委托 `BaseUserDataObjectDescription::parseInterface` → `parseProperties`）
   - `Volatile`
   - `ClientMethods` / `CellMethods` / `BaseMethods`
   - `TempProperties`
6. 服务端 final 阶段：转换 LOD 索引、校验无 cell 脚本的实体不应有 cell 属性、初始化压缩类型

### 11.2 典型 `.def` 结构

仓库 `src/` 下未直接携带示例 `.def`（它们属于资源目录 `scripts/`，不在源码树中），但根据 `constants.hpp` 与解析代码可还原典型结构：

```xml
<!-- scripts/entity_defs/Avatar.def -->
<Avatar>
    <Parent/>              <!-- 可选：父类型名，实现继承 -->
    <ClientName>Avatar</ClientName>   <!-- 可选：客户端类名 -->

    <NetworkCompression>
        <internal/>        <!-- 可选：内网压缩算法 -->
        <external/>        <!-- 可选：外网压缩算法 -->
    </NetworkCompression>

    <Volatile>
        <position/>        <!-- 位置优先级 -->
        <yaw/>             <!-- 偏航优先级 -->
        <pitch/>
        <roll/>
    </Volatile>

    <LoDLevels>            <!-- 仅服务端，LOD 层级定义 -->
        <level>
            <label> near </label>
            <start> 0 </start>
            <hyst> 10 </hyst>
        </level>
        <level>
            <label> far </label>
            <start> 50 </start>
            <hyst> 20 </hyst>
        </level>
    </LoDLevels>

    <Properties>
        <hp>
            <Type> INT32 </Type>
            <Flags> BASE </Flags>           <!-- 仅 base 可见，持久化 -->
            <Persistent> true </Persistent>
            <Default> 100 </Default>
            <DatabaseLength> 11 </DatabaseLength>
        </hp>
        <position>
            <Type> VECTOR3 </Type>
            <Flags> CELL_PUBLIC_AND_OWN </Flags>   <!-- ghost + 自己客户端 -->
            <Persistent> false </Persistent>
        </position>
        <name>
            <Type> UNICODE_STRING </Type>
            <Flags> ALL_CLIENTS </Flags>            <!-- 广播给所有客户端 -->
            <Persistent> true </Persistent>
            <IsIdentifier> true </IsIdentifier>     <!-- DATA_ID，数据库索引 -->
        </name>
        <inventory>
            <Type>
                <Array>
                    <element>
                        <Type>
                            <FixedDict>
                                <Properties>
                                    <itemId>  <Type> UINT32 </Type> </itemId>
                                    <count>   <Type> UINT16 </Type> </count>
                                </Properties>
                            </FixedDict>
                        </Type>
                    </element>
                </Array>
            </Type>
            <Flags> BASE_AND_CLIENT </Flags>
        </inventory>
    </Properties>

    <ClientMethods>
        <onHealthChanged>
            <Exposed/>                      <!-- 可被服务端调用 -->
            <Arg> UINT32 </Arg>             <!-- 新血量 -->
        </onHealthChanged>
    </ClientMethods>

    <CellMethods>
        <moveTo>
            <Arg> VECTOR3 </Arg>
            <Arg> FLOAT </Arg>              <!-- 速度 -->
        </moveTo>
        <attack>
            <Exposed/>                      <!-- 可被客户端调用 -->
            <Arg> ENTITY_ID </Arg>
            <ReturnValues>
                <success>  <Type> BOOL </Type> </success>
            </ReturnValues>
        </attack>
    </CellMethods>

    <BaseMethods>
        <saveToDB/>
        <tellBase>
            <Arg> STRING </Arg>
        </tellBase>
    </BaseMethods>

    <TempProperties>
        <tempBuffer>  <Type> STRING </Type>  </tempBuffer>
    </TempProperties>
</Avatar>
```

`entities.xml` 则是简单的类型清单：

```xml
<!-- scripts/entities.xml -->
<Entities>
    <Avatar/>
    <NPC/>
    <Projectile/>
    <Item/>
</Entities>
```

每个子节点名即为类型名，`EntityDescriptionMap::parse` 据此打开对应的 `.def`。

### 11.3 接口继承

`.def` 支持接口继承（`<Implements>` 节点），接口定义在 `entity_defs/interfaces/<Name>.def`。`BaseUserDataObjectDescription::parseImplements` 会递归加载接口的属性与方法，实现跨类型的复用。配合 `<Parent>` 的类型继承，BigWorld 提供了两层继承模型：类型继承（整个 `.def` 继承）+ 接口继承（按接口名组合）。

---

## 12. 属性变化从 cellapp → baseapp → client 的序列化流程

下面以“cellapp 上实体 `Avatar` 的 `position`（`CELL_PUBLIC_AND_OWN`）变化”为例，描述完整的属性广播链路。

### 12.1 ASCII 时序图

```
   cellapp (Avatar cell 实体)          baseapp (Avatar base 实体)         client (Avatar 镜像)
        │                                     │                                  │
   1. 脚本设置 entity.position = newPos       │                                  │
        │                                     │                                  │
   2. PropertyOwner 链向上冒泡               │                                  │
      onOwnedPropertyChanged(change)          │                                  │
        │                                     │                                  │
   3. cell 实体 getTopLevelOwner 找到顶层     │                                  │
      构造 SinglePropertyChange              │                                  │
      (path=[propIndex], value=newPos)        │                                  │
        │                                     │                                  │
   4. 对每个 ghost:                          │                                  │
      PropertyChange::addToStream            │                                  │
      -> BitWriter 编码路径+类型             │                                  │
      -> BinaryOStream 写值                  │                                  │
      -> Mercury bundle 发往 ghost cellapp    │                                  │
        │                                     │                                  │
   5. 若 DATA_OTHER_CLIENT: 对每个            │                                  │
      有该实体 AOI 的客户端 witness:          │                                  │
      a. 更新 PropertyEventStamps            │                                  │
         (eventStamp = ++lastEventNumber)     │                                  │
      b. EntityDescription::addDictionaryToStream│                              │
         (dataDomains=CELL_DATA|CLIENT_DATA)  │                                  │
         shouldConsiderData 按 pass 过滤      │                                  │
         -> DataDescription::addToStream      │                                  │
         -> DataType::addToStream             │                                  │
         -> 写入 client bundle                │                                  │
      c. 发往 baseapp 转发给 client           │                                  │
        │                                     │                                  │
        │                  6. baseapp 收到属性变更消息                          │
        │                     若是 OWN_CLIENT 属性:                            │
        │                     转发到对应 client 连接                           │
        │                                     │                                  │
        │                                     │  7. client 收到属性同步包        │
        │                                     │     PropertyChangeReader::       │
        │                                     │     readAndApply(stream, owner)  │
        │                                     │     -> 沿 path 定位属性           │
        │                                     │     -> setOwnedProperty          │
        │                                     │     -> 触发脚本 onPropertyChanged │
        │                                     │                                  │
   8. 若 DATA_BASE: 同时发往 baseapp           │                                  │
      baseapp 收到后 PropertyChangeReader 应用 │                                  │
      并可能触发 writeToDB（若持久化）          │                                  │
```

### 12.2 关键步骤详解

**步骤 2–3：变更冒泡**。当脚本修改 `entity.position` 时，`DataType::attach` 安装的 setter 触发 `PropertyOwnerBase::onOwnedPropertyChanged`，沿 `getChildPropertyOwner` 反向链向上传递，每个中间 owner 通过 `addToPath(index)` 把自己的索引加入 `change.path_`，直到 `getTopLevelOwner` 返回 entity 本身。

**步骤 4：ghost 同步**。`CELL_PUBLIC`（`DATA_GHOSTED`）属性的变更会广播给所有持有该 entity ghost 的 cellapp。用 `PropertyChange::addToStream` 紧凑编码：先写根索引（≤60 用 6 位，否则用 61/62 标记 + 完整字节），再用 `BitWriter` 写额外位，最后 `DataType::addToStream` 写新值。ghost 侧用 `PropertyChangeReader::readAndApply` 反向解码并应用。

**步骤 5a：事件时间戳**。`OTHER_CLIENT` 属性每次变化递增 `lastEventNumber`，并记录到 `PropertyEventStamps`。客户端重连或新进入 AOI 时，用时间戳判定需要补发哪些历史变更（`addedToHistoryQueue_` 统计的就是这部分流量）。

**步骤 5b：客户端同步**。`EntityDescription::addDictionaryToStream` 是核心入口，它用 `shouldSkipPass` / `shouldConsiderData` 按 4 个 pass 过滤属性：

| Pass | Base? | ClientServer? | 对应 dataDomains |
| --- | --- | --- | --- |
| 0 | 是 | 否 | `BASE_DATA` |
| 1 | 是 | 是 | `BASE_DATA\|CLIENT_DATA` |
| 2 | 否 | 是 | `CELL_DATA\|CLIENT_DATA` |
| 3 | 否 | 否 | `CELL_DATA` |

`EXACT_MATCH` 标志要求精确匹配某 pass；否则只要请求的域与 pass 有交集就执行。每个 pass 内，`AddToStreamVisitor` 的子类（`AddToStreamDictionaryVisitor` 从 dict 取值、`AddToStreamAttributeVisitor` 从对象属性取值、`AddToStreamSectionVisitor` 从 DataSection 取值）负责取出 Python 值，校验类型后调用 `DataDescription::addToStream` 写流。

**步骤 6–7：baseapp 中转与客户端应用**。baseapp 收到 cellapp 发来的属性变更后，若是 `OWN_CLIENT` 属性则转发给客户端；若是 `BASE` 属性则本地应用。客户端用 `PropertyChangeReader` 沿 `path_` 定位到具体属性，调用 `setOwnedProperty` 写入新值并触发脚本回调。

**步骤 8：持久化**。`DATA_PERSISTENT` 属性的变化最终会触发 `writeToDB`，通过 `EntityDescription::addSectionToStream(ONLY_PERSISTENT_DATA)` 只把持久属性写入流，经 baseapp 发往 dbmgr 落库。

---

## 13. 重要宏与代码生成约定

### 13.1 SIMPLE_DATA_TYPE 宏

```cpp
#define SIMPLE_DATA_TYPE( TYPE, NAME )					\
	SimpleMetaDataType< TYPE > s_##NAME##_metaDataType( #NAME );
```

在 `data_types.cpp` 中用此宏批量注册所有内建简单类型（`INT8`、`FLOAT`、`VECTOR3` 等）。每个宏展开为一个全局 `SimpleMetaDataType` 实例，其构造函数调用 `MetaDataType::addMetaType` 自动注册。

### 13.2 DATA_DISTRIBUTION_FLAGS 宏

```cpp
#define DATA_DISTRIBUTION_FLAGS (DATA_GHOSTED | DATA_OTHER_CLIENT | \
                                DATA_OWN_CLIENT | DATA_BASE | 		\
                                DATA_CLIENT_ONLY | DATA_EDITOR_ONLY)
```

标记“会参与分发的”属性标志集合（不含 `DATA_PERSISTENT` 与 `DATA_ID`）。

### 13.3 索引上限约定

`EntityDescriptionMap::checkCount` 中的 `maxEfficient` / `maxAllowed` 反映了协议编码约定：

| 资源 | 高效上限 | 硬上限 | 编码依据 |
| --- | --- | --- | --- |
| 客户端-服务端属性 | 61 | 256 | 6 位编码根索引，61/62 留作类型标记 |
| 客户端方法 | 62 | 62×256 | 单字节 top index + sub-slot |
| base 暴露方法 | 62 | 62×256 | 同上 |
| cell 暴露方法 | 62 | 62×256 | 同上 |

这些上限源自 `PropertyChange` 的位流编码宽度与 Mercury 协议的方法寻址宽度。

### 13.4 MF_SERVER / EDITOR_ENABLED 条件编译

`EntityDescription` 中多处用 `#ifdef MF_SERVER` 区分服务端独有逻辑（LOD、压缩类型、cell 脚本检查）；`#ifdef EDITOR_ENABLED` 区分编辑器独有逻辑（`editable_`、`widget`、`editorModel_`、`createEditorProperty`）。客户端编译时这些都被裁剪，保证发行版不携带服务端元数据。

---

## 14. 小结

`entitydef` 是 BigWorld 引擎的“类型系统中枢”，它的设计有几个值得注意的精妙之处：

1. **声明式 + 反射式**：`.def` 文件声明实体结构，运行时生成完整反射元数据，让 C++ 与 Python 都能动态访问属性与方法。
2. **三套可见域**：BASE / CELL / CLIENT 三套属性与方法集合，配合 `DataDomain` 掩码实现“一次序列化、按域过滤”，避免重复编码。
3. **增量广播**：`PropertyChange` + `PropertyOwner` 链实现字段级增量同步，配合 `BitReader/Writer` 紧凑编码，把 MMO 的属性同步带宽压到最低。
4. **类型对象池**：`DataType` 的 `SingletonMap` 保证相同结构的类型全局唯一，`MetaDataType` 工厂模式让自定义类型（`UserDataType` / `FixedDictDataType`）可无缝接入。
5. **双索引寻址**：方法与属性都有 internal / exposed 两套索引，服务端按名字高效查找，客户端按紧凑整数寻址，兼顾效率与协议紧凑性。

理解了 `entitydef`，就理解了 BigWorld 实体系统的“骨架”——后续 baseapp / cellapp / dbmgr 的实体实例都是在这套元数据上“填血肉”。
