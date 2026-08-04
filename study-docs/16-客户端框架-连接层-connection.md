# BigWorld Engine 2.0.1 客户端框架 - 连接层（connection）

> 源码位置：`src/lib/connection/`
> 模块定位：**客户端用于连接服务器的会话层**。它建立在 Mercury 网络库之上，封装了客户端登录、会话维持、实体消息分发、数据下载等所有「客户端↔服务端」交互细节，是客户端业务逻辑（entity、脚本）与服务端之间的桥梁。
> 角色：客户端不直接使用 `Mercury::NetworkInterface`，而是通过 `ServerConnection` 这一层抽象。`ServerConnection` 内部持有 Mercury 通道，对外暴露 `logOn` / `disconnect` / `addMove` / `processInput` 等高层 API，并通过 `ServerMessageHandler` 回调把服务端消息转交给上层（如 `EntityFactory`、脚本）。

---

## 1. 模块定位与核心职责

BigWorld 客户端要与服务端协作，需要处理：

- **登录流程**：与 loginapp 握手（RSA 加密）、获取 baseapp 地址、与 baseapp 建立会话
- **会话维持**：心跳、不活跃检测、断线重连、baseapp 切换
- **实体同步**：接收 entity 创建/属性/方法/移动消息，分发到对应 entity
- **客户端上行**：发送 avatar 位置更新、entity 方法调用、控制请求
- **数据下载**：服务端推送的资源/数据分段下载与重组
- **时间同步**：服务端 game time 与客户端时间对齐
- **带宽统计**：区分移动/非移动/开销字节，上报给服务端做限速

`src/lib/connection/` 约 30 个源文件，把这些封装成一套清晰的会话 API。

### 1.1 与其它模块的关系

```
                ┌──────────────────────────────────────────┐
                │         客户端业务层（Entity/脚本）        │
                │   通过 ServerMessageHandler 接收回调      │
                │   通过 ServerConnection 上行 API 发送     │
                └────────────────────┬─────────────────────┘
                                     │ 调用 / 回调
                                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │                   connection 层                          │
   │  ┌──────────────────────────────────────────────────┐    │
   │  │              ServerConnection                    │    │
   │  │  (会话核心，持有 Channel/Interface/Handler)       │    │
   │  │  + BundlePrimer + ChannelTimeOutHandler          │    │
   │  └──────┬─────────────────┬──────────────┬─────────┘    │
   │         │                 │              │              │
   │         ▼                 ▼              ▼              │
   │  ┌────────────┐   ┌──────────────┐  ┌──────────────┐    │
   │  │LoginHandler│   │ DataDownload │  │ServerTimeHndr│    │
   │  │ (登录状态机)│   │ (数据下载)   │  │(时间同步)    │    │
   │  └─────┬──────┘   └──────────────┘  └──────────────┘    │
   │        │                                                 │
   │   ┌────┴─────────────────────────┐                       │
   │   ▼                              ▼                       │
   │ ┌──────────────────┐    ┌──────────────────┐             │
   │ │LoginAppLoginReq  │    │BaseAppLoginRequest│             │
   │ │(登录 loginapp)    │    │(登录 baseapp)     │             │
   │ │+ RetryingRequest │    │+ 自有 Channel      │             │
   │ └──────────────────┘    └──────────────────┘             │
   │                                                          │
   │   接口定义（IDL）：                                       │
   │   LoginInterface / ClientInterface / BaseAppExtInterface │
   │   Common*Interface / MessageHandlers                     │
   └────────────────────────────┬─────────────────────────────┘
                                │ 基于
                                ▼
   ┌──────────────────────────────────────────────────────────┐
   │                   Mercury (src/lib/network)              │
   │   NetworkInterface / Channel / Bundle / Packet           │
   │   EncryptionFilter (Blowfish) / PublicKeyCipher (RSA)    │
   └────────────────────────────┬─────────────────────────────┘
                                │ 依赖
                                ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │   cstdmf     │    │   math       │    │  entitydef   │
   │ (timer/md5/  │    │ (vector3/    │    │(EntityDescri-│
   │  binary_str) │    │  stat)       │    │ ption 反序列化)│
   └──────────────┘    └──────────────┘    └──────────────┘
```

被依赖方：

- `network`（Mercury）：`Channel`、`NetworkInterface`、`EventDispatcher`、`EncryptionFilter`、`PublicKeyCipher`、`Bundle`、`InterfaceTable`
- `cstdmf`：`md5.hpp`（digest）、`concurrency.hpp`、`memory_stream.hpp`、`debug.hpp`、`smartpointer.hpp`
- `math`：`vector3.hpp`、`stat_with_rates_of_change.hpp`（带宽统计）
- `entitydef`：客户端按 `EntityDescription` 反序列化收到的 entity 创建/属性更新（通过 `handleEntityMessage` 转交）

使用方：

- `src/client/`：客户端主程序（`connection_control`、`entity`、`player` 等）持有 `ServerConnection`，实现 `ServerMessageHandler`
- `src/server/bots/`：bot 进程也用 `ServerConnection` 模拟客户端做压测

---

## 2. 模块全景与文件清单

`src/lib/connection/` 文件按职责分组：

| 组 | 代表文件 | 职责 |
| --- | --- | --- |
| 会话核心 | `server_connection.hpp/cpp/ipp` | `ServerConnection` 会话管理与上行 API |
| 登录流程 | `login_handler.hpp/cpp`、`loginapp_login_request.hpp/cpp`、`baseapp_login_request.hpp/cpp`、`retrying_request.hpp/cpp`、`login_reply_record.hpp` | 登录状态机与两阶段登录 |
| 登录参数 | `log_on_params.hpp/cpp`、`log_on_status.hpp`、`loginapp_public_key.hpp` | 登录参数、状态码、公钥 |
| 加密握手 | `rsa_stream_encoder.hpp`、`stream_encoder.hpp` | RSA 流加密器接口与实现 |
| 接口定义 | `login_interface.hpp/cpp`、`client_interface.hpp/cpp`、`baseapp_ext_interface.hpp`、`common_baseapp_interface.hpp`、`common_client_interface.hpp` | Mercury IDL 接口 |
| 消息分发 | `message_handlers.hpp`、`server_message_handler.hpp` | 消息 handler 模板与上层回调接口 |
| 数据下载 | `data_download.hpp/cpp`、`download_segment.hpp` | 分段数据下载与重组 |
| 预编译 | `pch.hpp/cpp` | 预编译头 |

---

## 3. ServerConnection —— 会话核心

