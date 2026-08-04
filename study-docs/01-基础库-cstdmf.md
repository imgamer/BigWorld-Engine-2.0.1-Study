# BigWorld 基础库 cstdmf 深度分析

> 源码位置：`/workspace/src/lib/cstdmf/`
> 适用版本：BigWorld Technology 2.0.1

## 1. 模块定位

### 1.1 cstdmf 是什么

`cstdmf` 是 **C Standard Multi-platform Foundation**（C 标准多平台基础库）的缩写，是 BigWorld 引擎自研的跨平台 C++ 基础设施库。它的角色类似于一个"轻量级 Boost"——在 STL 之外补齐了引擎在容器、智能指针、内存追踪、调试日志、并发原语、定时器、运行时变量观察等方面的需求，并对 Windows / Linux / Xbox360 / PlayStation3 多平台做了统一封装。

`cstdmf` 是引擎中**最底层的库**，几乎所有其它模块（`math`、`network`、`resmgr`、`chunk`、`moo`、各 server 进程）都直接或间接依赖它。它本身只依赖 STL 与少量系统 API，不依赖 D3D / OpenGL / Python。

### 1.2 模块关系图（文字描述）

```
                      ┌──────────────────────────────────────────┐
                      │              cstdmf                       │
                      │  (跨平台基础：容器/指针/内存/调试/并发/  │
                      │   时间/Watcher/后台任务/二进制流)        │
                      └──────────────────────────────────────────┘
        ┌──────────────┬──────────────┬───────────────┬──────────────┐
        ▼              ▼              ▼               ▼              ▼
   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  math   │   │ network  │   │ resmgr   │   │  server  │   │   moo    │
   │ 数学库  │   │ 通信库   │   │ 资源管理 │   │ 服务端库 │   │ 渲染层  │
   └─────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
        │              │              │               │              │
        └──────────────┴──────────────┴───────────────┴──────────────┘
                                  ▼
                    各 server 进程 (cellapp/baseapp/...)
```

`cstdmf` 内部还可再细分为若干子领域，下文按子领域逐一展开。

### 1.3 编译开关

