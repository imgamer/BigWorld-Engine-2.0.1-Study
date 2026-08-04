# BigWorld Engine 2.0.1 服务端通用框架（server）

> 源码位置：`src/lib/server/`
> 模块定位：所有 BigWorld 服务端进程（baseapp / cellapp / baseappmgr / cellappmgr / dbmgr / loginapp / reviver 等）共用的**公共基类与基础设施**。
> 角色：它不是某个具体业务进程，而是抽取了所有进程的共性——事件循环、网络接口、配置体系、信号处理、时间推进、脚本宿主、备份归档、ID 分配、Watcher 转发——形成一套“服务端微内核底座”。

---

## 1. 模块定位与核心职责

BigWorld 服务端是**多进程分布式**架构，每个进程角色不同，但都需要：

- 一个事件循环（`EventDispatcher`）驱动所有定时器与网络回调
- 一个网络接口（`NetworkInterface`）收发 Mercury 消息
- 一套配置体系（`bw.xml` 链式解析 + 声明式选项）
- 信号处理（SIGINT/SIGHUP 优雅退出）
- 游戏时间（game tick）推进
- Python 脚本宿主
- 与管理进程（mgr）的通信网关
- Watcher 运维监控接入

`src/lib/server/` 就是把这些共性抽成可复用的基类与工具，约 50 个源文件。具体进程（如 `src/server/cellapp/`）只需继承对应基类、实现纯虚方法、注册自己的 Mercury 接口即可。

### 1.1 与其它模块的关系

```
                    ┌─────────────────────────────────────┐
                    │        bwservice.hpp (入口)          │
                    │   BIGWORLD_MAIN 宏 / bwMainT 模板    │
                    └──────────────────┬──────────────────┘
                                       │ 创建
                                       ▼
                    ┌─────────────────────────────────────┐
                    │          ServerApp (根基类)          │
                    │  EventDispatcher + NetworkInterface │
                    │  + SignalProcessor + time()          │
                    └──────┬──────────────┬───────────────┘
                           │              │
              ┌────────────▼──┐     ┌─────▼──────────┐
              │   EntityApp   │     │  ManagerApp    │
              │ (baseapp/cellapp)│  │ (baseappmgr/   │
              │  + TimeQueue   │     │  cellappmgr)   │
              └───────┬───────┘     └────────────────┘
                      │
        ┌─────────────┼─────────────────┐
        │             │                 │
   ManagerAppGateway  IDClient        TimeKeeper
   (与 mgr 通信)    (ID 分配)        (时间同步)
```

---

## 2. 模块全景与文件清单

`src/lib/server/` 按职责可分为九大组：

| 组 | 代表文件 | 职责 |
| --- | --- | --- |
| 进程基类 | `server_app.hpp/cpp`、`entity_app.hpp/cpp`、`manager_app.hpp/cpp`、`manager_app_gateway.hpp/cpp` | 进程层级基类与 mgr 网关 |
| 配置体系 | `bwconfig.hpp/cpp`、`server_app_config.hpp/cpp`、`entity_app_config.hpp/cpp`、`manager_app_config.hpp/cpp`、`external_app_config.hpp/cpp`、`server_app_option.hpp/cpp`、`server_app_option_macros.hpp`、`config_reader.hpp/cpp`、`balance_config.hpp/cpp` | 配置解析 + 声明式选项 |
| 服务入口 | `bwservice.hpp/cpp`、`service.hpp/cpp` | main 宏 + Windows 服务封装 |
| ID 生成 | `id_client.hpp/cpp`、`id_generator.hpp` | 分层 ID 分配 |
| 时间 | `time_keeper.hpp/cpp` | game tick 同步 |
| 信号 | `signal_processor.hpp/cpp`、`signal_set.hpp` | 信号处理 |
| 脚本 | `python_server.hpp/cpp`、`script_timers.hpp/cpp`、`app_script_timers.cpp` | Python 宿主 + 脚本定时器 |
| 备份与归档 | `auto_backup_and_archive.hpp/cpp`、`backup_hash.hpp/cpp`、`backup_hash_chain.hpp/cpp`、`base_backup_switch_mailbox_visitor.hpp/cpp`、`migrate_mailbox_visitor.hpp` | 备份哈希环 + 迁移 |
| 共享数据/Watcher/其它 | `shared_data.hpp/cpp`、`shared_data_type.hpp`、`watcher_forwarding.hpp/cpp`、`watcher_forwarding_collector.hpp/cpp`、`watcher_forwarding_types.hpp`、`watcher_protocol.hpp/cpp`、`server_info.hpp/cpp`、`event_history_stats.hpp/cpp`、`plugin_library.hpp/cpp`、`add_to_manager_helper.hpp/cpp`、`anonymous_channel_client.hpp/cpp`、`common.hpp`、`reviver_common.hpp`、`reviver_subject.hpp/cpp`、`stream_helper.hpp`、`util.hpp`、`writedb.hpp`、`cell_app_init_data.hpp` | 跨进程共享数据 / 监控转发 / 插件 / 启动辅助 / Reviver / 流工具 |

---

## 3. 进程基类层级

### 3.1 ServerApp —— 所有进程的根基类

