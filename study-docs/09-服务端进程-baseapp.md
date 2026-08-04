# 09 服务端进程 — baseapp

> 仓库版本：BigWorld Technology 2.0.1
> 源码目录：[src/server/baseapp/](file:///workspace/src/server/baseapp/)
> 关联文档：[00-架构总览.md](file:///workspace/study-docs/00-架构总览.md)、[10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md)、[11-服务端进程-mgr-dbmgr-loginapp-reviver.md](file:///workspace/study-docs/11-服务端进程-mgr-dbmgr-loginapp-reviver.md)

---

## 1. 模块定位

`baseapp` 是 BigWorld 服务端三大"实体宿主进程"之一（另两个是 `cellapp` 与客户端）。它的核心定位可以用一句话概括：**baseapp 是玩家 base 实体（持久代理）的宿主进程，也是客户端的“直连”服务器**。

具体职责可以拆为以下五条：

1. **持有 Base 实体（持久代理）**：每个玩家/NPC/可持久的逻辑对象在 baseapp 侧都体现为一个 `Base` 实例。`Base` 持有 Python 脚本对象、与 cellapp 上对应 cell 实体的 `CellEntityMailBox`、以及向客户端投递事件的 `ClientEntityMailBox`。baseapp 是玩家“逻辑真身”的权威进程之一（cellapp 持有空间真身，baseapp 持有持久真身）。
2. **客户端直连入口**：客户端经 `loginapp` 鉴权后，会拿到一个 baseapp 的外网地址，并直接与该 baseapp 建立 UDP 通道（Mercury `Channel`）。这条通道承载玩家→服务端的方法调用、服务端→玩家的 AOI 推送、`avatarUpdate` 等高频消息。`Proxy`（`proxy.hpp`）就是 baseapp 内代表这条客户端连接的实体。
3. **与 dbmgr 协作完成持久化**：baseapp 通过 `WriteToDBReplyHandler`、`LoadEntityHandler`、`LoadingThread`、`CreateCellEntityHandler` 与 `dbmgr` 交互，完成实体的创建、加载、保存、删除。同时 baseapp 还可选维护一个本地 SQLite 二级库（`SqliteDatabase`），用于在主库不可达时本地落盘。
4. **跨 baseapp 备份与归档**：baseapp 把自己持有的每个 Base 实体按哈希环备份到另一个 baseapp（`BackupSender` + `BackedUpBaseApp`），并周期性把实体归档到 dbmgr（`Archiver`）。任一 baseapp 崩溃时，`baseappmgr` 通知其备份 baseapp 把实体恢复（`OffloadedBackups`、`BackupHashChain`）。
5. **baseapp 之间的内部通信**：通过 `BaseAppIntInterface`（[baseapp_int_interface.hpp](file:///workspace/src/server/baseapp/baseapp_int_interface.hpp)）这条 IDL 接口，baseapp 之间互相发送 `backupBaseEntity`、`handleBaseAppBirth`、`forwardedBaseMessage`、`setSharedData` 等消息。

> 关键洞察：baseapp 不做空间模拟。空间（位置、AOI、物理）由 `cellapp` 负责。baseapp 持有的 `Base` 通过 `CellEntityMailBox` 把“需要进世界的实体”委托给 cellapp 上的 cell 实体；反过来 cellapp 通过 `BaseAppIntInterface::currentCell` / `enterAoI` / `createCellPlayer` 等消息通知 baseapp 该 cell 实体的状态变化。baseapp 与 cellapp 是“逻辑/持久层”与“空间层”的解耦。

### 1.1 进程拓扑

```
                         ┌──────────────┐
                         │   dbmgr      │  MySQL / XML
                         │ (持久化总控)  │
                         └──────┬───────┘
                                │  loadEntity / writeEntity / lookUpDBID
                                ▼
   ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
   │  loginapp    │────▶│   baseappmgr  │◀───▶│  cellappmgr  │
   │ (登录入口)    │     │ (baseapp 注册 │     │ (cellapp 注册│
   └──────┬───────┘     │  + 路由 + 负载)│     │  + cell 划分)│
          │             └───────┬───────┘     └──────┬───────┘
          │ loginOK,baseappAddr │                    │
          │                     │ add/del/backupHash │ birth/death
          ▼                     ▼                    ▼
   ┌──────────────────────────────────────────────────────────┐
   │                       客户端                              │
   │   ◀──UDP Channel──▶ baseapp#A  ◀──BaseAppIntInterface──▶ baseapp#B
   │                         │                                  │
   │                         │ createCellEntity / currentCell   │
   │                         ▼                                  │
   │                     cellapp#1 … cellapp#N                  │
   └──────────────────────────────────────────────────────────┘
```

### 1.2 目录速览

baseapp 目录里**只有少量 .cpp**（`main.cpp`、`baseapp_int_interface.cpp`、`external_interfaces.cpp`、`eg_tcpecho.cpp`），绝大多数实现都被 BigWorld 编译系统放在了 `Hybrid64/*.o` 这种预编译产物里，源码侧以 `.hpp` 头文件为主。阅读时需要以 `.hpp` 为准（部分内联实现放在 `.ipp`，但 baseapp 的 `.ipp` 大多几乎为空，例如 [baseapp.ipp](file:///workspace/src/server/baseapp/baseapp.ipp)）。

| 文件 | 角色 |
| --- | --- |
| [main.cpp](file:///workspace/src/server/baseapp/main.cpp) | 进程入口，`BIGWORLD_MAIN` 宏 → `bwMainT<BaseApp>` |
| [baseapp.hpp](file:///workspace/src/server/baseapp/baseapp.hpp) | 核心单例 `BaseApp` 声明 |
| [baseapp.ipp](file:///workspace/src/server/baseapp/baseapp.ipp) | 内联实现占位（本版本基本为空） |
| [baseapp_config.hpp](file:///workspace/src/server/baseapp/baseapp_config.hpp) | 进程配置项（继承 `EntityAppConfig` + `ExternalAppConfig`） |
| [baseapp_int_interface.hpp](file:///workspace/src/server/baseapp/baseapp_int_interface.hpp) | baseapp 内部 IDL 接口（baseapp↔baseapp、cellapp→baseapp、dbmgr→baseapp） |
| [base.hpp](file:///workspace/src/server/baseapp/base.hpp) | `Base` 实体类（baseapp 侧的 entity 包装） |
| [bases.hpp](file:///workspace/src/server/baseapp/bases.hpp) | `Bases` 集合（EntityID → Base\*） |
| [global_bases.hpp](file:///workspace/src/server/baseapp/global_bases.hpp) | `GlobalBases`：跨进程的全局 base 字典 |
| [proxy.hpp](file:///workspace/src/server/baseapp/proxy.hpp) | `Proxy`：带客户端连接的特殊 `Base` |
| [entity_creator.hpp](file:///workspace/src/server/baseapp/entity_creator.hpp) | `EntityCreator`：创建 base（本地/远端/从 DB） |
| [entity_type.hpp](file:///workspace/src/server/baseapp/entity_type.hpp) | `EntityType`：实体类型元信息 + Python 类绑定 |
| [mailbox.hpp](file:///workspace/src/server/baseapp/mailbox.hpp) | `ServerEntityMailBox` / `CellEntityMailBox` / `BaseEntityMailBox` |
| [client_entity_mailbox.hpp](file:///workspace/src/server/baseapp/client_entity_mailbox.hpp) | `ClientEntityMailBox`：投递到客户端的 mailbox |
| [login_handler.hpp](file:///workspace/src/server/baseapp/login_handler.hpp) | `LoginHandler`：客户端经 loginapp 后接入 baseapp |
| [pending_logins.hpp](file:///workspace/src/server/baseapp/pending_logins.hpp) | `PendingLogins`：等待客户端直连确认的登录项 |
| [load_entity_handler.hpp](file:///workspace/src/server/baseapp/load_entity_handler.hpp) | `LoadEntityHandler`：dbmgr loadEntity 回复处理 |
| [loading_thread.hpp](file:///workspace/src/server/baseapp/loading_thread.hpp) | `WorkerThread` + `FileStreamingJob`（后台 IO） |
| [create_cell_entity_handler.hpp](file:///workspace/src/server/baseapp/create_cell_entity_handler.hpp) | `CreateCellEntityHandler`：cell 实体创建回复 |
| [write_to_db_reply.hpp](file:///workspace/src/server/baseapp/write_to_db_reply.hpp) | `WriteToDBReplyHandler` / `WriteToDBReplyStruct` |
| [worker_thread.hpp](file:///workspace/src/server/baseapp/worker_thread.hpp) | 后台 worker 线程 + `WorkerJob` |
| [archiver.hpp](file:///workspace/src/server/baseapp/archiver.hpp) | `Archiver`：周期归档到 dbmgr |
| [backed_up_base_app.hpp](file:///workspace/src/server/baseapp/backed_up_base_app.hpp) | `BackedUpBaseApp` / `BackedUpEntities`：本进程为别的 baseapp 持有的备份 |
| [backup_sender.hpp](file:///workspace/src/server/baseapp/backup_sender.hpp) | `BackupSender`：把本进程的 base 备份到哈希环上的另一个 baseapp |
| [offloaded_backups.hpp](file:///workspace/src/server/baseapp/offloaded_backups.hpp) | `OffloadedBackups`：retirement 期间为“卸载出去”的实体当备份 |
| [dead_cell_apps.hpp](file:///workspace/src/server/baseapp/dead_cell_apps.hpp) | `DeadCellApps`：最近死亡的 cellapp 列表（用于 ghost 切换） |
| [rate_limit_config.hpp](file:///workspace/src/server/baseapp/rate_limit_config.hpp) | `RateLimitConfig`：客户端消息限流配置 |
| [rate_limit_message_filter.hpp](file:///workspace/src/server/baseapp/rate_limit_message_filter.hpp) | `RateLimitMessageFilter` + `BufferedMessage`：每客户端限流器 |
| [proxy_buffered_message.hpp](file:///workspace/src/server/baseapp/proxy_buffered_message.hpp) | `ProxyBufferedMessage`：被限流缓存的客户端消息 |
| [proxy_rate_limit_callback.hpp](file:///workspace/src/server/baseapp/proxy_rate_limit_callback.hpp) | `ProxyRateLimitCallback`：限流器回调 |
| [baseappmgr_gateway.hpp](file:///workspace/src/server/baseapp/baseappmgr_gateway.hpp) | `BaseAppMgrGateway`：与 baseappmgr 通信的网关 |
| [add_to_baseappmgr_helper.hpp](file:///workspace/src/server/baseapp/add_to_baseappmgr_helper.hpp) | `AddToBaseAppMgrHelper`：启动时把自己注册到 baseappmgr |
| [ping_manager.hpp](file:///workspace/src/server/baseapp/ping_manager.hpp) | `PingManager`：客户端 ping 监控 |
| [shared_data_manager.hpp](file:///workspace/src/server/baseapp/shared_data_manager.hpp) | `SharedDataManager`：baseapp/global 共享数据 |
| [sqlite_database.hpp](file:///workspace/src/server/baseapp/sqlite_database.hpp) | `SqliteDatabase`：本地 SQLite 二级库 |
| [secondary_db_config.hpp](file:///workspace/src/server/baseapp/secondary_db_config.hpp) | `SecondaryDBConfig`：二级库配置 |
| [data_downloads.hpp](file:///workspace/src/server/baseapp/data_downloads.hpp) | `DataDownloads`：向客户端推送数据下载流 |
| [download_streamer.hpp](file:///workspace/src/server/baseapp/download_streamer.hpp) | `DownloadStreamer`：下载流控 |
| [py_bases.hpp](file:///workspace/src/server/baseapp/py_bases.hpp) | `PyBases`：把 `Bases` 暴露给 Python |
| [py_cell_data.hpp](file:///workspace/src/server/baseapp/py_cell_data.hpp) | `PyCellData`：cell 数据 blob 包装 |
| [script_bigworld.hpp](file:///workspace/src/server/baseapp/script_bigworld.hpp) | `BigWorldBaseAppScript`：`BigWorld` 模块在 baseapp 侧的初始化 |
| [controlled_shutdown_handler.hpp](file:///workspace/src/server/baseapp/controlled_shutdown_handler.hpp) | `ControlledShutdown`：受控停机流程 |
| [base_message_forwarder.hpp](file:///workspace/src/server/baseapp/base_message_forwarder.hpp) | `BaseMessageForwarder`：被 offload 的 base 消息转发 |
| [entity_channel_finder.hpp](file:///workspace/src/server/baseapp/entity_channel_finder.hpp) | `EntityChannelFinder`：按 channel id 找实体 |
| [message_handlers.hpp](file:///workspace/src/server/baseapp/message_handlers.hpp) | 内/外接口 handler 注册入口 |
| [id_config.hpp](file:///workspace/src/server/baseapp/id_config.hpp) | `IDConfig`：EntityID 池配置 |
| [bwtracer.hpp](file:///workspace/src/server/baseapp/bwtracer.hpp) | `BWTracerHolder`：trace 工具生命周期 |

---

## 2. 进程入口

### 2.1 `main.cpp` 与 `BIGWORLD_MAIN`

进程入口极简，全部交给 `bwMainT<BaseApp>` 模板：

```cpp
// main.cpp
#include "baseapp.hpp"
#include "baseapp_config.hpp"

#include "server/bwservice.hpp"

int BIGWORLD_MAIN( int argc, char * argv[] )
{
    return bwMainT< BaseApp >( argc, argv );
}
```

`BIGWORLD_MAIN` 宏（在 [bwservice.hpp](file:///workspace/src/lib/server/bwservice.hpp)）展开后大致是：

```cpp
int main( int argc, char * argv[] )
{
    BWResource bwresource;
    BWResource::init( argc, (const char **)argv );
    BWConfig::init( argc, argv );
    bwParseCommandLine( argc, argv );
    return bwMain( argc, argv );
}
```

而 `bwMainT<SERVER_APP>` 完成的是 BigWorld 服务端进程的标准启动序：

```cpp
// bwservice.hpp（节选）
template <class SERVER_APP>
int bwMainT( int argc, char * argv[], bool shouldLog = true )
{
    Mercury::EventDispatcher dispatcher;

    Mercury::NetworkInterface interface( &dispatcher,
            Mercury::NETWORK_INTERFACE_INTERNAL, 0,
            getBWInternalInterfaceSetting( SERVER_APP::configPath() ).c_str() );

    SignalProcessor signalProcessor( dispatcher );

    BW_MESSAGE_FORWARDER3( SERVER_APP::appName(), SERVER_APP::configPath(),
        /*ENABLED=*/shouldLog, dispatcher, interface );

    START_MSG( SERVER_APP::appName() );

    int result = doBWMainT< SERVER_APP >( dispatcher, interface, argc, argv );

    SERVER_APP::postDestruction();
    return result;
}
```

要点：
- `Mercury::EventDispatcher` 是 baseapp 的主事件循环，所有定时器、网络收包、Python tick 都挂在它上面。
- `Mercury::NetworkInterface` 这里以 `NETWORK_INTERFACE_INTERNAL` 模式创建——这是 baseapp 与其他服务端进程通信的“内部接口”（监听 machined 配置的内部网卡）。baseapp 还有第二条 `NetworkInterface`（外部接口，监听客户端），在 `BaseApp` 构造里创建。
- `BW_MESSAGE_FORWARDER3` 把日志转发到 `message_logger`。
- `START_MSG` 打印版本/机器/PID 等启动横幅。
- `doBWMainT` 调用 `ServerAppConfig::init(BaseAppConfig::postInit)` 完成配置项注册，再构造 `BaseApp` 实例并 `runApp`。

### 2.2 `BaseApp` 类骨架

`BaseApp` 是 baseapp 进程内的唯一单例（`Singleton<BaseApp>`），继承自：

- `EntityApp`（[entity_app.hpp](file:///workspace/src/lib/server/entity_app.hpp)）：baseapp/cellapp 共同基类，提供 `TimeQueue`、tick 统计、watcher 注册。
- `TimerHandler`：处理 game tick 定时器。
- `Mercury::ChannelTimeOutHandler`：处理 channel 超时（客户端掉线判定）。
- `Singleton<BaseApp>`：全局唯一访问点。

类声明（节选自 [baseapp.hpp](file:///workspace/src/server/baseapp/baseapp.hpp)）：

```cpp
class BaseApp : public EntityApp,
    public TimerHandler,
    public Mercury::ChannelTimeOutHandler,
    public Singleton< BaseApp >
{
public:
    typedef BaseAppConfig Config;

    ENTITY_APP_HEADER( BaseApp, baseApp )

    BaseApp( Mercury::EventDispatcher & mainDispatcher,
          Mercury::NetworkInterface & internalInterface );
    virtual ~BaseApp();

    void shutDown();
    static void postDestruction();

    // ... 内部接口、外部接口、C++ 接口分组 ...
};
```

`ENTITY_APP_HEADER` 宏（等价于 `SERVER_APP_HEADER`）会展开若干静态元信息：`appName()` 返回 `"BaseApp"`、`configPath()` 返回 `"baseapp"`、`postDestruction()` 等。

### 2.3 关键成员

`BaseApp` 持有的核心成员（节选自 `private` 段）：

```cpp
Mercury::NetworkInterface       extInterface_;          // 客户端外网接口

shared_ptr< EntityCreator >     pEntityCreator_;        // 创建 base 实体
BaseAppMgrGateway               baseAppMgr_;            // 与 baseappmgr 通信
Mercury::Address                cellAppMgr_;            // cellappmgr 地址
AnonymousChannelClient          dbMgr_;                 // 与 dbmgr 的匿名 channel

SqliteDatabase *                pSqliteDB_;             // 本地 SQLite 二级库
bool                            commitSecondaryDB_;

BWTracerHolder                  bwtracer_;

typedef std::map< Mercury::Address, Proxy * > Proxies;
Proxies                         proxies_;               // 客户端连接 → Proxy

Bases                           bases_;                 // EntityID → Base*

BaseAppID                       id_;

BasePtr                         baseForCall_;           // 当前正在被调用的 base
EntityID                        forwardingEntityIDForCall_;
bool                            baseForCallIsProxy_;
bool                            isExternalCall_;

PythonServer *                  pPythonServer_;         // Python 解释器宿主

GameTime                        commitTime_;
GameTime                        shutDownTime_;
Mercury::ReplyID                shutDownReplyID_;
bool                            isRetiring_;
TimeKeeper *                    pTimeKeeper_;
TimerHandle                     gameTimer_;

WorkerThread *                  pWorkerThread_;         // 后台 IO 线程

GlobalBases *                   pGlobalBases_;          // 全局 base 字典

shared_ptr< LoginHandler >              pLoginHandler_;
shared_ptr< BackedUpBaseApps >          pBackedUpBaseApps_;     // 我为别人存的备份
shared_ptr< DeadCellApps >              pDeadCellApps_;
shared_ptr< BackupSender >              pBackupSender_;         // 我把我的备份发出去
shared_ptr< Archiver >                  pArchiver_;
shared_ptr< SharedDataManager >         pSharedDataManager_;
shared_ptr< BaseMessageForwarder >      pBaseMessageForwarder_;
shared_ptr< BackupHashChain >           pBackupHashChain_;
shared_ptr< OffloadedBackups >          pOffloadedBackups_;
```

注意声明顺序的注释——“`pEntityCreator_` 必须在 `dbMgr_` 之前，因为 `dbMgr_` 析构时会取消挂起请求并回调 `EntityCreator::idClient_`”。这是 BigWorld 代码里典型的“成员析构顺序依赖”约束。

### 2.4 启动序列（init → finishInit）

`BaseApp::init` 是 `ServerApp::runApp` 调用的虚方法。结合 `AddToBaseAppMgrHelper` 与 `BaseAppMgrInterface::add` 的回复处理，baseapp 启动可拆为两段：

1. **本地 init**（`BaseApp::init`，由 `ServerApp::runApp` 调用）：
   - 初始化 Python（`initScript`）。
   - 创建 `WorkerThread`、`SharedDataManager`、`BaseMessageForwarder`、`BackupHashChain`、`DeadCellApps`、`BackedUpBaseApps`、`BackupSender`、`Archiver`、`LoginHandler`、`GlobalBases` 等子组件。
   - 注册内/外接口（`serveInterfaces`）。
   - 初始化 SQLite 二级库（`initSecondaryDB`）。
   - 通过 `findOtherProcesses` 找到 baseappmgr（用 machined 查询），构造 `AddToBaseAppMgrHelper` 并立即 `send()`。
2. **finishInit**（收到 baseappmgr 的 `add` 回复后，在 `AddToBaseAppMgrHelper::finishInit` 中调用 `BaseApp::finishInit(initData)`）：
   - 反序列化 `BaseAppInitData`（`id`、`time`、`isReady`）。
   - 设置 `id_`、`time`、`pTimeKeeper_`。
   - 启动 game tick 定时器（`startGameTickTimer`）。
   - 如果 `isBootstrap` 且配置允许，从 DB 自动加载实体（`EntityAutoLoader`，由 dbmgr 驱动）。
   - 调用 `ready(READY_BASE_APP_MGR)`，当所有依赖就绪后进入正常 tick。

`BaseAppInitData` 结构（[baseappmgr_interface.hpp](file:///workspace/src/server/baseappmgr/baseappmgr_interface.hpp)）：

```cpp
#pragma pack( push, 1 )
struct BaseAppInitData
{
    int32 id;            //!< ID of the new BaseApp
    GameTime time;       //!< Current game time
    bool isReady;        //!< Flag indicating whether the server is ready
};
#pragma pack( pop )
```

`AddToBaseAppMgrHelper`（[add_to_baseappmgr_helper.hpp](file:///workspace/src/server/baseapp/add_to_baseappmgr_helper.hpp)）是一个“自删除”的 `AddToManagerHelper`，构造即 `send()`，并在收到回复时调用 `app_.finishInit(initData)`：

```cpp
class AddToBaseAppMgrHelper : public AddToManagerHelper
{
public:
    AddToBaseAppMgrHelper( BaseApp & baseApp ) :
        AddToManagerHelper( baseApp.mainDispatcher() ),
        app_( baseApp )
    {
        // Auto-send on construction.
        this->send();
    }

    void handleFatalTimeout()
    {
        ERROR_MSG( "AddToBaseAppMgrHelper::handleFatalTimeout: Unable to add "
            "BaseApp to BaseAppMgr. Terminating.\n" );
        app_.shutDown();
    }

    void doSend()
    {
        app_.baseAppMgr().add( app_.intInterface().address(),
            app_.extInterface().address(), this );
    }

    bool finishInit( BinaryIStream & data )
    {
        app_.baseAppMgr().send();
        BaseAppInitData initData;
        data >> initData;
        return app_.finishInit( initData );
    }

private:
    BaseApp & app_;
};
```

`AddToManagerHelper`（[add_to_manager_helper.hpp](file:///workspace/src/lib/server/add_to_manager_helper.hpp)）负责重发与致命超时：

```cpp
class AddToManagerHelper :
    public Mercury::ShutdownSafeReplyMessageHandler,
    public TimerHandler
{
    // ...
    enum AddToManagerTimeouts
    {
        TIMEOUT_RESEND,
        TIMEOUT_FATAL
    };

    TimerHandle resendTimerHandle_;
    TimerHandle fatalTimerHandle_;
};
```

---

## 3. 核心类 `BaseApp`

### 3.1 继承体系与生命周期

```
ServerApp  (server_app.hpp)
   └── EntityApp (entity_app.hpp)        // 共用 baseapp/cellapp：TimeQueue + tickStats
          └── BaseApp (baseapp.hpp)      // + TimerHandler + ChannelTimeOutHandler + Singleton
```

`EntityApp::init` 完成 `ServerApp::init` 之外的实体级初始化（`timeQueue_`、`tickStatsPeriod_`）。`BaseApp::init` 在此基础上添加 baseapp 专属组件。

`BaseApp` 的 `hasStarted()` 由 `waitingFor_ == 0` 判定——`waitingFor_` 是位掩码，目前仅一位 `READY_BASE_APP_MGR`（见 [baseapp.hpp](file:///workspace/src/server/baseapp/baseapp.hpp)）。当 `baseappmgr` 回复 `add` 后调用 `ready(READY_BASE_APP_MGR)` 清掉该位，baseapp 才算“启动完成”。

### 3.2 内部接口（`BaseAppIntInterface`）

`BaseAppIntInterface` 是 baseapp 的“内部 IDL 总线”，定义在 [baseapp_int_interface.hpp](file:///workspace/src/server/baseapp/baseapp_int_interface.hpp)。它通过 `BEGIN_MERCURY_INTERFACE` / `END_MERCURY_INTERFACE` 宏声明一组消息，并由 `BW_STREAM_MSG_EX(BaseApp, xxx)` / `MF_BEGIN_BASE_STRUCT_MSG_EX(xxx)` / `MF_BEGIN_PROXY_MSG(xxx)` 等宏把消息和 `BaseApp` 或 `Base`/`Proxy` 的成员函数绑定。

按用途可分四组：

#### 3.2.1 baseapp 生命周期 / baseappmgr 通知

| 消息 | 处理者 | 含义 |
| --- | --- | --- |
| `startup` | `BaseApp::startup` | baseappmgr 通知 baseapp 启动（带 `bootstrap`、`didAutoLoadEntitiesFromDB`） |
| `shutDown` | `BaseApp::shutDown` | 受控停机（带 `isSigInt`） |
| `controlledShutDown` | `BaseApp::controlledShutDown` | 分阶段受控停机 |
| `handleCellAppMgrBirth` | `BaseApp::handleCellAppMgrBirth` | cellappmgr 上线 |
| `handleBaseAppMgrBirth` | `BaseApp::handleBaseAppMgrBirth` | baseappmgr 重生 |
| `handleCellAppDeath` | `BaseApp::handleCellAppDeath` | 某 cellapp 死亡 |
| `handleBaseAppBirth` | `BaseApp::handleBaseAppBirth` | 另一个 baseapp 上线（带备份哈希） |
| `handleBaseAppDeath` | `BaseApp::handleBaseAppDeath` | 另一个 baseapp 死亡 |
| `setBackupBaseApps` | `BaseApp::setBackupBaseApps` | baseappmgr 下发新的备份 baseapp 列表 |
| `startOffloading` | `BaseApp::startOffloading` | retirement：开始把实体 offload 到别的 baseapp |
| `setCreateBaseInfo` | `BaseApp::setCreateBaseInfo` | 设置 createBaseAnywhere 的远端地址 |
| `addGlobalBase` / `delGlobalBase` | `BaseApp::addGlobalBase` / `delGlobalBase` | 全局 base 注册/注销通知 |

#### 3.2.2 备份相关（new-style base backup）

| 消息 | 处理者 | 含义 |
| --- | --- | --- |
| `startBaseEntitiesBackup` | `BaseApp::startBaseEntitiesBackup` | baseappmgr 通知“开始为某个 baseapp（realBaseAppAddr）做备份”，带 `index`/`hashSize`/`prime`/`isInitial` |
| `stopBaseEntitiesBackup` | `BaseApp::stopBaseEntitiesBackup` | 停止为某个 baseapp 做备份 |
| `backupBaseEntity` | `BaseApp::backupBaseEntity` | 收到别的 baseapp 发来的某个实体的备份数据 |
| `stopBaseEntityBackup` | `BaseApp::stopBaseEntityBackup` | 停止备份某个 entityID |
| `ackOffloadBackup` | `BaseApp::ackOffloadBackup` | 确认某实体的 offload 备份已收到 |

#### 3.2.3 共享数据

```cpp
BW_STREAM_MSG( BaseApp, setSharedData )
BW_STREAM_MSG( BaseApp, delSharedData )
```

由 `SharedDataManager::setSharedData` / `delSharedData` 处理，用于跨 baseapp 同步 `baseAppData` 与 `globalData` 两张共享字典。

#### 3.2.4 来自 cellapp、与客户端相关的消息（绝大多数经 `Proxy` 转发到客户端）

这一组消息全部绑定到 `Proxy` 的成员函数（`MF_BEGIN_PROXY_MSG` / `MF_VARLEN_PROXY_MSG`），由 cellapp 发起，baseapp 收到后通常**直接转发给客户端 channel**：

| 消息 | 含义 |
| --- | --- |
| `setClient` | 标识本 bundle 内后续消息的目标 EntityID |
| `createCellPlayer` | 在客户端创建玩家自己的实体 |
| `spaceData` | 推送 space 数据（地理/天气/自定义 key-value） |
| `enterAoI` / `enterAoIOnVehicle` | 实体进入 AOI |
| `leaveAoI` | 实体离开 AOI |
| `createEntity` / `updateEntity` | AOI 内实体的创建/属性更新 |
| `detailedPosition` / `forcedPosition` | 精确位置同步 |
| `modWard` | 修改 ward（受控实体） |
| `callClientMethod` | 调用客户端的实体方法 |
| `acceptClient` | 接受客户端（登录收尾） |
| `sendToClient` | 通用“发往客户端”透传 |
| avatarUpdate 系列（来自 `common_client_interface.hpp`） | 玩家位置/朝向高频更新 |

此外还有 `Base` 自己处理的消息（`MF_BEGIN_BASE_STRUCT_MSG_EX` / `MF_BASE_STREAM_MSG`）：`currentCell`（cell 实体迁到新 cell）、`teleportOther`、`emergencySetCurrentCell`、`backupCellEntity`、`writeToDB`、`cellEntityLost`、`startKeepAlive`、`callBaseMethod`、`callCellMethod`、`getCellAddr`。

> 关键洞察：baseapp 的 `Proxy` 类用一段“宏循环”复用了客户端接口定义——
> ```cpp
> #define MF_BEGIN_COMMON_RELIABLE_MSG MF_BEGIN_PROXY_MSG
> #define MF_BEGIN_COMMON_PASSENGER_MSG MF_BEGIN_PROXY_MSG
> #define MF_BEGIN_COMMON_UNRELIABLE_MSG MF_BEGIN_PROXY_MSG
> #include "connection/common_client_interface.hpp"
> ```
> 即 `common_client_interface.hpp` 里定义的所有 client↔server 共享消息（avatarUpdate 等），在 baseapp 侧都被绑成 `Proxy::xxx`，在 cellapp 侧则由 `Witness` 直接处理。这是 BigWorld 让“客户端协议”在 baseapp/cellapp 两侧可复用的关键技巧。

### 3.3 外部接口（客户端 → baseapp）

外部接口定义在 `connection/baseapp_ext_interface.hpp`（baseapp 引擎库侧），主要消息：

| 消息 | 处理者 | 含义 |
| --- | --- | --- |
| `baseAppLogin` | `BaseApp::baseAppLogin` | 客户端首次连到 baseapp（带 loginApp 给的 sessionKey） |
| `authenticate` | `BaseApp::authenticate` | 客户端鉴权 |
| `avatarUpdateImplicit/Explicit` / `avatarUpdateWardImplicit/Explicit` | `Proxy::avatarUpdate*` | 客户端上报玩家位置/朝向 |
| `ackPhysicsCorrection` / `ackWardPhysicsCorrection` | `Proxy::ack*` | 客户端确认物理纠正 |
| `requestEntityUpdate` | `Proxy::requestEntityUpdate` | 客户端请求某实体的详细数据 |
| `enableEntities` | `Proxy::enableEntities` | 客户端就绪，开始接收 entity 创建 |
| `restoreClientAck` | `Proxy::restoreClientAck` | 客户端确认收到恢复（baseapp 切换后） |
| `disconnectClient` | `Proxy::disconnectClient` | 客户端主动断开 |

---

## 4. 实体：`Base`、`EntityType`、`EntityCreator`

### 4.1 `Base`（baseapp 侧的 entity 包装）

`Base` 是 baseapp 内**所有“逻辑实体”的 C++ 包装**，定义在 [base.hpp](file:///workspace/src/server/baseapp/base.hpp)。它继承自 `PyInstancePlus`（即 Python 实例对象），每个 `Base` 都对应一个 Python 脚本对象。

```cpp
class Base: public PyInstancePlus
{
    Py_InstanceHeader( Base )

public:
    Base( EntityID id, DatabaseID dbID, EntityTypePtr pType );
    ~Base();
    bool init( PyObject * pDict, PyObject * pCellArgs,
                bool isRestore = false );

    EntityID id() const                                 { return id_; }
    EntityMailBoxRef baseEntityMailBoxRef() const;

    Mercury::Channel& channel() { return *pChannel_; }
    const Mercury::Address & cellAddr() const           { return pChannel_->addr(); }

    void databaseID( DatabaseID );
    DatabaseID databaseID() const                       { return databaseID_; }

    // Has a valid DatabaseID or about to get one.
    bool hasWrittenToDB() const                         { return (databaseID_ != 0); }

    bool hasCellEntity() const;
    bool shouldSendToCell() const;

    bool isCreateCellPending() const                    { return isCreateCellPending_; }
    bool isGetCellPending() const                       { return isGetCellPending_; }
    bool isDestroyCellPending() const                   { return isDestroyCellPending_; }
    // ...
};
```

`Base` 的关键字段：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `pChannel_` | `Mercury::Channel *` | 指向 cellapp 上当前 cell 实体的 channel（即 cell 实体的真实地址） |
| `id_` | `EntityID` | 全局实体 ID（由 `IDClient` 向 dbmgr 申请） |
| `databaseID_` | `DatabaseID` | 数据库主键（首次 writeToDB 后由 dbmgr 分配） |
| `pType_` | `EntityTypePtr` | 实体类型（含 `EntityDescription` + Python 类） |
| `pCellData_` | `PyObjectPtr` | cell 数据（用于在 cellapp 上重建 cell 实体） |
| `pCellEntityMailBox_` | `CellEntityMailBox *` | 指向 cell 实体的 mailbox |
| `spaceID_` | `SpaceID` | 当前所在 space |
| `isProxy_` | `bool` | 是否是 `Proxy`（带客户端连接） |
| `isCreateCellPending_` / `isGetCellPending_` / `isDestroyCellPending_` | `bool` | cell 实体生命周期状态机标志 |
| `pTimers_` | `ScriptTimers *` | 脚本 `addTimer` 注册的定时器集合 |
| `cellBackupData_` | `std::string` | cell 实体最近一次 backup 来的 blob（用于 cellapp 崩溃后重建） |
| `shouldAutoBackup_` / `shouldAutoArchive_` | `AutoBackupAndArchive::Policy` | 自动备份/归档策略 |

`Base` 提供的核心能力：

1. **cell 实体生命周期管理**：
   - `createCellEntity(mailbox)` / `createInDefaultSpace()` / `createInNewSpace()`：在 cellapp 上创建对应的 cell 实体。`sendCreateCellEntity` 发请求，`CreateCellEntityHandler` 处理回复，`currentCell` 消息回填 cell 地址。
   - `destroyCellEntity`：销毁 cell 实体。
   - `setCurrentCell` / `currentCell` / `emergencySetCurrentCell`：cellapp 通知 cell 实体迁到了新 cell。
   - `cellEntityLost`：cell 实体已销毁。
   - `restoreTo(spaceID, cellAppAddr)`：在指定 cell 重建 cell 实体（用于 cellapp 死亡后恢复）。
   - `teleport(nearbyBaseMB)` / `teleportOther`：teleport 流程。

2. **持久化**：
   - `writeToDB(flags, pHandler, pCellData)`：把 base 数据 + cell 数据写到 dbmgr。`Base::addToStream` 负责序列化，`writeToDB` 消息发给 dbmgr。`flags` 控制是否同时写 cell 数据、是否创建新行等（具体常量见 [writedb.hpp](file:///workspace/src/lib/server/writedb.hpp)）。
   - `requestCellDBData`：请求 dbmgr 把 cell 数据也返回（用于离线实体的 cell 数据落盘）。
   - `getDBCellData`：从 dbmgr 回复中读取 cell 数据。
   - `archive()` / `autoArchive()`：周期性归档（区别于 `writeToDB` 的玩家主动写）。

3. **备份**：
   - `writeBackupData(stream, isOffload)` / `readBackupData(stream)`：把自身状态序列化到备份流，或从备份流恢复。
   - `offload(dstAddr)`：retirement 时把自己 offload 到另一个 baseapp。
   - `migrate()` / `migratedAll()`：脚本重载后迁移。

4. **Python 脚本接口**：`Base` 通过 `PY_AUTO_METHOD_DECLARE` / `PY_METHOD_DECLARE` / `PY_KEYWORD_METHOD_DECLARE` / `PY_RO_ATTRIBUTE_DECLARE` 等宏暴露大量方法到 Python，例如 `teleport`、`createCellEntity`、`createInDefaultSpace`、`createInNewSpace`、`py_destroy`、`py_writeToDB`、`py_addTimer` / `py_delTimer`、`py_registerGlobally` / `py_deregisterGlobally`。属性有 `id`、`isDestroyed`、`baseType`、`className`、`cell`、`cellData`、`databaseID`、`hasCell`、`shouldAutoBackup`、`shouldAutoArchive`。

`Base` 还内嵌一个 `BaseTimerHandler`（[base.hpp](file:///workspace/src/server/baseapp/base.hpp) 顶部）：

```cpp
class BaseTimerHandler : public TimerHandler
{
public:
    BaseTimerHandler( Base & base ) : base_( base ) {}
private:
    virtual void handleTimeout( TimerHandle handle, void * pUser );
    virtual void onRelease( TimerHandle handle, void  * pUser );
    Base & base_;
};
```

它把 C++ 定时器回调转交给 `Base::handleTimeout`，进而触发 `ScriptTimers` 上的 Python 回调。

### 4.2 `Bases` / `GlobalBases`

`Bases`（[bases.hpp](file:///workspace/src/server/baseapp/bases.hpp)）就是 `std::map<EntityID, Base*>` 的薄包装，提供 `add`、`erase`、`findEntity`、`discardAll`。

```cpp
class Bases
{
private:
    typedef std::map< EntityID, Base * > Container;
public:
    iterator begin();
    iterator end();
    size_type size() const;
    bool empty() const;
    iterator find( key_type key );
    Base * findEntity( EntityID id ) const;
    void add( Base * pBase );
    void erase( EntityID id )                   { container_.erase( id ); }
    void clear();
    void discardAll();
private:
    Container container_;
};
```

`GlobalBases`（[global_bases.hpp](file:///workspace/src/server/baseapp/global_bases.hpp)）则把“跨进程的全局 base 字典”暴露为 Python 字典对象 `BigWorld.globalBases`。脚本可以通过名字（pickled key）注册/查找全局 base mailbox：

```python
# 脚本侧用法（参见类注释）
globalBases = BigWorld.globalBases
print "The main mission entity is", globalBases[ "MainMission" ]
print "There are", len( globalBases ), "global bases."
```

`GlobalBases` 内部维护两张表：
- `pMap_`：Python 字典，存名字 → mailbox。
- `registeredBases_`：`multimap<EntityID, string>`，记录“本进程持有的哪些 base 注册了哪些名字”，用于 base 销毁时清理。

注册流程：`Base::py_registerGlobally(key, callback)` → `GlobalBases::registerRequest` → `BaseAppMgrGateway::registerBaseGlobally` → baseappmgr 广播 `addGlobalBase` 给所有 baseapp → 各 baseapp `BaseApp::addGlobalBase` → `GlobalBases::add`。

### 4.3 `EntityType`

`EntityType`（[entity_type.hpp](file:///workspace/src/server/baseapp/entity_type.hpp)）是 baseapp 侧的实体类型元信息，对应 `entities.xml` + `.def` 中的一个实体类型：

```cpp
class EntityType : public ReferenceCount
{
public:
    EntityType( const EntityDescription & desc, PyTypeObject * pType,
               bool isProxy );
    ~EntityType();

    BasePtr create( EntityID id, DatabaseID dbID, BinaryIStream & data,
        bool hasPersistentDataOnly );

    Base * newEntityBase( EntityID id, DatabaseID dbID );

    PyObject * createScript( BinaryIStream & data );
    PyObjectPtr createCellDict( BinaryIStream & data,
                                  bool strmHasPersistentDataOnly );

    const EntityDescription & description() const
                                            { return entityDescription_; }

    static EntityTypePtr getType( EntityTypeID typeID );
    static EntityTypePtr getType( const char * className );
    static EntityTypeID nameToIndex( const char * name );

    static bool init( bool isReload = false );
    static bool reloadScript( bool isRecover = false );
    static void migrate( bool isFullReload = true );
    static void cleanupAfterReload( bool isFullReload = true );

    bool isProxy() const            { return isProxy_; }
    // ...
};
```

关键点：
- 静态表 `s_curTypes_`（`vector<EntityTypePtr>`）持有当前所有类型；`s_newTypes_` 用于热重载期间的新类型。`s_nameToIndexMap_` 提供名字 → EntityTypeID 查找。
- `create` 是从二进制流（DB 或备份）反序列化一个 `Base`：先 `newEntityBase` 造 C++ 对象，再 `createScript` 造 Python 对象，最后 `Base::init`。
- `isProxy_` 标志区分普通 base 与 proxy 类型——在 `entities.xml` 里标记 `isProxy="true"` 的类型（通常是 `Account`/`Player`）才会创建 `Proxy` 子类实例。
- `s_persistentPropertiesMD5_` 用于校验持久化属性 schema 的一致性。
- `migrate` / `reloadScript` / `cleanupAfterReload` 支持 Python 脚本热重载：新类型 `setClass` 后，老实例逐步迁移到新类型。

### 4.4 `EntityCreator`

`EntityCreator`（[entity_creator.hpp](file:///workspace/src/server/baseapp/entity_creator.hpp)）是 baseapp 创建实体的统一入口，对外暴露三类 Python API：

```cpp
class EntityCreator
{
public:
    static shared_ptr< EntityCreator > create( DBMgr & idOwner,
            Mercury::NetworkInterface & intInterface,
            Mercury::NetworkInterface & extInterface );

    PyObject * createBaseRemotely( PyObject * args, PyObject * kwargs );
    PyObject * createBaseAnywhere( PyObject * args, PyObject * kwargs );
    PyObject * createBaseLocally( PyObject * args, PyObject * kwargs );
    PyObject * createBase( EntityType * pType, PyObject * pDict,
                            PyObject * pCellData = NULL ) const;

    bool createBaseFromDB( const std::string& entityType,
                    const std::string& name,
                    PyObjectPtr pResultHandler );
    bool createBaseFromDB( const std::string& entityType, DatabaseID dbID,
                    PyObjectPtr pResultHandler );
    // ...
private:
    IDClient        idClient_;
    Mercury::NetworkInterface & intInterface_;
    Mercury::NetworkInterface & extInterface_;
    Mercury::Address createBaseAnywhereAddr_;
};
```

对应 Python `BigWorld.createBase` / `createBaseFromDB` / `createBaseLocally` / `createBaseRemotely` / `createBaseAnywhere`：

| API | 行为 |
| --- | --- |
| `createBaseLocally` | 强制在本进程创建 |
| `createBaseRemotely` | 强制在指定 baseapp 创建（通过 `BaseAppIntInterface::createBaseWithCellData`） |
| `createBaseAnywhere` | 根据 `createBaseElsewhereThreshold`（[baseapp_config.hpp](file:///workspace/src/server/baseapp/baseapp_config.hpp)）决定本进程还是远端；本进程过载则发到 `createBaseAnywhereAddr_`（由 baseappmgr 周期下发） |
| `createBaseFromDB` | 向 dbmgr 发 `loadEntity`，由 `LoadEntityHandlerWithCallback` 处理回复，回调 Python |

`EntityCreator` 内嵌一个 `IDClient`（[id_client.hpp](file:///workspace/src/lib/server/id_client.hpp)），它向 dbmgr 批量申请 EntityID 区段（参见 `IDConfig` 的 `lowSize` / `desiredSize` / `highSize`），避免每次创建实体都跨进程要 ID。

---

## 5. 客户端连接

### 5.1 `Proxy`

`Proxy`（[proxy.hpp](file:///workspace/src/server/baseapp/proxy.hpp)）是 `Base` 的子类，**带客户端连接的特殊 base**。一个 `Proxy` 实例 = 一个在线玩家。

```cpp
class Proxy: public Base
{
    Py_Header( Proxy, Base )
public:
    static const int MAX_INCOMING_PACKET_SIZE = 1024;
    static const int MAX_OUTGOING_PACKET_SIZE = 1024;

    Proxy( EntityID id, DatabaseID dbID, EntityType * pType );
    ~Proxy();

    void onClientDeath( bool shouldExpectClient = true );
    void onDestroy();

    void restoreClient();
    void writeBackupData( BinaryOStream & stream );
    void offload( const Mercury::Address & dstAddr );
    void transferClient( const Mercury::Address & dstAddr, bool shouldReset );

    Mercury::Address readBackupData( BinaryIStream & stream, bool hasChannel );
    void onRestored( bool hasChannel, const Mercury::Address & clientAddr );
    // ...
};
```

`Proxy` 在 `Base` 基础上新增的字段：

| 字段 | 含义 |
| --- | --- |
| `pClientChannel_` | 指向客户端的 Mercury `Channel`（外网 UDP） |
| `clientBundlePrimer_` | `BundlePrimer`：在每个发往客户端的 bundle 头部插入 opportunistic 数据（如 entity 更新） |
| `encryptionKey_` | 客户端加密密钥 |
| `sessionKey_` | loginapp 分配的会话密钥（用于 baseAppLogin 鉴权） |
| `pClientEntityMailBox_` | 客户端 mailbox（向客户端投递方法调用） |
| `entitiesEnabled_` | 客户端是否已 `enableEntities` |
| `basePlayerCreatedOnClient_` | 玩家自己的实体是否已在客户端创建 |
| `wards_` | ward（受控实体）ID 列表 |
| `latencyTriggers_` / `latencyAtLastCheck_` | 延迟触发器 |
| `isRestoringClient_` | 是否正在恢复客户端（baseapp 切换） |
| `dataDownloads_` | 进行中的数据下载流 |
| `downloadRate_` / `apparentStreamingLimit_` / `avgUnackedPacketAge_` / `prevPacketsSent_` / `totalBytesDownloaded_` | 下载流控 |
| `pProxyPusher_` | 主动推送器 |
| `pBufferedClientBundle_` | 客户端 channel 尚未建立时缓冲的 bundle |
| `pRateLimiter_` / `rateLimitCallback_` | 限流器及回调 |
| `cellHasWitness_` / `cellBackupHasWitness_` / `numOutstandingEnableWitness_` | witness 状态 |

`Proxy` 的核心能力：

1. **客户端 channel 管理**：`attachToClient(clientAddr, loginReplyID)`、`detachFromClient`、`logOffClient`、`setClientChannel`。客户端 channel 断开时 `onClientDeath` 触发脚本回调。
2. **客户端登录**：`baseAppLogin`（外部接口）→ `LoginHandler::login` → `PendingLogins::find` → `Proxy::attachToClient`。
3. **AOI 转发**：cellapp 发 `enterAoI` / `leaveAoI` / `createEntity` / `updateEntity` / `avatarUpdate*` 给 baseapp，baseapp 的 `Proxy` 直接 forward 到客户端 channel。`Proxy::sendToClient` 是统一出口。
4. **`giveClientTo`**：把客户端连接转交给另一个 `Proxy`（用于 teleport 跨 baseapp 时）。`giveClientLocally` 是本进程内转交；跨进程则通过 `transferClient` 把 channel 迁移。
5. **数据下载流**：`streamStringToClient` / `streamFileToClient` 创建 `DataDownload`，由 `DataDownloads` 调度，每个 tick 把若干字节塞进客户端 bundle。
6. **限流**：每个 `Proxy` 持一个 `RateLimitMessageFilter`，所有来自客户端的消息先过 `filterMessage`，超限的进 `BufferedMessage` 队列，下个 tick 再 dispatch。
7. **Witness 控制**：`sendEnableDisableWitness` 让 cellapp 上的 cell 实体开/关 witness（决定是否向这个客户端推送 AOI）。
8. **网络仿真**：`delay(msecMin, msecMax, whichUDP)` / `loss(percentage, whichUDP)` 用于测试，给客户端 channel 加延迟/丢包。

### 5.2 `LoginHandler` 与 `PendingLogins`

`LoginHandler`（[login_handler.hpp](file:///workspace/src/server/baseapp/login_handler.hpp)）负责处理客户端经 loginapp 后接入 baseapp 的“第二跳”：

```cpp
class LoginHandler
{
public:
    LoginHandler();
    ~LoginHandler();

    SessionKey add( Proxy * pProxy, const Mercury::Address & loginAppAddr );
    void tick();
    void login( Mercury::NetworkInterface & networkInterface,
            const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );

    static WatcherPtr pWatcher();
private:
    void updateStatistics( const Mercury::Address & addr,
            const Mercury::Address & expectedAddr,
            uint32 attempt );

    PendingLogins * pPendingLogins_;

    // Statistics
    uint32 numLogins_;
    uint32 numLoginsAddrNAT_;
    uint32 numLoginsPortNAT_;
    uint32 numLoginsMultiAttempts_;
    uint32 maxLoginAttempts_;
    uint32 numLoginCollisions_;
};
```

登录流程的核心是 **`SessionKey` + `PendingLogins`**：

- loginapp 鉴权通过后，会先在 baseapp 上**预创建 Proxy**（通过 `BaseAppMgrInterface::createEntity` 或类似机制），baseapp 调 `LoginHandler::add(proxy, loginAppAddr)` 把 `(sessionKey → (proxy, loginAppAddr))` 存进 `PendingLogins`，并把 `sessionKey` 经 loginapp 回给客户端。
- 客户端拿到 `sessionKey` + baseapp 外网地址后，直接向 baseapp 发 `baseAppLogin`（外部接口）。baseapp 的 `LoginHandler::login` 收到后：
  1. 用 `sessionKey` 在 `PendingLogins` 里查 `PendingLogin`。
  2. 比对客户端实际地址与 `addrFromLoginApp_`——若 IP 不同则是 NAT 场景（`numLoginsAddrNAT_++`），若仅端口不同则 `numLoginsPortNAT_++`。
  3. 调 `Proxy::attachToClient(clientAddr, loginReplyID)` 把客户端 channel 绑到 proxy。
  4. 从 `PendingLogins` 移除该项。
- `PendingLogins::tick` 周期清理超时未确认的项（`QueueElement::hasExpired`），避免客户端从未直连时 proxy 永久泄漏。

`PendingLogins`（[pending_logins.hpp](file:///workspace/src/server/baseapp/pending_logins.hpp)）：

```cpp
class PendingLogins
{
public:
    typedef std::map< SessionKey, PendingLogin > Container;
    iterator find( SessionKey key );
    iterator end();
    void erase( iterator iter );
    SessionKey add( Proxy * pProxy, const Mercury::Address & loginAppAddr );
    void tick();
private:
    Container container_;
    class QueueElement
    {
    public:
        QueueElement( GameTime expiryTime,
                EntityID proxyID, SessionKey loginKey ) : ...
        bool hasExpired( GameTime time ) const { return (time == expiryTime_); }
        // ...
    };
    typedef std::list< QueueElement > Queue;
    Queue queue_;
};
```

`PendingLogin` 记录 proxy 与 loginApp 地址：

```cpp
class PendingLogin
{
public:
    PendingLogin( Proxy * pProxy, const Mercury::Address & loginAppAddr ) :
        pProxy_( pProxy ),
        addrFromLoginApp_( loginAppAddr ) {}
    const Mercury::Address & addrFromLoginApp() const { return addrFromLoginApp_; }
    SmartPointer< Proxy > pProxy() const              { return pProxy_; }
private:
    SmartPointer< Proxy > pProxy_;
    Mercury::Address addrFromLoginApp_;
};
```

### 5.3 `ClientEntityMailBox`

`ClientEntityMailBox`（[client_entity_mailbox.hpp](file:///workspace/src/server/baseapp/client_entity_mailbox.hpp)）是 baseapp 侧“向客户端投递方法调用”的 mailbox。它不直接发网络包，而是把方法调用塞进 `Proxy` 的客户端 bundle：

```cpp
class ClientEntityMailBox: public PyEntityMailBox
{
public:
    ClientEntityMailBox( Proxy & proxy ) :
        PyEntityMailBox(),
        proxy_( proxy ) {}

    virtual EntityID id() const;
    virtual void address( const Mercury::Address & address ) {}
    virtual const Mercury::Address address() const;

    virtual PyObject * pyGetAttribute( const char * attr );
    virtual BinaryOStream * getStream( const MethodDescription & methodDesc,
        Mercury::ReplyMessageHandler * pHandler = NULL );
    virtual void sendStream();
    virtual const MethodDescription * findMethod( const char * attr ) const;

    BinaryOStream * getStreamForEntityID( int methodID, EntityID entityID );
    const EntityDescription& getEntityDescription() const;
    EntityMailBoxRef ref() const;
private:
    Proxy & proxy_;
};
```

脚本侧 `entity.client.someMethod(...)` 实际就是通过 `ClientEntityMailBox::getStream` 拿到 `Proxy::clientBundle()` 的 `BinaryOStream`，写入方法 ID 与参数。

### 5.4 限流：`RateLimitMessageFilter` / `ProxyBufferedMessage` / `ProxyRateLimitCallback`

`RateLimitMessageFilter`（[rate_limit_message_filter.hpp](file:///workspace/src/server/baseapp/rate_limit_message_filter.hpp)）是 **per-client** 的消息限流器，挂在每个 `Proxy` 的客户端 channel 上（作为 `Mercury::MessageFilter`）。

```cpp
class RateLimitMessageFilter : public Mercury::MessageFilter
{
public:
    typedef RateLimitConfig Config;

    class Callback
    {
    public:
        virtual BufferedMessage * createBufferedMessage(
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data,
            Mercury::InputMessageHandler * pHandler );
        virtual void onMessageDeleted( BufferedMessage * pMessage ) {}
        virtual void onFilterLimitsExceeded( const Mercury::Address & srcAddr,
            BufferedMessage * pMessage ) {}
    };

    RateLimitMessageFilter( const Mercury::Address & addr );
    ~RateLimitMessageFilter();

    void filterMessage( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data,
        Mercury::InputMessageHandler * pHandler );

    void tick();
    void setCallback( Callback * pCallback ) { pCallback_ = pCallback; }
private:
    void replayAny();
    bool canSendNow( uint dataLen );
    BufferedMessage * front();
    void pop_front();
    void dispatch( ... );
    bool buffer( BufferedMessage * pMsg );

    Callback *              pCallback_;
    Mercury::Address        addr_;
    uint8                   warnFlags_;
    uint                    numReceivedSinceLastTick_;
    uint                    receivedBytesSinceLastTick_;
    uint                    numDispatchedSinceLastTick_;
    uint                    dispatchedBytesSinceLastTick_;
    MsgQueue                queue_;
    uint                    sumBufferedSizes_;
};
```

工作流：

1. 客户端消息到达 → `filterMessage`。
2. 累加 `numReceivedSinceLastTick_` / `receivedBytesSinceLastTick_`。
3. 若 `canSendNow(dataLen)` 成立（未超 `maxMessagesPerTick` / `maxBytesPerTick`），立即 `dispatch`。
4. 否则 `buffer(pMsg)`：调 `pCallback_->createBufferedMessage` 造一个 `BufferedMessage`（实际是 `ProxyBufferedMessage`）入队 `queue_`，累加 `sumBufferedSizes_`。
5. 每 tick `Proxy::tickRateLimitFilters` 调 `RateLimitMessageFilter::tick` → `replayAny`：在 `maxMessagesPerSecond` / `maxBytesPerSecond` 余量内把队列里的消息逐条 `dispatch`。
6. 队列超 `maxMessagesBuffered` / `maxBytesBuffered` 时 `onFilterLimitsExceeded` 触发，`ProxyRateLimitCallback` 默认实现是踢掉客户端。

`RateLimitConfig`（[rate_limit_config.hpp](file:///workspace/src/server/baseapp/rate_limit_config.hpp)）暴露三组限：`perSecond`、`perTick`、`buffered`，每组又分 `messages` / `bytes`，且各有 `warn` / `max` 两个阈值。

`BufferedMessage`（在 [rate_limit_message_filter.hpp](file:///workspace/src/server/baseapp/rate_limit_message_filter.hpp) 内）是基类，存了 `header_`、`data_`（`MemoryOStream`）、`pHandler_`。`ProxyBufferedMessage`（[proxy_buffered_message.hpp](file:///workspace/src/server/baseapp/proxy_buffered_message.hpp)）覆盖 `dispatch`，在派发前额外通知 `Proxy`：

```cpp
class ProxyBufferedMessage : public BufferedMessage
{
public:
    ProxyBufferedMessage( const Mercury::UnpackedMessageHeader & header,
                BinaryIStream & data,
                Mercury::InputMessageHandler * pHandler ) :
            BufferedMessage( header, data, pHandler ) {}

    virtual void dispatch( RateLimitMessageFilter::Callback * pCallback,
        const Mercury::Address & srcAddr );
};
```

`ProxyRateLimitCallback`（[proxy_rate_limit_callback.hpp](file:///workspace/src/server/baseapp/proxy_rate_limit_callback.hpp)）实现 `createBufferedMessage`（造 `ProxyBufferedMessage`）与 `onFilterLimitsExceeded`（断开客户端）。

---

## 6. 与 dbmgr 的协作

baseapp 与 dbmgr 的交互分四类：**ID 申请**、**实体加载**、**实体保存/删除**、**cell 数据落盘**。dbmgr 在 baseapp 侧表现为一个 `Mercury::ChannelOwner` 别名 `DBMgr`：

```cpp
// baseapp.hpp
typedef Mercury::ChannelOwner DBMgr;
// ...
AnonymousChannelClient          dbMgr_;
// ...
DBMgr & dbMgr()                     { return *dbMgr_.pChannelOwner(); }
```

### 6.1 `LoadEntityHandler`

`LoadEntityHandler`（[load_entity_handler.hpp](file:///workspace/src/server/baseapp/load_entity_handler.hpp)）是 `dbmgr::loadEntity` 回复的统一处理器。三个常量描述加载结果：

```cpp
static const uint8 LOAD_FROM_DB_FAILED          = 0;
static const uint8 LOAD_FROM_DB_SUCCEEDED       = 1;
static const uint8 LOAD_FROM_DB_FOUND_EXISTING  = 2;
```

`LoadEntityHandler` 是抽象基类（`onLoadedFromDB` 是空实现），两个具体子类：

- `LoadEntityHandlerWithCallback`：加载完成后回调一个 Python 函数（用于 `BigWorld.createBaseFromDB` 的脚本回调）。
- `LoadEntityHandlerWithReply`：加载完成后回复一个 Mercury 请求（用于跨 baseapp 的 `createBaseFromDB` 远端调用）。

`onLoadedFromDB` 的语义（见注释）：
- `pBase != NULL`：成功加载，得到一个 `Base` 实例。
- `pMailbox != NULL`：实体已被别的 baseapp checkout，得到指向它的 mailbox。
- 两者皆 NULL：加载失败，`dbID == 0`。
- `pBase` 与 `pMailbox` 互斥。

`unCheckoutEntity(databaseID)` 用于失败时回滚 dbmgr 的 checkout 状态。

### 6.2 `LoadingThread` / `WorkerThread` / `FileStreamingJob`

`WorkerThread`（[worker_thread.hpp](file:///workspace/src/server/baseapp/worker_thread.hpp)）是 baseapp 的后台 IO 线程，继承自 `SimpleThread`。它维护一个 `DoleQueue`（`map<uint64, WorkerJob*>`，按下次执行时间排序）：

```cpp
class WorkerThread : public SimpleThread
{
private:
    typedef std::map<uint64,WorkerJob*> DoleQueue;
    DoleQueue       queue_;
    SimpleMutex     queueLock_;
    bool            ready_;
    bool            done_;
    friend class WorkerJob;
};

class WorkerJob
{
public:
    WorkerJob();
    virtual ~WorkerJob();
    void submit( WorkerThread * pThread );
    void withdraw();
    virtual float operator()() = 0;     // 返回下次调度的间隔秒数
    static const float DONT_RESCHEDULE = -0.5f;
    static const float DONT_RESCHEDULE_AND_DESTROY = -1.5f;
    bool isDisowned() const             { return pThread_ == NULL; }
private:
    WorkerThread * pThread_;
    uint64          nextTime_;
    bool            running_;
    bool            deleting_;
    void disowned();
    friend class WorkerThread;
};
```

`WorkerJob::operator()` 返回值约定：
- `> 0`：下次调用的间隔秒数。
- `0`：尽快再次调用。
- `DONT_RESCHEDULE` (-0.5)：不再调度（保持存活）。
- `DONT_RESCHEDULE_AND_DESTROY` (-1.5)：不再调度并自删。

[loading_thread.hpp](file:///workspace/src/server/baseapp/loading_thread.hpp) 在 `WorkerJob` 之上扩展了：

- `TickedWorkerJob`：可被全局 `tickJobs()` 周期 tick 的 job（静态链表 `jobs_`）。
- `FileStreamingJob`：把磁盘文件分块读入内存缓冲（默认 64KB buffer），用于 `BigWorld.addProxyFileData()` / `streamFileToClient()`。

```cpp
class FileStreamingJob : public WorkerJob
{
public:
    FileStreamingJob( const std::string &path, int bufsize = 65536 );
    int size();
    int freeSpace();
    bool done();
    bool good();
    void read( BinaryOStream &os, int nBytes );
    void write( const char *src, int nBytes );
private:
    virtual float operator()();
protected:
    std::string path_;
    FILE *file_;
    char *buf_;
    typedef std::deque< std::string* > Data;
    Data data_;
    int offset_;
    int size_;
    int maxsize_;
    bool doneReading_;
    bool error_;
    SimpleMutex lock_;
};
```

### 6.3 `CreateCellEntityHandler`

`CreateCellEntityHandler`（[create_cell_entity_handler.hpp](file:///workspace/src/server/baseapp/create_cell_entity_handler.hpp)）处理 `Base::createCellEntity` 的回复。两个变体：

```cpp
class CreateCellEntityHandler : public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    CreateCellEntityHandler( Base * pBase );
    void handleMessage( const Mercury::Address & source,
            Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data, void * arg );
    void handleException( const Mercury::NubException & exception, void * arg );
private:
    BasePtr pBase_;
};

class CreateCellEntityViaBaseHandler :
    public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    CreateCellEntityViaBaseHandler( Base * pBase,
            Mercury::ReplyMessageHandler * pHandler,
            EntityID nearbyID );
    // ...
private:
    void cellCreationFailure();
    BasePtr pBase_;
    std::auto_ptr< Mercury::ReplyMessageHandler > pHandler_;
    EntityID nearbyID_;
};
```

- `CreateCellEntityHandler`：直接发给 cellapp（`Base::sendCreateCellEntity`）。
- `CreateCellEntityViaBaseHandler`：经过另一个 base 中转（`Base::sendCreateCellEntityViaBase`），用于“在某个 base 拥有的 cell 实体附近创建”的场景。`cellCreationFailure` 处理失败回滚。

`ShutdownSafeReplyMessageHandler` 是 Mercury 的基类，在 baseapp 正在 shutdown 时安全地丢弃回复，避免悬挂回调。

### 6.4 `WriteToDBReplyHandler` / `WriteToDBReplyStruct`

`WriteToDBReplyHandler`（[write_to_db_reply.hpp](file:///workspace/src/server/baseapp/write_to_db_reply.hpp)）是 dbmgr `writeToDB` 回复的抽象接口：

```cpp
class WriteToDBReplyHandler
{
public:
    virtual ~WriteToDBReplyHandler() {};
    virtual void onWriteToDBComplete( bool succeeded ) = 0;
};
```

`WriteToDBReplyStruct` 是引用计数的“回复聚合器”——它同时等待 **dbmgr 写盘完成** 与 **备份 baseapp 完成备份** 两件事，都完成后才回调 `pHandler_`：

```cpp
class WriteToDBReplyStruct : public ReferenceCount
{
public:
    WriteToDBReplyStruct( WriteToDBReplyHandler * pHandler );
    bool expectsReply() const { return pHandler_ != NULL; }
    void onWriteToDBComplete( bool succeeded );
    void onBackupComplete();
private:
    bool succeeded_;
    bool backedUp_;
    bool writtenToDB_;
    WriteToDBReplyHandler * pHandler_;
};
```

这是 BigWorld 把“持久化”与“备份”统一成一次回调的关键设计——脚本调 `base.writeToDB()` 时，可以同时等待 DB 落盘 + 备份 baseapp 收到备份，两者都成功才算“安全落盘”。

---

## 7. 备份与归档（关键）

baseapp 的备份系统是整个 BigWorld 高可用的核心。任何一个 baseapp 死掉，其上的所有 base 实体必须由“备份 baseapp”恢复，且不能丢失玩家的最近状态。

### 7.1 备份哈希环（`BackupHash` / `MiniBackupHash` / `BackupHashChain`）

备份选址用一致性哈希。核心类在引擎库 [backup_hash.hpp](file:///workspace/src/lib/server/backup_hash.hpp)：

```cpp
class MiniBackupHash
{
public:
    MiniBackupHash( uint32 prime = 0, uint32 size = 0 ) :
        prime_( prime ),
        size_( size ),
        virtualSize_( 0 )
    {
        this->handleSizeChange( size_ );
    };

    uint32 prime() const        { return prime_; }
    uint32 size() const         { return size_; }
    uint32 virtualSize() const  { return virtualSize_; }

    uint32 hashFor( EntityID id ) const;
    uint32 virtualSizeFor( uint32 index ) const;

protected:
    void handleSizeChange( uint32 newSize )
    {
        size_ = newSize;
        if (size_ > 0)
        {
            virtualSize_ = 1;
            while (virtualSize_ < size_)
            {
                virtualSize_ *= 2;      // 向上取整到 2 的幂
            }
        }
        else
        {
            virtualSize_ = 0;
        }
    }

protected:
    uint32 prime_;
private:
    uint32 size_;
    uint32 virtualSize_;
};
```

`MiniBackupHash` 把 `[0, size)` 的桶号映射到 `[0, virtualSize)` 的“虚拟桶”空间（`virtualSize_` 是 `>= size_` 的最小 2 的幂）。`hashFor(id)` 用 `(prime * id) % virtualSize` 算出虚拟桶号，再映射回真实桶号——若虚拟桶号 `< size_` 直接用，否则折半（这是经典的一致性哈希“虚拟节点 + 取模折半”技巧，扩容/缩容时只迁移 `1/virtualSize` 的实体）。

`BackupHash` 在 `MiniBackupHash` 基础上加了**地址表**：

```cpp
class BackupHash : public MiniBackupHash
{
public:
    BackupHash();   // 构造时随机选一个 prime
    Mercury::Address addressFor( EntityID id ) const;

    class DiffVisitor
    {
    public:
        virtual void onAdd( const Mercury::Address & addr,
                uint32 index, uint32 virtualSize, uint32 prime ) = 0;
        virtual void onChange( const Mercury::Address & addr,
                uint32 index, uint32 virtualSize, uint32 prime ) = 0;
        virtual void onRemove( const Mercury::Address & addr,
                uint32 index, uint32 virtualSize, uint32 prime ) = 0;
    };

    void diff( const BackupHash & other, DiffVisitor & visitor );
    void clear();
    bool empty() const;
    const Mercury::Address & operator[]( const int index ) const;
    void swap( BackupHash & other );
    void push_back( const Mercury::Address & addr );
    bool erase( const Mercury::Address & addr );
private:
    static uint32 choosePrime();
    typedef std::vector< Mercury::Address > Container;
    Container addrs_;
};
```

要点：
- 每个 baseapp 维护一个 `BackupHash`（`entityToAppHash_`）——“我的实体应该备份到哪个 baseapp”。`addressFor(entityID)` 返回的就是该实体的备份 baseapp 地址。
- `prime_` 是随机素数（`choosePrime()`），保证不同 baseapp 的哈希函数不同——避免几个 baseapp 同时死亡时实体集中迁移到同一目标。
- `diff(other, visitor)` 是一致性哈希的“差异同步”接口：baseappmgr 在变更 baseapp 列表后下发新 hash，baseapp 用 `diff` 算出哪些实体需要重新备份（`onAdd`/`onChange`/`onRemove`），只迁移差异部分。

`BackupHashChain`（[backup_hash_chain.hpp](file:///workspace/src/lib/server/backup_hash_chain.hpp)）维护**哈希历史链**：

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
};
```

当一个 baseapp 死亡时，它的备份 baseapp 需要知道**死亡 baseapp 当时的哈希函数**，才能找到自己为之备份的实体。`BackupHashChain` 按 `deadApp address → BackupHash` 保存历史快照，`adjustForDeadBaseApp(deadApp, hash)` 把当时的 hash 压入链；`addressFor(address, entityID)` 用某 baseapp 的 hash 算出 entityID 在该 baseapp 上的备份地址。这样即使多次 baseapp 死亡 + 哈希变更，恢复时仍能定位到正确的备份。

### 7.2 `BackupSender`（出向：把我的实体备份给别人）

`BackupSender`（[backup_sender.hpp](file:///workspace/src/server/baseapp/backup_sender.hpp)）负责把本进程的 base 实体周期性备份到哈希环选定的另一个 baseapp：

```cpp
class BackupSender
{
public:
    BackupSender( BaseApp & baseApp );
    void tick( const Bases & bases,
               Mercury::NetworkInterface & networkInterface );

    Mercury::Address addressFor( EntityID entityID ) const
    {
        return entityToAppHash_.addressFor( entityID );
    }

    void addToStream( BinaryOStream & stream );
    void handleBaseAppDeath( const Mercury::Address & addr );
    void setBackupBaseApps( BinaryIStream & data,
       Mercury::NetworkInterface & networkInterface );

    void restartBackupCycle( const Bases & bases );
    void startOffloading() { isOffloading_ = true; }
    bool autoBackupBase( Base & base,
                     Mercury::BundleSendingMap & bundles,
                     Mercury::ReplyMessageHandler * pHandler = NULL );
    bool backupBase( Base & base,
                     Mercury::BundleSendingMap & bundles,
                     Mercury::ReplyMessageHandler * pHandler = NULL );
private:
    void ackNewBackupHash();

    int                 offloadPerTick_;
    float               backupRemainder_;          // 跨 tick 的余数
    std::vector<EntityID>  basesToBackUp_;
    BackupHash          entityToAppHash_;          // 当前生效
    BackupHash          newEntityToAppHash_;       // 切换中的新 hash
    bool                isUsingNewBackup_;
    bool                isOffloading_;
    BaseApp &           baseApp_;
};
```

`tick` 流程：
1. 维护 `basesToBackUp_`（一个 EntityID 列表），按 `backupPeriodInTicks` 周期遍历（`restartBackupCycle` 重建列表）。
2. 每个 tick 处理一定数量的实体（`backupRemainder_` 处理小数余量，保证平均速率）。
3. 对每个实体调 `backupBase`：`Base::writeBackupData(stream, isOffload=false)` 序列化 → 通过 `BaseAppIntInterface::backupBaseEntity` 发到 `addressFor(entityID)` 对应的 baseapp。
4. `setBackupBaseApps` 接收 baseappmgr 下发的新 baseapp 列表，构造 `newEntityToAppHash_`，与 `entityToAppHash_` 做 `diff`，逐步把实体从旧备份迁移到新备份，全部迁移完 `ackNewBackupHash` → 切换 hash。

`handleBaseAppDeath` 处理“我的某个备份 baseapp 死了”——把那些实体标记为需要重新备份。

### 7.3 `BackedUpBaseApp` / `BackedUpBaseApps`（入向：我为别人存的备份）

`BackedUpBaseApp`（[backed_up_base_app.hpp](file:///workspace/src/server/baseapp/backed_up_base_app.hpp)）保存“我为某个 baseapp 持有的备份数据”：

```cpp
class BackedUpEntities
{
public:
    std::string & getDataFor( EntityID entityID ) { return data_[ entityID ]; }
    bool contains( EntityID entityID ) const      { return data_.count( entityID ) != 0; }
    bool erase( EntityID entityID )               { return data_.erase( entityID ); }
    void clear()                                  { data_.clear(); }
    void restore();
    bool empty() const                            { return data_.empty(); }
    void restoreRemotely( const Mercury::Address & deadAddr,
          const BackupHashChain & backedUpHashChain );
protected:
    typedef std::map< EntityID, std::string > Container;
    Container data_;
};

class BackedUpEntitiesWithHash : public BackedUpEntities
{
public:
    void init( uint32 index, const MiniBackupHash & hash,
               const BackedUpEntitiesWithHash & current );
    void swap( BackedUpEntitiesWithHash & other );
private:
    uint32 index_;
    MiniBackupHash hash_;
};

class BackedUpBaseApp
{
public:
    BackedUpBaseApp() : usingNew_( false ), canSwitchToNewBackup_( false ) {}
    void startNewBackup( uint32 index, const MiniBackupHash & hash );
    std::string & getDataFor( EntityID entityID );
    bool erase( EntityID entityID );
    void switchToNewBackup();
    void discardNewBackup();
    bool canSwitchToNewBackup() const             { return canSwitchToNewBackup_; }
    void canSwitchToNewBackup( bool value )       { canSwitchToNewBackup_ = value; }
    void restore();
private:
    BackedUpEntitiesWithHash currentBackup_;      // 当前生效的备份
    BackedUpEntitiesWithHash newBackup_;          // 切换中的新备份
    bool usingNew_;
    bool canSwitchToNewBackup_;
};
```

设计要点：
- `BackedUpBaseApp` 同时持有 `currentBackup_` 与 `newBackup_` 两份，实现“双缓冲切换”——baseappmgr 下发新 hash 时，新备份写入 `newBackup_`，等所有实体都迁移完，`switchToNewBackup` 原子切换。
- `BackedUpEntitiesWithHash::init(index, hash, current)` 用旧 hash 算出哪些 entityID 现在归我（落在 `index` 桶），把它们从 `current` 复制到 `newBackup_`。
- `restore()` 在源 baseapp 死亡时被调用——把 `data_` 里的每个 entity 反序列化为 `Base` 实例并加入本进程 `bases_`。
- `restoreRemotely(deadAddr, backedUpHashChain)` 处理“我并不是死亡的 baseapp 的直接备份，而是间接备份（哈希链上的更下游）”的复杂场景。

### 7.4 `Archiver`（周期归档到 dbmgr）

`Archiver`（[archiver.hpp](file:///workspace/src/server/baseapp/archiver.hpp)）周期性把本进程的 base 实体**归档**到 dbmgr（与 `BackupSender` 的“备份到另一个 baseapp”不同，归档是持久化到 MySQL）：

```cpp
class Archiver
{
public:
    Archiver();
    void tick( DBMgr & dbMgr, BaseAppMgrGateway & baseAppMgr,
        Bases & bases, SqliteDatabase * pSecondaryDB );
    void handleBaseAppDeath( const Mercury::Address & addr,
        Bases & bases, SqliteDatabase * pSecondaryDB );
    static bool isEnabled();
private:
    void restartArchiveCycle( Bases & bases );
    int                 archiveIndex_;
    std::vector<EntityID>  basesToArchive_;
    Mercury::Address    deadBaseAppAddr_;
};
```

- `tick`：按 `archivePeriodInTicks` 周期遍历 `basesToArchive_`，每个 tick 归档若干实体。`Base::archive()` 调 `writeToDB` 走 dbmgr。
- `handleBaseAppDeath`：当某个 baseapp 死亡时，其上的实体的最近备份在我这里，我要把这些备份归档到 dbmgr（避免恢复后丢失最近状态）。
- `isEnabled`：归档可配置开关。

`Base::shouldAutoArchive`（`AutoBackupAndArchive::Policy`）控制单个实体是否参与自动归档。

### 7.5 `OffloadedBackups`（retirement 期间的备份）

`OffloadedBackups`（[offloaded_backups.hpp](file:///workspace/src/server/baseapp/offloaded_backups.hpp)）专门处理 **retirement** 场景：baseapp 在受控停机时，会把自己的实体 **offload** 到其他 baseapp（`Base::offload` / `Proxy::offload`）。offload 之后，本 baseapp 不再持有实体，但**作为这些 offload 出去的实体的备份**，直到接替的 baseapp 确认收到：

```cpp
class OffloadedBackups
{
public:
    OffloadedBackups();
    bool wasOffloaded( EntityID entityID ) const;
    void backUpEntity( const Mercury::Address & srcAddr,
                       BinaryIStream & data );
    void handleBaseAppDeath( const Mercury::Address & deadAddr,
                             const BackupHashChain & backedUpHashChain );
    void stopBackingUpEntity( EntityID entityID );
    bool isEmpty()      { return apps_.empty(); }
private:
    typedef std::map< Mercury::Address, BackedUpEntities > Container;
    Container apps_;
};
```

`apps_` 按 offload 目标 baseapp 地址分组。`BaseApp::ackOffloadBackup` 收到目标 baseapp 的确认后调 `stopBackingUpEntity`。

### 7.6 `DeadCellApps`

`DeadCellApps`（[dead_cell_apps.hpp](file:///workspace/src/server/baseapp/dead_cell_apps.hpp)）记录最近死亡的 cellapp 地址，用于 cellapp 崩溃后 baseapp 上的 base 实体恢复 cell 实体：

```cpp
class DeadCellApps
{
public:
    bool isRecentlyDead( const Mercury::Address & addr ) const;
    void addApp( const Mercury::Address & addr, BinaryIStream & data );
    void tick( const Bases & bases, Mercury::NetworkInterface & intInterface );
private:
    void removeOldApps();
    typedef std::vector< shared_ptr< DeadCellApp > > Container;
    Container apps_;
    typedef std::list< shared_ptr< DyingCellApp > > DyingCellApps;
    DyingCellApps dyingApps_;
};
```

`BaseApp::isRecentlyDeadCellApp` 用于判断来自某地址的 ghost 消息是否来自已死 cellapp（避免幽灵消息）。`tick` 周期清理过期项。`Base::restoreTo` 在 cellapp 死亡后被调用，把 cell 实体在新的 cellapp 上重建。

### 7.7 `BaseAppIntInterface`（baseapp↔baseapp 通信）

`BaseAppIntInterface`（[baseapp_int_interface.hpp](file:///workspace/src/server/baseapp/baseapp_int_interface.hpp)）已在 §3.2 详述。baseapp 之间的关键消息：

- `handleBaseAppBirth` / `handleBaseAppDeath`：新 baseapp 上线/死亡通知。
- `setBackupBaseApps`：baseappmgr 下发新备份 baseapp 列表。
- `startBaseEntitiesBackup` / `stopBaseEntitiesBackup` / `stopBaseEntityBackup`：开始/停止为某 baseapp 做备份。
- `backupBaseEntity`：实际传输某个实体的备份数据（`BW_BIG_STREAM_MSG_EX`，允许大包）。
- `ackOffloadBackup`：确认 offload 备份。
- `forwardedBaseMessage`：转发给已 offload 实体的消息（由 `BaseMessageForwarder` 处理）。
- `setSharedData` / `delSharedData`：共享数据同步。
- `createBaseWithCellData` / `createBaseFromDB`：远端创建 base。
- `callWatcher`：watcher 转发。

### 7.8 `BaseMessageForwarder`

`BaseMessageForwarder`（[base_message_forwarder.hpp](file:///workspace/src/server/baseapp/base_message_forwarder.hpp)）维护“已被 offload 的实体 → 新地址”的映射，把发往这些实体的消息转发到新 baseapp：

```cpp
class BaseMessageForwarder
{
public:
    BaseMessageForwarder( Mercury::NetworkInterface & networkInterface );
    void addForwardingMapping( EntityID entityID,
            const Mercury::Address & destAddr );
    bool forwardIfNecessary( EntityID entityID,
            const Mercury::Address & srcAddr,
            const Mercury::UnpackedMessageHeader & header,
            BinaryIStream & data );
    Mercury::ChannelPtr getForwardingChannel( EntityID entityID );
private:
    typedef std::map< EntityID, Mercury::Address > Map;
    Map                             map_;
    Mercury::NetworkInterface &     networkInterface_;
};
```

`BaseApp::forwardBaseMessageIfNecessary` 在每条内部消息到达时先调 `forwardIfNecessary`——若该 entityID 已 offload，直接转发，避免老地址收到无效消息。

---

## 8. 与 baseappmgr 的协作

### 8.1 `BaseAppMgrGateway`

`BaseAppMgrGateway`（[baseappmgr_gateway.hpp](file:///workspace/src/server/baseapp/baseappmgr_gateway.hpp)）是 baseapp 与 baseappmgr 通信的网关，继承自 `ManagerAppGateway`：

```cpp
class BaseAppMgrGateway : public ManagerAppGateway
{
public:
    BaseAppMgrGateway( Mercury::NetworkInterface & interface );

    void onManagerRebirth( BaseApp & baseApp, const Mercury::Address & addr );

    void add( const Mercury::Address & addrForCells,
        const Mercury::Address & addrForClients,
        Mercury::ReplyMessageHandler * pHandler );

    void useNewBackupHash( const BackupHash & entityToAppHash,
        const BackupHash & newEntityToAppHash );

    void informOfArchiveComplete( const Mercury::Address & deadBaseAppAddr );

    void registerBaseGlobally( const std::string & pickledKey,
        const EntityMailBoxRef & mailBox,
        Mercury::ReplyMessageHandler * pHandler );

    void deregisterBaseGlobally( const std::string & pickledKey );

    const Mercury::Address & addr() const     { return channel_.addr(); }
    Mercury::Bundle & bundle()                { return channel_.bundle(); }
    void send()                               { channel_.send(); }
};
```

主要职责：
- `add`：启动时注册（由 `AddToBaseAppMgrHelper` 调用）。
- `onManagerRebirth`：baseappmgr 重生时重新注册（避免 baseappmgr 崩溃影响 baseapp）。
- `useNewBackupHash`：通知 baseappmgr“我已经切换到新备份 hash”。
- `informOfArchiveComplete`：通知 baseappmgr“我已经把死亡 baseapp 的备份归档完毕”。
- `registerBaseGlobally` / `deregisterBaseGlobally`：全局 base 注册/注销（带回复，用于 `Base::py_registerGlobally` 的回调）。

### 8.2 `AddToBaseAppMgrHelper`

已在 §2.4 详述。

### 8.3 `PingManager`

`PingManager`（[ping_manager.hpp](file:///workspace/src/server/baseapp/ping_manager.hpp)）是个极简的命名空间，提供 `init` / `fini`：

```cpp
namespace PingManager
{
    bool init( Mercury::EventDispatcher & dispatcher,
            Mercury::NetworkInterface & networkInterface );
    void fini();
}
```

它周期性地测量 baseapp 与 baseappmgr / cellappmgr / dbmgr 之间的延迟，用于 watcher 监控与负载决策。

---

## 9. 共享数据 / SQLite 离线缓存

### 9.1 `SharedDataManager`

`SharedDataManager`（[shared_data_manager.hpp](file:///workspace/src/server/baseapp/shared_data_manager.hpp)）维护两张跨 baseapp 共享的字典：

```cpp
class SharedDataManager
{
public:
    static shared_ptr< SharedDataManager > create( Pickler * pPickler );
    ~SharedDataManager();

    void setSharedData( BinaryIStream & data );
    void delSharedData( BinaryIStream & data );
    void addToStream( BinaryOStream & stream );
private:
    SharedDataManager();
    bool init( Pickler * pPickler );
    SharedData *    pBaseAppData_;      // 本 baseapp 私有但跨进程可见
    SharedData *    pGlobalData_;       // 全局共享
};
```

- `baseAppData`：每个 baseapp 一份，别的 baseapp 可读（用于跨进程查询某 baseapp 的状态）。
- `globalData`：所有 baseapp 共享一份（用于全局配置、跨进程锁等）。

变更由 baseappmgr 广播 `setSharedData` / `delSharedData` 给所有 baseapp。`Pickler` 负责把 Python 对象序列化为 blob。

### 9.2 `SqliteDatabase`（二级库）

`SqliteDatabase`（[sqlite_database.hpp](file:///workspace/src/server/baseapp/sqlite_database.hpp)）是 baseapp 本地的 SQLite 二级库，作为 dbmgr（MySQL）不可达时的本地落盘兜底：

```cpp
class SqliteDatabase
{
public:
    SqliteDatabase( const std::string & filename, const std::string & dir );
    ~SqliteDatabase();
    bool init( const std::string & checksum );
    const std::string& dbFilePath() const      { return path_; }

    void writeToDB( DatabaseID & dbID, EntityTypeID typeID, GameTime & time,
            MemoryOStream & stream, WriteToDBReplyStructPtr pReplyStruct ) const;
    void commit();
    void commitBgTask( Transaction * pTrans, bool shouldFlip );
    void commitFgTask( Transaction * pTrans );
    void shouldFlip( bool flag )               { shouldFlip_ = flag; }
    void isRegistered( bool flag )             { isRegistered_ = flag; }
    void tick();
private:
    bool open();
    void close();
    bool writeChecksumTable( const std::string & checksum ) const;
    void flipTable();
    // ...
};
```

设计要点：
- **双表 flip**：维护 `flipTable_` 与 `flopTable_` 两张表，写入累积到 `pCurrTable_`，commit 时通过 `flipTable` 切换——保证读取端看到的是一致的快照。
- **后台事务**：`Transaction` 攒一批 `Row`，由 `BgTaskManager` 在后台线程 `commit` 到 SQLite（不阻塞主 tick）。`TransactionPool` 复用 `Transaction` 对象避免频繁分配。
- **checksum 校验**：`writeChecksumTable` 写入 schema 的 MD5，启动时 `init(checksum)` 比对，schema 不一致则报错（避免老数据被新代码误读）。
- **与 dbmgr 同步**：`BaseApp::registerSecondaryDB` 把本二级库注册到 dbmgr，dbmgr 维护一个“二级库 → 主库”的同步队列（参见 `consolidate_dbs` 工具）。

`SecondaryDBConfig`（[secondary_db_config.hpp](file:///workspace/src/server/baseapp/secondary_db_config.hpp)）：

```cpp
class SecondaryDBConfig
{
public:
    static ServerAppOption< bool > enable;
    static ServerAppOption< float > maxCommitPeriod;
    static ServerAppOption< uint > maxCommitPeriodInTicks;
    static ServerAppOption< std::string > directory;
    static bool postInit();
};
```

---

## 10. 客户端下载流

### 10.1 `DataDownloads` / `DataDownload`

`DataDownloads`（[data_downloads.hpp](file:///workspace/src/server/baseapp/data_downloads.hpp)）管理向客户端推送的多个下载流：

```cpp
class DataDownloads
{
public:
    DataDownloads();
    ~DataDownloads();
    PyObject * streamToClient( DataDownloadFactory & factory,
           PyObjectPtr pDesc, int id );
    int addToBundle( Mercury::Bundle & bundle, int & remaining,
           IFinishedDownloads & finishedDownloads );
    bool push_back( DataDownload *pDL );
    bool contains( uint16 id );
    bool empty() const;
protected:
    typedef std::deque< DataDownload* > Container;
    Container::iterator erase( Container::iterator it );
    Container dls_;
    uint16 freeID_;
    std::set< uint16 > usedIDs_;
};

class DataDownload
{
public:
    DataDownload( PyObjectPtr pDesc, uint16 id, DataDownloads &dls );
    virtual void read( BinaryOStream &os, int nBytes ) = 0;
    virtual bool done() const = 0;
    virtual bool good() const { return good_; }
    int addToBundle( Mercury::Bundle & bundle, int & remaining );
    void onDone();
    uint16 id() const          { return id_; }
    void id( uint16 v )        { id_ = v; }
protected:
    virtual int available() const = 0;
    bool good_;
private:
    uint16 id_;
    uint8 seq_;
    PyObjectPtr pDesc_;
    int bytesSent_;
    int packetsSent_;
    uint64 start_;
};
```

`DataDownload` 是抽象基类，`read` / `available` / `done` 由子类实现。两个工厂：
- `StringDataDownloadFactory`：从 Python 字符串下载（`Proxy::streamStringToClient`）。
- `FileDataDownloadFactory`：从文件下载（`Proxy::streamFileToClient`），底层用 `FileStreamingJob` 后台读盘。

`DataDownloads::addToBundle` 在每个 tick 被 `Proxy` 调用，按 `DownloadStreamer` 给的预算把若干字节塞进客户端 bundle，并通知 `IFinishedDownloads::onFinished` 完成的下载。

### 10.2 `DownloadStreamer`

`DownloadStreamer`（[download_streamer.hpp](file:///workspace/src/server/baseapp/download_streamer.hpp)）是全局下载流控：

```cpp
class DownloadStreamer
{
public:
    DownloadStreamer();
    int maxDownloadRate() const;
    int curDownloadRate() const;
    int maxClientDownloadRate() const;
    int downloadRampUpRate() const;
    int downloadBacklogLimit() const;
    float downloadScaleBack() const;
    void modifyDownloadRate( int delta );
private:
    int curDownloadRate_;
};
```

它根据所有 `Proxy` 的总带宽预算与丢包情况，动态调整 `curDownloadRate_`（`Proxy::modifyDownloadRate` 调用）。每个 `Proxy` 在自己的 `downloadRate_` 上叠加 `apparentStreamingLimit_`（最近一次丢包时的实际速率），实现“试探性加速 + 丢包回退”。

---

## 11. Python 脚本绑定

### 11.1 `PyBases`

`PyBases`（[py_bases.hpp](file:///workspace/src/server/baseapp/py_bases.hpp)）把 `Bases` 集合暴露为 Python 字典对象 `BigWorld.bases`：

```cpp
class PyBases : public PyObjectPlus
{
    Py_Header( PyBases, PyObjectPlus )
public:
    PyBases( const Bases & bases, PyTypePlus * pType = &PyBases::s_type_ );
    PyObject *         pyGetAttribute( const char * attr );
    PyObject *         subscript( PyObject * entityID );
    int                length();
    PY_METHOD_DECLARE( py_has_key )
    PY_METHOD_DECLARE( py_keys )
    PY_METHOD_DECLARE( py_values )
    PY_METHOD_DECLARE( py_items )
    PY_METHOD_DECLARE( py_get )
    static PyObject *  s_subscript( PyObject * self, PyObject * entityID );
    static Py_ssize_t  s_length( PyObject * self );
private:
    const Bases & bases_;
};
```

脚本侧用法：

```python
for id, base in BigWorld.bases.items():
    print id, base.className
```

### 11.2 `PyCellData`

`PyCellData`（[py_cell_data.hpp](file:///workspace/src/server/baseapp/py_cell_data.hpp)）包装 cell 数据 blob，按需展开为 Python 字典：

```cpp
class PyCellData : public PyObjectPlus
{
    Py_Header( PyCellData, PyObjectPlus )
public:
    PyCellData( EntityTypePtr pEntityType, BinaryIStream & data,
        bool persistentDataOnly, PyTypePlus * pType = &PyCellData::s_type_ );
    PyObject *         pyGetAttribute( const char * attr );
    bool    addToStream( BinaryOStream & stream, bool addPosAndDir,
                        bool addPersistentOnly );
    void    migrate( EntityTypePtr pType );
    PyObjectPtr getDict();
    PY_AUTO_METHOD_DECLARE( RETDATA, getDict, END )
    PyObjectPtr createPyDictOnDemand();
    // ...
private:
    EntityTypePtr pEntityType_;
    PyObjectPtr   pDict_;
    std::string   data_;
    bool          dataHasPersistentOnly_;
};
```

关键点：`data_` 保留原始 blob，`pDict_` 仅在脚本访问时（`getDict` / `createPyDictOnDemand`）才反序列化。这对备份/恢复路径很重要——绝大多数 `PyCellData` 只是被透传，无需展开，节省 CPU。

### 11.3 `ScriptBigWorld`

`ScriptBigWorld`（[script_bigworld.hpp](file:///workspace/src/server/baseapp/script_bigworld.hpp)）是 baseapp 侧 `BigWorld` Python 模块的初始化入口：

```cpp
enum LogOnAttemptResult
{
    LOG_ON_REJECT = 0,
    LOG_ON_ACCEPT = 1,
    LOG_ON_WAIT_FOR_DESTROY = 2,
};

namespace BigWorldBaseAppScript
{
bool init( const Bases & bases,
        Mercury::EventDispatcher & dispatcher,
        Mercury::NetworkInterface & intInterface );
}
```

`BaseApp::initScript` 调用 `BigWorldBaseAppScript::init`，注册 `BigWorld.createBase*`、`BigWorld.bases`、`BigWorld.globalBases`、`BigWorld.addProxyFileData` 等 baseapp 专属 API。`LogOnAttemptResult` 枚举对应 `BaseApp::logOnAttempt` 的三种结果。

---

## 12. 客户端死亡 / 受控停机

### 12.1 `ControlledShutdownHandler`

`ControlledShutdown` 命名空间（[controlled_shutdown_handler.hpp](file:///workspace/src/server/baseapp/controlled_shutdown_handler.hpp)）启动受控停机流程：

```cpp
namespace ControlledShutdown
{
void start( SqliteDatabase * pSecondaryDB,
            const Bases & bases,
            Mercury::ReplyID replyID,
            const Mercury::Address & srcAddr );
}
```

受控停机的目标：把本 baseapp 的所有实体安全 offload 到其他 baseapp，并保证 DB 落盘 + 备份完成，最后才真正退出。流程大致：

1. `BaseApp::controlledShutDown` 收到 baseappmgr 的 `controlledShutDown` 消息（带 `stage`、`shutDownTime`）。
2. 调 `ControlledShutdown::start`：先把所有 base 写到二级库（`SqliteDatabase::commit`），再 `startOffloading`。
3. `Base::offload` 把实体通过 `BaseAppIntInterface::backupBaseEntity`（isOffload=true）发到目标 baseapp，目标 baseapp 调 `Base::readBackupData` 重建。
4. 实体 offload 完成后，本 baseapp 调 `BaseAppMgrGateway::useNewBackupHash` 通知 baseappmgr 切换 hash。
5. 所有实体 offload 完毕，回复 baseappmgr，baseappmgr 把该 baseapp 从列表删除，本 baseapp 退出。

### 12.2 `BaseMessageForwarder` / `EntityChannelFinder` / `MessageHandlers`

- `BaseMessageForwarder`：已在 §7.8 详述，负责 offload 期间的消息转发。
- `EntityChannelFinder`（[entity_channel_finder.hpp](file:///workspace/src/server/baseapp/entity_channel_finder.hpp)）：实现 `Mercury::ChannelFinder` 接口，在收到的包不属于任何已知 channel 时，按 channel id 查找实体——用于客户端 channel 与 entity 的关联。
- `MessageHandlers`（[message_handlers.hpp](file:///workspace/src/server/baseapp/message_handlers.hpp)）：聚合内/外接口 handler 的注册入口：

```cpp
namespace InternalInterfaceHandlers
{
void init( Mercury::InterfaceTable & interfaceTable );
}

namespace ExternalInterfaceHandlers
{
void init( Mercury::InterfaceTable & interfaceTable );
}
```

### 12.3 ID/Trace

- `IDConfig`（[id_config.hpp](file:///workspace/src/server/baseapp/id_config.hpp)）：EntityID 池阈值——`criticallyLowSize` / `lowSize` / `desiredSize` / `highSize`，由 `IDClient` 向 dbmgr 批量申请。
- `BWTracerHolder`（[bwtracer.hpp](file:///workspace/src/server/baseapp/bwtracer.hpp)）：管理 `BWTracer` 对象生命周期，用于网络 trace（包级别日志）。

---

## 13. 典型时序

### 13.1 baseapp 完整启动时序

```
main() (main.cpp)
  │ BWResource::init / BWConfig::init / bwParseCommandLine
  ▼
bwMainT<BaseApp> (bwservice.hpp)
  │ 创建 EventDispatcher + NetworkInterface(INTERNAL)
  │ SignalProcessor / BW_MESSAGE_FORWARDER3 / START_MSG
  ▼
doBWMainT<BaseApp>
  │ ServerAppConfig::init(BaseAppConfig::postInit)   ← 注册所有配置项
  │ 构造 BaseApp(dispatcher, interface)              ← BaseApp 构造函数
  │   ├─ EntityApp(dispatcher, interface)
  │   ├─ extInterface_(EXTERNAL)                     ← 第二条网卡，监听客户端
  │   └─ 各 shared_ptr 子组件构造（懒初始化）
  ▼
BaseApp::runApp(argc, argv)  [ServerApp]
  │ BaseApp::init(argc, argv)
  │   ├─ initScript()                                ← Python + BigWorldBaseAppScript::init
  │   ├─ EntityType::init()                          ← 加载 entities.xml + .def
  │   ├─ serveInterfaces()                           ← 注册 InternalInterfaceHandlers + ExternalInterfaceHandlers
  │   ├─ initSecondaryDB()                           ← SqliteDatabase::init(checksum)
  │   ├─ findOtherProcesses()                        ← 通过 machined 查 baseappmgr 地址
  │   ├─ AddToBaseAppMgrHelper(this)                 ← 构造即 send()
  │   │     └─ doSend(): BaseAppMgrGateway::add(intAddr, extAddr, this)
  │   ├─ PingManager::init
  │   └─ BWTracerHolder::init
  ▼
[等待 baseappmgr 回复 add]
  ▼
AddToBaseAppMgrHelper::handleMessage
  └─ finishInit(stream)
        ├─ BaseAppMgrGateway::send()                 ← flush pending ACKs
        ├─ stream >> BaseAppInitData{id, time, isReady}
        └─ BaseApp::finishInit(initData)
              ├─ id_ = initData.id
              ├─ pTimeKeeper_ = new TimeKeeper(time, ...)
              ├─ startGameTickTimer()                ← 注册 TIMEOUT_GAME_TICK
              ├─ ready(READY_BASE_APP_MGR)           ← waitingFor_ &= ~READY_BASE_APP_MGR
              └─ if bootstrap && autoLoad: 触发 dbmgr 自动加载实体
  ▼
[hasStarted() == true, 进入正常 tick]
  ▼
handleTimeout(TIMEOUT_GAME_TICK)                     ← 每 game tick
  ├─ tickGameTime()
  ├─ BackupSender::tick / Archiver::tick
  ├─ LoginHandler::tick / PendingLogins::tick
  ├─ DeadCellApps::tick
  ├─ tickRateLimitFilters() / sendIdleProxyChannels()
  ├─ SqliteDatabase::tick
  ├─ SharedDataManager 同步
  └─ Python 脚本 tick (Base::handleTimeout → ScriptTimers)
```

### 13.2 客户端登录时序（loginapp → baseapp）

```
客户端                loginapp              baseappmgr            dbmgr              baseapp
  │                      │                      │                  │                    │
  │──loginRequest────────▶                      │                  │                    │
  │                      │──checkStatus─────────────────────────────▶                    │
  │                      │                      │                  │                    │
  │                      │◀──loadEntity(account)────────────────────────────────────────│
  │                      │                      │                  │──loadEntity────────▶│ (找已有 baseapp 或新建)
  │                      │                      │                  │                    │
  │                      │                      │◀─────────────createBaseFromDB / createBase─│
  │                      │                      │                  │                    │
  │                      │                      │  baseappmgr 选 baseapp，下发 sessionKey │
  │                      │◀──baseappAddr+sessionKey────────────────────────────────────│
  │◀──baseappAddr+sessionKey──                  │                  │                    │
  │                      │                      │                  │                    │
  │  BaseApp::createBaseFromDB 收到 → 创建 Proxy(pType=Account) ──▶ Proxy 入 bases_      │
  │                      │                      │                  │                    │
  │  LoginHandler::add(proxy, loginAppAddr) → PendingLogins[sessionKey]=(proxy,loginAppAddr)
  │                      │                      │                  │                    │
  │──baseAppLogin(sessionKey)───────────────────────────────────────────────────────────▶│
  │                      │                      │                  │   LoginHandler::login
  │                      │                      │                  │   ├─ PendingLogins::find(sessionKey)
  │                      │                      │                  │   ├─ updateStatistics(NAT 判定)
  │                      │                      │                  │   └─ Proxy::attachToClient(clientAddr, replyID)
  │                      │                      │                  │       └─ setClientChannel(channel)
  │◀──loginOK + entitiesEnabled─────────────────────────────────────────────────────────│
  │                      │                      │                  │                    │
  │──enableEntities─────────────────────────────────────────────────────────────────────▶│
  │   Proxy::enableEntities → entitiesEnabled_=true                                  │
  │   Base::createCellEntity → cellapp 创建 cell 实体 → cellEntityCreated              │
  │   cellapp → baseapp: createCellPlayer / enterAoI / ... → Proxy forward → 客户端    │
  │◀──createCellPlayer / enterAoI / updateEntity───────────────────────────────────────│
```

### 13.3 玩家 base 实体创建/恢复时序

**新建玩家（首次进入）**：

```
脚本: BigWorld.createBaseFromDB("Account", playerName, callback)
  │
  ▼
EntityCreator::createBaseFromDB(type, name, pResultHandler)
  │ idClient_.getID()  ← 取一个 EntityID
  │ 向 dbmgr 发 loadEntity(entityType, name)
  ▼
[dbmgr 查 MySQL]
  │ 若无记录 → 返回 LOAD_FROM_DB_FAILED
  │ 若有记录 → 返回 LOAD_FROM_DB_SUCCEEDED + 二进制流
  ▼
LoadEntityHandlerWithCallback::handleMessage
  └─ onLoadedFromDB(pBase, NULL, dbID)
        ├─ pBase = EntityType::getType(typeID).create(id, dbID, data, persistentOnly)
        │     ├─ newEntityBase(id, dbID)
        │     ├─ createScript(data)         ← 构造 Python Account 对象
        │     └─ Base::init(pDict, pCellArgs)
        ├─ BaseApp::addBase(pBase)          ← bases_.add(base)
        ├─ base.createCellEntity()          ← 在 cellapp 创建 cell 实体
        │     └─ CreateCellEntityHandler 处理回复
        └─ 回调 Python callback(base)
```

**恢复玩家（baseapp 崩溃后从备份恢复）**：

```
baseapp#A 死亡
  │
  ▼
baseappmgr 检测到 (handleBaseAppDeath)
  │ 通知所有 baseapp: BaseApp::handleBaseAppDeath(addr, data)
  ▼
备份 baseapp#B 收到
  ├─ BackedUpBaseApps 里找到 addr#A 对应的 BackedUpBaseApp
  ├─ BackupHashChain::adjustForDeadBaseApp(addr#A, hash)  ← 把当时 hash 压入链
  ├─ BackedUpEntities::restore() 或 restoreRemotely(addr#A, hashChain)
  │     对每个备份的 entityID:
  │       ├─ Base::readBackupData(stream)  ← 反序列化为 Base 实例
  │       ├─ BaseApp::addBase(base)
  │       ├─ base.hasCellEntity? → base.restoreTo(spaceID, newCellAppAddr)  ← 在新 cellapp 重建 cell 实体
  │       └─ 若是 Proxy 且 hasClient: 等待客户端重连（restoreClient 流程）
  ├─ Archiver::handleBaseAppDeath(addr#A, bases, pSecondaryDB)  ← 把恢复的实体归档到 dbmgr
  └─ BaseAppMgrGateway::informOfArchiveComplete(addr#A)  ← 通知 baseappmgr 归档完毕
```

### 13.4 备份哈希环工作原理

```
                   实体 ID 空间
       ┌──────────────────────────────────────────┐
       │  E1   E2   E3   E4   E5   E6   E7   E8   │
       └──────────────────────────────────────────┘
                          │
                          ▼ hashFor(id) = (prime * id) % virtualSize
                          │   virtualSize = 2^ceil(log2(N))
                          │
       ┌──────────────────────────────────────────┐
       │ 虚拟桶 [0, virtualSize)                    │
       │  0    1    2    3    4    5    6    7     │
       └──────────────────────────────────────────┘
                          │
                          │ 桶号 >= N 时折半：bucket = vBucket 若 vBucket<N，否则 vBucket-N
                          ▼
       ┌──────────────────────────────────────────┐
       │ 真实桶 [0, N)  N=4 个 baseapp               │
       │  B0(B@10.0.0.1) B1(B@10.0.0.2)            │
       │  B2(B@10.0.0.3) B3(B@10.0.0.4)            │
       └──────────────────────────────────────────┘

扩容 B4 加入:
  N: 4 → 5, virtualSize: 4 → 8 (重新取 2 的幂)
  diff(old, new):
    对每个 entityID:
      oldAddr = oldHash.addressFor(id)
      newAddr = newHash.addressFor(id)
      if oldAddr != newAddr: onChange → 把备份从 oldAddr 迁到 newAddr
  迁移完成后 ackNewBackupHash → 切换 entityToAppHash_ = newEntityToAppHash_

B2 崩溃:
  N: 5 → 4
  diff: B2 上的实体按新 hash 重映射到剩余 baseapp
  BackupHashChain 记录 (B2.addr → 当时 hash)
  B2 的备份 baseapp 用 BackupHashChain 找到自己为之备份的实体 → restore
```

关键性质：
- **最小迁移**：扩容/缩容只迁移 `1/virtualSize` 的实体（折半技巧保证）。
- **随机化**：每个 baseapp 用不同 `prime_`，避免多个 baseapp 同时死亡时实体集中迁移。
- **历史链**：`BackupHashChain` 保留每次死亡时的 hash 快照，支持多级备份恢复。

---

## 14. 与 cellapp 的协作（补充）

baseapp 与 cellapp 的交互主要通过 `Base` 持有的 `pChannel_`（指向当前 cell 实体的 Mercury channel）和 `pCellEntityMailBox_`。关键消息（在 `CellAppInterface` 里定义，由 cellapp 处理）：

- `createCellEntity` / `createCellEntityNearEntity`：在 cellapp 上创建 cell 实体。
- `destroyCellEntity`：销毁 cell 实体。
- `callCellMethod`：base 调 cell 的实体方法（经 `CellEntityMailBox`）。

cellapp → baseapp 的消息（在 `BaseAppIntInterface` 里，§3.2.4）：
- `currentCell` / `emergencySetCurrentCell`：cell 实体迁到新 cell。
- `backupCellEntity`：cell 实体周期备份自己的状态给 base（用于 cellapp 崩溃后 base 重建 cell 实体）。
- `cellEntityLost`：cell 实体已销毁。
- `callBaseMethod`：cell 调 base 的方法。
- `getCellAddr`：cell 查询 base 的当前 cell 地址。
- `enterAoI` / `leaveAoI` / `createEntity` / `updateEntity` / `createCellPlayer` / `spaceData` / `avatarUpdate*`：cell → base → 客户端的转发链。

---

## 15. 小结

baseapp 是 BigWorld 服务端“逻辑/持久层”的承载者，它的设计有几个值得注意的工程取舍：

1. **base 与 cell 的双实体模型**：`Base`（baseapp）+ `Entity`（cellapp）共同表达一个游戏对象，前者管持久化与客户端连接，后者管空间。这种解耦让 cellapp 崩溃只影响空间、baseapp 崩溃只影响持久化，故障域隔离。
2. **一致性哈希 + 历史链的备份选址**：`BackupHash` + `BackupHashChain` 在最小迁移与可恢复性之间取得平衡，且通过随机 `prime_` 避免雪崩。
3. **双缓冲切换**：`BackedUpBaseApp` 的 `currentBackup_` / `newBackup_` 让 hash 切换过程中既能写新备份、又能读老备份，无锁。
4. **限流 + 缓冲**：`RateLimitMessageFilter` 把“每客户端每 tick/秒”的消息数与字节数都限制住，超限进 `BufferedMessage` 队列，保护 baseapp 主 tick 不被恶意客户端打爆。
5. **二级库兜底**：`SqliteDatabase` 在 dbmgr 不可达时本地落盘，配合 `consolidate_dbs` 工具后续同步回主库，是金融级“最终一致”思路的工程化。
6. **IDL 复用**：`common_client_interface.hpp` 在 baseapp/cellapp 两侧用不同宏展开，复用同一份消息定义，避免协议漂移。

阅读 baseapp 时建议按以下顺序：[baseapp.hpp](file:///workspace/src/server/baseapp/baseapp.hpp) → [base.hpp](file:///workspace/src/server/baseapp/base.hpp) → [proxy.hpp](file:///workspace/src/server/baseapp/proxy.hpp) → [baseapp_int_interface.hpp](file:///workspace/src/server/baseapp/baseapp_int_interface.hpp) → [backup_sender.hpp](file:///workspace/src/server/baseapp/backup_sender.hpp) + [backed_up_base_app.hpp](file:///workspace/src/server/baseapp/backed_up_base_app.hpp) + [backup_hash.hpp](file:///workspace/src/lib/server/backup_hash.hpp) → [archiver.hpp](file:///workspace/src/server/baseapp/archiver.hpp) → [login_handler.hpp](file:///workspace/src/server/baseapp/login_handler.hpp) + [pending_logins.hpp](file:///workspace/src/server/baseapp/pending_logins.hpp)。