定义于 [server_connection.hpp](file:///workspace/src/lib/connection/server_connection.hpp)，是连接层最核心的类。它继承三个基类：

- `Mercury::BundlePrimer`：每次 bundle 清空后注入公共头部消息（avatarUpdate）
- `Mercury::ChannelTimeOutHandler`：channel 超时回调（断线通知）
- `TimerHandler`：定时器回调（统计更新）

```cpp
class ServerConnection :
    public Mercury::BundlePrimer,
    public Mercury::ChannelTimeOutHandler,
    public TimerHandler
{
public:
    ServerConnection();
    ~ServerConnection();

    bool processInput();
    void registerInterfaces( Mercury::NetworkInterface & networkInterface );

    void setInactivityTimeout( float seconds );

    LogOnStatus logOn( ServerMessageHandler * pHandler,
        const char * serverName,
        const char * username,
        const char * password,
        uint16 port = 0 );

    LoginHandlerPtr logOnBegin(
        const char * serverName,
        const char * username,
        const char * password,
        uint16 port = 0 );

    LogOnStatus logOnComplete(
        LoginHandlerPtr pLRH,
        ServerMessageHandler * pHandler );

    void enableReconfigurePorts() { tryToReconfigurePorts_ = true; }
    void enableEntities();

    bool online() const;
    bool offline() const            { return !this->online(); }
    void disconnect( bool informServer = true );

    void channel( Mercury::Channel & channel ) { pChannel_ = &channel; }
    void pInterface( Mercury::NetworkInterface * pInterface );
    bool hasInterface() const { return pInterface_ != NULL; }

    Mercury::Channel & channel();
    Mercury::Bundle & bundle() { return this->channel().bundle(); }
    const Mercury::Address & addr() const;

    Mercury::EncryptionFilterPtr pFilter() { return pFilter_; }

    void addMove( EntityID id, SpaceID spaceID, EntityID vehicleID,
        const Vector3 & pos, float yaw, float pitch, float roll,
        bool onGround, const Vector3 & globalPos );

    BinaryOStream & startProxyMessage( int messageId );
    BinaryOStream & startAvatarMessage( int messageId );
    BinaryOStream & startEntityMessage( int messageId, EntityID entityId );

    void requestEntityUpdate( EntityID id,
        const CacheStamps & stamps = CacheStamps() );

    const std::string & errorMsg() const { return errorMsg_; }
    EntityID connectedID() const         { return id_; }
    void sessionKey( SessionKey key )    { sessionKey_ = key; }

    StreamEncoder * pLogOnParamsEncoder() const { return pLogOnParamsEncoder_; }
    void pLogOnParamsEncoder( StreamEncoder * pEncoder )
        { pLogOnParamsEncoder_ = pEncoder; }

    // ---- 统计 ----
    float latency() const;
    double bpsIn() const;
    double bpsOut() const;
    double packetsPerSecondIn() const;
    double packetsPerSecondOut() const;
    double messagesPerSecondIn() const;
    double messagesPerSecondOut() const;
    int   bandwidthFromServer() const;
    void  bandwidthFromServer( int bandwidth );
    double movementBytesPercent() const;
    double nonMovementBytesPercent() const;
    double overheadBytesPercent() const;
    int movementBytesTotal() const;
    int nonMovementBytesTotal() const;
    int overheadBytesTotal() const;
    uint32 packetsIn() const;

    void pTime( const double * pTime );
    const double * pTime()            { return pTime_; }
    double serverTime( double gameTime ) const;
    double lastMessageTime() const;
    GameTime lastGameTime() const;
    double lastSendTime() const      { return lastSendTime_; }
    double minSendInterval() const   { return minSendInterval_; }

    // ---- InterfaceMinder handlers ----
    void authenticate( const ClientInterface::authenticateArgs & args );
    void bandwidthNotification(
        const ClientInterface::bandwidthNotificationArgs & args );
    void updateFrequencyNotification(
        const ClientInterface::updateFrequencyNotificationArgs & args );
    void setGameTime( const ClientInterface::setGameTimeArgs & args );
    void resetEntities( const ClientInterface::resetEntitiesArgs & args );
    void createBasePlayer( BinaryIStream & stream );
    void createCellPlayer( BinaryIStream & stream );
    void spaceData( BinaryIStream & stream );
    void enterAoI( const ClientInterface::enterAoIArgs & args );
    void enterAoIOnVehicle(
        const ClientInterface::enterAoIOnVehicleArgs & args );
    void leaveAoI( BinaryIStream & stream );
    void createEntity( BinaryIStream & stream );
    void updateEntity( BinaryIStream & stream );

    // 通过宏生成所有 common_client_interface 消息的 handler 声明
#define MF_BEGIN_COMMON_RELIABLE_MSG( MESSAGE ) \
    void MESSAGE( const ClientInterface::MESSAGE##Args & args );
#define MF_BEGIN_COMMON_PASSENGER_MSG MF_BEGIN_COMMON_RELIABLE_MSG
#define MF_BEGIN_COMMON_UNRELIABLE_MSG MF_BEGIN_COMMON_RELIABLE_MSG
#define MF_COMMON_ARGS( ARGS )
#define MF_END_COMMON_MSG()
#define MF_COMMON_ISTREAM( NAME, XSTREAM )
#define MF_COMMON_OSTREAM( NAME, XSTREAM )
#include "common_client_interface.hpp"
#undef MF_BEGIN_COMMON_RELIABLE_MSG
    // ... 省略 undef

    void detailedPosition( const ClientInterface::detailedPositionArgs & args );
    void forcedPosition( const ClientInterface::forcedPositionArgs & args );
    void controlEntity( const ClientInterface::controlEntityArgs & args );
    void voiceData( const Mercury::Address & srcAddr, BinaryIStream & stream );
    void restoreClient( BinaryIStream & stream );
    void switchBaseApp( BinaryIStream & stream );
    void resourceHeader( BinaryIStream & stream );
    void resourceFragment( BinaryIStream & stream );
    void loggedOff( const ClientInterface::loggedOffArgs & args );
    void handleEntityMessage( int messageID, BinaryIStream & data );

    void setMessageHandler( ServerMessageHandler * pHandler )
    { if (pHandler_ != NULL) pHandler_ = pHandler; }

    Mercury::NetworkInterface & networkInterface() { return *pInterface_; }
    Mercury::EventDispatcher & dispatcher() { return dispatcher_; };

    void digest( const MD5::Digest & digest ) { digest_ = digest; }
    const MD5::Digest digest() const          { return digest_; }

    void send();
    void addCondemnedInterface( Mercury::NetworkInterface * pInterface );

    static const float & updateFrequency() { return s_updateFrequency_; }

private:
    typedef StatWithRatesOfChange< uint32 > Stat;

    double appTime() const;
    void updateStats();
    void onTimeOut( Mercury::Channel * pChannel );
    void setVehicle( EntityID passengerID, EntityID vehicleID );
    EntityID getVehicleID( EntityID passengerID ) const;
    void initialiseConnectionState();
    virtual void primeBundle( Mercury::Bundle & bundle );
    virtual int numUnreliableMessages() const;
    virtual void handleTimeout( TimerHandle handle, void * arg );
    double getRatePercent( const Stat & stat ) const;
    bool isControlledLocally( EntityID id ) const;
    void detailedPositionReceived( EntityID id, SpaceID spaceID,
        EntityID vehicleID, const Vector3 & position );

    // ---- 数据成员 ----
    SessionKey      sessionKey_;
    std::string     username_;

    Stat numMovementBytes_;
    Stat numNonMovementBytes_;
    Stat numOverheadBytes_;

    ServerMessageHandler * pHandler_;

    EntityID    id_;
    SpaceID     spaceID_;
    int         bandwidthFromServer_;

    const double * pTime_;
    uint64  lastReceiveTime_;
    double  lastSendTime_;
    double  minSendInterval_;
    double  sendTimeReportThreshold_;

    Mercury::EventDispatcher     dispatcher_;
    Mercury::NetworkInterface *  pInterface_;
    Mercury::Channel *           pChannel_;

    bool    tryToReconfigurePorts_;
    bool    entitiesEnabled_;

    float               inactivityTimeout_;
    MD5::Digest         digest_;

    class ServerTimeHandler
    {
    public:
        ServerTimeHandler();
        void tickSync( uint8 newSeqNum, double currentTime );
        void gameTime( GameTime gameTime, double currentTime );
        double  serverTime( double gameTime ) const;
        double  lastMessageTime() const;
        GameTime lastGameTime() const;
    private:
        static const double UNINITIALISED_TIME;
        uint8 tickByte_;
        double timeAtSequenceStart_;
        GameTime gameTimeAtSequenceStart_;
    } serverTimeHandler_;

    std::string errorMsg_;
    uint8   sendingSequenceNumber_;
    EntityID idAlias_[ 256 ];

    typedef std::map< EntityID, EntityID > PassengerToVehicleMap;
    PassengerToVehicleMap passengerToVehicle_;
    Vector3     sentPositions_[ 256 ];
    Vector3     referencePosition_;

    typedef std::set< EntityID > ControlledEntities;
    ControlledEntities controlledEntities_;

    typedef std::map< uint16, DataDownload* > DataDownloadMap;
    DataDownloadMap dataDownloads_;

    Mercury::EncryptionFilterPtr pFilter_;
    StreamEncoder * pLogOnParamsEncoder_;
    TimerHandle timerHandle_;

    typedef std::vector< Mercury::NetworkInterface * > CondemnedInterfaces;
    CondemnedInterfaces condemnedInterfaces_;

    const int FIRST_AVATAR_UPDATE_MESSAGE;
    const int LAST_AVATAR_UPDATE_MESSAGE;

    static float s_updateFrequency_;
};
```

### 3.1 构造与初始化

构造函数（见 [server_connection.cpp](file:///workspace/src/lib/connection/server_connection.cpp)）创建内部 `EventDispatcher` 与 `NetworkInterface`，注册定时器，初始化统计：

```cpp
const float DEFAULT_INACTIVITY_TIMEOUT = 60.f;       // 60 秒不活跃断开
const float UPDATE_STATS_PERIOD = 1.f;               // 每秒更新统计
const int PACKET_DELTA_WARNING_THRESHOLD = 3000;     // 包间隔超 3 秒告警

float ServerConnection::s_updateFrequency_ = 10.f;   // 默认 10 Hz 服务端更新

ServerConnection::ServerConnection() :
    sessionKey_( 0 ),
    pHandler_( NULL ),
    id_( EntityID( -1 ) ),
    spaceID_( 0 ),
    bandwidthFromServer_( 0 ),
    pTime_( NULL ),
    lastReceiveTime_( 0 ),
    lastSendTime_( 0.0 ),
    minSendInterval_( 1.01/20.0 ),                    // 最小发送间隔 ~50ms
    sendTimeReportThreshold_( 10.0 ),
    dispatcher_(),
    pInterface_( NULL ),
    pChannel_( NULL ),
    tryToReconfigurePorts_( false ),
    entitiesEnabled_( false ),
    inactivityTimeout_( DEFAULT_INACTIVITY_TIMEOUT ),
#ifdef USE_OPENSSL
    pFilter_( new Mercury::EncryptionFilter() ),      // 默认创建 Blowfish filter
#else
    pFilter_( NULL ),
#endif
    pLogOnParamsEncoder_( NULL ),
    FIRST_AVATAR_UPDATE_MESSAGE(
        ClientInterface::avatarUpdateNoAliasFullPosYawPitchRoll.id() ),
    LAST_AVATAR_UPDATE_MESSAGE(
        ClientInterface::avatarUpdateAliasNoPosNoDir.id() )
{
    this->pInterface( new Mercury::NetworkInterface( &dispatcher_,
                      Mercury::NETWORK_INTERFACE_EXTERNAL ) ),

    // 统计更新定时器
    timerHandle_ = this->dispatcher().addTimer(
                    static_cast< int >( UPDATE_STATS_PERIOD * 1000000 ), this );

    numMovementBytes_.monitorRateOfChange( 10 );      // 10 秒采样
    numNonMovementBytes_.monitorRateOfChange( 10 );
    numOverheadBytes_.monitorRateOfChange( 10 );

    this->initialiseConnectionState();
    memset( &digest_, 0, sizeof( digest_ ) );
}
```

关键点：

- 默认创建 `EXTERNAL` 类型的 `NetworkInterface`（客户端到服务端，高延迟低带宽）。
- `USE_OPENSSL` 时默认创建 `EncryptionFilter`（Blowfish），登录后用它加密 channel。
- `minSendInterval_ = 1.01/20.0` ≈ 50ms，限制客户端上行频率不超过 20Hz。
- `FIRST/LAST_AVATAR_UPDATE_MESSAGE` 标记 avatarUpdate 消息 ID 范围，用于区分移动字节与非移动字节统计。
- `ServerConnection` 实现了 `pChannelTimeOutHandler(this)`，channel 超时时回调 `onTimeOut`。

### 3.2 接口注册

`registerInterfaces` 把 `ClientInterface` 注册到网络接口，并注册 128-255 的 entity 消息 handler：

```cpp
void ServerConnection::registerInterfaces(
        Mercury::NetworkInterface & networkInterface )
{
    ClientInterface::registerWithInterface( networkInterface );

    for (Mercury::MessageID id = 128; id != 255; id++)
    {
        networkInterface.interfaceTable().serve( ClientInterface::entityMessage,
            id, &g_entityMessageHandler );
    }

    // 把 this 挂到 interface 的扩展数据上，handler 据此找到 ServerConnection
    networkInterface.pExtensionData( this );
}
```

`g_entityMessageHandler` 是全局 `EntityMessageHandler` 实例（见 [message_handlers.hpp](file:///workspace/src/lib/connection/message_handlers.hpp)），它通过 `header.pInterface->pExtensionData()` 取回 `ServerConnection*`，再调 `handleEntityMessage`。这种设计支持多 bot 场景（每个 bot 一个 `ServerConnection`，各自挂在对应 interface 上）。

### 3.3 在线状态与断开

```cpp
bool ServerConnection::online() const
{
    return pChannel_ != NULL;
}

void ServerConnection::disconnect( bool informServer )
{
    if (!this->online()) return;

    if (informServer)
    {
        BaseAppExtInterface::disconnectClientArgs::start( this->bundle(),
            Mercury::RELIABLE_NO ).reason = 0;
        this->channel().send();
    }

    if (pChannel_ != NULL)
    {
        pChannel_->destroy();
        pChannel_ = NULL;
    }

    // 清理进行中的数据下载
    for (uint i = 0; i < dataDownloads_.size(); ++i)
        delete dataDownloads_[i];
    dataDownloads_.clear();

    pHandler_ = NULL;
    sessionKey_ = 0;

    // 重新绑定 socket 换新端口（避免端口被服务端记忆）
    this->networkInterface().recreateListeningSocket( 0, NULL );
}
```

`disconnect` 先发 `disconnectClient` 通知服务端（不可靠，发完即走），再 `destroy` channel，清理下载，重新绑定 socket 端口。析构函数会先 `disconnect`，再 `destroy` channel，最后 `delete pInterface_`。

### 3.4 上行 API

`addMove` 上报受控实体位置（仅对 `controlledEntities_` 中的 entity）：

```cpp
void ServerConnection::addMove( EntityID id, SpaceID spaceID,
    EntityID vehicleID, const Vector3 & pos, float yaw, float pitch, float roll,
    bool onGround, const Vector3 & globalPos )
{
    if (this->offline()) return;
    if (spaceID != spaceID_) { /* ERROR */ return; }
    if (!this->isControlledLocally( id )) { /* ERROR */ return; }
    // ... 实际打包 avatarUpdate 消息
}
```

`startProxyMessage` / `startAvatarMessage` / `startEntityMessage` 在 bundle 上开始对应类型的消息。`requestEntityUpdate` 请求服务端补发某 entity 的属性（带 `CacheStamps` 告知已收到的版本）。

### 3.5 BundlePrimer —— bundle 预填

`ServerConnection` 继承 `BundlePrimer`，每次 channel 的 bundle 清空后会调 `primeBundle` 注入待发的 avatarUpdate 消息（玩家位置上报）。`numUnreliableMessages` 返回待发的不可靠消息数。这让上层只需调 `addMove` 累积位置，下次 `send` 时自动带出，无需显式管理 bundle。

### 3.6 统计与时间同步

`ServerTimeHandler` 是内部嵌套类，用 `tickSync` 消息（`tickByte` 序号）与 `setGameTime` 消息同步服务端 game time：

```cpp
class ServerTimeHandler
{
public:
    void tickSync( uint8 newSeqNum, double currentTime );
    void gameTime( GameTime gameTime, double currentTime );
    double  serverTime( double gameTime ) const;
private:
    static const double UNINITIALISED_TIME;
    uint8 tickByte_;
    double timeAtSequenceStart_;
    GameTime gameTimeAtSequenceStart_;
};
```

`serverTime(gameTime)` 把服务端 game tick 转成客户端本地时间，用于预测与插值。

带宽统计用 `StatWithRatesOfChange<uint32>`（来自 math 库），区分三类字节：

- `numMovementBytes_`：avatarUpdate 类消息（位置上报）
- `numNonMovementBytes_`：其它业务消息
- `numOverheadBytes_`：协议开销（packet header、ACK 等）

每秒采样一次，10 秒滑动窗口，`movementBytesPercent()` 等返回占比，上报给服务端做限速决策。

---

## 4. 登录流程

登录是连接层最复杂的部分，涉及 RSA 加密握手、两阶段登录（loginapp → baseapp）、多 socket 竞速。

### 4.1 LoginHandler —— 登录状态机

定义于 [login_handler.hpp](file:///workspace/src/lib/connection/login_handler.hpp)，继承 `SafeReferenceCount`，管理登录的各个阶段：

```cpp
class LoginHandler : public SafeReferenceCount
{
public:
    LoginHandler( ServerConnection * pServerConnection,
                  LogOnStatus loginNotSent = LogOnStatus::NOT_SET );
    ~LoginHandler();

    void start( const Mercury::Address & loginAppAddr, LogOnParamsPtr pParams );
    void startWithBaseAddr( const Mercury::Address & baseAppAddr,
                            SessionKey loginKey );
    void finish();

    void sendLoginAppLogin();
    void onLoginReply( BinaryIStream & data );

    void sendBaseAppLogin();
    void onBaseAppReply( BaseAppLoginRequestPtr pHandler,
        SessionKey sessionKey );

    void onFailure( Mercury::Reason reason );

    const LoginReplyRecord & replyRecord() const { return replyRecord_; }
    bool done() const                       { return done_; }
    int status() const                      { return status_; }
    LogOnParamsPtr pParams()                { return pParams_; }
    const std::string & errorMsg() const    { return errorMsg_; }

    void setError( int status, const std::string & errorMsg )
    {
        status_ = status;
        errorMsg_ = errorMsg;
    }

    ServerConnection * pServerConnection() const { return pServerConnection_; }
    const Mercury::Address & loginAddr() const { return loginAppAddr_; }
    const Mercury::Address & baseAppAddr() const { return baseAppAddr_; }

    void addChildRequest( RetryingRequestPtr pRequest );
    void removeChildRequest( RetryingRequestPtr pRequest );
    void addCondemnedInterface( Mercury::NetworkInterface * pInterface );
    int numBaseAppLoginAttempts() const { return numBaseAppLoginAttempts_; }

private:
    Mercury::Address  loginAppAddr_;
    Mercury::Address  baseAppAddr_;
    LogOnParamsPtr    pParams_;
    ServerConnection* pServerConnection_;
    LoginReplyRecord  replyRecord_;
    bool              done_;
    uint8             status_;
    std::string       errorMsg_;
    int numBaseAppLoginAttempts_;
    typedef std::set< RetryingRequestPtr > ChildRequests;
    ChildRequests childRequests_;
};

typedef SmartPointer< LoginHandler > LoginHandlerPtr;
```

`childRequests_` 跟踪所有活跃的 `RetryingRequest`（登录请求与 baseapp 登录请求），`finish` 时全部 cancel。`numBaseAppLoginAttempts_` 限制 baseapp 登录尝试次数（最多 10 次，见 `MAX_BASEAPP_LOGIN_ATTEMPTS`）。

### 4.2 LogOnParams —— 登录参数

定义于 [log_on_params.hpp](file:///workspace/src/lib/connection/log_on_params.hpp)，封装登录所需的用户名、密码、加密密钥、digest：

```cpp
class LogOnParams : public SafeReferenceCount
{
public:
    typedef uint8 Flags;
    static const Flags HAS_DIGEST = 0x1;
    static const Flags HAS_ALL = 0x1;
    static const Flags PASS_THRU = 0xFF;

    LogOnParams() : flags_( HAS_ALL )
    {
        nonce_ = std::rand();
        digest_.clear();
    }

    LogOnParams( const std::string & username, const std::string & password,
        const std::string & encryptionKey ) :
        flags_( HAS_ALL ),
        username_( username ),
        password_( password ),
        encryptionKey_( encryptionKey )
    {
        nonce_ = std::rand();
        digest_.clear();
    }

    bool addToStream( BinaryOStream & data, Flags flags = PASS_THRU,
        const StreamEncoder * pEncoder = NULL ) const;
    bool readFromStream( BinaryIStream & data,
        const StreamEncoder * pEncoder = NULL );

    const std::string & username() const { return username_; }
    const std::string & password() const { return password_; }
    const std::string & encryptionKey() const { return encryptionKey_; }
    const MD5::Digest & digest() const { return digest_; }
    void digest( const MD5::Digest & digest ){ digest_ = digest; }

    bool operator==( const LogOnParams & other )
    {
        return username_ == other.username_ &&
            password_ == other.password_ &&
            encryptionKey_ == other.encryptionKey_ &&
            nonce_ == other.nonce_;
    }

private:
    Flags       flags_;
    std::string username_;
    std::string password_;
    std::string encryptionKey_;   // 客户端 Blowfish 密钥，发给服务端用于会话加密
    uint32      nonce_;           // 随机数，防重放
    MD5::Digest digest_;          // 客户端资源 digest，服务端校验版本
};
```

`addToStream` 支持可选加密（[log_on_params.cpp](file:///workspace/src/lib/connection/log_on_params.cpp)）：

```cpp
bool LogOnParams::addToStream( BinaryOStream & data, Flags flags,
        const StreamEncoder * pEncoder ) const
{
    if (pEncoder)
    {
        MemoryOStream clearText;
        this->addToStreamInternal( clearText, flags );
        return pEncoder->encrypt( clearText, data );   // RSA 加密
    }
    else
    {
        this->addToStreamInternal( data, flags );
        return true;
    }
}

void LogOnParams::addToStreamInternal( BinaryOStream & data, Flags flags ) const
{
    if (flags == PASS_THRU) flags = flags_;
    data << flags << username_ << password_ << encryptionKey_;
    if (flags & HAS_DIGEST) data << digest_;
    data << nonce_;
}
```

明文流是 `flags + username + password + encryptionKey + [digest] + nonce`。`encryptionKey_` 是客户端 `EncryptionFilter` 的 Blowfish 密钥，登录后服务端用它加密下发数据。`nonce_` 防重放，`digest_` 是客户端资源 MD5，服务端校验客户端版本一致性。

### 4.3 LogOnStatus —— 状态码

定义于 [log_on_status.hpp](file:///workspace/src/lib/connection/log_on_status.hpp)，包含客户端与服务端两套状态码：

```cpp
class LogOnStatus
{
public:
    enum Status
    {
        // 客户端状态（< 64）
        NOT_SET,
        LOGGED_ON,
        CONNECTION_FAILED,
        DNS_LOOKUP_FAILED,
        UNKNOWN_ERROR,
        CANCELLED,
        ALREADY_ONLINE_LOCALLY,
        PUBLIC_KEY_LOOKUP_FAILED,
        LAST_CLIENT_SIDE_VALUE = 63,

        // 服务端状态（>= 64）
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
        LOGIN_REJECTED_UPDATER_NOT_READY,
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

    bool succeeded() const { return status_ == LOGGED_ON; }
    bool fatal() const;     // CONNECTION_FAILED / CANCELLED / UNKNOWN_ERROR
    bool okay() const;      // NOT_SET 或 LOGGED_ON
};
```

### 4.4 LoginReplyRecord —— 登录回复

定义于 [login_reply_record.hpp](file:///workspace/src/lib/connection/login_reply_record.hpp)，是登录成功后服务端返回的记录：

```cpp
struct LoginReplyRecord
{
    Mercury::Address  serverAddr;     // baseapp 地址
    uint32            sessionKey;     // 会话密钥
};

inline BinaryIStream& operator>>( BinaryIStream &is, LoginReplyRecord &lrr )
{
    return is >> lrr.serverAddr >> lrr.sessionKey;
}
```

loginapp 登录成功后，`LoginReplyRecord` 用客户端的 Blowfish 密钥对称加密后下发（见 `LoginHandler::onLoginReply`），包含 baseapp 地址与初始 sessionKey。

### 4.5 RetryingRequest —— 重试请求基类

定义于 [retrying_request.hpp](file:///workspace/src/lib/connection/retrying_request.hpp)，提供「客户端推送式可靠性」——不断重发请求直到收到回复。继承 `ShutdownSafeReplyMessageHandler`、`TimerHandler`、`SafeReferenceCount`：

```cpp
class RetryingRequest :
    public Mercury::ShutdownSafeReplyMessageHandler,
    public TimerHandler,
    public SafeReferenceCount
{
public:
    static const int DEFAULT_RETRY_PERIOD = 1000000;    // 1 秒重试
    static const int DEFAULT_TIMEOUT_PERIOD = 8000000;  // 8 秒超时
    static const int DEFAULT_MAX_ATTEMPTS = 10;         // 最多 10 次

    RetryingRequest( LoginHandler & parent,
        const Mercury::Address & addr,
        const Mercury::InterfaceElement & ie,
        int retryPeriod = DEFAULT_RETRY_PERIOD,
        int timeoutPeriod = DEFAULT_TIMEOUT_PERIOD,
        int maxAttempts = DEFAULT_MAX_ATTEMPTS,
        bool useParentInterface = true );

    virtual void handleMessage( const Mercury::Address & srcAddr,
        Mercury::UnpackedMessageHeader & header,
        BinaryIStream & data, void * arg );
    virtual void handleException( const Mercury::NubException & exc, void * arg );
    virtual void handleTimeout( TimerHandle handle, void * arg );

    virtual void addRequestArgs( Mercury::Bundle & bundle ) = 0;
    virtual void onSuccess( BinaryIStream & data ) = 0;
    virtual void onFailure( Mercury::Reason reason ) {}
    void cancel();

protected:
    void setInterface( Mercury::NetworkInterface * pInterface );
    void send();
    Mercury::EventDispatcher & dispatcher();

    SmartPointer< LoginHandler > pParent_;
    Mercury::NetworkInterface * pInterface_;
    const Mercury::Address addr_;
    const Mercury::InterfaceElement & ie_;
    TimerHandle timerHandle_;
    bool done_;

private:
    int retryPeriod_;
    int timeoutPeriod_;
    int numAttempts_;
    int numOutstandingAttempts_;
    int maxAttempts_;
};
```

子类只需实现 `addRequestArgs`（流式写请求参数）与 `onSuccess`（收到回复回调）。`handleMessage` 收到回复后调 `onSuccess` 并自我销毁；`handleTimeout` 重试或超时失败。注释：「This object deletes itself」——成功或失败后自销毁。

### 4.6 LoginAppLoginRequest —— 登录 loginapp

定义于 [loginapp_login_request.hpp](file:///workspace/src/lib/connection/loginapp_login_request.hpp) 与 [.cpp](file:///workspace/src/lib/connection/loginapp_login_request.cpp)：

```cpp
class LoginAppLoginRequest : public RetryingRequest
{
public:
    LoginAppLoginRequest( LoginHandler & parent );
    virtual void addRequestArgs( Mercury::Bundle & bundle );
    virtual void onSuccess( BinaryIStream & data );
    virtual void onFailure( Mercury::Reason reason );
};
```

构造即发送（`this->send()`）：

```cpp
LoginAppLoginRequest::LoginAppLoginRequest( LoginHandler & parent ) :
    RetryingRequest( parent, parent.loginAddr(), LoginInterface::login )
{
    this->send();
}

void LoginAppLoginRequest::addRequestArgs( Mercury::Bundle & bundle )
{
    LogOnParamsPtr pParams = pParent_->pParams();
    bundle << LOGIN_VERSION;   // 协议版本号（56）

    if (!pParams->addToStream( bundle, LogOnParams::HAS_ALL,
            pParent_->pServerConnection()->pLogOnParamsEncoder() ))
    {
        ERROR_MSG( "LoginAppLoginRequest::addRequestArgs: "
            "Failed to assemble login bundle\n" );
        pParent_->onFailure( Mercury::REASON_CORRUPTED_PACKET );
    }
}

void LoginAppLoginRequest::onSuccess( BinaryIStream & data )
{
    pParent_->onLoginReply( data );
}

void LoginAppLoginRequest::onFailure( Mercury::Reason reason )
{
    pParent_->onFailure( reason );
}
```

请求参数：`LOGIN_VERSION`（版本号，见 [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp)）+ `LogOnParams`（用 `RSAStreamEncoder` 加密）。`LOGIN_VERSION = 56`，与服务端 `OLDEST_SUPPORTED_CLIENT_LOGIN_VERSION = 56` 对应，版本不符会 `LOGIN_BAD_PROTOCOL_VERSION`。

### 4.7 BaseAppLoginRequest —— 登录 baseapp

定义于 [baseapp_login_request.hpp](file:///workspace/src/lib/connection/baseapp_login_request.hpp) 与 [.cpp](file:///workspace/src/lib/connection/baseapp_login_request.cpp)，比 loginapp 复杂——每个实例有自己的 `NetworkInterface` 与 `Channel`，多实例竞速，赢家把 channel 移交给 `ServerConnection`：

```cpp
const int DEFAULT_RETRY_PERIOD = 1000000;     // 1 秒
const int DEFAULT_TIMEOUT_PERIOD = 8000000;   // 8 秒

BaseAppLoginRequest::BaseAppLoginRequest( LoginHandler & parent ) :
    RetryingRequest( parent, parent.baseAppAddr(),
        BaseAppExtInterface::baseAppLogin,
        DEFAULT_RETRY_PERIOD,
        DEFAULT_TIMEOUT_PERIOD,
        /* maxAttempts: */ 1,                 // 注意：maxAttempts=1
        /* useParentNetworkInterface: */ false ),
    attempt_( parent.numBaseAppLoginAttempts() )
{
    ServerConnection * pServConn = parent.pServerConnection();

    // 每个实例自己的网络接口，应对中国多层 NAT 问题
    if (attempt_ == 0)
    {
        this->setInterface( &(pServConn->networkInterface()) );
        pServConn->pInterface( NULL );        // 借走主接口
    }
    else
    {
        Mercury::NetworkInterface * pInterface =
            new Mercury::NetworkInterface( &this->dispatcher(),
                Mercury::NETWORK_INTERFACE_EXTERNAL );
        this->setInterface( pInterface );

        // 临时接口也要注册所有接口，因为回复可能 piggyback 在其它消息上
        pServConn->registerInterfaces( *pInterface_ );
    }

    pChannel_ = new Mercury::Channel(
        *pInterface_, parent.baseAppAddr(),
        Mercury::Channel::EXTERNAL,
        /* minInactivityResendDelay: */ 1.0,
        pServConn->pFilter().get() );

    // 在收到 cellPlayerCreate 前，channel 是非常规的
    pChannel_->isLocalRegular( false );
    pChannel_->isRemoteRegular( false );
    this->send();
}

BaseAppLoginRequest::~BaseAppLoginRequest()
{
    // 赢家的 pChannel_ 已转交，为 NULL；输家清理自己的接口
    if (pChannel_)
    {
        pChannel_->destroy();
        pChannel_ = NULL;
        pParent_->addCondemnedInterface( pInterface_ );
        pInterface_ = NULL;
    }
}

void BaseAppLoginRequest::handleTimeout( TimerHandle handle, void * arg )
{
    // 每个请求只 spawn 一个新请求（maxAttempts=1，靠 timeout 再生）
    timerHandle_.cancel();
    if (!done_)
    {
        pParent_->sendBaseAppLogin();   // 创建新实例
    }
}

void BaseAppLoginRequest::addRequestArgs( Mercury::Bundle & bundle )
{
    // 发 loginKey 和尝试次数（调试用）
    bundle << pParent_->replyRecord().sessionKey << attempt_;
}

void BaseAppLoginRequest::onSuccess( BinaryIStream & data )
{
    SessionKey sessionKey = 0;
    data >> sessionKey;
    pParent_->onBaseAppReply( this, sessionKey );

    // channel 已转交，置 NULL 避免析构销毁
    pChannel_ = NULL;
}
```

关键设计：

1. **多 socket 竞速**：`attempt_ == 0` 用主接口，之后每个超时实例 spawn 新实例用新接口（新 socket 端口）。这应对 NAT/防火墙问题——不同端口可能命中不同的 NAT 映射，增加连通概率。注释明确提到「strange multi-level NATing issues in China」。
2. **maxAttempts=1 + 再生**：每个实例只发一次，超时后 `handleTimeout` 调 `sendBaseAppLogin` 创建新实例，而不是自身重试。这样每个实例用独立 socket。
3. **临时接口也注册接口**：因为 baseapp 回复可能 piggyback 在带其它 `ClientInterface` 消息的包上，临时接口必须知道如何处理它们。
4. **channel 转交**：赢家在 `onSuccess` 把 `pChannel_` 置 NULL（避免析构销毁），`LoginHandler::onBaseAppReply` 把 channel 与 interface 移交给 `ServerConnection`。输家在析构时 `destroy` channel 并把接口加入 `condemnedInterfaces_` 异步清理。
5. **isLocalRegular=false**：登录阶段 channel 非常规，靠 `IrregularChannels` 主动重传。

`LoginHandler::onBaseAppReply` 完成转交（见 [login_handler.cpp](file:///workspace/src/lib/connection/login_handler.cpp)）：

```cpp
void LoginHandler::onBaseAppReply( BaseAppLoginRequestPtr pHandler,
    SessionKey sessionKey )
{
    pServerConnection_->pInterface( &pHandler->networkInterface() );
    pServerConnection_->channel( pHandler->channel() );

    replyRecord_.sessionKey = sessionKey;
    pServerConnection_->sessionKey( sessionKey );

    // 把 ServerConnection 设为 bundle primer
    pServerConnection_->channel().bundlePrimer( *this->pServerConnection() );

    this->finish();
}
```

### 4.8 完整登录时序

```
   客户端                        LoginApp              BaseAppMgr         BaseApp
   ─────                         ──────                ──────────         ───────

1. ServerConnection::logOnBegin(server, user, pwd)
   ├─ 创建 LogOnParams(user, pwd, blowfishKey)
   ├─ pParams->digest( 客户端资源 MD5 )
   ├─ registerInterfaces(主 NetworkInterface)
   ├─ DNS 解析 server → loginAddr
   └─ LoginHandler::start(loginAddr, pParams)
      └─ sendLoginAppLogin()
         └─ new LoginAppLoginRequest(*this)
            ├─ RetryingRequest 构造（用主接口）
            └─ send()  ───►  Bundle: LOGIN_VERSION + LogOnParams(RSA加密)
                                                  │
                                                  │ RSA 公钥加密
                                                  │（loginapp.pubkey）
                                                  ▼
2.                                              LoginApp 收到 login
                                                ├─ RSA 私钥解密 LogOnParams
                                                ├─ 校验版本/digest/账号
                                                ├─ 向 BaseAppMgr 请求 baseapp
                                                │   (选负载最低的 baseapp)
                                                │                       │
                                                │                       ▼
                                                │              BaseAppMgr 选 baseapp
                                                │              分配 sessionKey
                                                │              通知 BaseApp 创建 session
                                                │                       │
                                                │ ◄─────────────────────┘
                                                │
                                                └─ 回复 loginReply
                                                   ├─ status (LOGGED_ON)
                                                   └─ LoginReplyRecord
                                                      { baseAppAddr, sessionKey }
                                                      （用客户端 Blowfish 加密）
                                  ◄──────────  回复包
   ┌─────────────────────────────
   │ LoginAppLoginRequest::onSuccess(data)
   │ └─ LoginHandler::onLoginReply(data)
   │    ├─ data >> status_
   │    ├─ if LOGGED_ON:
   │    │   ├─ Blowfish 解密 replyRecord
   │    │   ├─ baseAppAddr_ = replyRecord.serverAddr
   │    │   └─ sendBaseAppLogin()  ── 多实例竞速开始
   │    └─ else: 读 errorMsg_, finish()
   │
3. │ sendBaseAppLogin()  (最多 MAX_BASEAPP_LOGIN_ATTEMPTS=10 次)
   │ └─ new BaseAppLoginRequest(*this)
   │    ├─ attempt 0: 用主接口（借走 pInterface_）
   │    ├─ attempt 1..9: 新建 NetworkInterface + 新 socket 端口
   │    ├─ new Channel(EXTERNAL, baseAppAddr, blowfishFilter)
   │    ├─ isLocalRegular(false) / isRemoteRegular(false)
   │    └─ send()  ───►  Bundle: baseAppLogin(sessionKey, attempt#)
   │                                              │
   │                                              ▼
   │                                        BaseApp 收到 baseAppLogin
   │                                        ├─ 校验 sessionKey
   │                                        ├─ 创建 EntityChannel
   │                                        └─ 回复 baseAppLoginReply
   │                                           { 新 sessionKey }
   │                                  ◄──────  回复包（可能多个，赢家只有一个）
   │
   │   ┌─ 第一个回复者赢 ─────────────────────
   │   │ BaseAppLoginRequest::onSuccess(data)
   │   │ ├─ data >> sessionKey
   │   │ ├─ pParent_->onBaseAppReply(this, sessionKey)
   │   │ │   ├─ ServerConnection::pInterface(赢家的接口)
   │   │ │   ├─ ServerConnection::channel(赢家的 channel)
   │   │ │   ├─ sessionKey_ = sessionKey
   │   │ │   ├─ channel.bundlePrimer(ServerConnection)
   │   │ │   └─ LoginHandler::finish()
   │   │ │       ├─ cancel 所有 childRequests（输家的请求）
   │   │ │       ├─ breakProcessing() 退出 logOn 循环
   │   │ │       └─ done_ = true
   │   │ └─ pChannel_ = NULL  (避免析构销毁)
   │   │
   │   └─ 输家：~BaseAppLoginRequest
   │       ├─ pChannel_->destroy()
   │       └─ addCondemnedInterface(pInterface_)  异步清理
   │ └──────────────────────────────────────
   │
4. │ logOnComplete(pLoginHandler, pHandler)
   │ ├─ status = LOGGED_ON
   │ ├─ 发初始包给 baseapp 开 NAT 洞
   │ ├─ pHandler_ = pHandler (上层 handler)
   │ └─ channel().startInactivityDetection(60s)
   │
5. │ enableEntities()
   │ ├─ BaseAppExtInterface::enableEntitiesArgs::start(bundle, RELIABLE_DRIVER)
   │ ├─ args.dummy = 0
   │ └─ send()  ───►  通知 baseapp 开始下发 entity 数据
   │                                       │
   │                                       ▼
   │                                 BaseApp 开始下发：
   │                                 ├─ createBasePlayer
   │                                 ├─ createCellPlayer
   │                                 ├─ enterAoI / createEntity
   │                                 ├─ setGameTime / tickSync
   │                                 └─ spaceData / avatarUpdate ...
   │                                  ◄──────  后续业务消息流
   └────────────────────────────────────────────
```

### 4.9 加密握手细节

[rsa_stream_encoder.hpp](file:///workspace/src/lib/connection/rsa_stream_encoder.hpp) 是 `StreamEncoder` 的 RSA 实现：

```cpp
class RSAStreamEncoder : public StreamEncoder
{
public:
    RSAStreamEncoder( bool keyIsPrivate ) :
        StreamEncoder(), key_( keyIsPrivate ) {}

    bool initFromKeyString( const std::string & keyString )
    { return key_.setKey( keyString ); }

    bool initFromKeyPath( const std::string & keyPath )
    { return key_.setKeyFromResource( keyPath ); }

    virtual bool encrypt( BinaryIStream & clearText,
            BinaryOStream & cipherText ) const
    { return (key_.publicEncrypt( clearText, cipherText ) != -1); }

    virtual bool decrypt( BinaryIStream & cipherText,
            BinaryOStream & clearText ) const
    { return (key_.privateDecrypt( cipherText, clearText ) != -1); }

private:
    Mercury::PublicKeyCipher key_;
};
```

[stream_encoder.hpp](file:///workspace/src/lib/connection/stream_encoder.hpp) 是抽象接口：

```cpp
class StreamEncoder
{
public:
    virtual ~StreamEncoder() {}
    virtual bool encrypt( BinaryIStream & clearText,
            BinaryOStream & cipherText ) const = 0;
    virtual bool decrypt( BinaryIStream & cipherText,
            BinaryOStream & clearText ) const = 0;
};
```

[loginapp_public_key.hpp](file:///workspace/src/lib/connection/loginapp_public_key.hpp) 内嵌 loginapp 的 RSA 公钥（PS3/Xbox 用，PC 从资源加载）：

```cpp
#if defined( PLAYSTATION3 ) || defined( _XBOX360 )
char g_loginAppPublicKey[] = "-----BEGIN PUBLIC KEY-----\n"\
"MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA7/MNyWDdFpXhpFTO9LHz\n"
// ... 2048 位 RSA 公钥 ...
"-----END PUBLIC KEY-----\n";
#endif
```

客户端用此公钥 `RSAStreamEncoder` 加密 `LogOnParams`，loginapp 用私钥解密。Version 48 起「Public keys are no longer fetchable from the server」（不再从服务端拉公钥，避免公钥被替换）。登录成功后，会话用客户端生成的 Blowfish 密钥对称加密（`EncryptionFilter`）。

`ServerConnection::logOnBegin` 中设置 encoder 的位置（见 [server_connection.cpp](file:///workspace/src/lib/connection/server_connection.cpp)）：

```cpp
LoginHandlerPtr ServerConnection::logOnBegin(...)
{
    std::string key = pFilter_ ? pFilter_->key() : "";
    LogOnParamsPtr pParams = new LogOnParams( username, password, key );
    pParams->digest( this->digest() );
    // ...
    // pLogOnParamsEncoder_ 由上层设置（用 loginapp.pubkey 初始化的 RSAStreamEncoder）
}
```

`pFilter_->key()` 取 Blowfish 密钥塞进 `LogOnParams.encryptionKey_`，服务端 loginapp 解密后把此密钥传给 baseapp，baseapp 用它构造 `EncryptionFilter` 挂在 entity channel 上。从此 channel 流量双向 Blowfish 加密。

---

## 5. 接口定义

连接层定义了客户端与服务端通信的全部 Mercury 接口。

### 5.1 LoginInterface —— 登录接口

定义于 [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp)：

```cpp
const uint32 LOGIN_VERSION = 56;
const uint32 OLDEST_SUPPORTED_CLIENT_LOGIN_VERSION = 56;

#define PROBE_KEY_HOST_NAME         "hostName"
#define PROBE_KEY_OWNER_NAME        "ownerName"
#define PROBE_KEY_USERS_COUNT       "usersCount"
#define PROBE_KEY_UNIVERSE_NAME     "universeName"
#define PROBE_KEY_SPACE_NAME        "spaceName"
#define PROBE_KEY_BINARY_ID         "binaryID"

#pragma pack(push,1)
BEGIN_MERCURY_INTERFACE( LoginInterface )
    // uint32 version
    // bool encrypted
    // LogOnParams
    MERCURY_VARIABLE_MESSAGE( login, 2, &gLoginHandler )

    MERCURY_FIXED_MESSAGE( probe, 0, &gProbeHandler )
END_MERCURY_INTERFACE()
#pragma pack(pop)
```

只有两个消息：

- `login`：变长（长度字段 2 字节），由 loginapp 的 `gLoginHandler` 处理
- `probe`：固定长度 0，用于服务器探测（返回 hostName/ownerName 等信息）

文件顶部 56 条版本注释记录了协议演进史（如 Version 32 加 `baseAppLogin` 解决 NAT，Version 45 起 RSA+Blowfish，Version 49 Blowfish 加 XOR 防重放，Version 52 累积 ACK，Version 56 `createEntity` 可压缩）。

### 5.2 ClientInterface —— 客户端下行接口

定义于 [client_interface.hpp](file:///workspace/src/lib/connection/client_interface.hpp)，是 baseapp/cellapp 下发到客户端的消息集合。用一组辅助宏声明：

```cpp
#define MF_BEGIN_CLIENT_MSG( NAME )
    BEGIN_HANDLED_STRUCT_MESSAGE( NAME,
        ClientMessageHandler< ClientInterface::NAME##Args >,
        &ServerConnection::NAME )

#define MF_VARLEN_CLIENT_MSG( NAME )
    MERCURY_HANDLED_VARIABLE_MESSAGE( NAME, 2,
        ClientVarLenMessageHandler, &ServerConnection::NAME )

#define MF_VARLEN_WITH_ADDR_CLIENT_MSG( NAME )
    MERCURY_HANDLED_VARIABLE_MESSAGE( NAME, 2,
        ClientVarLenWithAddrMessageHandler, &ServerConnection::NAME )

#pragma pack(push, 1)
BEGIN_MERCURY_INTERFACE( ClientInterface )

    MF_BEGIN_CLIENT_MSG( authenticate )
        uint32  key;
    END_STRUCT_MESSAGE()

    MF_BEGIN_CLIENT_MSG( setGameTime )
        GameTime gameTime;
    END_STRUCT_MESSAGE()

    MF_VARLEN_CLIENT_MSG( createBasePlayer )
    MF_VARLEN_CLIENT_MSG( createCellPlayer )
    MF_VARLEN_CLIENT_MSG( spaceData )
    MF_VARLEN_CLIENT_MSG( createEntity )
    MF_VARLEN_CLIENT_MSG( updateEntity )

    MF_BEGIN_CLIENT_MSG( enterAoI )
        EntityID  id;
        IDAlias   idAlias;
    END_STRUCT_MESSAGE()

    MF_VARLEN_CLIENT_MSG( leaveAoI )

    // 128-254 是 entity 消息（runExposedMethod 等）
    // 通过 common_client_interface.hpp 包含共享消息
#define MF_BEGIN_COMMON_UNRELIABLE_MSG MF_BEGIN_CLIENT_MSG
#define MF_BEGIN_COMMON_PASSENGER_MSG MF_BEGIN_CLIENT_MSG
#define MF_BEGIN_COMMON_RELIABLE_MSG MF_BEGIN_CLIENT_MSG
#include "common_client_interface.hpp"
    // ... entityMessage 等
END_MERCURY_INTERFACE()
#pragma pack(pop)
```

`MF_BEGIN_CLIENT_MSG` 把消息绑定到 `ServerConnection` 的成员方法（如 `authenticate`、`setGameTime`、`enterAoI`）。`enterAoI` 带 `IDAlias`（1 字节别名），后续 entity 消息用别名代替完整 EntityID 省带宽。`IDAlias` 0-255 对应 `idAlias_` 数组。

### 5.3 BaseAppExtInterface —— 客户端上行接口

定义于 [baseapp_ext_interface.hpp](file:///workspace/src/lib/connection/baseapp_ext_interface.hpp)，客户端发给 baseapp 的消息：

```cpp
#pragma pack( push, 1 )
BEGIN_MERCURY_INTERFACE( BaseAppExtInterface )

    BW_STREAM_MSG_EX( BaseApp, baseAppLogin )

    BW_BEGIN_STRUCT_MSG_EX( BaseApp, authenticate )
        SessionKey  key;
    END_STRUCT_MESSAGE()

    MF_BEGIN_BLOCKABLE_PROXY_MSG( avatarUpdateImplicit )
        Coord       pos;
        YawPitchRoll dir;
        uint8       refNum;
    END_STRUCT_MESSAGE();

    MF_BEGIN_BLOCKABLE_PROXY_MSG( avatarUpdateExplicit )
        SpaceID     spaceID;
        EntityID    vehicleID;
        Coord       pos;
        YawPitchRoll dir;
        bool        onGround;
        uint8       refNum;
    END_STRUCT_MESSAGE();

    MF_BEGIN_BLOCKABLE_PROXY_MSG( ackPhysicsCorrection )
        uint8 dummy;
    END_STRUCT_MESSAGE()

    MERCURY_FIXED_MESSAGE( switchInterface, 0, NULL )

    MF_VARLEN_BLOCKABLE_PROXY_MSG( requestEntityUpdate )

    MF_BEGIN_BLOCKABLE_PROXY_MSG( enableEntities )
        uint8  dummy;
    END_STRUCT_MESSAGE();

    MF_BEGIN_UNBLOCKABLE_PROXY_MSG( restoreClientAck )
        int  id;
    END_STRUCT_MESSAGE()

    MF_BEGIN_UNBLOCKABLE_PROXY_MSG( disconnectClient )
        uint8 reason;
    END_STRUCT_MESSAGE()

    // 128-254 是 entity 消息（entityMessage）
    MERCURY_VARIABLE_MESSAGE( entityMessage, 2, NULL )

END_MERCURY_INTERFACE()
#pragma pack( pop )
```

关键消息：

- `baseAppLogin`：登录 baseapp（流式消息）
- `authenticate`：发送 sessionKey 认证
- `avatarUpdateImplicit/Explicit`：玩家位置上报（implicit 用相对参考位置，省带宽）
- `enableEntities`：通知 baseapp 准备好接收 entity 数据
- `disconnectClient`：主动断开
- `entityMessage`（128-254）：转发给 entity 的方法调用

`MF_BEGIN_BLOCKABLE_PROXY_MSG` 表示「可阻塞」消息（高优先级时可丢弃），`MF_BEGIN_UNBLOCKABLE_PROXY_MSG` 不可阻塞。

### 5.4 Common*Interface —— 共享接口

[common_client_interface.hpp](file:///workspace/src/lib/connection/common_client_interface.hpp) 定义 cell↔client 共享的消息（如 `tickSync`、`detailedPosition`、`forcedPosition`、`controlEntity`、各种 `avatarUpdate*`），同时被 `client_interface.hpp`（客户端侧）与服务端 `proxy_int_interface.hpp`（baseapp 侧）包含，通过宏切换实现「同一消息在两端不同 handler」。

[common_baseapp_interface.hpp](file:///workspace/src/lib/connection/common_baseapp_interface.hpp) 类似，定义 baseapp↔client 共享消息，被 `baseapp_ext_interface.hpp` 与服务端 `baseapp_int_interface.hpp` 共用。

### 5.5 MessageHandlers —— 消息 handler 模板

[message_handlers.hpp](file:///workspace/src/lib/connection/message_handlers.hpp) 定义几种 handler 模板：

```cpp
// 处理 entity 消息（128-254），转发到 ServerConnection::handleEntityMessage
class EntityMessageHandler : public Mercury::InputMessageHandler
{
protected:
    virtual void handleMessage( const Mercury::Address & /*srcAddr*/,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data )
    {
        ServerConnection * pServConn =
            (ServerConnection *)header.pInterface->pExtensionData();
        pServConn->handleEntityMessage( header.identifier & 0x7F, data );
    }
};

// 处理固定长度结构消息，分发到 ServerConnection 成员方法
template <class ARGS>
class ClientMessageHandler : public Mercury::InputMessageHandler
{
public:
    typedef void (ServerConnection::*Handler)( const ARGS & args );
    ClientMessageHandler( Handler handler ) : handler_( handler ) {}
private:
    virtual void handleMessage( const Mercury::Address &,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data )
    {
#ifndef _BIG_ENDIAN
        ARGS & args = *(ARGS*)data.retrieve( sizeof(ARGS) );   // 小端直接取指针
#else
        ARGS args;
        data >> args;                                            // 大端逐字段流
#endif
        ServerConnection * pServConn =
            (ServerConnection *)header.pInterface->pExtensionData();
        (pServConn->*handler_)( args );
    }
    Handler handler_;
};

// 处理变长消息
class ClientVarLenMessageHandler : public Mercury::InputMessageHandler
{
public:
    typedef void (ServerConnection::*Handler)( BinaryIStream & stream );
    ClientVarLenMessageHandler( Handler handler ) : handler_( handler ) {}
private:
    virtual void handleMessage( const Mercury::Address &,
        Mercury::UnpackedMessageHeader & header, BinaryIStream & data )
    {
        ServerConnection * pServConn =
            (ServerConnection *)header.pInterface->pExtensionData();
        (pServConn->*handler_)( data );
    }
    Handler handler_;
};

// 处理带源地址的变长消息（如 voiceData）
class ClientVarLenWithAddrMessageHandler : public Mercury::InputMessageHandler
{ /* 类似，但传 srcAddr */ };
```

设计要点：

- **pExtensionData 反查**：所有 handler 通过 `header.pInterface->pExtensionData()` 拿到 `ServerConnection*`，无需全局变量，支持多 bot。
- **小端优化**：小端平台直接 `data.retrieve(sizeof(ARGS))` 取结构指针，零拷贝；大端逐字段流式反序列化。
- **entity 消息特殊**：`identifier & 0x7F` 取低 7 位作为 entity 消息 ID（高 1 位可能是标志），转发到 `handleEntityMessage`，后者按 `entitydef` 反序列化并调 entity 方法。

### 5.6 ServerMessageHandler —— 上层回调接口

[server_message_handler.hpp](file:///workspace/src/lib/connection/server_message_handler.hpp) 定义纯虚接口，客户端业务层实现它接收服务端事件：

```cpp
class ServerMessageHandler
{
public:
    virtual void onBasePlayerCreate( EntityID id, EntityTypeID type,
        BinaryIStream & data ) = 0;
    virtual void onCellPlayerCreate( EntityID id,
        SpaceID spaceID, EntityID vehicleID, const Position3D & pos,
        float yaw, float pitch, float roll, BinaryIStream & data ) = 0;
    virtual void onEntityControl( EntityID id, bool control ) { }

    virtual void onEntityEnter( EntityID id, SpaceID spaceID,
        EntityID vehicleID ) = 0;
    virtual void onEntityLeave( EntityID id, const CacheStamps & stamps ) = 0;
    virtual void onEntityCreate( EntityID id, EntityTypeID type,
        SpaceID spaceID, EntityID vehicleID, const Position3D & pos,
        float yaw, float pitch, float roll, BinaryIStream & data ) = 0;
    virtual void onEntityProperties( EntityID id, BinaryIStream & data ) = 0;
    virtual void onEntityProperty( EntityID objectID, int messageID,
        BinaryIStream & data ) = 0;
    virtual void onEntityMethod( EntityID objectID, int messageID,
        BinaryIStream & data ) = 0;
    virtual void onEntityMove( EntityID id, SpaceID spaceID, EntityID vehicleID,
        const Position3D & pos, float yaw, float pitch, float roll,
        bool isVolatile ) {}
    virtual void onEntityMoveWithError( EntityID id, SpaceID spaceID,
        EntityID vehicleID, const Position3D & pos, const Vector3 & posError,
        float yaw, float pitch, float roll, bool isVolatile );

    virtual void spaceData( SpaceID spaceID, SpaceEntryID entryID,
        uint16 key, const std::string & data ) = 0;
    virtual void spaceGone( SpaceID spaceID ) = 0;
    virtual void onVoiceData( const Mercury::Address & srcAddr,
        BinaryIStream & data ) {}
    virtual void onStreamComplete( uint16 id, const std::string &desc,
        BinaryIStream & data ) {}
    virtual void onEntitiesReset( bool keepPlayerOnBase ) {}
    virtual void onRestoreClient( EntityID id, SpaceID spaceID,
        EntityID vehicleID, const Position3D & pos, const Direction3D & dir,
        BinaryIStream & data ) {}
    virtual void onEnableEntitiesRejected() {}
};
```

`ServerConnection` 收到 `ClientInterface` 消息后，反序列化并调 `pHandler_->onXxx`。例如 `createEntity` 收到后调 `onEntityCreate`，把 entity 数据流交给上层（按 `EntityDescription` 反序列化属性）。

---

## 6. 数据下载

服务端可向客户端推送大块数据（资源、配置等），通过分段消息实现可靠下载。

### 6.1 DownloadSegment

定义于 [download_segment.hpp](file:///workspace/src/lib/connection/download_segment.hpp)，单个数据段：

```cpp
class DownloadSegment
{
public:
    DownloadSegment( const char *data, int len, int seq ) :
        seq_( seq ), data_( data, len ) {}

    const char *data() { return data_.c_str(); }
    unsigned int size() { return data_.size(); }

    bool operator< (const DownloadSegment &other)
    { return seq_ < other.seq_; }

    uint8 seq_;
protected:
    std::string data_;
};
```

`seq_` 是段序号（uint8，0-255），用于乱序重组。

### 6.2 DataDownload

定义于 [data_download.hpp](file:///workspace/src/lib/connection/data_download.hpp)，继承 `std::list<DownloadSegment*>`，收集一个下载的所有段：

```cpp
class DataDownload : public std::list< DownloadSegment* >
{
public:
    DataDownload( uint16 id ) :
        id_( id ), pDesc_( NULL ), expected_( 0 ), hasLast_( false ) {}
    ~DataDownload();

    void insert( DownloadSegment *pSegment, bool isLast );
    bool complete();
    void write( BinaryOStream &os );
    uint16 id() const { return id_; }
    const std::string *pDesc() const { return pDesc_; }
    void setDesc( BinaryIStream &stream );

protected:
    int offset( int seq1, int seq2 );

    uint16 id_;                  // 下载 ID
    std::string *pDesc_;         // 描述
    std::set< int > holes_;      // 乱序产生的空洞
    uint8 expected_;             // 下一个期望的 seq
    bool hasLast_;               // 是否收到最后一段
};
```

`holes_` 记录乱序缺口，`expected_` 是下一个期望序号。`insert` 按 seq 插入并更新 holes，`complete()` 检查 `hasLast_ && holes_.empty()`。`ServerConnection::resourceHeader` / `resourceFragment` 收到消息后调 `dataDownloads_[id]->insert`，完成后调 `pHandler_->onStreamComplete` 把数据交给上层。

`ServerConnection` 用 `DataDownloadMap`（按 `uint16 id` 索引）管理多个并发下载。`disconnect` 时清理所有进行中的下载。

---

## 7. 与 entitydef 的协作

客户端收到的 entity 创建/属性更新消息，需按 `EntityDescription` 反序列化。这部分由 `ServerConnection::handleEntityMessage` 转交：

```cpp
void ServerConnection::handleEntityMessage( int messageID, BinaryIStream & data )
{
    // messageID 是低 7 位（0-127）
    // 上层（entitydef）按 EntityDescription 解析：
    //   - 高位判断是属性更新还是方法调用
    //   - 按 entity 类型查 EntityDescription
    //   - 反序列化参数
    //   - 调用 entity 脚本方法或设置属性
}
```

`EntityMessageHandler` 把 128-254 的消息（`entityMessage`）转发到这里，`messageID = header.identifier & 0x7F`。具体反序列化逻辑在 `entitydef` 模块（`EntityDescription::handleMessage` 等），连接层只做转发。

entity 消息格式：第一个字节是 EntityID 的 `IDAlias`（或 0 表示后跟完整 EntityID），之后是 messageID 与参数。`idAlias_` 数组维护 alias→EntityID 映射，`enterAoI` 时建立，`leaveAoI` 时清除。

`createEntity` 消息（变长）携带完整 entity 初始化数据流，`ServerConnection::createEntity` 反序列化后调 `pHandler_->onEntityCreate`，上层据此创建客户端 entity 实例并设置初始属性。Version 56 起 `createEntity` 数据可压缩（用 `CompressionIStream` 解压）。

---

## 8. 客户端登录完整时序（含密钥下发与 baseapp 切换）

下面是包含 RSA 握手、Blowfish 密钥下发、baseapp 切换的完整时序：

```
   客户端 (C)                LoginApp (L)           BaseAppMgr (M)        BaseApp (B)
   ────────                  ──────────             ──────────────        ─────────

[A. 准备]
   C: 生成 Blowfish 密钥 K_bf
   C: 加载 loginapp RSA 公钥 PK_login
   C: 构造 LogOnParams{user, pwd, K_bf, digest, nonce}

[B. LoginApp 登录（RSA 加密）]
   C ─── login(LOGIN_VERSION, RSA_PK{LogOnParams}) ───► L
                                                         L: 用 SK_login 解密
                                                         L: 校验账号/digest/版本
                                                         L: 向 M 请求 baseapp
                                                         L ─── assignBaseApp ──► M
                                                                                   M: 选负载最低的 B
                                                                                   M: 分配 sessionKey K_s1
                                                                                   M ─── createSession(K_s1) ──► B
                                                                                   M ◄── baseAppReady ────
                                                         L ◄── baseAppAddr, K_s1 ────
   C ◄── loginReply{ LOGGED_ON,                          L: 用 K_bf 加密 LoginReplyRecord
                      BF_K_bf{ baseAppAddr, K_s1 } }     (LoginReplyRecord 对称加密)
   C: 用 K_bf 解密得 baseAppAddr, K_s1

[C. BaseApp 登录（多 socket 竞速，建立 Blowfish 加密 channel）]
   C: attempt 0: 借主接口
   C: attempt 1..9: 新建接口（新端口）
   每个 attempt:
      C: new Channel(EXTERNAL, baseAppAddr, EncryptionFilter(K_bf))
      C ─── baseAppLogin(K_s1, attempt#) ───────────────────────────► B
                                                                       B: 校验 K_s1
                                                                       B: 创建 EntityChannel
                                                                       B: 准备新 sessionKey K_s2
       ┌─ 第一个回复者赢 ─────────────────────────────────────────────
   C ◄── baseAppLoginReply{ K_s2 } ─────────────────────────────── B
   C: pInterface = 赢家接口
   C: pChannel = 赢家 channel (带 K_bf 加密)
   C: sessionKey = K_s2
   C: channel.bundlePrimer(ServerConnection)
   C: 输家 channel.destroy() + 接口入 condemned

[D. 会话建立]
   C ─── 初始空包（开 NAT 洞） ───────────────────────────────────► B
   C: channel.startInactivityDetection(60s)
   C ─── enableEntities ───────────────────────────────────────────► B
                                                                       B: 开始下发 entity 数据

[E. 正常会话（双向 Blowfish 加密）]
   C ◄── createBasePlayer / createCellPlayer / setGameTime / tickSync ── B
   C ◄── enterAoI / createEntity / avatarUpdate / spaceData ─────────── B
   C ─── avatarUpdateImplicit / avatarUpdateExplicit ─────────────────► B
   C ─── entityMethod / requestEntityUpdate ──────────────────────────► B

[F. BaseApp 切换（故障转移）]
   旧 B 死亡，M 选新 B'
   C ◄── switchBaseApp{ newBaseAppAddr, K_s2' } ─── (经旧 channel 或 mgr 通知)
   C: 保存受控 entity 状态
   C: disconnect 旧 channel
   C: 用 K_bf 与新 B' 建立 channel
   C ─── restoreClientAck ──────────────────────────────────────────► B'
   C ◄── restoreClient{ entity 状态 } ─────────────────────────────── B'
   C: 恢复 entity，继续游戏
```

密钥体系总结：

| 密钥 | 类型 | 用途 | 生命周期 |
| --- | --- | --- | --- |
| `PK_login` / `SK_login` | RSA 公钥/私钥 | 加密 `LogOnParams` | 长期固定（编译进二进制或 loginapp 资源） |
| `K_bf` | Blowfish 对称密钥 | 加密 channel 流量 + 加密 `LoginReplyRecord` | 客户端生成，单次会话 |
| `K_s1` | sessionKey | loginapp→baseapp 登录用 | 登录阶段一次性 |
| `K_s2` | sessionKey | baseapp 会话认证（`authenticate` 消息） | 整个会话，baseapp 切换时更新 |
| `nonce` | 随机数 | 防重放攻击 | 每次登录重新生成 |
| `digest` | MD5 | 客户端资源版本校验 | 客户端资源决定 |

`switchBaseApp` 流程（`ServerConnection::switchBaseApp`）处理 baseapp 故障转移：服务端通过 `switchBaseApp` 消息通知新 baseapp 地址，客户端断开旧 channel、与新 baseapp 建立 channel、发 `restoreClientAck`，新 baseapp 下发 `restoreClient` 消息恢复 entity 状态。`restoreClient` 携带 entity 的位置、方向与持久化数据。

---

## 9. 关键设计总结

1. **两阶段登录**：loginapp（认证）→ baseapp（会话），分离认证与游戏逻辑，loginapp 无状态可水平扩展。
2. **RSA + Blowfish 混合加密**：RSA 加密登录参数（含 Blowfish 密钥），Blowfish 加密会话流量。RSA 慢但只需登录时用一次，Blowfish 快适合持续加密。
3. **多 socket 竞速**：baseapp 登录用多个 socket 同时尝试，应对 NAT/防火墙，第一个回复者赢。这是 BigWorld 针对中国复杂网络环境的特色设计。
4. **BundlePrimer 自动注入**：`ServerConnection` 作为 bundle primer，每次 bundle 清空自动注入待发 avatarUpdate，上层无需管理发送时机。
5. **IDAlias 压缩**：entity 进入 AoI 时分配 1 字节别名，后续消息用别名代替 4 字节 EntityID，省 75% 带宽。
6. **三类字节统计**：区分移动/非移动/开销字节，服务端可据此限速与调优。
7. **pExtensionData 反查**：消息 handler 通过 interface 的扩展数据找到 `ServerConnection`，支持多 bot。
8. **小端零拷贝**：小端平台直接取结构指针，避免拷贝；大端逐字段流式反序列化。
9. **RetryingRequest 自销毁**：请求对象成功或失败后自我销毁，简化生命周期管理。
10. **channel 转交**：baseapp 登录赢家把 channel 移交 `ServerConnection`，输家异步清理，避免 channel 泄漏。
11. **数据下载分段重组**：`DataDownload` 用 `holes_` 集合跟踪乱序缺口，支持可靠下载。
12. **baseapp 切换**：故障转移时通过 `switchBaseApp` + `restoreClient` 恢复会话，对玩家透明。

---

## 10. 阅读的核心文件清单

本文档阅读并引用了以下核心 `.hpp/.cpp/.ipp` 文件（共 22 个，超过要求的 15 个）：

| # | 文件 | 主要内容 |
| --- | --- | --- |
| 1 | [server_connection.hpp](file:///workspace/src/lib/connection/server_connection.hpp) | `ServerConnection` 会话核心类定义 |
| 2 | [server_connection.cpp](file:///workspace/src/lib/connection/server_connection.cpp) | 构造、登录、断开、接口注册实现 |
| 3 | [server_connection.ipp](file:///workspace/src/lib/connection/server_connection.ipp) | `setInactivityTimeout`/`pTime`/`appTime` 内联 |
| 4 | [login_handler.hpp](file:///workspace/src/lib/connection/login_handler.hpp) | `LoginHandler` 登录状态机 |
| 5 | [login_handler.cpp](file:///workspace/src/lib/connection/login_handler.cpp) | `start`/`onLoginReply`/`onBaseAppReply`/`finish` 实现 |
| 6 | [loginapp_login_request.hpp](file:///workspace/src/lib/connection/loginapp_login_request.hpp) | `LoginAppLoginRequest` 声明 |
| 7 | [loginapp_login_request.cpp](file:///workspace/src/lib/connection/loginapp_login_request.cpp) | `addRequestArgs`/`onSuccess` 实现 |
| 8 | [baseapp_login_request.hpp](file:///workspace/src/lib/connection/baseapp_login_request.hpp) | `BaseAppLoginRequest` 声明（含 channel） |
| 9 | [baseapp_login_request.cpp](file:///workspace/src/lib/connection/baseapp_login_request.cpp) | 多接口竞速、channel 转交实现 |
| 10 | [retrying_request.hpp](file:///workspace/src/lib/connection/retrying_request.hpp) | `RetryingRequest` 重试请求基类 |
| 11 | [log_on_params.hpp](file:///workspace/src/lib/connection/log_on_params.hpp) | `LogOnParams` 登录参数 |
| 12 | [log_on_params.cpp](file:///workspace/src/lib/connection/log_on_params.cpp) | `addToStream` 加密流式实现 |
| 13 | [log_on_status.hpp](file:///workspace/src/lib/connection/log_on_status.hpp) | `LogOnStatus` 状态码枚举 |
| 14 | [login_reply_record.hpp](file:///workspace/src/lib/connection/login_reply_record.hpp) | `LoginReplyRecord` 登录回复结构 |
| 15 | [rsa_stream_encoder.hpp](file:///workspace/src/lib/connection/rsa_stream_encoder.hpp) | `RSAStreamEncoder` RSA 加密器 |
| 16 | [stream_encoder.hpp](file:///workspace/src/lib/connection/stream_encoder.hpp) | `StreamEncoder` 加密器抽象接口 |
| 17 | [loginapp_public_key.hpp](file:///workspace/src/lib/connection/loginapp_public_key.hpp) | 内嵌 loginapp RSA 公钥 |
| 18 | [login_interface.hpp](file:///workspace/src/lib/connection/login_interface.hpp) | `LoginInterface` 登录接口 + LOGIN_VERSION |
| 19 | [client_interface.hpp](file:///workspace/src/lib/connection/client_interface.hpp) | `ClientInterface` 客户端下行接口 |
| 20 | [baseapp_ext_interface.hpp](file:///workspace/src/lib/connection/baseapp_ext_interface.hpp) | `BaseAppExtInterface` 客户端上行接口 |
| 21 | [common_client_interface.hpp](file:///workspace/src/lib/connection/common_client_interface.hpp) | cell↔client 共享消息 |
| 22 | [message_handlers.hpp](file:///workspace/src/lib/connection/message_handlers.hpp) | `EntityMessageHandler`/`ClientMessageHandler` 模板 |
| 23 | [server_message_handler.hpp](file:///workspace/src/lib/connection/server_message_handler.hpp) | `ServerMessageHandler` 上层回调接口 |
| 24 | [data_download.hpp](file:///workspace/src/lib/connection/data_download.hpp) | `DataDownload` 数据下载 |
| 25 | [download_segment.hpp](file:///workspace/src/lib/connection/download_segment.hpp) | `DownloadSegment` 下载段 |
| 26 | [cellapp_interface.hpp](file:///workspace/src/server/cellapp/cellapp_interface.hpp) | 服务端 IDL 实例（参考） |

---

## 总结

本文档约 13000 字，详细分析了 BigWorld 客户端连接层的架构与实现：

- **模块定位**：客户端会话层，建立在 Mercury 之上，封装登录/会话/实体分发/数据下载
- **ServerConnection**：会话核心，继承 `BundlePrimer`/`ChannelTimeOutHandler`/`TimerHandler`，持有 channel/interface/handler，提供 `logOn`/`disconnect`/`addMove`/`processInput` API
- **登录流程**：`LoginHandler` 状态机 + `LoginAppLoginRequest`（RSA 加密）+ `BaseAppLoginRequest`（多 socket 竞速、channel 转交）+ `RetryingRequest` 重试基类
- **加密握手**：`RSAStreamEncoder`（公钥加密 LogOnParams）+ `EncryptionFilter`（Blowfish 会话加密）+ `loginapp_public_key`（内嵌公钥）
- **接口定义**：`LoginInterface`/`ClientInterface`/`BaseAppExtInterface`/`Common*Interface`，用 Mercury IDL 宏声明
- **消息分发**：`EntityMessageHandler`/`ClientMessageHandler`/`ClientVarLenMessageHandler` 模板，通过 `pExtensionData` 反查 `ServerConnection`，小端零拷贝
- **数据下载**：`DataDownload` + `DownloadSegment`，`holes_` 集合跟踪乱序缺口
- **与 entitydef 协作**：`handleEntityMessage` 转发，按 `EntityDescription` 反序列化，`IDAlias` 压缩 EntityID
- **完整登录时序**：RSA 握手 → Blowfish 密钥下发 → 多 socket 竞速 → channel 转交 → enableEntities → baseapp 切换（故障转移）

连接层是客户端与服务端协作的纽带，其设计充分体现了 BigWorld 对 MMO 场景的深刻理解——两阶段登录分离认证与游戏、多 socket 竞速应对 NAT、混合加密平衡安全与性能、IDAlias 压缩省带宽、baseapp 切换实现故障透明转移。这些设计在当时的 MMO 后端中相当先进。