定义于 [server_app.hpp](file:///workspace/src/lib/server/server_app.hpp)，是所有服务端进程的根基类，持有事件循环、网络接口、信号处理器与游戏时间。

```cpp
class ServerApp
{
public:
    ServerApp( Mercury::EventDispatcher & mainDispatcher,
            Mercury::NetworkInterface & interface );

    // 从消息头取回所属 ServerApp 实例（Mercury 回调中常用）
    template <class APP_TYPE>
    static APP_TYPE & getApp( const Mercury::UnpackedMessageHeader & header );

    virtual bool init( int argc, char * argv[] );
    virtual bool run();
    virtual void shutDown();
    bool runApp( int argc, char * argv[] );        // 顶层入口

    GameTime time() const { return time_; }
    double gameTimeInSeconds() const;              // time_ / updateHertz

    virtual void onSignalled( int sigNum );        // 信号回调

    static const char * shutDownStageToString( ShutDownStage value );

protected:
    void enableSignalHandler( int sigNum, bool enable=true );
    virtual SignalHandler * createSignalHandler();
    void addWatchers( Watcher & watcher );

    GameTime time_;
    Mercury::EventDispatcher & mainDispatcher_;
    Mercury::NetworkInterface & interface_;
private:
    std::auto_ptr< SignalHandler > pSignalHandler_;
};
```

**`SERVER_APP_HEADER` 宏**（见 [server_app.hpp](file:///workspace/src/lib/server/server_app.hpp) 第 32 行）让子类声明自己的进程名与配置路径根：

```cpp
#define SERVER_APP_HEADER( APP_NAME, CONFIG_PATH ) 							\
	static const char * appName()				{ return #APP_NAME; }		\
	static const char * configPath()			{ return #CONFIG_PATH; }	\
	virtual const char * getAppName() const		{ return #APP_NAME; }		\
	virtual const char * getConfigPath() const	{ return #CONFIG_PATH; }
```

例如 `CellApp` 会写 `SERVER_APP_HEADER( CellApp, cellApp )`，使其配置路径前缀为 `cellApp/`。

**`runApp` 生命周期**（见 [server_app.cpp](file:///workspace/src/lib/server/server_app.cpp)）：

```cpp
bool ServerApp::runApp( int argc, char * argv[] )
{
    stampsPerSecond();                    // 校准时钟
    if (!this->init( argc, argv )) {      // 子类重写：注册接口、连 mgr 等
        ERROR_MSG( "Failed to initialise %s\n", this->getAppName() );
        return false;
    }
    INFO_MSG( "---- %s is running ----\n", this->getAppName() );
    bool result = this->run();            // 子类重写：主循环
    interface_.prepareForShutdown();      // 网络层收尾
    return result;
}
```

**`init` 默认实现**：检测 `-machined` 参数（是否由 bwmachined 拉起）、创建信号处理器、注册 SIGINT/SIGHUP。

**`shutDown` 默认实现**：调用 `mainDispatcher_.breakProcessing()` 打破事件循环。

**`onSignalled` 默认实现**：SIGINT/SIGHUP 都触发 `shutDown()`。

**`ShutDownStage` 枚举**（定义于 [common.hpp](file:///workspace/src/lib/server/common.hpp)）描述受控关闭的阶段：

| 阶段 | 含义 |
| --- | --- |
| `SHUTDOWN_NONE` | 未关闭 |
| `SHUTDOWN_REQUEST` | 请求开始受控关闭（可能级联到其它进程） |
| `SHUTDOWN_INFORM` | 通知父进程当前关闭进度 |
| `SHUTDOWN_DISCONNECT_PROXIES` | BaseApp 断开代理 |
| `SHUTDOWN_PERFORM` | 立即关闭本进程 |
| `SHUTDOWN_TRIGGER` | 从最“资深”进程触发全局关闭（LoginApp → DBMgr → BaseAppMgr） |

### 3.2 EntityApp —— BaseApp 与 CellApp 的共同基类

定义于 [entity_app.hpp](file:///workspace/src/lib/server/entity_app.hpp)，继承 `ServerApp`，增加 `TimeQueue`（脚本定时器队列）与退休请求。

```cpp
class EntityApp : public ServerApp
{
public:
    virtual bool init( int argc, char * argv[] );
    virtual void requestRetirement();     // 请求“退休”（让 mgr 把负载迁走后退出）
    TimeQueue & timeQueue() { return timeQueue_; }
    virtual void onSignalled( int sigNum );
protected:
    virtual ManagerAppGateway & managerAppGateway() = 0;  // 纯虚：子类提供与 mgr 的网关
    void tickStats();                                       // 每 tick 统计
    TimeQueue timeQueue_;
    uint32 tickStatsPeriod_;
};
```

`ENTITY_APP_HEADER` 宏就是 `SERVER_APP_HEADER`（仅为语义清晰）。`requestRetirement` 是“优雅退出”的入口——与直接 `shutDown` 不同，退休会让 mgr 把该进程上的实体迁移到其它进程后再退出，用于滚动升级或缩容。

### 3.3 ManagerApp —— baseappmgr / cellappmgr 的基类

定义于 [manager_app.hpp](file:///workspace/src/lib/server/manager_app.hpp)，非常薄，仅继承 `ServerApp` 并提供 `addWatchers`。管理进程的复杂逻辑（负载均衡、进程分配）在具体子类 `src/server/baseappmgr/` 与 `src/server/cellappmgr/` 中。

### 3.4 ManagerAppGateway —— 与 mgr 通信的网关

定义于 [manager_app_gateway.hpp](file:///workspace/src/lib/server/manager_app_gateway.hpp)。baseapp/cellapp 通过它向对应的管理进程（baseappmgr/cellappmgr）通信，本质是一个 Mercury Channel 封装。

```cpp
class ManagerAppGateway
{
public:
    ManagerAppGateway( Mercury::NetworkInterface & networkInterface,
        const Mercury::InterfaceElement & retireAppIE );

    bool init( const char * interfaceName, int numRetries );   // 连接 mgr
    void addWatchers( const char * name, Watcher & watcher );
    void isRegular( bool localValue, bool remoteValue = false ); // 设为常规 channel
    bool isInitialised() const;                                  // 地址是否已就绪
    void retireApp();                                            // 通知 mgr 本进程退休
protected:
    Mercury::ChannelOwner channel_;            // 底层 Mercury channel
private:
    const Mercury::InterfaceElement & retireAppIE_;   // 退休消息的接口描述
};
```

`init` 通过 `interfaceName`（如 `"cellappmgr"`）向 bwmachined 查询 mgr 的地址，建立可靠 channel。`retireApp` 发送 `retireAppIE_` 消息通知 mgr 本进程要退休。

---

## 4. 配置体系

BigWorld 的配置体系是**链式 XML + 声明式选项**两层结构。

### 4.1 BWConfig —— 全局配置入口

定义于 [bwconfig.hpp](file:///workspace/src/lib/server/bwconfig.hpp)。入口文件是 `server/bw.xml`，可通过 `<parentFile>` 节点链式引用其它配置文件，形成一条配置链。

```cpp
namespace BWConfig
{
    typedef std::pair< DataSectionPtr, std::string > ConfigElement;
    typedef std::vector< ConfigElement > Container;

    bool init( int argc, char * argv[] );           // 解析 bw.xml 链

    DataSectionPtr getSection( const char * path, DataSectionPtr defaultValue = NULL );

    // 模板方法：从配置链读取值，找不到则保留默认
    template <class C>
    bool update( const char * path, C & value );

    template <class C>
    C get( const char * path, const C & defaultValue );

    // 遍历配置链中所有匹配 path 的 section（跨多个 parentFile）
    class SearchIterator;
    SearchIterator beginSearchSections( const char * path );
    const SearchIterator & endSearch();
};
```

`SearchIterator` 是关键——它用一个 `ConfigElementQueue`（配置链尾部）+ `DataSection::SearchIterator`（当前文件内搜索）实现跨文件遍历，让“在多个 bw.xml 片段中查找同名配置项”成为可能。`shouldDebug` 开关会打印每次读取的命中情况与新旧值对比，便于排查配置问题。

`BW_REGISTER_WATCHER` 宏（见 `bwconfig.hpp` 末尾）把进程的 Watcher Nub 注册到 bwmachined，使运维工具能远程访问本进程的 Watcher 树。

### 4.2 ServerAppOption —— 声明式选项框架

定义于 [server_app_option.hpp](file:///workspace/src/lib/server/server_app_option.hpp)。这是 BigWorld 配置体系的精髓——用一个模板类把“读配置 + 注册 Watcher + 启动打印”三件事封装成一行声明。

```cpp
template <class TYPE>
class ServerAppOption
{
public:
    ServerAppOption( TYPE value, const char * configPath,
            const char * watcherPath,
            Watcher::Mode watcherMode = Watcher::WT_READ_WRITE );
    TYPE operator()() const { return value_; }    // 像函数一样取值
    void set( TYPE value );
private:
    TYPE value_;
};
```

其构造时创建一个 `ServerAppOptionIniterT<TYPE>`（继承 `ServerAppOptionIniter`，用 `IntrusiveObject` 串成全局链表），在 `ServerAppConfig::init` 时统一调用 `initAll()`：

```cpp
virtual void init()
{
    if (configPath_[0]) {
        BWConfig::update( configPath_, value_ );    // 从 bw.xml 读值
    }
    if (watcherPath_[0]) {
        MF_WATCH( watcherPath_, value_, watcherMode_ );  // 注册 Watcher
    }
}
```

这样每个配置选项既能在 `bw.xml` 中配置，又能在运行时通过 Watcher 修改，还能在启动日志中打印当前值与默认值的差异。

### 4.3 ServerAppOptionMacros —— 选项声明宏集

定义于 [server_app_option_macros.hpp](file:///workspace/src/lib/server/server_app_option_macros.hpp)。要求使用前定义 `BW_CONFIG_CLASS`（如 `CellAppConfig`）与 `BW_CONFIG_PREFIX`（如 `"cellApp/"`），然后即可用宏声明选项：

```cpp
#define BW_OPTION( TYPE, NAME, VALUE )          // 读写选项
#define BW_OPTION_RO( TYPE, NAME, VALUE )       // 只读选项
#define BW_OPTION_AT( TYPE, NAME, VALUE, CONFIG_PATH )   // 自定义配置路径
#define BW_OPTION_FULL( TYPE, NAME, VALUE, CONFIG_PATH, WATCHER_PATH )  // 完全自定义
#define DERIVED_BW_OPTION( TYPE, NAME )         // 派生选项（仅 Watcher，不读配置）
```

例如在 `cell_app_config.cpp` 中：

```cpp
BW_OPTION( int, shutdownTimeout, 30 );   // 展开为 cellApp/shutdownTimeout 配置项
```

### 4.4 各级 Config 类

| 类 | 文件 | 继承 | 用途 |
| --- | --- | --- | --- |
| `ServerAppConfig` | [server_app_config.hpp](file:///workspace/src/lib/server/server_app_config.hpp) | — | 所有进程通用：`updateHertz`、`personality`、`isProduction`、`timeSyncPeriod`、`useDefaultSpace` |
| `EntityAppConfig` | [entity_app_config.hpp](file:///workspace/src/lib/server/entity_app_config.hpp) | `ServerAppConfig` | baseapp/cellapp：`numStartupRetries` |
| `ManagerAppConfig` | [manager_app_config.hpp](file:///workspace/src/lib/server/manager_app_config.hpp) | `ServerAppConfig` | mgr 进程：`shutDownServerOnBadState`、`shutDownServerOnBaseAppDeath`、`shutDownServerOnCellAppDeath` |
| `ExternalAppConfig` | [external_app_config.hpp](file:///workspace/src/lib/server/external_app_config.hpp) | — | 对外（客户端）进程：`externalLatencyMin/Max`、`externalLossRatio`、`externalInterface` |
| `BalanceConfig` | [balance_config.hpp](file:///workspace/src/lib/server/balance_config.hpp) | — | 负载均衡：`maxCPUOffload`、`minEntityOffload`、`minMovement`、`slowApproachFactor`、`demo` |

`ServerAppConfig` 还提供时间换算助手：`secondsToStamps`、`secondsToTicks`、`expectedTickPeriod`（= `1/updateHertz`）。

### 4.5 ConfigReader —— INI 风格配置读取

定义于 [config_reader.hpp](file:///workspace/src/lib/server/config_reader.hpp)。与 `BWConfig` 的 XML 链不同，`ConfigReader` 读取 Windows INI 风格的 `[section] key=value` 文件，主要用于服务端工具（非主进程）。

### 4.6 BWService —— 服务定位

定义于 [bwservice.hpp](file:///workspace/src/lib/server/bwservice.hpp)。它不是“服务发现”，而是**进程入口的封装**，包含：

- `START_MSG` 宏：打印进程启动 banner（版本、配置、构建时间、UID、PID、CPU/RAM 信息、资源路径）
- `getBWInternalInterfaceSetting`：从配置选择内网接口
- `doBWMainT<SERVER_APP>` 模板：初始化配置 → 创建 app → runApp
- `bwMainT<SERVER_APP>` 模板：创建 EventDispatcher + NetworkInterface + SignalProcessor + 消息转发器 → 调 `doBWMainT` → `postDestruction`

**`BIGWORLD_MAIN` 宏**（Linux 版）：

```cpp
#define BIGWORLD_MAIN										\
bwMain( int argc, char * argv[] );							\
int main( int argc, char * argv[] )							\
{															\
	BWResource bwresource;									\
	BWResource::init( argc, (const char **)argv );			\
	BWConfig::init( argc, argv );							\
	bwParseCommandLine( argc, argv );						\
	return bwMain( argc, argv );							\
}															\
int bwMain
```

具体进程的 `main.cpp` 只需写：

```cpp
BIGWORLD_MAIN( int argc, char * argv[] )
{
    return bwMainT<CellApp>( argc, argv );
}
```

Windows 版（`_WIN32`）额外封装了 `BigWorldService`（继承 `CService`），支持以 Windows Service 方式运行、`-install`/`-remove` 注册/卸载服务、`-machined` 由 bwmachined 拉起时跳过 Service 包装。

---

## 5. ID 生成

BigWorld 的 EntityID 是全局唯一的 32 位整数，采用**分层预分配**策略：dbmgr 是 ID 总源，baseappmgr/cellappmgr 向 dbmgr 批量取 ID 块，再分给 baseapp/cellapp，后者再分给具体实体。

### 5.1 IDGenerator —— 接口

定义于 [id_generator.hpp](file:///workspace/src/lib/server/id_generator.hpp)。纯虚接口，定义“取块 / 还块”：

```cpp
struct IDBlock { uint32 start; uint32 end; };
TYPEDEF_STREAMING_VECTOR( IDBlock, IDBlockVector );

class IDGenerator
{
public:
    virtual void getBlocks( IDBlockVector& blockVector, uint32 count, uint32 tolerance ) = 0;
    virtual void putBlocks( const IDBlockVector& blockVector ) = 0;
};
```

`IDBlock` 表示一段连续 ID 区间 `[start, end]`。`tolerance` 允许返回的块数与请求量有偏差（避免为凑数而拆碎连续块）。

### 5.2 IDClient —— 客户端 ID 池

定义于 [id_client.hpp](file:///workspace/src/lib/server/id_client.hpp)。每个 baseapp/cellapp 持有一个 `IDClient`，维护本地 ID 池，按水位线自动向上游（mgr）取/还 ID。

```cpp
class IDClient : private Mercury::ShutdownSafeReplyMessageHandler
{
public:
    bool init( Mercury::ChannelOwner * pChannelOwner,
        const Mercury::InterfaceElement& getMoreMethod,
        const Mercury::InterfaceElement& putBackMethod,
        size_t criticallyLowSize, size_t lowSize,
        size_t desiredSize, size_t highSize );

    void putUsedID( EntityID );     // 归还已用 ID（异步锁定一段时间防复用）
    EntityID getID();               // 取一个新 ID
    void returnIDs();               // 进程退出时还回所有 ID
private:
    typedef std::queue<EntityID> IDQueue;
    IDQueue readyIDs_;              // 就绪可用的 ID
    std::queue<LockedID> lockedIDs_;  // 刚释放、暂锁定的 ID（防短期复用）
    size_t highSize_, desiredSize_, lowSize_, criticallyLowSize_;
    bool pendingRequest_;           // 是否已向上游请求（防贪婪）
    bool inEmergency_;              // 紧急模式（水位过低，放大请求量）
};
```

**水位线机制**：`performUpdates` 根据当前 `readyIDs_` 大小决策——低于 `lowSize_` 则 `getMoreIDs()` 向上游请求；高于 `highSize_` 则 `putBackIDs()` 还回一部分；低于 `criticallyLowSize_` 进入紧急模式，放大请求量并告警。`LockedID` 带 `unlockTime_`，过期后才回到 `readyIDs_`，防止刚销毁的实体 ID 被立即复用导致客户端混淆。

---

## 6. 时间推进

### 6.1 TimeKeeper

定义于 [time_keeper.hpp](file:///workspace/src/lib/server/time_keeper.hpp)。负责 game tick 的推进与跨进程时间同步。

```cpp
class TimeKeeper : public TimerHandler,
    public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    TimeKeeper( Mercury::NetworkInterface & interface,
        TimerHandle trackingTimerHandle, GameTime & gameTime,
        int idealTickFrequency );

    // 带主时钟地址的版本：作为从节点向主同步
    TimeKeeper( Mercury::NetworkInterface & interface,
        TimerHandle trackingTimerHandle, GameTime & gameTime,
        int idealTickFrequency,
        const Mercury::Address & masterAddress,
        const Mercury::InterfaceElement * pMasterRequest );

    bool inputMasterReading( double reading );     // 接收主时钟读数
    double readingAtLastTick() const;
    double readingNow() const;
    double readingAtNextTick() const;

    void synchroniseWithPeer( const Mercury::Address & address,
        const Mercury::InterfaceElement & request );
    void synchroniseWithMaster();
private:
    virtual void handleTimeout( TimerHandle handle, void * arg );
    Mercury::NetworkInterface & interface_;
    TimerHandle trackingTimerHandle_;
    GameTime & gameTime_;                          // 引用 ServerApp::time_
    double idealTickFrequency_;                    // 理想 tick 频率（如 10Hz）
    uint64 nominalIntervalStamps_;                 // 名义 tick 间隔（CPU 戳数）
    bool isMaster_;                                // 是否是主时钟
    Mercury::Address masterAddress_;               // 主时钟地址
    const Mercury::InterfaceElement * pMasterRequest_;
};
```

**主从同步**：`isMaster_` 为真的进程（通常是 cellappmgr 或 baseappmgr）作为主时钟源；从进程周期性（`timeSyncPeriod`）调用 `synchroniseWithMaster()` 向主发请求，主通过 `inputMasterReading` 回复读数，从据此调整自己的 tick 节奏（`offsetOfReading` 计算偏差）。这保证整个集群的 game tick 对齐——对 AOI、ghost 同步、客户端插值至关重要。

`handleTimeout` 在每个 tick 触发，递增 `gameTime_`（即 `ServerApp::time_`）。

---

## 7. 信号处理

### 7.1 SignalProcessor

定义于 [signal_processor.hpp](file:///workspace/src/lib/server/signal_processor.hpp)。继承 `Mercury::FrequentTask` 与 `Singleton`，是进程级的信号分发中心。

```cpp
class SignalProcessor : private Mercury::FrequentTask,
    public Singleton< SignalProcessor >
{
public:
    SignalProcessor( Mercury::EventDispatcher & dispatcher );
    void ignoreSignal( int sigNum );
    void setDefaultSignalHandler( int sigNum );
    void addSignalHandler( int sigNum, SignalHandler * pSignalHandler, int flags = 0 );
    void clearSignalHandlers( int sigNum );
    void clearSignalHandler( int sigNum, SignalHandler * pSignalHandler );
    int waitForSignals( const Signal::Set & signalSet );
    static const char * signalNumberToString( int sigNum );
private:
    virtual void doTask() { this->dispatch(); }   // FrequentTask 回调
    void dispatch();
    void dispatchSignal( int sigNum );
    typedef std::multimap< int, SignalHandler * > SignalHandlers;
    SignalHandlers signalHandlers_;               // 信号号 -> 多个 handler
    SignalHandler * pSigQuitHandler_;
    Signal::Set signals_;                         // 待处理的信号集
};
```

**关键设计**：信号不在信号处理函数中直接执行业务逻辑（不安全），而是用 `Signal::Set` 记录待处理信号，由 `FrequentTask::doTask` 在事件循环中**派发**（`dispatch`）。这避免了信号处理函数与主线程的竞态。`multimap` 允许同一信号号注册多个 handler。

`SignalHandler` 是抽象基类：

```cpp
class SignalHandler
{
public:
    virtual void handleSignal( int sigNum ) = 0;
};
```

`ServerApp::createSignalHandler` 默认创建 `ServerAppSignalHandler`（见 [server_app.cpp](file:///workspace/src/lib/server/server_app.cpp)），它把信号转发给 `ServerApp::onSignalled`。

### 7.2 SignalSet

定义于 [signal_set.hpp](file:///workspace/src/lib/server/signal_set.hpp)。`Signal::Set` 是 `sigset_t` 的薄封装，`Signal::Blocker` 是 RAII 信号阻塞器（构造时 `pthread_sigmask` 设置阻塞集，析构时恢复），用于在敏感区段临时屏蔽信号。

---

## 8. 脚本宿主

### 8.1 PythonServer

定义于 [python_server.hpp](file:///workspace/src/lib/server/python_server.hpp)。当 `ENABLE_PYTHON_TELNET_SERVICE` 开启时，提供一个可通过 **telnet 远程连接的 Python 交互式解释器**，用于线上调试。

```cpp
class PythonServer : public PyOutputWriter, public Mercury::InputNotificationHandler
{
public:
    bool startup( Mercury::EventDispatcher & dispatcher,
        uint32_t ip, uint16_t port, const char * ifspec = 0 );
    void shutdown();
    void deleteConnection( PythonConnection* pConnection );
private:
    std::vector<PythonConnection*> connections_;
    Endpoint listener_;                          // TCP 监听 socket
    Mercury::EventDispatcher * pDispatcher_;
    PyObject * prevStderr_, * prevStdout_;       // 临时接管 stdout/stderr
};
```

`PythonConnection` 处理单条 telnet 连接，支持行编辑、历史（上/下键）、telnet 子协商、VT 指令。`PythonServer` 在启动时把 `sys.stdout`/`sys.stderr` 重定向到自己，使脚本 print 输出能转发到 telnet 客户端。

### 8.2 ScriptTimers

定义于 [script_timers.hpp](file:///workspace/src/lib/server/script_timers.hpp)。管理一组“带脚本 ID”的定时器，让 Python 脚本能创建/删除定时器。

```cpp
class ScriptTimers
{
public:
    typedef int32 ScriptID;
    static void init( EntityApp & app );
    static void fini( EntityApp & app );

    ScriptID addTimer( float initialOffset, float repeatOffset, int userArg,
            TimerHandler * pHandler );
    bool delTimer( ScriptID timerID );
    void cancelAll();
    void writeToStream( BinaryOStream & stream ) const;     // 持久化（用于备份）
    void readFromStream( BinaryIStream & stream, uint32 numTimers, TimerHandler * pHandler );
private:
    typedef std::map< ScriptID, TimerHandle > Map;
    Map map_;
};
```

定时器挂在 `EntityApp::timeQueue_` 上。`writeToStream` / `readFromStream` 让定时器能在 baseapp 备份/恢复时迁移，保证实体切换 baseapp 后定时器不丢失。`ScriptTimersUtil` 命名空间提供指针式接口（`ScriptTimers**` 懒初始化），避免无定时器时分配内存。

### 8.3 AppScriptTimers

`app_script_timers.cpp`（无对应头文件）是应用层的胶水：维护一个全局 `g_pTimers`（`ScriptTimers*`），注册 `ScriptTimerHandler`（持有 `PyObject*`，超时时回调脚本），并注册 `TimerFini`（`Script::FiniTimeJob`）保证 Python 解释器关闭前清理定时器。

---

## 9. 备份与归档

### 9.1 AutoBackupAndArchive

定义于 [auto_backup_and_archive.hpp](file:///workspace/src/lib/server/auto_backup_and_archive.hpp)。定义备份/归档策略枚举：

```cpp
namespace AutoBackupAndArchive
{
    enum Policy { NO = 0, YES = 1, NEXT_ONLY = 2 };
    void addNextOnlyConstant( PyObject * pModule );   // 向 Python 模块注入 NEXT_ONLY 常量
}
```

`NEXT_ONLY` 表示“仅下次备份一次”，用于触发即时备份而不改变长期策略。`Script::setData` 提供 Python 字符串到 `Policy` 的转换。

### 9.2 BackupHash —— 备份哈希环

定义于 [backup_hash.hpp](file:///workspace/src/lib/server/backup_hash.hpp)。baseapp 的核心容错机制——把所有 baseapp 的地址组织成一个**一致性哈希环**，每个 entity ID 哈希到某个 baseapp 作为其“备份 baseapp”。

```cpp
class MiniBackupHash
{
public:
    MiniBackupHash( uint32 prime = 0, uint32 size = 0 );
    uint32 hashFor( EntityID id ) const;        // id -> 桶号
    uint32 virtualSize() const;                  // 虚拟大小（>= size 的 2 的幂）
protected:
    void handleSizeChange( uint32 newSize );     // 调整 virtualSize_
    uint32 prime_;                               // 随机质数（扰动）
    uint32 size_;                                // 实际桶数
    uint32 virtualSize_;                         // 虚拟桶数
};

class BackupHash : public MiniBackupHash
{
public:
    BackupHash();                                // 构造时 choosePrime() 选随机质数
    Mercury::Address addressFor( EntityID id ) const;   // id -> baseapp 地址
    void diff( const BackupHash & other, DiffVisitor & visitor );  // 对比两个环的差异
    void push_back( const Mercury::Address & addr );
    bool erase( const Mercury::Address & addr );
private:
    typedef std::vector< Mercury::Address > Container;
    Container addrs_;
};
```

**一致性哈希**：`virtualSize_` 是不小于 `size_` 的 2 的幂，`hashFor(id)` 用 `prime_` 做扰动后取模到 `virtualSize_`，再映射到 `size_` 个实际桶。当 baseapp 增减时，`diff` 通过 `DiffVisitor`（`onAdd`/`onChange`/`onRemove`）通知受影响的 entity 迁移，最小化迁移量。`prime_` 随机化避免多个 baseapp 死亡后分布恶化。

`MiniBackupHash` 是不含地址列表的精简版（只有 prime + size），用于只需哈希计算不需地址查表的场景。两者都支持 `BinaryStream` 序列化。

### 9.3 BackupHashChain —— 哈希环历史链

定义于 [backup_hash_chain.hpp](file:///workspace/src/lib/server/backup_hash_chain.hpp)。保存历史上所有版本的 `BackupHash`，用于 baseapp 死亡后定位“旧的备份映射”。

```cpp
class BackupHashChain
{
public:
    void adjustForDeadBaseApp( const Mercury::Address & deadApp, const BackupHash & hash );
    Mercury::Address addressFor( const Mercury::Address & address, EntityID entityID ) const;
private:
    typedef std::map< Mercury::Address, BackupHash > History;
    History history_;    // 死亡 baseapp 地址 -> 其死亡时的哈希环
};
```

`adjustForDeadBaseApp` 记录死亡 baseapp 当时的哈希环快照；`addressFor` 给定一个（可能已失效的）baseapp 地址与 entity ID，沿历史链找到该 entity 现在应该去的备份 baseapp。

### 9.4 Mailbox 访问者

定义于 [base_backup_switch_mailbox_visitor.hpp](file:///workspace/src/lib/server/base_backup_switch_mailbox_visitor.hpp) 与 [migrate_mailbox_visitor.hpp](file:///workspace/src/lib/server/migrate_mailbox_visitor.hpp)。当 baseapp 死亡、备份切换时，需要遍历所有 Mailbox 把指向死亡 baseapp 的引用重映射到新 baseapp。

```cpp
class BaseBackupSwitchMailBoxVisitor : public PyEntityMailBoxVisitor
{
public:
    BaseBackupSwitchMailBoxVisitor( const BackupHashChain & hashChain,
            const Mercury::Address & deadAddr = Mercury::Address::NONE );
    virtual void onMailBox( PyEntityMailBox * pMailBox );   // 用 hashChain 重映射地址
private:
    Mercury::Address deadAddr_;
    const BackupHashChain & hashChain_;
};

class MigrateMailBoxVisitor : public PyEntityMailBoxVisitor
{
    virtual void onMailBox( PyEntityMailBox * pMailBox ) { pMailBox->migrate(); }
};
```

`MigrateMailBoxVisitor` 更简单——只调用 `migrate()` 让 Mailbox 重新解析自己的地址（用于实体跨 baseapp 迁移后）。

---

## 10. 共享数据

### 10.1 SharedData

定义于 [shared_data.hpp](file:///workspace/src/lib/server/shared_data.hpp)。提供一个**跨进程共享的“类字典”Python 对象**，让脚本能在不同进程间共享配置/状态。

```cpp
class SharedData : public PyObjectPlus
{
public:
    SharedData( SharedDataType dataType,
        SharedData::SetFn setFn, SharedData::DelFn delFn,
        SharedData::OnSetFn onSetFn, SharedData::OnDelFn onDelFn,
        Pickler * pPickler, ... );

    // Python 字典协议
    PyObject * subscript( PyObject * key );
    int ass_subscript( PyObject * key, PyObject * value );
    int length();
    PY_METHOD_DECLARE( py_has_key )
    PY_METHOD_DECLARE( py_keys )
    PY_METHOD_DECLARE( py_values )
    PY_METHOD_DECLARE( py_items )

    bool setValue( const std::string & key, const std::string & value );
    bool delValue( const std::string & key );
    bool addToStream( BinaryOStream & stream ) const;
private:
    std::string pickle( PyObject * pObj ) const;      // 用 Pickler 序列化值
    PyObject * unpickle( const std::string & str ) const;
    PyObject * pMap_;
    SharedDataType dataType_;
    SetFn setFn_;    // 值变化时的回调（向其它进程传播）
    DelFn delFn_;
};
```

`SetFn` / `DelFn` 是回调函数指针——当脚本修改共享数据时，`SharedData` 调用这些回调把变更通过网络传播给其它进程；`OnSetFn` / `OnDelFn` 则是收到远端变更后在本地触发。`Pickler` 负责 Python 对象与字符串的互转（因为跨进程传输需要序列化）。

### 10.2 SharedDataType

定义于 [shared_data_type.hpp](file:///workspace/src/lib/server/shared_data_type.hpp)。区分共享数据的作用域：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `SHARED_DATA_TYPE_CELL_APP` | 1 | 所有 cellapp 共享 |
| `SHARED_DATA_TYPE_BASE_APP` | 2 | 所有 baseapp 共享 |
| `SHARED_DATA_TYPE_GLOBAL` | 3 | 全局共享（cell + base） |
| `SHARED_DATA_TYPE_GLOBAL_FROM_BASE_APP` | 4 | 全局，由 baseapp 发起 |

---

## 11. Watcher 转发

BigWorld 的 Watcher 系统让运维能远程读取/修改进程内变量。对于 mgr 进程（baseappmgr/cellappmgr），它们管理的子进程（baseapp/cellapp）的 Watcher 需要能被“穿透”访问——这就是 Watcher 转发。

### 11.1 ForwardingWatcher

定义于 [watcher_forwarding.hpp](file:///workspace/src/lib/server/watcher_forwarding.hpp)。继承 `Watcher`，把对本进程某路径的 Watcher 请求**转发**给被管理的子进程。

```cpp
class ForwardingWatcher : public Watcher
{
public:
    enum ExposeHints
    {
        WITH_ENTITY = 0,   // 拥有特定实体的子进程
        ALL,               // 所有子进程
        WITH_SPACE,        // 特定 space 的子进程
        LEAST_LOADED       // 负载最低的子进程
    };

    virtual ForwardingCollector * newCollector(
        WatcherPathRequestV2 & pathRequest,
        const std::string & destWatcher,
        const std::string & targetInfo ) = 0;

    virtual bool setFromStream( void * base, const char * path,
        WatcherPathRequestV2 & pathRequest );
    virtual bool getAsString( const void * base, const char * path,
        std::string & result, std::string & desc, Mode & mode ) const;
protected:
    ComponentIDList getComponentIDList( const std::string & targetInfo );
};
```

`ExposeHints` 描述转发目标的选择策略：`ALL` 把请求发给所有子进程并汇总结果；`LEAST_LOADED` 只发给负载最低的；`WITH_ENTITY` 发给持有指定实体的；`WITH_SPACE` 发给持有指定 space 的。v1 协议（`getAsString`/`setFromString`）不支持，只支持 v2 流式协议。

### 11.2 ForwardingCollector

定义于 [watcher_forwarding_collector.hpp](file:///workspace/src/lib/server/watcher_forwarding_collector.hpp)。管理一次转发请求的全生命周期——向多个子进程发出请求，收集所有响应，汇总成一个 tuple 结果返回给发起方。

```cpp
class ForwardingCollector : public Mercury::ShutdownSafeReplyMessageHandler
{
public:
    bool init( const Mercury::InterfaceElement & ie,
        Mercury::NetworkInterface & interface, AddressList * addrList );
    bool start();
    void checkSatisfied();    // 检查是否所有响应都已收到
private:
    WatcherPathRequestV2 & pathRequest_;
    std::string queryPath_;
    bool callingComponents_;
    uint32 pendingRequests_;               // 待收响应数
    Mercury::InterfaceElement interfaceElement_;
    AddressList * pAddressList_;           // 目标地址列表
    std::string outputStr_;                // 收集的 stdout/stderr
    MemoryOStream resultStream_;           // 打包的 tuple 结果
    uint32 tupleCount_;

    class ResponseDecoder : public WatcherProtocolDecoder;  // 响应解码器
};
```

`ResponseDecoder`（继承 `WatcherProtocolDecoder`）有状态机（`EXPECTING_TUPLE` → `EXPECTING_OUTPUT` → `EXPECTING_ANY`），把每个子进程返回的 Watcher 值打包进 `resultStream_`，最终汇成一个 tuple。

### 11.3 WatcherProtocolDecoder

定义于 [watcher_protocol.hpp](file:///workspace/src/lib/server/watcher_protocol.hpp)。Watcher 协议 v2 的基础解码器，支持 `int`/`uint`/`float`/`bool`/`string`/`tuple` 等类型的流式解码。

### 11.4 WatcherForwardingTypes

定义于 [watcher_forwarding_types.hpp](file:///workspace/src/lib/server/watcher_forwarding_types.hpp)。定义转发用的类型别名：

```cpp
typedef int32 ComponentID;                       // cellapp/baseapp 的组件 ID
typedef std::vector<ComponentID> ComponentIDList;
typedef std::pair<Mercury::Address,ComponentID> AddressPair;
typedef std::vector<AddressPair> AddressList;
```

---

## 12. 服务器信息与统计

### 12.1 ServerInfo

定义于 [server_info.hpp](file:///workspace/src/lib/server/server_info.hpp)。采集本机硬件信息：主机名、CPU 描述与各核频率、内存描述/总量/用量。Linux 下通过 `/proc/cpuinfo` 与 `/proc/meminfo` 读取（`fetchLinuxCpuInfo`/`fetchLinuxMemInfo`）。`updateMem` 可刷新内存用量。这些信息在 `START_MSG` 中打印，并供负载均衡决策使用。

### 12.2 EventHistoryStats

定义于 [event_history_stats.hpp](file:///workspace/src/lib/server/event_history_stats.hpp)。统计 EventHistory（事件历史队列，用于断线重连补发）的使用情况，按“类型名.成员名”记录次数与字节数，通过 `MapWatcher` 暴露为 Watcher。`setEnabled(true)` 时会清空已有统计，便于做“操作前/操作后”对比测量。

---

## 13. 插件与启动辅助

### 13.1 PluginLibrary

定义于 [plugin_library.hpp](file:///workspace/src/lib/server/plugin_library.hpp)。从相对应用的目录加载所有插件动态库：

```cpp
namespace PluginLibrary
{
    void loadAllFromDirRelativeToApp( bool prefixWithAppName, const char * partialDir );
}
```

`prefixWithAppName` 决定目录名是否加进程名前缀（如 `cellapp_plugins/`），让不同进程加载不同插件集。

### 13.2 AddToManagerHelper

定义于 [add_to_manager_helper.hpp](file:///workspace/src/lib/server/add_to_manager_helper.hpp)。baseapp/cellapp 启动时向 mgr “注册自己”的辅助类，处理注册消息的发送、重发、超时与 mgr 的回复。

```cpp
class AddToManagerHelper :
    public Mercury::ShutdownSafeReplyMessageHandler,
    public TimerHandler
{
public:
    AddToManagerHelper( Mercury::EventDispatcher & dispatcher );
    void send();                         // 启动注册流程
protected:
    virtual void handleFatalTimeout() { }                // 致命超时回调
    virtual bool finishInit( BinaryIStream & data ) = 0; // 收到 mgr 回复（含初始化数据）
    virtual void doSend() = 0;                           // 子类发送具体注册消息
private:
    enum AddToManagerTimeouts { TIMEOUT_RESEND, TIMEOUT_FATAL };
    TimerHandle resendTimerHandle_, fatalTimerHandle_;
};
```

`TIMEOUT_RESEND` 定时重发注册消息（mgr 可能还没起来）；`TIMEOUT_FATAL` 是致命超时（注册彻底失败，进程退出）。`finishInit` 收到 mgr 回复的初始化数据（如 `CellAppInitData`）。

### 13.3 AnonymousChannelClient

定义于 [anonymous_channel_client.hpp](file:///workspace/src/lib/server/anonymous_channel_client.hpp)。用于与“可能不知道我们存在”的进程建立 channel——通过发一个 `Birth` 消息把自己的地址告诉对方，对方收到后建立反向 channel。

```cpp
class AnonymousChannelClient : public Mercury::InputMessageHandler
{
public:
    bool init( Mercury::NetworkInterface & interface,
        Mercury::InterfaceMinder & interfaceMinder,
        const Mercury::InterfaceElement & birthMessage,
        const char * componentName, int numRetries );
    Mercury::ChannelOwner * pChannelOwner() const { return pChannelOwner_; }
};
```

`BW_INIT_ANONYMOUS_CHANNEL_CLIENT` 与 `BW_ANONYMOUS_CHANNEL_CLIENT_MSG` 宏简化了 Birth 消息的声明与初始化。

### 13.4 CellAppInitData

定义于 [cell_app_init_data.hpp](file:///workspace/src/lib/server/cell_app_init_data.hpp)。cellapp 向 cellappmgr 注册成功后，mgr 回复的初始化数据：

```cpp
struct CellAppInitData
{
    int32 id;                       // 新 cellapp 的 ID
    GameTime time;                  // 当前 game time
    Mercury::Address baseAppAddr;   // 通信用的 baseapp 地址
    bool isReady;                   // 服务器是否就绪
};
```

### 13.5 其它工具

| 文件 | 用途 |
| --- | --- |
| [common.hpp](file:///workspace/src/lib/server/common.hpp) | 默认值常量（`DEFAULT_GAME_UPDATE_HERTZ=10` 等）、`ShutDownStage`/`UpdateAutoLoad` 枚举 |
| [stream_helper.hpp](file:///workspace/src/lib/server/stream_helper.hpp) | 实体初始化/销毁的流打包助手（`addEntity`/`removeEntity`/`addRealEntityWithWitnesses`/`CellEntityBackupFooter`） |
| [util.hpp](file:///workspace/src/lib/server/util.hpp) | 数学/带宽工具（`distSqrBetween`/`bpsToPacketSize`/`exePath`） |
| [writedb.hpp](file:///workspace/src/lib/server/writedb.hpp) | `WriteDBFlags` 枚举（`WRITE_LOG_OFF`/`WRITE_BASE_DATA`/`WRITE_CELL_DATA`/`WRITE_DELETE_FROM_DB`/`WRITE_AUTO_LOAD_*`） |
| [reviver_common.hpp](file:///workspace/src/lib/server/reviver_common.hpp) | Reviver 优先级常量（`REVIVER_PING_NO`/`REVIVER_PING_YES`） |
| [reviver_subject.hpp](file:///workspace/src/lib/server/reviver_subject.hpp) | Reviver 心跳应答器（单例，处理 reviver 进程的 ping） |

---

## 14. 服务器进程的完整生命周期

以 cellapp 为例，描述从 `main` 到退出的完整流程：

### 14.1 ASCII 时序图

```
main(argc, argv)
  │
  ├─ BIGWORLD_MAIN 宏展开:
  │    BWResource::init()          # 初始化资源系统（多文件系统）
  │    BWConfig::init()            # 解析 bw.xml 链
  │    bwParseCommandLine()        # 解析命令行（-machined 等）
  │
  ├─ bwMainT<CellApp>():
  │    EventDispatcher dispatcher
  │    NetworkInterface interface  # 绑定内网 UDP 端口
  │    SignalProcessor signalProcessor(dispatcher)   # 注册单例
  │    BW_MESSAGE_FORWARDER3(...)  # 日志转发器
  │    START_MSG("cellapp")        # 打印 banner
  │    │
  │    └─ doBWMainT<CellApp>():
  │         ServerAppConfig::init(CellAppConfig::postInit)  # 初始化所有 ServerAppOption
  │         CellApp serverApp(dispatcher, interface)        # 构造进程对象
  │         serverApp.runApp(argc, argv):
  │           │
  │           ├─ stampsPerSecond()              # 校准 CPU 时钟
  │           │
  │           ├─ CellApp::init(argc, argv):     # 子类重写
  │           │    ServerApp::init()             # 信号处理 SIGINT/SIGHUP
  │           │    注册 Mercury 接口 (cellAppInterfaces)
  │           │    ManagerAppGateway::init("cellappmgr")  # 连 cellappmgr
  │           │    AddToManagerHelper::send()    # 向 cellappmgr 注册自己
  │           │      ├─ doSend(): 发送 addCellApp 消息
  │           │      ├─ resend timer: 定时重发
  │           │      └─ 收到回复 -> finishInit(data):
  │           │           解析 CellAppInitData (id, time, baseAppAddr)
  │           │    TimeKeeper 启动 (向 cellappmgr 同步 game time)
  │           │    IDClient::init() (向 cellappmgr 取 ID 块)
  │           │    加载 entities.xml / .def (EntityDescriptionMap)
  │           │    Python 脚本初始化
  │           │    SharedData / Watcher 注册
  │           │
  │           ├─ INFO_MSG("---- cellapp is running ----")
  │           │
  │           ├─ CellApp::run():                 # 主循环
  │           │    mainDispatcher_.processUntilBreak():
  │           │      循环 {
  │           │        处理网络收包 (Mercury)
  │           │        执行定时器回调 (TimeQueue / tick)
  │           │        SignalProcessor::doTask() 派发信号
  │           │        ForwardingCollector 收集 Watcher 响应
  │           │        每 tick: time_++, 实体 tick, ghost 同步, AOI 更新
  │           │      } 直到 breakProcessing()
  │           │
  │           └─ interface_.prepareForShutdown()  # 网络收尾
  │
  ├─ CellApp::postDestruction()                 # 静态清理
  └─ INFO_MSG("cellapp has shut down.")
```

### 14.2 关键阶段说明

**启动阶段**（`init`）：进程首先连 mgr（`ManagerAppGateway::init`），但 mgr 可能还没起来，所以 `AddToManagerHelper` 带重发与致命超时。收到 mgr 回复后才能拿到自己的 ID、当前 game time、baseapp 地址等初始化数据（`CellAppInitData`）。然后才加载实体定义、初始化 Python、注册 Watcher。

**主循环**（`run`）：`EventDispatcher::processUntilBreak` 是单线程事件循环，依次处理：网络收包 → 定时器 → FrequentTask（含 `SignalProcessor`）→ 延迟通道等。每个 game tick（默认 10Hz）递增 `time_`，驱动实体逻辑、ghost 同步、AOI 更新。

**关闭阶段**（`shutDown`）：收到 SIGINT/SIGHUP → `onSignalled` → `shutDown` → `breakProcessing`。若走退休流程（`requestRetirement`），则先通知 mgr 迁走本进程实体，等迁移完成后才真正退出。`ShutDownStage` 枚举描述了受控关闭的多阶段协调。

---

## 15. baseapp/cellapp/mgr 与 ManagerAppGateway 的交互

### 15.1 注册流程

```
cellapp                          cellappmgr                     dbmgr
   │                                │                              │
   │ ManagerAppGateway::init        │                              │
   │ (向 machined 查 cellappmgr 地址)│                              │
   │──────────addCellApp──────────►│                              │
   │                                │ 分配 CellAppID               │
   │                                │ 分配 ID 块 ─────────────────►│
   │                                │◄────────────ID 块────────────│
   │                                │ 构造 CellAppInitData         │
   │◄────────回复(InitData)─────────│                              │
   │                                │                              │
   │ finishInit: 解析 id/time/addr  │                              │
   │ IDClient 收到初始 ID 块        │                              │
   │ TimeKeeper 向 cellappmgr 同步   │                              │
```

### 15.2 运行时交互

- **ID 补充**：`IDClient` 水位低于 `lowSize_` 时，通过 `getMoreMethod` 向 mgr 请求更多 ID；高于 `highSize_` 时通过 `putBackMethod` 还回。
- **时间同步**：`TimeKeeper` 周期性向主时钟（通常在 mgr）发 `synchroniseWithMaster` 请求，主时钟回复读数，从节点调整 tick 节奏。
- **退休**：`ManagerAppGateway::retireApp()` 发送 `retireAppIE_` 消息通知 mgr；mgr 把该进程的实体迁到其它进程后，mgr 回复允许退出。
- **负载均衡**：mgr 通过 `BalanceConfig`（`maxCPUOffload`/`minEntityOffload`）配置的策略，定期评估各 cellapp/baseapp 负载，决定迁移哪些实体。`ServerInfo` 提供的 CPU/内存数据是决策输入。
- **Watcher 转发**：运维工具访问 `cellappmgr/components/<id>/...` 路径时，`ForwardingWatcher` 把请求转发给对应 cellapp，`ForwardingCollector` 收集响应后汇总返回。
- **备份哈希**：baseappmgr 维护 `BackupHash`，baseapp 死亡时用 `BackupHashChain` 定位受影响实体，通过 `BaseBackupSwitchMailBoxVisitor` 重映射所有 Mailbox 引用。

### 15.3 baseapp 与 cellapp 的差异

二者都继承 `EntityApp`，但：

- **baseapp** 的 `ManagerAppGateway` 连 baseappmgr，持有 `BackupHash`（每个 baseapp 既是主也是别人的备份），处理客户端连接与持久化（`writeToDB`）
- **cellapp** 的 `ManagerAppGateway` 连 cellappmgr，管理 space/chunk 与 AOI，不直接持久化

它们的 `managerAppGateway()` 纯虚方法各自返回对应的网关实例。

---

## 16. 小结

`src/lib/server/` 是 BigWorld 服务端的“公共底座”，其设计有几个突出特点：

1. **模板方法 + 声明式**：`ServerApp::runApp` 是模板方法（init→run→shutdown），子类只重写钩子；`ServerAppOption` + 宏把“读配置+注册 Watcher+打印”三合一，新增配置选项只需一行 `BW_OPTION`。
2. **配置链式继承**：`BWConfig` 通过 `parentFile` 链支持配置分层覆盖（基础配置 → 项目配置 → 环境配置），`SearchIterator` 可跨链查找。
3. **信号安全派发**：`SignalProcessor` 用 `FrequentTask` 把信号处理延后到事件循环中执行，避免信号处理函数的竞态。
4. **分层 ID 分配**：`IDClient` 水位线机制 + `LockedID` 防复用，平衡了 ID 获取延迟与全局唯一性。
5. **一致性哈希容错**：`BackupHash` + `BackupHashChain` 实现 baseapp 故障时的最小迁移，`MailBoxVisitor` 自动重定向所有引用。
6. **Watcher 穿透**：`ForwardingWatcher` 让 mgr 成为子进程 Watcher 的代理入口，运维只需连 mgr 即可访问整个集群的运行时变量。

理解了这套底座，再去看具体进程（`src/server/cellapp/`、`src/server/baseapp/` 等）就只需关注各自的业务逻辑——它们都是在 `ServerApp` / `EntityApp` 的骨架上“填业务血肉”。
