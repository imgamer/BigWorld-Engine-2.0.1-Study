# 19. 功能纵切：AOI 与可见性

> 本文档从「功能主线」视角横切 BigWorld Engine 2.0.1 的多个模块，解析 **Area of Interest（兴趣区）与实体可见性** 这一 MMO 核心机制。所有结论来源于真实的 `.hpp/.cpp` 文件，必要处给出可点击源码链接；不确定的 API 处明确标注「参见 xxx.hpp」。
>
> 涉及模块：`src/server/cellapp/`、`src/server/cellappmgr/`、`src/lib/entitydef/`、`src/server/baseapp/`、`src/lib/connection/`、`src/lib/chunk/`、`src/lib/physics2/`。

---

## 目录

1. [问题背景：百万实体 + 千级并发](#1-问题背景百万实体--千级并发)
2. [AOI 的概念与 BigWorld 的分层解法](#2-aoi-的概念与-bigworld-的分层解法)
3. [Witness（目击者）：下行推送的源头](#3-witness目击者下行推送的源头)
4. [RangeList：cellapp 内的空间索引](#4-rangelistcellapp-内的空间索引)
5. [ProximityController：邻近触发器](#5-proximitycontroller邻近触发器)
6. [VisionController / ScanVisionController：可见性扫描](#6-visioncontroller--scanvisioncontroller可见性扫描)
7. [VisibilityController：被看见的高度](#7-visibilitycontroller被看见的高度)
8. [EntityCache 与 AoIUpdateScheme：优先级与更新节奏](#8-entitycache-与-aoiupdatescheme优先级与更新节奏)
9. [VolatileInfo 与 DataLoDLevel：决定推送什么](#9-volatileinfo-与-datalodlevel决定推送什么)
10. [跨 cell 边界：ghost 与 BufferedGhostMessage](#10-跨-cell-边界ghost-与-bufferedghostmessage)
11. [上行链路：客户端 → baseapp → cellapp](#11-上行链路客户端--baseapp--cellapp)
12. [下行链路：cellapp → baseapp → client](#12-下行链路cellapp--baseapp--client)
13. [cellappmgr 的负载均衡与 cell 分裂/合并](#13-cellappmgr-的负载均衡与-cell-分裂合并)
14. [cellapp 死亡时的可见性恢复](#14-cellapp-死亡时的可见性恢复)
15. [完整时序图：玩家移动 → AOI 变化 → 客户端增删实体](#15-完整时序图玩家移动--aoi-变化--客户端增删实体)
16. [chunk 边界与 cell 边界的关系](#16-chunk-边界与-cell-边界的关系)
17. [注意事项与未覆盖项](#17-注意事项与未覆盖项)

---

## 1. 问题背景：百万实体 + 千级并发

MMO 服务端的根本矛盾是：

- 世界里同时存在 **百万级** 的实体（NPC、怪物、掉落物、玩家……）。
- 每个在线玩家只关心自己周围一小撮实体的状态。
- 服务端要在每秒若干 tick 的频率下，为 **千级并发** 玩家各自计算出「该看见谁」，并把状态变化推送下去。

如果让每个客户端都收到全世界的更新，带宽和 CPU 都会瞬间崩溃。BigWorld 的解法是 **Area of Interest（AOI，兴趣区）**：客户端只接收「它能看到」的实体的更新，并且对每个实体的更新频率随距离衰减。

BigWorld 把这个问题的解法分散在三层进程上：

| 层 | 进程 | 职责 |
|---|---|---|
| 模拟层 | cellapp | 在 cell 上维护实体的空间位置、跑 RangeList、跑 Witness，计算每个玩家该看到谁 |
| 代理层 | baseapp | 持有 base 实体与客户端连接，把 cellapp 算出的下行数据转发给客户端 |
| 客户端层 | client | 接收 `enterAoI`/`createEntity`/`leaveAoI` 消息，创建/销毁 client entity |

本文档沿着「玩家移动 → cellapp 算 AOI → baseapp 转发 → 客户端增删实体」这条主线展开。

---

## 2. AOI 的概念与 BigWorld 的分层解法

### 2.1 AOI 是什么

AOI（Area of Interest）是一个以玩家为中心、随玩家移动的「关注圆盘」。落在这个圆盘内的其它实体，其状态变化会被推送给该玩家；离开圆盘的实体则从客户端移除。

BigWorld 的 AOI 不是一个单一的数据结构，而是几个机制的组合：

1. **空间索引**：cellapp 内每个 cell 维护一张 `RangeList`，按 X 轴和 Z 轴各排一条双向链表，实体移动时只需在链表里局部重排（`shuffle`），即可发现「谁越过我了」。
2. **触发器（Trigger）**：`RangeTrigger` 把「圆形/扇形区域」表示成两个触发节点（上界/下界），插进 RangeList。别的实体越过触发节点时回调 `triggerEnter`/`triggerLeave`。
3. **Witness**：每个有 client portion 的 entity 持有一个 `Witness`，它管理一个 `EntityCacheMap`（AoI 集合），并定期按优先级选 topN 推给客户端。
4. **优先级/LOD**：`EntityCache::updatePriority` 按距离算优先级，`DataLoDLevel` 决定不同距离推送哪些属性（细节层次）。

### 2.2 两种「看见」

BigWorld 区分两种基于 RangeList 的感知：

| 机制 | 类 | 触发对象 | 回调 |
|---|---|---|---|
| 邻近触发（trap） | `ProximityController` | 该实体自身半径内的**任意**实体 | `onEnterTrap`/`onLeaveTrap`（cell 脚本） |
| 视野触发（vision） | `VisionController`/`ScanVisionController` | 该实体视野扇形内的实体 | `onWitnessedEnter`/`onWitnessedLeave`（参见 entity_vision.hpp） |
| 被 AOI 看见 | `Witness` + `AoITrigger` | 玩家 AOI 半径内、且被 Witness 选中推送的实体 | 客户端 `enterAoI`/`createEntity` |

注意：**proximity/vision 是 cell 脚本层的感知**（用于怪物 aggro、AI 视野），而 **Witness 的 AoI 是面向客户端推送的感知**。两者底层共用同一套 RangeList/RangeTrigger 基础设施，但目的不同。

---

## 3. Witness（目击者）：下行推送的源头

### 3.1 Witness 是什么

[Witness](file:///workspace/src/server/cellapp/witness.hpp) 的类注释写得很清楚：

> This class is a witness to the movements and perceptions of a RealEntity. It is created when a client is attached to this entity. Its main activity centres around the management of an Area of Interest list.

也就是说，**Witness 是「带客户端的 real entity」的附加组件**。当 baseapp 通过 `enableWitness` 把一个 client portion 挂到 cellapp 的 real entity 上时，cellapp 就为它创建一个 `Witness`（参见 [real_entity.hpp](file:///workspace/src/server/cellapp/real_entity.hpp) 的 `enableWitness`/`disableWitness`/`pWitness()`）。

Witness 由 `RealEntity` 持有：

```cpp
// /workspace/src/server/cellapp/real_entity.hpp
class RealEntity
{
    // ...
    Witness * pWitness()                          { return pWitness_; }
    void enableWitness( BinaryIStream & data, Mercury::ReplyID replyID );
    void disableWitness( bool isRestore = false );
private:
    Witness * pWitness_;
    // ...
};
```

一个 entity 只有 real（非 ghost）才能有 Witness，因为只有 real 才是权威的、才能决定「我看到谁」。ghost 只是 real 在其它 cellapp 上的镜像。

### 3.2 Witness 的核心数据结构

```cpp
// /workspace/src/server/cellapp/witness.hpp
class Witness : public Updatable
{
private:
    typedef std::vector< EntityCache * > KnownEntityQueue;

    RealEntity       & real_;
    Entity           & entity_;

    KnownEntityQueue   entityQueue_;   // 已知实体（按优先级）队列
    EntityCacheMap     aoiMap_;        // AoI 集合：EntityID/Entity -> EntityCache

    float aoiHyst_;                    // AOI 半径的迟滞量
    float aoiRadius_;                  // AOI 半径

    int32 bandwidthDeficit_;           // 带宽欠账（上 tick 没发完的）
    int32 maxPacketSize_;              // 每包最大尺寸（bandwidthPerUpdate）

    IDAlias freeAliases_[256];         // IDAlias 池：客户端用 1 字节别名引用实体
    int    numFreeAliases_;

    Vector3 referencePosition_;        // 相对位置参考点（带宽压缩用）
    uint8   referenceSeqNum_;
    bool    hasReferencePosition_;

    int32  knownSpaceDataSeq_;         // 已发给客户端的 spaceData 序号
    AoITrigger * pAoITrigger_;         // AOI 触发器（决定进/出 AoI）
    // ...
};
```

关键点：

- `aoiMap_` 是当前 AoI 集合的权威记录，类型 [EntityCacheMap](file:///workspace/src/server/cellapp/entity_cache.hpp)。
- `entityQueue_` 是「已知实体」的优先级队列，Witness 每 tick 从中选 topN 推送。
- `pAoITrigger_` 是一个 `AoITrigger`（前置声明，参见 witness.hpp 顶部 `class AoITrigger;`），本质是个 RangeTrigger，半径 = `aoiRadius_`，负责把「进入/离开 AOI 圆盘」的事件回调到 `addToAoI`/`removeFromAoI`。
- `aoiHyst_` 是迟滞量：实体进入 AOI 用 `aoiRadius_`，离开要用 `aoiRadius_ + aoiHyst_`，避免在边界来回抖动造成频繁进出（参见 `setAoIRadius(radius, hyst)` 方法）。
- `IDAlias` 是 1 字节（0~254，`NO_ID_ALIAS = 0xff`）的客户端侧短 ID，避免每次推送都发 4 字节 EntityID。

### 3.3 进出 AoI

```cpp
// /workspace/src/server/cellapp/witness.hpp
void addToAoI( Entity * pEntity );
void removeFromAoI( Entity * pEntity );
```

- `addToAoI`：某实体进入 AoI 圆盘时由 `pAoITrigger_` 回调。它会创建一个 `EntityCache`，置 `ENTER_PENDING` 标志，放入 `aoiMap_` 和 `entityQueue_`。此时还**不立即**发 `createEntity`，而是先发一个轻量的 `enterAoI`（只含 EntityID + IDAlias），等客户端回 `requestEntityUpdate` 后才发完整 `createEntity`（参见 [entity_cache.hpp](file:///workspace/src/server/cellapp/entity_cache.hpp) 的 `ENTER_PENDING`/`REQUEST_PENDING`/`CREATE_PENDING` 标志位与第 8 节）。
- `removeFromAoI`：实体离开 AoI 圆盘（或被销毁）时回调，从 `aoiMap_` 删除，并向客户端发 `leaveAoI`。

`EntityCache` 的状态机标志（节选自 entity_cache.hpp）：

```cpp
enum   // Flags bits
{
    ENTER_PENDING   = 1 << 0, ///< Waiting to send enterAoI to client
    REQUEST_PENDING = 1 << 1, ///< Expecting requestEntityUpdate from client
    CREATE_PENDING  = 1 << 2, ///< Waiting to send createEntity to client
    GONE            = 1 << 3, ///< Waiting to remove from priority queue
    WITHHELD        = 1 << 4, ///< Do not send to client
    REFRESH         = 1 << 5, ///< Waiting to be removed and re-added to the AoI
};
```

这套「enterAoI → requestEntityUpdate → createEntity」的三段式握手，是 BigWorld 带宽优化的核心：先用极小包通知客户端「有新实体 N 进入你的 AoI」，客户端如果对该实体感兴趣（脚本可决定）才请求完整数据，否则可保持惰性。

---

## 4. RangeList：cellapp 内的空间索引

### 4.1 数据结构

[RangeListNode](file:///workspace/src/server/cellapp/range_list_node.hpp) 是 RangeList 中所有节点的基类。注释说：

> Range Nodes are used to keep track of the order of entities and triggers relative to the x axis or the z axis. This is used for AoI calculations and range queries.

每个节点维护**两条**双向链表（X 轴一条、Z 轴一条）：

```cpp
// /workspace/src/server/cellapp/range_list_node.hpp
class RangeListNode
{
public:
    enum Flags
    {
        FLAG_MAKES_CROSSINGS = 1,   // 实体节点：会制造「跨越」事件
        FLAG_WANTS_CROSSINGS = 2,   // 触发器节点：想知道谁跨越了我
        FLAG_LAST_BASE = 128
    };
    // ...
    RangeListNode *pPrevX_, *pNextX_, *pPrevZ_, *pNextZ_;
    uint16 flags_;
    uint16 order_;   // 用于稳定排序
};
```

链表两端是 [RangeListTerminator](file:///workspace/src/server/cellapp/range_list_node.hpp)，位置为 `±FLT_MAX`。[RangeList](file:///workspace/src/server/cellapp/cell_range_list.hpp) 只是首尾两个 terminator：

```cpp
// /workspace/src/server/cellapp/cell_range_list.hpp
class RangeList
{
public:
    void add( RangeListNode * pNode );
private:
    RangeListTerminator first_;
    RangeListTerminator last_;
};
```

**每个 cell 拥有一个 RangeList**（参见 [cell.hpp](file:///workspace/src/server/cellapp/cell.hpp)），cell 上的所有 real entity、所有 trigger 节点都插在这张表里。

### 4.2 节点类型

| 类 | 角色 | `x()`/`z()` 来源 |
|---|---|---|
| `RangeListTerminator` | 链表首尾 | `±FLT_MAX` |
| [EntityRangeListNode](file:///workspace/src/server/cellapp/entity_range_list_node.hpp) | 一个 entity 在表中的入口 | entity 当前位置 |
| [RangeTriggerNode](file:///workspace/src/server/cellapp/range_list_node.hpp) | 触发器的上/下界 | subject 位置 ± range |

`EntityRangeListNode` 由 entity 持有：

```cpp
// /workspace/src/server/cellapp/entity.hpp
EntityRangeListNode * pRangeListNode_;
// ...
EntityRangeListNode * pRangeListNode() const;
```

### 4.3 移动 = 局部 shuffle

当 entity 移动时，它的 `EntityRangeListNode` 在 X 链表和 Z 链表里的相对位置可能改变。BigWorld 不重排整张表，而是用 `shuffleXThenZ` / `shuffleX` / `shuffleZ` 做局部冒泡：

```cpp
// /workspace/src/server/cellapp/range_list_node.hpp
void shuffleXThenZ( float oldX, float oldZ );
void shuffleX( float oldX, float oldZ );
void shuffleZ( float oldX, float oldZ );
```

shuffle 过程中，如果某个 trigger 节点（`FLAG_WANTS_CROSSINGS`）被一个 entity 节点（`FLAG_MAKES_CROSSINGS`）越过，就回调：

```cpp
// /workspace/src/server/cellapp/range_list_node.hpp
virtual void crossedX( RangeListNode * node, bool positiveCrossing,
    float oldOthX, float oldOthZ ) {}
virtual void crossedZ( RangeListNode * node, bool positiveCrossing,
    float oldOthX, float oldOthZ ) {}
```

`RangeTriggerNode` 重写了 `crossedX`/`crossedZ`，它检查对方是否也落在另一条轴的范围内（`containsInZ`/`isInXRange`），若是则通知所属 `RangeTrigger` 调 `triggerEnter`/`triggerLeave`。这就是 BigWorld 用「两个轴各一条有序链表 + 跨越回调」近似实现圆形/矩形区域查询的方式——避免了每次都做 O(N) 距离扫描。

`RangeTrigger` 的核心判定（节选自 range_list_node.hpp）：

```cpp
bool wasInXRange( float x, float range ) const
{
    float subX = oldX_;
    volatile float lowerBound = subX - range;
    volatile float upperBound = subX + range;
    return (lowerBound < x) && (x <= upperBound);
}
```

注意 `volatile` 的注释：这是为了防止编译器用比节点定位时更高的浮点精度做判定，从而漏触发或误触发。这是一个非常细的工程细节。

---

## 5. ProximityController：邻近触发器

### 5.1 用途

[ProximityController](file:///workspace/src/server/cellapp/proximity_controller.hpp) 是「陷阱 / aggro 范围」的实现。注释：

> This class is the controller for placing proximity based triggers, or traps. When an entity crosses within a range of the source entity it will call the python script method onEnterTrap() and when an entity leaves the range, it will call the method onLeaveTrap()

典型用法：怪物 `self.addProximity(range=30)`，玩家进入 30 米内触发 `onEnterTrap`，怪物开始追击；玩家离开触发 `onLeaveTrap`，怪物回原位。

### 5.2 内部触发器

`ProximityController` 内部持有一个 `ProximityRangeTrigger`（定义在 [proximity_controller.cpp](file:///workspace/src/server/cellapp/proximity_controller.cpp)），它是 `RangeTrigger` 的子类：

```cpp
// /workspace/src/server/cellapp/proximity_controller.cpp
class ProximityRangeTrigger : public RangeTrigger
{
public:
    ProximityRangeTrigger( Entity & around, float range,
            ControllerID id, int userArg ) :
        RangeTrigger( around.pRangeListNode(), range ),
        // ...
    {}

    virtual void triggerEnter( RangeListNode * who );
    virtual void triggerLeave( RangeListNode * who );

    void standardEnter( Entity * pWho );
    void standardLeave( Entity * pWho );
    void nonstandardLeave( EntityID who );
};
```

`triggerEnter` 把回调转发到 entity 脚本：

```cpp
// /workspace/src/server/cellapp/proximity_controller.cpp
void ProximityRangeTrigger::standardEnter( Entity * pWho )
{
    PyObject * pArgs = (userArg_ != 0) ?
        Py_BuildValue( "(Ofii)", pWho, this->range(), id_, userArg_ ) :
        Py_BuildValue( "(Ofi)", pWho, this->range(), id_ );

    this->pEntity()->callback( "onEnterTrap", pArgs,
        "RealEntity::triggerEnter: ", false );
}
```

脚本侧签名（来自 proximity_controller.cpp 的 doc 注释）：

```python
def onEnterTrap( self, entityEntering, range, controllerID, userArg = 0 ):
def onLeaveTrap( self, entityLeaving, range, controllerID, userArg = 0 ):
def onLeaveTrapID( self, entityID, range, controllerID, userArg = 0 ):
```

### 5.3 onload 时的集合差异处理

当 entity 被 onload（从别的 cell 迁过来）时，新的 cell 上可能没有原来 trap 里的某些实体。`ProximityController::startReal` 用 `OnloadHandler` 处理这个差异：

```cpp
// /workspace/src/server/cellapp/proximity_controller.cpp
void ProximityController::startReal( bool isInitialStart )
{
    if (!isInitialStart && (pOnloadedSet_ != NULL))
    {
        pProximityTrigger_ = new ProximityRangeTrigger( this->entity(),
                range_, this->id(), this->userArg() );
        {
            OnloadHandler handler( pProximityTrigger_, *pOnloadedSet_ );
            pProximityTrigger_->pAlternateHandler( &handler );
            pProximityTrigger_->init();   // init 会把当前范围内的实体全部 triggerEnter
            pProximityTrigger_->pAlternateHandler( NULL );
        }
        // OnloadHandler 析构时，对「原来在 set 里但 init 没触发 enter」的实体
        // 调 standardLeave / nonstandardLeave，保证脚本状态一致
        delete pOnloadedSet_;
        pOnloadedSet_ = NULL;
    }
    else
    {
        this->setRange( range_ );
    }
}
```

`writeRealToStream` 会把当前 trap 内的实体 ID 列表一起序列化（遍历 lowerTrigger 到 upperTrigger 之间的 X 节点，并用 `containsInZ` 过滤），`readRealFromStream` 读回这个集合给 `OnloadHandler` 用。这就是跨 cell 迁移时 trap 状态保持一致的机制。

> 注意：proximity 的范围判定是**正方形**（X 和 Z 各独立判定），文档注释里明确说「This area is a square (done for efficiency)」，不是真正的圆。这是 RangeList 双轴设计的副作用。

---

## 6. VisionController / ScanVisionController：可见性扫描

### 6.1 VisionController

[VisionController](file:///workspace/src/server/cellapp/vision_controller.hpp) 是「视野扇形」的实现，用于 AI 的「我看得到谁」：

```cpp
// /workspace/src/server/cellapp/vision_controller.hpp
class VisionController : public Controller, public Updatable
{
public:
    VisionController( float visionAngle = 1.f, float visionRange = 20.f,
        float seeingHeight = 2.f, int updatePeriod = 10 );
    void update();   // 每 updatePeriod tick 跑一次
private:
    float visionAngle_;     // 视野张角（弧度）
    float visionRange_;     // 视野距离
    float seeingHeight_;    // 眼睛高度（用于高低差判定）
    int   updatePeriod_;    // 多少 tick 扫一次
    int   tickSinceLast_;
    VisionRangeTrigger* pVisionTrigger_;
    std::vector< EntityID > * pOnloadedVisible_;   // onload 时复用
};
```

它和 ProximityController 的区别：

- Proximity 是**圆形/正方形**，不关心朝向；Vision 是**扇形**，受 entity 朝向（yaw）约束。
- Vision 是 `Updatable`，按周期 `updatePeriod_` 周期性扫描，而不是纯靠 trigger 回调（因为朝向变化需要重新评估已有实体是否还在扇形内）。

`getYawOffset()` 返回当前朝向偏移，扇形以此为中心。`update()` 调用 `EntityVision::updateVisibleEntities(seeingHeight, yawOffset)` 重算可见集，并对进出的实体回调脚本（参见 [entity_vision.hpp](file:///workspace/src/server/cellapp/entity_vision.hpp) 的 `triggerVisionEnter`/`triggerVisionLeave`）。

### 6.2 ScanVisionController

[ScanVisionController](file:///workspace/src/server/cellapp/scan_vision_controller.hpp) 继承自 `VisionController`，增加「扫描」行为——视野扇形会周期性左右摆动：

```cpp
// /workspace/src/server/cellapp/scan_vision_controller.hpp
class ScanVisionController : public VisionController
{
private:
    float amplitude_;    // 摆动幅度
    float scanPeriod_;   // 摆动周期
    float timeOffset_;   // 相位偏移
};
```

它重写 `getYawOffset()`，在基础朝向上叠加一个 `amplitude * sin(2π * t / scanPeriod + timeOffset)` 之类的扫描量，使守卫的视线来回扫动。

### 6.3 EntityVision：vision 的 entity 侧入口

[EntityVision](file:///workspace/src/server/cellapp/entity_vision.hpp) 是 `EntityExtra`（entity 的可选附加组件），集中管理一个 entity 的「看」与「被看」：

```cpp
// /workspace/src/server/cellapp/entity_vision.hpp
class EntityVision : public EntityExtra
{
public:
    PyObject* addVision( float fov, float range, float seeingHeight,
                    int period = 10, int userArg = 0 );
    PyObject* addScanVision( float fov, float range, float seeingHeight,
        float amplitude, float scanPeriod, float timeOffset,
        int updatePeriod = 1, int userArg = 0 );

    // 属性
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, seeingHeight, seeingHeight );
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, visibleHeight, visibleHeight );
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, canBeSeen, canBeSeen );
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, visionAngle, visionAngle );
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, visionRange, visionRange );

    const EntitySet & updateVisibleEntities( float seeingHeight, float yawOffset );
    void triggerVisionEnter( Entity * who );
    void triggerVisionLeave( Entity * who );

    VisibilityController * getVisibility() const { return visibility_; }
    VisionController * getVision() const         { return vision_; }
private:
    EntitySet entitiesInVisionRange_;   // 落在视野距离内（扇形外也算）
    EntitySet visibleEntities_;         // 落在扇形内
    VisibilityController * visibility_;
    VisionController * vision_;
    bool shouldDropVision_;
};
```

脚本侧通过 `entity.vision.addVision(...)` / `entity.vision.addScanVision(...)` 添加视野，`entity.vision.entitiesInView()` 查询当前可见实体集。

---

## 7. VisibilityController：被看见的高度

[VisibilityController](file:///workspace/src/server/cellapp/visibility_controller.hpp) 控制的是「**这个 entity 能被别人看到多高**」，即它自身的可见性属性，而不是它去看别人：

```cpp
// /workspace/src/server/cellapp/visibility_controller.hpp
class VisibilityController : public Controller
{
public:
    VisibilityController( float visibleHeight = 2.f );
    virtual void writeGhostToStream( BinaryOStream & stream );
    virtual bool readGhostFromStream( BinaryIStream & stream );
    virtual void startGhost();
    virtual void stopGhost();
private:
    float visibleHeight_;
};
```

注意它只重写 `startGhost`/`stopGhost` 和 `writeGhostToStream`/`readGhostFromStream`——也就是说它是 **ghost 域** 的 controller，其状态会同步到所有 ghost。`visibleHeight` 决定这个 entity 在别人的 vision 判定里「眼睛高度」取多少（蹲下/趴下时降低，从而更难被发现）。这与 `EntityVision::visibleHeight()` 配对：`EntityVision` 既存「我看别人的眼睛高度 seeingHeight」，也存「别人看我的高度 visibleHeight」。

---

## 8. EntityCache 与 AoIUpdateScheme：优先级与更新节奏

### 8.1 EntityCache

[EntityCache](file:///workspace/src/server/cellapp/entity_cache.hpp) 是 Witness 对 AoI 内每个实体的缓存项：

```cpp
// /workspace/src/server/cellapp/entity_cache.hpp
class EntityCache
{
public:
    static const int MAX_LOD_LEVELS = 4;
    typedef double Priority;

    float updatePriority( const Vector3 & origin );   // 按距离算优先级
    void updateDetailLevel( Mercury::Bundle & bundle, float lodPriority );

    EntityConstPtr pEntity_;
    Flags flags_;
    AoIUpdateSchemeID updateSchemeID_;
    VehicleChangeNum vehicleChangeNum_;
    union { Priority priority_; EntityID dummyID_; };
    EventNumber    lastEventNumber_;            // 已推到客户端的事件号
    VolatileNumber lastVolatileUpdateNumber_;   // 已推的 volatile 号
    DetailLevel    detailLevel_;                // 当前 LOD 层级
    IDAlias        idAlias_;                    // 客户端侧短 ID
    EventNumber    lodEventNumbers_[ MAX_LOD_LEVELS ];  // 各 LOD 层的事件号
};
```

每个 cache 记录：

- `priority_`：到 witness 主人的距离优先级，决定推送先后。
- `detailLevel_`：当前对该实体使用的 LOD 层级（0 最详细，越大越粗）。
- `lastEventNumber_`/`lastVolatileUpdateNumber_`：增量推送的游标，避免重复发已推过的属性变更。
- `idAlias_`：1 字节客户端别名。
- `updateSchemeID_`：该实体用哪一套更新方案（见下）。

### 8.2 AoIUpdateScheme

[AoIUpdateScheme](file:///workspace/src/server/cellapp/aoi_update_schemes.hpp) 决定「距离如何影响更新权重」：

```cpp
// /workspace/src/server/cellapp/aoi_update_schemes.hpp
class AoIUpdateScheme
{
public:
    bool init( float minDelta, float maxDelta );
    double apply( float distance ) const
    {
        return (distance * distanceWeighting_ + 1.f) * weighting_;
    }
private:
    float weighting_;
    float distanceWeighting_;
};

class AoIUpdateSchemes
{
    static double apply( SchemeID scheme, float distance )
    {
        return schemes_[ scheme ].apply( distance );
    }
    static AoIUpdateScheme schemes_[ 256 ];   // 最多 256 套方案
};
```

不同实体可以绑定不同的 scheme（比如 BOSS 总是高优先级、风景 NPC 低优先级）。`EntityCache::updatePriority` 用 `AoIUpdateSchemes::apply(schemeID, distance)` 算出优先级，Witness 每 tick 据此对 `entityQueue_` 排序，选 topN 推送。这就是「每 tick 选 topN entity 推给客户端」的实现：**不是均匀分配带宽，而是按优先级贪心选取**，距离近的/重要的实体更新更频繁。

### 8.3 三段式握手与状态机

`EntityCache` 的 flags 状态机驱动了 enterAoI/createEntity 的握手（见第 3.3 节）。Witness 的私有方法对应这套流程：

```cpp
// /workspace/src/server/cellapp/witness.hpp
void addToSeen( EntityCache * pCache );
void deleteFromSeen( Mercury::Bundle & bundle, KnownEntityQueue::iterator iter, EntityID id = 0 );
void deleteFromClient( Mercury::Bundle & bundle, EntityCache * pCache, EntityID id = 0 );
void handleEnterPending( Mercury::Bundle & bundle, KnownEntityQueue::iterator iter );
void sendEnter( Mercury::Bundle & bundle, EntityCache * pCache );
void sendCreate( Mercury::Bundle & bundle, EntityCache * pCache );
```

以及客户端回包入口：

```cpp
void requestEntityUpdate( EntityID id, EventNumber * pEventNumbers, int size );
```

客户端收到 `enterAoI` 后，按需调用 `requestEntityUpdate` 请求完整创建数据，Witness 收到后清 `REQUEST_PENDING`、置 `CREATE_PENDING`，并在后续 tick 发 `createEntity`。

---

## 9. VolatileInfo 与 DataLoDLevel：决定推送什么

### 9.1 VolatileInfo：哪些属性频繁变化

[VolatileInfo](file:///workspace/src/lib/entitydef/volatile_info.hpp) 描述 entity 的哪些朝向/位置属性是「频繁变化、需要高频推送」的：

```cpp
// /workspace/src/lib/entitydef/volatile_info.hpp
class VolatileInfo
{
public:
    VolatileInfo( float positionPriority = -1.f,
        float yawPriority = -1.f,
        float pitchPriority = -1.f,
        float rollPriority = -1.f );
    bool shouldSendPosition() const { return positionPriority_ > 0.f; }
    int dirType( float priority ) const;     // 哪些轴的朝向要发
    bool hasVolatile( float priority ) const;
    static const float ALWAYS;
private:
    float positionPriority_;
    float yawPriority_;
    float pitchPriority_;
    float rollPriority_;
};
```

`positionPriority_ > 0` 表示位置是 volatile（要持续推送），`-1` 表示不发。`EntityCache::lastVolatileUpdateNumber_` 就是 volatile 增量推送的游标。volatile 属性用相对位置参考点（`referencePosition_`）压缩成 3 个 uint8，极大节省带宽。

### 9.2 DataLoDLevel：距离决定细节层次

[DataLoDLevel](file:///workspace/src/lib/entitydef/data_lod_level.hpp) 定义 LOD 切换阈值：

```cpp
// /workspace/src/lib/entitydef/data_lod_level.hpp
class DataLoDLevel
{
public:
    float low() const;    // 优先级低于此 -> 升到更详细层
    float high() const;   // 优先级高于此 -> 降到更粗层
    float start() const;  // 起始距离
    float hyst() const;   // 迟滞量
    enum { OUTER_LEVEL = -2, NO_LEVEL = -1 };
};

class DataLoDLevels
{
public:
    bool needsMoreDetail( int level, float priority ) const;
    bool needsLessDetail( int level, float priority ) const;
private:
    DataLoDLevel level_[ MAX_DATA_LOD_LEVELS + 1 ];
};
```

`EntityCache::updateDetailLevel(bundle, lodPriority)` 用 `DataLoDLevels` 判断是否要切换 LOD 层。切换时，Witness 会把新 LOD 层对应的属性集（在 entitydef 的 `<DetailLevels>` 里定义）的初始值发给客户端，之后只发该层 volatile + 变更属性。距离越远，LOD 层越粗，推送的属性越少、频率越低。这就是「同一实体近处看到全身动作、远处只看到位置」的实现。

### 9.3 DataDescription 的分发标志

[DataDescription](file:///workspace/src/lib/entitydef/data_description.hpp) 用 `EntityDataFlags` 标记每个属性的分发范围：

```cpp
// /workspace/src/lib/entitydef/data_description.hpp
enum EntityDataFlags
{
    DATA_GHOSTED     = 0x01,  // 同步到 ghost
    DATA_OTHER_CLIENT= 0x02,  // 发给其它客户端（即 AoI 推送）
    DATA_OWN_CLIENT  = 0x04,  // 发给自己的客户端
    DATA_BASE        = 0x08,  // 发给 base
    DATA_CLIENT_ONLY = 0x10,  // 仅客户端静态数据
    DATA_PERSISTENT  = 0x20,  // 持久化到 DB
    DATA_EDITOR_ONLY = 0x40,  // 仅编辑器
    DATA_ID          = 0x80   // DB 索引列
};
```

Witness 推送时只关心带 `DATA_OTHER_CLIENT`（且按 LOD 层过滤）的属性；带 `DATA_GHOSTED` 的会同步到 ghost；带 `DATA_PERSISTENT` 的进 `writeToDB`/`loadFromDB` 流程（见文档 20）。

---

## 10. 跨 cell 边界：ghost 与 BufferedGhostMessage

### 10.1 为什么需要 ghost

一个 cellapp 进程只持有世界的一部分 cell。当一个 real entity 的 AOI 圆盘跨越 cell 边界时，它需要「看见」邻居 cellapp 上的实体。BigWorld 不把邻居 cell 的实体复制成 real，而是创建 **ghost**：real entity 在每个邻居 cellapp 上有一个轻量镜像（haunt），real 把状态变化广播给所有 haunt。

[RealEntity](file:///workspace/src/server/cellapp/real_entity.hpp) 用 `Haunt` 表示一个 ghost 的远端通道：

```cpp
// /workspace/src/server/cellapp/real_entity.hpp
class RealEntity
{
    class Haunt
    {
    public:
        Haunt( CellAppChannel * pChannel, GameTime creationTime );
        CellAppChannel & channel() { return *pChannel_; }
        Mercury::Bundle & bundle() { return pChannel_->bundle(); }
    private:
        CellAppChannel * pChannel_;
        GameTime creationTime_;
    };
    typedef std::vector< Haunt > Haunts;

    Haunts::iterator hauntsBegin();
    Haunts::iterator hauntsEnd();
    int numHaunts() const;
    void addHaunt( CellAppChannel & channel );
    Haunts::iterator delHaunt( Haunts::iterator iter );
};
```

real 把属性变更、位置变更、方法调用等打包成 bundle，发给每个 haunt 所在的 CellAppChannel；邻居 cellapp 收到后更新本地 ghost。ghost 不跑权威逻辑，只接收 real 的广播。

### 10.2 BufferedGhostMessage：乱序重排

问题：当 real 从 cellapp A 迁移（offload）到 cellapp B 时，A 上还没发完的 ghost 消息和 B 上新发的消息可能乱序到达某个邻居。BigWorld 用 [BufferedGhostMessage](file:///workspace/src/server/cellapp/buffered_ghost_message.hpp) 系列处理：

```cpp
// /workspace/src/server/cellapp/buffered_ghost_message.hpp
class BufferedGhostMessage
{
public:
    BufferedGhostMessage( bool isSubsequenceStart, bool isSubsequenceEnd ) :
        isSubsequenceStart_( isSubsequenceStart ),
        isSubsequenceEnd_( isSubsequenceEnd ) {}
    virtual void play() = 0;
    virtual bool isCreateGhostMessage() const { return false; }
    bool isSubsequenceStart() const;
    bool isSubsequenceEnd() const;
};
```

[buffered_ghost_messages.hpp](file:///workspace/src/server/cellapp/buffered_ghost_messages.hpp) 的注释提到消息会被「split into subsequence of messages from each CellApp that the real visits」——即 real 每访问一个 cellapp 产生一个子序列，用 `isSubsequenceStart`/`isSubsequenceEnd` 标记子序列边界，邻居 cellapp 按子序列顺序回放，保证 ghost 状态一致。相关类还有 `BufferedGhostMessageQueue`、`BufferedGhostMessagesForEntity`、`BufferedGhostMessageFactory`（均在 cellapp 目录下）。

### 10.3 cellapp 死亡时的 ghost 清理

cellapp 死亡时，它的 real 全丢，邻居 cellapp 上对应的 ghost 变成「坏 ghost」。清理流程见 [AckCellAppDeathHelper](file:///workspace/src/server/cellapp/ack_cell_app_death_helper.hpp)（第 14 节详述）。

---

## 11. 上行链路：客户端 → baseapp → cellapp

玩家自己的位置/输入更新走**上行链路**：

```
客户端 ──(client msg)──> baseapp(Proxy) ──(cellapp msg)──> cellapp(RealEntity)
```

### 11.1 客户端 → baseapp

客户端的玩家自己 entity（带 `DATA_OWN_CLIENT` 属性）由 [Proxy](file:///workspace/src/server/baseapp/proxy.hpp) 在 baseapp 上代理。`Proxy` 是 `Base` 的子类，持有一条到客户端的连接：

```cpp
// /workspace/src/server/baseapp/proxy.hpp
class Proxy: public Base
{
public:
    bool hasClient() const;
    void onClientDeath( bool shouldExpectClient = true );
    void restoreClient();
    void offload( const Mercury::Address & dstAddr );   // 把 base offload 到别的 baseapp
    void transferClient( const Mercury::Address & dstAddr, bool shouldReset );
    // ...
};
```

客户端发的位置/朝向更新、客户端方法调用，先到 Proxy。Proxy 对玩家的位置更新做限速（`rate_limit_*` 系列、`proxy_rate_limit_callback.hpp`），然后转发给该玩家 entity 在 cellapp 上的 real。

### 11.2 baseapp → cellapp

baseapp 通过 entity 的 cell mailbox 把位置更新发给 cellapp。cellapp 上 `RealEntity::newPosition` 处理：

```cpp
// /workspace/src/server/cellapp/real_entity.hpp
void newPosition( const Vector3 & position );
void sendPhysicsCorrection();   // 检测到客户端作弊/越界时纠正
```

`newPosition` 更新 entity 位置，触发 `EntityRangeListNode::shuffleXThenZ`，进而可能触发邻居的 trigger、更新该玩家自身 Witness 的 AoI。

---

## 12. 下行链路：cellapp → baseapp → client

其它 entity 的状态变化走**下行链路**：

```
cellapp(Witness) ──(bundle)──> baseapp(Proxy) ──(client msg)──> 客户端
```

### 12.1 cellapp → baseapp

Witness 每 tick 把要推送的数据打成 `Mercury::Bundle`，通过 `sendToProxy` 发给 baseapp：

```cpp
// /workspace/src/server/cellapp/witness.hpp
bool sendToClient( int entityMessageType, MemoryOStream & stream );
void sendToProxy( int mercuryMessageType, MemoryOStream & stream );
Mercury::Bundle & bundle();
void sendToClient();
```

注意 `sendToClient` 实际是「让 baseapp 帮我转发给 client」——cellapp 没有到客户端的直连，必须经过 baseapp 的 Proxy。bundle 里包含 `enterAoI`/`leaveAoI`/`createEntity`/属性变更/volatile 更新等。

### 12.2 baseapp → 客户端

Proxy 收到 cellapp 的 bundle，剥离/重组后通过客户端连接发出。客户端侧由 [ServerConnection](file:///workspace/src/lib/connection/server_connection.hpp) 解码：

```cpp
// /workspace/src/lib/connection/server_connection.hpp
void enterAoI( const ClientInterface::enterAoIArgs & args );
void enterAoIOnVehicle( const ClientInterface::enterAoIOnVehicleArgs & args );
void leaveAoI( BinaryIStream & stream );
void createEntity( BinaryIStream & stream );
```

对应的客户端消息定义在 [client_interface.hpp](file:///workspace/src/lib/connection/client_interface.hpp)：

```cpp
// /workspace/src/lib/connection/client_interface.hpp
MF_VARLEN_CLIENT_MSG( createEntity )
MF_BEGIN_CLIENT_MSG( enterAoI )
    MERCURY_ISTREAM( enterAoI, x.id >> x.idAlias )
    MERCURY_OSTREAM( enterAoI, x.id << x.idAlias )
MF_BEGIN_CLIENT_MSG( enterAoIOnVehicle )
    // ...
MF_VARLEN_CLIENT_MSG( leaveAoI )
```

`enterAoI` 只含 `id`（EntityID）和 `idAlias`（1 字节），极省带宽。`createEntity` 是变长消息，带完整属性。`leaveAoI` 也是变长。

### 12.3 客户端的处理

[ServerConnection](file:///workspace/src/lib/connection/server_connection.hpp) 收到 `enterAoI` 后：
1. 记录 idAlias → EntityID 映射。
2. 若脚本需要完整数据，回 `requestEntityUpdate`。
3. 收到 `createEntity` 后，调用客户端的 entity 工厂创建一个 client entity（Python 类实例），加入客户端世界。
4. 后续属性变更/volatile 更新按 idAlias 路由到对应 client entity。
5. `leaveAoI` 时销毁 client entity。

---

## 13. cellappmgr 的负载均衡与 cell 分裂/合并

### 13.1 cellappmgr 的角色

cellappmgr 是 cellapp 集群的管理者。它不跑模拟，只做调度：决定哪个 cellapp 持有哪个 cell 的哪片矩形区域，监控负载，在过载时分裂 cell、在空闲时合并/回收 cell。

cellapp 与 cellappmgr 之间的消息定义在 [cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp)。关键消息：

```cpp
// /workspace/src/server/cellappmgr/cellappmgr_interface.hpp
BW_STREAM_MSG_EX( CellAppMgr, addApp )              // cellapp 注册自己
BW_STREAM_MSG( CellAppMgr, recoverCellApp )         // 恢复死亡 cellapp 的 cell
BW_BEGIN_STRUCT_MSG( CellAppMgr, handleCellAppDeath )  // 通知 cellapp 死亡
BW_BEGIN_STRUCT_MSG( CellApp, informOfLoad )        // cellapp 上报负载
    float load;
    int  numEntities;
BW_STREAM_MSG( CellApp, updateBounds )              // cellapp 上报 cell 边界
BW_BEGIN_STRUCT_MSG( CellApp, retireApp )           // cellapp 请求退休（优雅退出）
```

cellapp 侧通过 [CellAppMgrGateway](file:///workspace/src/server/cellapp/cellappmgr_gateway.hpp) 与 cellappmgr 通信：

```cpp
// /workspace/src/server/cellapp/cellappmgr_gateway.hpp
class CellAppMgrGateway : public ManagerAppGateway
{
public:
    void add( const Mercury::Address & addr, uint16 viewerPort,
            Mercury::ReplyMessageHandler * pReplyHandler );
    void informOfLoad( float load );
    void updateBounds( const Cells & cells, int maxEntityOffload );
    void handleCellAppDeath( const Mercury::Address & addr );
    void ackCellAppDeath( const Mercury::Address & deadAddr );
};
```

### 13.2 负载上报与 cell 分裂

每个 cellapp 周期性地通过 `informOfLoad(load, numEntities)` 把自己的负载（实体数、CPU、bandwidthDeficit 等综合值）报给 cellappmgr，并通过 `updateBounds(cells, maxEntityOffload)` 上报它持有的 cell 边界和可 offload 上限。

cellappmgr 维护全局的 cell 矩形布局。当某个 cellapp 上报的 load 超过阈值时，cellappmgr 会决定**分裂**该 cellapp 上的某个 cell：

1. cellappmgr 选一个过载的 cell，沿 X 或 Z 轴切成两片矩形。
2. cellappmgr 通知原 cellapp：把切线一侧的 real entity offload 到目标 cellapp。
3. 原 cellapp 调 `Cell::offloadEntity` 把 entity 序列化后通过 `CellAppChannel` 发给目标 cellapp。

```cpp
// /workspace/src/server/cellapp/cell.hpp
void offloadEntity( Entity * pEntity, CellAppChannel * pChannel,
        bool shouldSendPhysicsCorrection = false );
```

4. 目标 cellapp 收到后 `addRealEntity`，重建 RealEntity、Witness、Controller，并广播 ghost create 给邻居。
5. 原 cellapp 删除该 real，发 ghost del。

整个过程对客户端透明：玩家的 Proxy（在 baseapp）不变，只是它对应的 real 从一个 cellapp 换到另一个，Witness 状态会通过 offload 数据完整迁移。

### 13.3 cell 合并/回收

负载过低或 cellapp `retireApp`（优雅退出）时，cellappmgr 反向操作：把某 cell 的 entity 全部 offload 到邻居，删除空 cell。`shouldOffload` 消息控制是否允许 offload：

```cpp
// /workspace/src/server/cellappmgr/cellappmgr_interface.hpp
BW_BEGIN_STRUCT_MSG( CellAppMgr, shouldOffload )
    bool enable;
END_STRUCT_MESSAGE()
```

---

## 14. cellapp 死亡时的可见性恢复

当某个 cellapp 进程崩溃，它上面所有 real entity 全丢。恢复流程横跨 cellappmgr / 邻居 cellapp / baseapp：

### 14.1 检测与通知

machined（机器守护进程）检测到 cellapp 进程死亡，通知 cellappmgr。cellappmgr 向所有 cellapp 广播 `handleCellAppDeath{addr}`。

### 14.2 邻居 cellapp 的清理

每个 cellapp 的 `Cells::handleCellAppDeath(addr)` 和 `CellApp::handleCellAppDeath` 被调用（参见 [cellapp.hpp](file:///workspace/src/server/cellapp/cellapp.hpp)、[cells.hpp](file:///workspace/src/server/cellapp/cells.hpp)、[cellapp_death_listener.hpp](file:///workspace/src/server/cellapp/cellapp_death_listener.hpp)）。它们要做两件事：

1. **清理坏 ghost**：本 cellapp 上指向死亡 cellapp 的 ghost 失效，发 `emergencySetCurrentCell` 把这些 entity 的 real 重新定位。
2. **等待 base 恢复 real**：死亡的 real 对应的 base（在 baseapp 上）还活着，baseapp 会重新在某个活着的 cellapp 上创建 real（通过 baseappmgr → cellappmgr 协调，参见文档 20）。

### 14.3 AckCellAppDeathHelper

[AckCellAppDeathHelper](file:///workspace/src/server/cellapp/ack_cell_app_death_helper.hpp) 决定「何时可以向 cellappmgr 确认本 cellapp 已处理完死亡事件」。它的注释说得很清楚：

> We can't do this until:
> - All the reals on this app have informed their bases of their addresses.
> - All emergencySetCurrentCell messages have been delivered and replied to.

```cpp
// /workspace/src/server/cellapp/ack_cell_app_death_helper.hpp
class AckCellAppDeathHelper :
    public TimerHandler,
    public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    AckCellAppDeathHelper( CellApp & app, const Mercury::Address & deadAddr );
    void addCriticalEntity( Entity * pEntity );  // 通道处于 critical 状态的 real
    void addBadGhost();                          // 每发一个 emergencySetCurrentCell 计数 +1
    void startTimer();
private:
    void checkFinished();                        // 轮询：所有 critical 解除 + 所有 badGhost 回复？
    RecentOnloads recentOnloads_;
    int numBadGhosts_;
};
```

它在确认所有「critical real 已通知 base」且「所有 emergencySetCurrentCell 已应答」后，才通过 `CellAppMgrGateway::ackCellAppDeath` 回复 cellappmgr。这保证：所有丢失的 real 都会被 base 重建，且没丢失的 real 不会被重复重建。

### 14.4 base 侧的恢复

死亡的 real 对应的 base 实体在 baseapp 上还活着（baseapp 有备份，见文档 20）。baseapp 通过 `create_cell_entity_handler.hpp` 重新在活着的 cellapp 上创建 real，重建 Witness，恢复 AoI。客户端连接（在 baseapp 的 Proxy 上）不中断，玩家可能短暂「看不到周围」，待 real 重建后 AoI 重新填充。

---

## 15. 完整时序图：玩家移动 → AOI 变化 → 客户端增删实体

下面是一次「玩家 P 向右移动，导致 NPC Q 进入 AoI、NPC R 离开 AoI」的完整跨模块时序：

```
 客户端            baseapp(Proxy)        cellapp(RealEntity.P)      cellapp(Neighbor, ghost of Q/R)
   │                   │                        │                              │
   │ 1. 玩家移动输入    │                        │                              │
   │──playerMove──────>│                        │                              │
   │  (位置/朝向, 限速) │                        │                              │
   │                   │ 2. 转发位置更新          │                              │
   │                   │──setPlayerPos─────────>│                              │
   │                   │                        │ 3. RealEntity::newPosition   │
   │                   │                        │    shuffleXThenZ on RangeList│
   │                   │                        │                              │
   │                   │                        │ 4. P 的 AoITrigger 检测跨越    │
   │                   │                        │    Q 的 EntityRangeListNode  │
   │                   │                        │    越过 P 的 lowerTrigger     │
   │                   │                        │    -> Witness::addToAoI(Q)   │
   │                   │                        │    新建 EntityCache(Q)        │
   │                   │                        │    置 ENTER_PENDING          │
   │                   │                        │                              │
   │                   │                        │ 5. R 离开 AoI(超出 aoiRadius+hyst)│
   │                   │                        │    -> Witness::removeFromAoI(R)│
   │                   │                        │                              │
   │                   │                        │ 6. Witness::update()(每 tick)│
   │                   │                        │    按 priority 选 topN        │
   │                   │                        │    打 bundle:                 │
   │                   │                        │      enterAoI(Q, aliasQ)     │
   │                   │                        │      leaveAoI(R, aliasR)     │
   │                   │                        │      volatileUpdate(Q, ...)  │
   │                   │                        │      propChange(...)         │
   │                   │ 7. sendToProxy(bundle) │                              │
   │                   │<──bundle───────────────│                              │
   │                   │                        │                              │
   │ 8. 转发客户端消息  │                        │                              │
   │<──enterAoI/leaveAoI/propChange/volatile───│                              │
   │                   │                        │                              │
   │ 9. ServerConnection::enterAoI(Q)           │                              │
   │    记录 aliasQ -> Q                         │                              │
   │    (脚本决定是否请求完整数据)                │                              │
   │──requestEntityUpdate(Q)────────────────────>│ (经 baseapp 转发)            │
   │                   │                        │                              │
   │                   │                        │ 10. Witness::requestEntityUpdate│
   │                   │                        │     清 REQUEST_PENDING        │
   │                   │                        │     置 CREATE_PENDING         │
   │                   │                        │     下 tick sendCreate(Q)     │
   │                   │<──createEntity(Q,props)─│                              │
   │<──createEntity(Q)──────────────────────────│                              │
   │                   │                        │                              │
   │ 11. 客户端创建 client entity Q (Python)      │                              │
   │     leaveAoI(R) -> 销毁 client entity R     │                              │
   │     后续 volatileUpdate 按 aliasQ 路由到 Q   │                              │
```

关键点：

- 第 3 步的 `shuffleXThenZ` 是 O(局部) 的，不是 O(N) 扫描。
- 第 4/5 步的进/出 AoI 由 `pAoITrigger_`（RangeTrigger）在 shuffle 时回调，带 `aoiHyst_` 迟滞。
- 第 6 步 topN 选择由 `EntityCache::priority`（距离 + AoIUpdateScheme）排序决定。
- 第 8 步的 `enterAoI` 只发 `id + idAlias`（见 client_interface.hpp），极省带宽。
- 第 9~10 步是惰性握手：客户端按需请求完整数据。
- NPC Q/R 可能是本 cellapp 的 real，也可能是邻居 cellapp 上 real 的 ghost（第 10 节）。若是 ghost，P 的 Witness 看到的是 ghost，ghost 的状态由其 real 通过 haunt 广播过来。

---

## 16. chunk 边界与 cell 边界的关系

BigWorld 的世界在客户端/编辑器视角是按 **chunk**（地形分块）组织的（`src/lib/chunk/`），而在服务端模拟视角是按 **cell**（矩形区域）组织的。两者关系：

- **chunk 是静态地理单元**：一块地形、上面的静态物品、shell（室内场景）等。chunk 有固定边界，按空间网格划分，用于客户端流式加载和渲染。
- **cell 是动态模拟单元**：cellappmgr 可以动态分裂/合并 cell，cell 边界会移动。cell 不与 chunk 边界对齐。
- **服务端用 chunk 做导航/碰撞**：cellapp 上的 `CellChunk`（[cell_chunk.hpp](file:///workspace/src/server/cellapp/cell_chunk.hpp)）把 chunk 概念引入服务端，用于导航网格、碰撞、chunk 加载状态跟踪。`Cell::checkChunkLoading()` 等待 chunk 加载完成才允许实体进入对应区域。
- **RangeList 与 chunk 无关**：AOI 的 RangeList 完全按 entity 的世界坐标 X/Z 排序，不关心 chunk 划分。chunk 只影响「这块地形加载好了没」，不影响「谁在谁 AOI 里」。
- **cell 边界跨越**：entity 从一个 cell 的矩形跨到另一个 cell 时，触发 offload/onload（第 13 节），此时 chunk 加载状态会随 entity 一起迁移（`loading_column.hpp`/`loading_edge.hpp` 跟踪加载中的列/边）。

`src/lib/physics2/` 提供的空间结构（如碰撞体、场景查询）与 RangeList 是平行的两套：physics2 用于**物理碰撞/射线检测**，RangeList 用于**AOI/trigger 的范围查询**。两者不共享数据结构，RangeList 专门为「大量实体 + 频繁小位移 + 跨越回调」优化，避免通用空间索引的开销。

---

## 17. 注意事项与未覆盖项

- **Witness 的 `update()` 完整实现**：`witness.cpp` 源文件在仓库中未提供（仅有 `.hpp`/`.ipp` 与编译产物 `witness.o`），topN 选取、bundle 打包的细节以 `witness.hpp` 的方法签名与 `entity_cache.hpp` 的状态机为准，精确实现参见 `witness.cpp`（若可获取）。
- **AoITrigger**：`witness.hpp` 中前置声明 `class AoITrigger;`，其完整定义未在 cellapp 公开头文件中找到，推断为 `RangeTrigger` 的子类，半径取 `aoiRadius_`/`aoiRadius_+aoiHyst_`，回调 `Witness::addToAoI`/`removeFromAoI`。具体参见 cellapp 内部实现。
- **IDAlias 分配**：`Witness::allocateIDAlias` 与 `freeAliases_[256]` 池的管理细节参见 `witness.cpp`。
- **cellappmgr 的分裂算法**：`cellappmgr_interface.cpp` 仅是接口定义的展开，真正的负载均衡/分裂决策逻辑在 cellappmgr 进程的实现中（仓库中 cellappmgr 目录仅含 interface 文件，主逻辑可能位于其它构建目标或未随源码提供）。本文基于 `informOfLoad`/`updateBounds`/`recoverCellApp` 等消息推断流程。
- **`src/lib/physics2/`**：本目录主要提供客户端/编辑器用的物理与空间结构，与服务端 RangeList 平行，本文不深入其内部结构。
- **Proximity 的「正方形」语义**：文档注释明确 proximity 判定是 square（X/Z 独立），如需圆形需脚本自行二次过滤。
- **VolatileInfo 的优先级含义**：`positionPriority` 等是 entitydef XML 里 `<Flags>` 的一部分，`> 0` 表示该属性 volatile，具体数值用于与 LOD 层交互，精确语义参见 `volatile_info.cpp`。

---

> **相关文档**：cellapp 进程细节见 [10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)；baseapp 与 Proxy 见 [09-服务端进程-baseapp.md](file:///workspace/study-docs/09-服务端进程-baseapp.md)；连接层消息见 [16-客户端框架-连接层-connection.md](file:///workspace/study-docs/16-客户端框架-连接层-connection.md)；entitydef 的属性标志见 [07-实体定义-entitydef.md](file:///workspace/study-docs/07-实体定义-entitydef.md)。
