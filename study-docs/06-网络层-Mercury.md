# BigWorld Engine 2.0.1 网络层（Mercury）

> 源码位置：`src/lib/network/`
> 模块定位：**Mercury（信使神）** 是 BigWorld 自研的 UDP 可靠传输 + RPC 框架，是整个引擎所有进程间通信（IPC）的统一基础设施。
> 角色：服务端进程之间（cellapp↔baseapp↔mgr）、客户端与服务端之间（client↔baseapp↔loginapp）、Watcher 运维通道、machined 守护通信，全部建立在 Mercury 之上。它把“不可靠、无连接”的 UDP socket，封装成“可靠、有序、可加密、可压缩、带 RPC 语义”的通信通道。

---

## 1. 模块定位与核心职责

BigWorld 是分布式多进程架构，节点间通信有如下需求：

- **低延迟**：游戏实时性要求 cellapp↔baseapp 之间毫秒级往返
- **可靠有序**：状态同步、RPC 调用不能丢、不能乱序
- **高吞吐**：客户端实体更新量大，需要聚合（bundle）与压缩
- **安全**：登录握手用 RSA，会话用 Blowfish
- **跨平台**：Windows / Linux / PS3 / Xbox360 统一抽象

Mercury 把这些需求封装在一套类型里。它不使用 TCP（避免队头阻塞与慢启动），而是在 UDP 之上自实现滑动窗口、ACK/重传、分片重组、捎带（piggyback）等机制。

### 1.1 与其它模块的关系

```
                ┌──────────────────────────────────────────┐
                │            应用层（ServerApp 等）          │
                │  通过 InterfaceTable 注册消息 handler      │
                └────────────────────┬─────────────────────┘
                                     │ 调用
                                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │                       Mercury                            │
   │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │
   │  │EventDispatcher│  │NetworkInterface│  │InterfaceTable│  │
   │  │ (事件循环)    │   │ (channel 集合)│   │ (IDL 分发)   │  │
   │  └──────┬───────┘   └───────┬────────┘   └──────┬───────┘│
   │         │                   │                   │        │
   │         ▼                   ▼                   ▼        │
   │  ┌────────────┐    ┌──────────────┐    ┌──────────────┐   │
   │  │ TimeQueue  │    │   Channel    │    │InterfaceElem │   │
   │  │FrequentTask│    │ (滑动窗口)    │    │InterfaceMindr│   │
   │  └────────────┘    └──────┬───────┘    └──────────────┘   │
   │                           │                              │
   │      ┌────────────────────┼────────────────────┐         │
   │      ▼                    ▼                    ▼         │
   │  ┌────────┐         ┌──────────┐         ┌──────────┐     │
   │  │ Bundle │         │  Packet  │         │Endpoint  │     │
   │  │(消息聚合)│        │ (网络包) │         │(UDP socket)│   │
   │  └────────┘         └──────────┘         └──────────┘     │
   └──────────────────────────────────────────────────────────┘
                                     │
                ┌────────────────────┼────────────────────┐
                ▼                    ▼                    ▼
          ┌──────────┐        ┌──────────┐         ┌──────────┐
          │  cstdmf  │        │   math   │         │ resmgr   │
          │(timer/str│        │(vector3) │         │(压缩配置)│
          └──────────┘        └──────────┘         └──────────┘
```

被依赖方：

- `cstdmf`：`timer_handler.hpp`、`time_queue`、`binary_stream.hpp`、`smartpointer.hpp`、`debug.hpp`、`watcher.hpp`、`timestamp.hpp`
- `math`：`vector3.hpp`、`ema.hpp`（指数加权平均，用于统计）
- `resmgr`：`datasection.hpp`（压缩类型配置）

使用方：

- `src/lib/server/`：`ServerApp` 持有 `NetworkInterface`，所有服务端进程
- `src/lib/connection/`：客户端 `ServerConnection` 基于 Mercury 登录与会话
- `src/server/*/`：各进程定义自己的 `*_interface.hpp`，用 Mercury 宏声明 IDL
- `src/lib/entitydef/`：实体属性/方法调用通过 Mercury 消息传递

---

## 2. 模块全景与文件清单

`src/lib/network/` 约有 90 个源文件，按职责分组：

| 组 | 代表文件 | 职责 |
| --- | --- | --- |
| 基础类型 | `basictypes.hpp/cpp/ipp`、`misc.hpp`、`msgtypes.hpp` | `Address`、`SeqNum`、`Reason` 等核心类型 |
| 事件循环 | `event_dispatcher.hpp/cpp/ipp`、`event_poller.hpp/cpp`、`frequent_tasks.hpp/cpp`、`dispatcher_coupling.hpp` | 主循环心脏，定时器/网络/文件事件分发 |
| 网络接口 | `network_interface.hpp/cpp/ipp`、`endpoint.hpp/cpp/ipp`、`netmask.hpp/cpp`、`portmap.hpp` | socket 包装与 channel 集合管理 |
| 通道 | `channel.hpp/cpp/ipp`、`circular_array.hpp`、`unacked_packet.hpp/cpp`、`reliable_order.hpp`、`channel_owner.hpp/cpp`、`channel_sender.hpp`、`channel_finder.hpp` | 可靠有序通道实现 |
| 通道子状态机 | `keepalive_channels.hpp/cpp`、`condemned_channels.hpp/cpp`、`delayed_channels.hpp/cpp`、`irregular_channels.hpp/cpp`、`monitored_channels.hpp/cpp` | channel 的各类监控/清理集合 |
| 消息聚合 | `bundle.hpp/cpp/ipp`、`bundle_piggyback.hpp`、`reply_order.hpp`、`unpacked_message_header.hpp/cpp` | 消息打包与捎带 |
| 网络包 | `packet.hpp/cpp`、`packet_receiver.hpp/cpp`、`packet_filter.hpp/cpp/ipp`、`packet_monitor.hpp`、`packet_receiver_stats.hpp/cpp/ipp`、`once_off_packet.hpp/cpp` | 实际网络包结构与收发 |
| 分片重组 | `fragmented_bundle.hpp/cpp` | 超长 bundle 的分片与重组 |
| IDL 系统 | `interface_table.hpp/cpp`、`interface_element.hpp/cpp`、`interface_minder.hpp/cpp/ipp`、`interface_macros.hpp`、`common_interface_macros.hpp`、`interfaces.hpp`、`common_message_handlers.hpp` | 接口描述与消息路由 |
| 请求-响应 | `request_manager.hpp/cpp`、`request.hpp/cpp`、`blocking_reply_handler.hpp/cpp` | RPC 请求配对 |
| 安全 | `encryption_filter.hpp/cpp`、`public_key_cipher.hpp/cpp` | Blowfish 对称加密 / RSA 非对称加密 |
| 压缩 | `compression_stream.hpp/cpp`、`compression_type.hpp`、`zip_stream.hpp/cpp`、`file_stream.hpp/cpp` | zlib 压缩封装 |
| 错误与统计 | `error_reporter.hpp/cpp`、`nub_exception.hpp/ipp`、`sending_stats.hpp/cpp`、`bsd_snprintf.cpp/h` | 异常聚合与流量统计 |
| 远程步骤 | `remote_stepper.hpp/cpp`、`rescheduled_sender.hpp/cpp` | 跨进程的延迟执行 |
| Watcher 网络 | `watcher_nub.hpp/cpp`、`watcher_connection.hpp/cpp`、`watcher_packet_handler.hpp/cpp`、`watcher_glue.hpp` | 运维监控通道 |
| machined 集成 | `machine_guard.hpp/cpp`、`machined_utils.hpp/cpp`、`process_socket_stats_helper.hpp/cpp` | 与 bwmachined 守护进程通信 |
| 日志转发 | `logger_message_forwarder.hpp/cpp` | 日志消息转发到 logger 进程 |
| 平台兼容 | `net_360.hpp/cpp`、`ps3_compatibility.hpp` | Xbox360 / PS3 平台适配 |
| 单元测试 | `unit_test/test_channel.cpp`、`test_fragment.cpp`、`test_overflow.cpp`、`test_compresslength.cpp`、`test_receive_window.cpp`、`test_threadsafety.cpp` 等 | Mercury 自测 |

---

## 3. 核心抽象与命名空间

所有网络类位于 `Mercury` 命名空间（信使神）。下面按数据流自底向上介绍核心抽象。

### 3.1 Mercury::Address —— 进程地址