`cstdmf` 的行为受 [config.hpp](file:///workspace/src/lib/cstdmf/config.hpp) 中一系列宏控制，关键开关包括：

| 宏 | 作用 |
|---|---|
| `ENABLE_DPRINTF` | 启用 `dprintf` 调试输出 |
| `ENABLE_WATCHERS` | 启用 Watcher 系统（核心运行时观察机制） |
| `ENABLE_DOG_WATCHERS` | 启用 DogWatch 帧内分段时间统计 |
| `ENABLE_MEMTRACKER` | 启用 MemTracker 内存追踪 |
| `ENABLE_RESOURCE_COUNTERS` | 启用 ResourceCounters 资源计数 |
| `ENABLE_STACK_TRACKER` | 启用调用栈追踪 |
| `MF_SERVER` | 编译为服务端（影响内存追踪实现） |
| `MF_SINGLE_THREADED` | 单线程模式（并发原语退化为空操作） |
| `BWCLIENT_AS_PYTHON_MODULE` | 客户端作为 Python 模块构建时影响 TLS 实现 |

---

## 2. 容器与智能指针

### 2.1 avector —— 对齐内存向量

[avector.hpp](file:///workspace/src/lib/cstdmf/avector.hpp) 是 `std::vector` 的对齐变体，仅在 Win32 平台生效。其核心设计见文件头部：

```cpp
// avector.hpp:5-8
#if !defined( _WIN32 )
// WIN32PORT the Win32 server used the DirectX Matrix Libraries which requires Aligned Vectors
#define avector vector
```

可以看到，在非 Win32 平台上 `avector<T>` 直接被宏替换为 `std::vector<T>`；而在 Win32 上，它是一个完整重写的 vector，使用 `aallocator`（见 [aalloc.hpp](file:///workspace/src/lib/cstdmf/aalloc.hpp)）来分配 16 字节对齐的内存，从而能够安全存放需要 SIMD 对齐的类型（如 `Matrix`、`Vector4`）。

```cpp
// avector.hpp:57-61
template<class _Ty,
    class _Ax = aallocator<_Ty> >
class avector
    : public _AVector_val<_Ty, _Ax>
{   // varying size array of values
```

### 2.2 safe_fifo —— 线程安全 FIFO

[safe_fifo.hpp](file:///workspace/src/lib/cstdmf/safe_fifo.hpp) 提供一个极简的线程安全队列，内部用 `std::list` + `SimpleMutex` 实现：

```cpp
// safe_fifo.hpp:15-42
template <class T>
class SafeFifo
{
public:
    void push(T& t)
    {
        SimpleMutexHolder smh(mutex_);
        data_.push_back(t);
    }

    T pop()
    {
        SimpleMutexHolder smh(mutex_);
        T returned = *data_.begin();
        data_.pop_front();
        return returned;
    }
    // ...
private:
    mutable SimpleMutex mutex_;
    std::list<T> data_;
};
```

`SimpleMutexHolder` 是 RAII 风格的锁守卫（见 §4.1），保证异常路径下也能解锁。

### 2.3 static_array —— 栈上定长数组

[static_array.hpp](file:///workspace/src/lib/cstdmf/static_array.hpp) 提供 `StaticArray<T, N>`，是一个固定容量的栈上数组，但暴露了类似 `std::vector` 的接口（`size()`、`operator[]`、迭代器等）。它在 `DogWatchManager`、`MemTracker` 等需要"小型固定集合且不想堆分配"的场景下大量使用：

```cpp
// 来自 dogwatch.hpp:155 的使用示例
class Stat : public StaticArray<uint64,NUM_SLICES>, public ReferenceCount
{
public:
    int         watchid;
    mutable int flags;
};
```

### 2.4 pool_allocator —— 对象池分配器

[pool_allocator.hpp](file:///workspace/src/lib/cstdmf/pool_allocator.hpp) 实现了一个"永不真正释放"的对象池。其设计理念见文件头注释：

```cpp
// pool_allocator.hpp:20-27
/**
 *  A PoolAllocator is an object that never truly frees instances once
 *  allocated.  Therefore it should be used for small, frequently
 *  constructed/destroyed objects that will maintain a roughly constant active
 *  population over time.
 *
 *  Mercury::Packet is a good example of an object that fits this description.
 */
```

核心数据结构是一个空闲链表 `FreeList`：

```cpp
// pool_allocator.hpp:31-34
struct FreeList
{
    FreeList * next;
};
```

`allocate` 优先从空闲链表摘取，不够时才 `new char[]`；`deallocate` 把对象头插回链表。模板参数 `MUTEX` 默认是 `DummyMutex`（空操作），需要线程安全时可传 `SimpleMutex`：

```cpp
// pool_allocator.hpp:76-104
void * allocate( size_t size )
{
    MF_ASSERT( size == size_ );
    void * ret;
    mutex_.grab();
    {
        if (pHead_)
        {
            ret = (void*)pHead_;
            pHead_ = pHead_->next;
        }
        else
        {
            ret = (void*)new char[ size ];
            ++numInPoolTotal_;
        }
        ++numInPoolUsed_;
        ++numAllocatesEver_;
    }
    mutex_.give();
    return ret;
}
```

`PoolAllocator` 还自带 Watcher 集成（§8 详述），可远程查看池使用情况：

```cpp
// pool_allocator.hpp:149-170
#if ENABLE_WATCHERS
static WatcherPtr pWatcher()
{
    DirectoryWatcherPtr pWatcher = new DirectoryWatcher();
    PoolAllocator< MUTEX > * pNull = NULL;
    pWatcher->addChild( "numInPoolUsed",
            makeWatcher( pNull->numInPoolUsed_ ) );
    pWatcher->addChild( "numInPoolUnused",
            makeWatcher( *pNull, &PoolAllocator< MUTEX >::numInPoolUnused ) );
    // ...
}
#endif
```

### 2.5 aalloc / aligned —— 对齐内存分配

[aalloc.hpp](file:///workspace/src/lib/cstdmf/aalloc.hpp) 提供 `aallocator`，是一个 STL 兼容的分配器适配器，把分配委托给 [aligned.hpp](file:///workspace/src/lib/cstdmf/aligned.hpp) 的对齐分配函数。`aligned.hpp` 定义了 `Aligned` 基类与 `aligned_alloc` / `aligned_free`，用于满足 SIMD 指令（SSE 的 `__m128`）对 16 字节对齐的要求。`math` 库的 `VectorFastBase`、`MatrixBase` 等均继承自 `Aligned`。

### 2.6 智能指针

[smartpointer.hpp](file:///workspace/src/lib/cstdmf/smartpointer.hpp) 实现了侵入式引用计数智能指针，是 BigWorld 对象生命周期的核心机制。

#### 2.6.1 ReferenceCount / SafeReferenceCount

`ReferenceCount` 是非线程安全的引用计数基类，内部持有一个 `volatile mutable long count_`：

```cpp
// smartpointer.hpp:84-91
ReferenceCount()
    : count_(0)
{
    // ...
}

inline void incRef() const
{
    REFERENCE_COUNT_THREAD_CHECK;
    ++count_;
}

inline void decRef() const
{
    REFERENCE_COUNT_THREAD_CHECK;
    if (--count_ == 0)
        delete const_cast<ReferenceCount*>(this);
}
```

`SafeReferenceCount` 继承自 `ReferenceCount`，重写 `incRef` / `decRef` 为原子操作。不同平台用不同原语：

| 平台 | 原子原语 |
|---|---|
| Win32 (x86) | 内联 `__asm lock add` / `lock xadd` |
| Xbox360 | `InterlockedIncrement` / `InterlockedDecrement` |
| PlayStation3 | `cellAtomicIncr32` / `cellAtomicDecr32` |
| Linux (GCC) | `__asm__ volatile ("lock addl ...")` |

Linux 版的实现示例：

```cpp
// smartpointer.hpp:349-357
inline void incRef() const
{
    __asm__ volatile (
        "lock addl $1, %0"
        :                       // no output
        : "m"   (this->count_)  // input: this->count_
        : "memory"              // clobbers memory
    );
}
```

值得注意的是 `decRef` 中"如果递减前为 1 则 delete"这一逻辑通过 `intDecRef()` 返回递减前的值实现：

```cpp
// smartpointer.hpp:362-367
inline void decRef() const
{
    // if it was 1 before it was decremented, delete it now
    if (this->intDecRef() == 1)
        delete const_cast<SafeReferenceCount*>(this);
}
```

#### 2.6.2 SmartPointer / ConstSmartPointer

`ConstSmartPointer<Ty>` 是引用计数智能指针的主体，`SmartPointer<Ty>` 是其可变访问派生类。它们通过全局模板函数 `incrementReferenceCount` / `decrementReferenceCount` 与被指对象解耦——只要对象有 `incRef()` / `decRef()` 方法即可（不强制继承 `ReferenceCount`）：

```cpp
// smartpointer.hpp:427-431
template <class Ty>
inline void incrementReferenceCount( const Ty &Q )
{
    Q.incRef();
}
```

构造函数支持"已递增引用"语义（`alreadyIncremented`）和"尝试递增"语义（`FALLIBLE`），后者用于从裸指针构造时的线程安全场景：

```cpp
// smartpointer.hpp:559-560
ConstSmartPointer( const Object *P, TRY_tag ) :
    object_( incrementReferenceCountAttempt( *P ) ? P : 0 ) { }
```

引擎中几乎所有跨模块共享的对象（如 `Chunk`、`Model`、`BackgroundTask`、`Watcher` 本身）都用 `SmartPointer<T>` 持有。

### 2.7 stringmap / cache / value_or_null / bw_functor / guard

这几个文件提供小型工具：

- [stringmap.hpp](file:///workspace/src/lib/cstdmf/stringmap.hpp)：基于哈希的字符串→指针映射，常用于资源名→对象查找。
- [cache.hpp](file:///workspace/src/lib/cstdmf/cache.hpp)：泛型 LRU 缓存模板。
- [value_or_null.hpp](file:///workspace/src/lib/cstdmf/value_or_null.hpp)：可选值包装（早于 C++17 `std::optional`）。
- [bw_functor.hpp](file:///workspace/src/lib/cstdmf/bw_functor.hpp)：早于 tr1 的仿函数工具。
- [guard.hpp](file:///workspace/src/lib/cstdmf/guard.hpp)：RAII 守卫通用模板。

### 2.8 shared_ptr

[shared_ptr.hpp](file:///workspace/src/lib/cstdmf/shared_ptr.hpp) 是 BigWorld 在 `std::shared_ptr` 普及前提供的非侵入式引用计数指针，与 `SmartPointer`（侵入式）互补。新代码倾向于使用 STL 的 `std::tr1::shared_ptr`，此文件主要用于历史代码兼容。

---

## 3. 单例与对象生命周期

### 3.1 Singleton

[singleton.hpp](file:///workspace/src/lib/cstdmf/singleton.hpp) 是引擎中最简洁也最常用的单例基类：

```cpp
// singleton.hpp:37-66
template <class T>
class Singleton
{
protected:
    static T * s_pInstance;

public:
    Singleton()
    {
        MF_ASSERT( NULL == s_pInstance );
        s_pInstance = static_cast< T * >( this );
    }

    ~Singleton()
    {
        MF_ASSERT( this == s_pInstance );
        s_pInstance = NULL;
    }

    static T & instance()
    {
        MF_ASSERT( s_pInstance );
        return *s_pInstance;
    }

    static T * pInstance()
    {
        return s_pInstance;
    }
};
```

派生类需要在 `.cpp` 中用宏 `BW_SINGLETON_STORAGE` 显式实例化静态成员：

```cpp
// singleton.hpp:71-73
#define BW_SINGLETON_STORAGE( TYPE )                        \
template <>                                                    \
TYPE * Singleton< TYPE >::s_pInstance = NULL;                \
```

**特点**：
- 非线程安全（构造/析构期间假设单线程）。
- 不自行创建实例——需要用户在合适位置（通常是全局/静态）`new` 或栈上构造一个派生类对象。
- `pInstance()` 返回 NULL 而非抛异常，便于"单例尚未初始化"的探测。
- 文件头注释明确建议："Generally singletons should be avoided. If they _need_ to be used, try to use this as the base class."

### 3.2 init_singleton / named_object / intrusive_object / list_node

- [init_singleton.hpp](file:///workspace/src/lib/cstdmf/init_singleton.hpp)：带初始化顺序控制的单例，通过 `InitSingletonManager` 管理多个单例的初始化与销毁顺序，避免跨模块静态初始化依赖问题。
- [named_object.hpp](file:///workspace/src/lib/cstdmf/named_object.hpp)：给对象附加名字标识，便于调试。
- [intrusive_object.hpp](file:///workspace/src/lib/cstdmf/intrusive_object.hpp)：侵入式链表节点基类，配合 `IntrusiveObject<T>` 模板，使对象可被多个侵入式链表持有而不需额外节点分配。`StatWithRatesOfChange` 即用了它（见 math 库文档）。
- [list_node.hpp](file:///workspace/src/lib/cstdmf/list_node.hpp)：双向链表节点，被 `MemTracker` 用于串起所有已分配内存块。

### 3.3 fini_job —— 静态析构钩子

[fini_job.hpp](file:///workspace/src/lib/cstdmf/fini_job.hpp) 提供一个"在静态析构期运行任务"的机制：

```cpp
// fini_job.hpp:26-36
class FiniJob
{
public:
    FiniJob();
    virtual ~FiniJob() {};

    static bool runAll();

protected:
    virtual bool fini() = 0;
};
```

其用途见文件头注释：`MemTracker` 用它确保在内存统计完成前所有对象已清理。派生类构造时把自己加入全局队列，`FiniJob::runAll()` 在程序退出早期被调用，按优先级依次执行各 `fini()`。

---

## 4. 并发原语

[concurrency.hpp](file:///workspace/src/lib/cstdmf/concurrency.hpp) 是 cstdmf 的并发抽象层，对 Win32 / Linux / Xbox360 / PS3 / 单线程模式做了统一封装。

### 4.1 SimpleMutex / SimpleMutexHolder

Win32 用 `CRITICAL_SECTION`，Linux 用 `pthread_mutex_t`，单线程模式退化为空操作：

```cpp
// concurrency.hpp:393-405 (Linux)
class SimpleMutex
{
public:
    SimpleMutex()   { pthread_mutex_init( &mutex_, NULL); }
    ~SimpleMutex()  { pthread_mutex_destroy( &mutex_ ); }

    void grab()     { pthread_mutex_lock( &mutex_ ); }
    bool grabTry()  { return (pthread_mutex_trylock( &mutex_ ) == 0); }
    void give()     { pthread_mutex_unlock( &mutex_ ); }

private:
    pthread_mutex_t mutex_;
};
```

`SimpleMutexHolder` 是 RAII 守卫：

```cpp
// concurrency.hpp:616-623
class SimpleMutexHolder
{
public:
    SimpleMutexHolder( SimpleMutex & sm ) : sm_( sm )    { sm_.grab(); }
    ~SimpleMutexHolder()                                { sm_.give(); }
private:
    SimpleMutex & sm_;
};
```

`MatrixMutexHolder` 是一个有趣的变体：它维护一个 271 元素的互斥量矩阵，通过对指针/整数取哈希来选择其中一个互斥量，从而让不同对象可并行加锁、相同对象串行加锁——这是一种分片锁策略。

### 4.2 SimpleSemaphore

Win32 用 `CreateSemaphore`，Linux 用 `pthread_mutex_t` + `pthread_cond_t` 手工实现：

```cpp
// concurrency.hpp:411-458 (Linux 实现)
class SimpleSemaphore
{
public:
    SimpleSemaphore() : value_( 0 )
    {
        pthread_mutex_init( &lock_, NULL );
        pthread_cond_init( &wait_, NULL );
    }
    void pull()
    {
        pthread_mutex_lock( &lock_ );
        value_--;
        if (value_ < 0)
            pthread_cond_wait( &wait_, &lock_ );
        pthread_mutex_unlock( &lock_ );
    }
    void push()
    {
        pthread_mutex_lock( &lock_ );
        value_++;
        if (value_ <= 0)
            pthread_cond_signal( &wait_ );
        pthread_mutex_unlock( &lock_ );
    }
    // ...
};
```

### 4.3 SimpleThread

`SimpleThread` 封装线程创建与等待。Linux 版用 `pthread_create` / `pthread_join`：

```cpp
// concurrency.hpp:468-508
class SimpleThread
{
public:
    SimpleThread( SimpleThreadFunc threadfunc, void * arg )
    {
        this->init( threadfunc, arg );
    }
    void init( SimpleThreadFunc threadfunc, void * arg )
    {
        ThreadTrampolineInfo * info = new ThreadTrampolineInfo( threadfunc, arg );
        pthread_create( &thread_, NULL, trampoline, info );
    }
    ~SimpleThread()
    {
        pthread_join( thread_, NULL );
    }
    // ...
};
```

由于 Windows `_beginthreadex` 与 Linux `pthread_create` 的函数签名不同，这里用 `ThreadTrampolineInfo` + 静态 `trampoline` 函数做了适配。

### 4.4 THREADLOCAL / ThreadLocal

`THREADLOCAL(type)` 宏在 Linux 上展开为 `__thread type`，在 Win32 上展开为 `__declspec( thread ) type`。但当客户端作为 Python 模块构建（`BWCLIENT_AS_PYTHON_MODULE`）时，由于 DLL 中不能用 `__declspec(thread)`，会改用基于 TLS API 的 `ThreadLocal<T>` 模板类（见 concurrency.hpp:235-363）。文件注释指出后者约有 20% 性能损失，但线程局部变量很少用于热点路径。

`MemTracker` 即用 `THREADLOCAL(ThreadState)` 保存每线程的 slot 栈（见 §5.2）。

### 4.5 atomic_swap

提供 CAS（Compare-And-Swap）语义的指针交换原语，Linux x86 用内联汇编 `lock cmpxchg`：

```cpp
// concurrency.hpp:594-607
inline bool atomic_swap( void *& dst, void * curVal, void * newVal )
{
    char ret;
    __asm__ volatile (
            "lock cmpxchg %2, %1\n\t"
            "setz %0\n"
        :   "=r"   (ret)
        :   "m"        (dst),
            "r"        (newVal),
            "a"        (curVal)
        : "memory" );
    return ret;
}
```

### 4.6 BWConcurrency

`BWConcurrency` 命名空间提供主线程空闲通知机制（`startMainThreadIdling` / `endMainThreadIdling`），用于在主线程等待时让后台任务感知（如降低 CPU 占用）。

---

## 5. 内存与诊断

### 5.1 memory_counter / resource_counters

[memory_counter.hpp](file:///workspace/src/lib/cstdmf/memory_counter.hpp) 提供 `MemoryCounter` 类与 `MEMORY_COUNTER_*` 宏，用于按类别统计内存分配量。

[resource_counters.hpp](file:///workspace/src/lib/cstdmf/resource_counters.hpp) 提供更细粒度的资源计数，按"描述+池"二维记录：

```cpp
// resource_counters.hpp:33-50
class ResourceCounters
{
public:
    typedef std::pair<std::string, uint>            DescriptionPool;
    typedef unsigned int                            Handle;

    enum GranularityMode {MEMORY, SYSTEM_VIDEO_MISC, SYSTEM_DEFAULT_MANAGED_MISC};
    enum MemoryPool {DEFAULT = 0, MANAGED = 1, SYSTEM = 2};
    enum MemoryPoolMask {DEFAULT_MASK = 1, MANAGED_MASK = 2, SYSTEM_MASK = 4, MISC_MASK = 8};

    static ResourceCounters& instance();
    // ...
    size_t add(const DescriptionPool& descriptionPool, size_t amount);
    size_t subtract(const DescriptionPool& descriptionPool, size_t amount);
};
```

`Entry` 结构记录 size / peakSize / instances / peakInstances / numberAdds / numberSubs / sum 等完整统计，可输出 CSV 报告。`ResourceCounters` 自身是单例，并通过 `RESOURCE_COUNTER_ADD` / `RESOURCE_COUNTER_SUB` 宏在 `ENABLE_RESOURCE_COUNTERS` 关闭时退化为空操作。

### 5.2 MemTracker —— 内存追踪系统

[memory_tracker.hpp](file:///workspace/src/lib/cstdmf/memory_tracker.hpp) 是 cstdmf 最复杂的内存诊断工具。它在 `ENABLE_MEMTRACKER` 开启时**重定义 `malloc` / `realloc` / `free` / `strdup`**，把所有堆分配纳入追踪：

```cpp
// memory_tracker.hpp:218-223
#define malloc bw_malloc
#define realloc bw_realloc
#define free bw_free
#define strdup bw_strdup
```

核心概念是 **slot（槽）**：每个 slot 代表一类内存用途（如 "Network/Packet"、"ResMgr/Chunk"）。用户用宏声明并进入 slot：

```cpp
// memory_tracker.hpp:38-50
#define MEMTRACKER_DECLARE( id, name, flags )    \
    int g_memTrackerSlot_##id = g_memTracker.declareSlot( name, flags )

#define MEMTRACKER_BEGIN( id )                    \
    extern int g_memTrackerSlot_##id;             \
    g_memTracker.begin( g_memTrackerSlot_##id )

#define MEMTRACKER_END()                          \
    g_memTracker.end()

#define MEMTRACKER_SCOPED( id )                   \
    extern int g_memTrackerSlot_##id;             \
    ScopedMemTracker scopedMemTracker_##id( g_memTrackerSlot_##id )
```

`MEMTRACKER_SCOPED` 用 RAII 对象 `ScopedMemTracker` 自动 push/pop slot 栈。slot 栈是**线程局部**的（`THREADLOCAL(ThreadState) s_threadState`），每个线程有 64 层深度的 slot 栈，因此嵌套的 `MEMTRACKER_SCOPED` 可正确归因。

每个分配块前会插入一个 `Header`，记录所属 slot、id、size、可选调用栈：

```cpp
// memory_tracker.hpp:104-111
struct Header
{
    ListNode            node;           // Node for list of all allocated blocks
    uint                slot;           // The user assigned slot for this allocation
    uint                id;             // The allocation id, unique for this slot
    uint                size;           // Size of the block, not counting overhead
    uint                callStackSize;  // Size of the callstack data
};
```

所有已分配块通过 `ListNode` 串成全局链表，`reportAllocations()` 可在退出时打印泄漏。Slot 支持 `FLAG_CALLSTACK`（记录调用栈，服务端用 `<execinfo.h>` 的 `backtrace`）和 `FLAG_TRACK_BLOCKS`（逐块记录）。还支持 `declareBreak(slotId, allocId)` 在第 N 次分配时断点，便于定位特定泄漏。

`MemTracker` 也集成了 Watcher（§8），并支持 `crashOnLeak`（单元测试泄漏即崩溃）和 `maxSize`（人为限制可用内存以测试 OOM 路径）。

### 5.3 memory_trace / stack_tracker

- [memory_trace.hpp](file:///workspace/src/lib/cstdmf/memory_trace.hpp) / [stack_tracker.hpp](file:///workspace/src/lib/cstdmf/stack_tracker.hpp)：轻量级调用栈追踪，`StackTracker::push` / `pop` 在每个函数入口/出口记录标识，`buildReport()` 生成可读栈报告。被 `job_system` 的 `CommandBuffer` 用于追踪命令来源（见附录 A）。

### 5.4 dlmalloc

[dlmalloc.c](file:///workspace/src/lib/cstdmf/dlmalloc.c) / [dlmalloc.h](file:///workspace/src/lib/cstdmf/dlmalloc.h) 是 Doug Lea 的经典 `dlmalloc` 通用内存分配器实现。BigWorld 在某些平台用它替代系统 `malloc` 以获得更稳定的多线程分配性能。本系列文档不深入其实现（属于第三方代码）。

---

## 6. 调试与性能

### 6.1 debug / dprintf —— 分级日志

[debug.hpp](file:///workspace/src/lib/cstdmf/debug.hpp) 与 [dprintf.hpp](file:///workspace/src/lib/cstdmf/dprintf.hpp) 共同构成 BigWorld 的分级日志系统。

#### 6.1.1 消息优先级

`DebugMessagePriority` 定义了 10 个级别：

```cpp
// debug.hpp:64-77
enum DebugMessagePriority
{
    MESSAGE_PRIORITY_TRACE,     // 进入方法时
    MESSAGE_PRIORITY_DEBUG,     // 变量值
    MESSAGE_PRIORITY_INFO,      // 一般信息
    MESSAGE_PRIORITY_NOTICE,    // 比 INFO 重要但非错误
    MESSAGE_PRIORITY_WARNING,   // 可能的错误
    MESSAGE_PRIORITY_ERROR,     // 错误，但程序可继续
    MESSAGE_PRIORITY_CRITICAL,  // 严重错误，程序无法继续
    MESSAGE_PRIORITY_HACK,      // 临时调试，不应提交
    MESSAGE_PRIORITY_SCRIPT,    // 来自 Python 脚本
    MESSAGE_PRIORITY_ASSET,     // 资源相关
    NUM_MESSAGE_PRIORITY
};
```

对应 `TRACE_MSG` / `DEBUG_MSG` / `INFO_MSG` / ... / `HACK_MSG` / `SCRIPT_MSG` 等宏。每个 `.cpp` 文件可用 `DECLARE_DEBUG_COMPONENT( priority )` 声明自己的组件优先级，最终是否输出由 `DebugFilter::shouldAccept` 决定：

```cpp
// dprintf.hpp:80-85
static bool shouldAccept( int componentPriority, int messagePriority )
{
    return (messagePriority >=
        std::max( DebugFilter::instance().filterThreshold(),
            componentPriority ));
}
```

#### 6.1.2 DebugFilter 与回调

`DebugFilter` 是单例，管理：
- `filterThreshold_`：全局过滤阈值。
- `hasDevelopmentAssertions_`：是否在开发期触发断言。
- `pCriticalCallbacks_`：严重消息回调列表（如服务端把 CRITICAL 转发到中央日志）。
- `pMessageCallbacks_`：普通消息回调列表，回调返回 `true` 时抑制默认输出。

```cpp
// dprintf.hpp:46-167
class DebugFilter
{
public:
    static DebugFilter & instance() { /* 懒汉单例 */ }
    static bool shouldAccept( int componentPriority, int messagePriority );
    void addCriticalCallback( CriticalMessageCallback * pCallback );
    void addMessageCallback( DebugMessageCallback * pCallback );
    // ...
};
```

`dprintf` 函数本身在 `ENABLE_DPRINTF` 关闭时退化为空函数，零开销：

```cpp
// dprintf.hpp:193-202
#else
    inline void dprintf( const char * format, ... ) {}
    inline void dprintf( int componentPriority, int messagePriority,
                            const char * format, ...) {}
    inline void vdprintf( const char * format, va_list argPtr,
                            const char * prefix = NULL ) {}
#endif
```

### 6.2 profiler / profile —— 函数级性能采样

[profiler.hpp](file:///workspace/src/lib/cstdmf/profiler.hpp) 提供基于 slot 的函数级性能采样。每个被测代码块用 `PROFILER_DECLARE` 声明 slot，用 `PROFILER_SCOPED` 进入：

```cpp
// profiler.hpp:32-41
#define PROFILER_DECLARE( id, name )            \
    int g_profilerSlot_##id = Profiler::instance().declareSlot( name )

#define PROFILER_SCOPED( id )            \
    extern int g_profilerSlot_##id;        \
    ScopedProfiler scopedProfiler_##id( g_profilerSlot_##id )
```

`Profiler` 内部为每个 slot 维护 64 帧（`NUM_FRAMES`）的时间与计数历史，用 `uint64` 时间戳（来自 [timestamp.hpp](file:///workspace/src/lib/cstdmf/timestamp.hpp)）记录。slot 栈同样是线程局部的。`tick()` 推进帧计数器，旧帧被覆盖。

[profile.hpp](file:///workspace/src/lib/cstdmf/profile.hpp) 提供更简单的"作用域计时器"，常用于一次性测量。

### 6.3 DogWatch —— 帧内分层计时

[dogwatch.hpp](file:///workspace/src/lib/cstdmf/dogwatch.hpp) 是 BigWorld 标志性的帧内性能分析工具，用于回答"这一帧的 X 毫秒花在哪了"。

`DogWatch` 对象代表一个命名计时器（如 "Animation"、"Render"），用 `start()` / `stop()` 包裹被测代码段，或用 `ScopedDogWatch` RAII：

```cpp
// dogwatch.hpp:46-67
class DogWatch
{
public:
    DogWatch( const char * title );
    void start();
    void stop();
    uint64 slice() const;
    const std::string & title() const;
    static uint64 s_nullSlice_;
private:
    uint64      started_;
    uint64      *pSlice_;
    int         id_;
    int         profilerId_;
    std::string title_;
};

class ScopedDogWatch
{
public:
    inline ScopedDogWatch( DogWatch & dogWatch ) : dogWatch_( dogWatch )
    {   this->dogWatch_.start(); }
    inline ~ScopedDogWatch()
    {   this->dogWatch_.stop();   }
private:
    DogWatch & dogWatch_;
};
```

`DogWatchManager` 是单例，维护：
- **120 个 slice（时间片）** 的环形历史（`NUM_SLICES = 120`），可回看过去 120 帧的每帧耗时。
- **最多 128 个 DogWatch**（`MAX_WATCHES = 128`）。
- **分层 manifestation 机制**：根据"哪些 DogWatch 在 start 时已有其它 DogWatch 在运行"，自动构建调用层级树（`Table` / `Stat`）。例如若 "Render" 内部总是包含 "Skinning"，则统计会自动归到 "Render/Skinning" 节点下，无需手工声明层级关系。

```cpp
// dogwatch.hpp:102-152
class DogWatchManager
{
public:
    enum { NUM_SLICES = 120, MAX_WATCHES = 128, };
    void tick();                        // 进入下一帧
    class iterator;
    iterator begin() const;
    iterator end() const;
    static DogWatchManager & instance();
    // ...
private:
    uint32  slice_;
    struct TableElt;
    std::vector<TableElt *> stack_;     // 当前活跃的 DogWatch 栈
    struct Cache { TableElt * stackNow; TableElt * stackNew; uint64 * pSlice; }
        cache_[MAX_WATCHES];            // manifestation 缓存
    // ...
};
```

`tick()` 在每帧末尾调用，把当前 slice 的累计时间固化，并推进 slice 索引。`iterator` 用于遍历所有 manifestation 节点并读取历史值。DogWatch 数据通过 Watcher 暴露（§8），是 cellapp/baseapp 等服务端运行时调优的主要数据源。

### 6.4 diary / message_box / critical_message_box

- [diary.hpp](file:///workspace/src/lib/cstdmf/diary.hpp) / [diary.ipp](file:///workspace/src/lib/cstdmf/diary.ipp)：结构化事件日志，比 `dprintf` 更适合长期运行的服务端。
- [message_box.hpp](file:///workspace/src/lib/cstdmf/message_box.hpp)：跨平台消息框（客户端用）。
- [critical_message_box.hpp](file:///workspace/src/lib/cstdmf/critical_message_box.hpp)：严重错误消息框。

---

## 7. 时间管理

### 7.1 timestamp —— 高精度时间戳

[timestamp.hpp](file:///workspace/src/lib/cstdmf/timestamp.hpp) 提供 `timestamp()` 函数，返回 `uint64` 时间戳。在 Win32/x86 上默认用 `RDTSC`（读 CPU 时间戳计数器），最快但与 CPU 频率耦合；在 Linux 上支持多种后端：

```cpp
// timestamp.hpp:35-41
enum BWTimingMethod
{
    RDTSC_TIMING_METHOD,
    GET_TIME_OF_DAY_TIMING_METHOD,
    GET_TIME_OF_DAY_TIMING_METHOD,
    NO_TIMING_METHOD,
};
extern BWTimingMethod g_timingMethod;
```

`timestamp_rdtsc()` 用内联汇编 `rdtsc`；`timestamp_gettimeofday()` 用 `gettimeofday`（慢 20~600 倍但有校时优势）；`timestamp_gettime()` 用 `clock_gettime(CLOCK_MONOTONIC)`。运行时按 `g_timingMethod` 选择，便于在 SpeedStep 等变频 CPU 上降级。

还提供 `stampsToSeconds` 等换算函数，把时间戳转为秒。

### 7.2 TimeQueue —— 定时器队列

[time_queue.hpp](file:///workspace/src/lib/cstdmf/time_queue.hpp) 实现基于优先队列的定时器。模板类 `TimeQueueT<TIME_STAMP>` 按时间戳类型参数化，预定义了两个别名：

```cpp
// time_queue.hpp:229-230
typedef TimeQueueT< uint32 > TimeQueue;
typedef TimeQueueT< uint64 > TimeQueue64;
```

`TimeQueue64` 用于高精度场景。核心 API：

| 方法 | 作用 |
|---|---|
| `add(startTime, interval, pHandler, pUser)` | 添加定时器，返回 `TimerHandle` |
| `process(now)` | 处理所有到期定时器 |
| `nextExp(now)` | 返回下一个定时器距 `now` 的剩余时间 |
| `legal(handle)` | 判断 handle 是否合法 |
| `clear()` | 清空队列 |

每个定时器节点 `Node` 持有到期时间 `time_`、周期 `interval_`、回调 `TimerHandler*`、用户数据 `void*`，状态机为 `STATE_PENDING` / `STATE_EXECUTING` / `STATE_CANCELLED`：

```cpp
// time_queue.hpp:32-61
class TimeQueueNode
{
public:
    TimeQueueNode( TimeQueueBase & owner, TimerHandler * pHandler, void * pUserData );
    void cancel();
    bool isCancelled() const    { return state_ == STATE_CANCELLED; }
protected:
    bool isExecuting() const    { return state_ == STATE_EXECUTING; }
    enum State { STATE_PENDING, STATE_EXECUTING, STATE_CANCELLED };
    // ...
};
```

优先队列是手写的（不直接用 `std::priority_queue`），以便访问底层容器做 `purgeCancelledNodes`：

```cpp
// time_queue.hpp:162-217
class PriorityQueue
{
public:
    void push( const value_type & x )
    {
        container_.push_back( x );
        std::push_heap( container_.begin(), container_.end(), Comparator() );
    }
    void pop()
    {
        std::pop_heap( container_.begin(), container_.end(), Comparator() );
        container_.pop_back();
    }
    Container & container()     { return container_; }
    void heapify() { std::make_heap( container_.begin(), container_.end(), Comparator() ); }
};
```

`Comparator` 让最小 `time_` 在堆顶（用 `>` 反向比较）。`process(now)` 循环 `pop` 并 `triggerTimer`，对周期性定时器更新 `time_ += interval_` 后重新 `push`。

[timer_handler.hpp](file:///workspace/src/lib/cstdmf/timer_handler.hpp) 定义 `TimerHandler` 抽象基类，`handleTimeout` 是回调入口。

### 7.3 date_time_utils

[date_time_utils.hpp](file:///workspace/src/lib/cstdmf/date_time_utils.hpp) 提供日历时间格式化/解析工具，用于日志时间戳、配置文件中的时间字段等。

---

## 8. ★ Watcher 系统（重点）

Watcher 是 BigWorld **最具标志性的基础设施**：一个运行时变量观察与修改系统，让开发者能在程序运行中远程查看和修改任意已注册的 C++ 变量、调用任意已注册的函数。它是 cellapp/baseapp/manager 等服务端进程运维的核心。

### 8.1 整体架构

```
┌────────────────┐   Watcher 协议    ┌──────────────────────┐
│  bwmachined /  │ ◄──────────────► │  server 进程          │
│  wbtool / Web  │   (TCP/UDP)       │  ┌──────────────────┐ │
│  控制台        │                   │  │ WatcherNub       │ │
└────────────────┘                   │  │  (network/watcher │ │
                                     │  │   _nub.hpp)       │ │
                                     │  └────────┬─────────┘ │
                                     │           │           │
                                     │  ┌────────▼─────────┐ │
                                     │  │ Watcher (root)   │ │
                                     │  │  树形结构         │ │
                                     │  │  cstdmf/watcher  │ │
                                     │  │  .hpp            │ │
                                     │  └────────┬─────────┘ │
                                     │   ┌──────┴──────┐    │
                                     │   ▼             ▼    │
                                     │ MemberWatcher DataWatcher ...
                                     │   │                  │
                                     │   ▼                  │
                                     │ 业务对象变量          │
                                     └──────────────────────┘
```

核心头文件是 [watcher.hpp](file:///workspace/src/lib/cstdmf/watcher.hpp)，路径请求抽象在 [watcher_path_request.hpp](file:///workspace/src/lib/cstdmf/watcher_path_request.hpp)，网络协议编解码在 [server/watcher_protocol.hpp](file:///workspace/src/lib/server/watcher_protocol.hpp) 与 [network/watcher_nub.hpp](file:///workspace/src/lib/network/watcher_nub.hpp)。

### 8.2 核心抽象：Watcher 基类

`Watcher` 是所有 watcher 的抽象基类，继承自 `SafeReferenceCount`（因此可用 `WatcherPtr` 即 `SmartPointer<Watcher>` 持有）：

```cpp
// watcher.hpp:586-607
class Watcher : public SafeReferenceCount
{
public:
    enum Mode
    {
        WT_INVALID,         ///< Indicates an error.
        WT_READ_ONLY,       ///< Indicates the watched value cannot be changed.
        WT_READ_WRITE,      ///< Indicates the watched value can be changed.
        WT_DIRECTORY,       ///< Indicates that the watcher has children.
        WT_CALLABLE         ///< Indicates the watcher is a callable function
    };

    Watcher( const char * comment = "" );
    virtual ~Watcher();
```

`Mode` 枚举决定 watcher 的访问语义。四个纯虚方法是访问器模式的核心：

| 方法 | 用途 |
|---|---|
| `getAsString(base, path, result, desc, mode)` | 把 path 处的值转为字符串（协议 v1） |
| `setFromString(base, path, valueStr)` | 从字符串设置 path 处的值（协议 v1） |
| `getAsStream(base, path, pathRequest)` | 把值写入二进制流（协议 v2，支持类型化数据） |
| `setFromStream(base, path, pathRequest)` | 从二进制流设置值（协议 v2） |

`base` 参数是"基址偏移"——指向宿主对象的指针，使同一个 `MemberWatcher` 可服务于多个对象实例（如遍历容器时）。`path` 是相对本 watcher 的子路径，用 `/` 分隔，空字符串表示"本 watcher 自身"。

`Watcher` 还有几个非纯虚方法：`visitChildren`（遍历子节点，目录 watcher 实现）、`addChild` / `removeChild`（增删子节点）、`setComment` / `getComment`（描述文本）。

静态方法 `rootWatcher()` 返回全局根 watcher（一个 `DirectoryWatcher`），所有业务 watcher 都挂在其下。`partitionPath` 把 `"a/b/c"` 拆为名字 `"c"` 与目录 `"a/b"`。

### 8.3 WatcherDataType —— 协议类型系统

协议 v2 是类型化的，`WatcherDataType` 枚举定义了线上可传输的类型：

```cpp
// watcher.hpp:35-49
enum WatcherDataType {
    WATCHER_TYPE_UNKNOWN = 0,
    WATCHER_TYPE_INT,
    WATCHER_TYPE_UINT,
    WATCHER_TYPE_FLOAT,
    WATCHER_TYPE_BOOL,
    WATCHER_TYPE_STRING,    // 5
    WATCHER_TYPE_TUPLE,
    WATCHER_TYPE_TYPE
    // Potential type additions
    //WATCHER_TYPE_VECTOR2,
    //WATCHER_TYPE_VECTOR3,
    //WATCHER_TYPE_VECTOR4
};
```

配套的全局模板函数 `watcherValueToStream` / `watcherStreamToValue` 处理各类型与 `BinaryOStream` / `BinaryIStream` 之间的序列化，并支持类型间的安全转换（如 int32↔int64、float↔double、字符串↔数值）：

```cpp
// watcher.hpp:827-834
template <class ValueType>
void watcherValueToStream( BinaryOStream & result, const ValueType & value,
                           const Watcher::Mode & mode )
{
    result << (uchar)WATCHER_TYPE_STRING;
    result << (uchar)mode;
    result << watcherValueToString( value );
}
```

非字符串类型有特化版本，写入 `[type:1B][mode:1B][length][value]` 格式。

### 8.4 Watcher 子类体系

watcher.hpp 内置了一整套子类，覆盖典型场景：

| 子类 | 模板参数 | 用途 |
|---|---|---|
| `DirectoryWatcher` | 无 | 目录节点，持有 `vector<DirData>`，实现树形结构 |
| `SequenceWatcher<SEQ>` | 序列类型 | 观察向量/数组，按下标或标签访问元素 |
| `MapWatcher<MAP>` | 映射类型 | 观察 `std::map` 等关联容器，按键访问 |
| `DereferenceWatcher` | 无（抽象） | 解引用 base 指针后委托给内部 watcher |
| `BaseDereferenceWatcher` | 无 | 把 base 当裸指针解引用 |
| `SmartPointerDereferenceWatcher` | 无 | 把 base 当 `SmartPointer` 解引用 |
| `ContainerBounceWatcher<CT,KEY>` | 容器+键 | 用 base 作为另一个容器的键 |
| `MemberWatcher<R,O,C>` | 返回/对象/构造 | 通过 get/set 成员函数访问对象属性 |
| `DataWatcher<TYPE>` | 值类型 | 直接观察一个变量引用 |
| `FunctionWatcher<R,C>` | 返回/构造 | 通过 get/set 全局函数访问 |
| `CallableWatcher` | 无 | 可调用 watcher（远程函数调用） |
| `NoArgCallableWatcher` | 无 | 无参可调用 |
| `NoArgFuncCallableWatcher` | 无 | 无参可调用，委托给 C 函数 |

#### 8.4.1 DirectoryWatcher

`DirectoryWatcher` 是树形结构的内部节点，`container_` 是 `vector<DirData>`，每个 `DirData` 持有子 `WatcherPtr`、`base` 偏移、`label`：

```cpp
// watcher.hpp:958-1007
class DirectoryWatcher : public Watcher
{
public:
    virtual bool getAsString( ... ) const;
    virtual bool getAsStream( ... ) const;
    virtual bool setFromString( ... );
    virtual bool setFromStream( ... );
    virtual bool visitChildren( ... );
    virtual bool addChild( const char * path, WatcherPtr pChild, void * withBase = NULL );
    virtual bool removeChild( const char * path );
    static const char * tail( const char * path );
private:
    struct DirData { WatcherPtr watcher; void * base; std::string label; };
    DirData * findChild( const char * path ) const;
    typedef std::vector<DirData> Container;
    Container container_;
};
```

`addChild` 支持路径自动递归创建：若 `path` 含 `/`，则切出第一段在自身找/建子目录，剩余路径递归下传。

#### 8.4.2 MemberWatcher —— 访问器模式典型

`MemberWatcher` 用 get/set 成员函数指针访问对象属性，是最常用的 watcher 之一：

```cpp
// watcher.hpp:1795-1800
template <class RETURN_TYPE,
    class OBJECT_TYPE,
    class CONSTRUCTION_TYPE = RETURN_TYPE>
class MemberWatcher : public Watcher
{
public:
    MemberWatcher( RETURN_TYPE (OBJECT_TYPE::*getMethod)() const,
            void (OBJECT_TYPE::*setMethod)( RETURN_TYPE ) = NULL );
```

`getAsString` 实现展示了 `base` 偏移的用法——`(OBJECT_TYPE*)(pObject_ + base)` 让同一个 watcher 可服务于不同对象实例：

```cpp
// watcher.hpp:1836-1860
virtual bool getAsString( const void * base, const char * path,
    std::string & result, std::string & desc, Watcher::Mode & mode ) const
{
    if (!isEmptyPath( path )) return false;
    if (getMethod_ == (GetMethodType)NULL) { /* ... */ }

    const OBJECT_TYPE & useObject = *(OBJECT_TYPE*)(
        ((const uintptr)pObject_) + ((const uintptr)base) );

    RETURN_TYPE value = (useObject.*getMethod_)();

    result = watcherValueToString( value );
    desc = comment_;
    mode = (setMethod_ != (SetMethodType)NULL) ? WT_READ_WRITE : WT_READ_ONLY;
    return true;
};
```

#### 8.4.3 SequenceWatcher / MapWatcher

`SequenceWatcher<SEQ>` 观察 STL 序列容器（`vector`/`list`/`deque`），元素可用下标、标签数组、子路径标签、或自定义 index↔string 转换器定位。`visitChildren` 遍历所有元素并通知 `pathRequest`：

```cpp
// watcher.hpp:1144-1214
virtual bool visitChildren( const void * base, const char * path,
    WatcherPathRequest & pathRequest )
{
    if (isEmptyPath(path))
    {
        SEQ & useVector = *(SEQ*)( ((uintptr)&toWatch_) + ((uintptr)base) );
        pathRequest.addWatcherCount( useVector.size() );
        int count = 0;
        for( SEQ_iterator iter = useVector.begin(); iter != useVector.end(); iter++, count++ )
        {
            // 决定 callLabel：优先 labels_，其次 labelsub_，再次 indexToString_，最后用数字
            // ...
            pathRequest.addWatcherPath( (void*)(subBase_ + (uintptr)&rChild),
                                      this->tail( path ), callLabel, *child_ );
        }
        return true;
    }
    // ...
}
```

`MapWatcher<MAP>` 类似，但用 `watcherStringToValue` 把路径段转为 `MAP::key_type` 后 `find`。

#### 8.4.4 CallableWatcher —— 远程函数调用

`CallableWatcher` 让远程可以调用本地函数。它支持 `ExposeHint`（`WITH_ENTITY`/`ALL`/`WITH_SPACE`/`LEAST_LOADED`/`LOCAL_ONLY`）控制调用范围，并通过 `addArg` 声明参数类型：

```cpp
// watcher.hpp:2433-2478
class CallableWatcher : public Watcher
{
public:
    enum ExposeHint { WITH_ENTITY, ALL, WITH_SPACE, LEAST_LOADED, LOCAL_ONLY };
    CallableWatcher( ExposeHint hint = LOCAL_ONLY, const char * comment = "" );
    void addArg( WatcherDataType type, const char * description = "" );
    // ...
};
```

`NoArgFuncCallableWatcher` 把回调委托给一个 `bool (*)(std::string& output, std::string& value)` 函数指针，是引擎中最常见的"远程命令"实现方式（如 `shutDown`、`setLogOn` 等）。

### 8.5 工厂函数与 MF_WATCH 宏

为了简化使用，watcher.hpp 提供了一组 `makeWatcher` / `addWatcher` / `addReferenceWatcher` 模板函数重载，自动推导类型并构造合适的 watcher 子类：

```cpp
// watcher.hpp:1970-1975
template <class RETURN_TYPE, class OBJECT_TYPE>
WatcherPtr makeWatcher( const RETURN_TYPE & (OBJECT_TYPE::*getMethod)() const )
{
    return new MemberWatcher<const RETURN_TYPE &, OBJECT_TYPE, RETURN_TYPE>(
            getMethod, NULL );
}
```

业务代码通过宏使用，最常见的入口是 `MF_WATCH`：

```cpp
// watcher.hpp:2764
#define MF_WATCH                ::addWatcher
```

典型用法（来自 watcher.hpp 注释）：

```cpp
// 静态变量
static int myValue = 72;
MF_WATCH( "myValue", myValue );                 // 默认 RW
MF_WATCH( "myValue", myValue, WT_READ_ONLY );   // 只读

// 对象成员访问器
class ExampleClass {
public:
    int getValue() const;
    void setValue( int setValue ) const;
};
ExampleClass exampleObject_;
MF_WATCH( "some value", exampleObject_,
        ExampleClass::getValue,
        ExampleClass::setValue );
```

`MF_ACCESSORS` 宏帮助处理重载访问器的类型转换：

```cpp
// watcher.hpp:2815-2817
#define MF_ACCESSORS( TYPE, CLASS, METHOD )                            \
    static_cast< TYPE (CLASS::*)() const >(&CLASS::METHOD),            \
    static_cast< void (CLASS::*)(TYPE)   >(&CLASS::METHOD)
```

在 `ENABLE_WATCHERS` 关闭时，`MF_WATCH` 退化为空操作（或 VS2003 上 `false &&` 表达式），零开销：

```cpp
// watcher.hpp:2840-2844
#if defined(_WIN32) && ( _MSC_VER < 1400 )
    #define MF_WATCH false &&
#else
    #define MF_WATCH( ... )
#endif
```

### 8.6 WatcherPathRequest —— 异步路径请求

[watcher_path_request.hpp](file:///workspace/src/lib/cstdmf/watcher_path_request.hpp) 抽象了一次 watcher 查询的请求/响应生命周期，支持异步。

`WatcherPathRequest` 是基类，定义了与 `visitChildren` 协作的方法：

```cpp
// watcher_path_request.hpp:30-115
class WatcherPathRequest
{
public:
    WatcherPathRequest( const std::string & path ) : pParent_( NULL ), requestPath_( path ) { }
    virtual void fetchWatcherValue() { }
    virtual bool setWatcherValue() { return false; }
    const std::string & getPath() const { return requestPath_; }
    void setParent( WatcherPathRequestNotification *parent ) { pParent_ = parent; }
    virtual void notifyParent( int32 replies=1 );
    virtual void addWatcherCount( int32 count ) {}
    virtual bool addWatcherPath( const void *base, const char *path,
                                 std::string & label, Watcher &watcher ) { return false; }
    virtual const char *getData() { return NULL; }
    virtual int32 getDataSize() { return 0; }
protected:
    WatcherPathRequestNotification *pParent_;
    std::string requestPath_;
};
```

`WatcherPathRequestV2` 是协议 v2 实现，持有一个 `MemoryOStream result_`（响应流）和一个 `MemoryIStream *setStream_`（设置操作的输入流）：

```cpp
// watcher_path_request.hpp:122-172
class WatcherPathRequestV2 : public WatcherPathRequest
{
public:
    WatcherPathRequestV2( const std::string & path );
    bool setPacketData( uint32 size, const char *data );
    void setSequenceNumber( uint32 seqNum );
    virtual bool setWatcherValue();
    virtual void fetchWatcherValue();
    virtual void setResult( const std::string & desc, const Watcher::Mode & mode,
                            const Watcher * watcher, const void *base );
    void addWatcherCount( int32 count );
    bool addWatcherPath( const void *base, const char *path,
                         std::string & label, Watcher & watcher );
    MemoryOStream & getResultStream() { return result_; }
    BinaryIStream * getValueStream() { return setStream_; }
private:
    MemoryOStream result_;
    std::string originalRequestPath_;
    bool hasSeqNum_;
    bool visitingDirectories_;
    char *streamData_;
    MemoryIStream *setStream_;
    WatcherPathRequestNotification *parent;
};
```

`WatcherPathRequestV1` 是协议 v1 实现，同时继承 `WatcherVisitor`，用 visitor 模式收集目录子节点的字符串表示。

`WatcherPathRequestNotification` 是回调接口，由 `WatcherNub` 实现，在请求完成时被 `notifyParent` 调用：

```cpp
// watcher_path_request.hpp:256-276
class WatcherPathRequestNotification : public ReferenceCount
{
public:
    virtual void notifyComplete( WatcherPathRequest & pathRequest, int32 count ) = 0;
    virtual WatcherPathRequest * newRequest( std::string & path ) = 0;
};
```

### 8.7 WatcherProtocol —— 远程协议

[server/watcher_protocol.hpp](file:///workspace/src/lib/server/watcher_protocol.hpp) 实现 watcher 协议 v2 的解码端，把二进制流还原为类型化的值并交给派生类的 handler：

```cpp
// watcher_protocol.hpp:23-38
class WatcherProtocolDecoder
{
public:
    virtual bool decode( BinaryIStream & stream );
    virtual bool decodeNext( BinaryIStream & stream );
    virtual int readSize( BinaryIStream & stream );
    virtual bool defaultHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool intHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool uintHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool floatHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool boolHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool stringHandler( BinaryIStream & stream, Watcher::Mode mode );
    virtual bool tupleHandler( BinaryIStream & stream, Watcher::Mode mode );
};
```

### 8.8 WatcherNub —— 网络接入

[network/watcher_nub.hpp](file:///workspace/src/lib/network/watcher_nub.hpp) 定义 `WatcherNub`、`WatcherRegistrationMsg`、`WatcherDataMsg` 及 `WatcherMsg` 枚举，负责：
- 向 `bwmachined` 注册本进程的 watcher 端口。
- 接收远程 watcher 请求包，构造 `WatcherPathRequestV2`，调用 `Watcher::rootWatcher()` 的 `getAsStream` / `setFromStream` / `visitChildren`。
- 把 `WatcherPathRequestV2::getResultStream()` 的内容打包回送。

### 8.9 典型调用流（ASCII 时序图）

以 `wbtool` 查询 `cellapp` 的 `components/CellApp/numas/0/load` 为例：

```
  wbtool                bwmachined              cellapp (WatcherNub)         rootWatcher(Directory)        MapWatcher(numas)      MemberWatcher(load)
     │                       │                         │                            │                            │                        │
     │ GET path=...          │                         │                            │                            │                        │
     ├──────────────────────►│                         │                            │                            │                        │
     │                       │ forward GET             │                            │                            │                        │
     │                       ├────────────────────────►│                            │                            │                        │
     │                       │                         │ new WatcherPathRequestV2   │                            │                        │
     │                       │                         │ fetchWatcherValue()        │                            │                        │
     │                       │                         ├───────────────────────────►│ getAsStream(base=0,path)   │                        │
     │                       │                         │                            │ 分解 path: components/...  │                        │
     │                       │                         │                            │ 递归到 numas 目录          │                        │
     │                       │                         │                            ├───────────────────────────►│ visitChildren           │
     │                       │                         │                            │                            │ addWatcherCount(size)  │
     │                       │                         │                            │                            │ addWatcherPath(elem0)──►│ getAsString
     │                       │                         │                            │                            │                        │ 返回 load 值
     │                       │                         │◄───────────────────────────┤                            │                        │
     │                       │                         │ notifyParent()             │                            │                        │
     │                       │                         │ 把 result_ 流打包回送      │                            │                        │
     │                       │ response                │                            │                            │                        │
     │                       │◄────────────────────────┤                            │                            │                        │
     │ response              │                         │                            │                            │                        │
     │◄──────────────────────┤                         │                            │                            │                        │
```

### 8.10 实战代码片段

引擎中到处可见 `MF_WATCH` 调用，例如 `PoolAllocator` 在构造时把自己挂到 watcher 树（见 §2.4 代码）。又如 `BgTaskManager` 暴露线程数：

```cpp
// 典型模式（伪代码，简化自 bgtask_manager.cpp）
MF_WATCH( "bgTask/numRunningThreads",
        *BgTaskManager::instance().pInstance(),
        &BgTaskManager::numRunningThreads );
```

业务侧常见做法是把对象指针作为 `base` 传入，让 watcher 作用于"该对象的某个成员"，从而避免为每个对象实例都注册一份 watcher。

---

## 9. ID / 编码 / 杂项

### 9.1 unique_id

[unique_id.hpp](file:///workspace/src/lib/cstdmf/unique_id.hpp) 提供 128 位全局唯一标识符 `UniqueID`，用于实体、空间等需要跨进程唯一性的对象。支持字符串互转（`"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` 格式）与 `operator<`（可放入 `std::map`/`set`）。

### 9.2 md5 / base64

- [md5.hpp](file:///workspace/src/lib/cstdmf/md5.hpp)：MD5 哈希，用于资源校验、密码（与服务端协议配合）。
- [base64.h](file:///workspace/src/lib/cstdmf/base64.h)：Base64 编解码，用于二进制数据嵌入文本协议。

### 9.3 bwrandom —— Mersenne Twister 随机数

[bwrandom.hpp](file:///workspace/src/lib/cstdmf/bwrandom.hpp) 实现基于 Mersenne Twister 的随机数生成器，比 `rand()` 快约 1.5 倍且分布更好：

```cpp
// bwrandom.hpp:27-54
class BWRandom
{
public:
    BWRandom();
    explicit BWRandom(uint32 seed);
    BWRandom(uint32 const *seed, size_t size);

    uint32 operator()();
    int operator()(int min, int max);
    float operator()(float min, float max);

    static BWRandom& instance()
    {
        static BWRandom s_theRandom;
        return s_theRandom;
    }
protected:
    void init(uint32 seed);
    uint32 generate();
private:
    uint32                  mt[624]; // state vector
    int                     mti;
};

#define bw_random (BWRandom::instance())
```

状态向量 `mt[624]` 是标准 MT19937 设计。`bw_random` 宏提供全局单例访问。支持从种子数组初始化，便于从文件载入可复现随机序列。

### 9.4 bwversion / bw_util / string_utils

- [bwversion.hpp](file:///workspace/src/lib/cstdmf/bwversion.hpp)：引擎版本号查询。
- [bw_util.hpp](file:///workspace/src/lib/cstdmf/bw_util.hpp)：路径处理工具（`sanitisePath` 把 `\` 转 `/`、`formatPath` 保证末尾分隔符、`getFilePath` 取目录、`executableDirectory` 取可执行文件目录）。Win32 还提供 `bw_fopen`（处理宽字符路径）。
- [string_utils.hpp](file:///workspace/src/lib/cstdmf/string_utils.hpp)：`bw_stricmp` / `bw_snprintf` 等跨平台字符串函数。

### 9.5 binary_stream / binaryfile / memory_stream

[binary_stream.hpp](file:///workspace/src/lib/cstdmf/binary_stream.hpp) 是 BigWorld 的二进制序列化抽象，定义 `BinaryOStream` / `BinaryIStream` 接口，支持基本类型流式读写、长度前缀字符串、`writeStringLength` / `readStringLength` 等。watcher 协议、网络协议都基于它。

[binaryfile.hpp](file:///workspace/src/lib/cstdmf/binaryfile.hpp) 提供基于 `FILE*` 的二进制流实现；[memory_stream.hpp](file:///workspace/src/lib/cstdmf/memory_stream.hpp) 提供基于内存缓冲的 `MemoryOStream` / `MemoryIStream`，被 `WatcherPathRequestV2` 等大量使用。

### 9.6 concurrency / processor_affinity / locale / restart / slow_task

- [processor_affinity.hpp](file:///workspace/src/lib/cstdmf/processor_affinity.hpp)：CPU 亲和性设置，让 worker 线程绑定到特定核心。
- [locale.hpp](file:///workspace/src/lib/cstdmf/locale.hpp)：本地化字符串。
- [restart.hpp](file:///workspace/src/lib/cstdmf/restart.hpp)：进程重启支持（服务端热更新）。
- [slow_task.hpp](file:///workspace/src/lib/cstdmf/slow_task.hpp)：长任务的分帧执行抽象，避免阻塞主循环。

### 9.7 main_loop_task —— 主循环任务

[main_loop_task.hpp](file:///workspace/src/lib/cstdmf/main_loop_task.hpp) 定义 `MainLoopTask` 抽象基类与 `MainLoopTasks` 容器，是引擎主循环的"任务注册表"。每个 `MainLoopTask` 有 `tick()` 方法，主循环每帧依次调用所有已注册任务的 `tick`。客户端与服务端的 `App` 类都基于它构建。

---

## 10. 后台任务系统：bgtask_manager

[bgtask_manager.hpp](file:///workspace/src/lib/cstdmf/bgtask_manager.hpp) 是 BigWorld 的**后台 IO 任务调度器**，被 `resmgr`（资源加载）、`chunk`（地块加载）等 IO 密集型子系统使用。

### 10.1 核心抽象

`BackgroundTask` 是任务抽象基类，继承 `SafeReferenceCount`：

```cpp
// bgtask_manager.hpp:29-50
class BackgroundTask : public SafeReferenceCount
{
public:
    virtual void doBackgroundTask( BgTaskManager & mgr,
           BackgroundTaskThread * pThread )
    {
        this->doBackgroundTask( mgr );
    }
    virtual void doMainThreadTask( BgTaskManager & mgr ) {};
protected:
    virtual ~BackgroundTask() {};
    virtual void doBackgroundTask( BgTaskManager & mgr ) = 0;
};
```

每个任务分两阶段：
1. `doBackgroundTask`：在工作线程执行（IO、解压等）。
2. `doMainThreadTask`：在主线程执行收尾（如把资源挂到主线程数据结构）。任务可在 `doBackgroundTask` 末尾调用 `mgr.addMainThreadTask(this)` 把自己转入主线程队列。

`CStyleBackgroundTask` 提供与 C 函数指针的兼容包装：

```cpp
// bgtask_manager.hpp:63-80
class CStyleBackgroundTask : public BackgroundTask
{
public:
    typedef void (*FUNC_PTR)( void * );
    CStyleBackgroundTask( FUNC_PTR bgFunc, void * bgArg,
        FUNC_PTR fgFunc = NULL, void * fgArg = NULL );
    void doBackgroundTask( BgTaskManager & mgr );
    void doMainThreadTask( BgTaskManager & mgr );
private:
    FUNC_PTR bgFunc_; void * bgArg_;
    FUNC_PTR fgFunc_; void * fgArg_;
};
```

### 10.2 BgTaskManager —— 线程池调度器

`BgTaskManager` 是单例，管理一个线程池：

```cpp
// bgtask_manager.hpp:168-260
class BgTaskManager
{
public:
    enum { MIN = 0, LOW = 32, MEDIUM = 64, DEFAULT = MEDIUM, HIGH = 96, MAX = 128 };

    void startThreads( int numThreads, BackgroundThreadDataPtr pData = NULL );
    void stopAll( bool discardPendingTasks = true, bool waitForThreads = true );

    void addBackgroundTask( BackgroundTaskPtr pTask, int priority = DEFAULT );
    void addMainThreadTask( BackgroundTaskPtr pTask );

    bool isWorking();
    int numRunningThreads() const;
    int numUnstoppedThreads() const;

    static BgTaskManager & instance();
    static void fini();

    void onThreadFinished( BackgroundTaskThread * pThread ); // 主线程
    BackgroundTaskPtr pullBackgroundTask();                   // 工作线程
private:
    BackgroundTaskList bgTaskList_;        // 后台任务队列（带优先级）
    ForegroundTaskList fgTaskList_;        // 主线程任务队列
    ForegroundTaskList newTasks_;          // 新加入的主线程任务（待合并）
    SimpleMutex fgTaskListMutex_;
    int numRunningThreads_;
    int numUnstoppedThreads_;
    volatile long workingCount_;
};
```

后台任务队列内部用 `std::list<pair<int, BackgroundTaskPtr>>` + `SimpleMutex` + `SimpleSemaphore` 实现优先级阻塞队列：

```cpp
// bgtask_manager.hpp:231-244
class BackgroundTaskList
{
public:
    void push( BackgroundTaskPtr pTask, int priority = DEFAULT );
    BackgroundTaskPtr pull();
    bool isEmpty();
    void clear();
private:
    typedef std::list< std::pair< int, BackgroundTaskPtr > > List;
    List list_;
    SimpleMutex mutex_;
    SimpleSemaphore semaphore_;
};
```

`pull()` 阻塞在信号量上，`push()` 唤醒一个等待线程。优先级范围 0~128，`DEFAULT = 64`。

### 10.3 BackgroundTaskThread

每个工作线程是一个 `BackgroundTaskThread`（继承 `SimpleThread`），持有可选的 `BackgroundThreadData`（线程启动/退出钩子，用于初始化线程局部状态）：

```cpp
// bgtask_manager.hpp:101-116
class BackgroundTaskThread : public SimpleThread
{
public:
    BackgroundTaskThread( BgTaskManager & mgr, BackgroundThreadDataPtr pData );
    BackgroundThreadDataPtr pData() const            { return pData_; }
    void pData( BackgroundThreadDataPtr pData )      { pData_ = pData; }
private:
    static void s_start( void * arg );
    void run();
    BgTaskManager & mgr_;
    BackgroundThreadDataPtr pData_;
};
```

`run()` 循环 `pullBackgroundTask` → 调用 `doBackgroundTask` → 必要时把主线程任务加入 `newTasks_`。`tick()` 在主线程被调用，把 `newTasks_` 合并到 `fgTaskList_` 并执行 `doMainThreadTask`。

### 10.4 与 job_system 的区别

| 维度 | `bgtask_manager` (cstdmf) | `job_system` (lib/job_system) |
|---|---|---|
| 任务类型 | IO 密集（文件、解压） | CPU 密集（动画、物理） |
| 线程模型 | 通用线程池，任务可阻塞 | 固定 worker + 消费线程，无锁命令缓冲 |
| 通信方式 | `SimpleMutex` + `SimpleSemaphore` | `CommandBuffer` 双缓冲 |
| 数据流 | 任务自带数据 | SyncBlock/Job 链式背补丁 |
| 主线程回调 | 支持（`addMainThreadTask`） | 通过 consumption 回调 |
| 适用场景 | resmgr/chunk 资源加载 | 客户端帧内并行计算 |

详见附录 A。

### 10.5 典型使用模式

```cpp
// 简化示例：异步加载资源
class LoadResourceTask : public BackgroundTask
{
public:
    LoadResourceTask(const std::string& path) : path_(path) {}
    void doBackgroundTask(BgTaskManager& mgr)
    {
        // 工作线程：读文件、解压
        data_ = readFile(path_);
        mgr.addMainThreadTask(this);  // 转主线程收尾
    }
    void doMainThreadTask(BgTaskManager& mgr)
    {
        // 主线程：把数据挂到资源管理器
        ResourceMgr::instance().onLoaded(path_, data_);
    }
private:
    std::string path_;
    BinaryData data_;
};

// 注册
BgTaskManager::instance().addBackgroundTask(
    new LoadResourceTask("models/hero.model") );
```

---

## 11. 文件结构总览

按功能分组的文件清单：

| 分组 | 文件 |
|---|---|
| 容器 | avector.hpp, vectornodest.hpp, safe_fifo.hpp, static_array.hpp, cache.hpp, stringmap.hpp |
| 内存分配 | pool_allocator.hpp, aalloc.hpp, aligned.hpp |
| 智能指针 | smartpointer.hpp, shared_ptr.hpp |
| 单例/生命周期 | singleton.hpp, init_singleton.hpp, named_object.hpp, intrusive_object.hpp, list_node.hpp, fini_job.hpp |
| 内存诊断 | memory_counter.hpp/cpp, memory_tracker.hpp/cpp, memory_trace.hpp/cpp, stack_tracker.hpp/cpp, resource_counters.hpp/cpp |
| 调试/日志 | debug.hpp/cpp, dprintf.hpp/cpp, message_box.hpp/cpp, critical_message_box.hpp/cpp, diary.hpp/cpp/ipp |
| 性能 | profiler.hpp/cpp, profile.hpp/cpp, dogwatch.hpp/cpp/ipp |
| 时间 | timestamp.hpp/cpp, time_queue.hpp/cpp/ipp, timer_handler.hpp, date_time_utils.hpp/cpp |
| 并发 | concurrency.hpp/cpp, processor_affinity.hpp/cpp |
| Watcher | watcher.hpp/cpp, watcher_path_request.hpp/cpp |
| 后台任务 | bgtask_manager.hpp/cpp |
| 主循环 | main_loop_task.hpp/cpp |
| ID/编码 | unique_id.hpp/cpp, md5.hpp/cpp, base64.h/cpp, bwrandom.hpp/cpp |
| 工具 | bwversion.hpp/cpp, bw_util.hpp/cpp, string_utils.hpp/cpp, locale.hpp/cpp |
| 流 | binary_stream.hpp/cpp/ipp, binaryfile.hpp/cpp, memory_stream.hpp/ipp |
| 配置 | config.hpp, stdmf.hpp |
| 其它 | guard.hpp, value_or_null.hpp, bw_functor.hpp, restart.hpp/cpp, slow_task.hpp, blob_or_null.hpp/cpp |

---

## 12. 小结

`cstdmf` 是 BigWorld 引擎的"地基"，其设计有几个鲜明特点：

1. **跨平台优先**：几乎所有功能都有 Win32/Linux/Xbox360/PS3 多份实现，通过统一抽象层暴露。
2. **零开销抽象**：通过 `ENABLE_*` 宏，调试/诊断功能在生产构建中可完全编译消失。
3. **侵入式设计**：智能指针、侵入式链表、`Singleton` 基类等都要求对象配合设计，换取性能与零额外分配。
4. **Watcher 优先**：从 `PoolAllocator` 到 `MemTracker` 到 `BgTaskManager`，几乎所有子系统都自觉把自己的内部状态通过 watcher 暴露，形成"运行时可观测"的工程文化。
5. **slot 栈式追踪**：`MemTracker` 和 `Profiler` 都用线程局部 slot 栈做归因，既能嵌套又能跨线程独立。

理解 `cstdmf` 是理解 BigWorld 其它所有模块的前提——后续的 `math`、`network`、`resmgr`、各 server 进程都建立在这些原语之上。
