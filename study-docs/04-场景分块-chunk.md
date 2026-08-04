# BigWorld Engine 2.0.1 场景分块模块（chunk）

> 源码目录：`/workspace/src/lib/chunk/`
> 模块定位：BigWorld 的**核心空间抽象**——把整个游戏世界切成网格化的 *chunk*，按相机位置流式加载 / 卸载，并通过 *portal* 把室内外、相邻 chunk 互相连通。它是客户端“看得见的世界”与服务端“物理空间模拟”的统一载体。
> 模块边界：上层（cellapp / 客户端 App / 编辑器）通过 `ChunkManager` 驱动场景；下层文件 IO 全部委托给 [resmgr](file:///workspace/src/lib/resmgr/) 的 `BWResource`，渲染提交委托给 `moo::RenderContext`，异步加载走 `cstdmf` 的 `BgTaskManager`。
> 编号：04（接 `03-资源管理-resmgr.md`，与 `05-地形系统-terrain.md` 并列）。

---

## 1. 模块定位

`chunk` 库解决四件事：

1. **空间划分**：把一个 *space*（一个独立的笛卡尔世界，例如一颗星球、一座副本地下城）划分为互不重叠的凸体积 *chunk*。室外按 100m×100m 网格切，室内为任意凸多面体。
2. **流式加载**：相机移动时，按 `maxLoadPath_` 半径内的 chunk 异步加载、按 `minUnloadPath_` 之外卸载，整个世界不必同时驻留内存。
3. **可见性连通**：chunk 之间通过 *portal*（边界上的多边形窗口）互相可见。`heaven`/`earth` 是特殊 portal，分别连接天空和地形。室内 chunk 通过 *internal portal* 嵌入到室外 chunk。
4. **可放置对象**：所有放进 chunk 的“东西”统一抽象为 `ChunkItem`——模型、地形、光源、声音、植被、水面、路点、用户数据对象、Umbra 遮挡器等。

### 1.1 三层结构：universe / space / chunk

`ChunkManager` 持有 `spaces_`（`std::map<ChunkSpaceID, ChunkSpace*>`），一个进程只有一个 universe：

```cpp
// chunk_manager.hpp:54
class ChunkManager : public Aligned
{
public:
    bool init( DataSectionPtr configSection = NULL );
    bool fini();

    // set the camera position
    void camera( const Matrix & cameraTransform, ChunkSpacePtr pSpace, Chunk* pOverride = NULL );
    const Matrix & cameraTrans() const { return cameraTrans_; }

    // call everyone's tick method, plus scan for
    // new chunks to load and old chunks to dispose
    void tick( float dTime );

    // draw the scene from the set camera position
    void draw();

    ChunkSpacePtr space( ChunkSpaceID spaceID, bool createIfMissing = true );
    ChunkSpacePtr cameraSpace() const;
    Chunk * cameraChunk() const			{ return cameraChunk_; }

    void addSpace( ChunkSpace * pSpace );
    void delSpace( ChunkSpace * pSpace );

    static ChunkManager & instance();
    // ...
private:
    typedef std::map<ChunkSpaceID,ChunkSpace*>	ChunkSpaces;
    ChunkSpaces			spaces_;

    Matrix			cameraTrans_;
    ChunkSpacePtr	pCameraSpace_;
    Chunk			* cameraChunk_;
    // ...
};
```

- `ChunkSpaceID` 是 `uint32`，`NULL_CHUNK_SPACE = 0` 表示“无空间”，定义在 [base_chunk_space.hpp](file:///workspace/src/lib/chunk/base_chunk_space.hpp)。
- 服务端的 `ChunkManager::instance()` 见 [server_chunk_manager.cpp](file:///workspace/src/lib/chunk/server_chunk_manager.cpp)，实现极简：构造函数只初始化相机矩阵与空指针，`space()` 在 `createIfMissing=true` 时打 WARNING（服务端不允许隐式建 space）。

### 1.2 模块依赖关系图

```
                         ┌──────────────────────────────┐
                         │        上层调用方             │
   cellapp / ServerChunk │  客户端 App / EditorChunk     │
        Manager          │  (camera, tick, draw)         │
                         └───────────────┬──────────────┘
                                         │
                                         ▼
        ┌────────────────────────────────────────────────────────┐
        │                     ChunkManager                       │  ← singleton
        │   spaces_ / cameraTrans_ / loadingChunks_ / fringe     │
        │   tick / draw / scan / camera / blindpanic              │
        └───────┬───────────────────────────┬────────────────────┘
                │                           │
                ▼                           ▼
   ┌────────────────────────┐    ┌──────────────────────────────┐
   │      ChunkSpace        │    │        GeometryMapping         │
   │ (ConfigChunkSpace 别名) │    │  path/ mapper/ invMapper       │
   │  BaseChunkSpace ── Column│   │  grid<->local 坐标变换        │
   │  ClientChunkSpace      │    │  outsideChunkIdentifier       │
   │  ServerChunkSpace      │    └──────────────┬─────────────────┘
   └────────────┬───────────┘                   │
                │ addChunk / findChunk           │ pSpace_
                ▼                               ▼
   ┌────────────────────────────────────────────────────────────┐
   │                          Chunk                              │
   │  identifier / x_ / z_ / pMapping_ / pSpace_                │
   │  load / bind / unbind / focus / tick / draw                 │
   │  bounds_ / joints_ (ChunkBoundaries)                        │
   │  selfItems_ / dynoItems_ / swayItems_ / lenders_            │
   │  caches_ (ChunkCache**)                                     │
   └──────────────┬─────────────────────────────────────────────┘
                  │ addStaticItem / addDynamicItem / addLoanItem
                  ▼
   ┌────────────────────────────────────────────────────────────┐
   │                       ChunkItem 体系                         │
   │  ChunkModel  ChunkTerrain  ChunkLight*  ChunkSound          │
   │  ChunkFlora  ChunkWater    ChunkFlare   ChunkMarker          │
   │  ChunkVLO   ChunkStationNode  ChunkOverlapper  ChunkUmbra    │
   │  ChunkUserDataObject  BaseChunkTree  ChunkItemTreeNode       │
   └────────────────────────────────────────────────────────────┘
                  │
                  ▼
   ┌────────────────────────────────────────────────────────────┐
   │  下游：BWResource (resmgr) / moo::RenderContext             │
   │       BgTaskManager (cstdmf) / Umbra (third_party)          │
   │       physics2 (BSP/QuadTree/HullTree/WorldTriangle)        │
   └────────────────────────────────────────────────────────────┘
```

模块对外的高频接口只有 `ChunkManager::camera()` / `tick()` / `draw()` 三个，几乎全部场景驱动都集中在这三处。

---

## 2. 顶层抽象

### 2.1 ChunkManager

声明在 [chunk_manager.hpp](file:///workspace/src/lib/chunk/chunk_manager.hpp)。它是 `Aligned`（不是 `Singleton<>`，而是手写 `static ChunkManager & instance()`），客户端与服务端共用同一头文件，实现分别在 `chunk_manager.cpp` 与 `server_chunk_manager.cpp`。

**关键职责**：

| 方法 | 作用 |
| --- | --- |
| `init(configSection)` | 读取 `resources.xml` 中的 chunk 配置（`maxLoadPath` / `minUnloadPath` 等） |
| `camera(transform, pSpace, pOverride)` | 设置相机变换与所在 space；触发 `checkCameraBoundaries()` 判定是否穿越 portal |
| `tick(dTime)` | 遍历所有 `tickItems` 调 `ChunkItem::tick`；调用 `scan()` 决定加载/卸载；维护 `tickMark_`/`totalTickTimeInMS_` |
| `draw()` | 从相机所在 chunk 出发，递归 portal 可见性遍历（`cullInsideChunks` / `cullOutsideChunks`），收集可见 chunk 后调 `drawBeg/drawSelf/drawEnd` |
| `scan()` | 内部方法：根据相机位置在 `maxLoadPath_` 内排队加载，在 `minUnloadPath_` 外排队卸载 |
| `blindpanic()` | 加载异常时的“盲 panic”——从已知 chunk 出发任意加载邻居以恢复 |
| `autoBootstrapSeedChunk()` | 启动时若空间为空，从 `space.settings` 找种子 chunk |
| `loadChunkExplicitly(id, pMapping, isOverlapper)` | 显式把一个 chunk 加入加载队列 |
| `findChunkByName(id, pMapping, createIfNotFound)` | 按名查找 chunk；不存在则创建空壳（不加载） |
| `findChunkByGrid(x, z, pMapping)` | 按网格坐标查找室外 chunk |
| `loadChunkNow(chunk)` | 同步加载（编辑器 / 服务端首块用） |
| `switchToSyncMode(bool)` | 切换同步/异步加载模式（`ScopedSyncMode` RAII 包装） |
| `switchToSyncTerrainLoad(bool)` | 单独切换地形同步加载 |
| `chunkDeleted(pChunk)` / `mappingCondemned(pMapping)` | chunk 销毁或 mapping 被废弃时通知 manager 清理 `pendingChunks_` |
| `closestUnloadedChunk(pSpace)` | 返回最近未加载 chunk 距离，供性能监控 |

**加载节流参数**：

```cpp
// chunk_manager.hpp:225
float			maxLoadPath_;	// bigger than sqrt(500^2 + 500^2)
float			minUnloadPath_;
unsigned int	maxUnloadChunks_;  // 编辑器设大值以积极回收内存
```

`autoSetPathConstraints(farPlane)` 会根据相机远裁剪面自动调整这两个阈值。

**统计计数器**（静态成员，调试 HUD 用）：

```cpp
// chunk_manager.hpp:127
static int      s_chunksTraversed;
static int      s_chunksVisible;
static int      s_chunksReflected;
static int      s_visibleCount;
static int      s_drawPass;
static bool     s_drawVisibilityBBoxes;
```

`Chunks_drawCullingHUD()`（见 [chunk.hpp](file:///workspace/src/lib/chunk/chunk.hpp) 末尾）把这些渲染到屏幕。

### 2.2 ChunkSpace 体系

`ChunkSpace` 是“一个连续的三维笛卡尔世界”，定义在 [chunk_space.hpp](file:///workspace/src/lib/chunk/chunk_space.hpp)。它通过编译期别名切换客户端/服务端实现：

```cpp
// chunk_space.hpp:15
#ifdef MF_SERVER
#include "server_chunk_space.hpp"
typedef ServerChunkSpace ConfigChunkSpace;
#else
#include "client_chunk_space.hpp"
class ClientChunkSpace;
typedef ClientChunkSpace ConfigChunkSpace;
#endif

class ChunkSpace : public ConfigChunkSpace
{
public:
    ChunkSpace( ChunkSpaceID id );

    typedef std::map< SpaceEntryID, GeometryMapping * > GeometryMappings;

    GeometryMapping * addMapping( SpaceEntryID mappingID, float * matrix,
        const std::string & path, DataSectionPtr pSettings = NULL );
    void addMappingAsync( SpaceEntryID mappingID, float * matrix,
        const std::string & path );
    GeometryMapping * getMapping( SpaceEntryID mappingID );
    void delMapping( SpaceEntryID mappingID );

    Chunk * findChunkFromPoint( const Vector3 & point );
    Column * column( const Vector3 & point, bool canCreate = true );

    // 碰撞接口：从 start 到 end 在空间内做 ray-cast
    float collide( const Vector3 & start, const Vector3 & end,
        CollisionCallback & cc = CollisionCallback_s_default ) const;
    float collide( const WorldTriangle & start, const Vector3 & end,
        CollisionCallback & cc = CollisionCallback_s_default ) const;

    bool setClosestPortalState( const Vector3 & point,
            bool isPermissive, WorldTriangle::Flags collisionFlags = 0 );

    Terrain::TerrainSettingsPtr terrainSettings() const;
    // ...
};
```

`addMappingAsync` 内部用 `LoadMappingTask`（`BackgroundTask` 子类）在后台线程打开 `space.settings` 并构造 `GeometryMapping`，然后通过 `doMainThreadTask` 把它装回 `mappings_`。这是 chunk 模块使用 `BgTaskManager` 的两个入口之一（另一个是 `ChunkLoader`）。

#### 2.2.1 BaseChunkSpace

定义在 [base_chunk_space.hpp](file:///workspace/src/lib/chunk/base_chunk_space.hpp)，是客户端/服务端共有的基类。它定义了**网格常量**：

```cpp
// base_chunk_space.hpp:49
const float GRID_RESOLUTION = 100.f;     // 室外 chunk 边长（米）
const float MAX_CHUNK_HEIGHT = 10000.f;
const float MAX_SPACE_WIDTH = 50000.f;
const float MIN_CHUNK_HEIGHT = -10000.f;
extern float g_gridResolution;
```

并提供 grid ↔ point 互转：

```cpp
// base_chunk_space.hpp:177
static inline float gridToPoint( int grid )  { return grid * GRID_RESOLUTION; }
static inline int   pointToGrid( float pt )  { return static_cast<int>( floorf( pt / GRID_RESOLUTION ) ); }
static inline float alignToGrid( float pt )  { return gridToPoint( pointToGrid( pt ) ); }
```

`BaseChunkSpace` 内嵌 `Column`（网格列）：

```cpp
// base_chunk_space.hpp:123
class Column
{
public:
    Column( int x, int z );
    void addChunk( HullBorder & border, Chunk * pChunk );
    Chunk * findChunk( const Vector3 & point );
    Chunk * findChunkExcluding( const Vector3 & point, Chunk * pNot );

    void addObstacle( const ChunkObstacle & obstacle );
    const ObstacleTree &	obstacles() const	{ return *pObstacleTree_; }
    void addDynamicObstacle( const ChunkObstacle & obstacle );
    void delDynamicObstacle( const ChunkObstacle & obstacle );

    Chunk *		pOutsideChunk() const	{ return pOutsideChunk_; }
    void		stale()				{ stale_ = true; }
    // ...
protected:
    HullTree *		pChunkTree_;       // 室内 chunk 的 HullTree（凸包索引）
    ObstacleTree *	pObstacleTree_;    // QuadTree<ChunkObstacle>，碰撞用
    HeldObstacles	heldObstacles_;
    Holdings		holdings_;
    Chunk *			pOutsideChunk_;    // 该网格的室外 chunk
    bool 			stale_;             // 标记需要重建
    Chunk *			shutTo_;
    std::vector< Chunk * >			seen_;
};
```

`Column` 是空间内**点→chunk**查找和**线段→障碍物**碰撞的核心索引。`HullTree`（来自 `physics2`）用于把室内 chunk 的凸包组织成树以加速“点是否在某室内 chunk 内”判定；`ObstacleTree` 是 `QuadTree<ChunkObstacle>`，用于室外碰撞。

`BaseChunkSpace` 还维护 `currentChunks_`（按名映射，`StringHashMap<std::vector<Chunk*>>`）与 `gridChunks_`（按网格坐标映射），以及 `blurred_`（已绑定但未 focus 的 chunk）列表。

#### 2.2.2 ClientChunkSpace

定义在 [client_chunk_space.hpp](file:///workspace/src/lib/chunk/client_chunk_space.hpp)，客户端版本。它在 `BaseChunkSpace::Column` 之上加了 `FocusGrid<Column>`：

```cpp
// client_chunk_space.hpp:51
template < class T, int ISPAN = 63 >
class FocusGrid
{
public:
    enum { SPANH = ISPAN/2 };
    enum { SPAN = ISPAN };
    enum { SPANX = ISPAN+1 };

    void origin( int cx, int cz );
    T * & operator()( int x, int z )  { return grid_[ z & SPAN ][ x & SPAN ]; }
    bool inSpan( int x, int z ) const;
    // ...
private:
    int		cx_;
    int		cz_;
    T *		grid_[SPANX][SPANX];   // 64×64 环形缓冲
};
```

`FocusGrid` 是一个 64×64 的环形缓冲，以相机所在网格为原点，覆盖周围 ±31 个网格。它让客户端不必遍历全空间 `column()`，只在焦点附近快速查列。

`ClientChunkSpace` 还管理：

- `tickItems_` / `pendingTickItems_`（`std::list<ChunkItemPtr>`，用 list 避免 vector 扩容时的 incRef/decRef 副作用）
- `homeless_`（`std::vector<ChunkItemPtr>`，找不到宿主 chunk 的孤儿 item）
- `sunLight_` / `ambientLight_` / `lights_`（`Moo::LightContainerPtr`，室外光照）
- `enviro_`（`EnviroMinder`，天空/云/雨/雾/水面环境）
- `chunkQuadTrees_`（`hash_map<uint32, ChunkQuadTree*>`，室外 chunk 的可见性四叉树）
- `umbraCell_` / `umbraInsideCell_`（Umbra 遮挡剔除集成）

`focus(point)` 是相机移动时的入口：更新 `currentFocus_` 原点，重新评估哪些列需要被激活。

#### 2.2.3 ServerChunkSpace

定义在 [server_chunk_space.hpp](file:///workspace/src/lib/chunk/server_chunk_space.hpp)，服务端版本。它把 `FocusGrid` 换成更朴素的 `ColumnGrid`（`std::vector<Column*>` + `xOrigin_`/`zOrigin_`/`xSize_`/`zSize_`），因为服务端不需要 64×64 环形缓冲：

```cpp
// server_chunk_space.hpp:34
class ColumnGrid
{
public:
    void rect( int xOrigin, int zOrigin, int xSize, int zSize );
    Column *& operator()( int x, int z );
    bool inSpan( int x, int z ) const;
    const int xBegin() const	{	return xOrigin_; }
    const int xEnd() const		{	return xOrigin_ + xSize_; }
    // ...
private:
    std::vector< Column * > grid_;
    int		xOrigin_;  int		zOrigin_;
    int		xSize_;    int		zSize_;
};
```

`ServerChunkSpace::addHomelessItem` 是空实现（服务端不持有孤儿 client item）；`seeChunk(Chunk*)` 用于 cellapp 在 AOI 模拟里标记某 chunk 被某个 cell 看到；`recalcGridBounds()` 在派生类里实际计算网格范围。

### 2.3 GeometryMapping

定义在 [geometry_mapping.hpp](file:///workspace/src/lib/chunk/geometry_mapping.hpp)。它把**一个资源目录**（含 chunk 文件）映射到一个 `ChunkSpace`，并附带世界变换矩阵：

```cpp
// geometry_mapping.hpp:37
class GeometryMapping : public Aligned, public SafeReferenceCount
{
public:
    GeometryMapping( ChunkSpacePtr pSpace, const Matrix & m,
        const std::string & path, DataSectionPtr pSettings );

    ChunkSpacePtr	pSpace() const			{ return pSpace_; }
    const Matrix &	mapper() const			{ return mapper_; }      // world = mapper * local
    const Matrix &	invMapper() const		{ return invMapper_; }
    const std::string & path() const		{ return path_; }
    const std::string & name() const		{ return name_; }
    DataSectionPtr	pDirSection();
    static DataSectionPtr openSettings( const std::string & path );

    int minGridX() const;  int maxGridX() const;
    int minGridY() const;  int maxGridY() const;
    int minLGridX() const; int maxLGridX() const;  // local 坐标
    int minLGridY() const; int maxLGridY() const;

    void gridToLocal( int x, int y, int& lx, int& ly ) const;
    void boundsToGrid( const BoundingBox& bb,
        int& minGridX, int& minGridZ, int& maxGridX, int& maxGridZ ) const;
    void gridToBounds( int minGridX, int minGridY,
        int maxGridX, int maxGridY, BoundingBox& retBB ) const;
    bool inLocalBounds( const int gridX, const int gridZ ) const;
    bool inWorldBounds( const int gridX, const int gridZ ) const;

    bool condemned()			{ return condemned_; }
    void condemn();

    std::string outsideChunkIdentifier( const Vector3 & localPoint, bool checkBounds = true ) const;
    std::string outsideChunkIdentifier( int x, int z, bool checkBounds = true ) const;
    static bool gridFromChunkName( const std::string& chunkName, int16& x, int16& z );
private:
    ChunkSpacePtr	pSpace_;
    Matrix			mapper_;
    Matrix			invMapper_;
    std::string		path_;
    std::string		name_;
    DataSectionPtr	pDirSection_;
    int				minGridX_, maxGridX_, minGridY_, maxGridY_;
    int				minLGridX_, maxLGridX_, minLGridY_, maxLGridY_;
    bool			condemned_;
    bool			singleDir_;
};
```

**关键点**：

- `mapper_` / `invMapper_` 是 local↔world 的变换。同一个 space 可以挂多个 mapping（同一世界的不同区域、或副本的不同入口变换）。
- `condemn()` 标记 mapping 被废弃，`ChunkManager::mappingCondemned()` 会清理所有引用它的 pending chunk。
- `outsideChunkIdentifier(x, z)` 把网格坐标编码成 chunk 文件名（见 §4）。

服务端可通过 `GeometryMappingFactory` 注入派生类型：

```cpp
// geometry_mapping.hpp:119
class GeometryMappingFactory
{
public:
    virtual GeometryMapping * createMapping( ChunkSpacePtr pSpace,
                const Matrix & m, const std::string & path,
                DataSectionPtr pSettings ) = 0;
};
```

`ChunkSpace::pMappingFactory_` 持有这个工厂，`addMapping` 时若设置了工厂则调它创建。

### 2.4 BaseChunkTree / ChunkTree

定义在 [base_chunk_tree.hpp](file:///workspace/src/lib/chunk/base_chunk_tree.hpp)。它是“作为 chunk item 的树”的基类（注意：不是空间可见性树，而是“种在 chunk 里的一棵树”这种实体，配合 SpeedTree 用）：

```cpp
// base_chunk_tree.hpp:27
class BaseChunkTree : public ChunkItem, public Aligned
{
public:
    BaseChunkTree();
    ~BaseChunkTree();
    virtual void toss( Chunk * pChunk );
protected:
    const Matrix & transform() const { return this->transform_; }
    virtual void setTransform( const Matrix & transform );
    const BSPTree * bspTree() const;
    void setBSPTree( const BSPTree * bspTree );
    const BoundingBox & boundingBox() const { return this->boundingBox_; }
    void setBoundingBox( const BoundingBox & bbox );
private:
    Matrix transform_;
    const BSPTree * bspTree_;
    BoundingBox     boundingBox_;
};
```

`ChunkTree`（[chunk_tree.hpp](file:///workspace/src/lib/chunk/chunk_tree.hpp)）和 `ServerChunkTree`（[server_chunk_tree.hpp](file:///workspace/src/lib/chunk/server_chunk_tree.hpp)）派生自它。

---

## 3. Chunk 主体

### 3.1 Chunk 类

`Chunk` 是场景图的节点，定义在 [chunk.hpp](file:///workspace/src/lib/chunk/chunk.hpp)。注释中明确：

> A chunk is a convex three dimensional volume. It contains a description of the scene objects that reside inside it. Scene objects include lights, entities, sounds, and general drawable scene objects called items. It also defines the set of planes that form its boundary (with the exception of chunks reached through internal portals). Some planes have portals defined on them which indicate that a neighbouring chunk is visible through them.

#### 3.1.1 核心字段

```cpp
// chunk.hpp:67
class Chunk : public Aligned
{
public:
    Chunk( const std::string & identifier, GeometryMapping * pMapping,
            const Matrix& transform=Matrix::identity,
            const BoundingBox& localBounds=BoundingBox::s_insideOut_ );
    ~Chunk();

    bool load( DataSectionPtr pSection );
    ChunkItemFactory::Result loadItem( DataSectionPtr pSection );
    void unload();
    void bind( bool form );
    void bindPortals( bool form, bool notifyCaches );
    void unbind( bool cut );
    void focus();
    void smudge();
    void resolveExterns( GeometryMapping * pDeadMapping = NULL );

    void addStaticItem( ChunkItemPtr pItem );
    void delStaticItem( ChunkItemPtr pItem );
    void addDynamicItem( ChunkItemPtr pItem );
    bool modDynamicItem( ChunkItemPtr pItem, const Vector3 & oldPos,
        const Vector3 & newPos, const float diameter = 1.f,
        bool bUseDynamicLending = false);
    void delDynamicItem( ChunkItemPtr pItem, bool bUseDynamicLending=true );
    void jogForeignItems();
    bool addLoanItem( ChunkItemPtr pItem );
    bool delLoanItem( ChunkItemPtr pItem, bool bCanFail=false );
    bool isLoanItem( ChunkItemPtr pItem ) const;

#ifndef MF_SERVER
    void drawBeg();
    void drawEnd();
    bool drawSelf( bool lentOnly = false );
    void drawCaches();
#endif

    Chunk *	findClosestUnloadedChunkTo( const Vector3 & point, float * pDist );
    bool	contains( const Vector3 & point, const float radius = 0.f ) const;
    bool	owns( const Vector3 & point );
    // ...
private:
    std::string		identifier_;
    int16			x_;
    int16			z_;
    GeometryMapping * pMapping_;
    ChunkSpace		* pSpace_;

    bool			isOutsideChunk_;
    bool			hasInternalChunks_;
    bool			isAppointed_;   // 主线程认可的权威版本
    bool			loading_;
    bool			loaded_;
    bool			isBound_;
    bool			completed_;     // 所有 shell 都 focussed
    int				focusCount_;

    Matrix			unmappedTransform_;
    Matrix			transform_;
    Matrix			transformInverse_;

    BoundingBox		localBB_;
    BoundingBox		boundingBox_;
    bool			boundingBoxReady_;
    bool			gotShellModel_;
#ifndef MF_SERVER
    BoundingBox		visibilityBox_;
    BoundingBox		visibilityBoxCache_;
    uint32			visibilityBoxMark_;
    StaticLightValueCachePtr	lightValueCache_;
#endif
    Vector3			centre_;

    ChunkBoundaries		bounds_;	// 物理边界（凸）
    ChunkBoundaries		joints_;	// 逻辑连接（散布）

    // 注意：以下字段不可被加载线程写——见源码注释
    uint32				drawMark_;
    uint32				traverseMark_;
    uint32				reflectionMark_;
    float				pathSum_;

    ChunkCache * *		caches_;

    typedef std::vector<ChunkItemPtr>	Items;
    Items				selfItems_;     // 自有静态物
    Items				dynoItems_;     // 动态物
    Items				swayItems_;     // 受摆动物

    class Lender : public ReferenceCount
    {
    public:
        Chunk *				pLender_;
        Items				items_;
    };
    typedef SmartPointer<Lender> LenderPtr;
    typedef std::vector<LenderPtr>	Lenders;
    Lenders				lenders_;
    typedef std::vector<Chunk *>	Borrowers;
    Borrowers			borrowers_;
    VectorNoDestructor<Items *>		lentItemLists_;

    std::string			label_;
    Chunk *				fringeNext_;
    Chunk *				fringePrev_;
    bool				inTick_;
    bool				removable_;
#if UMBRA_ENABLE
    Umbra::Cell*				pUmbraCell_;
    std::vector<UmbraDrawItemCollection*>	umbraDrawItemCollections_;
    std::vector<UmbraDrawItem*>	umbraDrawItems_;
    static std::vector<Chunk*>*	s_umbraChunks_;
#endif
    // ...
};
```

#### 3.1.2 Chunk 状态机

源码注释里强调：`loading_` / `loaded_` / `isBound_` / `completed_` 是 chunk 的四个关键状态。典型转移：

```
   ┌─────────┐ load()      ┌─────────┐ bind(form)   ┌─────────┐ all shells focussed  ┌────────────┐
   │ created │ ─────────► │ loaded  │ ───────────► │ bound   │ ────────────────────► │ completed  │
   └─────────┘            └─────────┘              └─────────┘                        └────────────┘
        │                       │                       │
        │ unload()              │ unload()              │ unbind(cut)
        ▼                       ▼                       ▼
   (destructed)           (destructed)            (back to loaded)
```

注意：

- `appointAsAuthoritative()` 把后台加载完的 chunk 标记为主线程认可的权威版本（`isAppointed_=true`）。
- `focus()` 把 `focusCount_` 自增；`focussed()` 返回 `focusCount_ > 0`。`completed_` 要求所有内嵌 shell 都 focussed。
- `removable_` 标记 chunk 可被回收（卸载）。
- 加载线程**禁止**触碰 `drawMark_`/`traverseMark_`/`reflectionMark_`/`pathSum_`——这些只能由主线程写。源码在 `chunk.hpp:389` 附近有显式注释。

#### 3.1.3 Item 分类

`Chunk` 持有四类 item：

| 容器 | 含义 | 添加接口 |
| --- | --- | --- |
| `selfItems_` | chunk 自有的静态 item（编辑器放置的模型/光源/地形等） | `addStaticItem` |
| `dynoItems_` | 动态 item（运行时进入的 entity、临时特效） | `addDynamicItem` |
| `swayItems_` | 需要做“风吹摆动”计算的 item 子集 | 由 item 自身 `WANTS_SWAY` 标志决定 |
| `lenders_` / `borrowers_` | “借出/借入”——item 跨 chunk 边界可见时由源 chunk 借给目标 chunk | `addLoanItem` / `addDynamicItem(bUseDynamicLending=true)` |

`jogForeignItems()` 在 chunk 边界变化时把“被外 chunk 借进来”的 item 重新洗牌。

#### 3.1.4 ChunkCache 注册机制

`Chunk` 持有 `caches_`（`ChunkCache**`），通过 `Chunk::registerCache(TouchFunction)` 分配 ID。`ChunkCache::Instance<T>` 模板提供类型安全访问：

```cpp
// chunk.hpp:471
class ChunkCache
{
public:
    virtual void draw() {}                  ///< chunk drawn
    virtual int focus() { return 0; }        ///< chunk focussed (ret focusCount)
    virtual void bind( bool isUnbind ) {}    ///< chunk bound / unbound
    virtual bool load( DataSectionPtr )		{ return true; }  ///< chunk loaded
    static void touch( Chunk & ) {}          ///< chunk touching this cache type

    template <class CacheT> class Instance
    {
    public:
        Instance() : id_( Chunk::registerCache( &CacheT::touch ) ) {}
        CacheT & operator()( Chunk & chunk ) const
        {
            ChunkCache * & cc = chunk.cache( id_ );
            if (cc == NULL) cc = new CacheT( chunk );
            return static_cast<CacheT &>(*cc);
        }
        bool exists( Chunk & chunk ) const;
        void clear( Chunk & chunk ) const;
    private:
        int id_;
    };
};
```

各 `Chunk*Cache`（如 `ChunkTerrainCache`、`ChunkLightCache`、`ChunkOverlappers`、`ChunkUserDataObjectCache`）都通过 `static Instance<XXX> instance;` 注册一个全局 ID，然后 `instance(chunk)` 取/建实例。

#### 3.1.5 边界迭代器

`Chunk` 提供 `piterator` 遍历所有已绑定的 portal：

```cpp
// chunk.hpp:216
class piterator
{
public:
    void operator++(int);
    bool operator==( const piterator & other ) const;
    ChunkBoundary::Portal & operator*();
    ChunkBoundary::Portal * operator->();
private:
    piterator( ChunkBoundaries & source, bool end );
    void scan();
    ChunkBoundaries::iterator			bit;
    ChunkBoundary::Portals::iterator	pit;
    ChunkBoundaries	& source_;
    friend class Chunk;
};

piterator pbegin() { return piterator( joints_, false ); }
piterator pend()   { return piterator( joints_, true ); }
```

`bounds_` 是物理边界（凸多面体的面集合），`joints_` 是逻辑连接（包括 internal portal 在内）。`pbegin/pend` 遍历的是 `joints_` 里所有已 bound 的 portal。

### 3.2 ChunkBoundary 与 Portal

定义在 [chunk_boundary.hpp](file:///workspace/src/lib/chunk/chunk_boundary.hpp)。`ChunkBoundary` 是一个面（`PlaneEq plane_`），上面可以开若干 `Portal`：

```cpp
// chunk_boundary.hpp:153
struct ChunkBoundary : public ReferenceCount
{
    ChunkBoundary(	DataSectionPtr pSection,
                    GeometryMapping * pMapping,
                    const std::string & ownerChunkName = "" );
    ~ChunkBoundary();

    const PlaneEq & plane() const		{ return plane_; }

    struct TraversalData
    {
        TraversalData(uint32 nextMark);
        Vector3 cameraPosition_;
        Vector3 nearPlanePositions_[4];
        uint32	nextMark_;
    };

    struct Portal
    {
        Portal(	DataSectionPtr pSection,
                const PlaneEq & plane,
                GeometryMapping * pMapping,
                const std::string & ownerChunkName = "" );
        ~Portal();

        bool resolveExtern( Chunk * pOwnChunk );
        void objectSpacePoint( int idx, Vector3& ret );
        Vector3 objectSpacePoint( int idx ) const;

        bool			internal;       // internal portal：连接的 chunk 嵌入本 chunk
        bool			permissive;     // 允许物体穿过（影响碰撞）

        Chunk			* pChunk;       // 目标 chunk（或特殊值 HEAVEN/EARTH/...）
        V2Vector		points;         // 2D 多边形顶点
        Vector3	uAxis;              // 局部 u 轴
        Vector3	vAxis;              // 局部 v 轴
        Vector3 origin;               // 局部原点
        Vector3	lcentre;            // 局部中心
        Vector3	centre;             // 世界中心
        PlaneEq			plane;         // 与 boundary 同
        std::string		label;

        Portal * findPartner( const Chunk * pCurrChunk ) const;

#ifndef MF_SERVER
        Portal2DRef traverse(
            const Matrix & world,
            const Matrix & worldInverse,
            Portal2DRef pClipPortal,
            const TraversalData& traversalData,
            float* nearDepth = NULL) const;
#endif
        enum
        {
            NOTHING = 0,
            HEAVEN = 1,        // 通往天空
            EARTH = 2,         // 通往地形
            INVASIVE = 3,      // 侵入式 portal（破坏外部凸包）
            EXTERN = 4,        // 外部未解析
            LAST_SPECIAL = 15
        };
        bool hasChunk() const  { return uintptr(pChunk) > uintptr(LAST_SPECIAL); }
        bool isHeaven() const  { return uintptr(pChunk) == uintptr(HEAVEN); }
        bool isEarth() const   { return uintptr(pChunk) == uintptr(EARTH); }
        bool isInvasive() const{ return uintptr(pChunk) == uintptr(INVASIVE); }
        bool isExtern() const  { return uintptr(pChunk) == uintptr(EXTERN); }
        // ...
#if UMBRA_ENABLE
        void		createUmbraPortal( Chunk* pOwner );
        Umbra::Model* createUmbraPortalModel();
        void		releaseUmbraPortal();
        Umbra::PhysicalPortal* pUmbraPortal_;
#endif
        mutable uint32	traverseMark_;
        static bool drawPortals_;
    };

    typedef std::vector<Portal*>	Portals;

    void validatePortals( DataSectionPtr boundarySection, DataSectionPtr chunkSection );
    void bindPortal( uint32 unboundIndex );
    void unbindPortal( uint32 boundIndex );
    void addInvasivePortal( Portal * pPortal );
    void splitInvasivePortal( Chunk * pChunk, uint i );

    PlaneEq		plane_;
    Portals		boundPortals_;     // 已绑定（目标 chunk 已加载）
    Portals		unboundPortals_;   // 未绑定（目标 chunk 未加载）
};

typedef std::vector<ChunkBoundaryPtr>	ChunkBoundaries;
```

**关键设计**：

1. **HEAVEN / EARTH 特殊 portal**：室外 chunk 顶部 portal 指向 `HEAVEN`，底部指向 `EARTH`。注释里说：“outside chunks will have six sides, and on top shall be heaven, and on bottom shall be earth.”——天空和地形不是普通 chunk，而是 portal 的特殊语义。
2. **internal portal**：当 portal 的 `internal=true` 时，它对应的 boundary 不是真正的边界，而是“被它指向的 chunk（及其连通的所有 chunk）所占据的空间应从本 chunk 减去”——这是室外 chunk 嵌入室内 chunk 的机制。
3. **boundPortals_ / unboundPortals_**：portal 的目标 chunk 还没加载时进 `unboundPortals_`；目标 chunk bound 后调用 `bindPortal` 把它移到 `boundPortals_`。
4. **invasive portal**：当一个 chunk 的凸包被另一个 chunk 部分侵入时（比如室内 chunk 探出室外），通过 `addInvasivePortal` / `splitInvasivePortal` 修复凸性。
5. **Portal2DRef**：portal 在屏幕空间投影后的复用包装，`Portal2DStore` 维护一个池，避免每次遍历都分配。
6. **Umbra 集成**：每个 portal 可创建一个 `Umbra::PhysicalPortal`，让 Umbra 在遮挡剔除时知道这里有个“可见通道”。

`ChunkExitPortal`（[chunk_exit_portal.hpp](file:///workspace/src/lib/chunk/chunk_exit_portal.hpp)）是 portal 的特殊子类，用于在 chunk 边界上做更精细的可见性裁剪。

### 3.3 ChunkBinding

定义在 [chunk_binding.hpp](file:///workspace/src/lib/chunk/chunk_binding.hpp)。注意它**不是 portal 的绑定**，而是“标记树节点之间的有向连接”：

```cpp
// chunk_binding.hpp:30
class ChunkBinding : public ChunkItem
{
public:
    virtual bool load( DataSectionPtr pSection );
    virtual bool save( DataSectionPtr pSection );

    UniqueID fromID_;
    UniqueID toID_;

    ChunkItemTreeNodePtr from() const { return from_; }
    ChunkItemTreeNodePtr to() const { return to_; }
private:
    ChunkItemTreeNodePtr from_;
    ChunkItemTreeNodePtr to_;
    static ChunkItemFactory::Result create( Chunk * pChunk, DataSectionPtr pSection );
    static ChunkItemFactory	factory_;
};

class ChunkBindingCache
{
public:
    void add(ChunkBindingPtr binding);
    void addBindingFrom_OnLoad(ChunkBindingPtr binding);
    void addBindingTo_OnLoad(ChunkBindingPtr binding);
    void connect(ChunkItemTreeNodePtr node);
private:
    typedef std::list<ChunkBindingPtr> BindingList;
    BindingList bindings_;
    BindingList waitingBindingFrom_;  // binding 先到、节点后到
    BindingList waitingBindingTo_;
};
```

它配合 `ChunkItemTreeNode`（见 §5.5）维护“marker 之间的引用关系”。

### 3.4 ChunkLoader

定义在 [chunk_loader.hpp](file:///workspace/src/lib/chunk/chunk_loader.hpp)，极简：

```cpp
// chunk_loader.hpp:23
class ChunkLoader
{
public:
    static void load( Chunk * pChunk, int priority = 0 );
    static void loadNow( Chunk * pChunk );
    static void findSeed( ChunkSpace * pSpace, const Vector3 & where,
        Chunk *& rpChunk );
};
```

`load` 把 chunk 的加载任务投递给 `BgTaskManager`，由后台线程读取 chunk 文件、构造 `ChunkBoundaries`、加载所有 `ChunkItem`。`loadNow` 在主线程同步加载（编辑器/服务端首块用）。`findSeed` 在空空间里找一个种子 chunk 作为流式加载的起点。

### 3.5 chunk 文件二进制 / XML 结构

格式定义在 [chunk format.txt](file:///workspace/src/lib/chunk/chunk%20format.txt) 与 [chunk_format.hpp](file:///workspace/src/lib/chunk/chunk_format.hpp)。原始 chunk 是 XML（被 `PackedSection` 包装后变成 `.chunk` 二进制），结构如下：

```
<root>	?LABEL
    *<include>
        <resource>	RES_FILE_NAME.chunk/.prefab/.mfo		</resource>
        <transform>
            ...
        </transform>
    </include>

    *<light>	?LABEL
        <type>			Omni/etc	</type>
        <enabled>		true		</enabled>
        <colour>		85.00 251.00 189.00		</colour>
        <innerRadius>	6.60		</innerRadius>
        <outerRadius>	13.50		</outerRadius>
        <transform>
            ...
        </transform>
    </light>

    *<model>	?LABEL
        +<resource>	RES_FILE_NAME.model/.mfo	</resource>
        ?<animation>
            <name>					ANIMATION_NAME		</name>
            <frameRateMultiplier>	.f					</frameRateMultiplier>
        </animation>
        <transform>
            ...
        </transform>
    </model>

    +<terrain>
        <resource>	RES_FILE_NAME.terrain </resource>
    </terrain>

    *<entity>
        ?<id>	ID			</id>
        <type>	TYPE_NAME	</type>
        <position>	.f	.f	.f	</position>
        <instantiate>	SERVER|CLIENT|IMPLICIT	</instantiate>
        [ other properties of this type ]
    <entity>

    *<sound>	?LABEL
        <resource>	RES_FILE_NAME.sound			</resource>
        <position>	.f	.f	.f	</position>
        <period>	.f	</period>
        <rate>		.f	</rate>
        <amplitude>	.f	</amplitude>
        <outerStartTime>	.f	</outerStartTime>
        <innerStartTime>	.f	</innerStartTime>
        <innerStopTime>		.f	</innerStopTime>
        <outerStopTime>		.f	</outerStopTime>
    </sound>

    +4<boundary>
        <normal>	.f	.f	.f	</normal>
        <d>	.f	</d>
        *<portal>	?LABEL
            ?<internal>		false	</internal>
            ?<permissive>	true	</permissive>
            ?<chunk>		SPACE_RELATIVE_FILE_NAME	</chunk>
            <uAxis>	.f	.f	.f	</uAxis>
            +3<point>	.f	.f	0	</point>	[0 for now]
        </portal>
    </boundary>

    <transform>
        ...
    </transform>

    <boundingBox>
        <min>	.f	.f	.f	</min>
        <max>	.f	.f	.f	</max>
    </boundingBox>
</root>
```

语法：`?`=0 或 1，`*`=0 或多，`+`=1 或多，`+n`=n 或多。

**关键说明**（来自 `chunk format.txt` 的 Notes）：

- portal 中若没有 `<chunk>` 引用，表示该 portal 不连通、不绘制。
- 特殊 chunk 标识 `'heaven'`：只画天空（gradient/clouds/sun/moon/stars）。`'earth'`：只画地形。
- 室外 chunk 有六个面，顶面是 heaven，底面是 earth。
- 被 include 的 chunk 边界面被忽略，只取 includes/models/lights/sounds 等。Include 在加载时展开内联，对客户端/服务端/脚本透明（编辑器除外）。Label 冲突通过追加 `_N` 解决。
- `internal` portal：被指向的 chunk（及其连通 chunk）所占空间从本 chunk 的非 internal boundary 定义的空间里减去。原本只用于“室外连室内”，但也可推广到“interior portal”。
- portal 中 `vAxis = normal × uAxis`，boundary 法线指向 chunk 内部。
- 只有 `IMPLICIT` 实例化的 entity 需要填 `id`；其它由 client 池或 cell 池分配。
- chunk 内除 `boundingBox` 外的所有数据都按 chunk 的 local space 解释（local space 由顶层 `<transform>` 定义）。

#### 3.5.1 室外 chunk 命名规则

`chunk_format.hpp` 提供了把网格坐标编码成 chunk 文件名的函数：

```cpp
// chunk_format.hpp:22
inline std::string outsideChunkIdentifier( int gridX, int gridZ, bool singleDir = false )
{
    char chunkIdentifierCStr[32];
    std::string gridChunkIdentifier;

    uint16 gridxs = uint16(gridX), gridzs = uint16(gridZ);
    if (!singleDir)
    {
        if (uint16(gridxs + 4096) >= 8192 || uint16(gridzs + 4096) >= 8192)
        {
                bw_snprintf( chunkIdentifierCStr, sizeof(chunkIdentifierCStr),
                    "%01xxxx%01xxxx/sep/", int(gridxs >> 12), int(gridzs >> 12) );
                gridChunkIdentifier = chunkIdentifierCStr;
        }
        if (uint16(gridxs + 256) >= 512 || uint16(gridzs + 256) >= 512)
        {
                bw_snprintf( chunkIdentifierCStr, sizeof(chunkIdentifierCStr),
                    "%02xxx%02xxx/sep/", int(gridxs >> 8), int(gridzs >> 8) );
                gridChunkIdentifier += chunkIdentifierCStr;
        }
        if (uint16(gridxs + 16) >= 32 || uint16(gridzs + 16) >= 32)
        {
                bw_snprintf( chunkIdentifierCStr, sizeof(chunkIdentifierCStr),
                    "%03xx%03xx/sep/", int(gridxs >> 4), int(gridzs >> 4) );
                gridChunkIdentifier += chunkIdentifierCStr;
        }
    }
    bw_snprintf( chunkIdentifierCStr, sizeof(chunkIdentifierCStr),
        "%04x%04xo", int(gridxs), int(gridzs) );
    gridChunkIdentifier += chunkIdentifierCStr;

    return gridChunkIdentifier;
}
```

命名采用**多层 hex 目录**：每 4 位一级（`x`/`z` 同步分桶），避免单目录下文件过多。例如网格 `(0,0)` → `0000000 0o`（实际 `00000000o`），网格 `(17, -3)` 会被分到对应的高层 `sep/` 子目录。`singleDir=true` 时退化为扁平命名（编辑器临时用）。

---

## 4. ChunkItem 体系

### 4.1 ChunkItem 与 ChunkItemBase

定义在 [chunk_item.hpp](file:///workspace/src/lib/chunk/chunk_item.hpp)。

#### 4.1.1 ChunkItemBase

`ChunkItemBase` 是所有 chunk item 的根，声明 client/editor 通用接口：

```cpp
// chunk_item.hpp:48
class ChunkItemBase
{
public:
    enum WantFlags
    {
        WANTS_NOTHING		= 0,
        WANTS_DRAW			= 1 <<  0,
        WANTS_TICK			= 1 <<  1,
        WANTS_SWAY			= 1 <<  2,
        WANTS_NEST			= 1 <<  3,
        FORCE_32_BIT		= 1 << 31
    };
    static const uint32 USER_FLAG_SHIFT  =  8;
    static const uint32 CHUNK_FLAG_SHIFT = 24;
    enum { TYPE_DEPTH_ONLY = ( 1 << 0 ), };

    typedef std::set<Chunk*> Borrowers;

    explicit ChunkItemBase( WantFlags wantFlags = WANTS_NOTHING );
    virtual ~ChunkItemBase();

    virtual void toss( Chunk * pChunk ) { this->chunk( pChunk ); }  // chunk 切换
    virtual void draw() { }
    virtual void tick( float /*dTime*/ ) { }
    virtual void sway( const Vector3 & /*src*/, const Vector3 & /*dst*/, const float /*radius*/ ) { }
    virtual void lend( Chunk * /*pLender*/ ) { }
    virtual void nest( ChunkSpace * /*pSpace*/ ) { }

    virtual const char * label() const				{ return ""; }
    virtual uint32 typeFlags() const				{ return 0; }

    virtual void incRef() const		{ ++__count; }
    virtual void decRef() const		{ if(--__count == 0) delete this; }
    virtual int refCount() const	{ return __count; }

    Chunk * chunk() const			{ return pChunk_; }
    void chunk( Chunk * pChunk )	{ pChunk_ = pChunk; }

    bool wantsDraw() const { return !!(wantFlags_ & WANTS_DRAW); }
    bool wantsTick() const { return !!(wantFlags_ & WANTS_TICK); }
    bool wantsSway() const { return !!(wantFlags_ & WANTS_SWAY); }
    bool wantsNest() const { return !!(wantFlags_ & WANTS_NEST); }

    virtual bool reflectionVisible() { return false; }

    uint32 drawMark() const; void drawMark( uint32 val );
    uint32 depthMark() const; void depthMark( uint32 val );
    uint8 userFlags() const { return uint8(wantFlags_ >> USER_FLAG_SHIFT); }

    virtual bool addYBounds( BoundingBox& bb ) const	{	return false;	}
    virtual void syncInit() {}  // Umbra 同步初始化

    void addBorrower( Chunk* pChunk );
    void delBorrower( Chunk* pChunk );
    void clearBorrowers();
#if UMBRA_ENABLE
    UmbraDrawItem* pUmbraDrawItem() const { return pUmbraDrawItem_; }
#endif
private:
    ChunkItemBase & operator=( const ChunkItemBase & other );	// undefined
    mutable int __count;
    uint32 drawMark_;
    uint32 depthMark_;
protected:
    WantFlags wantFlags_;
    Chunk * pChunk_;
    Borrowers	borrowers_;
#if UMBRA_ENABLE
    void updateUmbraLenders();
    UmbraObjectProxyPtr createLender(Umbra::Cell* pCell);
    UmbraDrawItem*	pUmbraDrawItem_;
    typedef std::map<Umbra::Cell*, UmbraObjectProxyPtr> UmbraLenders;
    UmbraLenders	umbraLenders_;
#endif
    void lendByBoundingBox( Chunk * pLender, const BoundingBox & worldbb );
};
```

**`WantFlags` 是 item 与 chunk 系统的契约**：

| 标志 | 含义 | chunk 端处理 |
| --- | --- | --- |
| `WANTS_DRAW` | 需要每帧 `draw()` | 加入 `drawCaches` 流水线 |
| `WANTS_TICK` | 需要每帧 `tick(dTime)` | 加入 `ClientChunkSpace::tickItems_` |
| `WANTS_SWAY` | 受风吹摆动 | 加入 `Chunk::swayItems_` |
| `WANTS_NEST` | 需要 `nest(pSpace)` 在空间级回调 | 加入 homeless 列表 |

#### 4.1.2 SpecialChunkItem / ChunkItem

通过 `#ifdef EDITOR_ENABLED` 切换基类：

```cpp
// chunk_item.hpp:166
#ifdef EDITOR_ENABLED
#include "editor_chunk_item.hpp"
// EditorChunkItem 提供编辑器专用方法
#else
class ClientChunkItem : public ChunkItemBase
{
public:
    explicit ClientChunkItem( WantFlags wantFlags = WANTS_NOTHING ) :
        ChunkItemBase( wantFlags ) { }
};
typedef ClientChunkItem SpecialChunkItem;
#endif

class ChunkItem : public SpecialChunkItem
{
public:
    explicit ChunkItem( WantFlags wantFlags = WANTS_NOTHING ) : SpecialChunkItem( wantFlags ) { }
    virtual void syncInit() {}
#ifndef MF_SERVER
    void	toss( Chunk* pChunk );  // 客户端 toss 实现见 cpp
#endif
};

typedef SmartPointer<ChunkItem>	ChunkItemPtr;
```

#### 4.1.3 ChunkItemFactory 工厂模式

每个具体 item 类用 `DECLARE_CHUNK_ITEM` / `IMPLEMENT_CHUNK_ITEM` 宏自动注册一个工厂：

```cpp
// chunk_item.hpp:313
#define DECLARE_CHUNK_ITEM( CLASS )                                         \
    static ChunkItemFactory::Result create( Chunk * pChunk, 				\
        DataSectionPtr pSection );											\
    static ChunkItemFactory factory_;										\

#define IMPLEMENT_CHUNK_ITEM( CLASS, LABEL, PRIORITY )						\
    ChunkItemFactory CLASS::factory_( #LABEL, PRIORITY, CLASS::create );	\
    ChunkItemFactory::Result CLASS::create( Chunk * pChunk, 				\
            DataSectionPtr pSection )										\
    {																		\
        CLASS * pItem = new CLASS();										\
        std::string errorString; 											\
        if (pItem->load IMPLEMENT_CHUNK_ITEM_ARGS)							\
        {																	\
            if (!pChunk->addStaticItem( pItem ))							\
            { /* error log */ }												\
            return ChunkItemFactory::Result( pItem );						\
        }																	\
        delete pItem;														\
        return ChunkItemFactory::Result( NULL, errorString );				\
    }
```

工厂按 section 标签名（"model"、"light"、"terrain"...）注册到 `Chunk::pFactories_`，`Chunk::load` 遍历子 section 时按名查表分发。`priority()` 用于多个工厂匹配同名 section 时的优先级（如编辑器版本覆盖客户端版本）。

`ChunkItemFactory::Result` 携带成功标志、`ChunkItemPtr`、错误字符串、`onePerChunk`（一个 chunk 只允许一个该类型 item）。

### 4.2 ChunkModel

定义在 [chunk_model.hpp](file:///workspace/src/lib/chunk/chunk_model.hpp)。它是 chunk 里的可视模型，包一个 `SuperModel`：

```cpp
// chunk_model.hpp:63
class ChunkModel : public ChunkItem, public Aligned
{
    DECLARE_CHUNK_ITEM( ChunkModel )
    DECLARE_CHUNK_ITEM_ALIAS( ChunkModel, shell )  // shell 是别名（壳模型）
public:
    ChunkModel();
    ~ChunkModel();

    bool load( DataSectionPtr pSection, Chunk * pChunk = NULL );
    virtual void toss( Chunk * pChunk );
    void toss( Chunk * pChunk, SuperModel* extraModel );
    virtual void draw();
    virtual void lend( Chunk * pLender );
    virtual const char * label() const;
    virtual bool reflectionVisible() { return reflectionVisible_; }
    BoundingBox localBB() const;
protected:
    class SuperModel			* pSuperModel_;
    SuperModelAnimationPtr		pAnimation_;
    uint32						tickMark_;
    uint64						lastTickTimeInMS_;
    std::map<std::string,SuperModelDyePtr> tintMap_;
    std::vector<ChunkMaterialPtr> materialOverride_;
    float						animRateMultiplier_;
    FashionVector				fv_;
    Matrix						transform_;
    std::string					label_;
    ModelCompoundInstancePtr	pModelCompound_;
    bool						reflectionVisible_;
    BorrowedLightCombiner		borrowedLightCombiner_;  // 借光合并器
    mutable bool				calculateIsShellModel_;
    mutable bool				cachedIsShellModel_;
    virtual void				syncInit();
#if UMBRA_ENABLE
    bool						umbraOccluder_;
    std::string					umbraModelName_;
#endif
#if FMOD_SUPPORT
    SoundOccluder soundOccluder_;
#endif
    virtual void addStaticLighting( DataSectionPtr ds, Chunk* pChunk  );
    virtual bool addYBounds( BoundingBox& bb ) const;
    bool isShellModel( const DataSectionPtr pSection ) const;
    void tickAnimation();
};
```

**关键设计**：

- `DECLARE_CHUNK_ITEM_ALIAS( ChunkModel, shell )` 让 `<shell>` section 也由 `ChunkModel` 工厂处理——shell 是“chunk 的可视外壳”（房间墙体等）。
- `tickAnimation()` 用 `tickMark_` 与 `ChunkManager::tickMark()` 比对，避免一帧内多次 tick（chunk 可能多次 `draw`）。
- `borrowedLightCombiner_` 处理“从邻居 chunk 借光”的光照合并。
- `ChunkMaterial`（[chunk_model.hpp:39](file:///workspace/src/lib/chunk/chunk_model.hpp)）是一个 `Fashion`，用于材质 override。
- `ChunkModelObstacle`（[chunk_model_obstacle.hpp](file:///workspace/src/lib/chunk/chunk_model_obstacle.hpp)）是它的碰撞代理（`ChunkBSPObstacle` 子类），把模型的 BSP 树注册到 `Column` 的 `ObstacleTree`。

### 4.3 ChunkTerrain

定义在 [chunk_terrain.hpp](file:///workspace/src/lib/chunk/chunk_terrain.hpp)。把一个 `Terrain::BaseTerrainBlock` 嵌入 chunk：

```cpp
// chunk_terrain.hpp:40
class ChunkTerrain : public ChunkItem
{
    DECLARE_CHUNK_ITEM( ChunkTerrain )
public:
    ChunkTerrain();
    ~ChunkTerrain();

    virtual void	toss( Chunk * pChunk );
    virtual void	draw();
    virtual uint32 typeFlags() const;

    Terrain::BaseTerrainBlockPtr block()				{ return block_; }
    const BoundingBox & bb() const						{ return bb_; }
    void calculateBB();

    static bool outsideChunkIDToGrid( const std::string& chunkID, int32& x, int32& z );

#if EDITOR_ENABLED
    static Terrain::BaseTerrainBlockPtr loadTerrainBlockFromChunk( Chunk * pChunk );
#endif
    bool doingBackgroundTask() const;
#if UMBRA_ENABLE
    void disableOccluder();
    void enableOccluder();
#endif
    virtual void syncInit();
    virtual bool reflectionVisible() { return true; }  // 地形总是反射可见
protected:
    bool load( DataSectionPtr pSection, Chunk * pChunk, std::string* errorString = NULL );
    virtual bool addYBounds( BoundingBox& bb ) const;
    Terrain::BaseTerrainBlockPtr    block_;
    BoundingBox						bb_;
#if UMBRA_ENABLE
    Terrain::BaseRenderTerrainBlock::UMBRAMesh	umbraMesh_;
    bool										umbraHasHoles_;
    UmbraModelProxyPtr	pUmbraWriteModel_;
#endif
#if FMOD_SUPPORT
    SoundOccluder soundOccluder_;
#endif
};

class ChunkTerrainCache : public ChunkCache
{
public:
    ChunkTerrainCache( Chunk & chunk );
    ~ChunkTerrainCache();
    virtual int focus();
    void pTerrain( ChunkTerrain * pT );
    ChunkTerrain * pTerrain()				{ return pTerrain_; }
    const ChunkTerrain * pTerrain() const	{ return pTerrain_; }
    static Instance<ChunkTerrainCache>	instance;
private:
    Chunk * pChunk_;
    ChunkTerrain * pTerrain_;
    SmartPointer<ChunkTerrainObstacle>	pObstacle_;
};
```

`ChunkTerrainCache` 是“每 chunk 一个”的缓存，把 terrain 的碰撞代理 `ChunkTerrainObstacle` 注册到 `Column`。`focus()` 在 chunk 被聚焦时返回 1，触发 terrain 的 LOD 评估。详细的 terrain 数据结构见 [05-地形系统-terrain.md](file:///workspace/study-docs/05-地形系统-terrain.md)。

### 4.4 ChunkLight 体系

定义在 [chunk_light.hpp](file:///workspace/src/lib/chunk/chunk_light.hpp)。光源体系如下：

```
ChunkLight (抽象基类，定义 colour/toss/updateLight)
   ├─ ChunkMooLight (基于 Moo::Light 的实现基类，加 dynamicLight_/specularLight_)
   │     ├─ ChunkDirectionalLight   平行光
   │     ├─ ChunkOmniLight           全向光
   │     ├─ ChunkSpotLight           聚光灯
   │     └─ ChunkPulseLight          脉动光（WANTS_TICK，颜色/位置动画）
   └─ ChunkAmbientLight  环境光（不进 Moo 容器，单独缓存）
```

`ChunkLightCache` 是每 chunk 一个的光照缓存，组织四种光源容器：

```cpp
// chunk_light.hpp:232
class ChunkLightCache : public ChunkCache
{
public:
    const static int MAX_LIGHT_SEEP_DEPTH = 2;  // 光透过 portal 渗透最多 2 层
    virtual void draw();
    virtual void bind( bool isUnbind );
    static void touch( Chunk & chunk );

    Moo::LightContainerPtr pOwnLights()			{ return pOwnLights_; }
    Moo::LightContainerPtr pOwnSpecularLights()	{ return pOwnSpecularLights_; }
    Moo::LightContainerPtr pAllLights()			{ return pAllLights_; }       // 含渗透
    Moo::LightContainerPtr pAllSpecularLights()	{ return pAllSpecularLights_; }

    static Instance<ChunkLightCache>	instance;
    void dirtySeep( int seepDepth=MAX_LIGHT_SEEP_DEPTH, Chunk* parent=NULL );
    void moveOmni( const Moo::OmniLightPtr& omni, const Vector3& opos, float oradius, bool transient=false );
    void moveSpot( const Moo::SpotLightPtr& spot, const Vector3& opos, float oradius, bool transient=false );
private:
    void dirty();
    void collectLights();
    Chunk & chunk_;
    Moo::LightContainerPtr	pOwnLights_;            // chunk 自有光
    Moo::LightContainerPtr	pAllLights_;            // 含从邻居渗透的光
    Moo::LightContainerPtr	pOwnSpecularLights_;
    Moo::LightContainerPtr	pAllSpecularLights_;
    bool				lightContainerDirty_;
    bool				heavenSeen_;            // 是否见过 heaven portal
};
```

**光渗透（light seep）**：`MAX_LIGHT_SEEP_DEPTH = 2` 表示一个 chunk 的光可以最多穿过 2 层 portal 影响邻居。`dirtySeep` 在光源变化时递归通知邻居重算 `pAllLights_`。环境光相关的 `ChunkAmbientLight` 不参与 `LightContainer`，而是写入 `StaticLightValueCache`（见 [chunk.hpp:379](file:///workspace/src/lib/chunk/chunk.hpp)）。

### 4.5 ChunkSound / ChunkFlora / ChunkWater / ChunkFlare

- **ChunkSound**（[chunk_sound.hpp](file:///workspace/src/lib/chunk/chunk_sound.hpp)）：3D 定位声音，包 `Base3DSound *`，`WANTS_TICK` 用于周期性触发。
- **ChunkFlora**（[chunk_flora.hpp](file:///workspace/src/lib/chunk/chunk_flora.hpp) + `chunk_flora.ipp`）：植被，配合 `romp::Flora` 系统。
- **ChunkWater**（[chunk_water.hpp](file:///workspace/src/lib/chunk/chunk_water.hpp)）：水面，**派生自 `VeryLargeObject`** 而非直接 `ChunkItem`——它是 VLO 的一个具体子类。`syncInit(ChunkVLO*)` 在 VLO 引用首次创建时被调用。

```cpp
// chunk_water.hpp:26
class ChunkWater : public VeryLargeObject
{
public:
    ChunkWater( );
    ChunkWater( std::string uid );
    ~ChunkWater();
    bool load( DataSectionPtr pSection, Chunk * pChunk );
    virtual void draw( ) {}
    virtual void draw( Chunk* pChunk );
    virtual void sway( const Vector3 & src, const Vector3 & dst, const float diameter );
    static void simpleDraw( bool state );
#ifdef EDITOR_ENABLED
    virtual void dirty();
    virtual void edCommonChanged() {}
#endif
    virtual BoundingBox chunkPP( Chunk* pChunk );
    virtual void syncInit(ChunkVLO* pVLO);
protected:
    Water *				pWater_;
    Chunk*	pChunk_;
    Water::WaterState	config_;
private:
    static bool create( Chunk * pChunk, DataSectionPtr pSection, std::string uid );
    static VLOFactory	factory_;
    static bool s_simpleDraw;
    virtual bool addYBounds( BoundingBox& bb ) const;
};
```

- **ChunkFlare**（[chunk_flare.hpp](file:///workspace/src/lib/chunk/chunk_flare.hpp)）：镜头光晕，配合 `romp::LensEffectManager`。

### 4.6 ChunkMarker / ChunkMarkerCluster / ChunkItemTreeNode

定义在 [chunk_marker.hpp](file:///workspace/src/lib/chunk/chunk_marker.hpp) 与 [chunk_item_tree_node.hpp](file:///workspace/src/lib/chunk/chunk_item_tree_node.hpp)。

`ChunkItemTreeNode` 是“可构成父子树”的 item 基类，marker 和 marker cluster 都派生自它：

```cpp
// chunk_item_tree_node.hpp:31
class ChunkItemTreeNode : public ChunkItem
{
public:
    ChunkItemTreeNode()
        : ChunkItem( WANTS_DRAW )
    {
        id_ = UniqueID::generate();
        parentID_ = UniqueID::zero();
        numberChildren_ = 0;
        children_.clear();
        parent_ = NULL;
    }
    virtual bool load( DataSectionPtr pSection );
    virtual bool save( DataSectionPtr pSection );
    UniqueID id() const { return id_; }
    UniqueID parentID() const { return parentID_; }
    virtual void setParent(ChunkItemTreeNodePtr parent);
    ChunkItemTreeNodePtr getParent() const { return parent_; }
    void getCopyOfChildren(std::list<ChunkItemTreeNodePtr>& outList) const;
    bool fullyLoaded() { return allChildrenLoaded() && allBindingsLoaded(); }
    bool allChildrenLoaded() { return numberChildren_ == (int)children_.size(); }
    bool allBindingsLoaded();
    void removeThisNode();
    virtual void onRemoveChild() {}
    virtual void onAddChild() {}
    bool isNodeConnected() const;
    void setNewNode();
    void addBindingFromMeToThat(ChunkBindingPtr binding);
    void removeBindingFromMeToThat(ChunkBindingPtr binding);
    int numberChildren() { return numberChildren_; }
    static ChunkItemTreeNodeCache& nodeCache() { return nodeCache_; }
protected:
    void addChild(ChunkItemTreeNodePtr child);
    void removeChild(ChunkItemTreeNodePtr child);
    UniqueID id_;
    UniqueID parentID_;
    typedef std::list<ChunkBindingPtr> BindingList;
    BindingList bindings_;          // 我指向别人的 binding
    BindingList bindingsToMe_;       // 别人指向我的 binding（load 时构建）
    // ...
private:
    static ChunkItemTreeNodeCache nodeCache_;
    std::list<ChunkItemTreeNodePtr> children_;
    ChunkItemTreeNodePtr parent_;
    int numberChildren_;
    static ChunkBindingCache bindingCache_;
};

class ChunkItemTreeNodeCache
{
public:
    ChunkItemTreeNodePtr find(UniqueID nodeID) const;
    void fini();
private:
    void add(ChunkItemTreeNodePtr node);
    void addChildOnParentLoad(ChunkItemTreeNodePtr child);
    typedef std::map<UniqueID, ChunkItemTreeNodePtr> ChunkItemTreeNodeMap;
    ChunkItemTreeNodeMap nodeMap_;
    // marker 可能先于 cluster 加载——挂起等父节点
    typedef std::list<ChunkItemTreeNodePtr> ChunkItemTreeNodeList;
    typedef std::map<UniqueID, ChunkItemTreeNodeList> ChunkItemTreeNodeListMap;
    ChunkItemTreeNodeListMap waitingNodeListMap_;
};
```

**`ChunkMarker`**（[chunk_marker.hpp:29](file:///workspace/src/lib/chunk/chunk_marker.hpp)）：标记点，用于动态放置 entity/道具的占位符，派生自 `ChunkItemTreeNode`，持 `transform_` 和 `category_`。`ChunkMarkerCluster` 是 marker 的容器簇。

`ChunkItemTreeNodeCache` 维护全局 ID→节点映射，并处理“子先到父后到”的情况（`waitingNodeListMap_`）。

### 4.7 ChunkObstacle / ChunkQuadTree / ChunkOverlapper

#### 4.7.1 ChunkObstacle

定义在 [chunk_obstacle.hpp](file:///workspace/src/lib/chunk/chunk_obstacle.hpp)。它是注册到 `Column::pObstacleTree_` 的碰撞代理：

```cpp
// chunk_obstacle.hpp:152
class ChunkObstacle : public ReferenceCount, public Aligned
{
public:
    ChunkObstacle( const Matrix & transform, const BoundingBox* bb,
        ChunkItemPtr pItem );
    virtual ~ChunkObstacle();
    mutable uint32 mark_;
    const BoundingBox & bb_;
private:
    ChunkItemPtr pItem_;
public:
    Matrix	transform_;
    Matrix	transformInverse_;
    bool mark() const;
    static void nextMark() { s_nextMark_++; }
    static uint32 s_nextMark_;
    virtual bool collide( const Vector3 & source, const Vector3 & extent,
        CollisionState & state ) const = 0;
    virtual bool collide( const WorldTriangle & source, const Vector3 & extent,
        CollisionState & state ) const = 0;
    virtual bool clipAgainstBB( Vector3 & start, Vector3 & extent, float bloat = 0.f ) const;
    ChunkItemPtr pItem() const;
    Chunk * pChunk() const;
};

class ChunkBSPObstacle : public ChunkObstacle
{
public:
    ChunkBSPObstacle( const BSPTree & bsp, const Matrix & transform,
        const BoundingBox * bb, ChunkItemPtr pItem );
    virtual bool collide( const Vector3 & source, const Vector3 & extent,
        CollisionState & state ) const;
    virtual bool collide( const WorldTriangle & source, const Vector3 & extent,
        CollisionState & state ) const;
private:
    const BSPTree & bspTree_;
};
```

**碰撞回调**：

```cpp
// chunk_obstacle.hpp:44
class CollisionCallback
{
public:
    virtual int operator()( const ChunkObstacle & obstacle,
        const WorldTriangle & triangle, float dist )
    { return COLLIDE_STOP; }
    static CollisionCallback s_default;
};

const int COLLIDE_STOP = 0;
const int COLLIDE_BEFORE = 1;
const int COLLIDE_AFTER = 2;
const int COLLIDE_ALL = COLLIDE_BEFORE | COLLIDE_AFTER;
```

`operator()` 返回值决定碰撞查询是否继续：`COLLIDE_BEFORE` 表示对更近的碰撞也感兴趣，`COLLIDE_AFTER` 表示对更远的也感兴趣，`COLLIDE_ALL` 两者都要，`COLLIDE_STOP` 停止。预定义实现 `ClosestObstacle`（最近碰撞）、`ClosestObstacleEx`（带单面/双面判定）、`ClosestTerrainObstacle`（只找地形）。

`CollisionState` 携带 `sTravel_`（源点沿线距离）、`eTravel_`（终点沿线距离）、`dist_`（上次回调距离）等状态。

`ChunkObstacleOccluder` 是给 `LensEffectManager` 用的“光线遮挡检查器”；`ChunkRompCollider` / `ChunkRompTerrainCollider` 是粒子系统“落地”判定用的碰撞器。

#### 4.7.2 ChunkQuadTree

定义在 [chunk_quad_tree.hpp](file:///workspace/src/lib/chunk/chunk_quad_tree.hpp)。室外 chunk 的可见性四叉树，按 chunk 的 XY 网格切分，加速“相机视锥内哪些室外 chunk 可见”判定。`ClientChunkSpace::chunkQuadTrees_` 是 `hash_map<uint32, ChunkQuadTree*>`，按 chunk 网格坐标索引。

#### 4.7.3 ChunkOverlapper

定义在 [chunk_overlapper.hpp](file:///workspace/src/lib/chunk/chunk_overlapper.hpp)。处理“跨 chunk 边界的 item”——一个 item 几何上可能延伸到邻居 chunk：

```cpp
// chunk_overlapper.hpp:23
class ChunkOverlapper : public ChunkItem
{
    DECLARE_CHUNK_ITEM( ChunkOverlapper )
public:
    ChunkOverlapper();
    ~ChunkOverlapper();
    bool load( DataSectionPtr pSection, Chunk * pChunk, std::string* errorString = NULL );
    virtual void toss( Chunk * pChunk );
    void bind( bool isUnbind );
    bool bbReady() const			{ return pOverlapper_->boundingBoxReady(); }
    const BoundingBox & bb() const	{ return pOverlapper_->boundingBox(); }
    bool focussed() const			{ return pOverlapper_->focussed(); }
    void findAppointedChunk();
    Chunk * pOverlapper() const		{ return pOverlapper_; }
    void alsoInAdd( ChunkOverlappers * ai );
    void alsoInDel( ChunkOverlappers * ai );
private:
    std::string		overlapperID_;
    Chunk *			pOverlapper_;
    std::vector<ChunkOverlappers*>	alsoIn_;
};

class ChunkOverlappers : public ChunkCache
{
public:
    ChunkOverlappers( Chunk & chunk );
    virtual void bind( bool isUnbind );
    static void touch( Chunk & chunk );
    bool empty() const						{ return overlappers_.empty(); }
    bool complete() const					{ return complete_; }
    void add( ChunkOverlapperPtr pOverlapper, bool foreign = false );
    void del( ChunkOverlapperPtr pOverlapper, bool foreign = false );
    typedef std::vector<ChunkOverlapperPtr> Overlappers;
    const Overlappers & overlappers() const	{ return overlappers_; }
    Overlappers& overlappers() { return overlappers_; }
    void findAppointedChunks();
    static Instance<ChunkOverlappers>	instance;
private:
    void share();
    void copyFrom( ChunkOverlappers & oth );
    void checkIfComplete( bool checkNeighbours );
    Chunk & chunk_;
    Overlappers	overlappers_;
    Overlappers foreign_;
    bool		bound_;
    bool		halfBound_;
    bool		complete_;
    bool		binding_;
};
```

`ChunkOverlapper` 是“在邻居 chunk 里登记的引用条目”，`ChunkOverlappers` 是“本 chunk 收到的所有 overlapper 引用”的缓存。`foreign_` 表示来自邻居的引用，`complete_` 表示所有 overlapper 都已加载完毕（用于决定 chunk 是否可以 `completed`）。

### 4.8 ChunkVLO 与 VeryLargeObject

定义在 [chunk_vlo.hpp](file:///workspace/src/lib/chunk/chunk_vlo.hpp)。VLO = Very Large Object，**跨多个 chunk 的大型物体**（水面、超大桥梁、远景模型等）。一个 VLO 物理上存在一份，多个 chunk 通过 `ChunkVLO` 引用它：

```cpp
// chunk_vlo.hpp:37
class VLOFactory
{
public:
    typedef bool (*Creator)( Chunk * pChunk, DataSectionPtr pSection, std::string uid );
    VLOFactory( const std::string & section, int priority = 0, Creator creator = NULL );
    virtual bool create( Chunk * pChunk, DataSectionPtr pSection, std::string uid ) const;
    int priority() const;
private:
    int		priority_;
    Creator creator_;
};

class VeryLargeObject : public SafeReferenceCount, public EditorChunkCommonLoadSave
{
public:
    typedef StringHashMap<VeryLargeObjectPtr> UniqueObjectList;
    typedef std::list<ChunkVLO*> ChunkItemList;
    VeryLargeObject();
    VeryLargeObject( std::string uid, std::string type );
    ~VeryLargeObject();
    void setUID( std::string uid );
#ifdef EDITOR_ENABLED
    virtual void cleanup() {}
    virtual void saveFile(Chunk* pChunk=NULL) {}
    virtual void save();
    virtual void drawRed(bool val) {}
    virtual void highlight(bool val) {}
    virtual void edDelete( ChunkVLO* instigator );
    virtual const char * edClassName() { return "VLO"; }
    virtual const Matrix & edTransform() { return Matrix::identity; }
    virtual bool edEdit( class GeneralEditor & editor, const ChunkItemPtr pItem ) { return false; }
    virtual bool edShouldDraw();
    virtual std::string type() { return type_; }
    virtual bool isObjectCreated() const;
    MetaData::MetaData& metaData() { return metaData_; }
    ChunkItemList chunkItems() const;
    virtual bool visibleInside() const { return true; }
    virtual bool visibleOutside() const { return true; }
    void lastDbItem( ChunkItem * item ) { lastDbItem_ = item; }
    ChunkItem * lastDbItem() const { return lastDbItem_; }
    virtual int numTriangles() const { return 0; }
    virtual int numPrimitives() const { return 0; }
    virtual std::string edAssetName() const { return "VLO"; }
    static std::string generateUID();
    static void updateSelectionMark() { currentSelectionMark_++; }
    static uint32 selectionMark() { return currentSelectionMark_; }
    virtual bool edCheckMark(uint32 mark);
    static void deleteUnused();
    static void saveAll();
#endif
    virtual void objectCreated();
    bool shouldRebuild() { return rebuild_; }
    void shouldRebuild( bool rebuild ) { rebuild_ = rebuild; }
    virtual void dirty() {}
    virtual void draw( Chunk* pChunk ) {}
    virtual void lend( Chunk * pChunk ) {}
    virtual void unlend( Chunk * pChunk ) {}
    virtual void updateLocalVars( const Matrix & m ) {}
    virtual void updateWorldVars( const Matrix & m ) {}
    virtual const Matrix & origin() { return Matrix::identity; }
    virtual const Matrix & localTransform( ) { return Matrix::identity; }
    virtual const Matrix & localTransform( Chunk * pChunk ) { return Matrix::identity; }
    virtual void sway( const Vector3 & src, const Vector3 & dst, const float diameter ) {}
    virtual void tick( float dTime ){};
    virtual BoundingBox chunkPP( Chunk* pChunk ) { return BoundingBox::s_insideOut_; };
    std::string getUID() const { return uid_; }
    void addItem( ChunkVLO* item );
    void removeItem( ChunkVLO* item);
    BoundingBox& boundingBox() { return bb_; }
    ChunkVLO* containsChunk( const Chunk * pChunk ) const;
    static VeryLargeObjectPtr getObject( const std::string& uid );
    virtual void syncInit(ChunkVLO* pVLO) {}
    static void tickAll( float dTime );
protected:
    static UniqueObjectList s_uniqueObjects_;
    friend class EditorChunkVLO;
    std::string chunkPath_;
    BoundingBox bb_;
    std::string uid_;
    std::string type_;
    ChunkItemList itemList_;
#ifdef EDITOR_ENABLED
    DataSectionPtr dataSection_;
    bool listModified_;
    bool objectCreated_;
    ChunkItem * lastDbItem_;
    MetaData::MetaData metaData_;
#endif
private:
    bool rebuild_;
#ifdef EDITOR_ENABLED
    uint32 selectionMark_;
    static uint32 currentSelectionMark_;
#endif
};

class ChunkVLO : public ChunkItem
{
public:
    explicit ChunkVLO( WantFlags wantFlags = WANTS_DRAW );
    ~ChunkVLO();
    virtual void draw();
    virtual void objectCreated() { }
    virtual void lend( Chunk * pChunk );
    virtual void toss( Chunk * pChunk );
    virtual void removeCollisionScene() {}
    virtual void updateTransform( Chunk * pChunk ) {}
    virtual void sway( const Vector3 & src, const Vector3 & dst, const float diameter );
    bool load( DataSectionPtr pSection, Chunk * pChunk );
#ifdef EDITOR_ENABLED
    bool root() const { return creationRoot_; }
    void root(bool val) { creationRoot_ = val; }
    bool createVLO( DataSectionPtr pSection, Chunk* pChunk );
    bool createLegacyVLO( DataSectionPtr pSection, Chunk* pChunk, std::string& type );
    bool cloneVLO( DataSectionPtr pSection, Chunk* pChunk, VeryLargeObjectPtr pSource );
    bool buildVLOSection( DataSectionPtr pObjectSection, Chunk* pChunk, std::string& type, std::string& uid );
    virtual int edNumTriangles() const;
    virtual int edNumPrimitives() const;
#endif
    static bool loadItem( Chunk* pChunk, DataSectionPtr pSection );
    VeryLargeObjectPtr object() const{ return pObject_; }
    static void registerFactory( const std::string & section, const VLOFactory & factory );
    static void fini();
    virtual void syncInit();
protected:
    VeryLargeObjectPtr		pObject_;
private:
    typedef StringHashMap<const VLOFactory*> Factories;
    bool				dirty_;
    bool				creationRoot_;
    static Factories	*pFactories_;
    DECLARE_CHUNK_ITEM( ChunkVLO )
};
```

**关键设计**：

- `s_uniqueObjects_` 是全局 UID→VLO 映射；编辑器版本会强制把 UID 转小写。
- `creationRoot_` 标记“VLO 在哪个 chunk 创建”（其它 chunk 只是引用）。删除 VLO 必须从 creationRoot 触发。
- `VLOFactory` 是独立于 `ChunkItemFactory` 的二级工厂——因为 `<vlo>` section 内还嵌套 `<type>` 决定具体 VLO 子类（如 `ChunkWater`）。
- `tickAll(dTime)` 静态方法驱动所有 VLO 的 tick。
- `ChunkVLOObstacle`（[chunk_vlo_obstacle.hpp](file:///workspace/src/lib/chunk/chunk_vlo_obstacle.hpp)）是 VLO 的碰撞代理。

### 4.9 ChunkStationNode + StationGraph

定义在 [chunk_stationnode.hpp](file:///workspace/src/lib/chunk/chunk_stationnode.hpp) 与 [station_graph.hpp](file:///workspace/src/lib/chunk/station_graph.hpp)。这是 BigWorld 的 **NPC 寻路路点系统**：

```cpp
// chunk_stationnode.hpp:24
class ChunkStationNode : public ChunkItem
{
public:
    typedef std::map<UniqueID, bool>    LinkInfo;
    typedef LinkInfo::const_iterator    LinkInfoConstIter;

    ChunkStationNode();
    ~ChunkStationNode();
    bool load( DataSectionPtr pSection, Chunk* pChunk );

    UniqueID			id() const;
    StationGraph*		graph() const;
    Vector3				position() const;
    std::string         const &userString() const;
    void                id(UniqueID const &newid);
    void                graph(StationGraph *g);
    void                position(Vector3 const &pos);
    void                userString(std::string const &str);

    LinkInfoConstIter   beginLinks() const;
    LinkInfoConstIter   endLinks() const;
    LinkInfoConstIter	findLink(UniqueID const &id) const;
    size_t              numberLinks() const;
    bool                canTraverse(UniqueID const &other) const;
    bool                isLinkedTo(UniqueID const &other) const;
    virtual void    toss(Chunk *chunk);
    ChunkLinkPtr setLink(ChunkStationNode *other, bool canTraverse);
#ifdef EDITOR_ENABLED
    virtual bool isValid(std::string &failureMsg) const;
    virtual void makeDirty();
    void unlink();
#endif
protected:
    virtual bool loadName( DataSectionPtr pSection, Chunk * pChunk );
    virtual ChunkLinkPtr createLink() const;
    void removeLink(ChunkStationNode *other);
    ChunkLinkPtr findLink(ChunkStationNode const *other) const;
    ChunkLinkPtr getChunkLink(size_t idx) const;
    void beginMove();
    void endMove();
    void updateRegistration( Chunk *chunk );
private:
    void removeLink(ChunkLinkPtr link);
    void delLink(ChunkLinkPtr link);
    static ChunkItemFactory::Result create( Chunk * pChunk, DataSectionPtr pSection );
    static ChunkItemFactory	factory_;
    Vector3				        position_;
    UniqueID			        id_;
    StationGraph                *graph_;
    std::string                 userString_;
    std::vector<ChunkLinkPtr>   links_;
    LinkInfo                    preloadLinks_;  // 加载时 link 目标还没到，先记下
    size_t                      moveCnt_;
};
```

`StationGraph`（[station_graph.hpp:31](file:///workspace/src/lib/chunk/station_graph.hpp)）维护图的全局索引：

```cpp
class StationGraph
{
public:
    static StationGraph* getGraph( const UniqueID& graphName );
    static std::vector<StationGraph*> getAllGraphs();
    const UniqueID &    name() const;
    bool                isReady() const;
    bool   canTraverseFrom( const UniqueID& src, const UniqueID& dst );
    uint32 traversableNodes( const UniqueID& src, std::vector<UniqueID>& retNodeIDs );
    bool   worldPosition( const UniqueID& node, Vector3& retWorldPos );
    const  UniqueID& nearestNode( const Vector3& worldPos );
    void   registerNode( ChunkStationNode* node, Chunk* pChunk );
    void   deregisterNode( ChunkStationNode* node );
#ifdef EDITOR_ENABLED
    ChunkStationNode* getNode( const UniqueID& id );
    std::vector<ChunkStationNode*> getAllNodes();
    static ChunkStationNode* getNode( const UniqueID& graphName, const UniqueID& id );
    static void mergeGraphs( UniqueID const &graph1Id, UniqueID const &graph2Id,
        GeometryMapping * pMapping );
    void loadAllChunks( GeometryMapping * pMapping );
    bool save();
    static void saveAll();
    virtual bool isValid(std::string &failureMsg) const;
    bool updateNodeIds(const UniqueID &nodeId, const std::vector<UniqueID> &links);
#endif
private:
    class Node
    {
    public:
        Node();
        bool load( DataSectionPtr pSect, const Matrix& mapping );
#ifdef EDITOR_ENABLED
        bool save( DataSectionPtr pSect );
#endif
        bool hasTraversableLinkTo( const UniqueID& nodeId ) const;
        void addLink( const UniqueID& id );
        void delLink( const UniqueID& id );
        std::vector<UniqueID>& links();
        Vector3		worldPosition_;
        UniqueID	id_;
        std::string userString_;
        // ...
    };
};
```

**关键设计**：

- 节点用 `UniqueID` 标识，跨 chunk / 跨 mapping 持久化。
- `preloadLinks_` 处理 link 目标后到的情况（与 `ChunkItemTreeNodeCache::waitingNodeListMap_` 类似）。
- 编辑器支持 `mergeGraphs` 把两个独立图合并（开发期常见）。
- `StationGraph::Node` 是服务端运行时用的纯数据节点，`ChunkStationNode` 是它的编辑期 + chunk item 形态——服务端不需要 `ChunkItem` 部分。
- 与 `waypoint` 库协作：cellapp 用 `StationGraph` 做 NPC 路径规划。

### 4.10 ChunkUmbra / UmbraChunkItem / UmbraDrawItem

定义在 [chunk_umbra.hpp](file:///workspace/src/lib/chunk/chunk_umbra.hpp)、[umbra_chunk_item.hpp](file:///workspace/src/lib/chunk/umbra_chunk_item.hpp)、[umbra_draw_item.hpp](file:///workspace/src/lib/chunk/umbra_draw_item.hpp)、[umbra_draw_item_collection.hpp](file:///workspace/src/lib/chunk/umbra_draw_item_collection.hpp)、[umbra_proxies.hpp](file:///workspace/src/lib/chunk/umbra_proxies.hpp)。这是与 **Umbra** 第三方遮挡剔除库的集成层。

`ChunkUmbra` 是初始化器：

```cpp
// chunk_umbra.hpp:36
class ChunkUmbra
{
public:
    ChunkUmbra( DataSectionPtr configSection = NULL );
    ~ChunkUmbra();
    static void init( DataSectionPtr configSection = NULL );
    static void fini();
    static Umbra::Commander* pCommander();
    static void terrainOverride( SmartPointer< BWBaseFunctor0 > pTerrainOverride );
    static void repeat();
    static void tick();
    static bool softwareMode();
    static bool clipPlaneSupported();
    static void minimiseMemUsage();
private:
    static ChunkUmbra* s_pInstance_;
    ChunkCommander* pCommander_;
    ChunkUmbraServices*	pServices_;
    bool softwareMode_;
    bool clipPlaneSupport_;
};
```

`UmbraHelper` 提供调试开关（`drawTestModels` / `drawWriteModels` / `drawVoxels` / `drawSilhouettes` / `drawQueries` / `occlusionCulling` / `umbraEnabled` / `flushTrees` / `