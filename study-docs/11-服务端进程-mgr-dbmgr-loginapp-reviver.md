# 11 服务端进程 — mgr / dbmgr / loginapp / reviver

> 仓库版本：BigWorld Technology 2.0.1
> 源码目录：
> - [src/server/baseappmgr/](file:///workspace/src/server/baseappmgr/)
> - [src/server/cellappmgr/](file:///workspace/src/server/cellappmgr/)
> - [src/server/dbmgr/](file:///workspace/src/server/dbmgr/)
> - [src/server/loginapp/](file:///workspace/src/server/loginapp/)
> - [src/server/reviver/](file:///workspace/src/server/reviver/)
> 关联文档：[00-架构总览.md](file:///workspace/study-docs/00-架构总览.md)、[08-服务端框架-server.md](file:///workspace/study-docs/08-服务端框架-server.md)、[09-服务端进程-baseapp.md](file:///workspace/study-docs/09-服务端进程-baseapp.md)、[10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)

---

## 1. 总览：四个"控制面"进程的角色

BigWorld 服务端把"业务进程"（`baseapp` / `cellapp`，承载玩家与空间逻辑）和"控制进程"严格分离。本文档聚焦后者，共四类：

| 进程 | 单/多实例 | 角色 | 关键源码 |
| --- | --- | --- | --- |
| **baseappmgr** | 单实例 | baseapp 集群的总控：注册/退服、负载均衡、备份哈希环、createBase 路由、recovery | [baseappmgr.hpp](file:///workspace/src/server/baseappmgr/baseappmgr.hpp) |
| **cellappmgr** | 单实例 | cellapp 集群的总控：注册/退服、空间划分、cell 路由、负载、game time 推进 | [cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp) |
| **dbmgr** | 单实例 | 持久化总控：MySQL/XML 主库、实体存取、登录鉴权、二级库合并、auto-load | [database.hpp](file:///workspace/src/server/dbmgr/database.hpp) |
| **loginapp** | 多实例（可水平扩展） | 客户端登录入口：外网接口、登录限流、把登录请求转发到 dbmgr | [loginapp.hpp](file:///workspace/src/server/loginapp/loginapp.hpp) |
| **reviver** | 每机器一个 | 看门狗：监控 cellappmgr/baseappmgr/dbmgr/loginapp 死亡并自动重启 | [reviver.hpp](file:///workspace/src/server/reviver/reviver.hpp) |

> **关键洞察**：baseappmgr 与 cellappmgr 在源码层都继承自 [ManagerApp](file:///workspace/src/lib/server/manager_app.hpp)（即 `ServerApp` 的薄封装），它们本身**不持有任何业务实体**，只持有到子进程的 `ChannelOwner` 列表与若干全局状态。真正承载实体的是 baseapp/cellapp，而控制面只做"调度、路由、容灾"。

### 1.1 进程拓扑（控制面视角）

```
                                  ┌────────────────────────┐
                                  │      bwmachined         │  每机器常驻守护
                                  │  (进程注册/出生死亡广播) │  (Mercury MachineDaemon)
                                  └──────┬──────────────────┘
                                         │ birth/death 多播
            ┌────────────────────────────┼────────────────────────────┐
            │                            │                            │
            ▼                            ▼                            ▼
   ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
   │   loginapp #1   │          │   loginapp #N   │   ...    │   reviver       │
   │  (外网入口)      │          │  (外网入口)      │          │ (看门狗)         │
   └────────┬────────┘          └────────┬────────┘          └────────┬────────┘
            │ DBInterface::logOn          │                            │ handleXXXBirth
            │ (匿名 channel)              │                            │ handleXXXDeath
            ▼                             ▼                            │ reviverPing
   ┌────────────────────────────────────────────────────────┐          │
   │                        dbmgr                           │◀─────────┘
   │  IDatabase(MySQL/XML) + BillingSystem + Consolidator   │
   │  + EntityAutoLoader                                    │
   └──────────────┬─────────────────────────┬───────────────┘
                  │ BaseAppMgrInterface::   │ CellAppMgrInterface::
                  │ initData/startup/       │ prepareForRestoreFromDB/
                  │ createEntity 回路        │ startup
                  ▼                          ▼
        ┌──────────────────┐          ┌──────────────────┐
        │   baseappmgr     │◀────────▶│   cellappmgr     │
        │ (BaseApps map +  │ setBaseApp│ (CellApps +     │
        │  BackupHashChain)│ handleBaseAppDeath│ Spaces)  │
        └────────┬─────────┘          └────────┬─────────┘
                 │ add/del/informOfLoad        │ addApp/delApp/informOfLoad
                 │ useNewBackupHash            │ updateBounds/retireApp
                 ▼                             ▼
        ┌──────────────────┐          ┌──────────────────┐
        │ baseapp #1..#N   │          │ cellapp #1..#N   │
        └──────────────────┘          └──────────────────┘
```

### 1.2 与 09/10 文档的衔接

- [09-服务端进程-baseapp.md](file:///workspace/study-docs/09-服务端进程-baseapp.md) 详细描述了 baseapp 单体；本文聚焦 baseapp 启动时如何向 baseappmgr 注册（`BaseAppMgrInterface::add`）、如何被路由（`createEntity`）、崩溃后如何被恢复（`handleBaseAppDeath` + `BackupHashChain`）。
- [10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md) 描述 cellapp 单体；本文聚焦 cellappmgr 如何把"创建 cell 实体"请求路由到具体 cellapp（`createEntity` / `createEntityInNewSpace`）。
- 登录全链路（客户端 → loginapp → dbmgr → baseappmgr → baseapp → cellapp）的端到端时序在本文第 6 节给出。

### 1.3 源码目录速览

| 目录 | 主要 .hpp | 主要 .cpp | 说明 |
| --- | --- | --- | --- |
| baseappmgr/ | baseappmgr.hpp, baseapp.hpp, baseappmgr_interface.hpp, baseappmgr_config.hpp, reply_handlers.hpp, watcher_forwarding_baseapp.hpp | baseappmgr.cpp, baseapp.cpp, message_handlers.cpp, reply_handlers.cpp | baseappmgr 全部实现都在 |
| cellappmgr/ | cellappmgr_interface.hpp | cellappmgr_interface.cpp | **本仓库只有 interface 定义，CellAppMgr 类的实现未随源码开放**（见第 3 节说明） |
| dbmgr/ | database.hpp, db_interface.hpp, dbmgr_config.hpp, consolidator.hpp, login_handler.hpp, get_entity_handler.hpp, write_entity_handler.hpp, relogon_attempt_handler.hpp, load_entity_handler.hpp, look_up_entity_handler.hpp, look_up_dbid_handler.hpp, delete_entity_handler.hpp, entity_auto_loader.hpp, custom.hpp, py_billing_system.hpp | database.cpp, consolidator.cpp, login_handler.cpp, …, main.cpp | dbmgr 是除 baseapp/cellapp 外最重的一个进程 |
| loginapp/ | loginapp.hpp, loginapp_config.hpp, login_int_interface.hpp | (实现见 loginapp.cpp，本仓库以 .hpp 为主) | 外网入口进程 |
| reviver/ | reviver.hpp, reviver_config.hpp, reviver_interface.hpp, component_reviver.hpp | reviver.cpp, component_reviver.cpp, reviver_config.cpp | 看门狗 |

> 注意：和 baseapp/cellapp 一样，很多目录只放 `.hpp`，真正的实现 `.cpp` 由 BigWorld 编译系统产出 `Hybrid64/*.o`。本文以可读源码为准，必要时点出"参见 xxx.hpp"。

---

## 2. baseappmgr — baseapp 集群总控

### 2.1 模块定位

`baseappmgr` 是单例进程，是整个 baseapp 集群的"大脑"。它不持有任何 `Base` 实体，但维护：

1. **baseapp 注册表** `BaseApps baseApps_`（`Address → shared_ptr<BaseApp>`），每个 `BaseApp` 是对远端 baseapp 进程的 `ChannelOwner` 封装 + 元信息（id、load、numBases、numProxies、backupHash、newBackupHash、isRetiring/isOffloading）。
2. **备份哈希环** `std::auto_ptr<BackupHashChain> pBackupHashChain_`，记录历史死亡 baseapp 的备份哈希，用于在 baseapp 崩溃后把 mailbox 重定向到正确的备份。
3. **共享数据** `sharedBaseAppData_`（baseapp 集群共享）与 `sharedGlobalData_`（来自 cellappmgr 的全局共享），用于跨进程的 key-value 状态同步。
4. **全局 base 字典** `GlobalBases globalBases_`，把"按名字注册的全局 base"（如 `BigWorld.createBaseFromDB` 创建的全局服务实体）映射到 `EntityMailBoxRef`。
5. **TimeKeeper** `pTimeKeeper_`，与 cellappmgr 协同推进 game time（`TIMEOUT_GAME_TICK` 每个 tick 自增 `time_`）。
6. **到 cellappmgr / dbmgr 的 channel**：`cellAppMgr_`（`ChannelOwner`）与 `dbMgr_`（`AnonymousChannelClient`）。

### 2.2 核心类：BaseAppMgr

声明见 [baseappmgr.hpp](file:///workspace/src/server/baseappmgr/baseappmgr.hpp)：

```cpp
class BaseAppMgr : public ManagerApp,
    public TimerHandler,
    public Singleton< BaseAppMgr >
{
public:
    MANAGER_APP_HEADER( BaseAppMgr, baseAppMgr )
    typedef BaseAppMgrConfig Config;

    BaseAppMgr( Mercury::EventDispatcher & mainDispatcher,
            Mercury::NetworkInterface & interface );

    // ---- baseapp 生命周期 ----
    void add( const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            const BaseAppMgrInterface::addArgs & args );
    void recoverBaseApp( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void del( const Mercury::Address & addr,
        const Mercury::UnpackedMessageHeader & header,
        const BaseAppMgrInterface::delArgs & args );
    void informOfLoad( const Mercury::Address & addr,
        const Mercury::UnpackedMessageHeader & header,
        const BaseAppMgrInterface::informOfLoadArgs & args );

    // ---- 实体创建路由 ----
    void createEntity( const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );

    // ---- 受控停机 ----
    void shutDown( bool shutDownOthers );
    void controlledShutDown(
            const BaseAppMgrInterface::controlledShutDownArgs & args );
    void controlledShutDownServer();
    void startAsyncShutDownStage( ShutDownStage stage );

    // ---- 死亡处理 ----
    bool onBaseAppDeath( const Mercury::Address & addr );
    void handleBaseAppDeath( const Mercury::Address & addr );
    void handleCellAppDeath( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );

    // ---- 备份哈希 ----
    void useNewBackupHash( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void informOfArchiveComplete( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void requestBackupHashChain( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );

    BaseApp * findBestBaseApp() const;
    void onBaseAppRetire( BaseApp & baseApp );

private:
    virtual bool init( int argc, char* argv[] );
    virtual void handleTimeout( TimerHandle handle, void * arg );

    enum AdjustBackupLocationsOp
    {
        ADJUST_BACKUP_LOCATIONS_OP_ADD,
        ADJUST_BACKUP_LOCATIONS_OP_RETIRE,
        ADJUST_BACKUP_LOCATIONS_OP_CRASH
    };

    void checkForDeadBaseApps();
    void adjustBackupLocations( const Mercury::Address & addr,
            AdjustBackupLocationsOp op );
    void updateCreateBaseInfo();
    void updateBestBaseApp();
    bool calculateOverloaded( bool baseAppsOverloaded );

    CellAppMgr                cellAppMgr_;        // 到 cellappmgr 的 channel
    bool                      cellAppMgrReady_;
    AnonymousChannelClient    dbMgr_;             // 到 dbmgr 的匿名 channel

    BaseApps                  baseApps_;          // Address → BaseAppPtr
    std::auto_ptr< BackupHashChain > pBackupHashChain_;

    SharedData                sharedBaseAppData_; // 权威副本
    SharedData                sharedGlobalData_;  // 来自 cellappmgr

    BaseAppID                 lastBaseAppID_;
    bool                      allowNewBaseApps_;
    GlobalBases               globalBases_;
    TimeKeeper *              pTimeKeeper_;

    float                     baseAppOverloadLevel_;
    Mercury::Address          bestBaseAppAddr_;
    bool                      isRecovery_;
    bool                      hasInitData_;
    bool                      hasStarted_;
    bool                      shouldShutDownOthers_;
    Mercury::Address          deadBaseAppAddr_;
    unsigned int              archiveCompleteMsgCounter_;
    GameTime                  shutDownTime_;
    ShutDownStage             shutDownStage_;
    uint64                    baseAppOverloadStartTime_;
    int                       loginsSinceOverload_;
    bool                      hasMultipleBaseAppMachines_;
    TimerHandle               gameTimer_;
};
```

`MANAGER_APP_HEADER` 宏（[manager_app.hpp](file:///workspace/src/lib/server/manager_app.hpp)）等价于 `SERVER_APP_HEADER`，提供 `appName()`/`configPath()` 静态接口，让 `bwMainT<BaseAppMgr>` 模板能找到进程名 `"BaseAppMgr"` 与配置前缀 `"baseAppMgr/"`。

### 2.3 BaseApp：远端 baseapp 的本地句柄

[baseapp.hpp](file:///workspace/src/server/baseappmgr/baseapp.hpp) 定义了 baseappmgr 侧的 `BaseApp` 类（注意：**与 [src/server/baseapp/baseapp.hpp](file:///workspace/src/server/baseapp/baseapp.hpp) 里那个 `BaseApp` 单例不是同一个类**，后者是 baseapp 进程内的核心类，前者是 baseappmgr 内代表一个远端 baseapp 的句柄）：

```cpp
class BaseApp: public Mercury::ChannelOwner
{
public:
    BaseApp( BaseAppMgr & baseAppMgr, const Mercury::Address & intAddr,
            const Mercury::Address & extAddr, int id );

    float load() const { return load_; }
    void updateLoad( float load, int numBases, int numProxies );
    bool hasTimedOut( uint64 currTime, uint64 timeoutPeriod,
           uint64 timeSinceHeardAny ) const;

    const Mercury::Address & externalAddr() const { return externalAddr_; }
    int numBases() const    { return numBases_; }
    int numProxies() const  { return numProxies_; }
    BaseAppID id() const    { return id_; }

    const BackupHash & backupHash() const     { return backupHash_; }
    BackupHash & backupHash()                 { return backupHash_; }
    const BackupHash & newBackupHash() const  { return newBackupHash_; }
    BackupHash & newBackupHash()              { return newBackupHash_; }

    bool isRetiring() const   { return isRetiring_; }
    bool isOffloading() const { return isOffloading_; }

    void retireApp( const Mercury::Address & addr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    void startBackup( const Mercury::Address & addr );
    void stopBackup( const Mercury::Address & addr );
    void checkToStartOffloading();

private:
    BaseAppMgr &            baseAppMgr_;
    Mercury::Address        externalAddr_;   // 给客户端用的外网地址
    BaseAppID               id_;
    float                   load_;
    int                     numBases_;
    int                     numProxies_;
    BackupHash              backupHash_;     // 当前生效的备份目标哈希
    BackupHash              newBackupHash_;  // 正在切换到的新哈希
    bool                    isRetiring_;
    bool                    isOffloading_;
    typedef std::set< Mercury::Address > BackingUp;
    BackingUp               backingUp_;      // 谁在向我备份
};
```

要点：
- `externalAddr_` 是该 baseapp 对客户端暴露的地址，登录成功后 dbmgr 把它返回给 loginapp → 客户端，客户端直接连这个地址建立 UDP channel。
- `backupHash_` / `newBackupHash_` 双哈希机制：当 baseapp 集群成员变化时，baseappmgr 通知每个 baseapp 计算新的 `newBackupHash_`，baseapp 把存量备份逐步迁移到新哈希，迁移完成后通过 `useNewBackupHash` 消息把 `newBackupHash_` 提升为 `backupHash_`。
- `isRetiring_` / `isOffloading_` 标志 retirement 流程：retire 的 baseapp 不再接受新 base，但继续服务存量 base 直到它们被 offload 到其他 baseapp。

### 2.4 进程入口与初始化

入口见 [main.cpp](file:///workspace/src/server/baseappmgr/main.cpp)（与其它进程同构）：

```cpp
int BIGWORLD_MAIN( int argc, char * argv[] )
{
    return bwMainT< BaseAppMgr >( argc, argv );
}
```

`BIGWORLD_MAIN` 宏把 `main` 包装成调用 `ServerApp::runApp`，再调用 `BaseAppMgr::init`。`init` 流程（节选自 [baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp)）：

```cpp
bool BaseAppMgr::init( int argc, char * argv[] )
{
    if (!this->ManagerApp::init( argc, argv )) return false;
    if (!interface_.isGood()) { ERROR_MSG( "Failed to open internal interface.\n" ); return false; }

    ReviverSubject::instance().init( &interface_, "baseAppMgr" );   // 注册为 reviver 监控对象

    for (int i = 0; i < argc; ++i)
    {
        if (strcmp( argv[i], "-recover" ) == 0)  // -recover 表示是 reviver 拉起的恢复实例
            isRecovery_ = true;
    }
    // ... 注册 BaseAppMgrInterface、监听 CellAppMgrInterface/DBInterface birth ...
    return true;
}
```

关键初始化点：
1. `ReviverSubject::instance().init( &interface_, "baseAppMgr" )` —— 让本进程响应 reviver 的 `reviverPing`，把自己纳入看门狗监控（见第 7 节）。
2. 通过 `Mercury::MachineDaemon::registerBirthListener` 监听 `CellAppMgrInterface` 与 `DBInterface` 的 birth 消息，等到 cellappmgr/dbmgr 上线后建立 channel。
3. `-recover` 命令行参数标记本次启动是 reviver 触发的"故障恢复"，此时已有的 baseapp 会通过 `recoverBaseApp` 消息把自己重新注册回来（带上原有的 id、backupHash、sharedData、globalBases）。

### 2.5 baseapp 注册流程：add 消息

baseapp 启动后会向 baseappmgr 发送 `BaseAppMgrInterface::add` 请求（带 `addrForCells` 内网地址 + `addrForClients` 外网地址）。baseappmgr 处理（[baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp)）：

```cpp
void BaseAppMgr::add( const Mercury::Address & srcAddr,
    const Mercury::UnpackedMessageHeader & header,
    const BaseAppMgrInterface::addArgs & args )
{
    const Mercury::ReplyID replyID = header.replyID;
    MF_ASSERT( srcAddr == args.addrForCells );

    // 还没准备好（cellappmgr 未就绪或还没拿到 initData）就回空 reply，让 baseapp 等待重试
    if (!cellAppMgr_.channel().isEstablished() || !hasInitData_)
    {
        Mercury::ChannelSender sender( BaseAppMgr::getChannel( srcAddr ) );
        sender.bundle().startReply( replyID );
        return;
    }
    if (!allowNewBaseApps_ || (shutDownStage_ != SHUTDOWN_NONE)) return; // 让它超时

    // 第一个 baseapp：通知 cellappmgr "base app 已就绪"，cellapp 才能创建 cell 实体
    if (baseApps_.empty())
    {
        Mercury::Bundle & bundle = cellAppMgr_.bundle();
        CellAppMgrInterface::setBaseAppArgs setBaseAppArgs;
        setBaseAppArgs.addr = args.addrForCells;
        bundle << setBaseAppArgs;
        cellAppMgr_.send();
        bestBaseAppAddr_ = args.addrForCells;
    }

    // 分配 id 并登记
    BaseAppID id = this->getNextID();
    BaseAppPtr pBaseApp( new BaseApp( *this, args.addrForCells, args.addrForClients, id ) );
    baseApps_[ srcAddr ] = pBaseApp;

    // 回 reply：带 BaseAppInitData（id, time, isReady）
    Mercury::Bundle & bundle = pBaseApp->bundle();
    bundle.startReply( replyID );
    BaseAppInitData initData;
    initData.id = id; initData.time = time_; initData.isReady = hasStarted_;
    bundle << initData;

    // 把已知的 globalBases、sharedBaseAppData、sharedGlobalData 同步给新 baseapp
    // 把新 baseapp 通告给其它 baseapp（BaseAppIntInterface::handleBaseAppBirth）
    // 通知其它 baseapp 也通告给新 baseapp
    // ...

    // 调整备份哈希环（新增节点）
    this->adjustBackupLocations( pBaseApp->addr(), ADJUST_BACKUP_LOCATIONS_OP_ADD );
    pBaseApp->send();
}
```

`BaseAppInitData` 结构（[baseappmgr_interface.hpp](file:///workspace/src/server/baseappmgr/baseappmgr_interface.hpp)）：

```cpp
#pragma pack( push, 1 )
struct BaseAppInitData
{
    int32 id;        //!< ID of the new BaseApp
    GameTime time;   //!< Current game time
    bool isReady;    //!< Flag indicating whether the server is ready
};
#pragma pack( pop )
```

> **设计要点**：baseappmgr 在 `add` reply 里把当前 game time 同步给新 baseapp，让所有 baseapp 的 game time 与 baseappmgr 对齐；`isReady=false` 时 baseapp 会继续等待 `requestHasStarted` 后才接受玩家。

### 2.6 负载均衡：findBestBaseApp 与 updateCreateBaseInfo

#### 2.6.1 最小负载选择

`createEntity`（来自 dbmgr 的登录回路）每次都调用 `findBestBaseApp` 选出当前 load 最小、且未 retiring 的 baseapp（[baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp)）：

```cpp
BaseApp * BaseAppMgr::findBestBaseApp() const
{
    const BaseApp * pBest = NULL;
    float lowestLoad = 0.f;
    BaseAppMgr::BaseApps::const_iterator iter = baseApps_.begin();
    while (iter != baseApps_.end())
    {
        float currLoad = iter->second->load();
        if (!iter->second->isRetiring() &&
                (!pBest || currLoad < lowestLoad))
        {
            lowestLoad = currLoad;
            pBest = iter->second.get();
        }
        ++iter;
    }
    return const_cast< BaseApp * >( pBest );
}
```

`load` 来自 baseapp 周期上报的 `informOfLoad` 消息（参数 `load` / `numBases` / `numProxies`）。

#### 2.6.2 createBaseAnywhere 方案

除了"登录走 baseappmgr 路由"外，baseapp 之间也可以直接互相创建 base（`Base.createBaseAnywhere`）。baseappmgr 周期性（`updateCreateBaseInfoPeriodInTicks`）调用 `updateCreateBaseInfo`，把所有 baseapp 按 load 排序后，让前 `1/createBaseRatio` 比例的 baseapp 作为"目标 baseapp"，其余 baseapp 各自指向一个目标 baseapp（[baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp)）：

```cpp
void BaseAppMgr::updateCreateBaseInfo()
{
    // 收集所有 baseapp 指针
    typedef std::vector< BaseApp * > BaseAppsVec;
    BaseAppsVec apps; apps.reserve( baseApps_.size() );
    /* ... 填充 apps ... */

    // 按 load 升序排序（loadCmp）
    std::sort( apps.begin(), apps.end(), loadCmp<BaseApp>() );

    int totalSize = apps.size();
    int destSize = int(totalSize/Config::createBaseRatio() + 0.99f);
    destSize = std::min( totalSize, std::max( 1, destSize ) );

    // 后段随机洗牌，避免退化场景
    std::random_shuffle( apps.begin() + destSize, apps.end() );

    // 通知每个 baseapp 它的 createBase 目标
    for (size_t i = 0; i < apps.size(); ++i)
    {
        Mercury::Bundle & bundle = apps[ i ]->bundle();
        int destIndex = i % destSize;
        bundle.startMessage( BaseAppIntInterface::setCreateBaseInfo );
        bundle << apps[ destIndex ]->addr();
        apps[ i ]->send();
    }
}
```

相关配置（[baseappmgr_config.cpp](file:///workspace/src/server/baseappmgr/baseappmgr_config.cpp)）：

```cpp
BW_OPTION( float, baseAppOverloadLevel, 0.8f );      // baseapp load 超过此值视为过载
BW_OPTION( float, createBaseRatio, 4.f );             // 1/4 的 baseapp 作为 createBase 目标
BW_OPTION_RO( float, updateCreateBaseInfoPeriod, 5.f ); // 每 5 秒重算一次
BW_OPTION( bool, hardKillDeadBaseApps, true );        // 死亡 baseapp 直接 SIGQUIT
BW_OPTION_RO( float, baseAppTimeout, 5.f );           // 5 秒无消息视为超时
BW_OPTION_RO( float, overloadTolerancePeriod, 5.f );  // 过载容忍期
BW_OPTION( int, overloadLogins, 10 );                 // 过载期允许的登录数
```

### 2.7 过载保护：calculateOverloaded

baseappmgr 在 `createEntity` 里检测：若最优 baseapp 的 load > `baseAppOverloadLevel`，进入过载计时；若持续超过 `overloadTolerancePeriod` 或累计登录数超过 `overloadLogins`，则拒绝登录（返回 `CREATE_ENTITY_ERROR_BASEAPPS_OVERLOADED`）：

```cpp
bool BaseAppMgr::calculateOverloaded( bool baseAppsOverloaded )
{
    if (baseAppsOverloaded)
    {
        if (baseAppOverloadStartTime_ == 0) baseAppOverloadStartTime_ = timestamp();
        uint64 overloadTime = timestamp() - baseAppOverloadStartTime_;

        if ((overloadTime > Config::overloadTolerancePeriodInStamps()) ||
            (loginsSinceOverload_ >= Config::overloadLogins()))
        {
            return true;   // 真正拒绝
        }
        else
        {
            loginsSinceOverload_++;   // 容忍期内放行
        }
    }
    else
    {
        baseAppOverloadStartTime_ = 0;
        loginsSinceOverload_ = 0;
    }
    return false;
}
```

> **设计要点**：过载不是"一旦超阈值就拒绝"，而是有一个容忍窗口，避免短时尖峰导致误拒。这与 dbmgr 侧的 `overloadTolerancePeriod`（见第 5.7 节）是同一套思路。

### 2.8 备份哈希环：BackupHashChain

[backup_hash_chain.hpp](file:///workspace/src/lib/server/backup_hash_chain.hpp) 定义了 `BackupHashChain`，它保存一份"历史死亡 baseapp 的备份哈希"映射：

```cpp
class BackupHashChain
{
public:
    BackupHashChain();
    ~BackupHashChain();

    void adjustForDeadBaseApp( const Mercury::Address & deadApp,
                               const BackupHash & hash );
    Mercury::Address addressFor( const Mercury::Address & address,
                                 EntityID entityID ) const;

private:
    typedef std::map< Mercury::Address, BackupHash > History;
    History history_;
    // 流式支持
    friend BinaryOStream & operator<<( BinaryOStream & os, const BackupHashChain & hashChain );
    friend BinaryIStream & operator>>( BinaryIStream & is, BackupHashChain & hashChain );
};
```

当一个 baseapp 崩溃时，`onBaseAppDeath` 调用 `pBackupHashChain_->adjustForDeadBaseApp( addr, baseApp.backupHash() )`，把死亡 baseapp 的地址 → 它当时的 `backupHash` 记录到历史。后续查找 mailbox 时，`addressFor` 会沿历史链回溯，找到该实体当时被备份到的 baseapp。

### 2.9 baseapp 死亡处理：onBaseAppDeath

[baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp)：

```cpp
bool BaseAppMgr::onBaseAppDeath( const Mercury::Address & addr )
{
    BaseApps::iterator iter = baseApps_.find( addr );
    if (iter == baseApps_.end()) return false;

    BaseApp & baseApp = *iter->second;
    bool controlledShutDown = false;

    // 1) 强杀：确保死亡 baseapp 真的死了，否则备份方接管其地址会冲突
    if (Config::hardKillDeadBaseApps())
    {
        sendSignalViaMachined( baseApp.addr(), SIGQUIT );
    }

    // 2) 决定是否升级为整服受控停机
    if (Config::shutDownServerOnBaseAppDeath())
    {
        controlledShutDown = true;
    }
    else if (baseApp.backupHash().empty())
    {
        // 没有备份或备份尚未就绪
        if (Config::shutDownServerOnBadState()) controlledShutDown = true;
    }

    // 3) 把死亡 baseapp 的 backupHash 记入 BackupHashChain
    pBackupHashChain_->adjustForDeadBaseApp( addr, baseApp.backupHash() );

    // 4) 通知 cellappmgr：baseapp 死了，cell 实体要切 ghost / 销毁
    {
        Mercury::Bundle & bundle = cellAppMgr_.bundle();
        bundle.startMessage( CellAppMgrInterface::handleBaseAppDeath );
        bundle << addr << baseApp.backupHash();
        cellAppMgr_.send();
    }

    // 5) 通知其它所有 baseapp：handleBaseAppDeath，触发它们从备份恢复实体
    unsigned int numBaseAppsAlive = baseApps_.size() - 1;
    if (numBaseAppsAlive > 0 && !controlledShutDown)
    {
        MemoryOStream args;
        args << addr << baseApp.backupHash();
        this->sendToBaseApps( BaseAppIntInterface::handleBaseAppDeath, args, &baseApp );
        deadBaseAppAddr_ = addr;
        archiveCompleteMsgCounter_ = numBaseAppsAlive;
    }

    // 6) 重映射 globalBases_ 中指向死亡 baseapp 的 mailbox
    {
        GlobalBases::iterator gbIter = globalBases_.begin();
        while (gbIter != globalBases_.end())
        {
            EntityMailBoxRef & mailbox = gbIter->second;
            if (mailbox.addr == addr)
            {
                Mercury::Address newAddr = baseApp.backupHash().addressFor( mailbox.id );
                mailbox.addr.ip = newAddr.ip;
                mailbox.addr.port = newAddr.port;
            }
            ++gbIter;
        }
    }

    baseApps_.erase( iter );
    this->adjustBackupLocations( addr, ADJUST_BACKUP_LOCATIONS_OP_CRASH );

    // 7) 通知所有 baseapp 停止向死亡 baseapp 备份
    iter = baseApps_.begin();
    while (iter != baseApps_.end()) { iter->second->stopBackup( addr ); ++iter; }

    if (controlledShutDown) this->controlledShutDownServer();
    return true;
}
```

死亡检测有两种触发源：
1. **被动**：baseapp 主动发 `del` 消息（受控退出）。
2. **主动**：`checkForDeadBaseApps` 在每个 game tick 检查 `BaseApp::hasTimedOut`，若超过 `baseAppTimeoutInStamps` 未收到该 baseapp 的任何消息则判超时。`hasTimedOut` 有一个保护：若"任何 baseapp 都很久没消息"则不判超时（说明可能是 baseappmgr 自己卡了），见 [baseapp.cpp](file:///workspace/src/server/baseappmgr/baseapp.cpp)。

### 2.10 受控停机：controlledShutDownServer

baseappmgr 把整服停机分成多个 `ShutDownStage`（定义在 [server/common.hpp](file:///workspace/src/lib/server/common.hpp)，与 cellappmgr 共用）。`controlledShutDownServer` 依次推进 stage，并通过 `startAsyncShutDownStage` 异步等待 cellappmgr/baseapp 的 ack：

```cpp
void BaseAppMgr::controlledShutDownServer()
{
    // ... 检查 shutDownStage_，推进到下一阶段 ...
    // 通知 cellappmgr controlledShutDown
    // 通知所有 baseapp controlledShutDown
    // 等待 ackBaseAppsShutDown / ackCellAppShutDown
}
```

`archiveCompleteMsgCounter_` 用于等待所有 baseapp 完成最后一轮归档（`informOfArchiveComplete`），归档完成后才能进入下一 stage。

### 2.11 baseappmgr 接口一览

[baseappmgr_interface.hpp](file:///workspace/src/server/baseappmgr/baseappmgr_interface.hpp) 用 Mercury IDL 宏声明所有消息：

```cpp
BEGIN_MERCURY_INTERFACE( BaseAppMgrInterface )

    BW_ANONYMOUS_CHANNEL_CLIENT_MSG( DBInterface )   // 复用 DBInterface 的匿名 channel 消息

    enum CreateEntityError
    {
        CREATE_ENTITY_ERROR_NO_BASEAPPS = 1,
        CREATE_ENTITY_ERROR_BASEAPPS_OVERLOADED
    };

    BW_STREAM_MSG_EX( BaseAppMgr, createEntity )      // dbmgr → baseappmgr：登录路由

    BW_BEGIN_STRUCT_MSG_EX( BaseAppMgr, add )         // baseapp → baseappmgr：注册
        Mercury::Address addrForCells;
        Mercury::Address addrForClients;
    END_STRUCT_MESSAGE()

    BW_STREAM_MSG_EX( BaseAppMgr, recoverBaseApp )    // baseapp → baseappmgr：恢复注册

    BW_BEGIN_STRUCT_MSG_EX( BaseAppMgr, del )         // baseapp → baseappmgr：受控退出
        BaseAppID id;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG_EX( BaseAppMgr, informOfLoad )// baseapp → baseappmgr：负载上报
        float load;
        int numBases;
        int numProxies;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( BaseAppMgr, shutDown )
        bool shouldShutDownOthers;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( BaseAppMgr, controlledShutDown )
        ShutDownStage stage;
        GameTime shutDownTime;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( BaseAppMgr, handleBaseAppDeath )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( BaseAppMgr, handleCellAppMgrBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( BaseAppMgr, handleBaseAppMgrBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    BW_STREAM_MSG_EX( BaseAppMgr, handleCellAppDeath )
    BW_STREAM_MSG_EX( BaseAppMgr, registerBaseGlobally )   // 注册全局 base
    BW_STREAM_MSG_EX( BaseAppMgr, deregisterBaseGlobally )
    BW_STREAM_MSG_EX( BaseAppMgr, requestHasStarted )      // dbmgr 启动时探测
    BW_STREAM_MSG_EX( BaseAppMgr, initData )               // dbmgr → baseappmgr：初始 game time 等
    BW_STREAM_MSG_EX( BaseAppMgr, startup )                // dbmgr → baseappmgr：开始服务
    BW_STREAM_MSG_EX( BaseAppMgr, checkStatus )            // 健康检查
    BW_STREAM_MSG_EX( BaseAppMgr, spaceDataRestore )       // 转发给 cellappmgr
    BW_STREAM_MSG( BaseAppMgr, setSharedData )             // 共享数据 set
    BW_STREAM_MSG( BaseAppMgr, delSharedData )             // 共享数据 del
    BW_STREAM_MSG_EX( BaseAppMgr, useNewBackupHash )       // baseapp → baseappmgr：备份迁移完成
    BW_STREAM_MSG_EX( BaseAppMgr, informOfArchiveComplete )// baseapp → baseappmgr：归档完成
    BW_STREAM_MSG_EX( BaseApp, retireApp )                 // baseapp → baseappmgr：申请退役
    BW_STREAM_MSG_EX( BaseAppMgr, requestBackupHashChain ) // baseapp → baseappmgr：请求历史哈希链

    MF_REVIVER_PING_MSG()                                  // reviver 心跳

END_MERCURY_INTERFACE()
```

`MF_REVIVER_PING_MSG()` 宏（[reviver_subject.hpp](file:///workspace/src/lib/server/reviver_subject.hpp)）为每个被 reviver 监控的进程注入 `reviverPing` 消息，由 `ReviverSubject` 单例处理。

### 2.12 baseappmgr 配置

[baseappmgr_config.hpp](file:///workspace/src/server/baseappmgr/baseappmgr_config.hpp)：

```cpp
class BaseAppMgrConfig : public ManagerAppConfig
{
public:
    static ServerAppOption< float > baseAppOverloadLevel;
    static ServerAppOption< float > createBaseRatio;
    static ServerAppOption< float > updateCreateBaseInfoPeriod;
    static ServerAppOption< int > updateCreateBaseInfoPeriodInTicks;
    static ServerAppOption< bool > hardKillDeadBaseApps;
    static ServerAppOption< float > baseAppTimeout;
    static ServerAppOption< uint64 > baseAppTimeoutInStamps;
    static ServerAppOption< float > overloadTolerancePeriod;
    static ServerAppOption< uint64 > overloadTolerancePeriodInStamps;
    static ServerAppOption< int > overloadLogins;
    static bool postInit();
};
```

`postInit` 把秒级配置预换算成 ticks/stamps（[baseappmgr_config.cpp](file:///workspace/src/server/baseappmgr/baseappmgr_config.cpp)）：

```cpp
bool BaseAppMgrConfig::postInit()
{
    bool result = ManagerAppConfig::postInit();
    updateCreateBaseInfoPeriodInTicks.set( secondsToTicks( updateCreateBaseInfoPeriod(), 1 ) );
    baseAppTimeoutInStamps.set( uint64( stampsPerSecondD() * baseAppTimeout() ) );
    overloadTolerancePeriodInStamps.set( uint64( stampsPerSecondD() * overloadTolerancePeriod() ) );
    return result;
}
```

---

## 3. cellappmgr — cellapp 集群总控

### 3.1 模块定位与源码开放情况

`cellappmgr` 同样是单例进程，是 cellapp 集群的总控。它负责：
1. 维护 cellapp 注册表（`CellApps` map）。
2. 维护 Space → Cell 的映射（一个 Space 在多个 cellapp 上有多个 Cell）。
3. 把"创建 cell 实体"请求路由到合适的 cellapp。
4. 与 baseappmgr 协同：baseappmgr 把"第一个 baseapp 地址"通过 `setBaseApp` 通知 cellappmgr，cellappmgr 再下发给 cellapp，让 cell 实体能找到对应的 base。
5. 推进 game time（与 baseappmgr 通过 `gameTimeReading` 对齐）。
6. cellapp 死亡处理：触发 cell 重划分、ghost 切换。

> **重要说明**：本仓库的 `src/server/cellappmgr/` 目录**只包含接口定义** [cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp) 与 [cellappmgr_interface.cpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.cpp)，**`CellAppMgr` 类的完整实现未随源码开放**（与 baseappmgr 不同，baseappmgr 的 `.cpp` 是完整的）。因此本节以接口契约为线索，结合 cellapp 侧与 baseappmgr 侧的调用反推其行为；不能确定的实现细节会明确标注"参见 cellappmgr_interface.hpp"。

### 3.2 cellappmgr 接口契约

[cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp) 声明的全部消息：

```cpp
BEGIN_MERCURY_INTERFACE( CellAppMgrInterface )

    BW_ANONYMOUS_CHANNEL_CLIENT_MSG( DBInterface )    // 复用 DBInterface 匿名 channel

    // ---- 实体创建 ----
    BW_STREAM_MSG_EX( CellAppMgr, createEntity )              // baseappmgr/dbmgr → cellappmgr
    BW_STREAM_MSG_EX( CellAppMgr, createEntityInNewSpace )    // 在新 space 创建
    BW_STREAM_MSG_EX( CellAppMgr, prepareForRestoreFromDB )   // 从 DB 恢复空间

    // ---- 生命周期 ----
    BW_STREAM_MSG_EX( CellAppMgr, startup )                   // dbmgr → cellappmgr：开始服务
    BW_BEGIN_STRUCT_MSG( CellAppMgr, shutDown )
        bool isSigInt;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG( CellAppMgr, controlledShutDown )
        ShutDownStage stage;
    END_STRUCT_MESSAGE()

    // ---- 负载与退役 ----
    BW_BEGIN_STRUCT_MSG( CellAppMgr, shouldOffload )
        bool enable;
    END_STRUCT_MESSAGE()
    BW_STREAM_MSG_EX( CellAppMgr, addApp );                   // cellapp → cellappmgr：注册
    BW_STREAM_MSG( CellAppMgr, recoverCellApp );              // cellapp → cellappmgr：恢复注册
    BW_BEGIN_STRUCT_MSG( CellAppMgr, delApp )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    // ---- 与 baseappmgr 协同 ----
    BW_BEGIN_STRUCT_MSG( CellAppMgr, setBaseApp )              // baseappmgr → cellappmgr
        Mercury::Address addr;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG( CellAppMgr, handleCellAppMgrBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG( CellAppMgr, handleBaseAppMgrBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG_EX( CellAppMgr, handleCellAppDeath )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()
    BW_STREAM_MSG( CellAppMgr, handleBaseAppDeath )            // baseappmgr → cellappmgr
    BW_BEGIN_STRUCT_MSG_EX( CellAppMgr, ackCellAppDeath )
        Mercury::Address deadAddr;
    END_STRUCT_MESSAGE()

    // ---- game time ----
    BW_BEGIN_STRUCT_MSG_EX( CellAppMgr, gameTimeReading )
        double gameTimeReadingContribution;
    END_STRUCT_MESSAGE()

    // ---- 空间数据 ----
    BW_STREAM_MSG_EX( CellAppMgr, updateSpaceData )
    BW_BEGIN_STRUCT_MSG( CellAppMgr, shutDownSpace )
        SpaceID spaceID;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG( CellAppMgr, ackBaseAppsShutDown )
        ShutDownStage stage;
    END_STRUCT_MESSAGE()
    BW_STREAM_MSG_EX( CellAppMgr, checkStatus )

    // ---- CellApp 上报消息（处理方在 cellappmgr） ----
    BW_BEGIN_STRUCT_MSG( CellApp, informOfLoad )
        float load;
        int numEntities;
    END_STRUCT_MESSAGE()
    BW_STREAM_MSG( CellApp, updateBounds );                    // cellapp 上报 cell 边界
    BW_BEGIN_STRUCT_MSG( CellApp, retireApp )
        int8 dummy;
    END_STRUCT_MESSAGE()
    BW_BEGIN_STRUCT_MSG( CellApp, ackCellAppShutDown )
        ShutDownStage stage;
    END_STRUCT_MESSAGE()

    MF_REVIVER_PING_MSG()

    BW_STREAM_MSG( CellAppMgr, setSharedData )
    BW_STREAM_MSG( CellAppMgr, delSharedData )

END_MERCURY_INTERFACE()
```

### 3.3 关键消息语义

| 消息 | 方向 | 含义 |
| --- | --- | --- |
| `addApp` | cellapp → cellappmgr | cellapp 启动后注册自己，携带内/外地址、空间恢复数据 |
| `recoverCellApp` | cellapp → cellappmgr | reviver 拉起的恢复实例注册 |
| `delApp` | cellapp → cellappmgr | cellapp 受控退出 |
| `informOfLoad` (CellApp) | cellapp → cellappmgr | 周期上报 load + numEntities |
| `updateBounds` | cellapp → cellappmgr | 上报本 cellapp 上各 cell 的边界（用于空间划分决策） |
| `retireApp` (CellApp) | cellapp → cellappmgr | 申请退役（不再接受新 cell） |
| `createEntity` | baseappmgr → cellappmgr | 创建 cell 实体（参数同 `Cell::createEntity`，前两个固定为 EntityID + Position3D） |
| `createEntityInNewSpace` | baseappmgr → cellappmgr | 在新 space 创建实体 |
| `setBaseApp` | baseappmgr → cellappmgr | 通告"主 baseapp"地址，cellappmgr 下发给所有 cellapp |
| `handleBaseAppDeath` | baseappmgr → cellappmgr | baseapp 死亡，cellappmgr 通知 cellapp 销毁对应 base 实体的 cell 镜像 |
| `handleCellAppDeath` | cellappmgr → cellappmgr（内部）/ baseappmgr | cellapp 死亡通告 |
| `ackCellAppDeath` | cellappmgr → baseappmgr | cellapp 死亡处理完成 ack |
| `gameTimeReading` | cellappmgr → baseappmgr | 贡献 game time 读数（用于同步） |
| `controlledShutDown` | baseappmgr ↔ cellappmgr | 受控停机阶段推进 |
| `ackBaseAppsShutDown` | cellappmgr → baseappmgr | baseapp 停机阶段 ack |
| `shouldOffload` | cellappmgr → cellappmgr/外部 | 启用/禁用 offload |
| `startup` | dbmgr → cellappmgr | 服务开始 |
| `checkStatus` | 外部 → cellappmgr | 健康检查 |
| `setSharedData` / `delSharedData` | cellappmgr ↔ baseappmgr | 全局共享数据同步 |

### 3.4 cellapp 注册与 space 划分（推断）

虽然 `CellAppMgr` 类源码未开放，但从 cellapp 侧（[10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)）与 baseappmgr 侧可推断流程：

```
cellapp 启动
   │
   │  CellApp::init → MachineDaemon::registerBirthListener("CellAppMgrInterface")
   │  + 向 cellappmgr 发 addApp（带本进程地址、已恢复的 space 数据）
   ▼
cellappmgr::addApp
   │
   │  1. 把 cellapp 加入 CellApps 表
   │  2. 若是首个 cellapp，设置为主 cellapp
   │  3. 通知所有 cellapp：handleCellAppBirth（让它们互相同步）
   │  4. 调整 space 划分：把现有 space 的部分 cell 迁移到新 cellapp（负载均衡）
   │  5. 回复 cellapp：分配 id、game time、space 列表
   ▼
cellapp 收到 reply 后开始接受 createEntity
```

cellappmgr 通过 `updateBounds` 收集每个 cellapp 上 cell 的边界，决定何时切分 cell（一个 cell 太大就分裂，把一半区域交给新 cellapp）。这部分逻辑在 cellapp 侧的 `Cell` 类（见 [10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)）里也有配合：cellapp 主动上报 bounds，cellappmgr 下发"切分到某 cellapp"指令。

### 3.5 cellapp 死亡处理（推断）

当 cellapp 死亡（`delApp` 或超时），cellappmgr 需要：
1. 把死亡 cellapp 上所有 cell 重新分配给其它 cellapp（触发 cell 重建 + 实体迁移）。
2. 通知 baseappmgr（`handleCellAppDeath`），baseappmgr 再通知所有 baseapp，让 base 实体重新创建对应的 cell 实体。
3. 通知所有 cellapp：销毁指向死亡 cellapp 的 ghost，重建 ghost 到新 cellapp。

具体实现细节参见 [cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp) 的消息契约与 cellapp 侧的 `CellApp` / `Cell` / `RealEntity` 类。

### 3.6 game time 同步

`gameTimeReading` 消息（cellappmgr → baseappmgr）携带一个 `double gameTimeReadingContribution`。注释里写 `// double is good for ~100 000 years`，说明 game time 用 double 累积，cellappmgr 把自己的读数贡献给 baseappmgr，baseappmgr 的 `TimeKeeper` 与 cellappmgr 协同对齐。

---

## 4. dbmgr — 持久化与登录总控

### 4.1 模块定位

`dbmgr` 是单例进程，承担三大职责：

1. **持久化总控**：通过 `IDatabase` 抽象层支持 XML / MySQL 两种主库，负责实体的 load / write / delete / lookup。baseapp 周期性把实体归档到这里（`writeEntity`），登录时从这里 load 实体（`loadEntity` / `getEntity`）。
2. **登录鉴权**：客户端登录请求经 loginapp 转发到 dbmgr，dbmgr 通过 `BillingSystem` 把"用户名/密码"映射到 `EntityKey`，再 load/create 实体，最后通过 baseappmgr 路由到具体 baseapp。
3. **二级库合并（consolidation）**：每个 baseapp 可选维护本地 SQLite 二级库（用于主库不可达时本地落盘）。dbmgr 启动/停机时会触发 `consolidate_dbs` 子进程，把所有二级库的数据合并回主库。
4. **服务编排**：dbmgr 是整个服务端启动的"发令枪"。它先启动，等 baseappmgr/cellappmgr 就绪后发 `initData` / `startup`，再触发 `autoLoadEntities` 把上次停机时活跃的实体重新加载。

### 4.2 核心类：Database

[database.hpp](file:///workspace/src/server/dbmgr/database.hpp)：

```cpp
class Database : public ServerApp,
    public TimerHandler,
    public IDatabase::IGetBaseAppMgrInitDataHandler,
    public IDatabase::IUpdateSecondaryDBshandler,
    public Singleton< Database >
{
public:
    SERVER_APP_HEADER( DBMgr, dbMgr )
    typedef DBMgrConfig Config;

    Database( Mercury::EventDispatcher & dispatcher,
            Mercury::NetworkInterface & interface );
    virtual ~Database();

    BaseAppMgr & baseAppMgr() { return baseAppMgr_; }   // 到 baseappmgr 的 channel

    // ---- 生命周期消息 ----
    void handleBaseAppMgrBirth( const DBInterface::handleBaseAppMgrBirthArgs & args );
    void handleDatabaseBirth( const DBInterface::handleDatabaseBirthArgs & args );
    void shutDown( const DBInterface::shutDownArgs & args );
    void startSystemControlledShutdown();
    void shutDownNicely();
    void shutDown();
    void controlledShutDown( const DBInterface::controlledShutDownArgs & args );
    void cellAppOverloadStatus( const DBInterface::cellAppOverloadStatusArgs & args );

    // ---- 实体消息 ----
    void logOn( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void logOn( const Mercury::Address & srcAddr, Mercury::ReplyID replyID,
        LogOnParamsPtr pParams, const Mercury::Address & addrForProxy,
        bool offChannel );
    void onLogOnLoggedOnUser( EntityTypeID typeID, DatabaseID dbID,
        LogOnParamsPtr pParams, const Mercury::Address & proxyAddr,
        const Mercury::Address & replyAddr, bool offChannel,
        Mercury::ReplyID replyID, const EntityMailBoxRef* pExistingBase );
    void onEntityLogOff( EntityTypeID typeID, DatabaseID dbID );

    bool calculateOverloaded( bool isOverloaded );
    void sendFailure( Mercury::ReplyID replyID, const Mercury::Address & dstAddr,
        bool offChannel, LogOnStatus reason, const char * pDescription = NULL );

    void loadEntity( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void writeEntity( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    void deleteEntity( ... );
    void deleteEntityByName( ... );
    void lookupEntity( ... );
    void lookupEntityByName( ... );
    void lookupDBIDByName( ... );

    // ---- 杂项 ----
    void executeRawCommand( ... );     // 执行原始 SQL 等
    void putIDs( ... );                // 写回 EntityID 池
    void getIDs( ... );                // 获取 EntityID 池
    void writeSpaces( ... );           // 持久化 space 数据
    void writeGameTime( const DBInterface::writeGameTimeArgs & args );
    void handleBaseAppDeath( ... );
    void checkStatus( ... );

    // ---- 二级库 ----
    void secondaryDBRegistration( ... );
    void updateSecondaryDBs( ... );
    void getSecondaryDBDetails( ... );
    virtual void onUpdateSecondaryDBsComplete( const IDatabase::SecondaryDBEntries& removedEntries );

    // ---- IDatabase 透传方法（拦截以做额外处理）----
    void getEntity( const EntityDBKey & entityKey, BinaryOStream * pStream,
            bool shouldGetBaseEntityLocation, const char * pPasswordOverride,
            GetEntityHandler & handler );
    void putEntity( const EntityKey & ekey, EntityID entityID,
            BinaryIStream * pStream, EntityMailBoxRef * pBaseMailbox,
            bool removeBaseMailbox, UpdateAutoLoad updateAutoLoad,
            IDatabase::IPutEntityHandler& handler );
    void setBaseEntityLocation( const EntityKey & entityKey,
            EntityMailBoxRef & mailbox, IDatabase::IPutEntityHandler & handler,
            UpdateAutoLoad updateAutoLoad = UPDATE_AUTO_LOAD_RETAIN );
    void clearBaseEntityLocation( const EntityKey & entityKey,
            IDatabase::IPutEntityHandler & handler );
    void delEntity( const EntityDBKey & ekey, EntityID entityID,
            IDatabase::IDelEntityHandler& handler );

    // ---- 状态 ----
    void hasStartBegun( bool hasStartBegun );
    bool hasStartBegun() const { return status_.status() > DBStatus::WAITING_FOR_APPS; }
    bool isConsolidating() const { return pConsolidator_.get() != NULL; }
    bool isReady() const { return status_.status() >= DBStatus::WAITING_FOR_APPS; }
    void startServerBegin( bool isRecover = false );
    void startServerEnd( bool isRecover, bool didAutoLoadEntitiesFromDB );
    void startServerError();

    // ---- 重登录 ----
    RelogonAttemptHandler* getInProgRelogonAttempt( EntityTypeID typeID, DatabaseID dbID );
    void onStartRelogonAttempt( EntityTypeID typeID, DatabaseID dbID, RelogonAttemptHandler* pHandler );
    void onCompleteRelogonAttempt( EntityTypeID typeID, DatabaseID dbID );

    // ---- checkout 跟踪 ----
    bool onStartEntityCheckout( const EntityKey& entityID );
    bool onCompleteEntityCheckout( const EntityKey& entityID, const EntityMailBoxRef* pBaseRef );
    bool registerCheckoutCompletionListener( EntityTypeID typeID, DatabaseID dbID,
            ICheckoutCompletionListener& listener );

    // ---- mailbox 重映射 ----
    bool hasMailboxRemapping() const { return !mailboxRemapInfo_.empty(); }
    void remapMailbox( EntityMailBoxRef& mailbox ) const;

private:
    virtual bool init( int argc, char * argv[] );
    virtual bool run();
    void finalise();
    void initSecondaryDBPrefix();
    bool initBillingSystem();
    void addWatchers( Watcher & watcher );
    void endMailboxRemapping();
    void sendInitData();
    virtual void onGetBaseAppMgrInitDataComplete( GameTime gameTime, int32 appID );

    // 数据合并
    void consolidateData();
    bool startConsolidationProcess();
    bool sendRemoveDBCmd( uint32 destIP, const std::string& dbLocation );

    EntityDefs*         pEntityDefs_;      // 实体定义（从 entities.xml 加载）
    IDatabase *         pDatabase_;        // XML / MySQL 实现
    BillingSystem *     pBillingSystem_;   // 计费/鉴权系统
    DBStatus            status_;           // 当前状态机
    BaseAppMgr          baseAppMgr_;       // 到 baseappmgr 的 channel
    bool                shouldSendInitData_;
    bool                shouldConsolidate_;
    TimerHandle         statusCheckTimerHandle_;

    typedef std::map< EntityKey, RelogonAttemptHandler * > PendingAttempts;
    PendingAttempts pendingAttempts_;      // 进行中的重登录
    typedef std::set< EntityKey > EntityKeySet;
    EntityKeySet    inProgCheckouts_;      // 进行中的 checkout（避免重复加载）
    typedef std::multimap< EntityKey, ICheckoutCompletionListener* >
            CheckoutCompletionListeners;
    CheckoutCompletionListeners checkoutCompletionListeners_;

    float           curLoad_;              // 当前负载（用于过载判断）
    bool            anyCellAppOverloaded_; // 是否有 cellapp 过载
    uint64          overloadStartTime_;

    typedef std::map< Mercury::Address, BackupHash > MailboxRemapInfo;
    MailboxRemapInfo mailboxRemapInfo_;    // baseapp 死亡后的 mailbox 重映射
    int             mailboxRemapCheckCount_;

    std::string     secondaryDBPrefix_;    // 二级库文件名前缀（runID）
    uint            secondaryDBIndex_;

    std::auto_ptr< Consolidator > pConsolidator_;  // 合并子进程管理
};
```

### 4.3 IDatabase 抽象层

[idatabase.hpp](file:///workspace/src/lib/dbmgr_lib/idatabase.hpp) 定义了数据库抽象接口，dbmgr 通过它屏蔽 XML / MySQL 差异：

```cpp
class IDatabase
{
public:
    virtual ~IDatabase() {}
    virtual bool startup( const EntityDefs& entityDefs,
            Mercury::EventDispatcher & dispatcher, bool isFaultRecovery ) = 0;
    virtual bool shutDown() = 0;
    virtual BillingSystem * createBillingSystem() = 0;

    // 实体 CRUD（异步，通过 handler 回调）
    virtual void getEntity( const EntityDBKey & entityKey, BinaryOStream * pStream,
            bool shouldGetBaseEntityLocation, const char * pPasswordOverride,
            IGetEntityHandler & handler ) = 0;
    virtual void putEntity( const EntityKey & entityKey, EntityID entityID,
            BinaryIStream * pStream, const EntityMailBoxRef * pBaseMailbox,
            bool removeBaseMailbox, UpdateAutoLoad updateAutoLoad,
            IPutEntityHandler & handler ) = 0;
    virtual void delEntity( const EntityDBKey & ekey, EntityID entityID,
            IDelEntityHandler& handler ) = 0;

    // BaseAppMgr 初始数据（game time + 二级库最大 appID）
    virtual void getBaseAppMgrInitData( IGetBaseAppMgrInitDataHandler& handler ) = 0;

    // 原始命令、ID 池、space 数据
    virtual void executeRawCommand( const std::string & command,
        IExecuteRawCommandHandler& handler ) = 0;
    virtual void putIDs( int count, const EntityID * ids ) = 0;
    virtual void getIDs( int count, IGetIDsHandler& handler ) = 0;
    virtual void writeSpaceData( BinaryIStream& spaceData ) = 0;
    virtual bool getSpacesData( BinaryOStream& strm ) = 0;

    // 自动加载（停机恢复）
    virtual void autoLoadEntities( IEntityAutoLoader & autoLoader ) = 0;

    // mailbox 重映射（baseapp 死亡后把 DB 里的 mailbox 改指向新 baseapp）
    virtual void remapEntityMailboxes( const Mercury::Address& srcAddr,
            const BackupHash & destAddrs ) = 0;

    // ---- 二级库 ----
    struct SecondaryDBEntry
    {
        Mercury::Address 	addr;     // BaseApp 地址
        int32				appID;    // BaseApp ID
        std::string			location; // 二级库文件路径
    };
    typedef std::vector< SecondaryDBEntry > SecondaryDBEntries;

    virtual void addSecondaryDB( const SecondaryDBEntry& entry ) = 0;
    virtual void updateSecondaryDBs( const BaseAppIDs& ids,
            IUpdateSecondaryDBshandler& handler ) = 0;
    virtual void getSecondaryDBs( IGetSecondaryDBsHandler& handler ) = 0;
    virtual uint32 numSecondaryDBs() = 0;       // 阻塞
    virtual int clearSecondaryDBs() = 0;        // 阻塞

    // 锁/解锁（启动时自动锁，合并时临时解锁）
    virtual bool lockDB() = 0;
    virtual bool unlockDB() = 0;
};
```

> **设计要点**：几乎所有操作都是异步的（通过 `IXxxHandler` 回调），这允许 MySQL 后端用线程池并发执行而不阻塞 dbmgr 主事件循环。`Database` 类自己只是 `IDatabase` 的"拦截代理"，在调用前后做额外处理（如 checkout 去重、mailbox 重映射、过载判断）。

### 4.4 BillingSystem：鉴权抽象

[billing_system.hpp](file:///workspace/src/lib/dbmgr_lib/billing_system.hpp)：

```cpp
class BillingSystem
{
public:
    BillingSystem( const EntityDefs & entityDefs );

    // 把 user/pass 映射到 EntityKey，结果通过 handler 回调
    virtual void getEntityKeyForAccount(
        const std::string & username, const std::string & password,
        const Mercury::Address & clientAddr,
        IGetEntityKeyForAccountHandler & handler ) = 0;

    virtual void setEntityKeyForAccount( const std::string & username,
            const std::string & password, const EntityKey & ekey ) = 0;

    virtual bool isOkay() const
    {
        return (entityTypeIDForUnknownUsers_ != INVALID_ENTITY_TYPE_ID) ||
            (!shouldAcceptUnknownUsers_ && !authenticateViaBaseEntity_);
    }

protected:
    bool shouldAcceptUnknownUsers_;        // 是否接受未知用户（自动建号）
    bool shouldRememberUnknownUsers_;      // 是否记住新用户
    bool authenticateViaBaseEntity_;       // 是否用 base 实体属性做密码校验
    EntityTypeID entityTypeIDForUnknownUsers_;
    std::string entityTypeForUnknownUsers_;
};
```

`IGetEntityKeyForAccountHandler` 有四种回调，对应"已有账号 / 用 username 加载 / 创建新实体 / 拒绝"四条分支（见 4.7 节登录时序）。dbmgr 默认用 `PyBillingSystem`（Python 实现的 `BWPersonality`），可通过 `USE_CUSTOM_BILLING_SYSTEM` 编译选项换成 `CustomBillingSystem`。

### 4.5 DBStatus 状态机

[db_status.hpp](file:///workspace/src/lib/dbmgr_lib/db_status.hpp)：

```cpp
class DBStatus
{
public:
    enum Status
    {
        STARTING,                // 启动中
        STARTUP_CONSOLIDATING,   // 启动期合并二级库
        WAITING_FOR_APPS,        // 等待 baseappmgr/cellappmgr 就绪
        RESTORING_STATE,         // 恢复实体、space 等
        RUNNING,                 // 正常运行
        SHUTTING_DOWN,           // 停机中
        SHUTDOWN_CONSOLIDATING   // 停机期合并二级库
    };
    // ...
};
```

状态机迁移：

```
STARTING
   │ (init 完成，pDatabase_->startup OK)
   ▼
STARTUP_CONSOLIDATING  ──(若 shouldConsolidate_)──▶  consolidate_dbs 子进程
   │ (合并完成)
   ▼
WAITING_FOR_APPS  ◀────(isRecover 时直接进入)────  reviver 拉起的恢复实例
   │ (baseappmgr/cellappmgr 都 ready，发 initData)
   ▼
RESTORING_STATE  ──▶  autoLoadEntities（从 DB 加载上次活跃实体）
   │ (恢复完成)
   ▼
RUNNING  ◀──── 登录/写库正常服务
   │ (收到 controlledShutDown)
   ▼
SHUTTING_DOWN
   │
   ▼
SHUTDOWN_CONSOLIDATING  ──▶  consolidate_dbs 子进程
   │
   ▼
退出
```

`status_` 通过 watcher 暴露，外部工具可查询 `status` 与 `statusDetail` 字段。`hasStartBegun()` 用于判断是否已过 `WAITING_FOR_APPS`，`isReady()` 判断是否至少进入 `WAITING_FOR_APPS`。

### 4.6 dbmgr 初始化流程

[database.cpp](file:///workspace/src/server/dbmgr/database.cpp) 的 `init` 主要步骤：

```cpp
bool Database::init( int argc, char * argv[] )
{
    if (!this->ServerApp::init( argc, argv )) return false;
    if (!interface_.isGood()) { ERROR_MSG("..."); return false; }

    ReviverSubject::instance().init( &interface_, "dbMgr" );   // 注册 reviver 监控

    // 1. 初始化 Python（dbmgr 也跑 Python，用于 personality/billing 脚本）
    PyImportPaths paths;
    paths.addPath( EntityDef::Constants::databasePath() );
    paths.addPath( EntityDef::Constants::serverCommonPath() );
    if (!Script::init( paths, "database", true )) return false;
    if (!PyNetwork::init( mainDispatcher_, interface_ )) return false;
    if (Personality::import( Config::personality() )) Personality::callOnInit();

    // 2. 加载实体定义（entities.xml），计算 digest（用于客户端 defs 校验）
    pEntityDefs_ = new EntityDefs();
    pEntityDefs_->init( BWResource::openSection( EntityDef::Constants::entitiesFile() ) );

    // 3. 注册 watcher
    BW_REGISTER_WATCHER( 0, "dbmgr", "DBMgr", "dbMgr", mainDispatcher_, interface_.address() );

    // 4. 选择数据库后端
    std::string databaseType = BWConfig::get( "dbMgr/type", "xml" );
    if (databaseType == "xml") {
        pDatabase_ = new XMLDatabase();
        shouldConsolidate_ = false;          // XML 库不需要合并
    } else if (databaseType == "mysql") {
        pDatabase_ = createMySqlDatabase( this->interface(), this->mainDispatcher() );
    }

    // 5. 监听 BaseAppMgrInterface birth，尝试找已存在的 baseappmgr
    Mercury::MachineDaemon::registerBirthListener( interface_.address(),
            DBInterface::handleBaseAppMgrBirth, "BaseAppMgrInterface" );
    Mercury::Address baseAppMgrAddr;
    if (Mercury::MachineDaemon::findInterface( "BaseAppMgrInterface", 0,
                baseAppMgrAddr ) == Mercury::REASON_SUCCESS)
    {
        baseAppMgr_.addr( baseAppMgrAddr );
    }

    // 6. 注册 DBInterface，监听自己的 birth
    DBInterface::registerWithInterface( interface_ );
    Mercury::MachineDaemon::registerBirthListener( interface_.address(),
            DBInterface::handleDatabaseBirth, "DBInterface" );

    // 7. 判断是否是恢复模式（baseappmgr 已经在跑说明是 reviver 拉起的恢复实例）
    bool isRecover = false;
    if (baseAppMgr_.addr() != Mercury::Address::NONE)
    {
        Mercury::BlockingReplyHandlerWithResult<bool> handler( interface_ );
        Mercury::Bundle & bundle = baseAppMgr_.bundle();
        bundle.startRequest( BaseAppMgrInterface::requestHasStarted, &handler );
        baseAppMgr_.send();
        if (handler.waitForReply( &baseAppMgr_.channel() ) == Mercury::REASON_SUCCESS)
            isRecover = handler.get();
        shouldSendInitData_ = !isRecover;
    }

    // 8. 注册到 machined
    DBInterface::registerWithMachined( interface_, 0 );

    // 9. 启动数据库后端
    pDatabase_->startup( this->getEntityDefs(), mainDispatcher_, isRecover );

    // 10. 初始化计费系统
    this->initBillingSystem();

    // 11. 初始化二级库前缀（runID）
    if (shouldConsolidate_) this->initSecondaryDBPrefix();

    // 12. 根据模式进入对应状态
    if (isRecover)                          this->startServerBegin( true );
    else if (shouldConsolidate_)            this->consolidateData();
    else                                    status_.set( DBStatus::WAITING_FOR_APPS, "..." );

    // 13. 1 秒周期状态检查 timer
    statusCheckTimerHandle_ = mainDispatcher_.addTimer( 1000000, this );

    return true;
}
```

`initSecondaryDBPrefix` 生成一个 runID（用户名_年月日_时分秒），作为本次运行的二级库文件前缀，便于停机时定位合并：

```cpp
void Database::initSecondaryDBPrefix()
{
    time_t epochTime = ::time( NULL );
    tm timeAndDate; localtime_r( &epochTime, &timeAndDate );
    uid_t uid = getuid();
    passwd * pUserDetail = getpwuid( uid );
    std::string username = pUserDetail ? pUserDetail->pw_name : ...;

    char runIDBuf[ BUFSIZ ];
    snprintf( runIDBuf, sizeof(runIDBuf), "%s_%04d%02d%02d_%02d%02d%02d",
            username.c_str(),
            timeAndDate.tm_year + 1900, timeAndDate.tm_mon + 1, timeAndDate.tm_mday,
            timeAndDate.tm_hour, timeAndDate.tm_min, timeAndDate.tm_sec );
    secondaryDBPrefix_ = runIDBuf;
}
```

### 4.7 登录全链路：logOn 与 LoginHandler

登录是 dbmgr 最核心的流程。入口 `Database::logOn`（[database.cpp](file:///workspace/src/server/dbmgr/database.cpp)）：

```cpp
void Database::logOn( const Mercury::Address & srcAddr,
        const Mercury::UnpackedMessageHeader & header, BinaryIStream & data )
{
    Mercury::Address addrForProxy;
    bool offChannel;
    LogOnParamsPtr pParams = new LogOnParams();
    data >> addrForProxy >> offChannel >> *pParams;

    // 1. 校验 defs digest（防止客户端用不一致的实体定义登录）
    const MD5::Digest & digest = pParams->digest();
    bool goodDigest = (digest == this->getEntityDefs().getDigest());
    if (!goodDigest && Config::allowEmptyDigest() && digest.isEmpty())
        goodDigest = true;
    if (!goodDigest) {
        this->sendFailure( header.replyID, srcAddr, offChannel,
            LogOnStatus::LOGIN_REJECTED_BAD_DIGEST, "Defs digest mismatch." );
        return;
    }

    this->logOn( srcAddr, header.replyID, pParams, addrForProxy, offChannel );
}

void Database::logOn( const Mercury::Address & srcAddr, Mercury::ReplyID replyID,
        LogOnParamsPtr pParams, const Mercury::Address & addrForProxy,
        bool offChannel )
{
    // 2. 服务未就绪
    if (status_.status() != DBStatus::RUNNING) {
        this->sendFailure( replyID, srcAddr, offChannel,
            LogOnStatus::LOGIN_REJECTED_SERVER_NOT_READY, "Server not ready." );
        return;
    }

    // 3. dbmgr 自身过载
    bool isOverloaded = curLoad_ > Config::overloadLevel();
    if (this->calculateOverloaded( isOverloaded )) {
        this->sendFailure( replyID, srcAddr, offChannel,
            LogOnStatus::LOGIN_REJECTED_DBMGR_OVERLOAD, "DBMgr is overloaded." );
        return;
    }

    // 4. 任一 cellapp 过载
    if (anyCellAppOverloaded_) {
        this->sendFailure( replyID, srcAddr, offChannel,
            LogOnStatus::LOGIN_REJECTED_CELLAPP_OVERLOAD, "..." );
        return;
    }

    // 5. 创建 LoginHandler，开始异步流程
    LoginHandler * pHandler = new LoginHandler( pParams, addrForProxy, srcAddr,
            offChannel, replyID );
    pHandler->login();
}
```

`LoginHandler`（[login_handler.hpp](file:///workspace/src/server/dbmgr/login_handler.hpp)）是登录流程的状态机，串联 BillingSystem → IDatabase → BaseAppMgr 三个异步环节：

```cpp
class LoginHandler : public Mercury::ReplyMessageHandler,
                     public IGetEntityKeyForAccountHandler,
                     public GetEntityHandler,
                     public IDatabase::IPutEntityHandler
{
public:
    LoginHandler( LogOnParamsPtr pParams, const Mercury::Address & clientAddr,
            const Mercury::Address & replyAddr, bool offChannel,
            Mercury::ReplyID replyID );
    void login();

    // IGetEntityKeyForAccountHandler 回调
    virtual void onGetEntityKeyForAccountSuccess( const EntityKey & ekey );
    virtual void onGetEntityKeyForAccountLoadFromUsername( EntityTypeID typeID,
            const std::string & username, bool shouldCreateNewOnLoadFailure );
    virtual void onGetEntityKeyForAccountCreateNew( EntityTypeID typeID,
            bool shouldRemember );
    virtual void onGetEntityKeyForAccountFailure( LogOnStatus status,
            const std::string & errorMsg );

    // GetEntityHandler 回调
    virtual void onGetEntityCompleted( bool isOK, const EntityDBKey & entityKey,
                const EntityMailBoxRef * pBaseEntityLocation );

    // PutEntityHandler 回调
    void onPutEntityComplete( bool isOK, DatabaseID dbID );
    void onReservedBaseMailbox( bool isOK, DatabaseID dbID );
    void onSetBaseMailbox( bool isOK, DatabaseID dbID );

private:
    void handleFailure( BinaryIStream * pData, LogOnStatus reason );
    void checkOutEntity();
    void createNewEntity( EntityTypeID entityTypeID, bool shouldRemember );
    void loadEntity( const EntityDBKey & ekey );
    void sendCreateEntityMsg();      // 发到 baseappmgr
    void sendReply();
    void sendFailureReply( LogOnStatus status, const char * msg = NULL );

    EntityKey           ekey_;
    LogOnParamsPtr      pParams_;
    Mercury::Address    clientAddr_;     // 客户端地址（经 loginapp 透传）
    Mercury::Address    replyAddr_;      // 回复地址（loginapp）
    bool                offChannel_;
    Mercury::ReplyID    replyID_;
    Mercury::Bundle     bundle_;
    EntityMailBoxRef    baseRef_;
    EntityMailBoxRef *  pBaseRef_;
    PutEntityHandler    putEntityHandler_;
    PutEntityHandler    reserveBaseMailboxHandler_;
    PutEntityHandler    setBaseMailboxHandler_;
    DatabaseID *        pStrmDbID_;
    bool                shouldCreateNewOnLoadFailure_;
};
```

登录时序（端到端，含 loginapp/baseappmgr/baseapp）见第 6 节。dbmgr 内部的状态机如下：

```
LoginHandler::login()
   │
   │ BillingSystem::getEntityKeyForAccount(user, pass, clientAddr, *this)
   ▼
   ┌───────────────┬───────────────────────────┬───────────────────────┐
   │               │                           │                       │
   ▼               ▼                           ▼                       ▼
onGetEntityKey     onGetEntityKey               onGetEntityKey          onGetEntityKey
ForAccountSuccess  ForAccountLoadFromUsername   ForAccountCreateNew     ForAccountFailure
   │               │                           │                       │
   │ loadEntity    │ loadEntity(用 username)    │ createNewEntity       │ sendFailureReply
   ▼               ▼                           │                       ▼
Database::getEntity                           │                     (结束)
   │                                           │
   ▼                                           │
onGetEntityCompleted                          ▼
   │                                       IDatabase::putEntity
   ├─ 实体已在线?                              │
   │   └─ onLogOnLoggedOnUser              onPutEntityComplete
   │      (走 RelogonAttemptHandler)           │
   │                                           ▼
   │                                       sendCreateEntityMsg()
   │ 实体不在线?                                │
   │   └─ sendCreateEntityMsg()                 │ BaseAppMgrInterface::createEntity
   │       │                                   ▼
   │       │ BaseAppMgrInterface::createEntity  handleMessage(reply)
   │       ▼                                       │
   │   handleMessage(reply)                        │ 成功 → baseRef 已设置
   │       │                                       │
   │       │ 成功：拿到 baseapp 外网地址            │
   │       ▼                                       ▼
   │   setBaseEntityLocation (putEntity)        sendReply()
   │       │                                       │
   │       ▼                                       ▼
   │   onSetBaseMailbox                        (返回 LoginReplyRecord 给 loginapp)
   │       │
   │       ▼
   │   sendReply()
   ▼
(结束)
```

`LogOnParams`（[log_on_params.hpp](file:///workspace/src/lib/connection/log_on_params.hpp)）封装客户端登录参数（username/password/encryptionKey/digest/nonce），从 loginapp 一路透传到 baseapp：

```cpp
class LogOnParams : public SafeReferenceCount
{
public:
    typedef uint8 Flags;
    static const Flags HAS_DIGEST = 0x1;
    static const Flags HAS_ALL = 0x1;
    static const Flags PASS_THRU = 0xFF;

    bool addToStream( BinaryOStream & data, Flags flags = PASS_THRU,
        const StreamEncoder * pEncoder = NULL ) const;
    bool readFromStream( BinaryIStream & data, const StreamEncoder * pEncoder = NULL );

    const std::string & username() const;
    const std::string & password() const;
    const std::string & encryptionKey() const;
    const MD5::Digest & digest() const;
    // ...
};
```

`LoginReplyRecord`（[login_reply_record.hpp](file:///workspace/src/lib/connection/login_reply_record.hpp)）是登录成功的回复体，只有两个字段：

```cpp
struct LoginReplyRecord
{
    Mercury::Address serverAddr;   // send to here（baseapp 外网地址）
    uint32           sessionKey;   // use this session key
};
```

### 4.8 重登录：RelogonAttemptHandler

当 `getEntity` 返回的 `pBaseEntityLocation` 表明实体已经在线（mailbox 活跃），dbmgr 走重登录路径，发 `BaseAppIntInterface::logOnAttempt` 给原 baseapp，由 `RelogonAttemptHandler` 等待结果（[relogon_attempt_handler.hpp](file:///workspace/src/server/dbmgr/relogon_attempt_handler.hpp)）：

```cpp
class RelogonAttemptHandler : public Mercury::ReplyMessageHandler,
                             public TimerHandler
{
public:
    RelogonAttemptHandler( EntityTypeID entityTypeID, DatabaseID dbID,
            const Mercury::Address & replyAddr, bool offChannel,
            Mercury::ReplyID replyID, LogOnParamsPtr pParams,
            const Mercury::Address & addrForProxy );

    virtual void handleMessage( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data, void * arg );
    virtual void handleException( const Mercury::NubException & exception, void * arg );
    virtual void handleShuttingDown( const Mercury::NubException & exception, void * arg );

    void onEntityLogOff();
    virtual void handleTimeout( TimerHandle handle, void * pUser );

private:
    void terminateRelogonAttempt( const char *clientMessage );
    void abort();

    bool                hasAborted_;
    EntityDBKey         ekey_;
    Mercury::Address    replyAddr_;
    bool                offChannel_;
    Mercury::ReplyID    replyID_;
    LogOnParamsPtr      pParams_;
    Mercury::Address    addrForProxy_;
    Mercury::Bundle     replyBundle_;
    TimerHandle         waitForDestroyTimer_;
};
```

`pendingAttempts_` map（`EntityKey → RelogonAttemptHandler*`）防止同一实体的重登录并发：若已有进行中的重登录，新的登录请求直接返回 `LOGIN_REJECTED_ALREADY_LOGGED_IN`。

### 4.9 实体写入：writeEntity 与 WriteEntityHandler

baseapp 周期性把实体归档到 dbmgr（`DBInterface::writeEntity`），dbmgr 用 `WriteEntityHandler` 处理（[write_entity_handler.hpp](file:///workspace/src/server/dbmgr/write_entity_handler.hpp)）：

```cpp
class WriteEntityHandler : public IDatabase::IPutEntityHandler,
                           public IDatabase::IDelEntityHandler
{
public:
    WriteEntityHandler( const EntityDBKey ekey, EntityID entityID,
            int8 flags, bool shouldReply, Mercury::ReplyID replyID,
            const Mercury::Address & srcAddr );

    void writeEntity( BinaryIStream & data, EntityID entityID );
    void deleteEntity();

    virtual void onPutEntityComplete( bool isOK, DatabaseID );
    virtual void onDelEntityComplete( bool isOK );

private:
    void putEntity( BinaryIStream * pStream, UpdateAutoLoad updateAutoLoad,
            EntityMailBoxRef * pBaseMailbox = NULL, bool removeBaseMailbox = false );
    void finalise( bool isOK );

    EntityDBKey          ekey_;
    EntityID             entityID_;
    int8                 flags_;          // cell? base? log off?
    bool                 shouldReply_;
    Mercury::ReplyID     replyID_;
    const Mercury::Address srcAddr_;
};
```

`flags` 区分这次写入是"只写 cell 数据"、"只写 base 数据"还是"玩家下线触发写入"（带 logoff 标志）。

### 4.10 二级库合并：Consolidator

[consolidator.hpp](file:///workspace/src/server/dbmgr/consolidator.hpp) / [consolidator.cpp](file:///workspace/src/server/dbmgr/consolidator.cpp) 实现了 `consolidate_dbs` 子进程管理：

```cpp
class Consolidator : public Mercury::InputNotificationHandler,
        public SignalHandler
{
public:
    Consolidator( Database & database );
    virtual ~Consolidator();

    bool onChildProcessExit( int status );
    pid_t pid() const { return pid_; }
    virtual int handleInputNotification( int fd );

private:
    bool startProcess();
    void execInFork( int readFD, int writeFD );
    void kill();
    void closeFile();
    Mercury::EventDispatcher & dispatcher();
    bool readFromChildProcess();
    void outputErrorLogs();
    virtual void handleSignal( int sigNum );

    Database &      database_;
    pid_t           pid_;
    std::stringstream stderrOutput_;
    FILE *          pStdErr_;
};
```

合并流程：

```cpp
bool Consolidator::startProcess()
{
    database_.getIDatabase().unlockDB();   // 临时解锁，让子进程能访问主库

    int filedes[2];
    pipe( filedes );                       // 捕获子进程 stderr

    if ((pid_ = fork()) == 0)
        this->execInFork( filedes[0], filedes[1] );   // 子进程

    pStdErr_ = fdopen( filedes[0], "r" );
    this->dispatcher().registerFileDescriptor( filedes[0], this );   // 监听 stderr
    close( filedes[1] );
    return true;
}

void Consolidator::execInFork( int readFD, int writeFD )
{
    // 构造 --res 参数（资源路径）
    // 重定向 stderr 到管道
    // chdir 到 exe 目录
    // execv "commands/consolidate_dbs"
}
```

子进程退出时（`SIGCHLD`），`handleSignal` 调用 `onChildProcessExit`：

```cpp
bool Consolidator::onChildProcessExit( int status )
{
    bool isOK = false;
    if (WIFEXITED( status ))
    {
        int exitCode = WEXITSTATUS( status );
        if (exitCode == 0) isOK = true;
        else if (exitCode == CONSOLIDATE_DBS_EXEC_FAILED_EXIT_CODE)
            ERROR_MSG("... consolidate_dbs 不存在 ...");
        else
            ERROR_MSG("... consolidate_dbs 退出码 %d ...", exitCode);
    }
    else if (WIFSIGNALED( status ))
        ERROR_MSG("... 被信号 %d 终止 ...", WTERMSIG( status ));

    // 重新获取 DB 锁（最多重试 20 次）
    int attempt = 0;
    const int MAX_ATTEMPTS = 20;
    while (!database_.getIDatabase().lockDB() && attempt < MAX_ATTEMPTS)
    {
        WARNING_MSG("... 重试 (%d/%d) ...", ++attempt, MAX_ATTEMPTS);
        sleep(1);
    }
    return isOK;
}
```

`Database::onConsolidateProcessEnd` 收到结果后推进状态机（启动期进入 `WAITING_FOR_APPS`，停机期完成退出）。

### 4.11 自动加载：EntityAutoLoader

停机恢复时，dbmgr 把"上次活跃的实体"重新加载回 baseappmgr（[entity_auto_loader.hpp](file:///workspace/src/server/dbmgr/entity_auto_loader.hpp)）：

```cpp
class EntityAutoLoader : public IEntityAutoLoader
{
public:
    EntityAutoLoader();
    virtual void reserve( int numEntities );
    virtual void start();
    virtual void abort();
    virtual void addEntity( EntityTypeID entityTypeID, DatabaseID dbID );
    virtual void onAutoLoadEntityComplete( bool isOK );

private:
    void checkFinished();
    bool sendNext();
    bool allSent() const { return numSent_ >= int(entities_.size()); }

    typedef std::vector< std::pair< EntityTypeID, DatabaseID > > Entities;
    Entities    entities_;
    int         numOutstanding_;
    int         numSent_;
    bool        hasErrors_;
};
```

`IDatabase::autoLoadEntities(autoLoader)` 把所有 `autoLoad` 标记为 true 的实体通过 `addEntity` 加入，然后 `start` 开始批量 `getEntity` + 路由到 baseappmgr。

### 4.12 dbmgr 接口：DBInterface

[db_interface.hpp](file:///workspace/src/server/dbmgr/db_interface.hpp)：

```cpp
BEGIN_MERCURY_INTERFACE( DBInterface )

    MF_REVIVER_PING_MSG()                          // reviver 心跳

    BW_BEGIN_STRUCT_MSG( Database, handleBaseAppMgrBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( Database, shutDown )
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( Database, controlledShutDown )
        ShutDownStage stage;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( Database, cellAppOverloadStatus )
        bool anyOverloaded;
    END_STRUCT_MESSAGE()

    BW_STREAM_MSG_EX( Database, logOn )             // loginapp → dbmgr
        // std::string logOnName / password
        // Mercury::Address addrForProxy
        // bool offChannel
        // MD5::Digest digest

    BW_STREAM_MSG_EX( Database, loadEntity )        // baseapp → dbmgr
        // EntityTypeID entityTypeID; EntityID entityID; bool nameNotID;
        // nameNotID ? (std::string name) : (DatabaseID id);

    BW_BIG_STREAM_MSG_EX( Database, writeEntity )   // baseapp → dbmgr
        // int8 flags; EntityTypeID entityTypeID; DatabaseID databaseID;
        // properties

    BW_BEGIN_STRUCT_MSG_EX( Database, deleteEntity )
        EntityTypeID entityTypeID;
        DatabaseID   dbid;
    END_STRUCT_MESSAGE()

    BW_STREAM_MSG_EX( Database, deleteEntityByName )
    BW_BEGIN_STRUCT_MSG_EX( Database, lookupEntity )
        EntityTypeID entityTypeID;
        DatabaseID   dbid;
        bool         offChannel;
    END_STRUCT_MESSAGE()
    BW_STREAM_MSG_EX( Database, lookupEntityByName )
    BW_STREAM_MSG_EX( Database, lookupDBIDByName )
    BW_BIG_STREAM_MSG_EX( Database, executeRawCommand )
    BW_STREAM_MSG_EX( Database, putIDs )
    BW_STREAM_MSG_EX( Database, getIDs )
    BW_BIG_STREAM_MSG_EX( Database, writeSpaces )

    BW_BEGIN_STRUCT_MSG( Database, writeGameTime )
        GameTime gameTime;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( Database, handleDatabaseBirth )
        Mercury::Address addr;
    END_STRUCT_MESSAGE()

    BW_STREAM_MSG_EX( Database, handleBaseAppDeath )    // baseappmgr → dbmgr
    BW_STREAM_MSG_EX( Database, checkStatus )
    BW_STREAM_MSG_EX( Database, secondaryDBRegistration );   // baseapp → dbmgr
    BW_STREAM_MSG_EX( Database, updateSecondaryDBs );
    BW_STREAM_MSG_EX( Database, getSecondaryDBDetails );

END_MERCURY_INTERFACE()
```

注意 `BW_ANONYMOUS_CHANNEL_CLIENT_MSG( DBInterface )` 出现在 baseappmgr/cellappmgr/loginapp 的接口里——这意味着 baseappmgr/cellappmgr/loginapp 都"持有"一个匿名 channel 指向 dbmgr，可以无回复地向 dbmgr 发消息（如 `logOn`、`writeEntity`、`checkStatus`）。

### 4.13 dbmgr 配置

[dbmgr_config.hpp](file:///workspace/src/server/dbmgr/dbmgr_config.hpp) / [dbmgr_config.cpp](file:///workspace/src/server/dbmgr/dbmgr_config.cpp)：

```cpp
class DBMgrConfig : public ServerAppConfig
{
public:
    static ServerAppOption< bool > allowEmptyDigest;            // 允许空 digest（开发期）
    static ServerAppOption< int > dumpEntityDescription;        // dump 实体定义
    static ServerAppOption< uint32 > desiredBaseApps;           // 期望 baseapp 数
    static ServerAppOption< uint32 > desiredCellApps;           // 期望 cellapp 数
    static ServerAppOption< float > overloadLevel;              // 过载阈值
    static ServerAppOption< float > overloadTolerancePeriod;    // 过载容忍期
    static ServerAppOption< std::string > type;                 // "xml" / "mysql"

    static uint64 overloadTolerancePeriodInStamps() { return secondsToStamps( overloadTolerancePeriod() ); }
    static bool postInit();
};

// dbmgr_config.cpp
BW_OPTION( bool, allowEmptyDigest, false );
BW_OPTION_RO( int, dumpEntityDescription, 0 );
BW_OPTION_AT( uint32, desiredBaseApps, 1, "" );
BW_OPTION_AT( uint32, desiredCellApps, 1, "" );
BW_OPTION( float, overloadLevel, 1.f );
BW_OPTION( float, overloadTolerancePeriod, 5.f );
BW_OPTION_RO( std::string, type, "xml" );
```

> **生产警告**：`Database::init` 里会检查——若 `type == "xml"` 且 `Config::isProduction()`，直接 ERROR，因为 XML 库只适合演示。

### 4.14 dbmgr 过载保护

`calculateOverloaded`（[database.cpp](file:///workspace/src/server/dbmgr/database.cpp)）：

```cpp
bool Database::calculateOverloaded( bool isOverloaded )
{
    if (!isOverloaded)
    {
        overloadStartTime_ = 0;
        return false;
    }
    if (overloadStartTime_ == 0) overloadStartTime_ = timestamp();
    uint64 overloadTime = timestamp() - overloadStartTime_;
    return (overloadTime >= Config::overloadTolerancePeriodInStamps());
}
```

与 baseappmgr 的版本相比，dbmgr 没有 `overloadLogins` 计数，纯粹靠时间窗口。`anyCellAppOverloaded_` 由 cellappmgr 通过 `cellAppOverloadStatus` 消息更新——任一 cellapp 过载时 dbmgr 直接拒绝登录（`LOGIN_REJECTED_CELLAPP_OVERLOAD`）。

### 4.15 mailbox 重映射

baseapp 死亡后，dbmgr 里的实体 mailbox（指向死亡 baseapp）需要重映射到新 baseapp。`mailboxRemapInfo_` 保存 `死亡 baseapp 地址 → BackupHash` 映射：

```cpp
typedef std::map< Mercury::Address, BackupHash > MailboxRemapInfo;
MailboxRemapInfo mailboxRemapInfo_;
```

`remapMailbox` 在 `getEntity`/`lookupEntity` 返回 mailbox 前调用，把指向死亡 baseapp 的地址改写到备份 baseapp。`IDatabase::remapEntityMailboxes` 则批量更新数据库里的 mailbox 字段。

---

## 5. loginapp — 客户端登录入口

### 5.1 模块定位

`loginapp` 是客户端访问服务端的**第一个进程**，也是唯一对公网暴露的非业务进程。它：

1. **持有外网 UDP 接口** `extInterface_`，接受客户端的登录请求。
2. **不做业务鉴权**：把登录请求（用户名/密码/digest）透传给 dbmgr，由 dbmgr 的 `BillingSystem` 鉴权。
3. **限流**：通过 `loginRateLimit` / `rateLimitDuration` 做基于时间窗的登录限流，防止登录风暴。
4. **缓存成功登录**：`cachedLoginMap_` 缓存最近的登录成功结果，应对"回复丢失、客户端重发"场景。
5. **可水平扩展**：可以部署多个 loginapp 实例，客户端通过 bwmachined 的负载均衡选择。

> **关键洞察**：loginapp 本身**不持有任何持久状态**——它只是一个无状态的反向代理 + 限流器 + 加密协商器。所有状态都在 dbmgr。这意味着 loginapp 崩溃/重启对在线玩家无影响。

### 5.2 核心类：LoginApp

[loginapp.hpp](file:///workspace/src/server/loginapp/loginapp.hpp)：

```cpp
typedef Mercury::ChannelOwner DBMgr;

class LoginApp : public ServerApp, public Singleton< LoginApp >
{
public:
    SERVER_APP_HEADER( LoginApp, loginApp )
    typedef LoginAppConfig Config;

    LoginApp( Mercury::EventDispatcher & mainDispatcher,
            Mercury::NetworkInterface & interface );
    ~LoginApp();

    // ---- 外部方法（来自客户端）----
    virtual void login( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    virtual void probe( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data );

    // ---- 内部方法 ----
    void controlledShutDown( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data );

    const NetMask & netMask() const { return netMask_; }
    uint32 externalIPFor( uint32 ip ) const;

    void sendFailure( const Mercury::Address & addr, Mercury::ReplyID replyID,
        int status, const char * msg = NULL, LogOnParamsPtr pParams = NULL );
    void sendAndCacheSuccess( const Mercury::Address & addr,
            Mercury::ReplyID replyID, const LoginReplyRecord & replyRecord,
            LogOnParamsPtr pParams );

    Mercury::NetworkInterface & intInterface() { return interface_; }
    Mercury::NetworkInterface & extInterface() { return extInterface_; }

    DBMgr & dbMgr() { return *dbMgr_.pChannelOwner(); }
    bool isDBReady() const { return this->dbMgr().channel().isEstablished(); }

    void controlledShutDown();

    uint8 systemOverloaded() const { return systemOverloaded_; }
    void systemOverloaded( uint8 status )
    {
        systemOverloaded_ = status;
        systemOverloadedTime_ = timestamp();
    }

private:
    virtual bool init( int argc, char * argv[] );
    virtual bool run();
    virtual void onSignalled( int sigNum );

    // 缓存的登录项（应对回复丢失）
    class CachedLogin
    {
    public:
        CachedLogin() : creationTime_( 0 ) {}
        bool isTooOld() const;
        bool isPending() const;
        void pParams( LogOnParamsPtr pParams ) { pParams_ = pParams; }
        LogOnParamsPtr pParams() { return pParams_; }
        void replyRecord( const LoginReplyRecord & record );
        const LoginReplyRecord & replyRecord() const { return replyRecord_; }
        void reset() { creationTime_ = 0; }
    private:
        uint64 creationTime_;
        LogOnParamsPtr pParams_;
        LoginReplyRecord replyRecord_;
    };

    bool handleResentPendingAttempt( const Mercury::Address & addr, Mercury::ReplyID replyID );
    bool handleResentCachedAttempt( const Mercury::Address & addr,
            LogOnParamsPtr pParams, Mercury::ReplyID replyID );
    void sendSuccess( const Mercury::Address & addr, Mercury::ReplyID replyID,
            const LoginReplyRecord & replyRecord, const std::string & encryptionKey );

    std::auto_ptr< StreamEncoder > pLogOnParamsEncoder_;   // 登录参数加密器
    Mercury::NetworkInterface      extInterface_;          // 外网接口

    NetMask    netMask_;
    uint32     externalIP_;
    typedef std::map< uint32, uint32 > ExternalIPs;
    ExternalIPs externalIPs_;                              // 多网卡外网 IP 映射

    uint8      systemOverloaded_;                          // 系统过载状态
    uint64     systemOverloadedTime_;

    typedef std::map< Mercury::Address, CachedLogin > CachedLoginMap;
    CachedLoginMap cachedLoginMap_;                        // 缓存的成功登录

    AnonymousChannelClient dbMgr_;                         // 到 dbmgr 的匿名 channel

    // 限流状态
    uint64 lastRateLimitCheckTime_;                        // 当前时间窗起点
    uint    numAllowedLoginsLeft_;                         // 本时间窗剩余配额

    static LoginApp * pInstance_;
    static const uint32 UPDATE_STATS_PERIOD;

    // 登录统计（watcher 暴露）
    class LoginStats: public TimerHandler { /* EMA 平均值 */ };
    LoginStats  loginStats_;
    TimerHandle statsTimer_;
};
```

### 5.3 双网络接口

loginapp 与其它进程不同，它有**两个** `NetworkInterface`：

- `interface_`（继承自 `ServerApp`）：内部接口，与 dbmgr/cellappmgr/baseappmgr 通信。
- `extInterface_`：外部接口，监听公网 UDP 端口，接受客户端登录。

外网接口的配置（`externalInterface`、`externalLatencyMin/Max`、`externalLossRatio`）来自 `ExternalAppConfig` 基类（[loginapp_config.hpp](file:///workspace/src/server/loginapp/loginapp_config.hpp)）。

### 5.4 登录限流

限流基于"时间窗 + 配额"（[loginapp_config.hpp](file:///workspace/src/server/loginapp/loginapp_config.hpp)）：

```cpp
class LoginAppConfig : public ServerAppConfig, public ExternalAppConfig
{
public:
    static ServerAppOption< bool > shouldShutDownIfPortUsed;
    static ServerAppOption< bool > verboseExternalInterface;
    static ServerAppOption< float > maxLoginDelay;          // 单次登录最大延迟

    static ServerAppOption< std::string > privateKey;       // RSA 私钥
    static ServerAppOption< bool > shutDownSystemOnExit;
    static ServerAppOption< bool > allowLogin;              // 是否允许登录
    static ServerAppOption< bool > allowProbe;              // 是否允许探测
    static ServerAppOption< bool > logProbes;
    static ServerAppOption< bool > registerExternalInterface;
    static ServerAppOption< bool > allowUnencryptedLogins;  // 允许明文登录

    static ServerAppOption< int > loginRateLimit;           // 每时间窗允许登录数
    static ServerAppOption< int > rateLimitDuration;        // 时间窗长度（秒）

    static uint64 rateLimitDurationInStamps() { return secondsToStamps( rateLimitDuration() ); }
    static bool rateLimitEnabled() { return (rateLimitDuration() > 0); }
    static bool postInit();
};
```

配置默认值（[loginapp_config.cpp](file:///workspace/src/server/loginapp/loginapp_config.cpp)）：

```cpp
BW_OPTION_RO( bool, shouldShutDownIfPortUsed, true );
BW_OPTION_RO( bool, verboseExternalInterface, false );
BW_OPTION( float, maxLoginDelay, 10.f );
BW_OPTION_RO( std::string, privateKey, "server/loginapp.privkey" );
BW_OPTION( bool, shutDownSystemOnExit, false );
BW_OPTION( bool, allowLogin, false );            // 默认拒绝登录（启动期）
BW_OPTION( bool, allowProbe, false );
BW_OPTION( bool, logProbes, true );
BW_OPTION_RO( bool, registerExternalInterface, false );
BW_OPTION( bool, allowUnencryptedLogins, false );
BW_OPTION( int, loginRateLimit, 0 );             // 0 = 不限流
BW_OPTION( int, rateLimitDuration, 0 );
```

限流逻辑（推断自 `lastRateLimitCheckTime_` / `numAllowedLoginsLeft_` 字段）：

```
loginapp::login(请求)
   │
   │ 1. 检查 allowLogin（启动期 dbmgr 未 ready 时为 false）
   │ 2. 检查 maxLoginDelay（请求太老直接拒）
   │ 3. 限流检查：
   │    if (rateLimitEnabled())
   │        if (now - lastRateLimitCheckTime_ > rateLimitDurationInStamps())
   │            lastRateLimitCheckTime_ = now
   │            numAllowedLoginsLeft_ = loginRateLimit
   │        if (numAllowedLoginsLeft_ == 0)
   │            loginStats_.incRateLimited()
   │            return sendFailure(LOGIN_REJECTED_RATE_LIMITED)
   │        --numAllowedLoginsLeft_
   │ 4. 检查 CachedLogin：是否是重发的 pending 或已成功登录
   │ 5. 转发到 dbmgr（DBInterface::logOn）
   ▼
dbmgr 异步处理，回复到达后 loginapp 调用 sendAndCacheSuccess / sendFailure
```

### 5.5 CachedLogin：应对回复丢失

`CachedLogin` 缓存最近的登录结果。当客户端因为 UDP 丢包没收到登录回复而重发时，loginapp 直接返回缓存结果：

```cpp
class CachedLogin
{
public:
    CachedLogin() : creationTime_( 0 ) {}    // creationTime_==0 表示 pending
    bool isTooOld() const;                   // 超过 maxLoginDelay 视为过期
    bool isPending() const;                  // 还在等 dbmgr 回复
    void pParams( LogOnParamsPtr pParams );
    LogOnParamsPtr pParams();
    void replyRecord( const LoginReplyRecord & record );
    const LoginReplyRecord & replyRecord() const;
    void reset() { creationTime_ = 0; }      // 重置为 pending
private:
    uint64 creationTime_;
    LogOnParamsPtr pParams_;
    LoginReplyRecord replyRecord_;
};
```

`handleResentPendingAttempt` 处理"dbmgr 还没回复时客户端重发"，`handleResentCachedAttempt` 处理"dbmgr 已回复但客户端没收到"。

### 5.6 LoginStats：登录统计

```cpp
class LoginStats: public TimerHandler
{
public:
    void incRateLimited()  { ++all_.value(); ++rateLimited_.value(); }
    void incFails()        { ++all_.value(); ++fails_.value(); }
    void incPending()      { ++all_.value(); ++pending_.value(); }
    void incSuccesses()    { ++all_.value(); ++successes_.value(); }

    float fails() const        { return fails_.average(); }
    float rateLimited() const  { return rateLimited_.average(); }
    float pending() const      { return pending_.average(); }
    float successes() const    { return successes_.average(); }
    float all() const          { return all_.average(); }

    void update() { /* sample 所有 EMA */ }
private:
    AccumulatingEMA< uint32 > fails_;
    AccumulatingEMA< uint32 > rateLimited_;
    AccumulatingEMA< uint32 > pending_;
    AccumulatingEMA< uint32 > successes_;
    AccumulatingEMA< uint32 > all_;
    static const float BIAS;
};
```

用 EMA（指数移动平均）平滑统计，通过 watcher 暴露给运维工具。

### 5.7 loginapp 接口：LoginIntInterface

[login_int_interface.hpp](file:///workspace/src/server/loginapp/login_int_interface.hpp)：

```cpp
BEGIN_MERCURY_INTERFACE( LoginIntInterface )

    BW_ANONYMOUS_CHANNEL_CLIENT_MSG( DBInterface )   // 复用 DBInterface 匿名 channel

    MERCURY_EMPTY_MESSAGE( controlledShutDown, &gShutDownHandler )   // 受控停机

    MF_REVIVER_PING_MSG()                            // reviver 心跳

END_MERCURY_INTERFACE()
```

loginapp 的内部接口很简洁——它主要是 dbmgr 的客户端，自己只暴露 `controlledShutDown` 与 `reviverPing`。客户端登录用的是另一个外网接口（`ExternalInterface`，定义在客户端连接库里，不在本仓库 server 目录）。

### 5.8 probe：探测接口

`probe` 方法（[loginapp.hpp](file:///workspace/src/server/loginapp/loginapp.hpp)）允许客户端在不真正登录的情况下探测服务端状态（用于负载均衡选择最优 loginapp）。受 `allowProbe` 配置控制，`logProbes` 控制是否记录探测日志。

### 5.9 systemOverloaded：紧急限流

`systemOverloaded_` 是一个 `uint8`，由 dbmgr 通过某条消息（推断为 `LoginIntInterface` 的扩展或 watcher 转发）设置。当系统过载时，loginapp 直接拒绝新登录，无需往返 dbmgr。`systemOverloadedTime_` 记录过载触发时间，用于判断是否已恢复。

---

## 6. 登录全链路时序（端到端）

把上述四个进程串起来，一次完整的登录流程：

```
客户端                loginapp              dbmgr                baseappmgr           baseapp              cellapp
  │                     │                     │                     │                   │                    │
  │ 1. loginRequest     │                     │                     │                   │                    │
  │  (username,pass,    │                     │                     │                   │                    │
  │   digest,encKey)    │                     │                     │                   │                    │
  ├────────────────────▶│                     │                     │                   │                    │
  │                     │ 2. 限流检查         │                     │                   │                    │
  │                     │    allowLogin?      │                     │                   │                    │
  │                     │    rateLimit?       │                     │                   │                    │
  │                     │    CachedLogin?     │                     │                   │                    │
  │                     │                     │                     │                   │                    │
  │                     │ 3. DBInterface::    │                     │                   │                    │
  │                     │    logOn            │                     │                   │                    │
  │                     ├────────────────────▶│                     │                   │                    │
  │                     │                     │ 4. digest 校验      │                   │                    │
  │                     │                     │    status==RUNNING? │                   │                    │
  │                     │                     │    overload?        │                   │                    │
  │                     │                     │                     │                   │                    │
  │                     │                     │ 5. new LoginHandler │                   │                    │
  │                     │                     │    BillingSystem::  │                   │                    │
  │                     │                     │    getEntityKeyFor  │                   │                    │
  │                     │                     │    Account          │                   │                    │
  │                     │                     │    ──▶ IGetEntityKeyForAccountHandler   │                    │
  │                     │                     │                     │                   │                    │
  │                     │                     │ 6. 分支：           │                   │                    │
  │                     │                     │  a) 已有账号 →      │                   │                    │
  │                     │                     │     IDatabase::     │                   │                    │
  │                     │                     │     getEntity       │                   │                    │
  │                     │                     │  b) 创建新实体 →    │                   │                    │
  │                     │                     │     IDatabase::     │                   │                    │
  │                     │                     │     putEntity       │                   │                    │
  │                     │                     │  c) 已在线 →        │                   │                    │
  │                     │                     │     onLogOnLogged   │                   │                    │
  │                     │                     │     OnUser          │                   │                    │
  │                     │                     │     ──▶ Relogon     │                   │                    │
  │                     │                     │         Attempt     │                   │                    │
  │                     │                     │         Handler     │                   │                    │
  │                     │                     │                     │                   │                    │
  │                     │                     │ 7. (a/b 分支)       │                   │                    │
  │                     │                     │    BaseAppMgrInter  │                   │                    │
  │                     │                     │    face::createEnt  │                   │                    │
  │                     │                     │    ity              │                   │                    │
  │                     │                     ├────────────────────▶│                   │                    │
  │                     │                     │                     │ 8. findBestBaseApp │                   │
  │                     │                     │                     │    overload?      │                   │
  │                     │                     │                     │                   │                    │
  │                     │                     │                     │ 9. BaseAppIntInt  │                   │
  │                     │                     │                     │    erface::create │                   │
  │                     │                     │                     │    BaseWithCellDa │                   │
  │                     │                     │                     │    ta            │                   │
  │                     │                     │                     ├──────────────────▶│                    │
  │                     │                     │                     │                   │ 10. 创建 Base     │
  │                     │                     │                     │                   │     + Proxy       │
  │                     │                     │                     │                   │     ──▶ cellappmgr│
  │                     │                     │                     │                   │      createEntity │
  │                     │                     │                     │                   │     ◀─────────────│
  │                     │                     │                     │                   │ 11. 创建 cell 实体│
  │                     │                     │                     │                   │     base 回复     │
  │                     │                     │                     │ 12. reply         │                    │
  │                     │                     │                     │   (baseAppAddr)   │                    │
  │                     │                     │◀────────────────────┤                   │                    │
  │                     │                     │ 13. setBaseEntity   │                   │                    │
  │                     │                     │     Location        │                   │                    │
  │                     │                     │     (putEntity)     │                   │                    │
  │                     │                     │ 14. 回复 LoginReply │                   │                    │
  │                     │                     │     Record          │                   │                    │
  │                     │◀────────────────────┤                   │                    │                    │
  │                     │ 15. sendAndCache    │                   │                    │                    │
  │                     │     Success         │                   │                    │                    │
  │                     │     (serverAddr=    │                   │                    │                    │
  │                     │      baseapp外网,   │                   │                    │                    │
  │                     │      sessionKey)    │                   │                    │                    │
  │ 16. loginReply      │                     │                   │                    │                    │
  │◀────────────────────┤                     │                   │                    │                    │
  │                     │                     │                     │                   │                    │
  │ 17. 直连 baseapp 外网地址，建立 UDP Channel                  │                   │                    │
  ├──────────────────────────────────────────────────────────────────────────────────▶│                    │
  │                                                                                   │ 18. Proxy 接入    │
  │                                                                                   │     进入 AOI       │
```

### 6.1 LogOnStatus 枚举

[log_on_status.hpp](file:///workspace/src/lib/connection/log_on_status.hpp) 定义了所有登录结果码：

```cpp
class LogOnStatus
{
public:
    enum Status
    {
        // 客户端状态值（0-63）
        NOT_SET, LOGGED_ON, CONNECTION_FAILED, DNS_LOOKUP_FAILED, UNKNOWN_ERROR,
        CANCELLED, ALREADY_ONLINE_LOCALLY, PUBLIC_KEY_LOOKUP_FAILED,
        LAST_CLIENT_SIDE_VALUE = 63,

        // 服务端状态值（64-255）
        LOGIN_MALFORMED_REQUEST,
        LOGIN_BAD_PROTOCOL_VERSION,
        LOGIN_REJECTED_NO_SUCH_USER,
        LOGIN_REJECTED_INVALID_PASSWORD,
        LOGIN_REJECTED_ALREADY_LOGGED_IN,
        LOGIN_REJECTED_BAD_DIGEST,
        LOGIN_REJECTED_DB_GENERAL_FAILURE,
        LOGIN_REJECTED_DB_NOT_READY,
        LOGIN_REJECTED_ILLEGAL_CHARACTERS,
        LOGIN_REJECTED_SERVER_NOT_READY,
        LOGIN_REJECTED_UPDATER_NOT_READY,      // 已废弃
        LOGIN_REJECTED_NO_BASEAPPS,
        LOGIN_REJECTED_BASEAPP_OVERLOAD,
        LOGIN_REJECTED_CELLAPP_OVERLOAD,
        LOGIN_REJECTED_BASEAPP_TIMEOUT,
        LOGIN_REJECTED_BASEAPPMGR_TIMEOUT,
        LOGIN_REJECTED_DBMGR_OVERLOAD,
        LOGIN_REJECTED_LOGINS_NOT_ALLOWED,
        LOGIN_REJECTED_RATE_LIMITED,

        LOGIN_CUSTOM_DEFINED_ERROR = 254,
        LAST_SERVER_SIDE_VALUE = 255
    };
    // ...
};
```

注意过载相关有三个独立状态：`LOGIN_REJECTED_BASEAPP_OVERLOAD`（baseappmgr 拒）、`LOGIN_REJECTED_CELLAPP_OVERLOAD`（dbmgr 因 cellapp 过载拒）、`LOGIN_REJECTED_DBMGR_OVERLOAD`（dbmgr 自己过载）。三层独立限流，任一层拒绝都不会把请求往下传。

---

## 7. reviver — 看门狗

### 7.1 模块定位

`reviver` 是 BigWorld 的看门狗进程，**每台机器一个**。它监控 cellappmgr / baseappmgr / dbmgr / loginapp 这四个"控制面"进程，发现死亡时通过 bwmachined 拉起新实例。

> **关键洞察**：reviver **不监控 baseapp/cellapp**——这两个业务进程由 baseappmgr/cellappmgr 自己监控（baseappmgr 的 `checkForDeadBaseApps`、cellappmgr 的对应逻辑）。reviver 只盯"控制面"，因为控制面挂了整服就瘫了。这种分层监控避免了"看门狗自己被业务进程拖垮"。

### 7.2 核心类：Reviver

[reviver.hpp](file:///workspace/src/server/reviver/reviver.hpp)：

```cpp
class Reviver : public ServerApp, public TimerHandler,
    public Singleton< Reviver >
{
public:
    SERVER_APP_HEADER( Reviver, reviver )
    typedef ReviverConfig Config;

    Reviver( Mercury::EventDispatcher & mainDispatcher,
            Mercury::NetworkInterface & interface );
    virtual ~Reviver();

    bool queryMachinedSettings();        // 查询本机 bwmachined 的 Components tags
    void shutDown();
    void revive( const char * createComponent );   // 通知 bwmachined 拉起进程
    bool hasEnabledComponents() const;

    virtual void handleTimeout( TimerHandle handle, void * arg );
    void markAsDirty() { isDirty_ = true; }         // 标记需要重新输出摘要

    // 处理 bwmachined 的 tags 回复
    class TagsHandler : public MachineGuardMessage::ReplyHandler
    {
    public:
        TagsHandler( Reviver &reviver ) : reviver_( reviver ) {}
        virtual bool onTagsMessage( TagsMessage &tm, uint32 addr );
    private:
        Reviver &reviver_;
    };

private:
    virtual bool init( int argc, char * argv[] );
    virtual bool run();

    enum TimeoutType { TIMEOUT_REATTACH };

    TimerHandle          timerHandle_;
    ComponentRevivers    components_;     // 监控的组件列表
    bool                 shuttingDown_;
    bool                 isDirty_;        // 是否需要重新输出摘要
};
```

### 7.3 ComponentReviver：单组件监控

[component_reviver.hpp](file:///workspace/src/server/reviver/component_reviver.hpp) 是单个被监控组件的抽象：

```cpp
class ComponentReviver : public Mercury::ShutdownSafeReplyMessageHandler,
    public TimerHandler,
    public Mercury::InputMessageHandler,
    public IntrusiveObject< ComponentReviver >
{
public:
    ComponentReviver( const char * configName, const char * name,
            const char * interfaceName, const char * createParam );

    bool init( Mercury::EventDispatcher & dispatcher,
            Mercury::NetworkInterface & interface );
    void revive();

    bool activate( ReviverPriority priority );   // 激活监控
    bool deactivate();                           // 停止监控

    // 处理 birth/death 消息
    virtual void handleMessage( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data );
    // 处理 ping 回复
    virtual void handleMessage( const Mercury::Address & source,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data, void * arg );

    virtual void handleTimeout( TimerHandle /*handle*/, void * /*arg*/ );
    virtual void handleException( const Mercury::NubException & ne, void * arg );

    ReviverPriority priority() const { return priority_; }
    void priority( ReviverPriority p ) { priority_ = p; }

    bool isAttached() const { return isAttached_; }
    const std::string & name() const { return name_; }
    const Mercury::Address & addr() const { return addr_; }

    bool isEnabled() const { return isEnabled_; }
    void isEnabled( bool v ) { isEnabled_ = v; }

    const std::string & configName() const { return configName_; }
    const char * createName() const { return createParam_; }
    int maxPingsToMiss() const { return maxPingsToMiss_; }
    int pingPeriod() const { return pingPeriod_; }

protected:
    virtual void initInterfaceElements() = 0;   // 子类绑定 birth/death/ping 消息

    const Mercury::InterfaceElement * pBirthMessage_;
    const Mercury::InterfaceElement * pDeathMessage_;
    const Mercury::InterfaceElement * pPingMessage_;

private:
    Mercury::EventDispatcher * pDispatcher_;
    Mercury::NetworkInterface * pInterface_;
    Mercury::Address addr_;                      // 被监控进程地址

    std::string  configName_;                    // 配置名（如 "cellAppMgr"）
    std::string  name_;                          // 显示名（如 "CellAppMgr"）
    std::string  interfaceName_;                 // 接口名（如 "CellAppMgrInterface"）
    const char * createParam_;                   // 创建参数（如 "cellappmgr"）

    ReviverPriority priority_;                   // 优先级（多 reviver 时用）

    TimerHandle timerHandle_;
    int pingsToMiss_;                            // 还能 miss 几次 ping
    int maxPingsToMiss_;                         // 阈值
    int pingPeriod_;                             // ping 周期（微秒）

    bool isAttached_;                            // 是否已确认 attached
    bool isEnabled_;                             // 是否启用监控
};
```

### 7.4 监控的四类组件

[component_reviver.cpp](file:///workspace/src/server/reviver/component_reviver.cpp) 末尾用宏生成四个子类：

```cpp
#define MF_REVIVER_HANDLER( CONFIG, COMPONENT, CREATE_WHAT )             \
    MF_REVIVER_HANDLER2( CONFIG, COMPONENT, COMPONENT, CREATE_WHAT )

#define MF_REVIVER_HANDLER2( CONFIG, COMPONENT, COMPONENT2, CREATE_WHAT ) \
class COMPONENT##Reviver : public ComponentReviver                       \
{                                                                        \
public:                                                                  \
    COMPONENT##Reviver() :                                               \
        ComponentReviver( #CONFIG, #COMPONENT, #COMPONENT2 "Interface",  \
                CREATE_WHAT )                                            \
    {}                                                                   \
    virtual void initInterfaceElements()                                 \
    {                                                                    \
        pBirthMessage_ = &ReviverInterface::handle##COMPONENT##Birth;    \
        pDeathMessage_ = &ReviverInterface::handle##COMPONENT##Death;    \
        pPingMessage_ = &COMPONENT2##Interface::reviverPing;             \
    }                                                                    \
} g_reviverOf##COMPONENT;

MF_REVIVER_HANDLER( cellAppMgr, CellAppMgr, "cellappmgr" )
MF_REVIVER_HANDLER( baseAppMgr, BaseAppMgr, "baseappmgr" )
MF_REVIVER_HANDLER( dbMgr,      DB,         "dbmgr" )
MF_REVIVER_HANDLER2( loginApp,   Login, LoginInt,   "loginapp" )
```

每个子类绑定三个消息：
- `ReviverInterface::handleXXXBirth`：组件出生通知（来自 bwmachined）。
- `ReviverInterface::handleXXXDeath`：组件死亡通知。
- `XXXInterface::reviverPing`：向组件发的心跳请求。

### 7.5 ReviverInterface

[reviver_interface.hpp](file:///workspace/src/server/reviver/reviver_interface.hpp)：

```cpp
#define BW_REVIVER_MSGS( COMPONENT )                                      \
    MERCURY_FIXED_MESSAGE( handle##COMPONENT##Birth,                      \
                            sizeof( Mercury::Address ),                   \
                            &g_reviverOf##COMPONENT )                     \
    MERCURY_FIXED_MESSAGE( handle##COMPONENT##Death,                      \
                            sizeof( Mercury::Address ),                   \
                            &g_reviverOf##COMPONENT )                     \

BEGIN_MERCURY_INTERFACE( ReviverInterface )

    BW_REVIVER_MSGS( CellAppMgr )
    BW_REVIVER_MSGS( BaseAppMgr )
    BW_REVIVER_MSGS( DB )
    BW_REVIVER_MSGS( Login )

END_MERCURY_INTERFACE()
```

birth/death 消息的 handler 直接绑定到对应的 `g_reviverOfXXX` 全局实例，由 `ComponentReviver::handleMessage` 处理。

### 7.6 监控状态机

`ComponentReviver` 的状态机：

```
                ┌─────────────────┐
                │  init()         │
                │  找到组件地址   │
                │  注册 birth/    │
                │  death 监听     │
                └────────┬────────┘
                         │ activate(priority)
                         ▼
                ┌─────────────────┐  收到 birth 消息     ┌─────────────┐
                │  activate       │ ─────────────────▶  │  更新 addr_  │
                │  pingsToMiss_=  │                     │  (不改变状态)│
                │  maxPingsToMiss_│                     └─────────────┘
                │  启动 ping timer│
                └────────┬────────┘
                         │ ping 回复 REVIVER_PING_YES
                         ▼
                ┌─────────────────┐
                │  isAttached_=   │
                │  true           │ ◀── 持续 ping ──┐
                └────────┬────────┘                │
                         │                         │
            ┌────────────┼────────────┐            │
            │            │            │            │
            ▼            ▼            ▼            │
     ping 超过        ping 回复     收到 death     │
     maxPingsToMiss   REVIVER_NO   消息且 addr    │
            │            │         匹配           │
            │            │            │            │
            └────────────┴────────────┘            │
                         │                         │
                         ▼                         │
                ┌─────────────────┐                │
                │  revive()       │                │
                │  deactivate()   │                │
                │  addr_.clear()  │                │
                │  通知 bwmachined│                │
                │  拉起新进程     │                │
                └────────┬────────┘                │
                         │                         │
                         │ 新进程出生 birth 消息    │
                         ▼                         │
                ┌─────────────────┐                │
                │  addr_=新地址   │ ───────────────┘
                │  activate()     │
                └─────────────────┘
```

### 7.7 ping 机制

`handleTimeout` 周期发 ping（[component_reviver.cpp](file:///workspace/src/server/reviver/component_reviver.cpp)）：

```cpp
void ComponentReviver::handleTimeout( TimerHandle /*handle*/, void * /*arg*/ )
{
    if (pingsToMiss_ > 0)
    {
        --pingsToMiss_;
        Mercury::Bundle bundle;
        bundle.startRequest( *pPingMessage_, this );
        bundle << priority_;            // 携带本 reviver 的优先级
        pInterface_->send( addr_, bundle );
    }
    else
    {
        INFO_MSG( "ComponentReviver::handleTimeout: Missed too many\n" );
        this->revive();                 // 连续 miss 太多次，触发恢复
    }
}
```

被监控进程通过 `ReviverSubject`（[reviver_subject.hpp](file:///workspace/src/lib/server/reviver_subject.hpp)）处理 `reviverPing`，回复 `REVIVER_PING_YES`（1）表示存活。`priority_` 字段用于多 reviver 抢占场景——同一机器可能有多个 reviver 实例（高可用），优先级高的胜出。

回复处理：

```cpp
void ComponentReviver::handleMessage( const Mercury::Address & source,
    Mercury::UnpackedMessageHeader & header, BinaryIStream & data, void * arg )
{
    uint8 returnCode;
    data >> returnCode;
    if (returnCode == REVIVER_PING_YES)
    {
        pingsToMiss_ = maxPingsToMiss_;       // 重置 miss 计数
        if (!isAttached_)
        {
            Reviver::pInstance()->markAsDirty();
            INFO_MSG( "ComponentReviver: %s (%s) has attached.\n",
                addr_.c_str(), name_.c_str() );
            isAttached_ = true;
        }
    }
    else
    {
        this->deactivate();                   // 被监控方主动说不
    }
}
```

`REVIVER_PING_NO`（0）让被监控方主动 detach（如受控停机时）。

### 7.8 birth/death 处理

```cpp
void ComponentReviver::handleMessage( const Mercury::Address & source,
    Mercury::UnpackedMessageHeader & header, BinaryIStream & data )
{
    MF_ASSERT( (header.identifier == pBirthMessage_->id()) ||
                (header.identifier == pDeathMessage_->id()) );

    Mercury::Address addr;
    data >> addr;

    if (header.identifier == pBirthMessage_->id())
    {
        addr_ = addr;                         // 记录新进程地址
        INFO_MSG( "ComponentReviver::handleMessage: %s at %s has started.\n",
            name_.c_str(), addr.c_str() );
        return;
    }

    INFO_MSG( "ComponentReviver::handleMessage: %s at %s has died.\n",
        name_.c_str(), addr.c_str() );

    if (addr == addr_)
    {
        this->revive();                       // 是我监控的进程死了，恢复
    }
    else if (isAttached_)
    {
        ERROR_MSG( "ComponentReviver::handleMessage: "
                "%s component died at %s. Expected %s\n",
            name_.c_str(), addr.c_str(), addr_.c_str() );
    }
}
```

### 7.9 revive：通知 bwmachined 拉起进程

[reviver.cpp](file:///workspace/src/server/reviver/reviver.cpp)：

```cpp
void Reviver::revive( const char * createComponent )
{
    if (shuttingDown_)
    {
        INFO_MSG( "Reviver::revive: Trying to revive a process while shutting down.\n" );
        return;
    }

    CreateMessage cm;
    cm.uid_ = getUserId();
    cm.recover_ = 1;                          // 标记是恢复（让新进程走 -recover 路径）
    cm.name_ = createComponent;
#ifdef _DEBUG
    cm.config_ = "Debug";
#endif
#ifdef _HYBRID
    cm.config_ = "Hybrid";
#endif
#ifdef _RELEASE
    cm.config_ = "Release";
#endif

    uint32 srcaddr = 0, destaddr = htonl( 0x7f000001U );   // 发给本机 bwmachined
    if (cm.sendAndRecv( srcaddr, destaddr ) != Mercury::REASON_SUCCESS)
    {
        ERROR_MSG( "ComponentReviver::revive: Could not send request.\n" );
    }

    if (Config::shutDownOnRevive())
    {
        shuttingDown_ = true;
        this->shutDown();                     // 拉起后自身退出（默认行为）
    }
}
```

> **设计要点**：`shutDownOnRevive` 默认 true——reviver 触发一次恢复后就自杀，让 bwmachined 重新拉起一个新的 reviver。这避免了"reviver 自己状态错乱后持续作恶"。`recover_=1` 让新进程走 `-recover` 路径（如 dbmgr 的 `isRecover=true`、baseappmgr 的 `isRecovery_=true`），从已有状态恢复而不是冷启动。

### 7.10 reattach 周期与优先级重排

`Reviver::handleTimeout` 在每个 `reattachPeriod`（默认 10 秒）执行一次重排（[reviver.cpp](file:///workspace/src/server/reviver/reviver.cpp)）：

```cpp
void Reviver::handleTimeout( TimerHandle handle, void * arg )
{
    typedef std::map< ReviverPriority, ComponentReviver * > Map;

    switch (uintptr(arg))
    {
        case TIMEOUT_REATTACH:
        {
            Map activeSet;
            ComponentRevivers deactive;

            components_ = *g_pComponentRevivers;
            // 把已激活的按 priority 排序，未激活的收集到 deactive
            /* ... */

            // 重新分配连续的 priority（1, 2, 3, ...）
            {
                Map::iterator mapIter = activeSet.begin();
                ReviverPriority priority = 0;
                while (mapIter != activeSet.end())
                {
                    ++priority;
                    if (mapIter->first != priority)
                        mapIter->second->priority( priority );
                    ++mapIter;
                }

                // 未激活的随机洗牌后激活
                std::random_shuffle( deactive.begin(), deactive.end() );
                /* ... activate( ++priority ) ... */
            }

            // 输出当前 attached 组件摘要
            if (isDirty_) { /* INFO_MSG ... */ isDirty_ = false; }
        }
        break;
    }
}
```

### 7.11 reviver 配置

[reviver_config.hpp](file:///workspace/src/server/reviver/reviver_config.hpp) / [reviver_config.cpp](file:///workspace/src/server/reviver/reviver_config.cpp)：

```cpp
class ReviverConfig : public ServerAppConfig
{
public:
    static ServerAppOption< float > reattachPeriod;       // reattach 周期（秒）
    static ServerAppOption< float > pingPeriod;           // ping 周期（秒）
    static ServerAppOption< bool > shutDownOnRevive;      // 恢复后是否自杀
    static ServerAppOption< int > timeoutInPings;          // 连续 miss 多少 ping 触发恢复
};

// reviver_config.cpp
BW_OPTION_RO( float, reattachPeriod, 10.f );     // 每 10 秒重排
BW_OPTION( float, pingPeriod, 0.1f );            // 每 0.1 秒 ping 一次
BW_OPTION( bool, shutDownOnRevive, true );       // 默认恢复后自杀
BW_OPTION( int, timeoutInPings, 5 );             // 连续 5 次 miss 触发恢复
```

每个组件还可以单独覆盖（[component_reviver.cpp](file:///workspace/src/server/reviver/component_reviver.cpp)）：

```cpp
float pingPeriodInSeconds = BWConfig::get( (prefix + configName_ + "/pingPeriod").c_str(), -1.f );
if (pingPeriodInSeconds < 0.f) pingPeriodInSeconds = ReviverConfig::pingPeriod();
pingPeriod_ = int( pingPeriodInSeconds * 1000000 );

maxPingsToMiss_ = BWConfig::get( (prefix + configName_ + "/timeoutInPings").c_str(),
                                ReviverConfig::timeoutInPings() );
```

即 `<reviver><cellAppMgr><pingPeriod>` 可以单独为 cellappmgr 设置 ping 周期。

### 7.12 命令行：--add / --del

[main.cpp](file:///workspace/src/server/reviver/main.cpp) 支持 `--add` / `--del` 命令行参数，显式指定只监控哪些组件：

```cpp
void printHelp( const char * commandName )
{
    printf( "Usage: %s [OPTION]\n", commandName );
    printf( "Monitors BigWorld server components and spawns a new process if a component fails.\n\n"
"  --add {baseAppMgr|cellAppMgr|dbMgr|loginApp}\n"
"  --del {baseAppMgr|cellAppMgr|dbMgr|loginApp}\n" );
}
```

`Reviver::init` 解析命令行（[reviver.cpp](file:///workspace/src/server/reviver/reviver.cpp)）：

```cpp
for (int i = 1; i < argc - 1; ++i)
{
    const bool isAdd = (strcmp( argv[i], "--add" ) == 0);
    const bool isDel = (strcmp( argv[i], "--del" ) == 0);

    if (isAdd || isDel)
    {
        ++i;
        if (isAdd && isFirstAdd)
        {
            isFirstAdd = false;
            // --add 模式：先全部禁用，再启用指定的
            ComponentRevivers::iterator iter = components_.begin();
            while (iter != components_.end()) { (*iter)->isEnabled( false ); ++iter; }
        }
        // 找到对应组件，启用/禁用
        /* ... */
    }
}
if (!isFirstAdd && !isFirstDel)
{
    ERROR_MSG( "Cannot mix --add and --del command line options\n" );
    return false;
}
```

另外 `queryMachinedSettings` 会查询 bwmachined 的 `Components` tags，根据本机配置自动启用/禁用组件监控：

```cpp
bool Reviver::queryMachinedSettings()
{
    TagsMessage query;
    query.tags_.push_back( std::string( "Components" ) );
    TagsHandler handler( *this );
    int reason;
    if ((reason = query.sendAndRecv( 0, LOCALHOST, &handler )) != Mercury::REASON_SUCCESS)
    {
        ERROR_MSG( "Reviver::queryMachinedSettings: MGM query failed (%s)\n", ... );
        return false;
    }
    return true;
}
```

`TagsHandler::onTagsMessage` 把本机 bwmachined 配置的 `Components` tags 与组件列表比对，只启用 tags 里出现的组件。

### 7.13 ReviverSubject：被监控方

每个被监控进程（cellappmgr/baseappmgr/dbmgr/loginapp）在 `init` 时都调用 `ReviverSubject::instance().init( &interface_, "xxx" )`，让自己响应 `reviverPing`。`ReviverSubject`（[reviver_subject.hpp](file:///workspace/src/lib/server/reviver_subject.hpp)）是一个单例，处理 ping 消息并回复 `REVIVER_PING_YES`。

受控停机时，被监控方可以回复 `REVIVER_PING_NO` 让 reviver detach，避免在停机过程中被误恢复。

### 7.14 reviver 接口与消息流

```
bwmachined                  reviver                     被监控进程(cellappmgr/baseappmgr/dbmgr/loginapp)
    │                          │                                    │
    │  handleXXXBirth/Death    │                                    │
    │  (多播)                  │                                    │
    ├─────────────────────────▶│                                    │
    │                          │                                    │
    │                          │  reviverPing (携带 priority)        │
    │                          ├───────────────────────────────────▶│
    │                          │                                    │
    │                          │  REVIVER_PING_YES / NO             │
    │                          │◀───────────────────────────────────┤
    │                          │                                    │
    │  CreateMessage           │                                    │
    │  (recover=1)             │                                    │
    │◀─────────────────────────┤                                    │
    │                          │                                    │
    │  拉起新进程              │                                    │
    │──────────────────────────────────────────────────────────────▶│
    │                          │                                    │
    │  handleXXXBirth          │                                    │
    ├─────────────────────────▶│                                    │
    │                          │  activate() + ping                 │
    │                          ├───────────────────────────────────▶│
    │                          │                                    │
```

---

## 8. 受控停机全链路

整服受控停机由 dbmgr 发起（`controlledShutDown` 消息），通过 baseappmgr/cellappmgr 协同推进 stage：

```
dbmgr                    baseappmgr                   cellappmgr                  baseapp/cellapp
  │                          │                            │                          │
  │ 1. controlledShutDown    │                            │                          │
  │     (stage=...)          │                            │                          │
  ├─────────────────────────▶│                            │                          │
  │                          │ 2. controlledShutDown      │                          │
  │                          │    (stage, shutDownTime)   │                          │
  │                          ├───────────────────────────▶│                          │
  │                          │                            │                          │
  │                          │ 3. controlledShutDown      │                          │
  │                          │    (BaseAppIntInterface)   │                          │
  │                          ├──────────────────────────────────────────────────────▶│
  │                          │                            │                          │
  │                          │                            │ 4. controlledShutDown    │
  │                          │                            │    (CellApp)             │
  │                          │                            ├──────────────────────────▶│
  │                          │                            │                          │
  │                          │                            │                          │ 5. 完成最后归档
  │                          │                            │                          │    writeEntity
  │                          │                            │                          │◀─── dbmgr
  │                          │ 6. informOfArchiveComplete │                          │
  │                          │◀──────────────────────────────────────────────────────┤
  │                          │                            │                          │
  │                          │ 7. ackBaseAppsShutDown     │                          │
  │                          │◀───────────────────────────┤                          │
  │                          │                            │                          │
  │                          │ 8. ackCellAppShutDown      │                          │
  │                          │                            │◀─────────────────────────┤
  │                          │                            │                          │
  │ 9. 推进到下一 stage      │                            │                          │
  │                          │                            │                          │
  │ ... 重复直到 SHUTDOWN_DONE ...                                                       │
  │                                                                                     │
  │ 10. SHUTDOWN_CONSOLIDATING                                                         │
  │     consolidate_dbs 子进程                                                          │
  │     合并二级库                                                                       │
  │                                                                                     │
  │ 11. 退出                                                                            │
```

`ShutDownStage` 枚举定义在 [server/common.hpp](file:///workspace/src/lib/server/common.hpp)，由 `ServerApp::shutDownStageToString` 提供字符串化。每个 stage 都需要等所有子进程 ack 才能推进，`startAsyncShutDownStage` 处理异步等待。

---

## 9. 进程启动顺序与依赖

### 9.1 冷启动顺序

整个服务端的冷启动由 bwmachined 编排，但进程间有依赖关系：

```
bwmachined (机器守护，常驻)
   │
   │ 1. dbmgr 启动
   │    - init: 加载 entities.xml、选数据库后端、启动 Python
   │    - 注册 DBInterface、监听 BaseAppMgrInterface birth
   │    - 若 baseappmgr 未就绪，进入 WAITING_FOR_APPS
   ▼
dbmgr
   │ 2. baseappmgr 启动
   │    - init: ReviverSubject::init("baseAppMgr")
   │    - 注册 BaseAppMgrInterface、监听 CellAppMgrInterface birth
   │    - 找到 dbmgr 后建立 channel
   ▼
baseappmgr
   │ 3. cellappmgr 启动
   │    - init: ReviverSubject::init("cellAppMgr")
   │    - 注册 CellAppMgrInterface
   │    - 与 baseappmgr 互发 handleCellAppMgrBirth / handleBaseAppMgrBirth
   ▼
cellappmgr
   │ 4. baseapp 启动（多个）
   │    - 向 baseappmgr 发 add（带 addrForCells / addrForClients）
   │    - baseappmgr 回 BaseAppInitData（id, time, isReady）
   │    - 第一个 baseapp 注册后，baseappmgr 通知 cellappmgr setBaseApp
   ▼
baseapp #1..#N
   │ 5. cellapp 启动（多个）
   │    - 向 cellappmgr 发 addApp
   │    - cellappmgr 分配 cell、通知其它 cellapp
   ▼
cellapp #1..#N
   │ 6. loginapp 启动（多个）
   │    - 注册到 bwmachined
   │    - allowLogin 初始为 false
   ▼
loginapp #1..#N
   │ 7. dbmgr 检测到所有组件就绪
   │    - 发 initData 给 baseappmgr（game time 等）
   │    - 发 startup 给 baseappmgr / cellappmgr
   │    - allowLogin 置 true
   ▼
服务就绪，开始接受客户端登录
```

### 9.2 reviver 的位置

reviver 不在上面的启动链里——它在 bwmachined 启动后就可以启动，独立监控。reviver 启动后会：

1. `queryMachinedSettings` 查询本机 `Components` tags，确定监控哪些组件。
2. 对每个启用的 `ComponentReviver` 调用 `init` + `activate`。
3. 周期 ping 被监控进程。

如果 reviver 启动时被监控进程还没起来，`ComponentReviver::init` 里 `Mercury::MachineDaemon::findInterface` 会失败，但 reviver 会继续运行，等 birth 消息到达后再 activate。

### 9.3 恢复（recover）模式

当控制面进程崩溃，reviver 拉起新实例时带 `recover_=1`，新进程走恢复路径：

| 进程 | 恢复行为 |
| --- | --- |
| dbmgr | `isRecover=true`，跳过 consolidateData，直接 `startServerBegin(true)`，从已有 baseappmgr 探测状态 |
| baseappmgr | `isRecovery_=true`，接受 baseapp 的 `recoverBaseApp` 消息（带原 id、backupHash、sharedData、globalBases） |
| cellappmgr | 接受 cellapp 的 `recoverCellApp` 消息（恢复 space 状态） |
| loginapp | 无状态，直接重启即可 |

baseappmgr 恢复的关键：原有的 baseapp 会通过 `recoverBaseApp` 把自己的 id、backupHash、sharedBaseAppData、sharedGlobalData、globalBases 全部重新上报，让新 baseappmgr 重建状态。这部分代码在 [baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp) 的 `recoverBaseApp` 方法里（见 2.5 节引用）。

---

## 10. watcher 与可观测性

四个进程都通过 `BW_REGISTER_WATCHER` 注册 watcher，暴露内部状态给运维工具（如 `bwtest`、web watcher）。

### 10.1 baseappmgr watcher

[baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp) 的 `addWatchers`：

```cpp
void BaseAppMgr::addWatchers()
{
    Watcher * pRoot = &Watcher::rootWatcher();
    this->ServerApp::addWatchers( *pRoot );

    MF_WATCH( "numBaseApps", *this, &BaseAppMgr::numBaseApps );
    MF_WATCH( "numBases", *this, &BaseAppMgr::numBases );
    MF_WATCH( "numProxies", *this, &BaseAppMgr::numProxies );
    MF_WATCH( "config/shouldShutDownOthers", shouldShutDownOthers_ );

    MF_WATCH( "baseAppLoad/min", *this, &BaseAppMgr::minBaseAppLoad );
    MF_WATCH( "baseAppLoad/average", *this, &BaseAppMgr::avgBaseAppLoad );
    MF_WATCH( "baseAppLoad/max", *this, &BaseAppMgr::maxBaseAppLoad );

    WatcherPtr pBaseAppWatcher = BaseApp::pWatcher();
    pRoot->addChild( "baseApps", new MapWatcher<BaseApps>( baseApps_ ) );
    pRoot->addChild( "baseApps/*", new BaseDereferenceWatcher( pBaseAppWatcher ) );

    MF_WATCH( "lastBaseAppIDAllocated", lastBaseAppID_ );
    pRoot->addChild( "cellAppMgr", Mercury::Channel::pWatcher(), &cellAppMgr_.channel() );
    pRoot->addChild( "forwardTo", new BAForwardingWatcher() );   // 转发到子 baseapp
}
```

`forwardTo`（[watcher_forwarding_baseapp.hpp](file:///workspace/src/server/baseappmgr/watcher_forwarding_baseapp.hpp)）允许把 watcher 请求转发到具体 baseapp，实现"从 baseappmgr 查询某个 baseapp 的内部状态"。

### 10.2 dbmgr watcher

dbmgr 通过 `status_`（`DBStatus`）暴露 `status` 与 `statusDetail` 两个字段（`DBSTATUS_WATCHER_STATUS_DETAIL_PATH`），运维可查询当前状态机阶段。还暴露 `curLoad_`、`anyCellAppOverloaded_`、`isConsolidating` 等。

### 10.3 loginapp watcher

`LoginStats` 的 EMA 平均值（`fails`、`rateLimited`、`pending`、`successes`、`all`）通过 watcher 暴露，用于监控登录健康度。`systemOverloaded_` 也暴露，用于判断是否触发紧急限流。

---

## 11. 关键设计要点总结

### 11.1 控制面与数据面分离

BigWorld 把"调度/路由/容灾"（baseappmgr/cellappmgr/dbmgr/loginapp/reviver）与"业务承载"（baseapp/cellapp）严格分离。控制面进程都是单例（除 loginapp），不持有业务实体，可以快速重启；数据面进程多实例，承载实体，崩溃由控制面恢复。

### 11.2 三层独立限流

登录路径上有三层独立过载保护，任一层拒绝都不会把请求往下传：

| 层 | 进程 | 拒绝状态码 | 机制 |
| --- | --- | --- | --- |
| 1 | loginapp | `LOGIN_REJECTED_RATE_LIMITED` | 时间窗 + 配额（`loginRateLimit` / `rateLimitDuration`） |
| 2 | dbmgr | `LOGIN_REJECTED_DBMGR_OVERLOAD` / `LOGIN_REJECTED_CELLAPP_OVERLOAD` | `curLoad_` + `anyCellAppOverloaded_` + `overloadTolerancePeriod` |
| 3 | baseappmgr | `CREATE_ENTITY_ERROR_BASEAPPS_OVERLOADED` | `findBestBaseApp().load > baseAppOverloadLevel` + `overloadTolerancePeriod` + `overloadLogins` |

三层都有"容忍窗口"——短时尖峰不立即拒绝，避免误杀。

### 11.3 备份哈希环 + BackupHashChain

baseapp 之间的实体备份用哈希环路由（`BackupHash`），baseappmgr 维护历史死亡 baseapp 的哈希链（`BackupHashChain`），确保 baseapp 崩溃后能找到正确的备份方。双层哈希（`backupHash_` / `newBackupHash_`）支持平滑迁移：成员变化时先迁移到新哈希，迁移完成才提升。

### 11.4 reviver 的"恢复后自杀"策略

`shutDownOnRevive` 默认 true——reviver 触发一次恢复后就退出，让 bwmachined 拉起新 reviver。这避免了 reviver 自身状态错乱后持续作恶，是一种"自我牺牲"的容错设计。

### 11.5 异步 IDatabase

dbmgr 的 `IDatabase` 几乎所有操作都是异步的（通过 `IXxxHandler` 回调），允许 MySQL 后端用线程池并发而不阻塞主事件循环。`Database` 类自己只是 `IDatabase` 的拦截代理，在调用前后做 checkout 去重、mailbox 重映射、过载判断。

### 11.6 双网络接口

loginapp 是唯一有双 `NetworkInterface` 的进程：内部接口与控制面通信，外部接口监听公网。这种隔离让公网攻击面最小化——即使外网接口被攻破，攻击者也只能影响登录流程，无法直接访问内部协议。

### 11.7 game time 由 baseappmgr 推进

`TimeKeeper` 在 baseappmgr 侧，每个 game tick 自增 `time_`。cellappmgr 通过 `gameTimeReading` 贡献读数对齐。所有 baseapp/cellapp 的 game time 都来自 baseappmgr，保证全服时间一致。

---

## 12. 与其它文档的关联

- **[00-架构总览.md](file:///workspace/study-docs/00-架构总览.md)**：服务端整体架构与进程关系总览。
- **[06-网络层-Mercury.md](file:///workspace/study-docs/06-网络层-Mercury.md)**：Mercury 网络库，本文中所有 `ChannelOwner` / `AnonymousChannelClient` / `BW_STREAM_MSG` / `BEGIN_MERCURY_INTERFACE` 都基于此。
- **[08-服务端框架-server.md](file:///workspace/study-docs/08-服务端框架-server.md)**：`ServerApp` / `ManagerApp` / `ServerAppConfig` / `ReviverSubject` / `BackupHash` 等基础设施。
- **[09-服务端进程-baseapp.md](file:///workspace/study-docs/09-服务端进程-baseapp.md)**：baseapp 单体，本文 baseappmgr 是其控制面。
- **[10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)**：cellapp 单体，本文 cellappmgr 是其控制面。

---

## 13. 阅读源码的建议路径

1. 先读 [reviver_interface.hpp](file:///workspace/src/server/reviver/reviver_interface.hpp) + [component_reviver.cpp](file:///workspace/src/server/reviver/component_reviver.cpp) 末尾的宏——理解 reviver 监控哪些进程。
2. 读 [baseappmgr_interface.hpp](file:///workspace/src/server/baseappmgr/baseappmgr_interface.hpp) + [cellappmgr_interface.hpp](file:///workspace/src/server/cellappmgr/cellappmgr_interface.hpp) + [db_interface.hpp](file:///workspace/src/server/dbmgr/db_interface.hpp)——理解控制面之间的消息契约。
3. 读 [baseappmgr.hpp](file:///workspace/src/server/baseappmgr/baseappmgr.hpp) + [baseappmgr.cpp](file:///workspace/src/server/baseappmgr/baseappmgr.cpp) 的 `add` / `createEntity` / `onBaseAppDeath` / `controlledShutDownServer`——理解 baseapp 集群管理。
4. 读 [database.hpp](file:///workspace/src/server/dbmgr/database.hpp) + [database.cpp](file:///workspace/src/server/dbmgr/database.cpp) 的 `init` / `logOn` / `startServerBegin`——理解持久化与登录。
5. 读 [login_handler.hpp](file:///workspace/src/server/dbmgr/login_handler.hpp) + [login_handler.cpp](file:///workspace/src/server/dbmgr/login_handler.cpp)——理解登录状态机。
6. 读 [loginapp.hpp](file:///workspace/src/server/loginapp/loginapp.hpp) + [loginapp_config.hpp](file:///workspace/src/server/loginapp/loginapp_config.hpp)——理解登录入口。
7. 读 [reviver.hpp](file:///workspace/src/server/reviver/reviver.hpp) + [reviver.cpp](file:///workspace/src/server/reviver/reviver.cpp) + [component_reviver.hpp](file:///workspace/src/server/reviver/component_reviver.hpp) + [component_reviver.cpp](file:///workspace/src/server/reviver/component_reviver.cpp)——理解看门狗。
8. 读 [consolidator.hpp](file:///workspace/src/server/dbmgr/consolidator.hpp) + [consolidator.cpp](file:///workspace/src/server/dbmgr/consolidator.cpp)——理解二级库合并。
9. 读 [idatabase.hpp](file:///workspace/src/lib/dbmgr_lib/idatabase.hpp) + [billing_system.hpp](file:///workspace/src/lib/dbmgr_lib/billing_system.hpp) + [db_status.hpp](file:///workspace/src/lib/dbmgr_lib/db_status.hpp)——理解 dbmgr 的抽象层。
10. 读 [log_on_params.hpp](file:///workspace/src/lib/connection/log_on_params.hpp) + [login_reply_record.hpp](file:///workspace/src/lib/connection/login_reply_record.hpp) + [log_on_status.hpp](file:///workspace/src/lib/connection/log_on_status.hpp)——理解登录协议数据结构。

---

## 14. 已知未开放源码的部分

- **`CellAppMgr` 类实现**：`src/server/cellappmgr/` 只有接口定义，`CellAppMgr` 类的 `.cpp` 未随源码开放。本文第 3 节以接口契约 + cellapp/baseappmgr 侧调用反推其行为。
- **MySQL 后端**：`src/server/dbmgr_mysql/` 目录的 `mysql_database_creation.hpp` 在 `USE_MYSQL` 编译选项下才包含，本仓库默认未启用。
- **CustomBillingSystem**：`USE_CUSTOM_BILLING_SYSTEM` 编译选项才启用 [custom_billing_system.hpp](file:///workspace/src/server/dbmgr/custom_billing_system.hpp)，默认用 `PyBillingSystem`。
- **`consolidate_dbs` 子进程**：[consolidator.cpp](file:///workspace/src/server/dbmgr/consolidator.cpp) 调用的 `commands/consolidate_dbs` 可执行文件需要单独构建，不在标准包里。
- **部分 loginapp 实现**：`loginapp.cpp` 的部分实现未随源码开放（如 `login` / `probe` / `sendAndCacheSuccess` 的具体实现），本文以 `.hpp` 声明 + 字段语义推断其行为。

遇到不确定的实现细节时，请参考对应的 `.hpp` 声明与 Mercury 接口定义。
