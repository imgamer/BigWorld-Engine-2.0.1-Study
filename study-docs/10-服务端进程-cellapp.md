# 10. 服务端进程：cellapp（实体模拟与空间分布）

> 本文档基于 BigWorld Engine 2.0.1 源码（`/workspace/src/server/cellapp/`），对 cellapp 进程做深入解析。所有结论都来源于真实的 `.hpp/.cpp` 文件，必要处给出可点击的源码链接；不确定的 API 处明确标注「参见 xxx.hpp」。

---

## 目录

1. [模块定位](#1-模块定位)
2. [进程拓扑与启动流程](#2-进程拓扑与启动流程)
3. [核心类全景](#3-核心类全景)
4. [CellApp：进程级单例](#4-cellapp进程级单例)
5. [Space / Cell / CellInfo：空间与网格](#5-space--cell--cellinfo空间与网格)
6. [Entity：cell 上的游戏对象](#6-entitycell-上的游戏对象)
7. [Real 与 Ghost：实体迁移与镜像](#7-real-与-ghost实体迁移与镜像)
8. [Witness 与 AOI：客户端可见性管理](#8-witness-与-aoi客户端可见性管理)
9. [Range List：交叉触发与范围查询](#9-range-list交叉触发与范围查询)
10. [Controller 系统：可插拔行为组件](#10-controller-系统可插拔行为组件)
11. [导航与移动：EntityNavigate 与 Navigator](#11-导航与移动entitynavigate-与-navigator)
12. [车辆系统：PassengerController / Passengers](#12-车辆系统passengercontroller--passengers)
13. [Ghost 消息缓冲与乱序重排](#13-ghost-消息缓冲与乱序重排)
14. [CellApp 间通信与 CellAppChannel](#14-cellapp-间通信与-cellappchannel)
15. [CellApp 死亡检测与恢复](#15-cellapp-死亡检测与恢复)
16. [负载均衡与流控](#16-负载均衡与流控)
17. [备份（Backup）机制](#17-备份backup机制)
18. [Game Tick 与更新调度](#18-game-tick-与更新调度)
19. [Python 脚本集成](#19-python-脚本集成)
20. [关键消息接口（CellAppInterface）](#20-关键消息接口cellappinterface)
21. [典型调用流（ASCII 时序图）](#21-典型调用流ascii-时序图)
22. [配置项一览](#22-配置项一览)
23. [注意事项与未覆盖项](#23-注意事项与未覆盖项)

---

## 1. 模块定位

cellapp 是 BigWorld 服务端体系中的**实体模拟进程**。它承担两类核心职责：

- **空间模拟**：在三维空间中维护实体（`Entity`）的位置、方向、运动状态，执行物理校验、跨 cell 迁移、AOI（Area of Interest）等模拟计算。
- **客户端可见性推送**：通过 `Witness` 机制为带客户端的实体挑选并推送其 AoI 范围内的实体状态变化，是 server→client 数据流的源头。

cellapp 与 baseapp 一一对应但职责不同：
- baseapp 持有 base 实体（持久代理、客户端连接）；
- cellapp 持有 cell 实体（空间模拟、ghost 镜像、可见性广播）。

一个 cellapp 内部可以同时拥有多个 `Cell`（一个 Space 的一片矩形区域），多个 cellapp 共同协作覆盖整个游戏世界。cellapp 之间通过 `CellAppChannel`（基于 Mercury 网络）互相通信，并由 `CellAppMgr`（cellappmgr 进程）统一调度。

```
                  ┌──────────────────────────────────────────────┐
                  │                  CellAppMgr                  │
                  │   (划分 Cell 矩形，监控负载，触发 Shutdown)    │
                  └──────────────────────┬───────────────────────┘
                                          │ addCell / retireCell / removeCell
        ┌─────────────────────────────────┼──────────────────────────────────┐
        ▼                                 ▼                                  ▼
  ┌───────────┐                     ┌───────────┐                       ┌───────────┐
  │  CellApp  │◀──ghost/offload────▶│  CellApp  │◀──ghost/offload─────▶│  CellApp  │
  │  (本机)   │   CellAppChannel    │  (邻居1)  │   CellAppChannel      │  (邻居2)  │
  └─────┬─────┘                     └─────┬─────┘                       └─────┬─────┘
        │                                 │                                   │
        │ base offload/onload             │ base offload/onload               │ base offload/onload
        ▼                                 ▼                                   ▼
  ┌───────────┐                     ┌───────────┐                       ┌───────────┐
  │  BaseApp  │                     │  BaseApp  │                       │  BaseApp  │
  └───────────┘                     └───────────┘                       └───────────┘
```

源码入口：[main.cpp](file:///workspace/src/server/cellapp/main.cpp)（`bwMainT<CellApp>`），所有逻辑围绕 `CellApp` 单例展开。

---

## 2. 进程拓扑与启动流程

### 2.1 启动入口

`main.cpp` 极其简洁，所有工作交给 `bwMainT<CellApp>`：

```cpp
// /workspace/src/server/cellapp/main.cpp
int BIGWORLD_MAIN( int argc, char * argv[] )
{
    return bwMainT< CellApp >( argc, argv );
}
```

`bwMainT` 是 BigWorld 通用模板（来自 `server/bwservice.hpp`），它负责：
1. 解析命令行；
2. 构造 `Mercury::EventDispatcher` 与 `Mercury::NetworkInterface`；
3. 实例化 `CellApp` 单例；
4. 调用 `CellApp::init(argc, argv)` 完成业务初始化；
5. 进入主事件循环。

### 2.2 与 CellAppMgr 的握手（AddToCellAppMgrHelper）

cellapp 启动后第一件大事是**向 CellAppMgr 注册自己**。这由 [add_to_cellappmgr_helper.hpp](file:///workspace/src/server/cellapp/add_to_cellappmgr_helper.hpp) 中的 `AddToCellAppMgrHelper` 完成：

```cpp
// /workspace/src/server/cellapp/add_to_cellappmgr_helper.hpp
class AddToCellAppMgrHelper : public AddToManagerHelper
{
public:
    AddToCellAppMgrHelper( CellApp & cellApp, uint16 viewerPort ) :
        AddToManagerHelper( cellApp.mainDispatcher() ),
        app_( cellApp ),
        viewerPort_( viewerPort )
    {
        // Auto-send on construction.
        this->send();
    }

    void handleFatalTimeout()
    {
        ERROR_MSG( "AddToCellAppMgrHelper::handleFatalTimeout: Unable to add "
            "CellApp to CellAppMgr. Terminating.\n" );
        app_.mainDispatcher().breakProcessing();
    }

    void doSend()
    {
        app_.cellAppMgr().add( app_.interface().address(), viewerPort_, this );
    }

    bool finishInit( BinaryIStream & data )
    {
        CellAppInitData initData;
        data >> initData;
        return app_.finishInit( initData );
    }
private:
    CellApp & app_;
    uint16 viewerPort_;
};
```

关键点：
- 构造时**自动发送**注册请求（`this->send()`）；
- 若 CellAppMgr 在致命超时内无响应，直接 `breakProcessing()` 终止进程；
- CellAppMgr 的回复中携带 `CellAppInitData`（参见 `server/cell_app_init_data.hpp`），由 `CellApp::finishInit()` 消费。

### 2.3 启动状态机

```cpp
// /workspace/src/server/cellapp/cellapp.hpp （节选）
class CellApp : public EntityApp, public TimerHandler,
    public Singleton< CellApp >
{
public:
    bool finishInit( const CellAppInitData & initData );
    void onGetFirstCell( bool isFromDB );
    bool hasStarted() const { return gameTimer_.isSet(); }
    bool isShuttingDown() const { return shutDownTime_ != 0; }
};
```

启动状态可归纳为：

```
[init] ──▶ [握手 CellAppMgr] ──▶ [finishInit] ──▶ [onGetFirstCell] ──▶ [gameTimer 启动] ──▶ [运行中]
                                      │
                                      └─▶ 失败：breakProcessing()
```

`hasStarted()` 通过 `gameTimer_.isSet()` 判断——只有 game tick 计时器被装上之后，cellapp 才算正式开始模拟。

---

## 3. 核心类全景

cellapp 目录下文件众多，按职责可归为若干层：

| 层次 | 主要类 | 文件 | 职责 |
|------|--------|------|------|
| 进程层 | `CellApp` | [cellapp.hpp](file:///workspace/src/server/cellapp/cellapp.hpp) | 单例入口，调度 tick、握手机制、shutdown |
| 空间层 | `Spaces`, `Space`, `Cell`, `CellInfo`, `CellRangeList` | spaces.hpp, [space.hpp](file:///workspace/src/server/cellapp/space.hpp), [cell.hpp](file:///workspace/src/server/cellapp/cell.hpp), [cell_info.hpp](file:///workspace/src/server/cellapp/cell_info.hpp), [cell_range_list.hpp](file:///workspace/src/server/cellapp/cell_range_list.hpp) | Space 管理、Cell 矩形划分、BSP 树 |
| 实体层 | `Entity`, `RealEntity`, `EntityPopulation`, `EntityType` | [entity.hpp](file:///workspace/src/server/cellapp/entity.hpp), [real_entity.hpp](file:///workspace/src/server/cellapp/real_entity.hpp), [entity_population.hpp](file:///workspace/src/server/cellapp/entity_population.hpp), entity_type.hpp | 实体定义、real 状态、人口管理 |
| Witness/AOI 层 | `Witness`, `EntityCache`, `EntityCacheMap`, `AoIUpdateSchemes` | [witness.hpp](file:///workspace/src/server/cellapp/witness.hpp), [entity_cache.hpp](file:///workspace/src/server/cellapp/entity_cache.hpp), [aoi_update_schemes.hpp](file:///workspace/src/server/cellapp/aoi_update_schemes.hpp) | AoI 列表、优先级、LoD |
| 范围触发层 | `RangeListNode`, `RangeTrigger`, `RangeTriggerNode`, `RangeListTerminator` | [range_list_node.hpp](file:///workspace/src/server/cellapp/range_list_node.hpp), [cell_range_list.hpp](file:///workspace/src/server/cellapp/cell_range_list.hpp) | X/Z 双向链表、trap 触发 |
| 控制器层 | `Controller`, `Controllers`, `MoveController`, `NavigationController`, `VisionController`, `ProximityController`, `VisibilityController`, `TimerController`, `PassengerController`, … | controller.hpp 等多个 | 可插拔行为 |
| 视觉层 | `EntityVision`, `VisibilityController`, `VisionController`, `ScanVisionController` | [entity_vision.hpp](file:///workspace/src/server/cellapp/entity_vision.hpp), visibility_controller.hpp, vision_controller.hpp, scan_vision_controller.hpp | 视野范围、可见性 |
| Ghost 缓冲层 | `BufferedGhostMessage`, `BufferedGhostMessages`, `BufferedGhostMessagesForEntity`, `BufferedEntityMessages` | buffered_ghost_message.hpp, [buffered_ghost_messages.hpp](file:///workspace/src/server/cellapp/buffered_ghost_messages.hpp), [buffered_entity_messages.hpp](file:///workspace/src/server/cellapp/buffered_entity_messages.hpp) | 跨进程消息乱序重排 |
| 通信层 | `CellAppChannel`, `CellAppChannels`, `CellAppMgrGateway`, `CellAppDeathListeners`, `CellAppDeathListener` | [cell_app_channel.hpp](file:///workspace/src/server/cellapp/cell_app_channel.hpp), [cell_app_channels.hpp](file:///workspace/src/server/cellapp/cell_app_channels.hpp), [cellappmgr_gateway.hpp](file:///workspace/src/server/cellapp/cellappmgr_gateway.hpp), cellapp_death_listener.hpp | 跨进程 channel、死亡通知 |
| 调度层 | `Updatable`, `Updatables`, `EmergencyThrottle`, `ThrottleConfig` | [updatable.hpp](file:///workspace/src/server/cellapp/updatable.hpp), [updatables.hpp](file:///workspace/src/server/cellapp/updatables.hpp), [emergency_throttle.hpp](file:///workspace/src/server/cellapp/emergency_throttle.hpp), throttle_config.hpp | 每 tick 更新、应急限流 |
| Mailbox | `ServerEntityMailBox`, `CellEntityMailBox`, `BaseEntityMailBox`, `CommonBaseEntityMailBox` | [mailbox.hpp](file:///workspace/src/server/cellapp/mailbox.hpp) | 远程实体引用 |
| 恢复层 | `AckCellAppDeathHelper` | [ack_cell_app_death_helper.hpp](file:///workspace/src/server/cellapp/ack_cell_app_death_helper.hpp) | CellApp 死亡后恢复确认 |
| 历史事件 | `HistoryEvent`, `EventHistory` | [history_event.hpp](file:///workspace/src/server/cellapp/history_event.hpp) | 事件去重与回放 |
| 配置 | `CellAppConfig`, `ThrottleConfig`, `IdConfig`, `NoiseConfig` | cellapp_config.hpp, throttle_config.hpp, id_config.hpp, noise_config.hpp | ServerAppOption 配置 |

---

## 4. CellApp：进程级单例

`CellApp` 是整个进程的中枢。声明在 [cellapp.hpp](file:///workspace/src/server/cellapp/cellapp.hpp)：

```cpp
// /workspace/src/server/cellapp/cellapp.hpp
class CellApp : public EntityApp, public TimerHandler,
    public Singleton< CellApp >
{
public:
    typedef CellAppConfig Config;

private:
    enum TimeOutType
    {
        TIMEOUT_GAME_TICK,
        TIMEOUT_TRIM_HISTORIES,
        TIMEOUT_LOADING_TICK
    };

public:
    ENTITY_APP_HEADER( CellApp, cellApp )

    CellApp( Mercury::EventDispatcher & mainDispatcher,
            Mercury::NetworkInterface & interface );
    virtual ~CellApp();

    bool finishInit( const CellAppInitData & initData );
    void onGetFirstCell( bool isFromDB );

    // ---- Message handlers ----
    void addCell( const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void startup( const CellAppInterface::startupArgs & args );
    void setGameTime( const CellAppInterface::setGameTimeArgs & args );
    void handleCellAppMgrBirth(
        const CellAppInterface::handleCellAppMgrBirthArgs & args );
    void handleCellAppDeath(
        const CellAppInterface::handleCellAppDeathArgs & args );
    void handleBaseAppDeath( BinaryIStream & data );
    virtual void shutDown();
    void shutDown( const CellAppInterface::shutDownArgs & args );
    void controlledShutDown(
        const CellAppInterface::controlledShutDownArgs & args );
    void setSharedData( BinaryIStream & data );
    void delSharedData( BinaryIStream & data );
    void setBaseApp( const CellAppInterface::setBaseAppArgs & args );
    void onloadTeleportedEntity( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void cellAppMgrInfo( const CellAppInterface::cellAppMgrInfoArgs & args );

    // ---- Utility methods ----
    Entity * findEntity( EntityID id ) const;
    Cell *  findCell( SpaceID id ) const;
    Space * findSpace( SpaceID id ) const;

    static Mercury::Channel & getChannel( const Mercury::Address & addr )
    {
        return CellApp::instance().interface_.findOrCreateChannel( addr );
    }

    CellAppMgrGateway & cellAppMgr()        { return cellAppMgr_; }
    DBMgr & dbMgr()                         { return *dbMgr_.pChannelOwner(); }
    Cells & cells()                         { return cells_; }
    const Cells & cells() const             { return cells_; }
    float getLoad() const                   { return persistentLoad_; }
    uint64 lastGameTickTime() const         { return lastGameTickTime_; }
    float maxCellAppLoad() const            { return maxCellAppLoad_; }
    float emergencyThrottle() const         { return throttle_.value(); }
    IDClient & idClient()                   { return idClient_; }

    // ---- Update methods ----
    bool registerForUpdate( Updatable * pObject, int level = 0 );
    bool deregisterForUpdate( Updatable * pObject );
    bool nextTickPending() const; // are we running out of time?

    // ---- Misc ----
    void destroyCell( Cell * pCell );
    void detectDeadCellApps( const std::vector< Mercury::Address > & addrs );

    BufferedEntityMessage & bufferedEntityMessages()  { return *pBufferedEntityMessages_; }
    BufferedGhostMessages & bufferedGhostMessages()  { return *pBufferedGhostMessages_; }

    bool shouldOffload() const { return shouldOffload_; }
    void shouldOffload( bool b ) { shouldOffload_ = b; }
    int id() const              { return id_; }
    virtual void onSignalled( int sigNum );

private:
    virtual ManagerAppGateway & managerAppGateway() { return cellAppMgr_; }
    virtual bool init( int argc, char *argv[] );

    void initExtensions();
    bool initScript();
    void addWatchers();
    void checkSendWindowOverflows();
    void checkPython();
    int  secondsToTicks( float seconds, int lowerBound );
    void startGameTime();
    void sendShutdownAck( ShutDownStage stage );
    bool inShutDownPause() const
    { return (shutDownTime_ != 0) && (time_ == shutDownTime_); }

    void handleTimeout( TimerHandle handle, void * arg );
    void handleGameTickTimeSlice();
    void handleTrimHistoriesTimeSlice();
    void tickShutdown();

    double calcTickPeriod();
    double calcTransientLoadTime();
    double calcSpareTime();
    double calcThrottledLoadTime();
    void checkTickWarnings( double persistentLoadTime, double tickTime,
           double spareTime );
    void addToLoad( float timeSpent, float & result ) const;
    void updateLoad();
    void updateBoundary();
    void callTimers();
    void tickBackup();
    void checkOffloads();
    void syncTime();
    float numSecondsBehind() const;

    // ---- Data ----
    Cells               cells_;
    Spaces *            pSpaces_;
    IDClient            idClient_;          // 必须先于 dbMgr_ 构造
    CellAppMgrGateway   cellAppMgr_;
    AnonymousChannelClient dbMgr_;
    GameTime            shutDownTime_;
    TimeKeeper *        pTimeKeeper_;
    Pickler *           pPickler_;
    Updatables          updatables_;
    EmergencyThrottle   throttle_;          // 紧急限流器
    uint64              lastGameTickTime_;
    timeval             oldTimeval_;
    PythonServer *      pPythonServer_;
    SharedData *        pCellAppData_;
    SharedData *        pGlobalData_;
    Mercury::Address    baseAppAddr_;
    int                 backupIndex_;
    TimerHandle         gameTimer_;
    TimerHandle         loadingTimer_;
    TimerHandle         trimHistoryTimer_;
    uint64              reservedTickTime_;
    CellViewerServer *  pViewerServer_;
    CellAppID           id_;
    float               persistentLoad_;
    float               transientLoad_;
    float               totalLoad_;
    float               maxCellAppLoad_;
    bool                shouldOffload_;
    bool                hasAckedCellAppMgrShutDown_;
    BufferedEntityMessages * pBufferedEntityMessages_;
    BufferedGhostMessages *  pBufferedGhostMessages_;
    CellAppChannels *   pCellAppChannels_;

    friend class CellAppResourceReloader;
};
```

### 4.1 关键设计点

1. **多重身份**：继承 `EntityApp`（提供 entity app 通用框架）、`TimerHandler`（响应计时器）、`Singleton<CellApp>`（全局唯一访问点）。
2. **三类计时器**：
   - `TIMEOUT_GAME_TICK`：主 tick，驱动所有 `Updatable` 与 game time；
   - `TIMEOUT_TRIM_HISTORIES`：定期修剪实体的 `EventHistory`；
   - `TIMEOUT_LOADING_TICK`：驱动 chunk 异步加载。
3. **负载分两类**：
   - `persistentLoad_`：长期负载，定期上报给 CellAppMgr 用于均衡决策；
   - `transientLoad_`：瞬态负载，用于本地 throttle 决策。
4. **idClient 的位置约束**：源码注释明确说 `idClient_` 必须先于 `dbMgr_` 构造，因为 `dbMgr_` 析构时会取消挂起请求并回调 `idClient_`，反过来则会引用已析构对象。
5. **三种进程间通信出口**：
   - `cellAppMgr_`：与 CellAppMgr 的 gateway；
   - `dbMgr_`：匿名 channel，与 DBMgr 通信（例如 `writeToDBRequest`）；
   - `baseAppAddr_`：本进程对应的 base app 地址（通过 `setBaseApp` 设置）。

### 4.2 主要消息处理

| 消息 | 处理函数 | 来源 | 用途 |
|------|----------|------|------|
| `addCell` | `CellApp::addCell` | CellAppMgr | 给本进程分配一个新 Cell |
| `startup` | `CellApp::startup` | CellAppMgr | 通知 BaseApp 地址，进入运行态 |
| `setGameTime` | `CellApp::setGameTime` | CellAppMgr | 同步全局 GameTime |
| `handleCellAppMgrBirth` | `CellApp::handleCellAppMgrBirth` | CellAppMgr | CellAppMgr 重启通知 |
| `handleCellAppDeath` | `CellApp::handleCellAppDeath` | CellAppMgr | 邻居 cellapp 死亡通知 |
| `handleBaseAppDeath` | `CellApp::handleBaseAppDeath` | BaseAppMgr | 对应 baseapp 死亡 |
| `shutDown` | `CellApp::shutDown` | CellAppMgr | 立即停机 |
| `controlledShutDown` | `CellApp::controlledShutDown` | CellAppMgr | 受控分阶段停机 |
| `setSharedData`/`delSharedData` | `CellApp::setSharedData`/`delSharedData` | CellAppMgr | 同步 cellAppData / globalData |
| `setBaseApp` | `CellApp::setBaseApp` | CellAppMgr | 设置本进程对应 baseapp |
| `onloadTeleportedEntity` | `CellApp::onloadTeleportedEntity` | 其他 cellapp | teleport 后接收端 onload |
| `cellAppMgrInfo` | `CellApp::cellAppMgrInfo` | CellAppMgr | 推送 `maxCellAppLoad` 阈值 |

---

## 5. Space / Cell / CellInfo：空间与网格

### 5.1 Spaces：进程内 Space 集合

[spaces.hpp](file:///workspace/src/server/cellapp/spaces.hpp)：

```cpp
// /workspace/src/server/cellapp/spaces.hpp
class Spaces
{
public:
    ~Spaces();
    Space * find( SpaceID id ) const;
    Space * create( SpaceID id );
    void prepareNewlyLoadedChunksForDelete();
    void tickChunks();
    void deleteOldSpaces();
    void writeRecoveryData( BinaryOStream & stream );
    size_t size() const { return container_.size(); }
    WatcherPtr pWatcher();
private:
    typedef std::map< SpaceID, Space * > Container;
    Container container_;
};
```

`Spaces` 是一个 `SpaceID → Space*` 的简单映射，负责创建、查找、tick chunk 加载、写出恢复数据。

### 5.2 Space：一个游戏世界

[space.hpp](file:///workspace/src/server/cellapp/space.hpp)：

```cpp
// /workspace/src/server/cellapp/space.hpp
class Space : public TimerHandler
{
public:
    Space( SpaceID id );
    virtual ~Space();

    void reuse();

    const CellInfo * pCellAt( float x, float z ) const;
    void visitRect( const BW::Rect & rect, CellInfoVisitor & visitRect );

    SpaceID id() const             { return id_; }
    Cell * pCell() const           { return pCell_; }
    void   pCell( Cell * pCell );
    ChunkSpacePtr pChunkSpace() const;
    Vector3 cellCentre() const;

    // ---- Entity ----
    void createGhost( const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void createGhost( const EntityID entityID, BinaryIStream & data );
    void addEntity( Entity * pEntity );
    void removeEntity( Entity * pEntity );
    EntityPtr newEntity( EntityID id, EntityTypeID entityTypeID );
    Entity * findNearestEntity( const Vector3 & position );

    enum UpdateCellAppMgr { UPDATE_CELL_APP_MGR, DONT_UPDATE_CELL_APP_MGR };
    enum DataEffected     { ALREADY_EFFECTED, NEED_TO_EFFECT };

    // ---- Space data ----
    void spaceData( BinaryIStream & data );
    void allSpaceData( BinaryIStream & data );
    void updateGeometry( BinaryIStream & data );
    void spaceGeometryLoaded( BinaryIStream & data );
    void shutDownSpace( BinaryIStream & data );
    void requestShutDown();

    CellInfo * findCell( const Mercury::Address & addr ) const;
    SpaceNode * readTree( BinaryIStream & stream, const BW::Rect & rect );

    bool spaceDataEntry( const SpaceEntryID & entryID, uint16 key,
        const std::string & value,
        UpdateCellAppMgr cellAppMgrAction = UPDATE_CELL_APP_MGR,
        DataEffected effected = NEED_TO_EFFECT );

    int32 begDataSeq() const { return begDataSeq_; }
    int32 endDataSeq() const { return endDataSeq_; }
    const std::string * dataBySeq( int32 seq,
        SpaceEntryID & entryID, uint16 & key ) const;
    int dataRecencyLevel( int32 seq ) const;

    const RangeList & rangeList() const { return rangeList_; }
    bool getRealEntitiesBoundary( BW::Rect & boundary,
           int numToSkip = 0 ) const;

    SpaceEntities & spaceEntities() { return entities_; }
    const SpaceEntities & spaceEntities() const { return entities_; }

    void writeDataToStream( BinaryOStream & steam ) const;
    void readDataFromStream( BinaryIStream & stream );

    void chunkTick();
    void calcLoadedRect( BW::Rect & loadedRect ) const;
    void prepareNewlyLoadedChunksForDelete();
    bool isFullyUnloaded() const;
    float timeOfDay() const;
    bool isShuttingDown() const { return shuttingDownTimerHandle_.isSet(); }
    void writeRecoveryData( BinaryOStream & stream ) const;
    void setPendingCellDelete( const Mercury::Address & addr );
    size_t numEntities() const { return entities_.size(); }

private:
    typedef std::map< Mercury::Address, SmartPointer< CellInfo > > CellInfos;

    SpaceID            id_;
    Cell *             pCell_;
    ChunkSpacePtr      pChunkSpace_;
    SpaceEntities      entities_;
    CellInfos          cellInfos_;
    RangeList          rangeList_;
    int32              begDataSeq_;
    int32              endDataSeq_;

    struct RecentDataEntry
    {
        SpaceEntryID   entryID;
        GameTime       time;
        uint16         key;
    };
    typedef std::vector<RecentDataEntry> RecentDataEntries;
    RecentDataEntries  recentData_;
    ServerGeometryMappings geometryMappings_;
    std::list< Chunk * > loadingChunks_;
    float              initialTimeOfDay_;
    float              gameSecondsPerSecond_;
    std::string        lastMappedGeometry_;
    SpaceNode *        pCellInfoTree_;
    TimerHandle        shuttingDownTimerHandle_;

public:
    static uint32 s_allSpacesDataChangeSeq_;
};
```

要点：
- `Space` 持有 1 个 `Cell *`（**本进程拥有的那个 Cell**，注意不是 Space 的所有 Cell）；
- `cellInfos_` 是 BSP 树叶子节点集合，记录整个 Space 中所有 Cell 的归属 cellapp 地址；
- `rangeList_` 是 X/Z 双向链表的根，用于实体范围查询与触发器；
- `entities_` 是本 Space 内所有实体的 `vector`；
- `geometryMappings_` 管理地图块（chunk）的加载/卸载；
- `begDataSeq_/endDataSeq_` 与 `recentData_` 共同实现 SpaceData（如天气、地形路径等）的版本化与去重；
- `s_allSpacesDataChangeSeq_` 是全局静态序号，跨 Space 的全局数据变更时递增。

### 5.3 Cell：一片矩形区域

[cell.hpp](file:///workspace/src/server/cellapp/cell.hpp)：

```cpp
// /workspace/src/server/cellapp/cell.hpp
class Cell
{
public:
    class Entities
    {
    public:
        typedef std::vector< EntityPtr > Collection;
        typedef Collection::iterator iterator;
        bool add( Entity * pEntity );
        bool remove( Entity * pEntity );
        EntityPtr front()                { return collection_.front(); }
        // ... 其余 STL-like 接口
    private:
        void swapWithBack( Entity * pEntity );
        Collection collection_;
    };

    Cell( Space & space, const CellInfo & cellInfo );
    ~Cell();

    void shutDown();

    const CellInfo & cellInfo() { return *pCellInfo_; }

    // Entity maintenance
    void offloadEntity( Entity * pEntity, CellAppChannel * pChannel,
            bool shouldSendPhysicsCorrection = false );
    void addRealEntity( Entity * pEntity, bool shouldSendNow );
    void entityDestroyed( Entity * pEntity );

    EntityPtr createEntityInternal( BinaryIStream & data, PyObject * pDict,
        bool isRestore = false,
        Mercury::ChannelVersion channelVersion = Mercury::SEQ_NULL,
        EntityPtr pNearbyEntity = NULL );

    void backup( int index, int period );
    bool checkOffloadsAndGhosts();
    void checkChunkLoading();
    void onSpaceGone();
    void debugDump();

    // Entity creation message handlers
    void createEntity( const Mercury::Address& srcAddr,
        const Mercury::UnpackedMessageHeader& header,
        BinaryIStream & data,
        EntityPtr pNearbyEntity );
    void createEntity( const Mercury::Address& srcAddr,
        const Mercury::UnpackedMessageHeader& header,
        BinaryIStream & data )
    { this->createEntity( srcAddr, header, data, NULL ); }

    SpaceID spaceID() const;
    Space & space()                       { return space_; }
    const Space & space() const           { return space_; }
    const BW::Rect & rect() const         { return pCellInfo_->rect(); }
    int numRealEntities() const;
    void sendEntityPositions( Mercury::Bundle & bundle ) const;
    Entities & realEntities();

    // Load balancing
    bool shouldOffload() const;
    void shouldOffload( bool shouldOffload );
    void shouldOffload( BinaryIStream & data );

    void retireCell( BinaryIStream & data );
    void removeCell( BinaryIStream & data );
    void notifyOfCellRemoval( BinaryIStream & data );
    void ackCellRemoval( const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );

    bool reuse();
    void handleCellAppDeath( const Mercury::Address & addr );

    void restoreEntity( const Mercury::Address& srcAddr,
        const Mercury::UnpackedMessageHeader& header,
        BinaryIStream & data );

    bool isRemoved() const { return isRemoved_; }

private:
    Entities realEntities_;
    bool shouldOffload_;
    mutable float lastERTFactor_;
    mutable uint64 lastERTCalcTime_;
    friend class CellViewerConnection;

    float initialTimeOfDay_;
    float gameSecondsPerSecond_;
    bool isRetiring_;
    bool isRemoved_;
    Space & space_;
    int backupIndex_;
    ConstCellInfoPtr pCellInfo_;

    typedef std::multiset< Mercury::Address > RemovalAcks;
    RemovalAcks pendingAcks_;
    RemovalAcks receivedAcks_;
};
```

要点：
- `Cell::Entities` 用 `vector` 存储本 Cell 的 real 实体，`remove` 通过 `swapWithBack` 实现 O(1) 删除（保留 `EntityRemovalHandle`）；
- `offloadEntity` 是把 real 实体迁出到邻居 cellapp 的核心入口；
- `backup(index, period)` 配合 `CellApp::tickBackup` 做循环备份；
- `shouldOffload_` 由 CellAppMgr 控制，决定该 Cell 是否还参与负载均衡；
- `retireCell/removeCell/notifyOfCellRemoval/ackCellRemoval` 是 Cell 退役四阶段；
- `pendingAcks_/receivedAcks_` 跟踪所有邻居对 cell removal 的 ACK。

### 5.4 CellInfo：BSP 叶子

[cell_info.hpp](file:///workspace/src/server/cellapp/cell_info.hpp)：

```cpp
// /workspace/src/server/cellapp/cell_info.hpp
class CellInfo : public SpaceNode, public ReferenceCount
{
public:
    CellInfo( SpaceID spaceID, const BW::Rect & rect,
            const Mercury::Address & addr, BinaryIStream & stream );
    ~CellInfo();

    static WatcherPtr pWatcher();
    void updateFromStream( BinaryIStream & stream );

    virtual void deleteTree() {};
    virtual const CellInfo * pCellAt( float x, float z ) const;
    virtual void visitRect( const BW::Rect & rect, CellInfoVisitor & visitor );
    virtual void addToStream( BinaryOStream & stream ) const;

    const Mercury::Address & addr() const  { return addr_; }
    float getLoad() const                  { return load_; }

    bool shouldDelete() const              { return shouldDelete_; }
    void shouldDelete( bool v )            { shouldDelete_ = v; }
    const BW::Rect & rect() const          { return rect_; }
    void rect( const BW::Rect & rect )     { rect_ = rect; }

    bool contains( const Vector3 & pos ) const
    { return rect_.contains( pos.x, pos.z ); }

    void setPendingDelete()                { isDeletePending_ = true; }
    bool isDeletePending() const           { return isDeletePending_; }
    bool hasBeenCreated() const            { return hasBeenCreated_; }

private:
    SpaceID          spaceID_;
    Mercury::Address addr_;
    float            load_;
    BW::Rect         rect_;
    bool             shouldDelete_;
    bool             isDeletePending_;
    bool             hasBeenCreated_;
};

typedef SmartPointer< CellInfo > CellInfoPtr;
typedef ConstSmartPointer< CellInfo > ConstCellInfoPtr;
```

`CellInfo` 是 BSP 树（`SpaceNode`）的叶子节点，描述「Space 中的一片矩形属于哪个 cellapp」。`pCellAt(x, z)` 是 BSP 查询：给定一个坐标，找到它对应的 CellInfo。

---

## 6. Entity：cell 上的游戏对象

### 6.1 类骨架

[entity.hpp](file:///workspace/src/server/cellapp/entity.hpp) 是 cellapp 中最大的文件之一。核心声明：

```cpp
// /workspace/src/server/cellapp/entity.hpp
class Entity : public PyInstancePlus
{
    Py_InstanceHeader( Entity )

public:
    static const EntityPopulation & population() { return population_; }
    static void addWatchers();

    static bool isValidPosition( const Position3D &c )
    {
        const float MAX_ENTITY_POS = 1000000000.f;
        return (-MAX_ENTITY_POS < c.x && c.x < MAX_ENTITY_POS &&
            -MAX_ENTITY_POS < c.y && c.y < MAX_ENTITY_POS &&
            -MAX_ENTITY_POS < c.z && c.z < MAX_ENTITY_POS);
    }

    Entity( EntityTypePtr pEntityType );
    void setToInitialState( EntityID id, Space * pSpace );
    ~Entity();

    bool initReal( BinaryIStream & data, PyObject * pDict,
        bool isRestore,
        Mercury::ChannelVersion channelVersion,
        EntityPtr pNearbyEntity );

    void initGhost( BinaryIStream & data );

    void offload( CellAppChannel * pChannel, bool isTeleport );
    void onload( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data );
    void createGhost( Mercury::Bundle & bundle );

    void callback( const char * methodName );

    // ---- Accessors ----
    EntityID id() const;
    void setShouldReturnID( bool shouldReturnID );
    const Position3D & position() const;
    const Direction3D & direction() const;
    const VolatileInfo & volatileInfo() const;

    bool isReal() const;
    bool isRealToScript() const;
    RealEntity * pReal() const;

    const Mercury::Address & realAddr() const;
    const Mercury::Address & nextRealAddr() const { return nextRealAddr_; }
    CellAppChannel * pRealChannel() { return pRealChannel_; }

    Space & space();
    const Space & space() const;
    Cell & cell();
    const Cell & cell() const;

    EventHistory & eventHistory();
    const EventHistory & eventHistory() const;

    bool isDestroyed() const;
    bool inDestroy() const        { return inDestroy_; }
    void destroy();

    EntityTypeID entityTypeID() const;
    EntityTypeID clientTypeID() const;

    VolatileNumber volatileUpdateNumber() const { return volatileUpdateNumber_; }

    float topSpeed() const        { return topSpeed_; }
    float topSpeedY() const       { return topSpeedY_; }
    uint8 physicsCorrections() const { return physicsCorrections_; }

    EntityRangeListNode * pRangeListNode() const;
    ChunkSpace * pChunkSpace() const;
    AoIUpdateSchemeID aoiUpdateSchemeID() const { return aoiUpdateSchemeID_; }

    void incRef() const;
    void decRef() const;

    HistoryEvent * addHistoryEventLocally( uint8 type,
        MemoryOStream & stream, HistoryEvent::Level level,
        EntityMemberStats * pChangedDescription,
        const std::string * pName = NULL );

    void writeClientUpdateDataToBundle( Mercury::Bundle & bundle,
            const Vector3 & basePos,
            EntityCache & cache,
            float lodPriority ) const;

    void writeVehicleChangeToBundle( Mercury::Bundle & bundle,
        EntityCache & cache ) const;

    static void forwardMessageToReal( CellAppChannel & realChannel,
        EntityID entityID, uint8 messageID,
        BinaryIStream & data,
        const Mercury::Address & srcAddr, Mercury::ReplyID replyID );

    bool sendMessageToReal( const MethodDescription * pDescription,
            PyObject * args );

    const Mercury::Address & addrForMessagesFromReal() const;
    void trimEventHistory( GameTime cleanUpTime );
    void setPositionAndDirection( const Position3D & position,
        const Direction3D & direction );

    int numHaunts() const;
    INLINE EntityTypePtr pType() const;
    void reloadScript();
    bool migrate();
    void migratedAll();

    // ---- Message handlers (节选) ----
    void avatarUpdateImplicit( const CellAppInterface::avatarUpdateImplicitArgs & args );
    void avatarUpdateExplicit( const CellAppInterface::avatarUpdateExplicitArgs & args );
    void ackPhysicsCorrection( const CellAppInterface::ackPhysicsCorrectionArgs & args );
    void ghostAvatarUpdate( const CellAppInterface::ghostAvatarUpdateArgs & args );
    void ghostHistoryEvent( BinaryIStream & data );
    void ghostedDataUpdate( BinaryIStream & data );
    void ghostSetReal( const CellAppInterface::ghostSetRealArgs & args );
    void ghostSetNextReal( const CellAppInterface::ghostSetNextRealArgs & args );
    void delGhost( const CellAppInterface::delGhostArgs & args );
    void ghostVolatileInfo( const CellAppInterface::ghostVolatileInfoArgs & args );
    void ghostControllerCreate( BinaryIStream & data );
    void ghostControllerDelete( BinaryIStream & data );
    void ghostControllerUpdate( BinaryIStream & data );
    void witnessed( const CellAppInterface::witnessedArgs & args );
    void checkGhostWitnessed( const CellAppInterface::checkGhostWitnessedArgs & args );
    void aoiUpdateSchemeChange( const CellAppInterface::aoiUpdateSchemeChangeArgs & args );
    void delControlledBy( const CellAppInterface::delControlledByArgs & args );
    void forwardedBaseEntityPacket( BinaryIStream & data );
    void onBaseOffloaded( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void teleport( const CellAppInterface::teleportArgs & args );
    void onTeleportSuccess( Entity * pNearbyEntity );
    void enableWitness( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void witnessCapacity( const CellAppInterface::witnessCapacityArgs & args );
    void requestEntityUpdate( BinaryIStream & data );
    void writeToDBRequest( const Mercury::Address & srcAddr,
            Mercury::UnpackedMessageHeader & header, BinaryIStream & stream );
    void destroyEntity( const CellAppInterface::destroyEntityArgs & args );
    void runScriptMethod( BinaryIStream & data );
    void callBaseMethod( BinaryIStream & data );
    void callClientMethod( BinaryIStream & data );
    void runExposedMethod( int type, BinaryIStream & data );

    // ---- Script methods (节选) ----
    PY_METHOD_DECLARE( py_destroy )
    PY_METHOD_DECLARE( py_cancel )
    PY_METHOD_DECLARE( py_isReal )
    PY_METHOD_DECLARE( py_isRealToScript )
    PY_METHOD_DECLARE( py_clientEntity )
    PY_METHOD_DECLARE( py_debug )
    PY_PICKLING_METHOD_DECLARE( MailBox )

    PY_AUTO_METHOD_DECLARE( RETOK, destroySpace, END );
    bool destroySpace();
    PY_AUTO_METHOD_DECLARE( RETOK, writeToDB, END );
    bool writeToDB();
    PY_AUTO_METHOD_DECLARE( RETOWN, entitiesInRange, ARG( float,
                                OPTARG( PyObjectPtr, NULL ,
                                OPTARG( PyObjectPtr, NULL, END ) ) ) )
    PyObject * entitiesInRange( float range, PyObjectPtr pClass = NULL,
            PyObjectPtr pActualPos = NULL );

    bool outdoorPropagateNoise( float range, int event, int info );
    PY_AUTO_METHOD_DECLARE( RETOK, makeNoise,
            ARG( float, ARG( int, OPTARG( int, 0, END ) ) ) )
    bool makeNoise( float noiseLevel, int event, int info=0);

    PY_AUTO_METHOD_DECLARE( RETOWN, getGroundPosition, END )
    PyObject * getGroundPosition( ) const;

    PY_RO_ATTRIBUTE_DECLARE( periodsWithoutWitness_, periodsWithoutWitness )
    PY_RO_ATTRIBUTE_DECLARE( pType()->name(), className );
    PY_RO_ATTRIBUTE_DECLARE( id_, id )
    PY_RO_ATTRIBUTE_DECLARE( isDestroyed_, isDestroyed )

    PyObject * pyGet_spaceID();
    PY_RO_ATTRIBUTE_SET( spaceID )
    SpaceID spaceID() const;

    PyObject * pyGet_position();
    int pySet_position( PyObject * value );

    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( Vector3, directionPy, direction )
    const Vector3 & directionPy() const;
    void directionPy( const Vector3 & newDir );

    PY_RO_ATTRIBUTE_DECLARE( globalDirection_.yaw, yaw )
    PY_RO_ATTRIBUTE_DECLARE( globalDirection_.pitch, pitch )
    PY_RO_ATTRIBUTE_DECLARE( globalDirection_.roll, roll )

    PY_RO_ATTRIBUTE_DECLARE( (Vector3 &)localPosition_, localPosition );
    PY_RO_ATTRIBUTE_DECLARE( localDirection_.yaw, localYaw );
    PY_RO_ATTRIBUTE_DECLARE( localDirection_.pitch, localPitch );
    PY_RO_ATTRIBUTE_DECLARE( localDirection_.roll, localRoll );

    PY_RO_ATTRIBUTE_DECLARE( pVehicle_, vehicle );

    bool isOutdoors() const;
    bool isIndoors() const;
    PY_RO_ATTRIBUTE_DECLARE( isOutdoors(), isOutdoors )
    PY_RO_ATTRIBUTE_DECLARE( isIndoors(), isIndoors )

    PY_READABLE_ATTRIBUTE_GET( volatileInfo_, volatileInfo )
    int pySet_volatileInfo( PyObject * value );

    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, isOnGround, isOnGround )

    PyObject * pyGet_velocity();
    PY_RO_ATTRIBUTE_SET( velocity )

    PY_RW_ATTRIBUTE_DECLARE( topSpeed_, topSpeed )
    PY_RW_ATTRIBUTE_DECLARE( topSpeedY_, topSpeedY )

    PyObject * pyGet_aoiUpdateScheme();
    int pySet_aoiUpdateScheme( PyObject * value );

    PyObject * trackEntity( int entityId, float velocity = 2*MATH_PI,
               int period = 10, int userArg = 0 );
    PY_AUTO_METHOD_DECLARE( RETOWN, trackEntity,
        ARG( int, OPTARG( float, 2*MATH_PI,
                OPTARG( int, 5, OPTARG( int, 0,  END ) ) ) ) )

    bool setPortalState( bool isOpen, WorldTriangle::Flags collisionFlags );
    PY_AUTO_METHOD_DECLARE( RETOK, setPortalState,
            ARG( bool, OPTARG( WorldTriangle::Flags, 0, END ) ) )

    PyObject * getDict();
    PY_AUTO_METHOD_DECLARE( RETOWN, getDict, END )

    bool sendToClient( const MethodDescription & description,
            MemoryOStream & argStream, bool isForOwn = true, bool isForOthers = false );
    bool sendToClientViaReal( const MethodDescription & description,
            MemoryOStream & argStream, bool isForOwn = true, bool isForOthers = false );

    SpaceRemovalHandle removalHandle() const        { return removalHandle_; }
    void removalHandle( SpaceRemovalHandle handle ) { removalHandle_ = handle; }

    bool isInAoIOffload() const;
    void isInAoIOffload( bool isInAoIOffload );

    INLINE bool isOnGround() const;
    void isOnGround( bool isOnGround );

    static WatcherPtr pWatcher();
    static const Vector3 INVALID_POSITION;

    const Position3D & localPosition() const        { return localPosition_; }
    const Direction3D & localDirection() const      { return localDirection_; }

    void setLocalPositionAndDirection( const Position3D & localPosition,
            const Direction3D & localDirection );
    void setGlobalPositionAndDirection( const Position3D & globalPosition,
            const Direction3D & globalDirection );

    Entity * pVehicle() const;
    uint8 vehicleChangeNum() const;
    EntityID vehicleID() const { return pVehicle_ ? pVehicle_->id() : 0; }

    typedef uintptr SetVehicleParam;
    enum SetVehicleParamEnum
    {
        KEEP_LOCAL_POSITION,
        KEEP_GLOBAL_POSITION,
        IN_LIMBO
    };
    void setVehicle( Entity * pVehicle, SetVehicleParam keepWho );
    void onVehicleMove();

    NumTimesRealOffloadedType numTimesRealOffloaded() const
                                    { return numTimesRealOffloaded_; }

    EventNumber lastEventNumber() const;
    EventNumber getNextEventNumber();
    const PropertyEventStamps & propertyEventStamps() const;

    void debugDump();
    void fakeID( EntityID id );

    void addTrigger( RangeTrigger * pTrigger );
    void modTrigger( RangeTrigger * pTrigger );
    void delTrigger( RangeTrigger * pTrigger );

    bool hasBase() const { return baseAddr_.ip != 0; }
    const Mercury::Address & baseAddr() const { return baseAddr_; }
    void adjustForDeadBaseApp( const BackupHash & backupHash );
    void informBaseOfAddress( const Mercury::Address & addr, SpaceID spaceID,
           bool shouldSendNow );

    // ---- PropertyOwnerLink 实现 ----
    void onOwnedPropertyChanged( PropertyChange & change );
    bool getTopLevelOwner( PropertyChange & change,
            PropertyOwnerBase *& rpTopLevelOwner );
    int getNumOwnedProperties() const;
    PropertyOwnerBase * getChildPropertyOwner( int ref ) const;
    PyObjectPtr setOwnedProperty( int ref, BinaryIStream & data );

    PyObjectPtr propertyByLocalIndex( int index ) const;

    Chunk * pChunk() const { return pChunk_; };
    Entity * prevInChunk() const { return pPrevInChunk_;}
    Entity * nextInChunk() const { return pNextInChunk_;}
    void prevInChunk( Entity* pEntity ) { pPrevInChunk_ = pEntity; }
    void nextInChunk( Entity* pEntity ) { pNextInChunk_ = pEntity; }
    void removedFromChunk();

    void heardNoise( const Entity * who, float propRange, float distance,
                        int event, int info );

    ControllerID  addController( ControllerPtr pController, int userArg );
    void          modController( ControllerPtr pController );
    bool          delController( ControllerID controllerID,
                        bool warnOnFailure = true );

    static int registerEntityExtra(
        EntityExtra * (*touchFn)( Entity & e ) = NULL,
        PyDirInfo * pTouchDir = NULL );
    EntityExtra * & entityExtra( int eeid )        { return extras_[eeid]; }
    EntityExtra * entityExtra( int eeid ) const    { return extras_[eeid]; }

    void checkChunkCrossing();

    bool callback( const char * funcName, PyObject * args,
        const char * errorPrefix, bool okIfFunctionNull );

    static void callbacksPermitted( bool permitted );
    static bool callbacksPermitted()
        { return s_callbacksPrevented_ <= s_allowCallbacksOverride; }

    static void nominateRealEntity( Entity & e );
    static void nominateRealEntityPop();
    static void s_init();

private:
    Entity( const Entity & );

    void updateLocalPosition();
    bool updateGlobalPosition( bool shouldUpdateGhosts = true );
    void updateInternalsForNewPositionOfReal( const Vector3 & oldPos );
    void updateInternalsForNewPosition( const Vector3 & oldPosition );

    bool getEntitiesInRange( EntityVisitor & visitor, float range,
            PyObjectPtr pClass = NULL, PyObjectPtr pActualPos = NULL );
    void findEntitiesInSquare( float range, EntityVisitor & visitor ) const;

    void callScriptInit( bool isRestore, EntityPtr pNearbyEntity );
    void clearPythonProperties();

    bool readRealDataInEntityFromStreamForInitOrRestore( BinaryIStream & data,
        PyObject * pDict );

    void readGhostDataFromStream( BinaryIStream & data );
    void writeGhostDataToStream( BinaryOStream & stream ) const;
    void readGhostDataFromStreamInternal( BinaryIStream & data );
    void writeGhostDataToStreamInternal( BinaryOStream & stream ) const;

    void convertRealToGhost( BinaryOStream * pStream = NULL,
            CellAppChannel * pChannel = NULL,
            bool isTeleport = false );
    void writeRealDataToStream( BinaryOStream & data,
        const Mercury::Address & dstAddr, bool isTeleport ) const;
    void convertGhostToReal( BinaryIStream & data,
        const Mercury::Address * pBadHauntAddr = NULL );
    void readRealDataFromStreamForOnload( BinaryIStream & data,
        const Mercury::Address * pBadHauntAddr = NULL );
    void readRealDataFromStreamForOnloadInternal( BinaryIStream & data,
            const Mercury::Address * pBadHauntAddr );
    void writeRealDataToStreamInternal( BinaryOStream & data,
                    const Mercury::Address & dstAddr, bool isTeleport ) const;

    void setGlobalPosition( const Vector3 & v );
    void avatarUpdateCommon( const Coord & pos, const YawPitchRoll & dir,
        bool onGround, uint8 refNum );
    void setVolatileInfo( const VolatileInfo & newInfo );

    void writeVolatileDataToStream( Mercury::Bundle & bundle,
            const Vector3 & basePos, IDAlias idAlias,
            float priorityThreshold ) const;

    PyObject * pyGetAttribute( const char * attr );
    int pySetAttribute( const char * attr, PyObject * value );
    PyObject * pyAdditionalMembers( PyObject * pBaseSeq );
    PyObject * pyAdditionalMethods( PyObject * pBaseSeq );

    bool writeCellMessageToBundle( Mercury::Bundle & bundle,
        const MethodDescription * pDescription, PyObject * args ) const;
    bool writeClientMessageToBundle( Mercury::Bundle & bundle,
        const MethodDescription & description,
        MemoryOStream & argstream, int callingMode ) const;

    bool physicallyPossible( const Coord & newPosition, Entity * pVehicle,
        float propMove = 1.f );
    bool traverseChunks( Chunk * pCurChunk, const Chunk * pDstChunk,
        Vector3 cSrcPos, Vector3 cDstPos,
        std::vector< Chunk * > & visitedChunks );
    bool validateAvatarVehicleUpdate( Entity * pNewVehicle );

    void readGhostControllersFromStream( BinaryIStream & data );
    void writeGhostControllersToStream( BinaryOStream & stream ) const;
    void readRealControllersFromStream( BinaryIStream & data );
    void writeRealControllersToStream( BinaryOStream & stream ) const;
    void startRealControllers();
    void stopRealControllers();

    void runMethodHelper( BinaryIStream & data, int methodID, bool isExposed );

    bool sendDBDataToBase( const Mercury::Address * pReplyAddr = NULL,
        Mercury::ReplyID replyID = 0 );
    bool sendCellEntityLostToBase();

    // ---- Private data ----
    Space *             pSpace_;
    SpaceRemovalHandle  removalHandle_;
    EntityID            id_;
    EntityTypePtr       pEntityType_;
    Position3D          globalPosition_;
    Direction3D         globalDirection_;
    Position3D          localPosition_;
    Direction3D         localDirection_;
    Mercury::Address    baseAddr_;
    Entity *            pVehicle_;
    uint8               vehicleChangeNum_;
    AoIUpdateSchemeID   aoiUpdateSchemeID_;
    NumTimesRealOffloadedType numTimesRealOffloaded_;
    CellAppChannel *    pRealChannel_;
    Mercury::Address    nextRealAddr_;
    RealEntity *        pReal_;
    typedef std::vector<PyObjectPtr> Properties;
    Properties          properties_;
    PropertyOwnerLink<Entity> propertyOwner_;
    EventHistory        eventHistory_;
    bool                isDestroyed_;
    bool                inDestroy_;
    bool                isInAoIOffload_;
    bool                isOnGround_;
    VolatileInfo        volatileInfo_;
    VolatileNumber      volatileUpdateNumber_;
    float               topSpeed_;
    float               topSpeedY_;
    uint8               physicsCorrections_;
    uint64              physicsLastValidated_;
    float               physicsNetworkJitterDebt_;
    static float        s_maxPhysicsNetworkJitter_;
    PropertyEventStamps propertyEventStamps_;
    EventNumber         lastEventNumber_;
    EntityRangeListNode * pRangeListNode_;
    Controllers *       pControllers_;
    bool                shouldReturnID_;
    EntityExtra **      extras_;
    static std::vector<EntityExtraInfo*> & s_entityExtraInfo();

    typedef std::vector<RangeTrigger*> Triggers;
    Triggers            triggers_;

    enum { NOT_WITNESSED_THRESHOLD = 3 };
    mutable int         periodsWithoutWitness_;

    Chunk*              pChunk_;
    Entity*             pPrevInChunk_;
    Entity*             pNextInChunk_;

    static EntityPopulation population_;
    friend class RealEntity;
    friend class EntityRangeListNode;

    struct BufferedScriptCall
    {
        EntityPtr       entity;
        PyObject *      callable;
        PyObject *      args;
        const char *    errorPrefix;
    };
    static std::vector<BufferedScriptCall> s_callbacksBuffer_;
    static int s_callbacksPrevented_;
    static int s_allowCallbacksOverride;
};

typedef bool (*CustomPhysicsValidator)( Entity * pEntity,
    const Vector3 & newLocalPos, Entity * pNewVehicle,
    double physValidateTimeDelta );
extern CustomPhysicsValidator g_customPhysicsValidator;

typedef void (*EntityMovementCallback)( const Vector3 & oldPosition,
        Entity * pEntity );
extern EntityMovementCallback g_entityMovementCallback;
```

### 6.2 关键设计点

1. **Python 对象**：`Entity` 继承 `PyInstancePlus`，每个 entity 同时是一个 Python 对象，脚本通过 `BigWorld.entities[id]` 访问（由 [py_entities.hpp](file:///workspace/src/server/cellapp/py_entities.hpp) 中的 `PyEntities` 暴露）。
2. **Real/Ghost 双形态**：`pReal_` 非空时是 real 实体，否则是 ghost。`isReal()`、`isRealToScript()` 区分两种状态。
3. **位置双轨**：维护 `globalPosition_` 与 `localPosition_`——当实体搭乘载具时，local 是相对载具的，global 是世界坐标。
4. **chunk 关联**：`pChunk_/pPrevInChunk_/pNextInChunk_` 把实体串成 chunk 内的双向链表，便于 chunk 卸载时批量处理。
5. **callback 缓冲**：`callbacksPermitted()` 与 `s_callbacksBuffer_` 实现回调抑制——在 offload 等关键路径中，回调被缓冲，待操作完成后再批量回放，避免脚本误用尚未完成迁移的实体。
6. **物理校验钩子**：`g_customPhysicsValidator` 与 `g_entityMovementCallback` 是两个全局函数指针，允许游戏侧注入自定义物理校验或移动监听。
7. **事件历史**：`eventHistory_` 保存最近的状态变更事件，用于客户端「重连后追赶」与 ghost 间一致性校验。
8. **witnessed 阈值**：`periodsWithoutWitness_` 计数器——0/1 表示被 witness 看到，2 表示 real 没被看到但 ghost 被看到，3 表示 real 和 ghost 都没被看到（即 NOT_WITNESSED_THRESHOLD）。

### 6.3 实体创建主要路径

| 创建方式 | 入口 | 说明 |
|----------|------|------|
| baseapp 远程创建 cell entity | `Cell::createEntity` 消息 → `Cell::createEntityInternal` | baseapp 通过 `BigWorld.createCellEntity` 触发 |
| 在已有实体附近创建 | `createEntityNearEntity` 消息 → `CreateEntityNearEntityHandler` → `Cell::createEntity` | 由 baseapp 或其他 cellapp 发起 |
| 从 DB 恢复 | `Cell::restoreEntity` | cellappmgr 触发的恢复流程 |
| ghost 创建 | `Space::createGhost` → `Entity::initGhost` | real 端通过 `createGhost` bundle 发起 |
| teleport 接收 | `CellApp::onloadTeleportedEntity` → `Entity::onload` | 跨 cellapp teleport 落地 |

---

## 7. Real 与 Ghost：实体迁移与镜像

### 7.1 RealEntity：real 实体的附加数据

[real_entity.hpp](file:///workspace/src/server/cellapp/real_entity.hpp)：

```cpp
// /workspace/src/server/cellapp/real_entity.hpp
class RealEntity
{
    PY_FAKE_PYOBJECTPLUS_BASE_DECLARE()
    Py_FakeHeader( RealEntity, PyObjectPlus )

public:
    class Haunt
    {
    public:
        Haunt( CellAppChannel * pChannel, GameTime creationTime ) :
            pChannel_( pChannel ),
            creationTime_( creationTime )
        {}

        CellAppChannel & channel() { return *pChannel_; }
        Mercury::Bundle & bundle() { return pChannel_->bundle(); }
        const Mercury::Address & addr() const { return pChannel_->addr(); }

        void creationTime( GameTime time )  { creationTime_ = time; }
        GameTime creationTime() const       { return creationTime_; }

    private:
        CellAppChannel * pChannel_;
        GameTime creationTime_;
    };

    typedef std::vector< Haunt > Haunts;

    static void addWatchers();

    RealEntity( Entity & owner );

    bool init( BinaryIStream & data, CreateRealInfo createRealInfo,
            Mercury::ChannelVersion channelVersion = Mercury::SEQ_NULL,
            const Mercury::Address * pBadHauntAddr = NULL );

    void destroy( const Mercury::Address * pNextRealAddr = NULL );

    void writeOffloadData( BinaryOStream & data,
            const Mercury::Address & dstAddr,
            bool shouldSendPhysicsCorrection );

    void enableWitness( BinaryIStream & data, Mercury::ReplyID replyID );
    void disableWitness( bool isRestore = false );

    Entity & entity()                            { return entity_; }
    const Entity & entity() const                { return entity_; }

    Witness * pWitness()                         { return pWitness_; }
    const Witness * pWitness() const             { return pWitness_; }

    Haunts::iterator hauntsBegin() { return haunts_.begin(); }
    Haunts::iterator hauntsEnd()   { return haunts_.end(); }
    int numHaunts() const          { return haunts_.size(); }
    void addHaunt( CellAppChannel & channel );
    Haunts::iterator delHaunt( Haunts::iterator iter );

    HistoryEvent * addHistoryEvent( uint8 type,
        MemoryOStream & stream,
        bool sendToGhosts,
        HistoryEvent::Level level,
        EntityMemberStats * pChangedDescription,
        const std::string * pName = NULL );

    void backup();
    void autoBackup();
    void writeBackupProperties( BinaryOStream & data ) const;

    void debugDump();

    PyObject * pyGetAttribute( const char * attr );
    int pySetAttribute( const char * attr, PyObject * value );

    void sendPhysicsCorrection();
    void newPosition( const Vector3 & position );

    void addDelGhostMessage( Mercury::Bundle & bundle );
    void deleteGhosts();

    const NavLoc & navLoc() const        { return navLoc_; }
    void navLoc( const NavLoc & n )      { navLoc_ = n; }
    Navigator & navigator()              { return navigator_; }

    const Vector3 & velocity() const     { return velocity_; }

    EntityRemovalHandle removalHandle() const        { return removalHandle_; }
    void removalHandle( EntityRemovalHandle h )      { removalHandle_ = h; }

    const EntityMailBoxRef & controlledByRef() const { return controlledBy_; }
    GameTime creationTime() const         { return creationTime_; }
    void delControlledBy( EntityID deadID );
    Mercury::Channel & channel()          { return *pChannel_; }

    bool controlledBySelf() const { return entity_.id() == controlledBy_.id; }
    bool controlledByOther() const
            { return !this->controlledBySelf() && (controlledBy_.id != 0); }

    void teleport( const EntityMailBoxRef & dstMailBoxRef );
    bool teleport( const EntityMailBoxRef & nearbyMBRef,
        const Vector3 & position, const Vector3 & direction );
    PY_AUTO_METHOD_DECLARE( RETOK, teleport, ARG( EntityMailBoxRef,
        ARG( Vector3, ARG( Vector3, END ) ) ) )

    BaseEntityMailBoxPtr controlledBy();
    void controlledBy( BaseEntityMailBoxPtr pNewMaster );
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE(
        BaseEntityMailBoxPtr, controlledBy, controlledBy )

    bool isWitnessed() const;
    PY_RO_ATTRIBUTE_DECLARE( isWitnessed(), isWitnessed )
    PY_RO_ATTRIBUTE_DECLARE( !!pWitness_, hasWitness )
    PY_RW_ATTRIBUTE_DECLARE( shouldAutoBackup_, shouldAutoBackup )

private:
    ~RealEntity();

    bool readOffloadData( BinaryIStream & data,
        const Mercury::Address * pBadHauntAddr = NULL,
        bool * pHasChangedSpace = NULL );
    void readBackupData( BinaryIStream & data );
    void readBackupDataInternal( BinaryIStream & data );
    void writeBackupData( BinaryOStream & data ) const;
    void writeBackupDataInternal( BinaryOStream & data ) const;
    void setWitness( Witness * pWitness );
    void notifyWardOfControlChange( bool hasControl );

    Entity & entity_;
    Witness * pWitness_;
    Haunts haunts_;
    NavLoc navLoc_;
    Navigator navigator_;
    EntityRemovalHandle removalHandle_;
    EntityMailBoxRef controlledBy_;
    Vector3 velocity_;
    Vector3 positionSample_;
    GameTime positionSampleTime_;
    GameTime creationTime_;
    AutoBackupAndArchive::Policy shouldAutoBackup_;
    Mercury::Channel * pChannel_;
};
```

### 7.2 Haunt：每个 ghost 的句柄

`Haunt`（出没处）是 real 实体对**某个 ghost 实体所在 cellapp 的引用**：

- `pChannel_` 指向对应的 `CellAppChannel`；
- `creationTime_` 标记 ghost 创建时刻，用于一致性校验。

real 通过 `haunts_` 列表知道：「我在哪些邻居 cellapp 上有 ghost」。当 real 状态变更时，会遍历所有 haunts 把变更打包发出去（即 `ghostedDataUpdate`、`ghostHistoryEvent`、`ghostAvatarUpdate` 等消息）。

### 7.3 Real/Ghost 迁移流

```
              ┌──── Real Entity (cellapp A) ────┐
              │  pReal_ != nullptr              │
              │  haunts_ = [B, C, D]            │
              │  → 给 B/C/D 发 ghost 数据        │
              └──────────────┬──────────────────┘
                             │ offload（A 负载过高或实体跨边界）
                             ▼
              ┌──── A 上的 Entity 转 Ghost ────┐
              │  writeOffloadData → 序列化       │
              │  convertRealToGhost             │
              │  pRealChannel_ 指向 B           │
              │  nextRealAddr_ = B              │
              └──────────────┬──────────────────┘
                             │ onBaseOffloaded → 通知 base 新地址
                             ▼
              ┌──── B 上的 Entity 转 Real ─────┐
              │  onload / convertGhostToReal   │
              │  pReal_ 重建                    │
              │  haunts_ 重建（向 A/C/D 发 ghostSetReal）│
              └────────────────────────────────┘
```

关键消息：
- `ghostSetReal`：从 real 端发往所有 ghost，告知「我现在在 X」；
- `ghostSetNextReal`：迁移前的预热，告知 ghost「我即将迁到 Y」，让 ghost 提前缓冲消息（参见 [buffered_ghost_messages.hpp](file:///workspace/src/server/cellapp/buffered_ghost_messages.hpp)）；
- `onBaseOffloaded`：通知 base 实体的 cell 地址已变更；
- `onloadTeleportedEntity`：teleport 落地时由目标 cellapp 接收。

`numTimesRealOffloaded_`（`NumTimesRealOffloadedType`，uint16）记录 real 被迁出的累计次数，用于 ghost 端判断消息子序列边界（每个 real 地址对应一段子序列）。

---

## 8. Witness 与 AOI：客户端可见性管理

### 8.1 Witness：客户端之眼

[witness.hpp](file:///workspace/src/server/cellapp/witness.hpp)：

```cpp
// /workspace/src/server/cellapp/witness.hpp
class Witness : public Updatable
{
    PY_FAKE_PYOBJECTPLUS_BASE_DECLARE()
    Py_FakeHeader( Witness, PyObjectPlus )

public:
    Witness( RealEntity & owner, BinaryIStream & data,
            CreateRealInfo createRealInfo, bool hasChangedSpace = false );
    virtual ~Witness();
private:
    void init();
public:
    RealEntity & real()                { return real_; }
    const RealEntity & real() const    { return real_; }
    Entity & entity()                  { return entity_; }
    const Entity & entity() const      { return entity_; }

    void writeOffloadData( BinaryOStream & data,
            const Mercury::Address & dstAddr ) const;
    void writeBackupData( BinaryOStream & data ) const;

    bool sendToClient( int entityMessageType, MemoryOStream & stream );
    void sendToProxy( int mercuryMessageType, MemoryOStream & stream );

    void setWitnessCapacity( EntityID id, int bps );

    void requestEntityUpdate( EntityID id,
            EventNumber * pEventNumbers, int size );

    void addToAoI( Entity * pEntity );
    void removeFromAoI( Entity * pEntity );

    void newPosition( const Vector3 & position );
    void updateReferencePosition( uint8 seqNum );
    void cancelReferencePosition();
    void dumpAoI();
    void debugDump();
    void update();

    // ---- Python ----
    virtual PyObject * pyGetAttribute( const char * attr );
    virtual int pySetAttribute( const char * attr, PyObject * value );

    PY_RW_ATTRIBUTE_DECLARE( maxPacketSize_, bandwidthPerUpdate );
    PY_RW_ATTRIBUTE_DECLARE( stealthFactor_, stealthFactor )

    void unitTest();
    PY_AUTO_METHOD_DECLARE( RETVOID, unitTest, END )
    PY_AUTO_METHOD_DECLARE( RETVOID, dumpAoI, END )

    PY_AUTO_METHOD_DECLARE( RETOK, withholdFromClient,
            ARG( PyObjectPtr, OPTARG( bool, true, END ) ) )
    PY_AUTO_METHOD_DECLARE( RETOWN, isWithheldFromClient,
            ARG( PyObjectPtr, END ) )

    PY_AUTO_METHOD_DECLARE( RETDATA, isInAoI, ARG( PyObjectPtr, END ) )

    PY_AUTO_METHOD_DECLARE( RETOK, setAoIUpdateScheme,
            ARG( PyObjectPtr, ARG( std::string, END ) ) )
    PY_AUTO_METHOD_DECLARE( RETOWN, getAoIUpdateScheme,
            ARG( PyObjectPtr, END ) )

    void setAoIRadius( float radius, float hyst = 5.f );
    PY_AUTO_METHOD_DECLARE( RETVOID, setAoIRadius,
        ARG( float, OPTARG( float, 5.f, END ) ) )

    static void addWatchers();
    void vehicleChanged();

private:
    typedef std::vector< EntityCache * > KnownEntityQueue;

    bool withholdFromClient( PyObjectPtr pEntityID, bool isVisible );
    PyObject * isWithheldFromClient( PyObjectPtr pEntityID ) const;

    PyObject * py_entitiesInAoI( PyObject * args, PyObject * kwargs );
    bool isInAoI( PyObject * pEntityOrID ) const;

    bool setAoIUpdateScheme( PyObjectPtr pEntityOrID, std::string schemeName );
    PyObject * getAoIUpdateScheme( PyObjectPtr pEntityOrID );

    void addToSeen( EntityCache * pCache );
    void deleteFromSeen( Mercury::Bundle & bundle,
            KnownEntityQueue::iterator iter,
            EntityID id = 0 );

    void deleteFromClient( Mercury::Bundle & bundle,
        EntityCache * pCache, EntityID id = 0 );
    void deleteFromAoI( KnownEntityQueue::iterator iter );

    void handleEnterPending( Mercury::Bundle & bundle, 
        KnownEntityQueue::iterator iter );

    void sendEnter( Mercury::Bundle & bundle, EntityCache * pCache );
    void sendCreate( Mercury::Bundle & bundle, EntityCache * pCache );
    void sendGameTime();

    void onLeaveAoI( EntityCache * pCache, EntityID id );

    Mercury::Bundle & bundle();
    void sendToClient();

    IDAlias allocateIDAlias( const Entity & entity );
    void addSpaceDataChanges( Mercury::Bundle & bundle );

    const Vector3 & addReferencePosition( Mercury::Bundle & bundle );
    void calculateReferencePosition();

    static Entity * findEntityFromPyArg( PyObject * pEntityOrID );
    EntityCache * findEntityCacheFromPyArg( PyObject * pEntityOrID ) const;
    PyObject * entitiesInAoI( bool includeAll, bool onlyWithheld ) const;

    RealEntity & real_;
    Entity & entity_;

    GameTime noiseCheckTime_;
    GameTime noisePropagatedTime_;
    bool noiseMade_;

    int32 maxPacketSize_;

    KnownEntityQueue entityQueue_;
    EntityCacheMap   aoiMap_;

    float stealthFactor_;
    float aoiHyst_;
    float aoiRadius_;

    int32 bandwidthDeficit_;

    IDAlias freeAliases_[256];
    int    numFreeAliases_;

    Vector3 referencePosition_;
    uint8   referenceSeqNum_;
    bool hasReferencePosition_;

    int32 knownSpaceDataSeq_;
    uint32 allSpacesDataChangeSeq_;
    AoITrigger * pAoITrigger_;

    friend BinaryIStream & operator>>( BinaryIStream & stream,
            EntityCache & entityCache );
    friend BinaryOStream & operator<<( BinaryOStream & stream,
            const EntityCache & entityCache );
};
```

### 8.2 AOI 工作流

```
                         ┌─────────────┐
                         │  Witness    │  ← 挂在 RealEntity 上
                         │  aoiRadius_ │
                         │  aoiHyst_   │
                         │  aoiMap_    │  ← EntityCacheMap（实体缓存表）
                         │  entityQueue_│ ← 已知实体的优先级队列
                         └──────┬──────┘
                                │
                  AoITrigger（基于 RangeList）
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
         Entity 跨入 AoI    Entity 在 AoI 内    Entity 跨出 AoI
                │               │               │
                ▼               ▼               ▼
         addToAoI()      updatePriority()   removeFromAoI()
                │               │               │
                ▼               ▼               ▼
        ENTER_PENDING     按优先级排序      deleteFromClient()
                │               │               │
                ▼               ▼               ▼
        sendEnter()       写入 LoD 数据    onLeaveAoI()
                │               │               │
                ▼               ▼               ▼
        sendCreate()      updateDetailLevel()  释放 IDAlias
                                │
                                ▼
                      按 maxPacketSize_ 分包
                      → sendToClient() → proxy → client
```

### 8.3 EntityCache：AoI 内的实体快照

[entity_cache.hpp](file:///workspace/src/server/cellapp/entity_cache.hpp)：

```cpp
// /workspace/src/server/cellapp/entity_cache.hpp
const IDAlias NO_ID_ALIAS = 0xff;

class EntityCache
{
public:
    static const int MAX_LOD_LEVELS = 4;

    typedef double Priority;

    EntityCache( const Entity * pEntity );
    EntityCache( EntityID dummyID );
    ~EntityCache();

    static EntityCache * newDummy( EntityID dummyID );
    INLINE void construct();

    float updatePriority( const Vector3 & origin );

    void updateDetailLevel( Mercury::Bundle & bundle, float lodPriority );
    void addOuterDetailLevel( BinaryOStream & stream );

    void addLeaveAoIMessage( Mercury::Bundle & bundle, EntityID id ) const;

    void reuse();

    INLINE int numLoDLevels() const;
    INLINE static int numLoDLevels( const Entity & e );

    EntityConstPtr pEntity() const      { return pEntity_; }
    EntityConstPtr & pEntity()          { return pEntity_; }

    enum
    {
        VEHICLE_CHANGE_NUM_OLD,
        VEHICLE_CHANGE_NUM_HAS_VEHICLE,
        VEHICLE_CHANGE_NUM_HAS_NO_VEHICLE
    };
    typedef uint8 VehicleChangeNum;

    VehicleChangeNum vehicleChangeNum() const       { return vehicleChangeNum_; }
    void vehicleChangeNum( VehicleChangeNum num )   { vehicleChangeNum_ = num; }

    Priority priority() const;
    void priority( Priority newPriority );

    INLINE EntityID dummyID() const;
    INLINE void dummyID( EntityID dummyID );

    void lastEventNumber( EventNumber eventNumber );
    EventNumber lastEventNumber() const;

    void lastVolatileUpdateNumber( VolatileNumber number );
    VolatileNumber lastVolatileUpdateNumber() const;

    void detailLevel( DetailLevel detailLevel );
    DetailLevel detailLevel() const;

    IDAlias idAlias() const;
    void idAlias( IDAlias idAlias );

    AoIUpdateSchemeID updateSchemeID() const { return updateSchemeID_; }
    void updateSchemeID( AoIUpdateSchemeID id ) { updateSchemeID_ = id; }

    void lodEventNumbers( EventNumber * pEventNumbers, int size );

    void setEnterPending()        { flags_ |= ENTER_PENDING; }
    void setRequestPending()      { flags_ |= REQUEST_PENDING; }
    void setCreatePending()       { flags_ |= CREATE_PENDING; }
    void setGone()                { flags_ |= GONE; }
    void setWithheld()            { flags_ |= WITHHELD; }
    void setRefresh()             { flags_ |= REFRESH; }

    void clearEnterPending()      { flags_ &= ~ENTER_PENDING; }
    void clearRequestPending()    { flags_ &= ~REQUEST_PENDING; }
    void clearCreatePending()     { flags_ &= ~CREATE_PENDING; }
    void clearGone()              { flags_ &= ~GONE; }
    void clearWithheld()          { flags_ &= ~WITHHELD; }
    void clearRefresh()           { flags_ &= ~REFRESH; }

    bool isEnterPending() const   { return (flags_ & ENTER_PENDING) != 0; }
    bool isRequestPending() const { return (flags_ & REQUEST_PENDING) != 0; }
    bool isCreatePending() const  { return (flags_ & CREATE_PENDING) != 0; }
    bool isGone() const           { return (flags_ & GONE) != 0; }
    bool isWithheld() const       { return (flags_ & WITHHELD) != 0; }
    bool isRefresh() const        { return (flags_ & REFRESH) != 0; }
    bool isUpdatable() const      { return (flags_ & NOT_UPDATABLE) == 0; }

private:
    typedef uint8 Flags;

    enum
    {
        ENTER_PENDING   = 1 << 0, ///< 等待发 enterAoI 给客户端
        REQUEST_PENDING = 1 << 1, ///< 等待客户端的 requestEntityUpdate
        CREATE_PENDING  = 1 << 2, ///< 等待发 createEntity 给客户端
        GONE            = 1 << 3, ///< 等待从优先级队列移除
        WITHHELD        = 1 << 4, ///< 不发给客户端
        REFRESH         = 1 << 5, ///< 等待从 AoI 移除并重新加入

        NOT_UPDATABLE =
            ENTER_PENDING|REQUEST_PENDING|CREATE_PENDING|GONE|WITHHELD|REFRESH,
    };

    void lodEventNumber( int level, EventNumber eventNumber );
    EventNumber lodEventNumber( int level ) const;

    EntityCache & operator=( const EntityCache & );

    void addChangedProperties( BinaryOStream & stream,
        Mercury::Bundle * pBundleForHeader = NULL );

    EntityConstPtr      pEntity_;
    Flags               flags_;
    AoIUpdateSchemeID   updateSchemeID_;
    VehicleChangeNum    vehicleChangeNum_;

    union
    {
        Priority    priority_;       // double
        EntityID    dummyID_;        // 仅在没有 entity 时使用
    };

    EventNumber     lastEventNumber_;            // int32
    VolatileNumber  lastVolatileUpdateNumber_;   // uint16
    DetailLevel     detailLevel_;                // uint8
    IDAlias         idAlias_;                    // uint8

    EventNumber     lodEventNumbers_[ MAX_LOD_LEVELS ];   // int32 * num lod levels

    friend BinaryIStream & operator>>( BinaryIStream & stream,
            EntityCache & entityCache );
    friend BinaryOStream & operator<<( BinaryOStream & stream,
            const EntityCache & entityCache );
};
```

### 8.4 IDAlias：客户端侧的实体短地址

`IDAlias` 是 uint8（0~255，0xff 即 `NO_ID_ALIAS` 表示无效）。Witness 在客户端首次看见实体时分配一个 IDAlias，之后所有与该实体的通信都可用 1 字节代替 4 字节的 EntityID，**显著降低带宽**。`freeAliases_[256]` 维护可用别名池。

### 8.5 AoIUpdateSchemes：基于距离的更新权重

[aoi_update_schemes.hpp](file:///workspace/src/server/cellapp/aoi_update_schemes.hpp)：

```cpp
// /workspace/src/server/cellapp/aoi_update_schemes.hpp
typedef uint8 AoIUpdateSchemeID;

class AoIUpdateScheme
{
public:
    AoIUpdateScheme();
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
public:
    typedef AoIUpdateSchemeID SchemeID;

    static double apply( SchemeID scheme, float distance )
    {
        return schemes_[ scheme ].apply( distance );
    }

    static bool init();
    static bool getNameFromID( SchemeID id, std::string & name );
    static bool getIDFromName( const std::string & name, SchemeID & rID );

private:
    static AoIUpdateScheme schemes_[ 256 ];
    typedef std::map< std::string, SchemeID > NameToSchemeMap;
    static NameToSchemeMap nameToScheme_;
    typedef std::map< SchemeID, std::string > SchemeToNameMap;
    static SchemeToNameMap schemeToName_;
};
```

每个 EntityCache 关联一个 `AoIUpdateSchemeID`，Witness 在 `updatePriority` 中调用 `AoIUpdateSchemes::apply(scheme, distance)` 得到一个权重，进而影响该实体被推送的频率。**距离越远、权重越低，更新越稀疏**——这就是 BigWorld 的距离 LOD 调度核心。

### 8.6 ReferencePosition：相对坐标压缩

Witness 维护 `referencePosition_`（参考位置）和 `referenceSeqNum_`。当客户端 avatar 移动时，会带一个 `refNum` 上来；Witness 据此把当前 avatar 位置作为参考点，把 AoI 内其他实体的位置编码为相对偏移（3 个 uint8），大幅降低位置数据带宽。`updateReferencePosition(seqNum)` 与 `cancelReferencePosition()` 是这套机制的开关。

---

## 9. Range List：交叉触发与范围查询

### 9.1 RangeListNode：双向链表节点

[range_list_node.hpp](file:///workspace/src/server/cellapp/range_list_node.hpp)：

```cpp
// /workspace/src/server/cellapp/range_list_node.hpp
class RangeListNode
{
public:
    enum Flags
    {
        FLAG_MAKES_CROSSINGS = 1,   // 实体节点：会被触发
        FLAG_WANTS_CROSSINGS = 2,   // 触发器节点：希望被告知
        FLAG_LAST_BASE = 128
    };

    RangeListNode( uint16 flags, uint16 order ) :
        pPrevX_( NULL ), pNextX_( NULL ),
        pPrevZ_( NULL ), pNextZ_( NULL ),
        flags_( flags ), order_( order )
    { }
    virtual ~RangeListNode()    {}

    void shuffleXThenZ( float oldX, float oldZ );
    void shuffleX( float oldX, float oldZ );
    void shuffleZ( float oldX, float oldZ );

    RangeListNode* prevX() const    { return pPrevX_; }
    RangeListNode* prevZ() const    { return pPrevZ_; }
    RangeListNode* nextX() const    { return pNextX_; }
    RangeListNode* nextZ() const    { return pNextZ_; }

    RangeListNode * getNeighbour( bool getNext, bool getZ ) const
    {
        return getNext ?
            (getZ ? pNextZ_ : pNextX_) :
            (getZ ? pPrevZ_ : pPrevX_);
    }

    void prevX( RangeListNode* rln )    { pPrevX_ = rln; }
    void prevZ( RangeListNode* rln )    { pPrevZ_ = rln; }
    void nextX( RangeListNode* rln )    { pNextX_ = rln; }
    void nextZ( RangeListNode* rln )    { pNextZ_ = rln; }

    uint16 flags() const               { return flags_; }
    uint16 order() const               { return order_; }

    bool isEntity() const              { return flags_ & FLAG_MAKES_CROSSINGS; }

    void removeFromRangeList();
    void insertBeforeX( RangeListNode* entry );
    void insertBeforeZ( RangeListNode* entry );

    virtual void  debugRangeX() const;
    virtual void  debugRangeZ() const;
    virtual std::string debugString() const { return std::string( "Base" ); };

    float getCoord( bool getZ ) const { return getZ ? this->z() : this->x(); }

    virtual float x() const = 0;
    virtual float z() const = 0;

    virtual void crossedX( RangeListNode * /*node*/, bool /*positiveCrossing*/,
        float /*oldOthX*/, float /*oldOthZ*/ ) {}
    virtual void crossedZ( RangeListNode * /*node*/, bool /*positiveCrossing*/,
        float /*oldOthX*/, float /*oldOthZ*/ ) {}

protected:
    RangeListNode *pPrevX_;
    RangeListNode *pNextX_;
    RangeListNode *pPrevZ_;
    RangeListNode *pNextZ_;
    uint16         flags_;
    uint16         order_;
};
```

每个实体都拥有一个 `RangeListNode`（在 `Entity::pRangeListNode_` 中），同时插入到 Space 的 RangeList 的 **X 链表与 Z 链表**两条链中。当实体位置变化时，调用 `shuffleXThenZ(oldX, oldZ)` 重新排序，过程中如果跨越了某个 `RangeTriggerNode`，会回调 `crossedX/crossedZ`，进而触发 `triggerEnter/triggerLeave`。

### 9.2 RangeTrigger：上/下界触发器

```cpp
// /workspace/src/server/cellapp/range_list_node.hpp
class RangeTrigger
{
public:
    RangeTrigger( RangeListNode * pSubject, float range );
    virtual ~RangeTrigger();

    void insert();
    void remove();
    void removeWithoutContracting();

    void shuffleXThenZ( float oldX, float oldZ );
    void shuffleXThenZExpand( bool xInc, bool zInc, float oldX, float oldZ );
    void shuffleXThenZContract( bool xInc, bool zInc, float oldX, float oldZ );

    void setRange( float range );

    virtual std::string debugString() const;

    virtual void triggerEnter( RangeListNode * who ) = 0;
    virtual void triggerLeave( RangeListNode * who ) = 0;

    bool contains( RangeListNode * pQuery ) const;
    bool containsInZ( RangeListNode * pQuery ) const;

    RangeListNode * pSubject() const                  { return pSubject_; }
    const RangeListNode * pUpperTrigger() const       { return &upperBound_; }
    const RangeListNode * pLowerTrigger() const       { return &lowerBound_; }

    bool wasInXRange( float x, float range ) const;
    bool isInXRange( float x, float range ) const;
    bool wasInZRange( float z, float range ) const;
    bool isInZRange( float z, float range ) const;

    float range() const    { return upperBound_.range(); }

protected:
    RangeListNode *     pSubject_;
    RangeTriggerNode    upperBound_;
    RangeTriggerNode    lowerBound_;
public:
    float               oldX_;
    float               oldZ_;
};
```

`RangeTrigger` 由两个 `RangeTriggerNode`（上界、下界）组成，形成一个 `[lower, upper]` 区间。当任何实体节点跨入/跨出这个区间，触发器回调 `triggerEnter/triggerLeave`。`volatile` 关键字在 `wasInXRange` 等方法中的使用是为了**规避浮点精度差异**——保证触发判断与链表位置判断使用相同精度，否则会出现「漏触发」或「假触发」。

### 9.3 范围查询：entitiesInRange

`Entity::entitiesInRange(range, pClass, pActualPos)` 通过 `findEntitiesInSquare` 在 RangeList 上做一次方形区域扫描，结果通过 `EntityVisitor` 接口回调。这是脚本侧最常用的范围查询 API。

### 9.4 三类典型 RangeTrigger

| 触发器 | 文件 | 用途 |
|--------|------|------|
| `AoITrigger` | witness.cpp 内部 | Witness 的 AoI 边界，触发 `addToAoI/removeFromAoI` |
| `ProximityRangeTrigger` | proximity_controller.cpp | 陷阱/接近触发，回调脚本 `onEnterTrap/onLeaveTrap` |
| `VisionRangeTrigger` | vision_controller.cpp | 视野范围触发，配合 `EntityVision` 计算可见实体 |

---

## 10. Controller 系统：可插拔行为组件

### 10.1 Controller 基类

[controller.hpp](file:///workspace/src/server/cellapp/controller.hpp)：

```cpp
// /workspace/src/server/cellapp/controller.hpp
enum ControllerDomain
{
    DOMAIN_GHOST = 1,
    DOMAIN_REAL = 2,
    DOMAIN_GHOST_AND_REAL = DOMAIN_GHOST | DOMAIN_REAL
};

class Controller : public ReferenceCount
{
public:
    static const ControllerID EXCLUSIVE_CONTROLLER_ID_MASK = 0x7fff;

    Controller();
    virtual ~Controller();

    void init( Entity * pEntity, ControllerID id, int userArg );
    void disowned();

    virtual ControllerType      type() const = 0;
    virtual ControllerDomain    domain() const = 0;
    virtual ControllerID        exclusiveID() const = 0;

    const char *                typeName() const;

    bool    isAttached() const      { return !!pEntity_; }
    Entity& entity()                { return *pEntity_; }
    ControllerID id() const         { return controllerID_; }
    int     userArg()               { return userArg_; }

    virtual void    writeRealToStream( BinaryOStream & stream );
    virtual bool    readRealFromStream( BinaryIStream & stream );

    virtual void    writeGhostToStream( BinaryOStream & stream );
    virtual bool    readGhostFromStream( BinaryIStream & stream );

private:
    virtual void    startReal( bool isInitialStart );
    virtual void    stopReal( bool isFinalStop );

    virtual void    startGhost();
    virtual void    stopGhost();

    friend class Controllers;

protected:
    void    cancel();
    void    ghost();
    void    standardCallback( const char * methodName );

public:
    static ControllerPtr   create( ControllerType type );
    static PyObject *      factory( Entity * pEntity, const char * name );
    static ControllerType  factories( PyObject * pTuple = NULL, uint first = 0,
        const char * prefix = NULL );

    typedef Controller * (*CreatorFn)();

    struct FactoryFnRet
    {
        FactoryFnRet( void * = NULL ) : pController( NULL ), userArg( 0 ) { }
        FactoryFnRet( Controller * pC, int ua ) : pController( pC ), userArg( ua ) { }
        Controller * pController;
        int userArg;
    };
    typedef FactoryFnRet (*FactoryFn)( PyObject * args, PyObject * kwargs );

    static void registerControllerType( CreatorFn cfn,
        const char * typeName, ControllerType & ct, FactoryFn ffn );

    static ControllerID getExclusiveID( const char * exclusiveClass,
                bool createIfNecessary );

    class TypeRegisterer
    {
    public:
        TypeRegisterer( CreatorFn cfn, const char * typeName,
                ControllerDomain cd, FactoryFn ffn,
                const char * exclusiveClass ) :
            type_( ControllerType( -1 ) ),
            domain_( cd )
        {
            Controller::registerControllerType( cfn, typeName, type_, ffn );
            exclusiveID_ = Controller::getExclusiveID( exclusiveClass, true );
        }
        ControllerType      type() const        { return type_; }
        ControllerDomain    domain() const      { return domain_; }
        ControllerID        exclusiveID() const { return exclusiveID_; }
    private:
        ControllerType      type_;
        ControllerDomain    domain_;
        ControllerID        exclusiveID_;
    };

    static Entity *         s_factoryFnEntity_;

private:
    Entity *                pEntity_;
    int                     userArg_;
    ControllerID            controllerID_;
};
```

### 10.2 设计要点

1. **ReferenceCount**：Controller 是引用计数对象，`ControllerPtr = SmartPointer<Controller>`，自动管理生命周期。
2. **三域（Domain）**：
   - `DOMAIN_REAL`：仅在 real 实体上运行；
   - `DOMAIN_GHOST`：仅在 ghost 实体上运行；
   - `DOMAIN_GHOST_AND_REAL`：两侧都运行，real 端的状态会 ghost 到所有 ghost 端。
3. **生命周期回调**：`startReal/startGhost`、`stopReal/stopGhost` 在 real/ghost 启停时被 `Controllers` 调用；`writeRealToStream/readRealFromStream`、`writeGhostToStream/readGhostFromStream` 用于 offload/onload 时的状态序列化。
4. **互斥（Exclusive）**：`exclusiveID()` 与 `EXCLUSIVE_CONTROLLER_ID_MASK` 实现同类型 controller 的互斥——同一实体上同 exclusive ID 的 controller，新增会替换旧的。
5. **工厂注册**：通过 `IMPLEMENT_CONTROLLER_TYPE` / `IMPLEMENT_EXCLUSIVE_CONTROLLER_TYPE` 等宏 + `TypeRegisterer` 静态对象，在模块加载时自动注册到全局表，脚本侧可通过 `entity.addController(...)` 或快捷方法创建。

### 10.3 Controllers：实体上的控制器集合

[controllers.hpp](file:///workspace/src/server/cellapp/controllers.hpp)：

```cpp
// /workspace/src/server/cellapp/controllers.hpp
class Controllers
{
public:
    Controllers();
    ~Controllers();

    void readGhostsFromStream( BinaryIStream & data, Entity * pEntity );
    void readRealsFromStream( BinaryIStream & data, Entity * pEntity );

    void writeGhostsToStream( BinaryOStream & data );
    void writeRealsToStream( BinaryOStream & data );

    void createGhost( BinaryIStream & data, Entity * pEntity );
    void deleteGhost( BinaryIStream & data, Entity * pEntity );
    void updateGhost( BinaryIStream & data );

    ControllerID addController( ControllerPtr pController, int userArg,
            Entity * pEntity );
    bool delController( ControllerID id, Entity * pEntity,
            bool warnOnFailure = true );
    void modController( ControllerPtr pController, Entity * pEntity );

    void startReals();
    void stopReals( bool isFinalStop );

    PyObject * py_cancel( PyObject * args, Entity * pEntity );

private:
    ControllerID nextControllerID();

    typedef std::map< ControllerID, ControllerPtr > Container;
    Container container_;

    ControllerID lastAllocatedID_;
};
```

`Controllers` 是 `std::map<ControllerID, ControllerPtr>`。`addController` 返回的 ID 由 `nextControllerID()` 分配，脚本侧 `entity.cancel(controllerID)` 即通过此 ID 取消。

### 10.4 内置 Controller 一览

| Controller | 文件 | 域 | 用途 |
|------------|------|----|------|
| `TimerController` | [timer_controller.hpp](file:///workspace/src/server/cellapp/timer_controller.hpp) | REAL | 周期性回调脚本 `onTimer` |
| `MoveToPointController` | [move_controller.hpp](file:///workspace/src/server/cellapp/move_controller.hpp) | REAL | 直线移动到指定点 |
| `MoveToEntityController` | move_controller.hpp | REAL | 跟随移动到目标实体附近 |
| `NavigationController` | [navigation_controller.hpp](file:///workspace/src/server/cellapp/navigation_controller.hpp) | REAL | 沿导航网格寻路移动 |
| `BaseAccelerationController` | [base_acceleration_controller.hpp](file:///workspace/src/server/cellapp/base_acceleration_controller.hpp) | REAL | 加速运动基类 |
| `AccelerateToPointController` | accelerate_to_point_controller.hpp | REAL | 加速运动到点 |
| `AccelerateToEntityController` | accelerate_to_entity_controller.hpp | REAL | 加速运动到实体 |
| `AccelerateAlongPathController` | accelerate_along_path_controller.hpp | REAL | 沿路径加速运动 |
| `YawRotatorController` | [turn_controller.hpp](file:///workspace/src/server/cellapp/turn_controller.hpp) | REAL | 旋转到目标 yaw |
| `FaceEntityController` | [face_entity_controller.hpp](file:///workspace/src/server/cellapp/face_entity_controller.hpp) | REAL | 持续面向某实体 |
| `ProximityController` | [proximity_controller.hpp](file:///workspace/src/server/cellapp/proximity_controller.hpp) | REAL | 范围陷阱，触发 `onEnterTrap/onLeaveTrap` |
| `VisibilityController` | [visibility_controller.hpp](file:///workspace/src/server/cellapp/visibility_controller.hpp) | GHOST | 控制实体可见高度（被他人看见的高度） |
| `VisionController` | [vision_controller.hpp](file:///workspace/src/server/cellapp/vision_controller.hpp) | REAL | 简单视野（FOV+range） |
| `ScanVisionController` | [scan_vision_controller.hpp](file:///workspace/src/server/cellapp/scan_vision_controller.hpp) | REAL | 扫描式视野（带 amplitude/period 摆动） |
| `PassengerController` | [passenger_controller.hpp](file:///workspace/src/server/cellapp/passenger_controller.hpp) | GHOST | 实体搭乘载具的标记 |
| `PortalConfigController` | portal_config_controller.hpp | REAL | 控制 chunk portal 开关 |

### 10.5 典型 Controller：NavigationController

```cpp
// /workspace/src/server/cellapp/navigation_controller.hpp
class NavigationController : public Controller, public Updatable
{
    DECLARE_CONTROLLER_TYPE( NavigationController )
public:
    NavigationController( const Position3D & destination = Position3D( 0, 0, 0 ),
        float velocity = 0.f, bool faceMovement = true,
        float maxDistance = 500.f, float girth = 0.5f,
        float closeEnough = 0.01f );

protected:
    enum NavigationStatus
    {
        NAVIGATION_CANCELLED = -2,
        NAVIGATION_FAILED = -1,
        NAVIGATION_IN_PROGRESS = 0,
        NAVIGATION_COMPLETE = 1
    };

    virtual void    startReal( bool isInitialStart );
    virtual void    stopReal( bool isFinalStop );

    NavigationStatus move();
    void            writeRealToStream( BinaryOStream & stream );
    bool            readRealFromStream( BinaryIStream & stream );
    void            update();

private:
    void generateTraversalPath( const NavLoc & srcLoc );

    float           metresPerTick_;
    float           maxDistance_;
    float           girth_;
    float           closeEnough_;
    bool            faceMovement_;
    Position3D      nextPosition_;
    Position3D      destination_;
    NavLoc *        pDstLoc_;

    Vector3Path     path_;
    int             currentNode_;
};
```

`NavigationController` 同时继承 `Controller` 和 `Updatable`——后者让它被注册到 `CellApp::updatables_`，每个 game tick 调用 `update()` 推进一步。`generateTraversalPath` 利用 `NavLoc` 与 `Navigator`（来自 `waypoint/` 模块）做基于航点网格的 A* 寻路，结果存入 `path_`，`update()` 沿 `path_` 节点逐一前进。

---

## 11. 导航与移动：EntityNavigate 与 Navigator

### 11.1 EntityNavigate：导航 EntityExtra

[entity_navigate.hpp](file:///workspace/src/server/cellapp/entity_navigate.hpp)：

```cpp
// /workspace/src/server/cellapp/entity_navigate.hpp
class EntityNavigate : public EntityExtra
{
    Py_EntityExtraHeader( EntityNavigate )

public:
    EntityNavigate( Entity & e );
    ~EntityNavigate();

    PyObject * pyGetAttribute( const char * attr );
    int pySetAttribute( const char * attr, PyObject * value );

    PY_AUTO_METHOD_DECLARE( RETOWN, moveToPoint,
        ARG( Vector3, ARG( float, OPTARG( int, 0,
        OPTARG( bool, true, OPTARG( bool, false, END ) ) ) ) ) )
    PyObject * moveToPoint( Vector3 destination, float velocity,
                            int userArg = 0,
                            bool faceMovement = true,
                            bool moveVertically = false );

    PY_AUTO_METHOD_DECLARE( RETOWN, moveToEntity,
        ARG( int, ARG( float, ARG( float, OPTARG( int, 0,
        OPTARG( bool, true, OPTARG( bool, false, END ) ) ) ) ) ) )
    PyObject * moveToEntity( int destEntityID, float velocity, float range,
                             int userArg = 0, bool faceMovement = true,
                             bool moveVertically = false );

    PY_AUTO_METHOD_DECLARE( RETOWN, accelerateToPoint,
        ARG( Position3D, ARG( float, ARG( float,
        OPTARG( int, 1, OPTARG( bool, false,
        OPTARG( int, 0, END ) ) ) ) ) ) );
    PyObject * accelerateToPoint( Position3D destination, float acceleration,
                                  float maxSpeed, int facing = 1,
                                  bool stopAtDestination = true,
                                  int userArg = 0);

    PY_AUTO_METHOD_DECLARE( RETOWN, accelerateAlongPath, /* ... */ );
    PyObject * accelerateAlongPath( std::vector<Position3D> waypoints,
                                    float acceleration, float maxSpeed,
                                    int facing = 1, int userArg = 0);

    PY_AUTO_METHOD_DECLARE( RETOWN, accelerateToEntity, /* ... */ );
    PyObject * accelerateToEntity( EntityID destinationEntity,
                                   float acceleration, float maxSpeed,
                                   float range, int facing = 1,
                                   int userArg = 0);

    PY_AUTO_METHOD_DECLARE( RETOWN, navigate,
        ARG( Vector3, ARG( float,
        OPTARG( bool, true, OPTARG( float, 500.f, OPTARG( float, 0.5f, 
        OPTARG( float, 0.01f, OPTARG(int, 0, END ) ) ) ) ) ) ) )
    PyObject * navigate( const Vector3 & dstPosition, float velocity,
        bool faceMovement = true, float maxDistance = 500.f,
        float girth = 0.5f, float closeEnough = 0.01f, int userArg = 0 );

    PY_AUTO_METHOD_DECLARE( RETOWN, navigateStep, /* ... */ );
    PyObject * navigateStep( const Vector3 & dstPosition, float velocity,
        float maxMoveDistance, float maxSearchDistance = 500.f,
        bool faceMovement = true, float girth = 0.5f, int userArg = 0 );

    PY_AUTO_METHOD_DECLARE( RETOWN, navigateFollow, /* ... */ );
    PyObject * navigateFollow( PyObjectPtr pEntity, float angle, float offset,
        float velocity, float maxMoveDistance, float maxSearchDistance = 500.f, 
        bool faceMovement = true, float girth = 0.5f, int userArg = 0 );

    PY_AUTO_METHOD_DECLARE( RETOWN, canNavigateTo,
        ARG( Vector3, OPTARG( float, 500.f, OPTARG( float, 0.5f, END ) ) ) )
    PyObject * canNavigateTo( const Vector3 & dstPosition,
        float maxDistance = 500.f, float girth = 0.5f );

    PY_AUTO_METHOD_DECLARE( RETDATA, waySetPathLength, END )
    int waySetPathLength();

    PY_AUTO_METHOD_DECLARE( RETOWN, getStopPoint, /* ... */ );
    PyObject * getStopPoint( const Vector3 & dstPosition,
            bool ignoreFirstStopPoint, float maxDistance = 500.f,
            float girth = 0.5f, float stopDistance = 1.5f, 
            float nearPortalDistance = 1.8f );

    PY_AUTO_METHOD_DECLARE( RETOWN, navigatePathPoints, /* ... */ );
    PyObject * navigatePathPoints( const Vector3 & dstPosition, 
        float maxSearchDistance = 500.f, float girth = 0.5 );

    bool getNavigatePosition( class NavLoc srcLoc, class NavLoc dstLoc,
        float maxDistance, Vector3& nextPosition, bool & passedActivatedPortal,
        float girth = 0.5f );

    static const Instance<EntityNavigate> instance;
    void validateNavLoc( const Vector3 & position,
            float girth, class NavLoc & out );
};
```

`EntityNavigate` 是一个 `EntityExtra`（参见 §19），脚本侧通过 `entity.navigate(...)`、`entity.moveToPoint(...)` 等方法访问。它内部创建对应的 `MoveController`、`NavigationController`、`BaseAccelerationController` 等子类，并通过 `entity.addController` 注册。

### 11.2 RealEntity 上的 Navigator

[real_entity.hpp](file:///workspace/src/server/cellapp/real_entity.hpp) 中可见：

```cpp
const NavLoc & navLoc() const    { return navLoc_; }
void navLoc( const NavLoc & n )  { navLoc_ = n; }
Navigator & navigator()          { return navigator_; }
```

每个 real 实体持有一个 `Navigator` 实例（封装 waypoint 网格的查询与路径搜索）和当前 `NavLoc`（实体在网格上的位置）。`NavigationController::generateTraversalPath` 通过这两个对象做寻路。

---

## 12. 车辆系统：PassengerController / Passengers

### 12.1 PassengerController

[passenger_controller.hpp](file:///workspace/src/server/cellapp/passenger_controller.hpp)：

```cpp
// /workspace/src/server/cellapp/passenger_controller.hpp
class PassengerController : public Controller, public EntityPopulation::Observer
{
    DECLARE_CONTROLLER_TYPE( PassengerController )
public:
    PassengerController( EntityID vehicleID = 0 );
    ~PassengerController();

    virtual void writeGhostToStream( BinaryOStream & stream );
    virtual bool readGhostFromStream( BinaryIStream & stream );

    virtual void startGhost();
    virtual void stopGhost();

    void onVehicleGone();

    // Override from PopulationWatcher
    virtual void onEntityAdded( Entity & entity );

private:
    PassengerController( const PassengerController & );
    PassengerController& operator=( const PassengerController & );

    EntityID        vehicleID_;
    Position3D      initialLocalPosition_;
    Direction3D     initialLocalDirection_;
    Position3D      initialGlobalPosition_;
    Direction3D     initialGlobalPosition_;
};
```

`PassengerController` 是 ghost 域的 controller，挂在乘客实体上。它实现了 `EntityPopulation::Observer`——当车辆实体在本进程被创建（onload 后），`onEntityAdded` 被回调，此时把乘客实际挂到车辆上。

### 12.2 Passengers：车辆侧的乘客集合

[passengers.hpp](file:///workspace/src/server/cellapp/passengers.hpp)：

```cpp
// /workspace/src/server/cellapp/passengers.hpp
class Passengers : public EntityExtra, public Aligned
{
public:
    Passengers( Entity & e );
    ~Passengers();

    bool add( Entity & entity );
    bool remove( Entity & entity );

    void updateInternalsForNewPosition( const Vector3 & oldPosition );

    const Matrix & transform() const    { return transform_; }

    static const Instance<Passengers> instance;

private:
    Passengers( const Passengers& );
    Passengers& operator=( const Passengers& );

    void adjustTransform();

    typedef std::vector< Entity * > Container;

    bool inDestructor_;

    Container passengers_;
    Matrix transform_;
};
```

挂在**车辆**实体上的 `EntityExtra`，维护乘客列表与车辆的变换矩阵。当车辆移动时，`updateInternalsForNewPosition` 会把所有乘客的 global 位置同步更新（乘客的 local 位置不变）。

### 12.3 PassengerExtra：乘客侧的辅助

[passenger_extra.hpp](file:///workspace/src/server/cellapp/passenger_extra.hpp)：

```cpp
// /workspace/src/server/cellapp/passenger_extra.hpp
class PassengerExtra : public EntityExtra
{
    Py_EntityExtraHeader( PassengerExtra )

public:
    PassengerExtra( Entity & e );
    ~PassengerExtra();

    void setController( PassengerController * pController );
    void onVehicleGone();
    static const Instance<PassengerExtra> instance;

    bool boardVehicle( EntityID vehicleID, bool shouldUpdateClient = true );
    bool alightVehicle( bool shouldUpdateClient = true );

    static bool isInLimbo( Entity & e );

    virtual void relocated();

protected:
    PyObject * pyGetAttribute( const char * attr );
    int pySetAttribute( const char * attr, PyObject * value );

    PY_AUTO_METHOD_DECLARE( RETOK, boardVehicle, ARG( EntityID, END ) );
    PY_AUTO_METHOD_DECLARE( RETOK, alightVehicle, END );

private:
    PassengerController * pController_;
};
```

挂在**乘客**实体上的 `EntityExtra`，提供 `boardVehicle/alightVehicle` 脚本接口，与 `PassengerController` 协同。

### 12.4 上下车流程

```
脚本: passenger.boardVehicle(vehicleID)
        │
        ▼
PassengerExtra::boardVehicle
        │
        ├─ 创建 PassengerController（domain=GHOST）
        ├─ entity.addController(...)
        ├─ 把 PassengerController 同步 ghost 到所有 haunt
        └─ Vehicle 端: Passengers::add(passenger)
                       └─ passenger.pVehicle_ = vehicle
                          passenger.localPosition = globalPos - vehiclePos

车辆移动:
  vehicle.globalPosition 变化
        │
        ▼
  Passengers::updateInternalsForNewPosition(oldPos)
        │
        ▼
  遍历 passengers_, 计算新的 globalPosition
        │
        ▼
  passenger.updateGlobalPosition() → ghost 同步、AOI 重新计算
```

---

## 13. Ghost 消息缓冲与乱序重排

### 13.1 问题背景

Real 实体在多个 cellapp 上有 ghost。当 real offload 到新 cellapp 时，旧 cellapp 上尚未发出的 ghost 消息可能比新 cellapp 上发出的消息**晚到达** ghost 端，造成状态错乱。

### 13.2 BufferedGhostMessage

[buffered_ghost_message.hpp](file:///workspace/src/server/cellapp/buffered_ghost_message.hpp)：

```cpp
// /workspace/src/server/cellapp/buffered_ghost_message.hpp
class BufferedGhostMessage
{
public:
    BufferedGhostMessage( bool isSubsequenceStart, bool isSubsequenceEnd ) :
        isSubsequenceStart_( isSubsequenceStart ),
        isSubsequenceEnd_( isSubsequenceEnd )
    {
    }

    virtual void play() = 0;
    virtual bool isCreateGhostMessage() const { return false; }

    bool isSubsequenceStart() const { return isSubsequenceStart_; }
    bool isSubsequenceEnd() const   { return isSubsequenceEnd_; }

private:
    bool isSubsequenceStart_;
    bool isSubsequenceEnd_;
};
```

每条缓冲消息知道自己是否是「子序列起点/终点」。

### 13.3 BufferedGhostMessages：单例重排器

[buffered_ghost_messages.hpp](file:///workspace/src/server/cellapp/buffered_ghost_messages.hpp)：

```cpp
// /workspace/src/server/cellapp/buffered_ghost_messages.hpp
class BufferedGhostMessages
{
public:
    void playSubsequenceFor( EntityID id, const Mercury::Address & srcAddr );
    void playNewLifespanFor( EntityID id );

    bool hasMessagesFor( EntityID entityID, 
        const Mercury::Address & addr ) const;

    bool isDelayingMessagesFor( EntityID entityID,
            const Mercury::Address & addr ) const;

    void delaySubsequence( EntityID entityID,
            const Mercury::Address & srcAddr,
            BufferedGhostMessage * pFirstMessage );

    void add( EntityID entityID, const Mercury::Address & srcAddr,
                   BufferedGhostMessage * pMessage );

    bool isEmpty() const    { return map_.empty(); }

    static BufferedGhostMessages & instance();

private:
    typedef std::map< EntityID, BufferedGhostMessagesForEntity > Map;
    Map map_;
};
```

注释明确说明设计：

> Messages from a single CellApp are ordered but if the real entity is offload, some ghost messages may arrive too soon. To get around this problem, we consider the lifespan of a ghost as being split into subsequence of messages from each CellApp that the real visits. The Real entity tells ghosts of the next address before offloading. This allows the ghost to chain these subsequences together and play them in the correct order.

### 13.4 子序列拼接

```
Real 在 cellapp A:                    Real 在 cellapp B:
   ghostSetNextReal(B) ──┐              (已 onload)
   real 数据更新 x 3     │              ghostSetReal(B, numOffloaded++)
   ghostSetReal(B, 1) ───┘                  │
                                            ▼
                                   ghost 上 ghostSetReal 触发：
                                   1. 把 A 的子序列标记为 "结束"
                                   2. 等待 B 的子序列到达后再 play
                                   3. B 的子序列开始 play
```

`numTimesRealOffloaded_` 就是用来标识「这是第几段子序列」——ghost 端按这个数字顺序拼接，避免新旧 cellapp 的消息交叉。

### 13.5 BufferedEntityMessages：实体消息缓冲

[buffered_entity_messages.hpp](file:///workspace/src/server/cellapp/buffered_entity_messages.hpp)：

```cpp
// /workspace/src/server/cellapp/buffered_entity_messages.hpp
class BufferedEntityMessages
{
public:
    void playBufferedMessages( CellApp & app );

    void add( EntityMessageHandler & handler,
        const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data, EntityID entityID );

    bool isEmpty() const
        { return bufferedMessages_.empty(); }

private:
    std::list< BufferedEntityMessage * > bufferedMessages_;
};
```

[entity_message_handler.hpp](file:///workspace/src/server/cellapp/entity_message_handler.hpp) 中的 `EntityMessageHandler` 在构造时可指定 `shouldBufferIfTickPending`——如果当前 tick 还在进行中，到达的实体消息会被缓冲到 `BufferedEntityMessages`，待 tick 完成后再统一回放，避免在 tick 中途改变实体集合导致状态不一致。

### 13.6 CreateEntityNearEntityHandler

[create_entity_near_entity_handler.hpp](file:///workspace/src/server/cellapp/create_entity_near_entity_handler.hpp) 是 `createEntityNearEntity` 消息的专用处理器，因为该消息需要一个「附近实体」作为参考——若附近实体尚未在本进程创建，必须缓冲等待。

---

## 14. CellApp 间通信与 CellAppChannel

### 14.1 CellAppChannel

[cell_app_channel.hpp](file:///workspace/src/server/cellapp/cell_app_channel.hpp)：

```cpp
// /workspace/src/server/cellapp/cell_app_channel.hpp
class CellAppChannel : public Mercury::ChannelOwner
{
public:
    bool isOverloaded() const;

    int mark() const                { return mark_; }
    void mark( int v )              { mark_ = v; }
    int offloadCapacity() const     { return offloadCapacity_; }
    void offloadCapacity( int v )   { offloadCapacity_ = v; }
    int ghostingCapacity() const    { return ghostingCapacity_; }
    void ghostingCapacity( int v )  { ghostingCapacity_ = v; }
    bool isGood() const             { return !this->channel().hasRemoteFailed(); }

private:
    CellAppChannel( const Mercury::Address & addr );

    int     mark_;
    int     offloadCapacity_;
    int     ghostingCapacity_;
    int     numHaunts_;

    friend class CellAppChannels;
};
```

`CellAppChannel` 是「本 cellapp → 邻居 cellapp」的连接，封装了 Mercury 的 `ChannelOwner`。除了基本的消息收发，还维护两个**配额**：
- `offloadCapacity_`：本轮还能 offload 多少实体给对方；
- `ghostingCapacity_`：本轮还能创建多少 ghost 在对方上。

这两个配额由 `CellAppChannels::setCapacities` 全局设置，并在 `Cell::checkOffloadsAndGhosts` 中消费，防止单帧集中迁移造成邻居过载。

### 14.2 CellAppChannels：全局管理与定时发送

[cell_app_channels.hpp](file:///workspace/src/server/cellapp/cell_app_channels.hpp)：

```cpp
// /workspace/src/server/cellapp/cell_app_channels.hpp
class CellAppChannels : public Singleton< CellAppChannels >,
    public TimerHandler
{
public:
    CellAppChannels( int microseconds, Mercury::EventDispatcher & dispatcher );
    ~CellAppChannels();

    CellAppChannel * get( const Mercury::Address & addr,
        bool shouldCreate = true );

    void remoteFailure( const Mercury::Address & addr );
    void setCapacities( int ghostingCapacity, int offloadCapacity );

private:
    void sendAll();
    virtual void handleTimeout( TimerHandle handle, void * arg );

    typedef std::map< Mercury::Address, CellAppChannel * > Map;
    typedef Map::iterator iterator;
    Map map_;

    static const int RECENTLY_DEAD_PERIOD = 10;
    typedef std::set< Mercury::Address > RecentlyDead;
    RecentlyDead recentlyDead_;

    uint64 lastTimeOfDeath_;
    TimerHandle timer_;
};
```

要点：
- 单例 + 定时器，每隔 `microseconds` 微秒统一 `sendAll()`——把所有 channel 的 bundle 发出去；
- `recentlyDead_` 记录最近死亡的 cellapp 地址，避免立刻重建 channel 浪费；
- `RECENTLY_DEAD_PERIOD = 10` 是死亡后保留记录的 tick 数。

### 14.3 CellAppMgrGateway：与 CellAppMgr 的网关

[cellappmgr_gateway.hpp](file:///workspace/src/server/cellapp/cellappmgr_gateway.hpp)：

```cpp
// /workspace/src/server/cellapp/cellappmgr_gateway.hpp
class CellAppMgrGateway : public ManagerAppGateway
{
public:
    CellAppMgrGateway( Mercury::NetworkInterface & interface );

    void add( const Mercury::Address & addr, uint16 viewerPort,
            Mercury::ReplyMessageHandler * pReplyHandler );

    void informOfLoad( float load );
    void handleCellAppDeath( const Mercury::Address & addr );
    void setSharedData( const std::string & key, const std::string & value,
           SharedDataType type );
    void delSharedData( const std::string & key, SharedDataType type );
    void ackCellAppDeath( const Mercury::Address & deadAddr );
    void ackShutdown( ShutDownStage stage );
    void updateBounds( const Cells & cells, int maxEntityOffload );
    void onManagerRebirth( CellApp & cellApp, const Mercury::Address & addr );
    void shutDownSpace( SpaceID spaceID );

    const Mercury::Address & addr() const   { return channel_.addr(); }
    Mercury::Bundle & bundle()              { return channel_.bundle(); }
    void send()                             { channel_.send(); }
};
```

`CellAppMgrGateway` 是 cellapp → cellappmgr 的所有调用的统一出口：注册、负载上报、共享数据同步、死亡通知 ACK、边界更新、停机 ACK 等。

---

## 15. CellApp 死亡检测与恢复

### 15.1 CellAppDeathListener 模式

[cellapp_death_listener.hpp](file:///workspace/src/server/cellapp/cellapp_death_listener.hpp)：

```cpp
// /workspace/src/server/cellapp/cellapp_death_listener.hpp
class CellAppDeathListeners
{
public:
    typedef std::list< CellAppDeathListener* > Listeners;
    typedef Listeners::iterator iterator;

    static CellAppDeathListeners & instance();

    void handleCellAppDeath( const Mercury::Address & addr );
    iterator addListener( CellAppDeathListener * pListener );
    void delListener( iterator & iter );

private:
    static CellAppDeathListeners * s_pInstance_;
    Listeners listeners_;
};

class CellAppDeathListener
{
public:
    CellAppDeathListener();
    virtual ~CellAppDeathListener();

    virtual void handleCellAppDeath( const Mercury::Address & addr ) = 0;

private:
    CellAppDeathListeners::iterator iter_;
};
```

经典观察者模式：任何需要在 cellapp 死亡时清理资源的对象（`EntityPopulation`、`ReplacedGhosts`、持有 `CellAppChannel` 的对象等）继承 `CellAppDeathListener`，自动注册到全局列表，死亡事件由 `CellApp::handleCellAppDeath` 触发广播。

### 15.2 EntityPopulation 的双重角色

[entity_population.hpp](file:///workspace/src/server/cellapp/entity_population.hpp)：

```cpp
// /workspace/src/server/cellapp/entity_population.hpp
class EntityPopulation :
    public std::map< EntityID, Entity * >,
    public CellAppDeathListener
{
public:
    class Observer
    {
    public:
        virtual ~Observer() {};
        virtual void onEntityAdded( Entity & entity ) = 0;
    };

    void add( Entity & entity );
    void remove( Entity & entity );

    void addObserver( EntityID id, Observer * pObserver ) const;
    bool removeObserver( EntityID id, Observer * pObserver ) const;
    void notifyObservers( Entity & entity ) const;

    void rememberRealChannel( EntityID id, CellAppChannel & channel );
    void forgetRealChannel( EntityID id )
    {
        currChannels_.erase( id );
        prevChannels_.erase( id );
    }
    void rememberBaseChannel( Entity & entity, const Mercury::Address & addr ) const;
    void expireRealChannels() const;

    void notifyBasesOfCellAppDeath( const Mercury::Address & addr,
                                    Mercury::BundleSendingMap & bundles,
                                    AckCellAppDeathHelper * pAckHelper ) const;

    CellAppChannel * findRealChannel( EntityID id ) const;

private:
    typedef std::map< EntityID, CellAppChannel* > RealChannels;
    virtual void handleCellAppDeath( const Mercury::Address & addr );
    void forgetRealChannel( RealChannels & channels,
        const Mercury::Address & addr );

    typedef std::multimap< EntityID, Observer * > Observers;
    mutable RealChannels currChannels_;
    mutable RealChannels prevChannels_;

    struct RecentTeleportData
    {
        Mercury::Address baseAddr;
        SpaceID spaceID;
        EntityID entityID;
    };
    typedef std::multimap< Mercury::Address, RecentTeleportData > BaseAddrs;
    mutable BaseAddrs currRecentTeleportData_;
    mutable BaseAddrs prevRecentTeleportData_;

    void notifyBaseRangeOfCellAppDeath( 
        BaseAddrs & container, 
        const Mercury::Address & addr,
        Mercury::BundleSendingMap & bundles,
        AckCellAppDeathHelper * pAckHelper ) const;

    mutable Observers observers_;
};
```

`EntityPopulation` 同时是：
- **实体集合**：继承 `std::map<EntityID, Entity*>`，是 `CellApp::findEntity` 的底层；
- **CellAppDeathListener**：邻居死亡时清理对应的 `currChannels_/prevChannels_`；
- **Observer 通知源**：`PassengerController` 等通过 `addObserver` 监听特定 entity 的创建。

`currChannels_/prevChannels_` 双 map 设计：保存「当前」与「上一轮」的 real channel，便于在 cellapp 死亡时区分刚迁移过去的实体（避免误清理）。

### 15.3 AckCellAppDeathHelper：确认死亡处理完成

[ack_cell_app_death_helper.hpp](file:///workspace/src/server/cellapp/ack_cell_app_death_helper.hpp)：

```cpp
// /workspace/src/server/cellapp/ack_cell_app_death_helper.hpp
class AckCellAppDeathHelper :
    public TimerHandler,
    public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    AckCellAppDeathHelper( CellApp & app, const Mercury::Address & deadAddr );

    ~AckCellAppDeathHelper()
    {
        --s_numInstances_;
    }

    virtual void handleTimeout( TimerHandle handle, void * arg );
    virtual void handleMessage( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data, void * arg );
    virtual void handleException( const Mercury::NubException & exc, void * );

    void addCriticalEntity( Entity * pEntity )
    {
        recentOnloads_.insert( pEntity );
    }

    void addBadGhost()
    {
        ++numBadGhosts_;
    }

    void startTimer();

private:
    void checkFinished();

    CellApp & app_;
    typedef std::set< EntityPtr > RecentOnloads;
    RecentOnloads recentOnloads_;
    Mercury::Address deadAddr_;
    int checkPeriod_;
    int numBadGhosts_;
    TimerHandle timerHandle_;
    uint64 startTime_;
    static int s_numInstances_;
};
```

注释清晰描述了它解决的死锁问题：

> We can't do this until:
> - All the reals on this app have informed their bases of their addresses.
> - All emergencySetCurrentCell messages have been delivered and replied to.
>
> We require both of these things so that all lost reals will be restored, and so that reals that weren't lost won't be restored.

也就是说，**只有当本进程把所有受影响的 real 都重新告知了 base，并且所有 `emergencySetCurrentCell` 消息都收到 reply 后**，才能向 CellAppMgr 回 ACK——否则 CellAppMgr 会过早认为恢复完成，导致部分 real 永久丢失或重复恢复。

---

## 16. 负载均衡与流控

### 16.1 EmergencyThrottle：紧急限流

[emergency_throttle.hpp](file:///workspace/src/server/cellapp/emergency_throttle.hpp)：

```cpp
// /workspace/src/server/cellapp/emergency_throttle.hpp
class EmergencyThrottle
{
public:
    EmergencyThrottle();

    void update( float numSecondsBehind, float spareTime, float tickPeriod );
    float value() const    { return value_; }
    float estimatePersistentLoadTime( float persistentLoadTime,
                                        float throttledLoadTime) const;

    static WatcherPtr pWatcher();

private:
    bool shouldScaleBack( float timeToNextTick, float spareTime ) const;
    bool scaleBack( float fraction );
    bool scaleForward( float fraction );

    float value_;
    bool shouldPrintScaleForward_;
    uint64 startTime_;
    float maxTimeBehind_;
};
```

`EmergencyThrottle` 是 cellapp 的「自动变速箱」：
- 当 `numSecondsBehind`（落后秒数）超过 `behindThreshold` 且 `spareTime` 不足时，调用 `scaleBack` 降低 `value_`（即减少每 tick 处理的实体数量）；
- 反之逐步 `scaleForward` 提速。

`value_` 是 0~1 的系数，影响 offload/ghost 创建、AoI 更新等「可选」工作的吞吐量。配置由 [throttle_config.hpp](file:///workspace/src/server/cellapp/throttle_config.hpp) 提供：

```cpp
// /workspace/src/server/cellapp/throttle_config.hpp
class ThrottleConfig
{
public:
    static ServerAppOption< float > behindThreshold;
    static ServerAppOption< float > spareTimeThreshold;
    static ServerAppOption< float > scaleForwardTime;
    static ServerAppOption< float > scaleBackTime;
    static ServerAppOption< float > min;
};
```

### 16.2 负载计算与上报

`CellApp` 内部维护：
- `persistentLoad_`：长期负载（每 tick 累积，平滑后上报给 CellAppMgr）；
- `transientLoad_`：瞬态负载（每 tick 重置）；
- `totalLoad_`：当前总负载。

`updateLoad()` 在每 tick 末尾计算，`CellAppMgrGateway::informOfLoad(persistentLoad_)` 上报。CellAppMgr 据此决定是否给本进程分配新 Cell、是否触发 offload。

### 16.3 边界更新与 offload 检查

```cpp
// CellApp 内部
void updateBoundary();  // 把本进程所有 Cell 的边界写到 stream 上报
void checkOffloads();   // 检查每个 Cell 是否需要 offload 实体
void tickBackup();      // 周期性备份
```

`Cell::shouldOffload_` 由 CellAppMgr 通过 `shouldOffload` 消息控制——当本进程负载过高时，CellAppMgr 会把某些 Cell 标记为「应该 offload」，`checkOffloadsAndGhosts` 据此把 real 实体迁出到邻居。

### 16.4 配置项

[cellapp_config.hpp](file:///workspace/src/server/cellapp/cellapp_config.hpp) 中关键负载相关项：

```cpp
// /workspace/src/server/cellapp/cellapp_config.hpp
class CellAppConfig : public EntityAppConfig
{
public:
    static ServerAppOption< float > loadSmoothingBias;
    static ServerAppOption< float > ghostDistance;
    static ServerAppOption< float > defaultAoIRadius;
    static ServerAppOption< bool >  fastShutdown;
    static ServerAppOption< bool >  shouldResolveMailBoxes;
    static ServerAppOption< int >   entitySpamSize;

    static ServerAppOption< int >   maxGhostsToDelete;
    static ServerAppOption< float > minGhostLifespan;
    static ServerAppOption< int >   minGhostLifespanInTicks;

    static ServerAppOption< bool >  loadDominantTextureMaps;

    static ServerAppOption< float > backupPeriod;
    static ServerAppOption< int >   backupPeriodInTicks;

    static ServerAppOption< float > checkOffloadsPeriod;
    static ServerAppOption< int >   checkOffloadsPeriodInTicks;

    static ServerAppOption< float > chunkLoadingPeriod;

    static ServerAppOption< int >   ghostUpdateHertz;
    static ServerAppOption< double > reservedTickFraction;

    static ServerAppOption< int >   obstacleTreeDepth;
    static ServerAppOption< float > sendWindowCallbackThreshold;

    static ServerAppOption< float > offloadHysteresis;
    static ServerAppOption< int >   offloadMaxPerCheck;
    static ServerAppOption< int >   ghostingMaxPerCheck;

    static ServerAppOption< int >   expectedMaxControllers;
    static ServerAppOption< int >   absoluteMaxControllers;

    static ServerAppOption< bool >  enforceGhostDecorators;
    static ServerAppOption< bool >  treatAllOtherEntitiesAsGhosts;

    static ServerAppOption< float > maxPhysicsNetworkJitter;
};
```

| 配置项 | 含义 |
|--------|------|
| `loadSmoothingBias` | persistent load 的指数平滑系数 |
| `ghostDistance` | ghost 创建的距离阈值 |
| `defaultAoIRadius` | 默认 AoI 半径 |
| `backupPeriod` / `backupPeriodInTicks` | 备份周期（秒/tick） |
| `checkOffloadsPeriod` / `checkOffloadsPeriodInTicks` | offload 检查周期 |
| `chunkLoadingPeriod` | chunk 加载检查周期 |
| `ghostUpdateHertz` | ghost 状态更新频率 |
| `reservedTickFraction` | 每 tick 保留给系统的时间比例 |
| `offloadHysteresis` | offload 滞回阈值（避免抖动） |
| `offloadMaxPerCheck` / `ghostingMaxPerCheck` | 每次 check 最多 offload/创建的实体数 |
| `expectedMaxControllers` / `absoluteMaxControllers` | 控制器数量软/硬上限 |
| `maxPhysicsNetworkJitter` | 客户端物理校正允许的网络抖动 |

---

## 17. 备份（Backup）机制

cellapp 的 backup 与 baseapp 的 backup 不同：
- **baseapp 的 backup**：把 base 实体状态镜像到对端 baseapp（hash ring），用于 baseapp 死亡时的快速接管；
- **cellapp 的 backup**：把 real 实体的核心属性序列化后发给 **dbmgr**，作为「最后已知状态」存档，用于 cellapp 全集群故障后的恢复。

### 17.1 RealEntity 的 backup 接口

```cpp
// /workspace/src/server/cellapp/real_entity.hpp
void backup();
void autoBackup();
void writeBackupProperties( BinaryOStream & data ) const;

PY_RW_ATTRIBUTE_DECLARE( shouldAutoBackup_, shouldAutoBackup )
```

- `shouldAutoBackup_` 是 `AutoBackupAndArchive::Policy`，决定该实体是否参与自动备份；
- `backup()` 主动触发一次备份；
- `autoBackup()` 在 `Cell::backup(index, period)` 周期性调度时被调用；
- `writeBackupProperties` 把实体的「备份属性」（在 .def 中标记为 `DATABASE` 的属性）序列化到流。

### 17.2 Cell 的循环备份

```cpp
// /workspace/src/server/cellapp/cell.hpp
void backup( int index, int period );
```

`index` 是当前备份轮次的偏移量，`period` 是总轮次数。`CellApp::tickBackup()` 每 tick 调用所有 Cell 的 `backup(index, period)`，每个 Cell 只备份自己负责的 1/period 实体——这样备份负载被均匀分散到多个 tick 上，避免单帧尖峰。

### 17.3 备份链路

```
RealEntity::autoBackup
   │
   ├─ writeBackupProperties(stream)
   ├─ 通过 dbMgr_ channel 发出 backupToDB 消息
   │
   ▼
DBMgr 接收 → 写入数据库（参见 dbmgr 文档）
```

注意：cellapp 的 backup **不是**实时的，而是按 `backupPeriod` 周期性进行，存在最大 `backupPeriod` 的数据丢失窗口。

---

## 18. Game Tick 与更新调度

### 18.1 Updatable / Updatables

[updatable.hpp](file:///workspace/src/server/cellapp/updatable.hpp)：

```cpp
// /workspace/src/server/cellapp/updatable.hpp
class Updatable
{
public:
    Updatable() : removalHandle_( -1 ) {}
    virtual ~Updatable() {};

    virtual void update() = 0;

private:
    friend class Updatables;
    int removalHandle_;
};
```

[updatables.hpp](file:///workspace/src/server/cellapp/updatables.hpp)：

```cpp
// /workspace/src/server/cellapp/updatables.hpp
class Updatables
{
public:
    Updatables( int numLevels = 2 );
    bool add( Updatable * pObject, int level );
    bool remove( Updatable * pObject );
    void call();
    size_t size() const    { return objects_.size(); }

private:
    Updatables( const Updatables & );
    Updatables & operator=( const Updatables & );

    void adjustForAddedObject();

    std::vector< Updatable * > objects_;
    std::vector< int > levelSize_;
    bool inUpdate_;
    int deletedObjects_;
};
```

要点：
- **多层调度**：默认 2 层（`level=0` 与 `level=1`）。level 0 是「每 tick 必须执行」（如 Witness AoI 更新），level 1 是「可延迟执行」（如某些 controller）。`call()` 先调用所有 level 0，剩余时间再调用 level 1。
- **删除安全**：`inUpdate_` 与 `deletedObjects_` 处理「update 中删除自己」的情况。

### 18.2 Game Tick 主循环

`CellApp::handleTimeout` 在 `TIMEOUT_GAME_TICK` 时进入 `handleGameTickTimeSlice()`，主要工作：

```
1. syncTime()              // 与 CellAppMgr 同步 GameTime
2. callTimers()            // 触发 TimerController 等
3. updatables_.call()      // 调度所有 Updatable
4. checkOffloads()         // 检查 offload
5. tickBackup()            // 周期备份
6. updateBoundary()        // 更新边界（如果需要）
7. updateLoad()            // 更新负载统计
8. cellAppMgr_.informOfLoad(...)  // 上报负载
9. checkTickWarnings(...)  // tick 超时告警
10. EmergencyThrottle::update(...)  // 调整限流
```

### 18.3 Profile

[profile.hpp](file:///workspace/src/server/cellapp/profile.hpp)：

```cpp
// /workspace/src/server/cellapp/profile.hpp
extern TimeStamp g_profileInitGhostTimeLevel;
extern int       g_profileInitGhostSizeLevel;
extern TimeStamp g_profileInitRealTimeLevel;
extern int       g_profileInitRealSizeLevel;
extern TimeStamp g_profileOnloadTimeLevel;
extern int       g_profileOnloadSizeLevel;
extern int       g_profileBackupSizeLevel;

extern ProfileVal SCRIPT_CALL_PROFILE;
extern ProfileVal ON_MOVE_PROFILE;
extern ProfileVal SHUFFLE_ENTITY_PROFILE;
extern ProfileVal SHUFFLE_TRIGGERS_PROFILE;
extern ProfileVal SHUFFLE_AOI_TRIGGERS_PROFILE;
extern ProfileVal CLIENT_UPDATE_PROFILE;
extern ProfileVal TRANSIENT_LOAD_PROFILE;

namespace CellProfileGroup { void init(); }
```

可见 cellapp 关注的性能热点：
- ghost/real init 与 onload（数据反序列化）；
- 实体移动（`ON_MOVE_PROFILE`）；
- RangeList 重排（`SHUFFLE_ENTITY_PROFILE` / `SHUFFLE_TRIGGERS_PROFILE` / `SHUFFLE_AOI_TRIGGERS_PROFILE`）；
- 客户端更新打包（`CLIENT_UPDATE_PROFILE`）；
- 瞬态负载（`TRANSIENT_LOAD_PROFILE`）。

---

## 19. Python 脚本集成

### 19.1 EntityExtra：实体扩展机制

[entity_extra.hpp](file:///workspace/src/server/cellapp/entity_extra.hpp)：

```cpp
// /workspace/src/server/cellapp/entity_extra.hpp
class EntityExtra
{
    PY_FAKE_PYOBJECTPLUS_BASE_DECLARE()

public:
    EntityExtra( Entity & e ) : entity_( e ) { }
    virtual ~EntityExtra() { }

    virtual void relocated() { }

    template <class Derived> class Instance
    {
    public:
        typedef EntityExtra * (*FactoryFn)( Entity & e );

        Instance( PyDirInfo * pDI = NULL, FactoryFn ff = &construct ) :
            eeid_( Entity::registerEntityExtra( ff, pDI ) ),
            ff_( ff )
        { }

        Derived & operator()( Entity & e ) const
        {
            Derived * & rpd = (Derived*&)e.entityExtra( eeid_ );
            if (rpd == NULL) rpd = (Derived*)(*ff_)( e );
            return *rpd;
        }

        bool exists( const Entity & e ) const
        { return !!e.entityExtra( eeid_ ); }

        void clear( Entity & e ) const
        {
            EntityExtra * & rpee = e.entityExtra( eeid_ );
            delete rpee;
            rpee = NULL;
        }

        int eeid() const { return eeid_; }

    private:
        static EntityExtra * construct( Entity & e )
        { return new Derived( e ); }

        int         eeid_;
        FactoryFn   ff_;
    };

    virtual PyObject * pyGetAttribute( const char * attr );
    virtual int pySetAttribute( const char * attr, PyObject * value );

    virtual PyObject * pyAdditionalMembers( PyObject * pSeq ) { return pSeq; }
    virtual PyObject * pyAdditionalMethods( PyObject * pSeq ) { return pSeq; }

protected:
    Entity & entity()                { return entity_; }
    const Entity & entity() const    { return entity_; }

    Entity & entity_;
};
```

`EntityExtra` 是 cellapp 的「**插件式扩展机制**」：
- 每个 Entity 持有一个 `EntityExtra** extras_` 数组（按 eeid 索引）；
- 想给 Entity 加新功能，继承 `EntityExtra` 并通过 `Instance<Derived>` 模板注册；
- 通过 `instance(e)` 懒加载访问，第一次访问时构造；
- **不会被 ghost**——文档明确说「Use a controller and get it to register itself with your EntityExtra if you want to have ghosted data」。

内置 EntityExtra 包括：
- `EntityNavigate`（导航，§11）
- `EntityVision`（视觉，§10.4）
- `Passengers`（载具侧乘客集合，§12.2）
- `PassengerExtra`（乘客侧辅助，§12.3）

### 19.2 PyEntities：脚本侧的实体字典

[py_entities.hpp](file:///workspace/src/server/cellapp/py_entities.hpp)：

```cpp
// /workspace/src/server/cellapp/py_entities.hpp
class PyEntities : public PyObjectPlus
{
    Py_Header( PyEntities, PyObjectPlus )

    public:
        PyEntities( PyTypePlus * pType = &PyEntities::s_type_ );

        PyObject *          pyGetAttribute( const char * attr );
        PyObject *          subscript( PyObject * entityID );
        int                 length();

        PY_METHOD_DECLARE(py_has_key)
        PY_METHOD_DECLARE(py_keys)
        PY_METHOD_DECLARE(py_values)
        PY_METHOD_DECLARE(py_items)
        PY_METHOD_DECLARE( py_get )

        static PyObject *   s_subscript( PyObject * self, PyObject * entityID );
        static Py_ssize_t   s_length( PyObject * self );
};
```

脚本侧 `BigWorld.entities` 就是 `PyEntities` 实例，行为类似只读 dict，背后是 `EntityPopulation`。

### 19.3 Mailbox

[mailbox.hpp](file:///workspace/src/server/cellapp/mailbox.hpp) 中的层次：

```
PyEntityMailBox (entitydef)
   └─ ServerEntityMailBox (cellapp)
        ├─ CellEntityMailBox      // 发往其他 cellapp 的实体
        └─ CommonBaseEntityMailBox
             └─ BaseEntityMailBox // 发往 baseapp 的实体
```

每个 mailbox 持有 `(EntityTypePtr, Address, EntityID)`，调用方法时：
- `CellEntityMailBox`：通过 `CellApp::getChannel(addr)` 获取 channel，把方法 ID + 参数写入 bundle；
- `BaseEntityMailBox`：通过对应的 base channel 发送。

`ServerEntityMailBox::migrate()` 与 `adjustForDeadBaseApp` 是 baseapp 死亡后 mailbox 地址迁移的入口——通过 `BackupHash` 找到新的 baseapp 地址。

### 19.4 RealCaller：脚本方法代理

[real_caller.hpp](file:///workspace/src/server/cellapp/real_caller.hpp)：

```cpp
// /workspace/src/server/cellapp/real_caller.hpp
class RealCaller : public PyObjectPlus
{
    Py_Header( RealCaller, PyObjectPlus )

public:
    RealCaller( Entity * pEntity,
            const MethodDescription * pMethodDescription,
            PyTypePlus * pType = &RealCaller::s_type_ ) :
        PyObjectPlus( pType ),
        pEntity_( pEntity ),
        pMethodDescription_( pMethodDescription )
    { }

    PY_METHOD_DECLARE( pyCall )

private:
    EntityPtr pEntity_;
    const MethodDescription * pMethodDescription_;
};
```

当 ghost 实体上调用某个标记为 REAL_ONLY 的方法时，会通过 `RealCaller` 转发到 real 端。`Entity::forwardMessageToReal` 是底层实现，把消息打包到 `realChannel`。

---

## 20. 关键消息接口（CellAppInterface）

[cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp) 定义了 cellapp 接受的所有 Mercury 消息。按层次整理：

### 20.1 CellApp 级消息

| 消息 | 参数 | 用途 |
|------|------|------|
| `addCell` | SpaceID | 由 CellAppMgr 发起，给本进程分配一个 Cell |
| `startup` | baseAppAddr | 通知 baseapp 地址，进程进入运行态 |
| `setGameTime` | GameTime | 同步全局 game tick |
| `handleCellAppMgrBirth` | addr | CellAppMgr 重生通知 |
| `handleCellAppDeath` | addr | 邻居 cellapp 死亡 |
| `handleBaseAppDeath` | (stream) | 对应 baseapp 死亡 |
| `shutDown` | isSigInt | 立即停机 |
| `controlledShutDown` | ShutDownStage, shutDownTime | 分阶段停机 |
| `setBaseApp` | baseAppAddr | 设置本进程对应的 baseapp |
| `setSharedData` / `delSharedData` | (stream) | 同步 cellAppData / globalData |
| `onloadTeleportedEntity` | (stream) | teleport 接收端 onload |
| `cellAppMgrInfo` | maxCellAppLoad | 推送负载阈值 |
| `sendEntityPositions` | (variable) | 由其他 cellapp 请求实体位置 |
| `callWatcher` | (stream) | 转发 watcher 请求 |

### 20.2 Space 级消息

| 消息 | 参数 | 用途 |
|------|------|------|
| `createGhost` | (stream) | 创建一个 ghost 实体 |
| `spaceData` | entryID, key, value | 增量 SpaceData（如天气、地形） |
| `allSpaceData` | numEntries + 列表 | 全量 SpaceData（初始化时） |
| `updateGeometry` | (stream) | 更新空间几何（chunk 路径） |
| `spaceGeometryLoaded` | flags, lastGeometryPath | 通知几何已加载 |
| `shutDownSpace` | (stream) | 关闭整个 Space |

### 20.3 Cell 级消息

| 消息 | 参数 | 用途 |
|------|------|------|
| `createEntity` | (stream) | 在 cell 上创建 real 实体 |
| `createEntityNearEntity` | (stream) | 在指定实体附近创建 |
| `shouldOffload` | (stream) | CellAppMgr 控制是否 offload |
| `retireCell` | (stream) | Cell 退役阶段 1：不再接收新实体 |
| `removeCell` | (stream) | Cell 退役阶段 2：移除 |
| `notifyOfCellRemoval` | (stream) | 通知邻居 cell 即将移除 |
| `ackCellRemoval` | (stream) | 邻居对 cell removal 的 ACK |

### 20.4 Entity 级消息（REAL_ONLY）

这些消息只在 real 实体上处理：

| 消息 | 参数 | 用途 |
|------|------|------|
| `avatarUpdateImplicit` | pos, dir, refNum | 客户端 avatar 快速位置更新（隐式） |
| `avatarUpdateExplicit` | spaceID, vehicleID, pos, dir, onGround, refNum | 客户端 avatar 完整位置更新 |
| `ackPhysicsCorrection` | (无) | 客户端确认收到物理校正 |
| `enableWitness` | (variable, request) | 启用 witness（绑定客户端） |
| `witnessCapacity` | witness, bps | 设置 witness 带宽 |
| `requestEntityUpdate` | id + 事件号数组 | 客户端请求补发某些事件 |
| `witnessed` | (无) | ghost 通知 real「我被看到了」 |
| `writeToDBRequest` | (variable, request) | 请求写入 DB |
| `destroyEntity` | flags | 销毁实体 |
| `runScriptMethod` | (variable) | 调用 entity 脚本方法 |
| `callBaseMethod` | (variable) | 通过 cell mailbox 调用 base 方法 |
| `callClientMethod` | (variable) | 通过 cell mailbox 调用 client 方法 |
| `delControlledBy` | deadController | 控制实体已死 |
| `forwardedBaseEntityPacket` | (variable) | 从 ghost 转发到 real 的 base 包 |
| `onBaseOffloaded` | (variable) | base 已 offload 通知 |
| `teleport` | dstMailBoxRef | 请求 teleport |

### 20.5 Entity 级消息（GHOST_ONLY）

只在 ghost 实体上处理：

| 消息 | 参数 | 用途 |
|------|------|------|
| `onload` | (variable) | ghost 创建/恢复时的初始数据 |
| `ghostAvatarUpdate` | pos, dir, isOnGround, updateNumber | real 推送给 ghost 的位置更新 |
| `ghostHistoryEvent` | (variable) | real 推送给 ghost 的历史事件 |
| `ghostSetReal` | numTimesRealOffloaded, owner | 通知 ghost：real 已迁到 owner |
| `ghostSetNextReal` | nextRealAddr | 通知 ghost：real 即将迁到 nextRealAddr |
| `delGhost` | (无) | 删除 ghost |
| `ghostVolatileInfo` | volatileInfo | 更新 ghost 的 volatile 信息 |
| `ghostControllerCreate` / `ghostControllerDelete` / `ghostControllerUpdate` | (variable) | ghost 上的 controller 同步 |
| `ghostedDataUpdate` | eventNumber + data | real → ghost 的属性变更 |
| `checkGhostWitnessed` | (无) | real 查询 ghost 是否被 witness |
| `aoiUpdateSchemeChange` | scheme | 修改 ghost 的 AoI 更新方案 |

### 20.6 通用 Entity 消息

| 消息 | 用途 |
|------|------|
| `runExposedMethod` | 调用 .def 中暴露的 cell 方法（128~254 段） |

### 20.7 EntityReality 与消息路由

[cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp) 中的 `EntityReality` 枚举：

```cpp
// /workspace/src/server/cellapp/cellapp_interface.hpp
enum EntityReality
{
    GHOST_ONLY,
    REAL_ONLY,
    WITNESS_ONLY
};
```

`MF_BEGIN_ENTITY_MSG` / `MF_VARLEN_ENTITY_MSG` 等宏配合 `EntityReality`，由 [entity_message_handler.hpp](file:///workspace/src/server/cellapp/entity_message_handler.hpp) 中的 `EntityMessageHandler` 在 `handleMessage` 中检查目标实体的 real/ghost 状态——若状态不匹配，消息会被丢弃或转发到 real 端：

```cpp
// /workspace/src/server/cellapp/entity_message_handler.hpp
class EntityMessageHandler : public Mercury::InputMessageHandler
{
public:
    void handleMessage( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data, EntityID entityID );

protected:
    EntityMessageHandler( EntityReality reality,
                bool shouldBufferIfTickPending = false );

    virtual void handleMessage( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data );

    virtual void callHandler( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data, Entity * pEntity ) = 0;

    virtual void sendFailure( const Mercury::Address & srcAddr,
            Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data, EntityID entityID );

    EntityReality reality_;
    bool shouldBufferIfTickPending_;
};
```

`shouldBufferIfTickPending_` 与 `BufferedEntityMessages`（§13.5）配合实现 tick 期间的消息缓冲。

---

## 21. 典型调用流（ASCII 时序图）

### 21.1 cellapp 启动 → 接收第一个 Cell

```
CellApp          CellAppMgr            BaseApp
  │                  │                    │
  │  bwMainT<CellApp>│                    │
  │────init()────────▶                    │
  │                  │                    │
  │  AddToCellAppMgrHelper::send()         │
  │───add(addr,viewerPort,reply)──────────▶│
  │                  │                    │
  │                  │ 分配 Cell 矩形      │
  │                  │ 选定 BaseApp       │
  │                  │                    │
  │◀────reply(CellAppInitData)─────────────│
  │                  │                    │
  │  finishInit(initData)                  │
  │  startGameTime()                       │
  │  hasStarted() = true                   │
  │                  │                    │
  │◀───addCell(spaceID)─────────────────── │
  │                  │                    │
  │  Cells::add(new Cell)                  │
  │  Space::pCell(cell)                    │
  │                  │                    │
  │◀───startup(baseAppAddr)─────────────── │
  │                  │                    │
  │  baseAppAddr_ = baseAppAddr            │
  │                  │                    │
  │  setBaseApp 通知 → 后续 base 通信走该地址│
  │                  │                    │
  │  onGetFirstCell(isFromDB)              │
  │  (若是新进程从 DB 恢复，触发 restoreEntity)│
```

### 21.2 客户端 avatar 移动 → AoI 更新 → 客户端推送

```
Client          BaseApp         CellApp(Real)        CellApp(Ghost邻居)
  │                │                  │                    │
  │  avatarUpdateImplicit              │                    │
  │──pos/dir/refNum──▶                 │                    │
  │                │                  │                    │
  │                │  forwardToCell   │                    │
  │                │──avatarUpdateImplicit──▶               │
  │                │                  │                    │
  │                │     Entity::avatarUpdateImplicit       │
  │                │     ├─ setVolatileInfo                 │
  │                │     ├─ updateGlobalPosition            │
  │                │     │   ├─ shuffle RangeList           │
  │                │     │   │   ├─ 跨越 AoITrigger         │
  │                │     │   │   │   → Witness::addToAoI    │
  │                │     │   │   │   → Witness::removeFromAoI│
  │                │     │   │   └─ 跨越 ProximityTrigger    │
  │                │     │   │       → onEnterTrap/onLeaveTrap│
  │                │     │   ├─ checkChunkCrossing          │
  │                │     │   └─ ghost 同步                  │
  │                │     │       ├─ ghostAvatarUpdate ─────▶│
  │                │     │       └─ (更新所有 haunts)        │
  │                │     ├─ RealEntity::newPosition         │
  │                │     │   └─ Witness::newPosition        │
  │                │     │       └─ 重新计算 AoI 优先级     │
  │                │     └─ Witness::update (下一 tick)      │
  │                │         ├─ 遍历 aoiMap_                 │
  │                │         ├─ updatePriority              │
  │                │         ├─ updateDetailLevel           │
  │                │         ├─ addChangedProperties        │
  │                │         └─ sendToClient                │
  │                │                  │                    │
  │                │◀──clientEntityUpdate──│               │
  │◀──createEntity/leaveAoI/etc───│        │               │
```

### 21.3 Real 实体 offload 到邻居 cellapp

```
CellApp A (Real)              CellApp B (即将成为 Real)        其他 Ghost 邻居
   │                                │                              │
   │  Cell::checkOffloadsAndGhosts  │                              │
   │  shouldOffload=true            │                              │
   │  选定 B 作为目标                │                              │
   │                                │                              │
   │  Entity::offload(B-channel)    │                              │
   │  ├─ writeRealDataToStream      │                              │
   │  │   ├─ volatileInfo           │                              │
   │  │   ├─ 控制器状态              │                              │
   │  │   ├─ witness 数据（若有）    │                              │
   │  │   └─ numTimesRealOffloaded+1│                              │
   │  ├─ ghostSetNextReal(B) ───────────────────────────────────────▶
   │  │   (让 ghost 知道下一段子序列来自 B)                          │
   │  ├─ createEntity 消息 ─────▶   │                              │
   │  │   (带完整 real 数据)         │                              │
   │  └─ convertRealToGhost         │                              │
   │      └─ pReal_ = nullptr       │                              │
   │      └─ pRealChannel_ = B      │                              │
   │      └─ nextRealAddr_ = B      │                              │
   │                                │                              │
   │                                │  Entity::onload             │
   │                                │  ├─ convertGhostToReal      │
   │                                │  │   └─ pReal_ = new RealEntity│
   │                                │  ├─ 重建 haunts              │
   │                                │  ├─ ghostSetReal(B, numOffloaded) ──▶
   │                                │  │   (告诉 ghost：现在 B 是 real)│
   │                                │  └─ onBaseOffloaded ──▶ BaseApp│
   │                                │      (更新 base 的 cell 地址)   │
   │                                │                              │
   │                                │  BufferedGhostMessages::     │
   │                                │    playSubsequenceFor(...)   │
   │                                │  (回放被缓冲的 B 子序列)       │
```

### 21.4 邻居 cellapp 死亡 → 恢复

```
CellAppMgr                CellApp(本进程)              BaseApp
   │                            │                          │
   │  检测到 cellapp X 死亡       │                          │
   │                            │                          │
   │───handleCellAppDeath(X)────▶│                          │
   │                            │                          │
   │                            │  CellAppDeathListeners::  │
   │                            │    handleCellAppDeath(X)  │
   │                            │  ├─ EntityPopulation::    │
   │                            │    handleCellAppDeath     │
   │                            │    ├─ 清理 currChannels_/  │
   │                            │    │  prevChannels_ 中 X  │
   │                            │    │  的记录               │
   │                            │    └─ notifyBasesOfCellAppDeath│
   │                            │        (通知 base：X 上的   │
   │                            │         real 已失效)       │
   │                            │        ├─ 对每个 dead real：│
   │                            │        │   触发 base restore│
   │                            │        └─ AckCellAppDeathHelper│
   │                            │            记录 numBadGhosts│
   │                            │                          │
   │                            │  AckCellAppDeathHelper:: │
   │                            │    startTimer()           │
   │                            │  (轮询检查所有 critical    │
   │                            │   channel 完成)            │
   │                            │                          │
   │                            │  checkFinished() (周期)   │
   │                            │  ├─ recentOnloads_ 已清空  │
   │                            │  └─ numBadGhosts_ 已 ACK   │
   │                            │                          │
   │◀────ackCellAppDeath(X)─────│                          │
   │                            │                          │
   │  (CellAppMgr 收到所有 ACK 后│                          │
   │   才认为恢复完成，避免过早   │                          │
   │   触发 base restore)        │                          │
```

### 21.5 Cell 退役四阶段

```
CellAppMgr                  CellApp                 邻居 CellApps
   │                            │                          │
   │  (负载迁移完成，准备移除 Cell)│                          │
   │───retireCell──────────────▶│                          │
   │                            │  Cell::retireCell         │
   │                            │  ├─ isRetiring_=true      │
   │                            │  └─ shouldOffload_=false  │
   │                            │     (不再接收新 real)      │
   │                            │                          │
   │  (等待所有 real 迁出)        │                          │
   │                            │                          │
   │───removeCell──────────────▶│                          │
   │                            │  Cell::removeCell         │
   │                            │  ├─ 通知 Space 该 cell 即将│
   │                            │  │  下线                   │
   │                            │  └─ notifyOfCellRemoval ─▶│
   │                            │      (告诉邻居：我即将消失)│
   │                            │                          │
   │                            │◀────ackCellRemoval────────│
   │                            │  (邻居确认收到，已迁移引用)│
   │                            │                          │
   │                            │  pendingAcks_ 收齐         │
   │                            │  → 真正销毁 Cell           │
   │                            │  → space_.pCell(nullptr)  │
   │                            │  → isRemoved_=true        │
```

---

## 22. 配置项一览

下表汇总 [cellapp_config.hpp](file:///workspace/src/server/cellapp/cellapp_config.hpp)、[throttle_config.hpp](file:///workspace/src/server/cellapp/throttle_config.hpp)、`id_config.hpp`、`noise_config.hpp` 中的关键 ServerAppOption 配置项（具体默认值参见 `bw.xml` 与各 .cpp 文件）：

### 22.1 CellAppConfig

| 配置项 | 类型 | 含义 |
|--------|------|------|
| `loadSmoothingBias` | float | persistent load 指数平滑系数 |
| `ghostDistance` | float | 创建 ghost 的距离阈值 |
| `defaultAoIRadius` | float | 默认 AoI 半径 |
| `fastShutdown` | bool | 是否启用快速停机 |
| `shouldResolveMailBoxes` | bool | 是否解析 mailbox（确保目标存在） |
| `entitySpamSize` | int | 测试用批量创建实体数 |
| `maxGhostsToDelete` | int | 单 tick 最多删除的 ghost 数 |
| `minGhostLifespan` | float | ghost 最短存活时间（秒） |
| `minGhostLifespanInTicks` | int | ghost 最短存活时间（tick） |
| `loadDominantTextureMaps` | bool | 是否加载主导纹理 |
| `backupPeriod` / `backupPeriodInTicks` | float/int | 备份周期 |
| `checkOffloadsPeriod` / `checkOffloadsPeriodInTicks` | float/int | offload 检查周期 |
| `chunkLoadingPeriod` | float | chunk 加载检查周期 |
| `ghostUpdateHertz` | int | ghost 状态更新频率 |
| `reservedTickFraction` | double | 每 tick 保留给系统的时间比例 |
| `obstacleTreeDepth` | int | 障碍树深度 |
| `sendWindowCallbackThreshold` | float | send window 溢出回调阈值 |
| `offloadHysteresis` | float | offload 滞回阈值 |
| `offloadMaxPerCheck` | int | 每次 check 最多 offload 实体数 |
| `ghostingMaxPerCheck` | int | 每次 check 最多创建 ghost 数 |
| `expectedMaxControllers` | int | 控制器数量软上限 |
| `absoluteMaxControllers` | int | 控制器数量硬上限 |
| `enforceGhostDecorators` | bool | 是否强制 ghost 装饰器 |
| `treatAllOtherEntitiesAsGhosts` | bool | 调试用：把所有其他实体当 ghost |
| `maxPhysicsNetworkJitter` | float | 物理校正允许的网络抖动 |

### 22.2 ThrottleConfig

| 配置项 | 类型 | 含义 |
|--------|------|------|
| `behindThreshold` | float | 落后多少秒触发限流 |
| `spareTimeThreshold` | float | 剩余时间低于此值触发限流 |
| `scaleForwardTime` | float | 加速时间常数 |
| `scaleBackTime` | float | 减速时间常数 |
| `min` | float | throttle 最小值（下限） |

---

## 23. 注意事项与未覆盖项

### 23.1 实现内联说明

cellapp 目录大量使用 `.hpp` + `.ipp` + 部分内联的实现模式。本文档主要依据 `.hpp` 推断行为，部分细节（如 `Entity::updateGlobalPosition` 的具体逻辑、`Witness::update` 的优先级排序算法）需要参考对应的 `.cpp` 文件，例如：
- `entity.cpp`：实体位置更新、chunk 跨越、物理校验；
- `witness.cpp`：AoI 列表维护、IDAlias 分配、客户端打包；
- `cell.cpp`：offload/onload、cell 退役；
- `real_entity.cpp`：real 状态机、haunt 管理、备份。

### 23.2 未深入展开的模块

以下模块在文档中只点到为止，深入分析请参见对应文件：
- **chunk 系统**：`server_geometry_mapping.hpp` / `loading_edge.hpp` / `loading_column.hpp` 描述 chunk 异步加载与四边边界推进；底层 `chunk/` 模块（不属于 cellapp）。
- **waypoint 导航**：`waypoint/navigator.hpp`、`waypoint/navloc.hpp`（不在 cellapp 目录）。
- **ServerViewer**：`cell_viewer_server.hpp` / `cell_viewer_connection.hpp` 提供调试用 viewer 接入。
- **DocWatcher / Profile**：性能与调试钩子。
- **SharedData**：`pCellAppData_` / `pGlobalData_` 是跨 cellapp 共享的 key-value 字典，由 CellAppMgr 同步。
- **IDClient**：与 DBMgr 协作分配全局唯一 EntityID。

### 23.3 与 baseapp 的协作要点

- cellapp 的 base 通信地址 `baseAppAddr_` 由 CellAppMgr 通过 `setBaseApp` 设置；
- 每个 real 实体的 `baseAddr_` 记录其对应 base 的地址，baseapp 死亡时通过 `adjustForDeadBaseApp(backupHash)` 重新路由；
- `informBaseOfAddress` 是 real 主动告知 base「我现在在哪个 cell」的入口，offload 后必须调用。

### 23.4 与 dbmgr 的协作要点

- cellapp 通过 `dbMgr_`（匿名 channel）与 DBMgr 通信；
- 主要消息：`writeToDBRequest`（写库）、`backupToDB`（备份）、ID 分配请求；
- 备份是**周期性**而非实时的，存在数据丢失窗口。

### 23.5 不确定之处

- `Entity::callback` 系列的「缓冲回放」精确顺序，需参见 `entity.cpp` 中 `s_callbacksBuffer_` 的使用；
- `Witness::calculateReferencePosition` 的具体算法（如何从客户端 refNum 推导参考位置）参见 `witness.cpp`；
- `AckCellAppDeathHelper::checkFinished` 的具体判定条件（哪些 channel 算 critical）参见 `ack_cell_app_death_helper.cpp`；
- `CellAppChannels` 的 `sendAll` 周期与 `CellApp` 的 game tick 周期的关系，参见 `cellapp.cpp` 中的初始化代码。

---

## 附录：核心文件清单

| 文件 | 行数（约） | 主要内容 |
|------|-----------|---------|
| [cellapp.hpp](file:///workspace/src/server/cellapp/cellapp.hpp) | ~330 | CellApp 单例声明 |
| [cellapp.ipp](file:///workspace/src/server/cellapp/cellapp.ipp) | - | CellApp 内联方法 |
| [cell.hpp](file:///workspace/src/server/cellapp/cell.hpp) | ~210 | Cell 类与 Entities 内嵌类 |
| [space.hpp](file:///workspace/src/server/cellapp/space.hpp) | ~225 | Space 类 |
| [spaces.hpp](file:///workspace/src/server/cellapp/spaces.hpp) | ~45 | Space 集合 |
| [cell_info.hpp](file:///workspace/src/server/cellapp/cell_info.hpp) | ~85 | BSP 叶子节点 |
| [entity.hpp](file:///workspace/src/server/cellapp/entity.hpp) | ~805 | Entity 类（最大） |
| [real_entity.hpp](file:///workspace/src/server/cellapp/real_entity.hpp) | ~275 | RealEntity + Haunt |
| [witness.hpp](file:///workspace/src/server/cellapp/witness.hpp) | ~250 | Witness + AoI |
| [witness.ipp](file:///workspace/src/server/cellapp/witness.ipp) | ~37 | WitnessMethod 内联 |
| [entity_cache.hpp](file:///workspace/src/server/cellapp/entity_cache.hpp) | ~230 | EntityCache + EntityCacheMap |
| [aoi_update_schemes.hpp](file:///workspace/src/server/cellapp/aoi_update_schemes.hpp) | ~70 | AoI 更新方案 |
| [range_list_node.hpp](file:///workspace/src/server/cellapp/range_list_node.hpp) | ~250 | RangeList 节点与触发器 |
| [cell_range_list.hpp](file:///workspace/src/server/cellapp/cell_range_list.hpp) | ~45 | RangeList 容器 |
| [controller.hpp](file:///workspace/src/server/cellapp/controller.hpp) | ~235 | Controller 基类 + 注册宏 |
| [controllers.hpp](file:///workspace/src/server/cellapp/controllers.hpp) | ~60 | Controllers 集合 |
| [move_controller.hpp](file:///workspace/src/server/cellapp/move_controller.hpp) | ~108 | MoveController 系列 |
| [navigation_controller.hpp](file:///workspace/src/server/cellapp/navigation_controller.hpp) | ~70 | NavigationController |
| [base_acceleration_controller.hpp](file:///workspace/src/server/cellapp/base_acceleration_controller.hpp) | ~73 | 加速运动基类 |
| [proximity_controller.hpp](file:///workspace/src/server/cellapp/proximity_controller.hpp) | ~57 | 陷阱触发 |
| [vision_controller.hpp](file:///workspace/src/server/cellapp/vision_controller.hpp) | ~66 | 视野 |
| [visibility_controller.hpp](file:///workspace/src/server/cellapp/visibility_controller.hpp) | ~50 | 可见性 |
| [scan_vision_controller.hpp](file:///workspace/src/server/cellapp/scan_vision_controller.hpp) | ~45 | 扫描视野 |
| [timer_controller.hpp](file:///workspace/src/server/cellapp/timer_controller.hpp) | ~73 | 定时器 |
| [passenger_controller.hpp](file:///workspace/src/server/cellapp/passenger_controller.hpp) | ~56 | 乘客控制器 |
| [passengers.hpp](file:///workspace/src/server/cellapp/passengers.hpp) | ~52 | 车辆侧乘客集合 |
| [passenger_extra.hpp](file:///workspace/src/server/cellapp/passenger_extra.hpp) | ~62 | 乘客侧扩展 |
| [entity_extra.hpp](file:///workspace/src/server/cellapp/entity_extra.hpp) | ~195 | EntityExtra 基类 |
| [entity_vision.hpp](file:///workspace/src/server/cellapp/entity_vision.hpp) | ~145 | 视觉 EntityExtra |
| [entity_navigate.hpp](file:///workspace/src/server/cellapp/entity_navigate.hpp) | ~155 | 导航 EntityExtra |
| [entity_population.hpp](file:///workspace/src/server/cellapp/entity_population.hpp) | ~105 | 实体人口 + 死亡监听 |
| [cellapp_death_listener.hpp](file:///workspace/src/server/cellapp/cellapp_death_listener.hpp) | ~67 | 死亡监听基类 |
| [ack_cell_app_death_helper.hpp](file:///workspace/src/server/cellapp/ack_cell_app_death_helper.hpp) | ~120 | 死亡恢复确认 |
| [cell_app_channel.hpp](file:///workspace/src/server/cellapp/cell_app_channel.hpp) | ~46 | cellapp 间 channel |
| [cell_app_channels.hpp](file:///workspace/src/server/cellapp/cell_app_channels.hpp) | ~66 | channel 集合 + 定时发送 |
| [cellappmgr_gateway.hpp](file:///workspace/src/server/cellapp/cellappmgr_gateway.hpp) | ~65 | CellAppMgr 网关 |
| [add_to_cellappmgr_helper.hpp](file:///workspace/src/server/cellapp/add_to_cellappmgr_helper.hpp) | ~55 | 启动握手 |
| [cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp) | ~335 | 全部 Mercury 消息定义 |
| [entity_message_handler.hpp](file:///workspace/src/server/cellapp/entity_message_handler.hpp) | ~55 | 实体消息路由基类 |
| [create_entity_near_entity_handler.hpp](file:///workspace/src/server/cellapp/create_entity_near_entity_handler.hpp) | ~62 | createEntityNearEntity 处理器 |
| [buffered_ghost_message.hpp](file:///workspace/src/server/cellapp/buffered_ghost_message.hpp) | ~36 | 单条 ghost 缓冲消息 |
| [buffered_ghost_messages.hpp](file:///workspace/src/server/cellapp/buffered_ghost_messages.hpp) | ~68 | ghost 消息重排器 |
| [buffered_entity_messages.hpp](file:///workspace/src/server/cellapp/buffered_entity_messages.hpp) | ~52 | 实体消息缓冲 |
| [history_event.hpp](file:///workspace/src/server/cellapp/history_event.hpp) | ~127 | 历史事件 |
| [mailbox.hpp](file:///workspace/src/server/cellapp/mailbox.hpp) | ~172 | Cell/Base mailbox |
| [real_caller.hpp](file:///workspace/src/server/cellapp/real_caller.hpp) | ~47 | real 方法代理 |
| [updatable.hpp](file:///workspace/src/server/cellapp/updatable.hpp) | ~36 | 可更新接口 |
| [updatables.hpp](file:///workspace/src/server/cellapp/updatables.hpp) | ~42 | 多层调度器 |
| [emergency_throttle.hpp](file:///workspace/src/server/cellapp/emergency_throttle.hpp) | ~44 | 紧急限流 |
| [throttle_config.hpp](file:///workspace/src/server/cellapp/throttle_config.hpp) | ~33 | 限流配置 |
| [cellapp_config.hpp](file:///workspace/src/server/cellapp/cellapp_config.hpp) | ~67 | cellapp 配置 |
| [profile.hpp](file:///workspace/src/server/cellapp/profile.hpp) | ~46 | 性能 profile |
| [py_entities.hpp](file:///workspace/src/server/cellapp/py_entities.hpp) | ~63 | PyEntities 脚本接口 |
| [server_geometry_mapping.hpp](file:///workspace/src/server/cellapp/server_geometry_mapping.hpp) | ~78 | chunk 加载映射 |
| [main.cpp](file:///workspace/src/server/cellapp/main.cpp) | ~22 | 进程入口 |

---

文档到此结束。本文基于 cellapp 目录下 50+ 个 `.hpp/.cpp` 文件，覆盖了从进程启动、空间划分、实体管理、real/ghost 迁移、AOI/Witness、Controller、导航、车辆、消息缓冲、跨进程通信、死亡恢复、负载均衡、备份、tick 调度、Python 集成、消息接口到典型调用流的全景。如需进一步深入某模块，请按「未覆盖项」与「不确定之处」中的指引查阅对应实现文件。