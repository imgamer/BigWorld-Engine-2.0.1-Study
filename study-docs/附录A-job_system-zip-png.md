# 附录 A：job_system / zip / png

> 源码位置：
> - `/workspace/src/lib/job_system/`
> - `/workspace/src/lib/zip/`
> - `/workspace/src/lib/png/`
> 适用版本：BigWorld Technology 2.0.1

本附录聚焦 `job_system` 库的设计（CommandBuffer 双缓冲模式、SyncBlock/Job 链式调度、与 cstdmf `bgtask_manager` 的区别），并简要说明 `zip`（zlib）与 `png`（libpng）两个第三方库的用途与 BigWorld 改动。

---

## 1. job_system 概览

### 1.1 模块定位

`job_system` 是 BigWorld **客户端帧内并行计算系统**，专为 CPU 密集型任务（动画混合、蒙皮、粒子更新等）设计。它与 cstdmf 的 `bgtask_manager`（IO 密集型后台任务）形成互补：

| 维度 | `job_system` | `bgtask_manager`（cstdmf） |
|---|---|---|
| 任务类型 | CPU 密集（计算） | IO 密集（文件、解压） |
| 触发频率 | 每帧 | 偶发（资源加载） |
| 线程模型 | 固定 worker + 1 消费线程 | 通用线程池 |
| 通信方式 | `CommandBuffer` 双缓冲（无锁/极少锁） | `SimpleMutex` + `SimpleSemaphore` |
| 任务粒度 | 小（微秒~毫秒级） | 大（毫秒~秒级） |
| 数据流 | SyncBlock/Job 链式背补丁 | 任务自带数据 |
| 主线程参与 | 主线程生产 Job，消费线程消费 | 主线程收尾（`addMainThreadTask`） |
| 典型用户 | 客户端动画/物理/渲染预处理 | resmgr/chunk 资源加载 |

### 1.2 文件结构

```
job_system/
├── job_system.hpp      # Job/SyncBlock/JobSystem 主定义
├── job_system.cpp      # 实现
└── command_buffer.hpp  # CommandBuffer 双缓冲无锁队列
```

---

## 2. CommandBuffer —— 双缓冲命令缓冲