定义于 [basictypes.hpp](file:///workspace/src/lib/network/basictypes.hpp)，是 Mercury 中标识一个通信端点的核心类型。

```cpp
namespace Mercury
{
    /**
     *  This class encapsulates an IP address and TCP port.
     */
    class Address
    {
    public:
        Address();
        Address( uint32 ipArg, uint16 portArg );

        uint32  ip;     ///< IP address.
        uint16  port;   ///< The port.
        uint16  salt;   ///< Different each time.

        int writeToString( char * str, int length ) const;
        char * c_str() const;
        const char * ipAsString() const;
        bool isNone() const         { return this->ip == 0; }

        static const Address NONE;
    };

    inline bool operator==(const Address & a, const Address & b);
    inline bool operator!=(const Address & a, const Address & b);
    inline bool operator<(const Address & a, const Address & b);
}
```

`Address` 由 IP + 端口 + salt 三部分组成。`salt` 字段在普通通道中用于区分同一地址上的多次连接（每次连接 salt 不同），在 `EntityMailBoxRef` 中则被复用为「组件类型 + 实体类型」的打包字段（高 3 位组件、低 13 位类型），见 [basictypes.hpp](file:///workspace/src/lib/network/basictypes.hpp)：

```cpp
struct EntityMailBoxRef
{
    EntityID            id;
    Mercury::Address    addr;

    enum Component
    {
        CELL = 0, BASE = 1, CLIENT = 2,
        BASE_VIA_CELL = 3, CLIENT_VIA_CELL = 4,
        CELL_VIA_BASE = 5, CLIENT_VIA_BASE = 6
    };

    Component component() const      { return (Component)(addr.salt >> 13); }
    EntityTypeID type() const        { return addr.salt & 0x1FFF; }
};
```

`basictypes.hpp` 还定义了大量全局类型别名：`GameTime`、`EntityID`、`SpaceID`、`SessionKey`、`EntityTypeID`、`DatabaseID`、`EventNumber`、`VolatileNumber` 等，它们在网络消息中作为字段类型使用。

`Capabilities` 类用 32 位位图表示一组能力开关，握手时协商双方支持哪些可选特性：

```cpp
class Capabilities
{
public:
    Capabilities();
    void add( uint cap );
    bool has( uint cap ) const;
    bool match( const Capabilities& on, const Capabilities& off ) const;
    static const uint s_maxCap_ = std::numeric_limits<CapsType>::digits - 1;
private:
    CapsType caps_;
};
```

### 3.2 核心数值类型 —— SeqNum / Reason

定义于 [misc.hpp](file:///workspace/src/lib/network/misc.hpp)，是 Mercury 内部最基础的常量与错误码。

```cpp
namespace Mercury
{
    /// 包的序列号类型，用于排序与标识包
    typedef uint32 SeqNum;
    const SeqNum SEQ_SIZE = 0x10000000U;   // 序列号空间大小（2^28）
    const SeqNum SEQ_MASK = SEQ_SIZE-1;
    const SeqNum SEQ_NULL = SEQ_SIZE;      // 表示“无序列号”

    typedef uint8 MessageID;               // 包上消息标识符类型
    typedef int32 ChannelID;               // 索引通道标识符
    const ChannelID CHANNEL_ID_NULL = 0;
    typedef SeqNum ChannelVersion;         // 通道版本号（复用 SeqNum 的环绕逻辑）
    typedef int32 ReplyID;                 // 请求的回复 ID
    const ReplyID REPLY_ID_NONE = -1;
    const ReplyID REPLY_ID_MAX = 1000000;

    enum Reason
    {
        REASON_SUCCESS = 0,
        REASON_TIMER_EXPIRED = -1,
        REASON_NO_SUCH_PORT = -2,
        REASON_GENERAL_NETWORK = -3,
        REASON_CORRUPTED_PACKET = -4,
        REASON_NONEXISTENT_ENTRY = -5,
        REASON_WINDOW_OVERFLOW = -6,
        REASON_INACTIVITY = -7,
        REASON_RESOURCE_UNAVAILABLE = -8,
        REASON_CLIENT_DISCONNECTED = -9,
        REASON_TRANSMIT_QUEUE_FULL = -10,
        REASON_CHANNEL_LOST = -11,
        REASON_SHUTTING_DOWN = -12,
    };
}
```

注意 `SEQ_SIZE` 取 `0x10000000`（2^28）而非 2^32，是为了让序列号比较使用「模运算环绕」算法。`seqMask` 与 `seqLessThan` 实现于 [channel.ipp](file:///workspace/src/lib/network/channel.ipp)：

```cpp
INLINE SeqNum Channel::seqMask( SeqNum x )
{
    return x & SEQ_MASK;
}

INLINE bool Channel::seqLessThan( SeqNum a, SeqNum b )
{
    return seqMask( a - b ) > SEQ_SIZE/2;
}
```

`seqLessThan` 利用了模 2^28 减法：若 `a-b`（模）落在序列号空间的「上半」，则说明 `a` 在 `b` 之前（环绕视角下）。这是典型的 TCP 序号比较算法。

### 3.3 EventDispatcher —— 主循环心脏

定义于 [event_dispatcher.hpp](file:///workspace/src/lib/network/event_dispatcher.hpp)，是每个进程事件循环的核心。它管理定时器、网络 I/O、文件描述符、频繁任务，并支持父子 dispatcher 级联。

```cpp
namespace Mercury
{

class EventDispatcher
{
public:
    EventDispatcher();
    ~EventDispatcher();

    void processContinuously();
    int  processOnce( bool shouldIdle = false );
    void processUntilBreak();
    void breakProcessing( bool breakState = true );
    bool processingBroken() const;

    void attach( EventDispatcher & childDispatcher );
    void detach( EventDispatcher & childDispatcher );

    bool registerFileDescriptor( int fd, InputNotificationHandler * handler );
    bool deregisterFileDescriptor( int fd );
    bool registerWriteFileDescriptor( int fd, InputNotificationHandler * handler );

    INLINE TimerHandle addTimer( int64 microseconds,
                    TimerHandler * handler, void* arg = NULL );
    INLINE TimerHandle addOnceOffTimer( int64 microseconds,
                    TimerHandler * handler, void * arg = NULL );

    void addFrequentTask( FrequentTask * pTask );
    bool cancelFrequentTask( FrequentTask * pTask );

    uint64 getSpareTime() const;
    double proportionalSpareTime() const;
    INLINE double maxWait() const;
    INLINE void maxWait( double seconds );

    ErrorReporter & errorReporter() { return *pErrorReporter_; }

private:
    void processFrequentTasks();
    void processTimers();
    void processStats();
    int processNetwork( bool shouldIdle );

    double calculateWait() const;

    bool breakProcessing_;
    EventPoller * pPoller_;
    TimeQueue64 * pTimeQueue_;
    FrequentTasks * pFrequentTasks_;
    ErrorReporter * pErrorReporter_;
    TimeStamp accSpareTime_, oldSpareTime_, totSpareTime_;
    uint32 numTimerCalls_;
    double maxWait_;
    DispatcherCoupling * pCouplingToParent_;
    typedef std::vector< EventDispatcher * > ChildDispatchers;
    ChildDispatchers childDispatchers_;
};

} // namespace Mercury
```

关键点：

- `processContinuously()`：阻塞式主循环，`processOnce()`：单次轮询，`processUntilBreak()`：循环直到 `breakProcessing()`。客户端登录时 `ServerConnection::logOn` 就用 `processUntilBreak` 同步等待登录完成（见 [server_connection.cpp](file:///workspace/src/lib/connection/server_connection.cpp)）。
- `addTimer` / `addOnceOffTimer`：底层委托给 `TimeQueue64`（来自 `cstdmf/time_queue.hpp`），定时器到期回调 `TimerHandler::handleTimeout`。
- `addFrequentTask`：注册每轮循环都执行的 `FrequentTask`，`DelayedChannels` 就是 `FrequentTask` 的子类。
- `registerFileDescriptor`：把 socket fd 注册到 `EventPoller`（Linux 用 epoll/select），有数据可读时回调 `InputNotificationHandler::handleInputNotification`。`PacketReceiver` 就是通过这个机制收到 UDP 包。
- `attach/detach`：父子 dispatcher 级联，子 dispatcher 的处理嵌入父 dispatcher 循环中（`DispatcherCoupling` 作为父的 frequent task）。`NetworkInterface::attach` 把自己的 dispatcher 挂到主 dispatcher。
- `ErrorReporter`：聚合错误日志，避免同一错误刷屏（见第 11 节）。

### 3.4 NetworkInterface —— channel 集合管理器

定义于 [network_interface.hpp](file:///workspace/src/lib/network/network_interface.hpp)，管理一组 channel 及其底层 socket。每个进程通常持有一个（外部）或多个（内部+外部）`NetworkInterface`。

```cpp
enum NetworkInterfaceType
{
    NETWORK_INTERFACE_INTERNAL,
    NETWORK_INTERFACE_EXTERNAL
};

class NetworkInterface : public TimerHandler
{
public:
    static const int RECV_BUFFER_SIZE;
    static const char * USE_BWMACHINED;

    NetworkInterface( EventDispatcher * pMainDispatcher,
        NetworkInterfaceType interfaceType,
        uint16 listeningPort = 0, const char * listeningInterface = NULL );
    ~NetworkInterface();

    void attach( EventDispatcher & mainDispatcher );
    void detach();
    void prepareForShutdown();
    void stopPingingAnonymous();
    void processUntilChannelsEmpty( float timeout = 10.f );

    bool recreateListeningSocket( uint16 listeningPort,
                            const char * listeningInterface );
    Reason registerWithMachined( const std::string & name, int id );

    bool registerChannel( Channel & channel );
    bool deregisterChannel( Channel & channel );
    void onAddressDead( const Address & addr );
    bool isDead( const Address & addr ) const;

    Channel * findChannel( const Address & addr,
                            bool createAnonymous  = false );
    INLINE Channel & findOrCreateChannel( const Address & addr );

    IndexedChannelFinderResult findIndexedChannel( ChannelID channelID,
            const Mercury::Address & srcAddr,
            Packet * pPacket, ChannelPtr & pChannel ) const;
    ChannelPtr findCondemnedChannel( ChannelID channelID ) const;
    void registerChannelFinder( ChannelFinder * pFinder );
    void delAnonymousChannel( const Address & addr );

    void setIrregularChannelsResendPeriod( float seconds );
    bool hasUnackedPackets() const;
    bool deleteFinishedChannels();
    void onChannelGone( Channel * pChannel );
    void onChannelTimeOut( Channel * pChannel );
    void cancelRequestsFor( Channel * pChannel );
    void delayedSend( Channel & channel );
    void sendIfDelayed( Channel & channel );

    Reason processPacketFromStream( const Address & addr, BinaryIStream & data );

    CondemnedChannels & condemnedChannels() { return *pCondemnedChannels_; }
    IrregularChannels & irregularChannels() { return *pIrregularChannels_; }
    KeepAliveChannels & keepAliveChannels() { return *pKeepAliveChannels_; }
    EventDispatcher & dispatcher()          { return *pDispatcher_; }
    EventDispatcher & mainDispatcher()      { return *pMainDispatcher_; }
    InterfaceTable & interfaceTable();
    const PacketReceiverStats & receivingStats() const;
    const SendingStats & sendingStats() const { return sendingStats_; }

    ChannelTimeOutHandler * pChannelTimeOutHandler() const;
    void pChannelTimeOutHandler( ChannelTimeOutHandler * pHandler );
    INLINE const Address & address() const;
    void setPacketMonitor( PacketMonitor * pPacketMonitor );
    bool isExternal() const          { return isExternal_; }
    Endpoint & socket()              { return socket_; }
    OnceOffSender & onceOffSender() { return *pOnceOffSender_; }
    void shouldUseChecksums( bool b );
    void setLatency( float latencyMin, float latencyMax );
    void setLossRatio( float lossRatio );
    void * pExtensionData() const;
    void pExtensionData( void * pData );

    void send( const Address & address, Bundle & bundle, Channel * pChannel = NULL );
    void sendPacket( const Address & addr, Packet * pPacket,
            Channel * pChannel, bool isResend );
    bool rescheduleSend( const Address & addr, Packet * pPacket );
    Reason basicSendWithRetries( const Address & addr, Packet * pPacket );
    Reason basicSendSingleTry( const Address & addr, Packet * pPacket );

private:
    Endpoint                    socket_;
    Address                     address_;
    PacketReceiver *            pPacketReceiver_;
    typedef std::map< Address, TimerHandle > RecentlyDeadChannels;
    RecentlyDeadChannels        recentlyDeadChannels_;
    typedef std::map< Address, Channel * >   ChannelMap;
    ChannelMap                  channelMap_;
    IrregularChannels *         pIrregularChannels_;
    CondemnedChannels *         pCondemnedChannels_;
    KeepAliveChannels *         pKeepAliveChannels_;
    DelayedChannels *           pDelayedChannels_;
    ChannelFinder *             pChannelFinder_;
    ChannelTimeOutHandler *     pChannelTimeOutHandler_;
    const bool                  isExternal_;
    bool                        isVerbose_;
    EventDispatcher *           pDispatcher_;
    EventDispatcher *           pMainDispatcher_;
    SeqNum                      nextSequenceID_;
    InterfaceTable *            pInterfaceTable_;
    RequestManager *            pRequestManager_;
    void *                      pExtensionData_;
    OnceOffSender *             pOnceOffSender_;
    PacketMonitor *             pPacketMonitor_;
    bool                        dropNextSend_;
    int                         artificialDropPerMillion_;
    int                         artificialLatencyMin_;
    int                         artificialLatencyMax_;
    bool                        shouldUseChecksums_;
    SendingStats                sendingStats_;
};
```

关键点：

- **两种类型**：`NETWORK_INTERFACE_INTERNAL`（服务端进程间，低延迟高带宽）与 `NETWORK_INTERFACE_EXTERNAL`（客户端到服务端，高延迟低带宽）。这决定 channel 的可靠性策略。
- **channelMap_**：按 `Address` 索引的通道表。`findChannel` 查找，`findOrCreateChannel` 查找或创建匿名通道。
- **子状态机集合**：`pIrregularChannels_`、`pCondemnedChannels_`、`pKeepAliveChannels_`、`pDelayedChannels_`，分别管理不同状态的 channel（见第 5 节）。
- **recentlyDeadChannels_**：外部接口记住最近注销的 channel 地址，短时间内丢弃来自这些地址的包，避免处理已断开客户端的残留包（尤其是带 filter 的包）。
- **pChannelFinder_**：自定义的索引 channel 解析器，当收到带 `FLAG_INDEXED_CHANNEL` 的包时调用，把 `ChannelID` 解析成 `Channel`。
- **pExtensionData_**：挂在接口上的用户数据。消息 handler 可通过 `header.pInterface->pExtensionData()` 取到。`ServerConnection::registerInterfaces` 把自己挂上去，这样 handler 能找到对应的 `ServerConnection`（用于多 bot 场景）。
- **pChannelTimeOutHandler_**：channel 超时回调，`ServerConnection` 实现它来响应通道死亡。
- **artificialDrop/Latency**：用于在测试环境模拟丢包与延迟。
- **send / sendPacket / basicSendWithRetries**：发送链路。`send` 接受 `Bundle`，内部转成 `Packet` 后调 `sendPacket`；`basicSendWithRetries` 处理 `ENOBUFS`（发送队列满）时重试。

### 3.5 Channel —— 可靠有序通道

定义于 [channel.hpp](file:///workspace/src/lib/network/channel.hpp)，是 Mercury 最核心的类，实现 UDP 之上的可靠有序通信。它继承 `TimerHandler`（用于不活跃检测）与 `ReferenceCount`（智能指针管理生命周期）。

```cpp
class Channel : public TimerHandler, public ReferenceCount
{
public:
    enum Traits
    {
        /// 服务端到服务端：低延迟、高带宽、低丢包
        INTERNAL = 0,
        /// 客户端到服务端：高延迟、低带宽、高丢包
        EXTERNAL = 1,
    };

    typedef void (*SendWindowCallback)( const Channel & );

    Channel( NetworkInterface & networkInterface, const Address & address,
        Traits traits,
        float minInactivityResendDelay = DEFAULT_INACTIVITY_RESEND_DELAY,
        PacketFilterPtr pFilter = NULL, ChannelID id = CHANNEL_ID_NULL );

    static Channel * get( NetworkInterface & networkInterface,
            const Address & address );

    void condemn();
    bool isCondemned() const { return isCondemned_; }
    void destroy();
    bool isDestroyed() const { return isDestroyed_; }
    bool isDead() const { return this->isCondemned() || this->isDestroyed(); }

    void initFromStream( BinaryIStream & data, const Address & addr );
    void addToStream( BinaryOStream & data );

    NetworkInterface & networkInterface() { return *pNetworkInterface_; }
    INLINE const Mercury::Address & addr() const;
    void addr( const Mercury::Address & addr );

    Bundle & bundle();
    const Bundle & bundle() const;
    bool hasUnsentData() const;

    void send( Bundle * pBundle = NULL );
    void delayedSend();
    void sendIfIdle();
    void reset( const Address & newAddr, bool warnOnDiscard = true );
    void configureFrom( const Channel & other );
    void switchInterface( NetworkInterface * pDestInterface );
    void startInactivityDetection( float inactivityPeriod, float checkPeriod = 1.f );

    int windowSize() const;
    int earliestUnackedPacketAge() const { return this->sendWindowUsage(); }

    PacketFilterPtr pFilter() const { return pFilter_; }
    void pFilter(PacketFilterPtr pFilter) { pFilter_ = pFilter; }

    bool isLocalRegular() const         { return isLocalRegular_; }
    void isLocalRegular( bool isLocalRegular );
    bool isRemoteRegular() const        { return isRemoteRegular_; }
    void isRemoteRegular( bool isRemoteRegular );
    bool hasRemoteFailed() const { return hasRemoteFailed_; }

    bool addResendTimer( SeqNum seq, Packet * p,
        const ReliableOrder * roBeg, const ReliableOrder * roEnd );
    bool handleCumulativeAck( SeqNum seq );
    bool handleAck( SeqNum seq );
    void checkResendTimers();
    void resend( SeqNum seq );
    uint64 roundTripTime() const { return roundTripTime_; }

    enum AddToReceiveWindowResult
    {
        SHOULD_PROCESS, SHOULD_NOT_PROCESS, PACKET_IS_CORRUPT
    };
    AddToReceiveWindowResult addToReceiveWindow(
        Packet * p, const Address & srcAddr, PacketReceiverStats & stats );

    typedef std::set< SeqNum > Acks;
    Acks acksToSend_;
    bool hasAcks() const { return !acksToSend_.empty(); }

    bool isAnonymous() const { return isAnonymous_; }
    bool isOwnedByInterface() const
        { return !isDestroyed_ && (isAnonymous_ || isCondemned_); }

    bool hasUnackedCriticals() const { return unackedCriticalSeq_ != SEQ_NULL; }
    void resendCriticals();
    bool wantsFirstPacket() const { return wantsFirstPacket_; }
    void gotFirstPacket() { wantsFirstPacket_ = false; }
    void dropNextSend() { shouldDropNextSend_ = true; }

    Traits traits() const { return traits_; }
    bool isExternal() const { return traits_ == EXTERNAL; }
    bool isIndexed() const  { return id_ != CHANNEL_ID_NULL; }
    bool isEstablished() const { return addr_.ip != 0; }

    SeqNum useNextSequenceID();
    void onPacketReceived( int bytes );

    ChannelID id() const         { return id_; }
    ChannelVersion version() const { return version_; }
    void version( ChannelVersion v ) { version_ = v; }

    void clearBundle();
    void bundlePrimer( BundlePrimer & primer );
    void writeFlags( Packet * p );
    void writeFooter( Packet * p );
    FragmentedBundlePtr pFragments() { return pFragments_; }

    static SeqNum seqMask( SeqNum x );
    static bool seqLessThan( SeqNum a, SeqNum b );

    bool hasUnackedPackets() const { return oldestUnackedSeq_ != SEQ_NULL; }
    int sendWindowUsage() const
    {
        return this->hasUnackedPackets() ?
            seqMask( largeOutSeqAt_ - oldestUnackedSeq_ ) : 0;
    }

    // ... 统计与 overflow 配置省略
};
```

`channel.hpp` 中关键的私有成员揭示了滑动窗口的实现：

```cpp
private:
    NetworkInterface *     pNetworkInterface_;
    Traits    traits_;
    ChannelID id_;
    TimerHandle inactivityTimerHandle_;
    uint64    inactivityExceptionPeriod_;
    ChannelVersion version_;
    ChannelVersion creationVersion_;
    uint64    lastReceivedTime_;
    PacketFilterPtr    pFilter_;
    Address            addr_;
    Bundle *           pBundle_;
    uint32    windowSize_;

    /// 一般是下一个要发送的包的序列号（不含 overflow 包）
    SeqNum    smallOutSeqAt_;
    /// 含 overflow 包的下一个序列号
    SeqNum    largeOutSeqAt_;
    /// 最老的未 ACK 包的序列号
    SeqNum    oldestUnackedSeq_;
    uint64    lastReliableSendTime_;
    uint64    lastReliableResendTime_;
    /// 该通道的平均往返时间（时间戳单位）
    uint64    roundTripTime_;
    uint64    minInactivityResendDelay_;
    SeqNum    unreliableInSeqAt_;

    class UnackedPacket;
    CircularArray< UnackedPacket * > unackedPackets_;
    bool hasSeenOverflowWarning_;

    /// 下一个期望接收的包
    SeqNum    inSeqAt_;
    /// 收到乱序包时缓存于此
    CircularArray< PacketPtr > bufferedReceives_;
    uint32 numBufferedReceives_;
    /// 入向分片重组链
    FragmentedBundlePtr pFragments_;
    /// 收到的最高 ACK 序列号
    uint32    highestAck_;

    friend class IrregularChannels;
    IrregularChannels::iterator irregularIter_;
    friend class KeepAliveChannels;
    KeepAliveChannels::iterator keepAliveIter_;

    bool isLocalRegular_;
    bool isRemoteRegular_;
    bool isCondemned_;
    bool isDestroyed_;
    BundlePrimer* pBundlePrimer_;
    bool hasRemoteFailed_;
    bool isAnonymous_;
    SeqNum unackedCriticalSeq_;
    unsigned int pushUnsentAcksThreshold_;
    bool shouldAutoSwitchToSrcAddr_;
    bool wantsFirstPacket_;
    bool shouldDropNextSend_;
};
```

**关键设计**：

1. **Traits**：`INTERNAL`（服务端间）与 `EXTERNAL`（客户端间）。注释说明：服务端间低延迟高带宽低丢包，重传整个包；客户端间高延迟低带宽高丢包，只重传可靠数据（不可靠数据从丢弃包中剥离丢弃）。这是 `EXTERNAL` 通道带宽优化的核心。
2. **滑动窗口**：`unackedPackets_` 是 `CircularArray<UnackedPacket*>`，存放已发但未 ACK 的包。`oldestUnackedSeq_` 是窗口左边界，`largeOutSeqAt_` 是右边界，`sendWindowUsage()` 返回当前窗口占用。窗口满时产生 `REASON_WINDOW_OVERFLOW`。
3. **序列号环绕**：`smallOutSeqAt_`（不含 overflow）与 `largeOutSeqAt_`（含 overflow）双指针，overflow 包是窗口满后仍要发送的包，暂存在 overflow 队列。
4. **ACK 机制**：`acksToSend_` 是待发 ACK 集合，`handleAck`/`handleCumulativeAck` 处理收到的 ACK，从 `unackedPackets_` 移除已确认包并滑动窗口。
5. **重传**：`addResendTimer` 为每个可靠包注册重传定时器，`resend` 执行重传，`checkResendTimers` 由 `IrregularChannels` 周期触发。`roundTripTime_` 是 RTT 估计，用于重传超时计算。
6. **入向有序**：`inSeqAt_` 是下一个期望序列号，`bufferedReceives_` 缓存乱序包，`addToReceiveWindow` 决定是否处理、缓存或丢弃。
7. **分片重组**：`pFragments_` 持有正在重组的分片链。
8. **BundlePrimer**：每次 `Bundle::clear()` 后调用 `primeBundle`，让拥有者（如 `ServerConnection`）往 bundle 注入公共头部消息（如 avatarUpdate）。`ServerConnection` 继承 `BundlePrimer`。
9. **Channel::get**：静态方法，对 `INTERNAL` 通道从 `NetworkInterface` 查找或创建（共享），`EXTERNAL` 通道则直接 new。
10. **condemn/destroy**：`condemn()` 标记待死（加入 `CondemnedChannels`，等所有未 ACK 包确认后真正销毁），`destroy()` 立即销毁。

### 3.6 Bundle —— 消息聚合单元

定义于 [bundle.hpp](file:///workspace/src/lib/network/bundle.hpp)，继承 `BinaryOStream`，是「一组待发消息」的容器。多个小消息聚合到一个 bundle，再拆成一个或多个 packet 发送，减少 UDP 包数量。

```cpp
enum ReliableTypeEnum
{
    RELIABLE_NO = 0,
    RELIABLE_DRIVER = 1,        // 驱动可靠：本消息会触发可靠发送
    RELIABLE_PASSENGER = 2,     // 搭车：仅当同 bundle 有 DRIVER 时才可靠发送
    RELIABLE_CRITICAL = 3       // 关键：DRIVER 且标记 bundle 为 critical
};

class ReliableType
{
public:
    ReliableType( ReliableTypeEnum e ) : e_( e ) { }
    bool isReliable() const { return e_ != RELIABLE_NO; }
    bool isDriver() const   { return e_ & RELIABLE_DRIVER; }
private:
    ReliableTypeEnum e_;
};

class Bundle : public BinaryOStream
{
public:
    Bundle( uint8 spareSize = 0, Channel * pChannel = NULL );
    Bundle( Packet * packetChain );
    virtual ~Bundle();

    void clear( bool newBundle = false );
    void cancelRequests();
    bool isEmpty() const;
    int size() const;
    int sizeInPackets() const;
    bool isMultiPacket() const { return firstPacket_->next() != NULL; }
    int numMessages() const    { return numMessages_; }

    void send( const Address & address, NetworkInterface & networkInterface,
        Channel * pChannel = NULL );

    void startMessage( const InterfaceElement & ie,
        ReliableType reliable = RELIABLE_DRIVER );
    void startRequest( const InterfaceElement & ie,
        ReplyMessageHandler * handler, void * arg = NULL,
        int timeout = DEFAULT_REQUEST_TIMEOUT,
        ReliableType reliable = RELIABLE_DRIVER );
    void startReply( ReplyID id, ReliableType reliable = RELIABLE_DRIVER );
    void * startStructMessage( const InterfaceElement & ie,
        ReliableType reliable = RELIABLE_DRIVER );

    void reliable( ReliableType currentMsgReliabile );
    bool isReliable() const;
    bool isCritical() const { return isCritical_; }
    bool isOnExternalChannel() const;
    Channel * pChannel() { return pChannel_; }

    virtual void * reserve( int nBytes );
    virtual void addBlob( const void * pBlob, int size );
    INLINE void * qreserve( int nBytes );

    void reliableOrders( Packet * p,
        const ReliableOrder *& roBeg, const ReliableOrder *& roEnd );
    void writeFlags( Packet * p ) const;

#ifdef USE_PIGGIES
    bool piggyback( SeqNum seq, const ReliableVector & reliableOrders, Packet * p );
#endif

    Reason dispatchMessages( InterfaceTable & interfaceTable,
            const Address & addr, Channel * pChannel,
            NetworkInterface & networkInterface,
            ProcessSocketStatsHelper * pStatsHelper );

    class iterator { /* 遍历 bundle 内消息 */ };
    iterator begin();
    iterator end();
    void finalise();
    void dumpMessages( InterfaceTable & interfaceTable );

    int addAck( SeqNum seq );
    SeqNum getAck() const;

private:
    PacketPtr firstPacket_;
    Packet * currentPacket_;
    bool finalised_;
    bool reliableDriver_;
    uint8 extraSize_;
    ReplyOrders replyOrders_;
    ReliableVector reliableOrders_;
    int reliableOrdersExtracted_;
    bool isCritical_;
#ifdef USE_PIGGIES
    BundlePiggybacks piggybacks_;
#endif
    Channel * pChannel_;
    SeqNum ack_;
    InterfaceElement const * curIE_;
    int msgLen_, msgExtra_;
    uint8 * msgBeg_;
    uint16 msgChunkOffset_;
    bool msgIsReliable_, msgIsRequest_;
    int numMessages_, numReliableMessages_;
};
```

关键点：

- **三种可靠级别**：`RELIABLE_DRIVER` 触发可靠发送；`RELIABLE_PASSENGER` 仅当同 bundle 有 driver 时才可靠（用于搭车的小消息）；`RELIABLE_CRITICAL` 标记 bundle 为关键，`Channel` 会优先重传 critical 包。
- **startMessage / startRequest / startReply**：开始一个新消息。`startRequest` 注册 `ReplyMessageHandler`，收到回复后回调。`startReply` 用于回复一个请求。
- **多 packet**：bundle 可跨多个 packet（`firstPacket_->next()` 链表），每个 packet 最大 `PACKET_MAX_SIZE`（1472 字节，见 [packet.hpp](file:///workspace/src/lib/network/packet.hpp)）。
- **reliableOrders_**：记录每个可靠消息在 packet 中的位置（`ReliableOrder`），用于丢失包时只重传可靠部分。
- **piggybacks_**：捎带机制，把已发但被丢的可靠消息「搭」在后续 packet 上重传（见第 8 节）。
- **dispatchMessages**：接收端把 bundle 内消息逐个分发给 `InterfaceTable` 的 handler。

`BundleSendingMap` 是辅助类，按地址聚合 bundle，批量发送给多个 app：

```cpp
class BundleSendingMap
{
public:
    BundleSendingMap( NetworkInterface & networkInterface ) :
        networkInterface_( networkInterface ) {}
    Bundle & operator[]( const Address & addr );
    void sendAll();
private:
    NetworkInterface & networkInterface_;
    typedef std::map< Address, Channel* > Channels;
    Channels channels_;
};
```

### 3.7 Packet —— 实际网络包

定义于 [packet.hpp](file:///workspace/src/lib/network/packet.hpp)，是真正上线的一个 UDP 数据报。继承 `ReferenceCount`，内部是定长字节数组。

```cpp
#define PACKET_MAX_SIZE 1472   // 以太网 MTU 1500 - IP 头 20 - UDP 头 8

class Packet : public ReferenceCount
{
public:
    typedef uint16 Flags;

    enum
    {
        FLAG_HAS_REQUESTS            = 0x0001,
        FLAG_HAS_PIGGYBACKS          = 0x0002,
        FLAG_HAS_ACKS                = 0x0004,
        FLAG_ON_CHANNEL              = 0x0008,
        FLAG_IS_RELIABLE             = 0x0010,
        FLAG_IS_FRAGMENT             = 0x0020,
        FLAG_HAS_SEQUENCE_NUMBER     = 0x0040,
        FLAG_INDEXED_CHANNEL         = 0x0080,
        FLAG_HAS_CHECKSUM            = 0x0100,
        FLAG_CREATE_CHANNEL          = 0x0200,
        FLAG_HAS_CUMULATIVE_ACK      = 0x0400,
        KNOWN_FLAGS                  = 0x07FF
    };

    typedef uint8 AckCount;
    typedef uint16 Offset;
    typedef uint32 Checksum;

private:
    static const int MAX_SIZE;   // 定义在 cpp，= PACKET_MAX_SIZE
public:
    static const int HEADER_SIZE = sizeof( Flags );

    static const int MAX_ACKS = (1 << (8 * sizeof( AckCount ))) - 1;  // 255

    /// 为固定长度尾部预留的空间（27 字节，约 1.5% 包容量）
    static const int RESERVED_FOOTER_SIZE =
        sizeof( Offset ) +                       // FLAG_HAS_REQUESTS
        sizeof( AckCount ) +                     // FLAG_HAS_ACKS
        sizeof( SeqNum ) +                       // FLAG_HAS_SEQUENCE_NUMBER
        sizeof( SeqNum ) * 2 +                   // FLAG_IS_FRAGMENT
        sizeof( ChannelID ) + sizeof( ChannelVersion ) + // FLAG_INDEXED_CHANNEL
        sizeof( Checksum );                      // FLAG_HAS_CHECKSUM

    Packet();
    Packet * next();
    void chain( Packet * pPacket ) { next_ = pPacket; }
    int chainLength() const;

    Flags flags() const { return BW_NTOHS( *(Flags*)data_ ); }
    bool hasFlags( Flags flags ) const { return (this->flags() & flags) == flags; }
    void setFlags( Flags flags ) { *(Flags*)data_ = BW_HTONS( flags ); }
    void enableFlags( Flags flags ) { *(Flags*)data_ |= BW_HTONS( flags ); }

    char * data() { return data_; }
    const char * body() const { return data_ + HEADER_SIZE; }
    char * back() { return data_ + msgEndOffset_; }
    int msgEndOffset() const  { return msgEndOffset_; }
    int bodySize() const      { return msgEndOffset_ - HEADER_SIZE; }
    int footerSize() const    { return footerSize_; }
    int totalSize() const     { return msgEndOffset_ + footerSize_; }

    int freeSpace() const
    {
        return MAX_SIZE - RESERVED_FOOTER_SIZE - msgEndOffset_ - footerSize_ - extraFilterSize_;
    }

    void reserveFooter( int nBytes ) { footerSize_ += nBytes; }
    void reserveFilterSpace( int nBytes ) { extraFilterSize_ = nBytes; }

    template <class TYPE> bool stripFooter( TYPE & value );
    template <class TYPE> void packFooter( TYPE value );

    int recvFromEndpoint( Endpoint & ep, Address & addr );

    enum { UNACKED_SEND, BUFFERED_RECEIVE, CHAINED_FRAGMENT };
    static void addToStream( BinaryOStream & data, const Packet * pPacket, int state );
    static PacketPtr createFromStream( BinaryIStream & data, int state );
    static int maxCapacity()
    {
        return MAX_SIZE - HEADER_SIZE - RESERVED_FOOTER_SIZE;
    }

private:
    PacketPtr next_;            // 链表下一个包
    int msgEndOffset_;          // 消息数据结尾偏移
    int footerSize_;            // 已预留尾部大小
    int extraFilterSize_;       // PacketFilter 预留空间
    Offset firstRequestOffset_;
    Offset* pLastRequestOffset_;
    AckCount nAcks_;
    Field piggyFooters_;
    SeqNum seq_;
    ChannelID channelID_;
    ChannelVersion channelVersion_;
    SeqNum fragBegin_, fragEnd_;
    Checksum checksum_;
    bool isPiggyback_;
    char data_[PACKET_MAX_SIZE];
};
```

关键点：

- **包头只有 2 字节 Flags**（`HEADER_SIZE = sizeof(Flags) = 2`），其余都是 footer（从包尾向前写）。
- **尾部预分配**：`RESERVED_FOOTER_SIZE`（27 字节）预先留好，让 bundle 逻辑不用担心尾部放不下。
- **Flags 语义**：
  - `FLAG_ON_CHANNEL`：包属于某个 channel（有 seq/ack）
  - `FLAG_IS_RELIABLE`：可靠包，需 ACK
  - `FLAG_HAS_SEQUENCE_NUMBER`：带序列号
  - `FLAG_HAS_ACKS` / `FLAG_HAS_CUMULATIVE_ACK`：带 ACK / 累积 ACK
  - `FLAG_IS_FRAGMENT`：分片包（带 fragBegin/fragEnd）
  - `FLAG_INDEXED_CHANNEL`：索引通道包（带 channelID/version）
  - `FLAG_HAS_CHECKSUM`：带校验和
  - `FLAG_CREATE_CHANNEL`：要求接收方创建匿名 channel
  - `FLAG_HAS_REQUESTS`：包含请求（带 firstRequestOffset 链）
  - `FLAG_HAS_PIGGYBACKS`：带捎带包
- **stripFooter / packFooter**：模板方法，按类型大小做字节序转换（`BW_NTOHS`/`BW_HTONS` 等），从包尾读写。
- **三种持久化状态**：`UNACKED_SEND`（发送方未 ACK 队列）、`BUFFERED_RECEIVE`（接收方乱序缓存）、`CHAINED_FRAGMENT`（分片重组链），用于 channel 状态迁移时序列化。

### 3.8 Endpoint —— UDP socket 包装

定义于 [endpoint.hpp](file:///workspace/src/lib/network/endpoint.hpp)，封装跨平台 socket 操作。

```cpp
class Endpoint
{
public:
    Endpoint();
    ~Endpoint();
    static const int NO_SOCKET = -1;

    operator int() const;
    void setFileDescriptor(int fd);
    bool good() const;

    void socket( int type );
    int setnonblocking( bool nonblocking );
    int setbroadcast( bool broadcast );
    int setreuseaddr( bool reuseaddr );
    int setkeepalive( bool keepalive );
    int bind( u_int16_t networkPort = 0, u_int32_t networkAddr = INADDR_ANY );
    int joinMulticastGroup( u_int32_t networkAddr );
    INLINE int close();
    INLINE int detach();

    int getlocaladdress( u_int16_t * networkPort, u_int32_t * networkAddr ) const;
    Mercury::Address getLocalAddress() const;
    Mercury::Address getRemoteAddress() const;

    // 无连接（UDP）
    int sendto( void * gramData, int gramSize,
        u_int16_t networkPort, u_int32_t networkAddr = BROADCAST);
    INLINE int recvfrom( void * gramData, int gramSize,
        u_int16_t * networkPort, u_int32_t * networkAddr );

    // 连接式（TCP，用于 watcher）
    int listen( int backlog = 5 );
    int connect( u_int16_t networkPort, u_int32_t networkAddr = BROADCAST );
    Endpoint * accept( u_int16_t * networkPort = NULL, u_int32_t * networkAddr = NULL );
    INLINE int send( const void * gramData, int gramSize );
    int recv( void * gramData, int gramSize );

    // 网络接口查询
    int getInterfaceFlags( char * name, int & flags );
    int getInterfaceAddress( const char * name, u_int32_t & address );
    int getInterfaceNetmask( const char * name, u_int32_t & netmask );
    bool getInterfaces( std::map< u_int32_t, std::string > &interfaces );
    int findDefaultInterface( char * name );
    static int convertAddress( const char * string, u_int32_t & address );

    int transmitQueueSize() const;
    int receiveQueueSize() const;
    int getBufferSize( int optname ) const;
    bool setBufferSize( int optname, int size );

private:
#if defined( unix ) || defined( PLAYSTATION3 )
    int socket_;
#else
    SOCKET socket_;
#endif
};
```

`Endpoint` 既能做 UDP（`sendto`/`recvfrom`），也能做 TCP（`listen`/`connect`/`accept`）。Mercury 的 `NetworkInterface` 用 UDP，`WatcherNub` 同时用 UDP 与 TCP。`netmask.hpp` 提供子网掩码计算，`portmap.hpp` 定义标准端口号（如 `PORT_LOGIN`）。

### 3.9 InterfaceTable / InterfaceElement / InterfaceMinder —— IDL 系统

Mercury 自带一套 IDL 描述系统，用宏声明接口，运行时通过 `InterfaceTable` 路由消息。

[interface_element.hpp](file:///workspace/src/lib/network/interface_element.hpp) 定义单个消息的元信息：

```cpp
const char FIXED_LENGTH_MESSAGE = 0;
const char VARIABLE_LENGTH_MESSAGE = 1;
const char INVALID_MESSAGE = 2;

class InterfaceElement
{
public:
    InterfaceElement( const char * name = "", MessageID id = 0,
            int8 lengthStyle = INVALID_MESSAGE, int lengthParam = 0,
            InputMessageHandler * pHandler = NULL );

    void set( const char * name, MessageID id, int8 lengthStyle, int lengthParam );

    int headerSize() const;
    int nominalBodySize() const;
    int compressLength( void * header, int length, Bundle & bundle, bool isRequest ) const;
    int expandLength( void * header, Packet * pPacket, bool isRequest ) const;
    bool canHandleLength( int length ) const;

    MessageID id() const           { return id_; }
    int8 lengthStyle() const       { return lengthStyle_; }
    int lengthParam() const        { return lengthParam_; }
    const char * name() const      { return name_; }
    InputMessageHandler * pHandler() const { return pHandler_; }
    void pHandler( InputMessageHandler * pHandler ) { pHandler_ = pHandler; }

    static const InterfaceElement REPLY;

private:
    MessageID           id_;
    int8                lengthStyle_;   // FIXED_LENGTH_MESSAGE 或 VARIABLE_LENGTH_MESSAGE
    int                 lengthParam_;   // 固定长度时是字节数，变长时是长度字段字节数
    const char *        name_;
    InputMessageHandler * pHandler_;
};
```

`InterfaceElementWithStats` 在此基础上加统计（接收字节数、消息数、EMA 速率、profile），用于 Watcher 监控。

[interface_table.hpp](file:///workspace/src/lib/network/interface_table.hpp) 定义消息路由表：

```cpp
class InterfaceTable : public TimerHandler
{
public:
    InterfaceTable( EventDispatcher & dispatcher );
    ~InterfaceTable();

    Reason registerWithMachined( const Address & addr );
    Reason registerWithMachined( const Address & addr, const std::string & name, int id = 0 );
    Reason deregisterWithMachined( const Address & addr );

    void serve( const InterfaceElement & ie, MessageID id, InputMessageHandler * pHandler );

    void onBundleStarted( Channel * pChannel );
    void onBundleFinished( Channel * pChannel );
    void pBundleEventHandler( BundleEventHandler * pHandler );

    INLINE const char * msgName( MessageID msgID ) const { return table_[ msgID ].name(); }

    InterfaceElementWithStats & operator[]( int id )              { return table_[ id ]; }
    const InterfaceElementWithStats & operator[]( int id ) const  { return table_[ id ]; }

private:
    void handleTimeout( TimerHandle handle, void * arg );
    std::string name_;
    int id_;
    typedef std::vector< InterfaceElementWithStats > Table;
    Table table_;
    BundleEventHandler * pBundleEventHandler_;
    TimerHandle statsTimerHandle_;
};
```

`InterfaceTable` 本质是一个 256 项的数组 `table_`（`MessageID` 是 `uint8`），按下标索引。`serve` 把一个 `InterfaceElement` 绑定到某 `MessageID` 并设 handler。`Bundle::dispatchMessages` 收到包后，按 `MessageID` 查表调用 handler。

[interface_minder.hpp](file:///workspace/src/lib/network/interface_minder.hpp) 用于声明式构造接口：

```cpp
class InterfaceMinder
{
public:
    InterfaceMinder( const char * name );
    InterfaceElement & add( const char * name, int8 lengthStyle,
            int lengthParam, InputMessageHandler * pHandler = NULL );
    InputMessageHandler * handler( int index );
    void handler( int index, InputMessageHandler * pHandler );
    const InterfaceElement & interfaceElement( uint8 id ) const;
    void registerWithInterface( NetworkInterface & networkInterface );
    Reason registerWithMachined( const Address & addr, int id ) const;
private:
    InterfaceElements elements_;
    const char * name_;
};
```

宏定义见 [interface_macros.hpp](file:///workspace/src/lib/network/interface_macros.hpp)，核心宏包括 `MERCURY_FIXED_MESSAGE`、`MERCURY_VARIABLE_MESSAGE`、`BEGIN_MERCURY_INTERFACE` / `END_MERCURY_INTERFACE`、`BW_BEGIN_STRUCT_MSG` / `END_STRUCT_MESSAGE`、`BW_STREAM_MSG` 等。例如 `MERCURY_STRUCT_GOODIES` 宏为每个结构消息生成 `start()` / `startRequest()` 静态方法：

```cpp
#define MERCURY_STRUCT_GOODIES( NAME )                                    \
    static NAME##Args & start( Mercury::Bundle & b,                        \
        Mercury::ReliableType reliable = Mercury::RELIABLE_DRIVER )        \
    {                                                                      \
        return *(NAME##Args*)b.startStructMessage( NAME, reliable );       \
    }                                                                      \
    static NAME##Args & startRequest( Mercury::Bundle & b,                 \
        Mercury::ReplyMessageHandler * handler,                             \
        void * arg = NULL,                                                 \
        int timeout = Mercury::DEFAULT_REQUEST_TIMEOUT,                    \
        Mercury::ReliableType reliable = Mercury::RELIABLE_DRIVER )         \
    {                                                                      \
        return *(NAME##Args*)b.startStructRequest( NAME,                   \
            handler, arg, timeout, reliable );                             \
    }
```

IDL 实例见 [cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp)，定义了 CellApp 接口（部分摘录）：

```cpp
#pragma pack( push, 1 )
BEGIN_MERCURY_INTERFACE( CellAppInterface )

    BW_STREAM_MSG_EX( CellApp, addCell )

    BW_BEGIN_STRUCT_MSG( CellApp, startup )
        Mercury::Address baseAppAddr;
    END_STRUCT_MESSAGE()

    BW_BEGIN_STRUCT_MSG( CellApp, setGameTime )
        GameTime        gameTime;
    END_STRUCT_MESSAGE()

    MF_BEGIN_ENTITY_MSG( avatarUpdateImplicit, REAL_ONLY )
        Coord            pos;
        YawPitchRoll     dir;
        uint8            refNum;
    END_STRUCT_MESSAGE()

    MF_VARLEN_ENTITY_REQUEST( enableWitness, REAL_ONLY )

    MF_BEGIN_ENTITY_MSG( ghostSetReal, GHOST_ONLY )
        NumTimesRealOffloadedType numTimesRealOffloaded;
        Mercury::Address    owner;
    END_STRUCT_MESSAGE()

    MERCURY_VARIABLE_MESSAGE( runExposedMethod, 2, NULL )

END_MERCURY_INTERFACE()
#pragma pack( pop )
```

注意 `#pragma pack(push, 1)` 保证结构体紧打包，与网络包格式一致。`MF_BEGIN_ENTITY_MSG` 是 CellApp 特有的宏，展开成带 `EntityID` 前缀的消息（消息先路由到 entity 再调方法），定义于同文件：

```cpp
#define MF_BEGIN_ENTITY_MSG( NAME, IS_REAL_ONLY )
    BEGIN_HANDLED_PREFIXED_MESSAGE( NAME, EntityID,
        CellEntityMessageHandler< CellAppInterface::NAME##Args >,
        std::make_pair( &Entity::NAME, IS_REAL_ONLY ) )
```

### 3.10 RequestManager —— 请求-响应配对

定义于 [request_manager.hpp](file:///workspace/src/lib/network/request_manager.hpp)，实现 RPC 的请求-响应配对。它是 `InputMessageHandler` 的子类，专门处理 `REPLY_MESSAGE_IDENTIFIER`（0xFF）回复消息。

```cpp
const unsigned char REPLY_MESSAGE_IDENTIFIER = 0xFF;  // 定义于 interface_element.hpp

class RequestManager : public InputMessageHandler
{
public:
    RequestManager( bool isExternal, EventDispatcher & dispatcher );
    ~RequestManager();

    void cancelRequestsFor( Channel * pChannel );
    void cancelRequestsFor( ReplyMessageHandler * pHandler,
            Reason reason = REASON_CHANNEL_LOST );
    void addReplyOrders( const ReplyOrders & replyOrders, Channel * pChannel );
    void failRequest( Request & request, Reason reason );

private:
    ReplyID getNextReplyID()
    {
        if (nextReplyID_ > REPLY_ID_MAX) nextReplyID_ = 1;
        return nextReplyID_++;
    }

    virtual void handleMessage( const Address & source,
        UnpackedMessageHeader & header, BinaryIStream & data );

    typedef std::map< int, Request * > RequestMap;
    RequestMap requestMap_;
    ReplyID nextReplyID_;
    EventDispatcher & dispatcher_;
    const bool isExternal_;
};
```

`Bundle::startRequest` 调用 `RequestManager::addReplyOrders` 注册一个 `Request`（含 handler、超时），分配 `ReplyID`，发送时把 `ReplyID` 写进包的 request footer。接收方回复时把同一 `ReplyID` 带回，`RequestManager::handleMessage` 据此找到原 `Request` 并回调 handler。`REPLY_ID_MAX = 1000000`，超过则环绕回 1。`cancelRequestsFor` 在 channel 死亡时取消所有未完成请求。

---

## 4. 可靠传输的 ACK / 重传 / 滑动窗口机制

这是 Mercury 的核心算法，结合 [circular_array.hpp](file:///workspace/src/lib/network/circular_array.hpp)、[unacked_packet.hpp](file:///workspace/src/lib/network/unacked_packet.hpp)、[reliable_order.hpp](file:///workspace/src/lib/network/reliable_order.hpp) 与 [channel.hpp](file:///workspace/src/lib/network/channel.hpp) 实现。

### 4.1 CircularArray —— 滑动窗口底层数据结构

[circular_array.hpp](file:///workspace/src/lib/network/circular_array.hpp) 是模板类，要求 size 是 2 的幂，用位掩码代替取模：

```cpp
template <class T> class CircularArray
{
public:
    typedef CircularArray<T> OurType;

    CircularArray( uint size ) : data_( new T[size] ), mask_( size-1 )
    {
        memset( data_, 0, sizeof(T) * this->size() );
    }
    ~CircularArray()    { delete [] data_; }

    uint size() const   { return mask_+1; }

    const T & operator[]( uint n ) const    { return data_[n&mask_]; }
    T & operator[]( uint n )                { return data_[n&mask_]; }

    void swap( OurType & other );
    void inflateToAtLeast( size_t newSize );
    void doubleSize( uint32 startIndex );

private:
    T * data_;
    uint mask_;
};
```

`operator[]` 用 `n & mask_` 直接定位，比 `%` 快。`inflateToAtLeast` / `doubleSize` 在窗口需要扩容时翻倍。Channel 用 `CircularArray<UnackedPacket*>` 存发送窗口，用 `CircularArray<PacketPtr>` 存接收乱序缓存。

### 4.2 UnackedPacket —— 未确认包记录

[unacked_packet.hpp](file:///workspace/src/lib/network/unacked_packet.hpp) 是 `Channel` 的内部嵌套类，记录每个已发未 ACK 包的元信息：

```cpp
class Channel::UnackedPacket
{
public:
    UnackedPacket( Packet * pPacket = NULL );
    SeqNum seq() const  { return pPacket_->seq(); }

    PacketPtr pPacket_;

    /// 该包上次被发送时，channel 的 outSeq 序号
    SeqNum  lastSentAtOutSeq_;
    /// 该包首次发送的时间
    uint64  lastSentTime_;
    /// 是否曾被重传
    bool    wasResent_;
    /// 该包内可靠消息的位置记录，用于 piggyback
    ReliableVector reliableOrders_;

    static UnackedPacket * initFromStream( BinaryIStream & data, uint64 timeNow );
    static void addToStream( UnackedPacket * pInstance, BinaryOStream & data );
};
```

`reliableOrders_` 是 `vector<ReliableOrder>`，记录包内每个可靠消息的起始位置与长度（见 [reliable_order.hpp](file:///workspace/src/lib/network/reliable_order.hpp)）：

```cpp
class ReliableOrder
{
public:
    uint8   * segBegin;          // 可靠段指针
    uint16  segLength;           // 段长度
    uint16  segPartOfRequest;    // 是否属于一个请求
};
typedef std::vector<ReliableOrder> ReliableVector;
```

注释说明：当一个含可靠消息的包在 client↔server 链路上丢失，**只重传可靠部分**（不可靠部分丢弃）。`ReliableOrder` 就是用来从已发 bundle 中「抽取」可靠消息的定位信息。

### 4.3 发送与窗口管理

发送流程（`Channel::send` → `Bundle::send` → `NetworkInterface::send`）：

1. 应用调 `Channel::bundle()` 取得 bundle，往里 stream 消息（`startMessage`/`startRequest`）。
2. `Channel::send()` 把 bundle 拆成 packet 链，每个 packet 分配序列号（`useNextSequenceID`，递增 `smallOutSeqAt_`/`largeOutSeqAt_`）。
3. 对每个可靠 packet，`addResendTimer` 创建 `UnackedPacket` 存入 `unackedPackets_`（按下标 = seq），记录 `ReliableOrder` 列表。
4. `NetworkInterface::sendPacket` 经 `PacketFilter`（如 `EncryptionFilter`）处理后调 `Endpoint::sendto` 发出。
5. 若窗口满（`sendWindowUsage() >= maxWindowSize()`），新包进 overflow 队列；overflow 超 `getMaxOverflowPackets()` 则抛 `REASON_WINDOW_OVERFLOW`。

### 4.4 ACK 处理

接收方收到可靠包后，把 seq 加入 `acksToSend_`，下次发包时捎带（`FLAG_HAS_ACKS`）或发累积 ACK（`FLAG_HAS_CUMULATIVE_ACK`）。`pushUnsentAcksThreshold_` 控制 ACK 积累到多少就强制发包。

发送方收到 ACK：

- `handleAck(seq)`：从 `unackedPackets_` 移除该 seq 的包，若 seq 是 `oldestUnackedSeq_` 则滑动窗口（向前推进 `oldestUnackedSeq_` 直到下一个未 ACK 包）。
- `handleCumulativeAck(seq)`：表示「seq 及之前的都已收到」，批量滑动窗口。

`highestAck_` 记录收到的最高 ACK，用于 RTT 估计。

### 4.5 重传

`checkResendTimers` 由 `IrregularChannels` 周期触发（仅对 `isLocalRegular_ == false` 的 channel）。对每个 `UnackedPacket`：

- 若 `now - lastSentTime_ > roundTripTime_ * factor`（或 `minInactivityResendDelay_`），调 `resend(seq)`。
- `resend` 取出 `UnackedPacket`：
  - `INTERNAL` channel：整个 packet 重发。
  - `EXTERNAL` channel：只重发 `reliableOrders_` 标记的可靠段（搭车或单独包），不可靠段丢弃。
- 重传次数计入 `numPacketsResent_`，`wasResent_ = true`。

`resendCriticals` 专门重传 `unackedCriticalSeq_` 之前的关键包（`RELIABLE_CRITICAL` 标记的），优先级最高。

`roundTripTime_` 是平滑 RTT 估计，更新时机在收到 ACK 时（结合 `lastSentTime_` 计算样本 RTT）。

### 4.6 入向有序接收

`addToReceiveWindow` 决定一个收到的包如何处理：

- `SHOULD_PROCESS`：seq == `inSeqAt_`（期望序号），立即处理，并检查 `bufferedReceives_` 中是否有连续的后续包可一并处理。
- `SHOULD_NOT_PROCESS`：seq > `inSeqAt_`，存入 `bufferedReceives_`（乱序缓存），等前面包到了再处理。
- `PACKET_IS_CORRUPT`：包损坏（校验和失败等），丢弃并报错。

`inSeqAt_` 每处理一个包就 +1，`bufferedReceives_` 也是 `CircularArray`，按下标 = seq 存。

### 4.7 状态机图

```
              ┌─────────────────────────────────────────────────────┐
              │                  发送方 Channel                       │
              │                                                     │
   useNextSequenceID  ──►  smallOutSeqAt_++                          │
              │            largeOutSeqAt_++                          │
              ▼                                                     │
   addResendTimer(seq, packet, roBeg, roEnd)                         │
              │                                                     │
              ▼                                                     │
   unackedPackets_[seq] = new UnackedPacket(packet)                  │
              │                                                     │
              ▼                                                     │
   NetworkInterface::sendPacket  ──►  Endpoint::sendto (UDP)         │
              │                                                     │
              │              ┌──────────────────────┐               │
              │              │  丢失 / 超时          │               │
              │              └──────────┬───────────┘               │
              │                         ▼                           │
              │              checkResendTimers (IrregularChannels)   │
              │                         │                           │
              │              ┌──────────▼───────────┐               │
              │              │  resend(seq)          │               │
              │              │  INTERNAL: 整包重发   │               │
              │              │  EXTERNAL: 只发可靠段 │               │
              │              └──────────┬───────────┘               │
              │                         │                           │
              ▼                         ▼                           │
   收到 ACK(seq) ◄─────────────────────────────  接收方 acksToSend_  │
              │                                                     │
              ▼                                                     │
   handleAck(seq) / handleCumulativeAck(seq)                         │
              │                                                     │
              ▼                                                     │
   unackedPackets_[seq] = NULL                                       │
   oldestUnackedSeq_ 向前推进（滑动窗口）                             │
              │                                                     │
              ▼                                                     │
   窗口空 + isCondemned_  ──►  CondemnedChannels 真正销毁             │
              └─────────────────────────────────────────────────────┘
```

---

## 5. 通道子状态机

`NetworkInterface` 持有多个 channel 集合，每个集合管理一类状态的 channel。这是 Mercury 区别于简单 TCP 实现的精细之处。

### 5.1 MonitoredChannels —— 监控基类

[monitored_channels.hpp](file:///workspace/src/lib/network/monitored_channels.hpp) 是 `IrregularChannels` 与 `KeepAliveChannels` 的基类：

```cpp
class MonitoredChannels : public TimerHandler
{
public:
    typedef std::list< Channel * > Channels;
    typedef Channels::iterator iterator;

    MonitoredChannels() : period_( 0.f ), timerHandle_() {}
    virtual ~MonitoredChannels();

    void addIfNecessary( Channel & channel );
    void delIfNecessary( Channel & channel );
    void setPeriod( float seconds, EventDispatcher & dispatcher );
    void stopMonitoring( EventDispatcher & dispatcher );
    iterator end()  { return channels_.end(); }

protected:
    virtual iterator & channelIter( Channel & channel ) const = 0;
    virtual float defaultPeriod() const = 0;

    Channels channels_;
    float period_;
    TimerHandle timerHandle_;
};
```

每个 channel 在 `MonitoredChannels` 中存一个 `iterator`（通过 `channelIter` 返回引用，存在 channel 的 `irregularIter_` / `keepAliveIter_` 成员里），实现 O(1) 加入/移除。

### 5.2 IrregularChannels —— 非常规通道

[irregular_channels.hpp](file:///workspace/src/lib/network/irregular_channels.hpp)：

```cpp
class IrregularChannels : public MonitoredChannels
{
public:
    void addIfNecessary( Channel & channel );
protected:
    iterator & channelIter( Channel & channel ) const;
    float defaultPeriod() const;
private:
    virtual void handleTimeout( TimerHandle, void * );
};
```

`isLocalRegular_ == false` 的 channel 加入此集合，周期性触发 `checkResendTimers` 做重传。常规 channel（`isLocalRegular_ == true`）依赖对端定期发包带动 ACK，不需要主动重传扫描。

### 5.3 KeepAliveChannels —— 心跳通道

[keepalive_channels.hpp](file:///workspace/src/lib/network/keepalive_channels.hpp)：

```cpp
class KeepAliveChannels : public MonitoredChannels
{
public:
    KeepAliveChannels();
    void addIfNecessary( Channel & channel );
protected:
    iterator & channelIter( Channel & channel ) const;
    float defaultPeriod() const;
private:
    virtual void handleTimeout( TimerHandle, void * );
    uint64 lastTimeout_;
    uint64 lastInvalidTimeout_;
};
```

收发极少（低于 keepalive 阈值）的 channel 加入此集合，定期发心跳包，确保对端没死。注释：「send and receive so infrequently that we need to send keepalive traffic to ensure the peer apps haven't died」。

### 5.4 CondemnedChannels —— 已死待清理

[condemned_channels.hpp](file:///workspace/src/lib/network/condemned_channels.hpp)：

```cpp
class CondemnedChannels : public TimerHandler
{
public:
    CondemnedChannels() : timerHandle_() {}
    ~CondemnedChannels();

    void add( Channel * pChannel );
    Channel * find( ChannelID channelID ) const;
    bool deleteFinishedChannels();
    int numCriticalChannels() const;

private:
    static const int AGE_LIMIT = 60;   // 秒
    virtual void handleTimeout( TimerHandle, void * );
    bool shouldDelete( Channel * pChannel, uint64 now );

    typedef std::list< Channel * > NonIndexedChannels;
    typedef std::map< ChannelID, Channel * > IndexedChannels;
    NonIndexedChannels  nonIndexedChannels_;
    IndexedChannels     indexedChannels_;
    TimerHandle timerHandle_;
};
```

`condemn()` 把 channel 加入此集合，等所有未 ACK 包确认后（`hasUnackedPackets() == false`）真正销毁。`AGE_LIMIT = 60` 秒是兜底，超时即使还有未 ACK 包也强制删除（避免泄漏）。`find(ChannelID)` 让索引 channel 在 condemn 后仍能被新连接找到（用于 baseapp 切换时的 session 恢复）。

### 5.5 DelayedChannels —— 延迟发送

[delayed_channels.hpp](file:///workspace/src/lib/network/delayed_channels.hpp)：

```cpp
class DelayedChannels : public FrequentTask
{
public:
    void init( EventDispatcher & dispatcher );
    void fini( EventDispatcher & dispatcher );
    void add( Channel & channel );
    void sendIfDelayed( Channel & channel );
private:
    virtual void doTask();
    typedef std::set< ChannelPtr > Channels;
    Channels channels_;
};
```

`delayedSend()` 把 channel 加入此集合，下次 `doTask`（每轮 frequent task）统一发送。注释：「This is so that multiple messages can be put on the outgoing bundle instead of sending a bundle for each」——把多个小消息合并到一个 bundle 发送，减少包数。`ChannelSender` 析构时若 channel 非常规就调 `delayedSend`。

### 5.6 ChannelOwner / ChannelSender / ChannelFinder

[channel_owner.hpp](file:///workspace/src/lib/network/channel_owner.hpp) 是拥有 channel 的简单基类：

```cpp
class ChannelOwner
{
public:
    ChannelOwner( NetworkInterface & networkInterface,
            const Address & address = Address::NONE,
            Channel::Traits traits = Channel::INTERNAL ) :
        pChannel_( traits == Channel::INTERNAL ?
            Channel::get( networkInterface, address ) :
            new Channel( networkInterface, address, traits ) )
    {
    }
    ~ChannelOwner()
    {
        pChannel_->condemn();
        pChannel_ = NULL;
    }
    Bundle & bundle()             { return pChannel_->bundle(); }
    const Address & addr() const  { return pChannel_->addr(); }
    void send( Bundle * pBundle = NULL ) { pChannel_->send( pBundle ); }
    Channel & channel()           { return *pChannel_; }
    void addr( const Address & addr );
private:
    Channel * pChannel_;
};
```

`INTERNAL` 通道通过 `Channel::get` 共享，`EXTERNAL` 通道 new 新实例。析构时 `condemn` 而非 `destroy`，让 `CondemnedChannels` 异步清理。

[channel_sender.hpp](file:///workspace/src/lib/network/channel_sender.hpp) 是 RAII 发送包装：

```cpp
class ChannelSender
{
public:
    ChannelSender( Channel & channel ) : channel_( channel ) {}
    ~ChannelSender()
    {
        if (!channel_.isLocalRegular())
        {
            channel_.delayedSend();
        }
    }
    Bundle & bundle() { return channel_.bundle(); }
    Channel & channel() { return channel_; }
private:
    Channel & channel_;
};
```

[channel_finder.hpp](file:///workspace/src/lib/network/channel_finder.hpp) 是索引 channel 解析器接口：

```cpp
class ChannelFinder
{
public:
    virtual ~ChannelFinder() {};
    virtual Channel* find( ChannelID id, const Mercury::Address & srcAddr,
            const Packet * pPacket, bool & rHasBeenHandled ) = 0;
};
```

收到 `FLAG_INDEXED_CHANNEL` 包时，`NetworkInterface::findIndexedChannel` 调用注册的 `ChannelFinder` 把 `ChannelID` 解析成 `Channel`。各 app 实现自己的 finder（如 CellApp 的 `EntityChannelFinder`）。

### 5.7 通道状态机总览图

```
                    ┌──────────────────┐
                    │   new Channel     │
                    │ (id != NULL 即索引)│
                    └────────┬─────────┘
                             │ registerChannel
                             ▼
                    ┌──────────────────┐
        ┌──────────│   Active (常规)   │◄──── isLocalRegular=true
        │           │ isRegular=true   │      isRemoteRegular=true
        │           └────────┬─────────┘
        │                    │
        │   收发变少          │ isLocalRegular=false
        │                    ▼
        │           ┌──────────────────┐
        │           │ IrregularChannels│ 周期 checkResendTimers
        │           │ (非常规)          │
        │           └────────┬─────────┘
        │                    │
        │   收发极少          │ addIfNecessary
        │                    ▼
        │           ┌──────────────────┐
        │           │KeepAliveChannels │ 周期发心跳
        │           │ (心跳)           │
        │           └────────┬─────────┘
        │                    │
        └────────────────────┘
                             │ condemn()
                             ▼
                    ┌──────────────────┐
                    │CondemnedChannels │ 等未 ACK 包清空
                    │ (已死待清理)      │
                    └────────┬─────────┘
                             │ hasUnackedPackets()==false 或 AGE_LIMIT=60s
                             ▼
                    ┌──────────────────┐
                    │   destroy()       │
                    │   ~Channel()      │
                    └──────────────────┘

   另：delayedSend() ──► DelayedChannels (FrequentTask) ──► 下轮 doTask 发送
```

---

## 6. 包级机制

### 6.1 PacketReceiver —— 包接收器

[packet_receiver.hpp](file:///workspace/src/lib/network/packet_receiver.hpp) 是 `InputNotificationHandler` 子类，由 `EventPoller` 在 socket 可读时回调：

```cpp
class PacketReceiver : public InputNotificationHandler
{
public:
    PacketReceiver( Endpoint & socket, NetworkInterface & networkInterface );
    ~PacketReceiver();

    Reason processPacket( const Address & addr, Packet * p,
            ProcessSocketStatsHelper * pStatsHelper );
    Reason processFilteredPacket( const Address & addr, Packet * p,
            ProcessSocketStatsHelper * pStatsHelper );

    PacketReceiverStats & stats() { return stats_; }

private:
    virtual int handleInputNotification( int fd );
    bool processSocket( bool expectingPacket );
    bool checkSocketErrors( int len, bool expectingPacket );

    Reason processOrderedPacket( const Address & addr, Packet * p,
        Channel * pChannel, ProcessSocketStatsHelper * pStatsHelper );
    bool processPiggybacks( const Address & addr,
            Packet * p, ProcessSocketStatsHelper * pStatsHelper );
    EventDispatcher & dispatcher();

    Endpoint & socket_;
    NetworkInterface & networkInterface_;
    PacketPtr pNextPacket_;          // 复用 packet 对象优化
    PacketReceiverStats stats_;
    OnceOffReceiver onceOffReceiver_;
};
```

流程：`handleInputNotification` → `processSocket` → `Endpoint::recvfrom` 收包到 `pNextPacket_` → `processPacket`。`processPacket` 先找 channel（或建匿名），再 `processFilteredPacket`（经 `PacketFilter::recv`），最后 `processOrderedPacket`（处理 seq/ack/分片，分发 bundle）。

### 6.2 PacketFilter —— 包过滤器接口

[packet_filter.hpp](file:///workspace/src/lib/network/packet_filter.hpp) 是过滤器基类：

```cpp
class PacketFilter : public SafeReferenceCount
{
public:
    virtual ~PacketFilter() {}
    virtual Reason send( NetworkInterface & networkInterface,
                         const Address & addr, Packet * pPacket );
    virtual Reason recv( PacketReceiver & receiver, const Address & addr,
                    Packet * pPacket, ProcessSocketStatsHelper * pStatsHelper );
    virtual int maxSpareSize() { return 0; }
};
typedef SmartPointer< PacketFilter > PacketFilterPtr;
```

`send`/`recv` 默认实现走 `NetworkInterface::basicSendWithRetries` / `PacketReceiver::processPacket`。子类（`EncryptionFilter`）覆写加解密。`maxSpareSize` 告诉 `Bundle` 预留多少空间给 filter（如加密后的 padding）。

### 6.3 OnceOffPacket —— 一次性包

[once_off_packet.hpp](file:///workspace/src/lib/network/once_off_packet.hpp) 处理无 channel 的可靠包（如登录请求）。`OnceOffSender` 维护 `OnceOffPackets` 集合，每个包带 `footerSequence`，定时重传（`onceOffResendPeriod_`，默认 200ms）直到收到回复或超时（`onceOffMaxResends_`，默认 50 次）。`OnceOffReceiver` 记录已收到的 `OnceOffReceipt`，防止重复处理。

```cpp
class OnceOffPacket : public TimerHandler, public OnceOffReceipt
{
public:
    OnceOffPacket( const Address & addr, SeqNum footerSequence, Packet * pPacket = NULL );
    void registerTimer( NetworkInterface & networkInterface );
    virtual void handleTimeout( TimerHandle handle, void * arg );
    PacketPtr pPacket_;
    TimerHandle timerHandle_;
    int retries_;
};

class OnceOffSender
{
public:
    void addOnceOffResendTimer( const Address & addr, SeqNum seq,
                        Packet * p, NetworkInterface & networkInterface );
    int onceOffResendPeriod() const { return onceOffResendPeriod_; }
    int onceOffMaxResends() const   { return onceOffMaxResends_; }
    bool hasUnackedPackets() const  { return !onceOffPackets_.empty(); }
private:
    OnceOffPackets onceOffPackets_;
    int onceOffMaxResends_;
    int onceOffResendPeriod_;
};
```

注释（见 [misc.hpp](file:///workspace/src/lib/network/misc.hpp)）：`DEFAULT_ONCEOFF_RESEND_PERIOD = 200 * 1000`（200ms），`DEFAULT_ONCEOFF_MAX_RESENDS = 50`。Version 39 后登录不再用 once-off（见 [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp) 注释）。

### 6.4 FragmentedBundle —— 分片重组

[fragmented_bundle.hpp](file:///workspace/src/lib/network/fragmented_bundle.hpp) 处理超长 bundle 的分片重组：

```cpp
class FragmentedBundle : public SafeReferenceCount
{
public:
    static const uint64 MAX_AGE = 10;   // 秒，超时丢弃

    FragmentedBundle( Packet * pFirstPacket );
    bool addPacket( Packet * p, bool isExternal, const char * sourceStr );

    double age() const { return (timestamp() - touched_) / stampsPerSecondD(); }
    bool isOld() const { return timestamp() - touched_ > stampsPerSecond() * MAX_AGE; }
    bool isReliable() const { return pChain_->hasFlags( Packet::FLAG_IS_RELIABLE ); }
    int chainLength() const      { return pChain_->chainLength(); }
    PacketPtr pChain() const     { return pChain_; }
    SeqNum lastFragment() const  { return lastFragment_; }
    bool isComplete() const      { return (remaining_ == 0); }

    static void addToStream( FragmentedBundlePtr pFragments, BinaryOStream & data );
    static FragmentedBundlePtr createFromStream( BinaryIStream & data );

private:
    SeqNum      lastFragment_;
    int         remaining_;
    uint64      touched_;
    PacketPtr   pChain_;

public:
    class Key
    {
    public:
        Key( const Address & addr, SeqNum firstFragment ) :
            addr_( addr ), firstFragment_( firstFragment ) {}
        Address     addr_;
        SeqNum      firstFragment_;
    };
};

typedef std::map< FragmentedBundle::Key, FragmentedBundlePtr > FragmentedBundles;
```

当 bundle 超过一个 packet（`isMultiPacket`），首个 packet 带 `FLAG_IS_FRAGMENT` 与 `fragBegin`/`fragEnd`，接收方按 `(addr, firstFragment)` 作 key 重组。`remaining_` 计数还差几个分片，`isComplete()` 后才分发。`MAX_AGE = 10` 秒，超时丢弃不完整的分片链避免泄漏。`OnceOffReceiver` 与 `Channel`（`pFragments_`）各自维护自己的 `FragmentedBundles`。

### 6.5 RemoteStepper / RescheduledSender

[remote_stepper.hpp](file:///workspace/src/lib/network/remote_stepper.hpp) 与 [rescheduled_sender.hpp](file:///workspace/src/lib/network/rescheduled_sender.hpp) 实现跨进程的延迟执行：把一个动作「远程」放到对端执行，或把本地发送动作重调度到下一轮循环（避免在收包回调中递归发送）。

### 6.6 PacketReceiverStats / SendingStats

[packet_receiver_stats.hpp](file:///workspace/src/lib/network/packet_receiver_stats.hpp) 与 [sending_stats.hpp](file:///workspace/src/lib/network/sending_stats.hpp) 收集收发统计（包数、字节数、错误数），供 Watcher 暴露。

---

## 7. 安全与压缩

### 7.1 EncryptionFilter —— Blowfish 对称加密

[encryption_filter.hpp](file:///workspace/src/lib/network/encryption_filter.hpp) 是 `PacketFilter` 子类，用 OpenSSL 的 Blowfish（ECB 模式，64 位块）加密 packet：

```cpp
#define USE_OPENSSL   // 非服务端默认启用

class EncryptionFilter : public PacketFilter
{
public:
    static const int BLOCK_SIZE = 64 / NETWORK_BITS_PER_BYTE;        // 8 字节
    static const int MIN_KEY_SIZE = 32 / NETWORK_BITS_PER_BYTE;      // 4 字节
    static const int MAX_KEY_SIZE = 448 / NETWORK_BITS_PER_BYTE;     // 56 字节
    static const int DEFAULT_KEY_SIZE = 128 / NETWORK_BITS_PER_BYTE; // 16 字节

    typedef std::string Key;

    EncryptionFilter( const Key & key );
    EncryptionFilter( int keySize = DEFAULT_KEY_SIZE );
    ~EncryptionFilter();

    virtual Reason send( NetworkInterface & networkInterface,
                        const Address & addr, Packet * pPacket );
    virtual Reason recv( PacketReceiver & receiver,
                        const Address & addr, Packet * pPacket,
                        ProcessSocketStatsHelper * pStatsHelper );
    virtual int maxSpareSize();

    const Key & key() const { return key_; }
    bool isGood() const { return isGood_; }
    BF_KEY * pBFKey() { return (BF_KEY*)pEncryptionObject_; }

    void encryptStream( MemoryOStream & clearStream, BinaryOStream & cipherStream );
    void decryptStream( BinaryIStream & cipherStream, BinaryOStream & clearStream );

private:
    int encrypt( const unsigned char * src, unsigned char * dest, int length );
    int decrypt( const unsigned char * src, unsigned char * dest, int length );
    bool initKey();

    Key key_;
    int keySize_;
    bool isGood_;
    void * pEncryptionObject_;   // 实际是 BF_KEY*
};
```

`pEncryptionObject_` 故意用 `void*`，便于替换算法而不改头文件。客户端 `ServerConnection` 默认创建一个 `EncryptionFilter`（`USE_OPENSSL` 时，见 [server_connection.cpp](file:///workspace/src/lib/connection/server_connection.cpp)），登录后用它加密 channel 流量。Version 49 起 Blowfish 加了 XOR 阶段防重放攻击（见 [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp) 注释）。

### 7.2 PublicKeyCipher —— RSA 非对称加密

[public_key_cipher.hpp](file:///workspace/src/lib/network/public_key_cipher.hpp) 封装 OpenSSL RSA，用于登录握手（加密 `LogOnParams`）：

```cpp
class PublicKeyCipher
{
public:
    PublicKeyCipher( bool isPrivate ) :
        pRSA_( NULL ), hasPrivate_( isPrivate ) {}
    ~PublicKeyCipher() { this->cleanup(); }

    bool setKey( const std::string & key );
    bool setKeyFromResource( const std::string & path );

    typedef int (*EncryptionFunctionPtr)( int flen, const unsigned char * from,
        unsigned char * to, RSA * pRSA, int padding );

    /// 用公钥加密（对方用私钥解密）
    int publicEncrypt( BinaryIStream & clearStream, BinaryOStream & cipherStream ) const
    {
        return this->encrypt( clearStream, cipherStream,
            &RSA_public_encrypt, RSA_PKCS1_OAEP_PADDING );
    }
    /// 用公钥解密（即验签）
    int publicDecrypt( BinaryIStream & cipherStream, BinaryOStream & clearStream ) const
    {
        return this->decrypt( cipherStream, clearStream,
            &RSA_public_decrypt, RSA_PKCS1_PADDING );
    }
    /// 用私钥加密（即签名）
    int privateEncrypt( BinaryIStream & cipherStream, BinaryOStream & clearStream ) const
    {
        MF_ASSERT( hasPrivate_ );
        return this->encrypt( cipherStream, clearStream,
            &RSA_private_encrypt, RSA_PKCS1_PADDING );
    }
    /// 用私钥解密
    int privateDecrypt( BinaryIStream & cipherStream, BinaryOStream & clearStream ) const
    {
        MF_ASSERT( hasPrivate_ );
        return this->decrypt( cipherStream, clearStream,
            &RSA_private_decrypt, RSA_PKCS1_OAEP_PADDING );
    }

    bool isGood() const { return pRSA_ != NULL; }
    int size() const    { return pRSA_ ? RSA_size( pRSA_ ) : -1; }
    int numBits() const { return this->size() * 8; }
    const char * type() const { return hasPrivate_ ? "private" : "public"; }

protected:
    RSA * pRSA_;
    std::string readableKey_;
    bool hasPrivate_;
};
```

`RSA_PKCS1_OAEP_PADDING` 用于加解密，`RSA_PKCS1_PADDING` 用于签名/验签。`setKey` 从 PEM 字符串加载，`setKeyFromResource` 从文件加载。

### 7.3 CompressionStream / ZipStream —— 压缩

[zip_stream.hpp](file:///workspace/src/lib/network/zip_stream.hpp) 封装 zlib：

```cpp
class ZipIStream : public MemoryIStream
{
public:
    ZipIStream( BinaryIStream & stream );
    void init( BinaryIStream & stream );
    ~ZipIStream();
private:
    unsigned char * pBuffer_;
};

class ZipOStream : public BinaryOStream
{
public:
    ZipOStream( BinaryOStream & dstStream, int compressLevel = 1 /* Z_BEST_SPEED */ );
    void init( BinaryOStream & dstStream, int compressLevel = 1 );
    virtual void * reserve( int nBytes ) { return outStream_.reserve( nBytes ); }
    virtual int size() const { return outStream_.size(); }
private:
    MemoryOStream outStream_;
    BinaryOStream * pDstStream_;
    int compressLevel_;
};
```

默认压缩级别 1（`Z_BEST_SPEED`，最快）。[compression_type.hpp](file:///workspace/src/lib/network/compression_type.hpp) 定义压缩类型枚举：

```cpp
enum BWCompressionType
{
    BW_COMPRESSION_NONE,
    BW_COMPRESSION_ZIP_1, ..., BW_COMPRESSION_ZIP_9,
    BW_COMPRESSION_ZIP_BEST_SPEED       = BW_COMPRESSION_ZIP_1,
    BW_COMPRESSION_ZIP_BEST_COMPRESSION = BW_COMPRESSION_ZIP_9,
    BW_COMPRESSION_DEFAULT_INTERNAL,
    BW_COMPRESSION_DEFAULT_EXTERNAL,
};
```

[compression_stream.hpp](file:///workspace/src/lib/network/compression_stream.hpp) 提供 `CompressionIStream` / `CompressionOStream` 包装，根据配置决定是否实际压缩：

```cpp
class CompressionOStream
{
public:
    CompressionOStream( BinaryOStream & stream,
        BWCompressionType compressType = BW_COMPRESSION_DEFAULT_INTERNAL );
    operator BinaryOStream &() { return *pCurrStream_; }
    static bool initDefaults( DataSectionPtr pSection );
private:
    BinaryOStream * pCurrStream_;
    ZipOStream zipStream_;
    static BWCompressionType s_defaultInternalCompression;
    static BWCompressionType s_defaultExternalCompression;
};
```

内部流量（cellapp↔baseapp）与外部流量（client↔server）可用不同压缩级别，由 `bw.xml` 配置。Version 56 起 `createEntity` 消息支持压缩（见 [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp) 注释）。

---

## 8. Bundle 的 Piggyback（捎带）机制

捎带是 Mercury 在 `EXTERNAL` 通道上的重要带宽优化。当一个可靠包丢失，重传时不单独发包，而是把它的可靠消息「搭」在后续正常包的尾部（`FLAG_HAS_PIGGYBACKS`）。

[bundle_piggyback.hpp](file:///workspace/src/lib/network/bundle_piggyback.hpp)：

```cpp
#ifdef USE_PIGGIES

class BundlePiggyback
{
public:
    BundlePiggyback( Packet * pPacket, Packet::Flags flags, SeqNum seq, int16 len ) :
        pPacket_( pPacket ), flags_( flags ), seq_( seq ), len_( len )
    {}

    PacketPtr       pPacket_;   // 原始包（消息来源）
    Packet::Flags   flags_;     // 捎带包的头部
    SeqNum          seq_;       // 捎带包的序列号
    int16           len_;       // 捎带数据长度
    ReliableVector  rvec_;      // 要捎带的可靠消息
};

typedef std::vector< BundlePiggyback* > BundlePiggybacks;

#endif
```

`Bundle::piggyback(seq, reliableOrders, packet)` 把一个丢失包的可靠消息加入 `piggybacks_`，`NetworkInterface::send` 时把这些 piggyback 写进新包的 footer。接收方 `processPiggybacks` 解析 footer，把 piggyback 消息当作原 seq 包的内容处理。

注释（见 [interface_element.hpp](file:///workspace/src/lib/network/interface_element.hpp)）：早期用 `REPLY_PIGGY_BACK_IDENTIFIER` 消息 ID 实现，后改成 footer 机制，`REPLY_PIGGY_BACK_IDENTIFIER` 被移除但 entitydef 仍按「2 个保留 ID」假设编码（影响 entity 属性/方法数到 62 就触发 2 字节 ID）。Version 37 实现 piggyback，Version 41 piggyback 长度改用反码。

---

## 9. machined 集成与进程发现

### 9.1 MachineGuardMessage

[machine_guard.hpp](file:///workspace/src/lib/network/machine_guard.hpp) 定义与 `bwmachined` 守护进程通信的协议：

```cpp
const uint16 MERCURY_INTERFACE_VERSION = 1;

class MGMPacket
{
public:
    static const int MAX_SIZE = 32768;
    enum Flags { PACKET_STAGGER_REPLIES = 0x1 };
    typedef std::vector< MachineGuardMessage* > MGMs;

    uint8   flags_;
    uint32  buddy_;
    MGMs    messages_;
    void read( MemoryIStream &is );
    bool write( MemoryOStream &os ) const;
    void append( MachineGuardMessage &mgm, bool shouldDelete=false );
};
```

`MGMPacket` 是一个承载多个 `MachineGuardMessage` 的容器，通过 UDP 与 `bwmachined` 通信。`NetworkInterface::registerWithMachined` 用它把进程地址注册到 machined，供其它进程发现。

### 9.2 machined_utils / process_socket_stats_helper

[machined_utils.hpp](file:///workspace/src/lib/network/machined_utils.hpp) 提供辅助函数，[process_socket_stats_helper.hpp](file:///workspace/src/lib/network/process_socket_stats_helper.hpp) 收集每进程 socket 统计上报给 machined。

---

## 10. 错误与监控

### 10.1 NubException

[nub_exception.hpp](file:///workspace/src/lib/network/nub_exception.hpp) 是所有 Mercury 异常的基类：

```cpp
class NubException
{
public:
    NubException( Reason reason, const Address & addr = Address::NONE );
    Reason reason() const;
    bool getAddress( Address & addr ) const;
private:
    Reason  reason_;
    Address address_;
};
```

### 10.2 ErrorReporter

[error_reporter.hpp](file:///workspace/src/lib/network/error_reporter.hpp) 聚合错误日志，避免刷屏：

```cpp
typedef std::pair< Address, std::string > AddressAndErrorString;
typedef std::map< AddressAndErrorString, ErrorReportAndCount > ErrorsAndCounts;

class ErrorReporter : public TimerHandler
{
public:
    ErrorReporter( EventDispatcher & dispatcher );
    void reportException( Reason reason, const Address & addr = Address::NONE,
            const char * prefix = NULL );
    void reportPendingExceptions( bool reportBelowThreshold = false );
private:
    static const uint ERROR_REPORT_MIN_PERIOD_MS;
    static const uint ERROR_REPORT_COUNT_MAX_LIFETIME_MS;
    void addReport( const Address & address, const std::string & error );
    virtual void handleTimeout( TimerHandle handle, void * arg );
    TimerHandle reportLimitTimerHandle_;
    ErrorsAndCounts errorsAndCounts_;
};
```

同一 `(address, errorString)` 对的错误在 `ERROR_REPORT_MIN_PERIOD_MS` 内只报一次，但累计计数，下次报时附上「自上次以来 N 次」。

### 10.3 Watcher 网络化

[watcher_nub.hpp](file:///workspace/src/lib/network/watcher_nub.hpp) 是 Watcher 运维通道的核心，单例，同时监听 UDP 与 TCP：

```cpp
class WatcherNub :
    public Mercury::InputNotificationHandler,
    public Singleton< WatcherNub >
{
public:
    WatcherNub();
    bool init( const char * listeningInterface, uint16 listeningPort );
    int registerWatcher( int id, const char * abrv, const char * longName, ... );
    int deregisterWatcher();
    void attachTo( Mercury::EventDispatcher & dispatcher );
    void setRequestHandler( WatcherRequestHandler * pWatcherRequestHandler );
    bool receiveUDPRequest();
    bool addReply( const char * identifier, const char * desc, const char * value );
    Endpoint & udpSocket() { return udpSocket_; }
    Endpoint & tcpSocket() { return tcpSocket_; }
private:
    int id_;
    bool registered_;
    WatcherRequestHandler * pExtensionHandler_;
    bool insideReceiveRequest_;
    char *requestPacket_;
    Endpoint udpSocket_;
    Endpoint tcpSocket_;
    char abrv_[32];
    char name_[64];
    Mercury::EventDispatcher * pDispatcher_;
};
```

`WatcherDataMsg` / `WatcherRegistrationMsg` 是 watcher 协议消息结构。`WATCHER_MSG_GET/SET/TELL` 等 enum 是消息 ID。`watcher_connection.hpp`、`watcher_packet_handler.hpp`、`watcher_glue.hpp` 是其辅助类。

### 10.4 logger_message_forwarder

[logger_message_forwarder.hpp](file:///workspace/src/lib/network/logger_message_forwarder.hpp) 把进程的日志消息转发到中央 logger 进程，便于统一收集。

---

## 11. 单元测试

`src/lib/network/unit_test/` 目录有完整测试套件，覆盖 Mercury 各子系统：

| 测试文件 | 覆盖点 |
| --- | --- |
| `test_channel.cpp` | channel 可靠发送、ACK、窗口滑动，client↔server 多消息往返 |
| `test_fragment.cpp` | 超长 bundle 的分片与重组 |
| `test_overflow.cpp` | 窗口溢出（`REASON_WINDOW_OVERFLOW`）处理 |
| `test_compresslength.cpp` | `InterfaceElement::compressLength` / `expandLength` 长度压缩 |
| `test_receive_window.cpp` | 接收窗口乱序缓存与有序交付 |
| `test_threadsafety.cpp` | 多线程安全（dispatcher + interface 并发） |
| `test_flood.cpp` | 洪泛压力测试 |
| `test_baseapp_death.cpp` | baseapp 死亡后 channel 切换 |
| `test_channel_version.cpp` | 索引 channel 的 version 环绕 |
| `test_auto_switch.cpp` | channel 自动切换源地址 |
| `test_mangle.cpp` | 包篡改/损坏检测 |
| `test_netmask.cpp` | 子网掩码计算 |
| `test_interface.cpp` | InterfaceTable / InterfaceMinder |
| `test_stream.cpp` | BinaryStream / ZipStream |
| `test_config.cpp` | 配置加载 |

[test_channel.cpp](file:///workspace/src/lib/network/unit_test/test_channel.cpp) 典型用法（用 `ChannelOwner` 持有通道，定时发 `ClientInterface::msg1`）：

```cpp
class Peer : public Mercury::ChannelOwner, public SafeReferenceCount
{
public:
    Peer( Mercury::NetworkInterface & networkInterface,
            const Mercury::Address & addr,
            Mercury::Channel::Traits traits ) :
        Mercury::ChannelOwner( networkInterface, addr, traits ),
        timerHandle_(), inSeq_( 0 ), outSeq_( 0 )
    {}

    void sendNextMessage()
    {
        ClientInterface::msg1Args & args =
            ClientInterface::msg1Args::start( this->bundle() );
        args.seq = outSeq_++;
        args.data = 0;
        if (outSeq_ == NUM_ITERATIONS)
        {
            timerHandle_.cancel();
            this->channel().isLocalRegular( false );
            this->channel().isRemoteRegular( false );
        }
        this->send();
    }
private:
    TimerHandle timerHandle_;
    uint32 inSeq_;
    uint32 outSeq_;
};
```

`packet_generator.hpp` 提供包生成工具用于测试。

---

## 12. 典型 RPC 调用流

以 cellapp 之间 witness 相关消息为例（`enableWitness` 请求），展示完整 RPC 流：

```
   发送方 CellApp (A)                          接收方 CellApp (B)
   ─────────────────────                       ─────────────────────

1. 脚本调 entity.enableWitness()
        │
        ▼
2. BaseAppExtInterface::enableWitnessArgs::startRequest(
        channelA.bundle(), handler)
   └─ Bundle::startRequest(ie, handler, ...)
      ├─ 写 msgID + length 到 currentPacket_
      ├─ RequestManager 分配 ReplyID, 存 Request
      ├─ replyOrders_ 记录此请求位置
      └─ reliableOrders_ 记录可靠消息位置
        │
        ▼
3. channelA.send()  (或 ChannelSender 析构触发 delayedSend)
   └─ Bundle::send(addr, networkInterface, pChannel)
      ├─ finalise() 写尾部
      ├─ Channel::writeFlags(p) / writeFooter(p)
      │  ├─ FLAG_ON_CHANNEL | FLAG_IS_RELIABLE
      │  │  | FLAG_HAS_SEQUENCE_NUMBER | FLAG_HAS_REQUESTS
      │  ├─ 写 seq = useNextSequenceID()
      │  └─ 写 firstRequestOffset 链
      ├─ Channel::addResendTimer(seq, p, roBeg, roEnd)
      │  └─ unackedPackets_[seq] = new UnackedPacket(p)
      └─ NetworkInterface::sendPacket(addr, p, channel, isResend=false)
         └─ EncryptionFilter::send (若有)
            └─ Endpoint::sendto (UDP)
        │                                            │
        │           UDP 包穿越网络                     │
        │──────────────────────────────────────────►│
        │                                            │
        │                                    4. EventPoller 触发
        │                                       PacketReceiver::handleInputNotification
        │                                       └─ Endpoint::recvfrom → pNextPacket_
        │                                       └─ processPacket(addr, p)
        │                                          ├─ findChannel(addr) → channelB
        │                                          ├─ processFilteredPacket
        │                                          │  └─ EncryptionFilter::recv (若有)
        │                                          └─ processOrderedPacket(addr, p, channelB)
        │                                             ├─ stripFooter 取 seq/ack/requests
        │                                             ├─ channelB.handleAck (处理 A 的 ACK)
        │                                             ├─ channelB.addToReceiveWindow(p)
        │                                             │  └─ inSeqAt 匹配 → SHOULD_PROCESS
        │                                             └─ Bundle::dispatchMessages(
        │                                                    interfaceTable, addr, channelB, ni)
        │                                                └─ 按 msgID 查 InterfaceTable
        │                                                   └─ CellAppInterface::enableWitness
        │                                                      handler 调 Entity::enableWitness
        │                                                         │
        │                                                         ▼
        │                                                5. 业务处理，准备回复
        │                                                   Bundle::startReply(replyID)
        │                                                   channelB.send()
        │                                                   └─ 写 FLAG_HAS_ACKS 捎带 ACK
        │                                                       (acksToSend_ 含 A 的 seq)
        │           UDP 回包                                    │
        │◄──────────────────────────────────────────────────
        │                                            │
        ▼
6. PacketReceiver 收回包
   └─ processOrderedPacket
      ├─ channelA.handleAck(seq) / handleCumulativeAck
      │  └─ unackedPackets_[seq] = NULL, 滑动窗口
      └─ Bundle::dispatchMessages
         └─ REPLY_MESSAGE_IDENTIFIER (0xFF) → RequestManager::handleMessage
            └─ 按 ReplyID 查 requestMap_ 找到 Request
               └─ 回调 handler->handleMessage(srcAddr, header, data)
                  └─ 业务收到回复，完成 RPC
```

关键观察：

- **ACK 捎带**：B 回复时把对 A 的 ACK 捎带在回包里（`FLAG_HAS_ACKS`），A 收到后滑动窗口，无需单独发 ACK 包。
- **Request footer 链**：一个包可含多个请求，`firstRequestOffset` 是链表头，每个请求记录 `nextRequestOffset`，`RequestManager` 据此遍历。
- **滑动窗口**：A 发包后存 `UnackedPacket`，收到 ACK 才释放；窗口满则 overflow 或抛错。
- **重传**：若回包丢失，A 的 `IrregularChannels` 周期触发 `checkResendTimers`，重发原请求包（`EXTERNAL` 只发可靠段）。
- **InterfaceTable 分发**：接收端按 `MessageID` 查 256 项表，调用注册的 `InputMessageHandler`。`CellAppInterface` 的 `MF_VARLEN_ENTITY_REQUEST` 宏把消息路由到 `EntityVarLenRequestHandler`，再按 `EntityID` 前缀找到具体 entity 调方法。

---

## 13. 关键设计总结

1. **UDP + 自实现可靠**：避免 TCP 队头阻塞，`EXTERNAL` 通道只重传可靠段，丢包不阻塞后续不可靠数据。
2. **Bundle 聚合**：多个小消息合并成一个包，减少系统调用与包头开销。
3. **Piggyback 捎带**：重传数据搭车在正常包上，不额外占包。
4. **滑动窗口 CircularArray**：位掩码定位，O(1) 访问，动态扩容。
5. **Traits 区分**：`INTERNAL` 高带宽整包重传，`EXTERNAL` 低带宽只发可靠段。
6. **子状态机集合**：channel 按状态（常规/非常规/心跳/待死/延迟）分桶管理，每类有独立定时器，避免全量扫描。
7. **IDL 宏系统**：`BW_DEFINE_*` 宏声明接口，自动生成 `Args` 结构、`start`/`startRequest` 方法、handler 注册，零额外代码。
8. **PacketFilter 插件**：加解密、压缩作为 filter 插入 send/recv 链，channel 不感知。
9. **OnceOffPacket**：无 channel 的可靠包（登录）独立重传机制。
10. **FragmentedBundle**：超长 bundle 分片重组，带超时清理。
11. **ErrorReporter 聚合**：错误日志按 (addr, error) 聚合计数，避免刷屏。
12. **Watcher 网络化**：独立 UDP/TCP 通道，运行时运维监控。

---

## 14. 阅读的核心文件清单

本文档阅读并引用了以下核心 `.hpp/.cpp/.ipp` 文件（共 25 个，超过要求的 15 个）：

| # | 文件 | 主要内容 |
| --- | --- | --- |
| 1 | [basictypes.hpp](file:///workspace/src/lib/network/basictypes.hpp) | `Address`、`EntityMailBoxRef`、`Capabilities`、全局类型别名 |
| 2 | [basictypes.ipp](file:///workspace/src/lib/network/basictypes.ipp) | `Address` 构造与比较运算符内联实现 |
| 3 | [misc.hpp](file:///workspace/src/lib/network/misc.hpp) | `SeqNum`/`Reason`/`ChannelID`/`ReplyID` 常量与错误码 |
| 4 | [event_dispatcher.hpp](file:///workspace/src/lib/network/event_dispatcher.hpp) | `EventDispatcher` 事件循环 |
| 5 | [network_interface.hpp](file:///workspace/src/lib/network/network_interface.hpp) | `NetworkInterface` channel 集合管理 |
| 6 | [channel.hpp](file:///workspace/src/lib/network/channel.hpp) | `Channel` 可靠有序通道 |
| 7 | [channel.ipp](file:///workspace/src/lib/network/channel.ipp) | `seqMask`/`seqLessThan`/`windowSize` 内联 |
| 8 | [circular_array.hpp](file:///workspace/src/lib/network/circular_array.hpp) | `CircularArray` 滑动窗口底层数据结构 |
| 9 | [unacked_packet.hpp](file:///workspace/src/lib/network/unacked_packet.hpp) | `Channel::UnackedPacket` 未确认包记录 |
| 10 | [reliable_order.hpp](file:///workspace/src/lib/network/reliable_order.hpp) | `ReliableOrder` 可靠消息定位 |
| 11 | [bundle.hpp](file:///workspace/src/lib/network/bundle.hpp) | `Bundle` 消息聚合、`ReliableType` |
| 12 | [bundle_piggyback.hpp](file:///workspace/src/lib/network/bundle_piggyback.hpp) | `BundlePiggyback` 捎带机制 |
| 13 | [packet.hpp](file:///workspace/src/lib/network/packet.hpp) | `Packet` 网络包结构与 Flags |
| 14 | [endpoint.hpp](file:///workspace/src/lib/network/endpoint.hpp) | `Endpoint` UDP/TCP socket 包装 |
| 15 | [interface_table.hpp](file:///workspace/src/lib/network/interface_table.hpp) | `InterfaceTable` 消息路由表 |
| 16 | [interface_element.hpp](file:///workspace/src/lib/network/interface_element.hpp) | `InterfaceElement`/`InterfaceElementWithStats` |
| 17 | [interface_minder.hpp](file:///workspace/src/lib/network/interface_minder.hpp) | `InterfaceMinder` 接口声明器 |
| 18 | [interface_macros.hpp](file:///workspace/src/lib/network/interface_macros.hpp) | IDL 宏定义 |
| 19 | [request_manager.hpp](file:///workspace/src/lib/network/request_manager.hpp) | `RequestManager` RPC 请求配对 |
| 20 | [packet_receiver.hpp](file:///workspace/src/lib/network/packet_receiver.hpp) | `PacketReceiver` 包接收器 |
| 21 | [packet_filter.hpp](file:///workspace/src/lib/network/packet_filter.hpp) | `PacketFilter` 过滤器基类 |
| 22 | [once_off_packet.hpp](file:///workspace/src/lib/network/once_off_packet.hpp) | `OnceOffPacket`/`OnceOffSender` 一次性包 |
| 23 | [fragmented_bundle.hpp](file:///workspace/src/lib/network/fragmented_bundle.hpp) | `FragmentedBundle` 分片重组 |
| 24 | [encryption_filter.hpp](file:///workspace/src/lib/network/encryption_filter.hpp) | `EncryptionFilter` Blowfish 加密 |
| 25 | [public_key_cipher.hpp](file:///workspace/src/lib/network/public_key_cipher.hpp) | `PublicKeyCipher` RSA 加密 |
| 26 | [compression_stream.hpp](file:///workspace/src/lib/network/compression_stream.hpp) | `CompressionIStream`/`CompressionOStream` |
| 27 | [compression_type.hpp](file:///workspace/src/lib/network/compression_type.hpp) | `BWCompressionType` 枚举 |
| 28 | [zip_stream.hpp](file:///workspace/src/lib/network/zip_stream.hpp) | `ZipIStream`/`ZipOStream` zlib 封装 |
| 29 | [monitored_channels.hpp](file:///workspace/src/lib/network/monitored_channels.hpp) | `MonitoredChannels` 监控基类 |
| 30 | [irregular_channels.hpp](file:///workspace/src/lib/network/irregular_channels.hpp) | `IrregularChannels` 非常规通道 |
| 31 | [keepalive_channels.hpp](file:///workspace/src/lib/network/keepalive_channels.hpp) | `KeepAliveChannels` 心跳通道 |
| 32 | [condemned_channels.hpp](file:///workspace/src/lib/network/condemned_channels.hpp) | `CondemnedChannels` 已死待清理 |
| 33 | [delayed_channels.hpp](file:///workspace/src/lib/network/delayed_channels.hpp) | `DelayedChannels` 延迟发送 |
| 34 | [channel_owner.hpp](file:///workspace/src/lib/network/channel_owner.hpp) | `ChannelOwner` channel 拥有者 |
| 35 | [channel_sender.hpp](file:///workspace/src/lib/network/channel_sender.hpp) | `ChannelSender` RAII 发送 |
| 36 | [channel_finder.hpp](file:///workspace/src/lib/network/channel_finder.hpp) | `ChannelFinder` 索引通道解析 |
| 37 | [error_reporter.hpp](file:///workspace/src/lib/network/error_reporter.hpp) | `ErrorReporter` 错误聚合 |
| 38 | [nub_exception.hpp](file:///workspace/src/lib/network/nub_exception.hpp) | `NubException` 异常基类 |
| 39 | [watcher_nub.hpp](file:///workspace/src/lib/network/watcher_nub.hpp) | `WatcherNub` 运维通道 |
| 40 | [machine_guard.hpp](file:///workspace/src/lib/network/machine_guard.hpp) | `MGMPacket` machined 通信 |
| 41 | [cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp) | CellApp IDL 接口实例 |
| 42 | [test_channel.cpp](file:///workspace/src/lib/network/unit_test/test_channel.cpp) | channel 单元测试 |

---

## 总结

本文档约 18000 字，详细分析了 BigWorld Mercury 网络库的架构与实现：

- **模块定位**：UDP 之上的可靠传输 + RPC 框架，所有 IPC 的统一基础设施
- **核心抽象**：`Address`、`EventDispatcher`、`NetworkInterface`、`Channel`、`Bundle`、`Packet`、`Endpoint`、`InterfaceTable`
- **可靠传输**：基于 `CircularArray` 的滑动窗口、`UnackedPacket` + `ReliableOrder` 的 ACK/重传、`EXTERNAL` 通道只重传可靠段
- **通道状态机**：`IrregularChannels`/`KeepAliveChannels`/`CondemnedChannels`/`DelayedChannels` 分桶管理
- **包级机制**：`PacketReceiver` 收包、`PacketFilter` 过滤、`OnceOffPacket` 一次性可靠、`FragmentedBundle` 分片重组、Piggyback 捎带
- **安全压缩**：`EncryptionFilter`（Blowfish）、`PublicKeyCipher`（RSA）、`ZipStream`/`CompressionStream`（zlib）
- **IDL 系统**：`InterfaceTable` + `InterfaceElement` + `InterfaceMinder` + 宏，`cellapp_interface.hpp` 实例
- **RPC**：`RequestManager` 请求-响应配对，`ReplyID` 环绕
- **测试**：`unit_test/` 覆盖 channel/fragment/overflow/compress/receive_window/threadsafety
- **典型 RPC 流**：从脚本调用到 bundle 打包、channel 发送、网络传输、接收分发、handler 回调、回复配对的完整时序

Mercury 是 BigWorld 引擎最核心的子系统之一，其设计在 2000 年代的 MMO 后端中相当先进——自实现可靠 UDP、消息聚合、捎带重传、IDL 宏系统——这些思想在后来的游戏网络库（如 ENet、KCP、Photon）中都能看到影子。