[command_buffer.hpp](file:///workspace/src/lib/job_system/command_buffer.hpp) 是 job_system 的核心数据结构：一个双缓冲、按需提交物理内存的命令缓冲区。

### 2.1 设计目标

- **生产者/消费者解耦**：主线程写一个缓冲，消费线程读另一个缓冲，交替进行。
- **无锁快路径**：写与读在不同缓冲上，多数操作无需加锁。
- **按需提交内存**：用 `VirtualAlloc` 预留大块地址空间，按需 `MEM_COMMIT` 提交物理页，避免一次性占用物理内存。
- **16 字节对齐**：所有写入按 16 字节对齐，便于 SSE 加载。
- **预取**：读操作后用 `_mm_prefetch` 预取下一段数据。

### 2.2 关键定义

```cpp
// command_buffer.hpp:77-254
class CommandBuffer
{
public:
    void init( uint maxSize );
    void fini();
    void reset();

    void nextRead();
    void nextWrite();

    uint getRemaining();
    void* getCurrent();
    void seek( uint bytes );
    void skipPadding();

    void* getCurrentWrite();
    void seekWrite( uint bytes );
    void writePadding();

    void writeRaw( const void* data, uint size );

    template <class A> void write( A a );
    template <class A, class B> void write( A a, B b );
    // ... 直至 8 个参数的重载

    template <class A> A read();

    static void memcpyNTA( void* dst, const void* src, uint size );

    static const uint32 NUM_BUFFERS = 2;
private:
    static const uint GROW_STEP = 1*1024*1024;

    char* buffer_[NUM_BUFFERS];
    uint curSize_[NUM_BUFFERS];      // 每个缓冲已提交的字节数
    uint maxSize_;

    uint writeOffset_;
    uint readOffset_;
    uint writeIndex_;
    uint readIndex_;
    uint readSize_;
public:
    #if ENABLE_STACK_TRACKER
    TraceCount traces_;              // 追踪写入来源（可选）
    #endif
    uint writeObjectSize_;
    uint writeRawSize_;
    uint writeSeekSize_;
};
```

### 2.3 双缓冲切换

`reset()` 是双缓冲切换的枢纽——它把当前写偏移固化为读尺寸，然后交换读写索引：

```cpp
// command_buffer.hpp:296-305
inline void CommandBuffer::reset()
{
    readSize_ = writeOffset_;                                    // 本帧写入量成为下帧读取量

    readIndex_ = ( readIndex_ + 1 ) % NUM_BUFFERS;              // 交换
    writeIndex_ = ( writeIndex_ + 1 ) % NUM_BUFFERS;

    readOffset_ = 0;
    writeOffset_ = 0;
}
```

切换后，消费者读 `buffer_[readIndex_]` 的 `[0, readSize_)` 区间，生产者写 `buffer_[writeIndex_]`，互不干扰。

### 2.4 按需内存提交

`prepWrite` 在写入前检查容量，不够时按 1MB 步长提交物理内存：

```cpp
// command_buffer.hpp:380-406
inline void CommandBuffer::prepWrite( uint writeSize )
{
    uint reqSize = writeOffset_ + writeSize;
    if ( reqSize > curSize_[writeIndex_] )
    {
        uint newSize = roundUp( reqSize, GROW_STEP );           // 1MB 对齐
        if ( newSize > maxSize_ )
        {
            CRITICAL_MSG( "Command buffer overflow\n" );
        }
        curSize_[writeIndex_] = newSize;
        void* mem = bw_virtualAlloc( buffer_[writeIndex_], curSize_[writeIndex_],
            MEM_COMMIT, PAGE_READWRITE );                       // 提交物理页
        if ( !mem )
        {
            CRITICAL_MSG( "Command buffer out of memory\n" );
        }
    }
}
```

`init` 时用 `MEM_RESERVE` 仅预留地址空间（不占物理内存），运行时按需 `MEM_COMMIT`。`maxSize_` 是上限，溢出即 `CRITICAL_MSG` 终止。

### 2.5 写入与读取

写入用模板重载，支持 1~8 个参数：

```cpp
// command_buffer.hpp:105-110
template <class A>
void write( A a )
{
    prepWrite( sizeof( A ) );
    writeObject( a );
}
```

`writeObject` 把对象拷贝到当前写指针位置并推进偏移。读取用模板 `read<A>()`，并在读后预取下一段：

```cpp
// command_buffer.hpp:193-199
template <class A> A read()
{
    A in = *((A*)readPtr());
    readOffset_ += sizeof( A );
    _mm_prefetch( ( char* )readPtr() + 128, _MM_HINT_T0 );      // 预取 128B 后的数据
    return in;
}
```

`writePadding` / `skipPadding` 把偏移按 16 字节对齐，保证后续 SSE 加载对齐。

### 2.6 memcpyNTA —— 非临时对齐拷贝

`memcpyNTA` 用 SSE 的 `movntdq`（非临时写，不污染 cache）做大批量拷贝，适合"写一次、读一次"的数据流（如顶点缓冲填充），避免把数据无谓地驻留 cache：

```cpp
// command_buffer.hpp:410-467
inline void CommandBuffer::memcpyNTA( void* dst, const void* src, uint size )
{
    // 用 movdqa 加载 128B 块，用 movntdq 非临时写
    // 尾部按 16B 处理
}
```

### 2.7 TraceCount —— 写入来源追踪

当 `ENABLE_STACK_TRACKER` 开启时，`CommandBuffer` 内嵌一个 `TraceCount`，按调用栈聚合计入的写入量，便于分析命令缓冲被谁占用：

```cpp
// command_buffer.hpp:25-72
class TraceCount
{
public:
    void addTrace( uint size );
    void reset();
    void print();
private:
    struct Trace { uint count_; uint size_; };
    typedef std::map<std::string, Trace> TraceMap;
    TraceMap traces_;
};
```

`addTrace` 调用 `StackTracker::buildReport()` 生成调用栈字符串作为 key，把同一栈的多次写入累加。这是 BigWorld"运行时可观测"哲学在 job_system 的体现。

---

## 3. Job / SyncBlock / JobSystem

[job_system.hpp](file:///workspace/src/lib/job_system/job_system.hpp) 定义三个核心类。

### 3.1 Job —— 最小并行单元

`Job` 是 16 字节对齐的抽象基类，每个 Job 是一个可在 worker 线程执行的原子任务：

```cpp
// job_system.hpp:26-35
class __declspec( align( 16 ) ) Job
{
    friend class JobSystem;
private:
    SyncBlock* syncBlock_;       // 本 Job 所属的 SyncBlock
private:
    virtual void execute() = 0;  // 派生类实现具体计算
};
```

`Job` 没有公共构造接口，只能通过 `JobSystem::allocJob<C>()` 在命令缓冲中分配（见 §3.4）。

### 3.2 SyncBlock —— 同步屏障

`SyncBlock` 是一组 Job 的同步屏障——同一 SyncBlock 内的所有 Job 执行完毕后，其 `consume()` 被调用（在消费线程）：

```cpp
// job_system.hpp:39-51
class SyncBlock
{
    friend class JobSystem;
private:
    uint numJobs_;              // 本块内 Job 数（随 allocJob 递增）
protected:
    void* next_;                // 背补丁：指向下一个 SyncBlock 在 consumptionCommands 中的位置
private:
    virtual void consume() = 0; // 所有 Job 完成后调用
};
```

`next_` 字段是"背补丁"（back-patching）机制的关键：SyncBlock 创建时还不知道它的"后继"是谁，等下一个 SyncBlock 创建时回填 `next_`，从而把一帧的所有 SyncBlock 串成链表，消费线程可顺序遍历。

### 3.3 JobSystem —— 调度器

`JobSystem` 是单例（继承 `Singleton<JobSystem>`），协调整个并行计算流水线：

```cpp
// job_system.hpp:55-162
class JobSystem : public Singleton<JobSystem>
{
private:
    class FirstSyncBlock : public SyncBlock
    {
    public:
        virtual void consume() { }
    public:
        LONG numFrameSyncBlocks_;       // 本帧 SyncBlock 数（不含首个）
    };

public:
    JobSystem();
    ~JobSystem();

    void init();
    void fini();

    void beginFrame();
    void endFrame();
    void flush();

    CommandBuffer* getConsumptionCommands();

    template<class C> C* allocSyncBlock();
    template<class C> C* allocJob( uint extraDataSize = 0 );
    template<class C> C* allocOutput( uint num = 1 );

    bool canBegin() const;
    void resetProfiling();

private:
    void jobLoop();                     // worker 线程主循环
    void consumptionLoop();             // 消费线程主循环
    void consumptionLoop2();
    static void jobThreadFunc( void* arg );
    static void consumptionThreadFunc( void* arg );
    static void consumptionThreadFunc2( void* arg );

private:
    static const uint OUTPUT_SIZE = 4*1024*1024;  // 4MB 输出缓冲

    SimpleThread* jobThreads_;          // worker 线程数组
    SimpleThread consumptionThread_;    // 消费线程

    uint numCores_;
    uint cores_[32];

    CommandBuffer jobCommands_;         // 主线程→worker 的命令
    CommandBuffer consumptionCommands_; // worker→消费线程的命令

    uint frameWriteIndex_;
    volatile bool grabFrame_;
    volatile LONG numAllocedFrames_;
    volatile LONG numQueuedFrames_;

    volatile LONG writeIndex_;          // worker 写，消费线程读
    volatile LONG readIndex_;           // 消费线程写，worker 读

    uint outputPos_;
    uint outputIndex_;
    uint8* outputBuffer_[2];            // 双缓冲输出

    FirstSyncBlock firstSyncBlock_[CommandBuffer::NUM_BUFFERS];
    SyncBlock* curSyncBlock_;           // 当前 SyncBlock（待背补丁）

    SimpleMutex jobReadMutex_;

    // 性能统计
    volatile uint64 frameTimeStamp_;
    volatile uint64 mainWait_;
    volatile uint64 consumeWaitOnBuffer_;
    volatile uint64 consumeWaitOnJobs_;
    volatile uint64 consumeDoJobs_;
    volatile uint64 jobWait_[30];
    // ... 各种 watcher 字段
};
```

### 3.4 allocJob / allocSyncBlock / allocOutput —— 命令缓冲分配

这三个模板方法是任务提交的核心。`allocJob` 在 `jobCommands_` 中分配并构造一个 Job，并把它关联到当前 SyncBlock：

```cpp
// job_system.hpp:173-190
template<class C> inline C* JobSystem::allocJob( uint extraDataSize )
{
    uint size = sizeof( C ) + extraDataSize;
    jobCommands_.write( size );              // 写入 Job 大小
    jobCommands_.writePadding();             // 16B 对齐
    C* job = (C*)jobCommands_.getCurrentWrite();
    jobCommands_.seekWrite( size );
    new( job ) C;                            // placement new 构造

    job->syncBlock_ = curSyncBlock_;         // 关联到当前 SyncBlock
    curSyncBlock_->numJobs_++;               // 计数

    numJobs_++;
    return job;
}
```

`allocSyncBlock` 切换输出缓冲、背补丁前一个 SyncBlock、在 `consumptionCommands_` 中分配新 SyncBlock、并把它的指针写入 `jobCommands_`（让 worker 知道新 SyncBlock 的存在）：

```cpp
// job_system.hpp:194-220
template<class C> inline C* JobSystem::allocSyncBlock()
{
    outputIndex_ ^= 1;                                   // 切换输出缓冲
    outputPos_ = 0;

    curSyncBlock_->next_ = consumptionCommands_.getCurrentWrite();  // 背补丁

    consumptionCommands_.write( sizeof( C ) );
    C* syncBlock = (C*)consumptionCommands_.getCurrentWrite();
    consumptionCommands_.seekWrite( sizeof( C ) );
    new( syncBlock ) C;
    syncBlock->numJobs_ = 0;

    jobCommands_.write( syncBlock );                     // 通知 worker

    firstSyncBlock_[frameWriteIndex_].numFrameSyncBlocks_++;
    curSyncBlock_ = syncBlock;

    numSyncBlocks_++;
    return syncBlock;
}
```

`allocOutput` 在双缓冲输出区分配一段内存，worker 把计算结果写到这里，消费线程读取：

```cpp
// job_system.hpp:224-230
template<class C> inline C* JobSystem::allocOutput( uint num )
{
    C* output = (C*)( outputBuffer_[outputIndex_] + outputPos_ );
    outputPos_ += num * sizeof( C );
    MF_ASSERT( outputPos_ <= OUTPUT_SIZE );
    return output;
}
```

### 3.5 帧循环

主线程每帧的典型流程：

```
beginFrame()
  ├─ 检查 canBegin()（上一帧是否已消费完）
  ├─ 重置 jobCommands_ / consumptionCommands_（双缓冲切换）
  └─ 初始化 firstSyncBlock_[frameWriteIndex_]

[主线程] 多次调用 allocSyncBlock<C>() + allocJob<C>() 提交任务
         （每个 SyncBlock 内可有多个 Job）

endFrame()
  ├─ 背补丁最后一个 SyncBlock
  ├─ 递增 numQueuedFrames_
  └─ 唤醒 worker 线程

[worker 线程] jobLoop()
  ├─ 从 jobCommands_ 读 SyncBlock 指针与 Job
  ├─ 对每个 Job 调用 execute()
  ├─ Job 完成后递减 SyncBlock::numJobs_
  └─ numJobs_==0 时把 SyncBlock 写入消费命令

[消费线程] consumptionLoop()
  ├─ 从 consumptionCommands_ 读已完成的 SyncBlock
  ├─ 调用 SyncBlock::consume()（在主线程或消费线程做收尾）
  └─ 通过 next_ 链跳到下一个 SyncBlock

flush()
  └─ 等待当前帧所有任务完成（用于退出/同步点）
```

`canBegin()` 防止主线程跑得过快超过双缓冲容量：

```cpp
// job_system.hpp:234-237
inline bool JobSystem::canBegin() const
{
    return ( numAllocedFrames_ < CommandBuffer::NUM_BUFFERS );
}
```

### 3.6 线程模型

job_system 用三类线程：

| 线程 | 数量 | 职责 |
|---|---|---|
| 主线程 | 1 | 生产 Job（每帧调用 `allocJob`），驱动 `beginFrame`/`endFrame` |
| worker 线程 | `numCores_`（最多 32） | 执行 `Job::execute()`，做实际计算 |
| 消费线程 | 1 | 执行 `SyncBlock::consume()`，做结果收尾 |

worker 与消费线程通过 `writeIndex_` / `readIndex_` 两个原子变量同步——worker 写入完成事件，消费线程读取。`SimpleMutex jobReadMutex_` 仅在少数边界情况加锁，热路径无锁。

`cores_[32]` 数组配合 cstdmf 的 `processor_affinity.hpp` 把 worker 绑定到特定 CPU 核心，减少缓存失效。

### 3.7 性能监控

`JobSystem` 内部维护大量 `volatile uint64` 时间戳与 `float` watcher 字段：

```cpp
// job_system.hpp:133-161
volatile uint64 frameTimeStamp_;
volatile uint64 mainWait_;                // 主线程等待时间
volatile uint64 consumeWaitOnBuffer_;     // 消费线程等缓冲时间
volatile uint64 consumeWaitOnJobs_;       // 消费线程等 Job 时间
volatile uint64 consumeDoJobs_;           // 消费线程执行时间
volatile uint64 jobWait_[30];             // 每个 worker 的等待时间

float frameTimeWatcher_;
float mainUsageTimeWatcher_;
float mainUsagePercentWatcher_;
float consumeUsageTimeWatcher_;
float consumeUsagePercentWatcher_;
float loadingUsageTimeWatcher_;
float loadingUsagePercentWatcher_;
float sumOfCoresTimeWatcher_;
float sumOfCoresPercentWatcher_;
float consumeWaitOnBufferWatcher_;
float consumeWaitOnJobsWatcher_;
float jobUsageTimeWatcher_[30];
float jobUsagePercentWatcher_[30];
```

这些字段通过 watcher 暴露，是客户端性能调优的主要数据源——可看到每帧主线程/消费线程/各 worker 的占用率与等待时间，定位并行瓶颈。

---

## 4. zip —— zlib 1.x

### 4.1 用途

`/workspace/src/lib/zip/` 是第三方 **zlib** 库（通用 DEFLATE 压缩/解压），BigWorld 用它处理：
- 资源包文件（`.packed`、`.zip`）的解压。
- 网络协议中压缩数据。
- 配置文件压缩。

### 4.2 BigWorld 改动

[bw_changes.txt](file:///workspace/src/lib/zip/bw_changes.txt) 记录的改动极小：

```
1. The file Makefile was modified to build using the BigWorld system.
2. Project files for Visual Studio were added to the top-level directory.
```

即只改了构建系统（Makefile 适配 BigWorld 构建链、添加 VS 工程文件），**未改动 zlib 源码本身**。目录中可见标准的 zlib 文件：`adler32.c`、`deflate.c`、`inflate.c`、`crc32.c`、`zlib.h`、`zconf.h` 等，以及 `contrib/`（含 `minizip` 等扩展）。

### 4.3 集成方式

业务代码通过标准 zlib API（`compress` / `uncompress` / `inflateInit` / `deflateInit` 等）使用，链接时把 `zip` 库加入依赖。resmgr 的资源包读取、chunk 的压缩地块加载等都依赖它。

---

## 5. png —— libpng 1.2.22

### 5.1 用途

`/workspace/src/lib/png/` 是第三方 **libpng 1.2.22** 库，BigWorld 用它加载/保存 PNG 图片：
- 纹理加载（客户端）。
- 截图保存。
- 资源处理工具（如 `bw_png` 命令行工具）。

### 5.2 BigWorld 改动

[bw_changes.txt](file:///workspace/src/lib/png/bw_changes.txt) 记录的改动同样很小：

```
1. In png.h the line:
        #include "zlib.h"
   has been changed to:
        #include "zip/zlib.h"

2. The Visual Studio projects have been updated to Visual Studio 2005
   from Visual Studio 2003.

3. A Makefile has been added for the Linux build to help build the server.
```

唯一功能性改动是把 `png.h` 中对 `zlib.h` 的 include 路径从 `"zlib.h"` 改为 `"zip/zlib.h"`，让 libpng 链接到 BigWorld 自带的 zlib（`/workspace/src/lib/zip/`），而不是系统 zlib——确保压缩/解压行为与资源打包时一致。其余只是工程文件升级与 Linux 构建支持。

### 5.3 集成方式

业务代码通过标准 libpng API（`png_create_read_struct` / `png_read_image` 等）使用。resmgr 的图片加载器、moo 的纹理加载管线等都依赖它。

---

## 6. job_system 与 bgtask_manager 的本质区别

为什么 BigWorld 需要两套并行系统？根本原因在于**任务特性不同**：

### 6.1 IO 密集 vs CPU 密集

`bgtask_manager` 处理 IO 密集任务（读文件、解压资源包）。这类任务的特点是：
- 单任务耗时长（毫秒~秒级）。
- 大部分时间在等系统调用（`read`、`fread`），CPU 空闲。
- 任务间通常无依赖。
- 适合**通用线程池 + 阻塞队列**——线程数可多于 CPU 核数（让 OS 在 IO 等待时切换），用信号量唤醒。

`job_system` 处理 CPU 密集任务（动画混合、蒙皮）。这类任务的特点是：
- 单任务耗时短（微秒~毫秒级）。
- 大部分时间在算术运算，CPU 满载。
- 任务间有数据依赖（A 的输出是 B 的输入）。
- 适合**固定 worker + 无锁命令缓冲**——线程数等于 CPU 核数（避免上下文切换开销），用双缓冲避免锁竞争。

### 6.2 通信机制对比

`bgtask_manager` 用 `SimpleMutex` + `SimpleSemaphore` 保护 `std::list<pair<int, BackgroundTaskPtr>>`。每次 `push`/`pull` 都加锁。对 IO 任务这没问题（锁开销远小于 IO 等待），但对 CPU 任务会成为瓶颈（锁开销可能超过任务本身）。

`job_system` 用 `CommandBuffer` 双缓冲：
- 主线程写 `buffer_[writeIndex_]`，worker 读 `buffer_[readIndex_]`，**完全无锁**。
- 帧边界用 `reset()` 切换缓冲，仅需原子地交换两个索引。
- 任务间依赖通过 SyncBlock 的 `numJobs_` 计数器表达，worker 用原子递减，归零时触发 `consume`。

### 6.3 数据流模型

`bgtask_manager` 的任务是自包含对象（`BackgroundTask` 派生类自带数据），任务间通过主线程中转（`doBackgroundTask` 末尾 `addMainThreadTask(this)` 转主线程收尾）。简单直接但有主线程开销。

`job_system` 的任务通过命令缓冲的"链式背补丁"组织：主线程一次性把一帧所有 Job 写入 `jobCommands_`，worker 顺序消费；SyncBlock 通过 `next_` 指针在 `consumptionCommands_` 中串成链，消费线程顺序遍历。整条流水线**无中转、无锁、顺序访问**，对 cache 友好。

### 6.4 适用场景

| 场景 | 选择 |
|---|---|
| 资源加载（读 .model 文件） | `bgtask_manager` |
| chunk 加载（读 .chunk + 解压） | `bgtask_manager` |
| 日志写入 | `bgtask_manager` |
| 角色动画混合 | `job_system` |
| 顶点蒙皮 | `job_system` |
| 粒子物理 | `job_system` |
| 渲染命令生成 | `job_system`（消费线程输出给渲染管线） |

---

## 7. 典型使用模式

### 7.1 job_system 使用骨架

```cpp
#include "job_system/job_system.hpp"

// 1. 自定义 SyncBlock（收尾逻辑）
class SkinSyncBlock : public SyncBlock
{
public:
    RenderBuffer* output_;
    virtual void consume()
    {
        // 所有蒙皮 Job 完成后调用，把结果提交给渲染管线
        RenderQueue::instance().submit( output_ );
    }
};

// 2. 自定义 Job（单个骨骼的蒙皮计算）
class SkinJob : public Job
{
public:
    const Matrix* boneTransforms_;
    const Vertex* sourceVertices_;
    Vertex* destVertices_;
    uint vertexCount_;
    virtual void execute()
    {
        for ( uint i = 0; i < vertexCount_; ++i )
        {
            // 用 boneTransforms_ 把 sourceVertices_[i] 变换到 destVertices_[i]
            // ...
        }
    }
};

// 3. 主线程每帧提交
void renderFrame()
{
    JobSystem& js = JobSystem::instance();
    if ( !js.canBegin() ) return;          // 上一帧还没消费完
    js.beginFrame();

    SkinSyncBlock* sb = js.allocSyncBlock<SkinSyncBlock>();
    sb->output_ = js.allocOutput<RenderBuffer>( 1 );

    // 把角色的每个骨骼拆成独立 Job
    for ( int b = 0; b < numBones; ++b )
    {
        SkinJob* job = js.allocJob<SkinJob>();
        job->boneTransforms_ = &bones[b];
        job->sourceVertices_ = &srcVerts[boneOffset[b]];
        job->destVertices_   = &sb->output_->verts[boneOffset[b]];
        job->vertexCount_    = boneCount[b];
    }

    js.endFrame();                          // 唤醒 worker
}

// 4. 主循环
void mainLoop()
{
    JobSystem::instance().init();
    JobSystem::instance().startThreads( numCores );  // 启动 worker + 消费线程
    while ( running )
    {
        renderFrame();
        // ... 其它主线程工作
        JobSystem::instance().flush();      // 可选：等当前帧完成
    }
    JobSystem::instance().fini();
}
```

### 7.2 bgtask_manager 使用骨架（对比）

```cpp
#include "cstdmf/bgtask_manager.hpp"

class LoadChunkTask : public BackgroundTask
{
public:
    LoadChunkTask(const std::string& path) : path_(path) {}
    void doBackgroundTask(BgTaskManager& mgr)
    {
        // 工作线程：读文件 + 解压
        data_ = readAndDecompress(path_);
        mgr.addMainThreadTask(this);        // 转主线程
    }
    void doMainThreadTask(BgTaskManager& mgr)
    {
        // 主线程：把数据挂到 chunk 系统
        ChunkManager::instance().onLoaded(path_, data_);
    }
private:
    std::string path_;
    BinaryData data_;
};

// 提交
BgTaskManager::instance().addBackgroundTask(
    new LoadChunkTask("spaces/foo.chunk") );
```

对比可见：`job_system` 是**面向帧的批量提交**，`bgtask_manager` 是**面向单个任务的零散提交**。

---

## 8. 小结

- **job_system** 是 BigWorld 客户端的高性能帧内并行框架，核心创新是 `CommandBuffer` 双缓冲 + SyncBlock/Job 链式背补丁，实现主线程、worker、消费线程三者间的无锁流水线。它专门服务于 CPU 密集的每帧计算（动画、蒙皮、粒子），与 cstdmf 的 `bgtask_manager`（IO 密集后台任务）形成互补。两者不可互相替代——把 IO 任务塞进 job_system 会阻塞 worker、把 CPU 任务塞进 bgtask_manager 会因锁竞争而低效。

- **zip**（zlib）与 **png**（libpng 1.2.22）是几乎未改动的第三方库，仅做了构建系统集成与（png 的）zlib 路径修正。它们是引擎资源管线的底层依赖，业务代码通过标准 zlib/libpng API 使用。
