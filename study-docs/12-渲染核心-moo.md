# BigWorld Engine 2.0.1 渲染核心（moo）

> 源码目录：`/workspace/src/lib/moo/`
> 模块定位：BigWorld **客户端最核心**的渲染库。"moo"是 BigWorld 自创的命名空间名（业内传闻是 "My Object Oriented"，但代码本身没有官方释义），它在 D3D9 之上提供了一整套**资源生命周期管理 / Effect/Shader 包装 / 网格资产（Visual）/ 骨骼动画 / 光照 / 渲染状态缓存 / 延迟渲染通道**的抽象。所有上层模块——`terrain`、`chunk`、`model`、`romp`（环境 / 后处理）、`particle`、`gui`——都直接调用 moo。
> 模块边界：下层是 Windows D3D9（`d3d9.h` / `d3dx9effect.h`）；上层通过 `BWResource`（[resmgr](file:///workspace/src/lib/resmgr/)）加载所有磁盘资源，通过 `cstdmf` 的 `BgTaskManager` / `DogWatch` / `Profiler` 提供后台任务与性能计数器，通过 `math` 库提供 `Matrix` / `Vector3` / `BoundingBox`。
> 编号：12（接 `11-服务端进程-mgr-dbmgr-loginapp-reviver.md`；客户端侧的渲染基础，与 `13-模型系统-model.md`、`05-地形系统-terrain.md`、`14-粒子与后处理-particle-postfx.md` 强关联）。

---

## 1. 模块定位

moo 库解决九件事：

1. **D3D9 设备的全局封装**：通过单例 `RenderContext`（即全局 `g_RC`，访问器 `Moo::rc()`）持有 `IDirect3DDevice9*`，管理 backbuffer、PresentParameters、显示模式切换、设备丢失 / 恢复、Gamma、VSync、窗口 / 全屏切换。
2. **渲染状态缓存**：把 `SetRenderState` / `SetTextureStageState` / `SetSamplerState` / `SetTexture` / `SetVertexShader` / `SetPixelShader` / `SetVertexDeclaration` 全部走 `RenderContext::setXxx`，命中缓存就不下发 D3D 调用，配合 `cacheValidityId_` 在 device reset 时整体失效。
3. **GPU 资源管理**：纹理（`BaseTexture` 派生类）、VertexBuffer、IndexBuffer、RenderTarget、CubeRenderTarget、VertexDeclaration、DynamicVB/IB（环形 ring-buffer）。所有这些资源都通过 `DeviceCallback` 接收 device lost / reset / created 通知。
4. **Effect / Shader 包装**：基于 D3DX Effect，`ManagedEffect` 包装 `ID3DXEffect`，`EffectMaterial` 在其上挂属性（`EffectProperty`），`EffectStateManager` 把 Effect 内部产生的状态变更也走 `RenderContext` 的缓存路径。`ShaderSet` / `ShaderManager` 是固定函数时代的 vertex shader 选择器。
5. **网格资产系统（Visual）**：`.visual` 文件解析、`RenderSet`（按骨骼分组）、`Geometry`（顶点 + 索引 + 材质组）、BSP 树（碰撞与可见性）、Portal（chunk 间可见性）。配合 `VisualManager` 缓存、`VisualCompound` 实例合并、`VisualChannels` 延迟渲染通道（solid / sorted / shimmer / distortion）。
6. **骨骼与动画**：`Node` 树（骨骼）、`NodeCatalogue` 全局共享、`Animation` / `AnimationChannel`（含插值 / 离散 / 形变 / 流式 / Cue / TranslationOverride 六种通道）、`SoftwareSkinner` CPU 蒙皮、`MorphVertices` 形变顶点。
7. **光照**：`LightContainer` 桶 + `OmniLight` / `SpotLight` / `DirectionalLight`，支持世界空间 / 模型空间变换、地形光照预计算（`getTerrainLight` 返回 `Vector4[3..4]`）、固定函数管线提交。
8. **渲染状态设置器**：`CameraPlanesSetter` / `ScissorsSetter` / `ViewportSetter` 等 RAII 作用域对象；`FogHelper` 处理 table fog vs vertex fog 的硬件差异。
9. **多线程与资源加载上下文**：`ResourceLoadContext` 跟踪"当前在加载谁"用于错误报告；`StreamedDataCache` 处理 `.anca` 流式动画数据；`DX` 命名空间提供 device wrapper（延迟命令缓冲）以支持渲染线程。

### 1.1 模块依赖关系图

```
            ┌────────────────────────────────────────────┐
            │   客户端 App / Editor (romp, appmgr)        │
            │   EnviroMinder / ChunkManager::draw        │
            └─────────────────────┬──────────────────────┘
                                  │  draw() / setWorld / setView / setProjection
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │                   Moo::RenderContext (g_RC)                 │
   │   device_ / camera_ / view_ / projection_ / lightContainer_ │
   │   rsCache_ / tssCache_ / sampCache_ / textureCache_          │
   │   beginScene / endScene / present / drawIndexedPrimitive    │
   └─────┬──────────────┬──────────────┬──────────────┬─────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Texture  │   │ Effect / │   │ Visual / │   │ Geometry │
   │ Manager  │   │ Material  │   │ Manager  │   │ VB / IB  │
   │  + Base  │   │ Manager   │   │  + Node  │   │ + Verts  │
   │ Texture  │   │  + Managed│   │  + Anim  │   │ + Prim   │
   └────┬─────┘   └─────┬─────┘   └─────┬────┘   └────┬─────┘
        │               │               │             │
        ▼               ▼               ▼             ▼
   ┌──────────────────────────────────────────────────────────┐
   │     DX::Device (IDirect3DDevice9, 可能被 DeviceWrapper 包装)  │
   │     DX::Texture / VertexBuffer / IndexBuffer / Surface      │
   └──────────────────────────────────────────────────────────────┘
        │                                              ▲
        │ resource load                                │ device lost/reset
        ▼                                              │
   ┌──────────────────────┐              ┌──────────────────────────┐
   │   BWResource (resmgr)│              │   DeviceCallback list    │
   │   .tga/.dds/.visual/ │              │  (Texture/RT/VB/Effect…) │
   │   .primitive/.fx      │              └──────────────────────────┘
   └──────────────────────┘
```

### 1.2 命名空间与文件组织

moo 主要使用两个命名空间：

- `Moo`：所有业务类（`RenderContext` / `BaseTexture` / `Visual` / `EffectMaterial` / `LightContainer` …）。
- `DX`：与 D3D 直接交互的薄层，定义 COM 接口的 typedef、device wrapper、错误字符串化、表面尺寸计算等。见 [moo_dx.hpp](file:///workspace/src/lib/moo/moo_dx.hpp)。

```cpp
// moo_dx.hpp:57
namespace DX
{
    typedef IDirect3D9             Interface;
    typedef IDirect3DDevice9       Device;
    typedef IDirect3DResource9     Resource;
    typedef IDirect3DBaseTexture9  BaseTexture;
    typedef IDirect3DTexture9      Texture;
    typedef IDirect3DCubeTexture9  CubeTexture;
    typedef IDirect3DSurface9      Surface;
    typedef IDirect3DVertexBuffer9 VertexBuffer;
    typedef IDirect3DIndexBuffer9  IndexBuffer;
    typedef IDirect3DPixelShader9  PixelShader;
    typedef IDirect3DVertexShader9 VertexShader;
    typedef IDirect3DVertexDeclaration9 VertexDeclaration;
    typedef IDirect3DQuery9        Query;
    typedef D3DLIGHT9              Light;
    typedef D3DVIEWPORT9           Viewport;
    typedef D3DMATERIAL9           Material;
    // ...
}
```

文件组织遵循 `.hpp` 声明 / `.cpp` 实现 / `.ipp` 内联三件套，`.ipp` 由 `.hpp` 末尾在定义了 `CODE_INLINE` 时 include。这套约定让头文件既能给外部用，又能把热路径内联到调用方。

---

## 2. 核心管理器与上下文

### 2.1 启动入口：`init.hpp` / `init.cpp`

moo 的生命周期从一对全局函数开始（[init.hpp](file:///workspace/src/lib/moo/init.hpp)）：

```cpp
// init.hpp:16
namespace Moo
{
    bool init();
    void fini();
}
```

实现非常简单（[init.cpp](file:///workspace/src/lib/moo/init.cpp)）：创建全局 `RenderContext` 并调用其 `init()`，`fini()` 反之销毁。

```cpp
// init.cpp:24
RenderContext*  g_RC = NULL;

bool init()
{
    BW_GUARD;
    if ( !s_initialised )
    {
        MF_ASSERT_DEV( g_RC == NULL );
        if( g_RC == NULL )
            g_RC = new RenderContext();
        s_initialised = g_RC->init();
        return s_initialised;
    }
    return true;
}
```

注意：`init()` **只**创建 `RenderContext`，并不创建 D3D 设备。真正的 device 创建发生在 `RenderContext::createDevice()`，由上层（通常是 `appmgr` / `romp`）在创建主窗口后显式调用。

### 2.2 全局渲染上下文：`RenderContext`

这是 moo 最大的类，约 580 行头文件（[render_context.hpp](file:///workspace/src/lib/moo/render_context.hpp)），承担四类职责：

#### 2.2.1 设备与显示模式

| 成员 / 方法 | 作用 |
|----|----|
| `d3d_` | `ComObjectWrap<DX::Interface>`，即 `IDirect3D9*`，`init()` 中由 `Direct3DCreate9(D3D_SDK_VERSION)` 创建 |
| `device_` | `ComObjectWrap<DX::Device>`，真正的 D3D 设备，可能被 `DX::DeviceWrapper` 包装 |
| `devices_` | `std::vector<DeviceInfo>`，枚举到的所有适配器及其显示模式 |
| `createDevice(hWnd, deviceIndex, modeIndex, windowed, wantStencil, windowedSize, hideCursor, forceRef, useWrapper)` | 创建 D3D 设备，详见 §8.1 |
| `changeMode(modeIndex, windowed, testCooperative, backBufferWidthOverride)` | 切换显示模式（窗口↔全屏） |
| `resetDevice()` | 设备丢失后 reset |
| `checkDevice(bool* reset)` / `checkDeviceYield(bool* reset)` | 检测设备协作级别 |
| `releaseUnmanaged()` / `createUnmanaged()` | 触发所有 `DeviceCallback::deleteAllUnmanaged` / `createAllUnmanaged` |

`DeviceInfo` 结构（render_context.hpp:54）记录每个适配器的标识、Caps、显示模式列表、兼容性标志：

```cpp
// render_context.hpp:54
struct DeviceInfo
{
    D3DADAPTER_IDENTIFIER9         identifier_;
    D3DCAPS9                      caps_;
    uint32                        adapterID_;
    bool                          windowed_;
    D3DDISPLAYMODE                windowedDisplayMode_;
    std::vector< D3DDISPLAYMODE > displayModes_;
    uint32                        compatibilityFlags_;  // NOOVERWRITE / NVIDIA / ATI
};
```

#### 2.2.2 相机与变换矩阵

RenderContext 持有一个 `Camera` 和四类矩阵：

```cpp
// render_context.hpp:478
Camera      camera_;
Matrix      projection_;
Matrix      view_;
Matrix      viewProjection_;
Matrix      lastViewProjection_;
Matrix      invView_;
AVectorNoDestructor< Matrix > world_;   // world 矩阵栈
```

`world_` 是一个 `AVectorNoDestructor<Matrix>` 栈，`push()` / `pop()` / `reset()` 管理世界矩阵嵌套；`preMultiply(m)` / `postMultiply(m)` 在栈顶左 / 右乘。`updateProjectionMatrix(detectAspectRatio)` 会根据 backbuffer 宽高比修正投影矩阵，`updateViewTransforms()` 重算 `viewProjection_` 与 `invView_`。

#### 2.2.3 渲染状态缓存

这是 moo 性能的关键。RenderContext 内部维护四张缓存表：

```cpp
// render_context.hpp:536
RSCacheEntry      rsCache_[D3DRS_MAX];                       // 渲染状态
TSSCacheEntry     tssCache_[D3DFFSTAGES_MAX][D3DTSS_MAX];   // 纹理阶段状态（固定函数）
SampCacheEntry    sampCache_[D3DSAMPSTAGES_MAX][D3DSAMP_MAX]; // 采样器状态
TextureCacheEntry textureCache_[D3DSAMPSTAGES_MAX];        // 当前绑定的纹理
uint              cacheValidityId_;                          // 失效版本号
```

每个 `setRenderState(state, value)` 都先查表，若值未变则直接返回；`invalidateStateCache()` 自增 `cacheValidityId_`，下次设置强制刷新。`initRenderStates()` 在 `createDevice` 后调用一次，把缓存填上设备默认值。

```cpp
// render_context.hpp:308
uint32 setRenderState(D3DRENDERSTATETYPE state, uint32 value);
uint32 setTextureStageState( uint32 stage, D3DTEXTURESTAGESTATETYPE type, uint32 value );
uint32 setSamplerState( uint32 stage, D3DSAMPLERSTATETYPE type, uint32 value);
uint32 setTexture(DWORD stage, DX::BaseTexture* pTexture);
uint32 setVertexShader( DX::VertexShader* pVS );
uint32 setPixelShader( DX::PixelShader* pPS );
HRESULT setIndices( DX::IndexBuffer* pIB );
uint32 setVertexDeclaration( DX::VertexDeclaration* pVD );
uint32 setFVF( uint32 fvf );
```

返回值是 `uint32`，非 0 表示"实际下发了 D3D 调用"（用于性能统计）。`pushRenderState(state)` / `popRenderState()` 提供作用域保存 / 恢复。

#### 2.2.4 绘制与场景管理

```cpp
// render_context.hpp:350
HRESULT beginScene();
HRESULT endScene();
HRESULT present();
uint32  drawIndexedPrimitive( D3DPRIMITIVETYPE type, INT baseVertexIndex,
                              UINT minIndex, UINT numVertices,
                              UINT startIndex, UINT primitiveCount );
uint32  drawPrimitive( D3DPRIMITIVETYPE primitiveType, UINT startVertex, UINT primitiveCount );
uint32  drawPrimitiveUP( ... );
uint32  drawIndexedPrimitiveUP( ... );
```

`beginScene` / `endScene` 是 D3D 的场景边界，`beginSceneCount_` 跟踪嵌套调用。`present()` 调用 `device_->Present()`，根据 `waitForVBL_` / `tripleBuffering_` 调整呈现参数。

#### 2.2.5 其他全局状态

- **Fog**：`fogEnabled_` / `fogColour_` / `fogNear_` / `fogFar_`，由 `romp` 的 `EnviroMinder` 设置。
- **光照**：`lightContainer_`（漫反射）/ `specularLightContainer_`（高光），都是 `LightContainerPtr`。
- **LOD**：`lodValue_` / `lodFar_` / `lodPower_` / `zoomFactor_`，`Visual` 根据 `lodValue()` 选择 mesh 细节。
- **时间戳**：`currentFrame_` / `frameTimestamp()` / `nextFrame()`，`frameDrawn(frame)` 判断某帧是否已绘制完。
- **RenderTarget 栈**：`renderTargetStack_` 嵌套保存 RT + viewport + camera，`pushRenderTarget()` / `popRenderTarget()`。
- **Occlusion Query**：`createOcclusionQuery()` / `beginQuery()` / `endQuery()` / `getQueryResult()`。
- **统计**：`liveProfilingData_` / `lastFrameProfilingData_`（`DrawcallProfilingData`：drawcall 数 / primitive 数）。
- **预加载**：`addPreloadResource(IDirect3DResource9*)` / `preloadDeviceResources(timeLimitMs)`，把 managed pool 资源按时间预算逐帧 `PreLoad` 到显存。
- **设备丢失保护**：`d3dxCreateMutex_` 防止加载线程在 stateblock 录制期间调用 `D3DXCreate*`。

#### 2.2.6 访问器

全局只有一个 `RenderContext`，通过 `Moo::rc()` 取：

```cpp
// render_context.hpp:580
RenderContext& rc();
```

`rc()` 几乎出现在 moo 内每个 `.cpp` 里——它是访问 device、矩阵、灯光、fog、时间的统一入口。

### 2.3 设备回调：`DeviceCallback`

D3D 设备丢失（lost）/ 恢复（reset）/ 销毁时，所有持有 GPU 资源的对象都需要被通知。moo 用一个全局链表 + 虚函数接口解决（[device_callback.hpp](file:///workspace/src/lib/moo/device_callback.hpp)）：

```cpp
// device_callback.hpp:27
class DeviceCallback
{
public:
    typedef std::list< DeviceCallback* > CallbackList;

    DeviceCallback();
    ~DeviceCallback();

    virtual void deleteUnmanagedObjects( );
    virtual void createUnmanagedObjects( );
    virtual void deleteManagedObjects( );
    virtual void createManagedObjects( );

    static void deleteAllUnmanaged( );
    static void createAllUnmanaged( );
    static void deleteAllManaged( );
    static void createAllManaged( );

    static void fini();
private:
    static CallbackList* callbacks_;
};
```

构造函数把自己加入 `callbacks_`，析构移除。`RenderContext::releaseUnmanaged()` 调用 `DeviceCallback::deleteAllUnmanaged()`，遍历链表调用每个对象的 `deleteUnmanagedObjects()`；`createUnmanaged()` 对称。

**Managed vs Unmanaged**：D3D 的 `D3DPOOL_MANAGED` 资源由 D3D 自己管理显存换页，device reset 时不需要重建；`D3DPOOL_DEFAULT` 资源（默认 render target、动态 VB/IB）则必须由应用重建。所以：

- `deleteUnmanagedObjects` / `createUnmanagedObjects`：处理 `D3DPOOL_DEFAULT` 资源，每次 device lost/reset 都调用。
- `deleteManagedObjects` / `createManagedObjects`：处理 `D3DPOOL_MANAGED` 资源，仅在显存紧张需要主动释放时调用。

`GenericUnmanagedCallback`（device_callback.hpp:60）是个便捷适配器，把两个 C 风格函数指针包成 `DeviceCallback`。

### 2.4 渲染上下文回调：`RenderContextCallback`

比 `DeviceCallback` 更细粒度，专门在 `RenderContext::fini()` 时通知"该释放 RenderContext 关联资源了"（[render_context_callback.hpp](file:///workspace/src/lib/moo/render_context_callback.hpp)）：

```cpp
// render_context_callback.hpp:24
class RenderContextCallback
{
public:
    virtual void renderContextFini() = 0;
    static void fini();
private:
    static Callbacks s_callbacks_;
};
```

例如 `QuickAlloc<N>`（effect_state_manager.hpp:66）就继承它，在 `renderContextFini()` 里清空每帧分配池。

### 2.5 D3D 交互层：`moo_dx.hpp` / `moo_dx.cpp`

`DX` 命名空间除了 typedef，还提供：

- `createDeviceWrapper(IDirect3DDevice9*)`：返回一个 `DeviceWrapper`，它实现 `IDirect3DDevice9` 接口但把所有调用记录到 command buffer，供多线程渲染（[moo_dx.hpp:78](file:///workspace/src/lib/moo/moo_dx.hpp)）。
- `createEffectWrapperStateManager()` / `createEffectWrapper(ID3DXEffect*, name)`：把 Effect 也接入 wrapper。
- `surfaceSize(width, height, format)` / `textureSize(texture)`：计算表面 / 纹理的内存占用。
- `errorAsString(HRESULT)`：D3D 错误码转字符串。
- `newBlock()` / `newFrame()` / `setCurrentFrame()` / `getCurrentFrame()`：wrapper 的帧边界管理。
- `setWrapperFlags(flags)`：`WRAPPER_FLAG_IMMEDIATE_LOCK` / `WRAPPER_FLAG_DEFERRED_LOCK` / `WRAPPER_FLAG_ZERO_TEXTURE_LOCK` / `WRAPPER_FLAG_QUERY_ISSUE_FLUSH`，控制 wrapper 的锁行为。
- `addSecondaryThread()` / `addFakeMainThread()`：让次线程也能往 wrapper 提交命令。

`DeviceWrapper`（moo_dx.hpp:186）是一个巨大的类，逐个实现 `IDirect3DDevice9` 的所有方法，内部把调用转发给真实 device 或写入 command buffer。这套机制是 BigWorld 多线程渲染的基础，但默认 `useWrapper=false`（直接同步调用）。

---

## 3. GPU 资源管理

### 3.1 COM 智能指针：`com_object_wrap.hpp`

所有 D3D COM 对象都用 `ComObjectWrap<T>` 包装（[com_object_wrap.hpp](file:///workspace/src/lib/moo/com_object_wrap.hpp)）：

```cpp
// com_object_wrap.hpp:32
template <typename COMOBJECTTYPE>
struct ComObjectWrap
{
    ComObjectWrap();
    ComObjectWrap(ComObject *object);
    ComObjectWrap(ComObjectWrap const &other);
    ~ComObjectWrap();

    ComObjectWrap &operator=(ComObjectWrap const &other);
    template<typename OTHER>
        ComObjectWrap &operator=(ComObjectWrap<OTHER> const &other);  // QueryInterface 转型

    bool hasComObject() const;
    void pComObject(ComObject *object);
    ComObjectPtr pComObject() const;
    ComObjectPtr operator->() const;
    void addAlloc(std::string const &desc);   // 资源计数器登记
private:
    ComObject *comObject_;
};
```

它做的事：构造时 `AddRef`，析构时 `Release`，赋值时先 Release 旧的再 AddRef 新的。`addAlloc(desc)` 在 `ENABLE_RESOURCE_COUNTERS` 时把这块内存登记到全局资源计数器，方便排查显存泄漏。整个 moo 里 `ComObjectWrap<DX::Texture>` / `ComObjectWrap<DX::VertexBuffer>` 等到处都是。

### 3.2 纹理层级

纹理类形成一个继承层级（基类在 [base_texture.hpp](file:///workspace/src/lib/moo/base_texture.hpp)）：

```
SafeReferenceCount
   └─ BaseTexture (abstract)
         ├─ ManagedTexture      .dds/.tga/.bmp 文件纹理（最常用）
         ├─ ImageTexture<T>     内存 Image<Texture> 包装
         ├─ SysMemTexture       SysMem pool 纹理（CPU 读写）
         ├─ AnimatingTexture    帧序列动画纹理
         ├─ RenderTarget        渲染目标纹理
         └─ CubeRenderTarget    立方体环境贴图渲染目标
```

#### 3.2.1 `BaseTexture`

抽象基类（base_texture.hpp:32），定义所有纹理的公共接口：

```cpp
// base_texture.hpp:32
class BaseTexture : public SafeReferenceCount
{
public:
    explicit BaseTexture(const std::string& allocator = "texture/unknown base texture");
    virtual ~BaseTexture();

    virtual DX::BaseTexture* pTexture( ) = 0;
    virtual uint32           width( ) const = 0;
    virtual uint32           height( ) const = 0;
    virtual D3DFORMAT        format( ) const = 0;
    virtual uint32           textureMemoryUsed( ) = 0;
    virtual const std::string& resourceID( ) const;

    virtual HRESULT load()      { return 0; }
    virtual HRESULT release()  { return 0; }
    virtual HRESULT reload( ) { return 0; }
    virtual HRESULT reload(const std::string & resourceID) { return 0; }

    virtual uint32 maxMipLevel() const { return 0; }
    virtual void   maxMipLevel( uint32 level ) {}
    virtual uint32 mipSkip() const { return 0; }
    virtual void   mipSkip( uint32 mipSkip ) {}
    virtual bool   isCubeMap() { return false; }
    virtual bool   isAnimated() { return false; }

protected:
    static uint32 textureMemoryUsed( DX::Texture* );
    void addToManager();
    void delFromManager();
};
```

`addToManager()` / `delFromManager()` 在构造 / 析构时把自己登记到 `TextureManager::textures_`，所以所有 `BaseTexture` 派生类都自动进缓存表。`textureMemoryUsed(DX::Texture*)` 静态方法用 `D3DLOCKED_RECT` + 表面尺寸算实际显存占用。

#### 3.2.2 `ManagedTexture`

最常用的纹理类型（[managed_texture.hpp](file:///workspace/src/lib/moo/managed_texture.hpp)），从磁盘 `.dds` / `.tga` / `.bmp` 加载，托管在 `D3DPOOL_MANAGED`：

```cpp
// managed_texture.hpp:33
class ManagedTexture : public BaseTexture
{
public:
    ManagedTexture( const std::string& resourceID, uint32 w, uint32 h, int nLevels,
                    DWORD usage, D3DFORMAT fmt, const std::string& allocator );
    ManagedTexture( const std::string& resourceID, bool mustExist, int mipSkip = 0,
                    bool noResize = false, bool noFilter = false, ... );
    // ...
private:
    uint32             width_, height_;
    D3DFORMAT          format_;
    uint32             mipSkip_;
    bool               loadedFromDisk_;
    bool               noResize_, noFilter_;
    uint32             textureMemoryUsed_;
    uint32             originalTextureMemoryUsed_;
    DX::BaseTexture*   texture_;        // 实际 D3D 对象（弱引用，见下）
    ComObjectWrap< DX::Texture >     tex_;
    ComObjectWrap< DX::CubeTexture > cubeTex_;
    std::string        resourceID_;
    std::string        qualifiedResourceID_;
    bool               valid_;
    bool               failedToLoad_;
    bool               cubemap_;
    uint32             localFrameTimestamp_;
    static uint32      frameTimestamp_;
    friend class TextureManager;
};
```

关键设计：
- 构造函数是 private，只能由 `TextureManager::get()` 通过 friend 关系创建——保证缓存唯一性。
- `texture_` 是裸指针，指向 `tex_` 或 `cubeTex_` 内部对象，避免每次 `pTexture()` 都判 cubemap。
- `load(bool mustExist)` 真正读盘，支持 `mipSkip`（跳过低 mip 节省显存）、`noResize`（不缩放到 2 的幂）、`noFilter`（不加 mipmap）。
- `tick()` 静态方法每帧调用，处理纹理的按需加载与显存换页。
- `s_accErrs` / `s_accErrStr` 是错误累积开关，编辑器批量转换纹理时用。

#### 3.2.3 `ImageTexture<T>`

把内存中的 `Image<PixelType>` 包装成纹理（[image_texture.hpp](file:///workspace/src/lib/moo/image_texture.hpp)）：

```cpp
// image_texture.hpp:23
template<typename PIXELTYPE>
class ImageTexture : public BaseTexture
{
public:
    ImageTexture(uint32 width, uint32 height,
                 D3DFORMAT format = recommendedFormat(),
                 uint32 mipLevels = 0);
    void lock(uint32 level = 0);
    ImageType &image();
    void unlock();
    static D3DFORMAT recommendedFormat();
private:
    ImageType   image_;
    ComObjectWrap<DX::Texture> texture_;
    uint32      lockCount_, lockLevel_;
};

typedef ImageTexture<Moo::PackedColour> ImageTextureARGB;
typedef ImageTexture<uint8>              ImageTexture8;
typedef ImageTexture<uint16>             ImageTexture16;
```

`lock()` 把 `Image` 数据 lock 到 D3D texture，`unlock()` 写回。用于粒子、GUI、地形 blend 等需要在 CPU 修改像素的场景。

#### 3.2.4 `SysMemTexture`

`D3DPOOL_SYSTEMMEM` 纹理，专门用于跨设备共享或 CPU 频繁读写（参见 sys_mem_texture.hpp）。`updateTexture` 把它 blit 到 GPU managed 纹理。

#### 3.2.5 `AnimatingTexture`

帧序列动画纹理（[animating_texture.hpp](file:///workspace/src/lib/moo/animating_texture.hpp)）：

```cpp
// animating_texture.hpp:25
class AnimatingTexture : public BaseTexture
{
public:
    AnimatingTexture( const std::string& resourceID = std::string(), ... );
    void open( const std::string& resourceID );
    static void tick( float dTime );
    bool isAnimated() { return true; }
private:
    std::vector< BaseTexturePtr > textures_;   // 各帧
    float fps_;
    uint64 lastTime_;
    float animFrame_;
    uint32 frameTimestamp_;
    static uint64 currentTime_;
};
```

`open()` 解析类似 `mytex_00.dds mytex_01.dds ...` 的序列或一张大图分块，`tick(dTime)` 推进 `animFrame_`，`pTexture()` 返回当前帧的 `BaseTexture`。`TextureManager::get(allowAnimation=true)` 检测到资源 ID 含特定后缀时自动返回 `AnimatingTexture`。

### 3.3 `TextureManager`

纹理缓存与生命周期单例（[texture_manager.hpp](file:///workspace/src/lib/moo/texture_manager.hpp)）：

```cpp
// texture_manager.hpp:40
class TextureManager : public Moo::DeviceCallback
{
public:
    typedef std::map< std::string, BaseTexture * > TextureMap;

    static TextureManager* instance();
    HRESULT initTextures( );
    HRESULT releaseTextures( );

    BaseTexturePtr get( const std::string & resourceID,
                        bool allowAnimation = true, bool mustExist = true,
                        bool loadIfMissing = true,
                        const std::string& allocator = "texture/unknown texture" );
    BaseTexturePtr get( DataSectionPtr data, const std::string & resourceID, ... );
    BaseTexturePtr getUnique( const std::string& resourceID, ... );  // 不进缓存
    BaseTexturePtr getSystemMemoryTexture( const std::string& resourceID );

    void reloadAllTextures();
    void recalculateDDSFiles();
    void reloadMipMaps(ProgessFunc progFunc);
    void setFormat( const std::string & fileName, D3DFORMAT format );

    static bool writeDDS( DX::BaseTexture* texture, const std::string& ddsName,
                          D3DFORMAT format, int numDestMipLevels = 0 );
    static TextureDetailLevelPtr detailLevel( const std::string& originalName );
    static bool convertToDDS( ... );
    void initDetailLevels( DataSectionPtr pDetailLevels, bool accumulate = false );

    // DeviceCallback
    void deleteManagedObjects( );
    void createManagedObjects( );

    static const char ** validTextureExtensions();
private:
    TextureMap textures_;
    SimpleMutex texturesLock_;
    TextureDetailLevels detailLevels_;
    bool fullHouse_;   // 不再接受新条目
    GraphicsSetting::GraphicsSettingPtr qualitySettings_;
    GraphicsSetting::GraphicsSettingPtr compressionSettings_;
    GraphicsSetting::GraphicsSettingPtr filterSettings_;
    typedef std::pair<BaseTexture *, std::string> TextureStringPair;
    std::vector<TextureStringPair> dirtyTextures_;
};
```

#### 3.3.1 缓存机制

`get(resourceID)` 流程：
1. `prepareResourceName()` 标准化路径（去扩展、转小写、`canonicalTextureName()`）。
2. `find()` 在 `textures_` 查；命中则返回 `BaseTexturePtr`。
3. 未命中则 `prepareResource()`：根据 `texture_detail_levels.xml` 决定是否转 DDS、压缩、降 mip，可能生成 `.dds` 缓存。
4. `new ManagedTexture(...)` 创建对象，`addInternal()` 插入 `textures_`，返回。

`getUnique()` 跳过缓存——用于需要单独修改纹理内容的场景（如动态生成的 terrain LOD 纹理）。`fullHouse(true)` 把 `fullHouse_` 置位，之后 `get()` 不再创建新对象，仅返回已缓存的——用在关卡切换预热阶段，避免运行时卡顿。

#### 3.3.2 `TextureDetailLevel`

`texture_detail_levels.xml` 配置（texture_manager.hpp:151）按文件名前缀 / 后缀 / 包含子串匹配，决定该纹理的：

```cpp
// texture_manager.hpp:151
class TextureDetailLevel : public SafeReferenceCount
{
public:
    void init( DataSectionPtr pSection );
    bool check( const std::string& resourceID );   // 文件名是否匹配本规则

    D3DFORMAT format() const;             // 目标格式（如 D3DFMT_A8R8G8B8）
    D3DFORMAT formatCompressed() const;    // 压缩格式（如 D3DFMT_DXT5）
    uint32    lodMode() const;            // 0=正常 1=降 mip 2=不缩放
    uint32    mipCount() const;            // 限制 mip 数
    uint32    mipSize() const;             // 限制最大尺寸
    bool      horizontalMips() const;      // 横向排列 mip（用于纹理图集）
    bool      noResize() const;
    bool      noFilter() const;
private:
    StringVector prefixes_, postfixes_, contains_;
};
```

#### 3.3.3 图形设置集成

`TextureManager` 注册三个 `GraphicsSetting`：纹理质量（`qualitySettings_`）、纹理压缩（`compressionSettings_`）、纹理过滤（`filterSettings_`）。用户在选项菜单切换时，`onOptionSelected` 回调重新设置 `detailLevels_` 并把所有受影响纹理加入 `dirtyTextures_`，下帧 `reloadDirtyTextures()` 重载。

### 3.4 纹理辅助类

#### 3.4.1 `TextureAggregator`

动态把多个小纹理拼成大图集（[texture_aggregator.hpp](file:///workspace/src/lib/moo/texture_aggregator.hpp)），用于减少 draw call 与纹理切换：

```cpp
// texture_aggregator.hpp:50
class TextureAggregator
{
public:
    typedef void(*ResetNotifyFunc)(void);
    TextureAggregator(ResetNotifyFunc notifyFunc = NULL);

    int  addTile( BaseTexturePtr tex, const Vector2 & min, const Vector2 & max );
    void getTileCoords(int id, Vector2 & min, Vector2 & max) const;
    void delTile(int id);
    void repack();
    bool tilesReset() const;          // 上次 repack 后坐标是否变了
    DX::Texture * texture() const;
    const Matrix & transform() const; // texel→[0,1] 的纹理变换矩阵
private:
    std::auto_ptr<struct TextureAggregatorPimpl> pimpl_;
};
```

`addTile` 返回 tile ID，`getTileCoords` 取该 tile 在大图中的 UV 范围。当大图空间不够会 `repack()` 重排，此时 `tilesReset()` 为 true，调用方需要重建引用了旧 UV 的顶点缓冲。`transform()` 给出一个 4×4 矩阵把顶点的 texel 坐标映射到 `[0,1]`，避免顶点缓冲随大图尺寸变化重建。

#### 3.4.2 `TextureCompressor`

把任意纹理压缩成 DXT 格式（[texture_compressor.hpp](file:///workspace/src/lib/moo/texture_compressor.hpp)）：

```cpp
// texture_compressor.hpp:19
class TextureCompressor
{
public:
    TextureCompressor( DX::Texture* src, D3DFORMAT fmt = D3DFMT_DXT5,
                       uint32 numRequestedMipLevels = 0 );
    bool save( const std::string & filename );
    bool stow( DataSectionPtr pSection, const std::string & childTag = "" );
    bool convertTo( ComObjectWrap<DX::Texture>& destTexture );
private:
    HRESULT bltAllLevels( ComObjectWrap<DX::Texture>& src,
                           ComObjectWrap<DX::Texture>& dest ) const;
    HRESULT changeFormat( ... ) const;
};
```

`bltAllLevels` 用 `D3DXLoadSurfaceFromSurface` 逐 mip 拷贝，`changeFormat` 用 `D3DXFilterTexture`。编辑器离线转 DDS 用。

#### 3.4.3 `TextureExposer`

把 D3D 纹理的某 mip level 暴露成可读写的 `Image`，编辑器调试用（参见 texture_exposer.hpp）。

### 3.5 RenderTarget 与 MRT

#### 3.5.1 `RenderTarget`

可作纹理使用的渲染目标（[render_target.hpp](file:///workspace/src/lib/moo/render_target.hpp)），同时继承 `BaseTexture` 和 `DeviceCallback`：

```cpp
// render_target.hpp:31
class RenderTarget : public BaseTexture, public DeviceCallback
{
public:
    RenderTarget( const std::string & identitifer );
    virtual bool create( int width, int height, bool reuseMainZBuffer = false,
        D3DFORMAT pixelFormat = D3DFMT_A8R8G8B8,
        RenderTarget* pDepthStencilParent = NULL,
        D3DFORMAT depthFormatOverride = D3DFMT_UNKNOWN );
    virtual HRESULT release();
    virtual bool push( void );     // 把当前 RT 压栈并切到本 RT
    virtual void pop( void );       // 恢复
    void clearOnRecreate( bool enable, const Colour& col = (DWORD)0x00000000 );
    virtual bool valid( );
    HRESULT pSurface( ComObjectWrap<DX::Surface>& ret );
    virtual bool copyTexture( Moo::BaseTexturePtr pTexture );  // 从别处拷入
    // BaseTexture 接口
    virtual DX::BaseTexture* pTexture( );
    virtual DX::Surface* depthBuffer( );
    virtual uint32 width( ) const;
    virtual uint32 height( ) const;
    virtual D3DFORMAT format( ) const;
    virtual uint32 textureMemoryUsed( );
    // DeviceCallback
    virtual void deleteUnmanagedObjects( );
    void setRT2( RenderTarget* rt2 ) { pRT2_ = rt2; }  // 临时 MRT
private:
    uint32 width_, height_;
    int32 origWidth_, origHeight_;          // 创建时的原始尺寸（resize 前的值）
    RenderTarget* pRT2_;                    // 第二个 RT，临时 MRT 实现
    ComObjectWrap<DX::Texture>  pRenderTarget_;
    ComObjectWrap<DX::Surface>   pDepthStencilTarget_;
    bool reuseZ_;                            // 复用主深度缓冲
    D3DFORMAT depthFormat_, pixelFormat_;
    bool autoClear_;  Colour clearColour_;
    RenderTargetPtr pDepthStencilParent_;   // 共享别人 depth buffer
};
```

`push()` 把当前 render target、viewport、camera 压入 `RenderContext::renderTargetStack_`，然后 `device_->SetRenderTarget` 切到本 RT；`pop()` 恢复。`clearOnRecreate(true, colour)` 让 device reset 后重建 RT 时自动用 `colour` 清屏。`setRT2()` 是临时 MRT——把第二个 RT 也设置上。`RenderTargetSetter`（render_target.hpp:99）是 `EffectConstantValue` 派生，把 RT 作为 effect 的纹理参数注入。

`calculateDimensions()` 会根据当前 backbuffer 尺寸和 `origWidth_/origHeight_` 决定实际分配尺寸（支持按比例缩放），由 `ensureAllocated()` 在第一次 push 时调用。

#### 3.5.2 `CubeRenderTarget`

立方体环境贴图渲染目标（[cube_render_target.hpp](file:///workspace/src/lib/moo/cube_render_target.hpp)）：

```cpp
// cube_render_target.hpp:24
class CubeRenderTarget : public BaseTexture, public DeviceCallback
{
public:
    CubeRenderTarget( const std::string& identifier );
    bool create( uint32 cubeDimensions, const Colour & clearColour = (DWORD)0x00000000 );
    bool pushRenderSurface( D3DCUBEMAP_FACES face );  // 切到某一面
    void setCubeViewMatrix( D3DCUBEMAP_FACES face, const Vector3& centre );  // 设 6 面相机
    void pop( void );
    void setupProj();
    void restoreProj();
    virtual bool isCubeMap() { return true; }
    // DeviceCallback
    virtual void deleteUnmanagedObjects( );
    virtual void createUnmanagedObjects( );
private:
    uint32 cubeDimensions_;
    D3DFORMAT pixelFormat_;
    Matrix  originalProj_;
    Camera  originalCamera_;
    Colour  clearColour_;
    ComObjectWrap< DX::CubeTexture > pRenderTarget_;
    ComObjectWrap< DX::Surface >     pDepthStencilTarget_;
};
```

典型用途：水体反射、天空盒动态捕获。渲染时对 6 个面分别 `pushRenderSurface` + `setCubeViewMatrix` + 渲染场景 + `pop`，最后把整张 cube map 绑到 effect sampler。

#### 3.5.3 `MRTSupport`

MRT（Multiple Render Targets）能力管理（[mrt_support.hpp](file:///workspace/src/lib/moo/mrt_support.hpp)），基于 `InitSingleton`：

```cpp
// mrt_support.hpp:25
class MRTSupport : private Moo::EffectManager::IListener,
                  public InitSingleton<MRTSupport>
{
public:
    MRTSupport();
    bool isEnabled();
    void bind();        // 绑定第二张 RT 纹理到 effect 常量
    void unbind();
    bool bound() const;
    bool doInit();
    bool doFini();
    void onSelectPSVersionCap(int psVerCap);
private:
    class TextureSetter : public Moo::EffectConstantValue
    {
        bool operator()(ID3DXEffect* pEffect, D3DXHANDLE constantHandle);
        void map(DX::BaseTexture* pTexture);
        ComObjectWrap<DX::BaseTexture> map_;
    };
    static Moo::EffectMacroSetting::EffectMacroSettingPtr s_mrtSetting_;
    SmartPointer<TextureSetter> mapSetter_;
    bool bound_;
};
```

它通过 `EffectMacroSetting` 注册一个 `MRT_ENABLE` 编译宏，effect 文件 `#ifdef MRT_ENABLE` 选择是否启用 MRT 路径。`RenderContext::createDevice` 中根据 caps 判断 `mrtSupported_`（要求 PS3.0 + 多 RT + 独立 write mask），并在 `createUnmanaged()` 中调用 `createSecondSurface()` 创建第二张 RT surface。

### 3.6 `TextureStage`

固定函数管线的纹理阶段包装（[texturestage.hpp](file:///workspace/src/lib/moo/texturestage.hpp)），D3D9 时代遗留，主要用于 `Material`：

```cpp
// texturestage.hpp:30
class TextureStage
{
public:
    enum ColourOperation { DISABLE, SELECTARG1, SELECTARG2, MODULATE, MODULATE2X,
        MODULATE4X, ADD, ADDSIGNED, ADDSIGNED2X, SUBTRACT, ADDSMOOTH,
        BLENDDIFFUSEALPHA, BLENDTEXTUREALPHA, BLENDFACTORALPHA, BLENDTEXTUREALPHAPM,
        BLENDCURRENTALPHA, PREMODULATE, MODULATEALPHA_ADDCOLOR,
        MODULATECOLOR_ADDALPHA, MODULATEINVALPHA_ADDCOLOR,
        MODULATEINVCOLOR_ADDALPHA, BUMPENVMAP, BUMPENVMAPLUMINANCE,
        DOTPRODUCT3, MULTIPLYADD, LERP, LAST_COLOROP = 0xFFFFFFFF };
    enum FilterType     { POINT=1, LINEAR, ANISOTROPIC, FLATCUBIC, GAUSSIANCUBIC };
    enum ColourArgument { CURRENT=0, DIFFUSE, TEXTURE, TEXTURE_FACTOR, TEXTURE_ALPHA,
        TEXTURE_INVERSE, TEXTURE_ALPHA_INVERSE, DIFFUSE_ALPHA, DIFFUSE_INVERSE,
        DIFFUSE_ALPHA_INVERSE };
    enum WrapMode        { REPEAT=1, MIRROR, CLAMP };

    FilterType minFilter() const;  FilterType magFilter() const;
    ColourOperation colourOperation() const;
    ColourArgument colourArgument1() const;  ColourArgument colourArgument2() const;
    ColourOperation alphaOperation() const;
    uint32 textureCoordinateIndex() const;
    WrapMode textureWrapMode() const;
    bool useMipMapping() const;
    BaseTexturePtr pTexture() const;
    // ... 全部带 setter
private:
    FilterType minFilter_, magFilter_;
    ColourOperation colourOperation_, alphaOperation_;
    ColourArgument colourArgument1_, colourArgument2_, colourArgument3_;
    ColourArgument alphaArgument1_, alphaArgument2_, alphaArgument3_;
    uint32 textureCoordinateIndex_;
    WrapMode wrapMode_;
    bool useMipMapping_;
    BaseTexturePtr pTexture_;
};
```

`Material` 持有 `std::vector<TextureStage>`，`Material::set()` 把每个 stage 的状态下发到 D3D。

### 3.7 顶点缓冲：`VertexBuffer` / `VertexLock`

简洁的 D3D VB 包装（[vertex_buffer.hpp](file:///workspace/src/lib/moo/vertex_buffer.hpp)）：

```cpp
// vertex_buffer.hpp:25
class VertexBuffer
{
    ComObjectWrap<DX::VertexBuffer> vertexBuffer_;
public:
    HRESULT create( uint32 size, DWORD usage, DWORD FVF, D3DPOOL pool,
                   const char* allocator = "vertex buffer/unknown" );
    HRESULT set( UINT streamNumber = 0, UINT offsetInBytes = 0, UINT stride = 0 ) const;
    static HRESULT reset( UINT streamNumber = 0 );
    bool valid() const;
    void release();
    HRESULT lock( UINT offset, UINT size, VOID** data, DWORD flags );
    HRESULT unlock();
    HRESULT getDesc( D3DVERTEXBUFFER_DESC* desc );
    uint32 size();
    void preload();              // PreLoad 到显存
    void addToPreloadList();     // 加入 RenderContext 预加载队列
};

template<typename VertexType>
class VertexLock
{
    void* vertices_;  VertexBuffer& vb_;
public:
    VertexLock( VertexBuffer& vb );                            // lock 全部
    VertexLock( VertexBuffer& vb, UINT offset, UINT size, DWORD flags );
    ~VertexLock();                                             // 自动 unlock
    operator void*() const;
    void fill( const void* buffer, uint32 size );
    void pull( void* buffer, uint32 size ) const;
    VertexType& operator[]( int index );
};
typedef VertexLock<unsigned char> SimpleVertexLock;
```

`VertexLock` 是 RAII，构造时 `lock`、析构时 `unlock`，避免忘记 unlock 导致 D3D 锁死。`addToPreloadList()` 把 managed pool 的 VB 加入 `RenderContext::preloadResourceList_`，下几帧由 `preloadDeviceResources(timeLimitMs)` 在时间预算内逐个 `PreLoad()`。

### 3.8 索引缓冲：`IndexBuffer` / `IndicesReference` / `IndicesHolder`

三件套（[index_buffer.hpp](file:///workspace/src/lib/moo/index_buffer.hpp)）：

```cpp
// index_buffer.hpp:44
class IndicesReference            // 索引数据的引用（不拥有）
{
protected:
    void* indices_;  uint32 size_;  D3DFORMAT format_;
public:
    virtual void assign( void* buffer, uint32 numOfIndices, D3DFORMAT format );
    bool valid() const;
    void fill( const void* buffer, uint32 numOfIndices );
    void pull( void* buffer, uint32 numOfIndices ) const;
    void copy( const IndicesReference& src, int count, int start = 0,
               int srcStart = 0, uint32 vertexBase = 0 );
    D3DFORMAT format() const;
    int entrySize() const;          // INDEX16→2, INDEX32→4
    void* indices();
    uint32 size() const;
    IndexReference operator[]( int32 index );
    uint32 operator[]( int32 index ) const;
    void set( int32 index, uint32 value );
    uint32 get( int32 index ) const;
    static int bestEntrySize( D3DFORMAT format );
    static D3DFORMAT bestFormat( uint32 vertexNum );  // ≤65536 用 INDEX16
};

class IndicesHolder : public IndicesReference   // 拥有内存的版本
{
public:
    IndicesHolder( D3DFORMAT format = D3DFMT_INDEX16, uint32 entryNum = 0 );
    void setSize( uint32 entryNum, D3DFORMAT format );
    virtual void assign( const void* buffer, uint32 entryNum, D3DFORMAT format );
};

class IndexBuffer                  // D3D index buffer 包装
{
    ComObjectWrap<DX::IndexBuffer> indexBuffer_;
    D3DFORMAT format_;  DWORD numOfIndices_;
public:
    HRESULT create( int numOfIndices, D3DFORMAT format, DWORD usage, D3DPOOL pool,
                    const char* allocator = "index buffer/unknown" );
    HRESULT set() const;
    IndicesReference lock( DWORD flags = 0 );
    IndicesReference lock( UINT offset, UINT numOfIndices, DWORD flags );
    HRESULT unlock();
    bool isCurrent() const;  // 是否已 set 到 device
    void preload();  void addToPreloadList();
};
```

`IndicesReference` 设计巧妙：它只持有一个 `void*` + `size` + `format`，可以指向 `IndicesHolder` 拥有的内存，也可以指向 `IndexBuffer::lock()` 返回的 D3D 锁定内存。`copy()` 支持带 `vertexBase` 偏移的批量拷贝（合并 mesh 时用）。

### 3.9 动态缓冲：`DynamicVertexBuffer` / `DynamicIndexBuffer`

每帧动态生成的几何（粒子、flare、debug line、terrain morph）用环形 ring buffer（[dynamic_vertex_buffer.hpp](file:///workspace/src/lib/moo/dynamic_vertex_buffer.hpp)）：

```cpp
// dynamic_vertex_buffer.hpp:30
class DynamicVertexBufferBase : public DeviceCallback
{
protected:
    uint32 lockIndex_;     // 当前已用到第几个顶点
    int    vertexSize_;
    bool   lockModeDiscard_;  // 用 D3DLOCK_DISCARD
    bool   lockFromStart_;    // 从开头锁还是从 lockIndex_ 锁
    bool   softwareBuffer_;   // 软件处理
    bool   readOnly_;
    BYTE*  lock( uint32 nLockElements );   // 从头锁（DISCARD）
    BYTE*  lock2( uint32 nLockElements );  // 从 lockIndex_ 锁（NOOVERWRITE）
    static std::list< DynamicVertexBufferBase* > dynamicVBS_;
};

template <class VertexType>
class DynamicVertexBufferBase2 : public DynamicVertexBufferBase
{
public:
    bool lockAndLoad( const Vertex* pSrc, uint32 count, uint32& base );
    Vertex* lock( uint32 nLockElements );     // 同步 lock，DISCARD
    Vertex* lock2( uint32 nLockElements );   // 异步 lock，NOOVERWRITE
    uint32 lockIndex() const;
    HRESULT set( uint32 stream = 0 );
    HRESULT unset( uint32 stream = 0 );
};

template <class VertexType>
class DynamicVertexBuffer : public DynamicVertexBufferBase2<VertexType>
{ /* video memory */ };

template <class VertexType>
class DynamicSoftwareVertexBuffer : public DynamicVertexBufferBase2<VertexType>
{ /* system memory, software processing */ };
```

`lock()` 用 `D3DLOCK_DISCARD`（丢弃整块，从头开始），适合一次性写入大量顶点；`lock2()` 用 `D3DLOCK_NOOVERWRITE`（追加在 `lockIndex_` 之后，前面的不动），适合每帧追加少量顶点。`lockAndLoad(src, count, base)` 是便捷方法：lock → memcpy → unlock → 返回 `base`（顶点起始索引），调用方 `DrawPrimitive(startVertex=base, ...)`。

`DynamicIndexBuffer` 完全对称（[dynamic_index_buffer.hpp](file:///workspace/src/lib/moo/dynamic_index_buffer.hpp)）：

```cpp
// dynamic_index_buffer.hpp:30
class DynamicIndexBufferBase : public DeviceCallback
{
public:
    IndicesReference lock( uint32 nLockIndices );
    IndicesReference lock2( uint32 nLockIndices );
    uint32 lockIndex() const;
    HRESULT unlock();
    IndexBuffer indexBuffer();
    void release();
    void resetLock();
private:
    IndexBuffer indexBuffer_;
    uint32 lockIndex_;  bool reset_;  bool locked_;
    int lockBase_;  uint maxIndices_;
    DWORD usage_;  D3DFORMAT format_;
protected:
    DynamicIndexBufferBase( DWORD usage, D3DFORMAT format );
};

class DynamicIndexBuffer16 : public DynamicIndexBufferBase
{ /* D3DUSAGE_DYNAMIC|WRITEONLY, INDEX16 */ };
class DynamicIndexBuffer32 : public DynamicIndexBufferBase
{ /* INDEX32 */ };
class DynamicSoftwareIndexBuffer16 : public DynamicIndexBufferBase
{ /* + SOFTWAREPROCESSING */ };
class DynamicSoftwareIndexBuffer32 : public DynamicIndexBufferBase {};

class DynamicIndexBufferInterface    // 聚合 4 种，按 format 取
{
public:
    DynamicIndexBufferBase& get( D3DFORMAT format );
private:
    DynamicIndexBuffer16  hwBuffer16_;
    DynamicIndexBuffer32  hwBuffer32_;
    DynamicSoftwareIndexBuffer16 swBuffer16_;
    DynamicSoftwareIndexBuffer32 swBuffer32_;
};
```

`RenderContext` 持有 `DynamicIndexBufferInterface* pDynamicIndexBufferInterface_`，调用方 `rc().dynamicIndexBufferInterface().get(D3DFMT_INDEX16)` 取到对应 buffer。

### 3.10 顶点声明：`VertexDeclaration`

D3D9 vertex declaration 包装（[vertex_declaration.hpp](file:///workspace/src/lib/moo/vertex_declaration.hpp)），从 `res/shaders/formats/*.xml` 加载：

```cpp
// vertex_declaration.hpp:66
class VertexDeclaration
{
public:
    typedef std::vector<std::string> Aliases;
    const Aliases& aliases() const;
    DX::VertexDeclaration* declaration();
    const std::string& name() const;
    static VertexDeclaration* combine( VertexDeclaration* orig, VertexDeclaration* extra );
    static VertexDeclaration* get( const std::string& declName );
    static void fini();
private:
    D3DVertexDeclarationPtr pDecl_;
    Aliases aliases_;
    VertexDeclaration( const std::string& name );
    bool load( DataSectionPtr pSection );
    bool merge( VertexDeclaration* orig, VertexDeclaration* extra );
    std::string name_;
    static SimpleMutex declarationsLock_;
};
```

XML 描述每个语义（POSITION / NORMAL / TEXCOORD / BLENDWEIGHT …）的 type / stream / offset：

```xml
<root>
    <POSITION><type> FLOAT3 </type></POSITION>
    <TEXCOORD><type> FLOAT2 </type></TEXCOORD>
    <BLENDWEIGHT><type> SHORT2 </type></BLENDWEIGHT>
</root>
```

`combine(orig, extra)` 把两个声明合并成多 stream 声明——例如把基础 mesh stream 与第二个 UV stream 合并。`aliases()` 返回声明的所有别名（同一声明可以多个名字引用）。

### 3.11 顶点流：`VertexStreams`

多 stream 支持（[vertex_streams.hpp](file:///workspace/src/lib/moo/vertex_streams.hpp)）：

```cpp
// vertex_streams.hpp:27
struct UV2Stream     { static const std::string TYPE_NAME; static const uint32 STREAM_NUMBER = 10; typedef Vector2 TYPE; };
struct ColourStream  { static const std::string TYPE_NAME; static const uint32 STREAM_NUMBER = 11; typedef DWORD TYPE; };

class VertexStream : public ReferenceCount
{
public:
    virtual void remapVertices( uint32 offset, uint32 nVerticesBeforeMapping,
                                const std::vector< uint32 >& mappingNewToOld ) = 0;
    virtual void load( const void* data, uint32 nVertices ) = 0;
    void set();       // SetStreamSource
    void release();
    void preload();
    BinaryPtr data();
protected:
    template<typename T> void loadInternal( const void* data, uint32 nVertices );
    template<typename T> void remap( uint32 offset, uint32 nVerticesBeforeMapping,
                                      const std::vector< uint32 >& mappingNewToOld );
private:
    std::string type_;  uint32 stream_;  uint32 stride_;
    Moo::VertexBuffer vertexBuffer_;  uint32 count_;  BinaryPtr pData_;
};
typedef SmartPointer<VertexStream> VertexStreamPtr;

class StreamContainer
{
public:
    StreamContainer();
    void release();   // 释放所有 stream
    void set();       // set 所有 stream
    void preload();
    void updateDeclarations( VertexDeclaration* pDecl );
    std::vector<VertexStreamPtr> streamData_;
    VertexDeclaration* pSoftwareDecl_;
    VertexDeclaration* pSoftwareDeclTB_;
};
```

`UV2Stream` 与 `ColourStream` 用独立的 stream（编号 10/11）传送第二套 UV 和顶点色，这样基础 mesh 数据可以与额外数据分离加载（chunk 的 static lighting 就是这么做的）。

### 3.12 顶点格式：`vertex_formats.hpp`

定义所有内置顶点结构（[vertex_formats.hpp](file:///workspace/src/lib/moo/vertex_formats.hpp)），用 `#pragma pack(push, 1)` 严格 1 字节对齐。每个结构用 `FVF(format)` 宏声明对应的 D3D FVF（`MF_SERVER` 编译时为空，因为服务端不创建 vertex declaration）。

下表列出主要格式：

| 结构 | 字段 | FVF | 用途 |
|----|----|----|----|
| `VertexXYZNUV` | pos/normal/uv | `XYZ\|NORMAL\|TEX1` | 静态 mesh 基本格式 |
| `VertexXYZNUV2` | + uv2 | `XYZ\|NORMAL\|TEX2` | 双 UV（光照贴图） |
| `VertexXYZNDUV` | + colour | `XYZ\|NORMAL\|DIFFUSE\|TEX1` | 顶点色光照 |
| `VertexXYZN` | pos/normal | `XYZ\|NORMAL` | shadow volume |
| `VertexXYZL` | pos/colour | `XYZ\|DIFFUSE` | BSP 调试 |
| `VertexXYZ` | pos | `XYZ` | shadow volume |
| `VertexXYZUV` | pos/uv | `XYZ\|TEX1` | sky / imposter |
| `VertexTL` | pos(rhw)/colour | `XYZRHW\|DIFFUSE` | 屏幕空间（GUI） |
| `VertexTLUV` | pos(rhw)/uv | `XYZRHW\|TEX1` | 屏幕空间 + UV |
| `VertexXYZDP` | pos/colour/psize | `XYZ\|DIFFUSE\|PSIZE` | 点精灵 |
| `VertexXYZNUVTB` | pos/**packed normal**/uv/**packed tangent**/**packed binormal** | (无 FVF, 用 declaration) | normal mapping，法线压缩到 uint32 |
| `VertexXYZNUVIIIWW` | pos/packed normal/uv/3 索引/2 权重 | (无 FVF) | 软件蒙皮 3 权重 |
| `VertexXYZNUVIIIWWTB` | 上面 + packed tangent/binormal | (无 FVF) | 软件蒙皮 + normal map |
| `VertexXYZNUVIIIWWPC` | 上面解包版（normal/tangent/binormal 为 Vector3） | (无 FVF) | CPU 蒙皮输出 |
| `VertexYNDS` | y/packed normal/diffuse/shadow | (无 FVF) | terrain vertex |
| `FilterVertex<N>` | pos/uv[N]/worldNormal/viewNormal | (无 FVF) | 多 tap 过滤（草地） |

**法线压缩**（`unpackIntNormal`，[software_skinner.hpp](file:///workspace/src/lib/moo/software_skinner.hpp):24）：

```cpp
// software_skinner.hpp:23
inline Vector3 unpackIntNormal( uint32 packed )
{
    int32 z = int32(packed) >> 22;
    int32 y = int32( packed << 10 ) >> 21;
    int32 x = int32( packed << 21 ) >> 21;
    return Vector3( float( x ) / 1023.f, float( y ) / 1023.f, float( z ) / 511.f );
}
```

x/y 各 11 bit（除以 1023），z 10 bit（除以 511），共 32 bit。tangent / binormal 同样压缩。

### 3.13 `Vertices` / `VerticesManager` / `VertexSnapshot`

#### 3.13.1 `Vertices`

顶点数据加载与 set（[vertices.hpp](file:///workspace/src/lib/moo/vertices.hpp)）：

```cpp
// vertices.hpp:103
class Vertices : public SafeReferenceCount
{
public:
    Vertices( const std::string& resourceID, int numNodes );
    virtual HRESULT load( );
    virtual HRESULT release( );
    virtual HRESULT setVertices( bool software, bool staticLighting = false );
    virtual HRESULT setTransformedVertices( bool tb, const NodePtrVector& nodes );

    virtual VertexSnapshotPtr getSnapshot( const NodePtrVector& nodes,
                                            bool skinned, bool bumpMapped );
    virtual VertexSnapshotPtr getSnapshot( const std::avector<Matrix>& transforms,
                                            bool skinned, bool bumpMapped );

    const std::string& resourceID( ) const;
    uint32 nVertices( ) const;
    const std::string& format( ) const;
    uint32 vertexStride() const;
    Moo::VertexBuffer vertexBuffer( ) const;
    HRESULT bindStreams( bool staticLighting, bool softwareSkinned = false,
                         bool bumpMapped = false );
    typedef std::vector< Vector3 > VertexPositions;
    virtual const VertexPositions& vertexPositions();
    const VertexDeclaration* pDecl() const;
    void clearSoftwareSkinner();
    BaseSoftwareSkinner* softwareSkinner();
    bool bumpedFormat() const;  // 是否含 tangent/binormal
protected:
    StreamContainer* streams_;
    Moo::VertexBuffer vertexBuffer_;
    VertexDeclaration* pDecl_;
    VertexDeclaration* pStaticDecl_;
    uint32 nVertices_;
    std::string resourceID_;
    std::string format_;
    uint32 vertexStride_;
    mutable VertexPositions vertexPositions_;
    BaseSoftwareSkinnerPtr pSoftwareSkinner_;
    VertexSnapshotCache skinnedSnapshotCache_;
    VertexSnapshotCache rigidSnapshotCache_;
    Moo::VertexBuffer pSkinnerVertexBuffer_;
    bool vbBumped_;
    int numNodes_;   // 用于校验骨骼索引
};
```

`format_` 是字符串如 `"xyznuv"` / `"xyznuviiiwwtb"`，决定如何解析 `.primitive` 文件中的顶点数据并创建对应 `VertexDeclaration`。`setVertices(software, staticLighting)` 把 VB set 到 stream 0，并把额外 stream（UV2/Colour）也 set 上。`setTransformedVertices(tb, nodes)` 用于世界空间对象（chunk flora），把顶点预先变换到世界空间。

`getSnapshot(nodes, skinned, bumpMapped)` 返回一个 `VertexSnapshot`——这是延迟渲染（visual channel）的关键：把当前骨骼姿态下的顶点状态打包，之后在通道里按 z 排序绘制时再用。

#### 3.13.2 `VerticesManager`

`Vertices` 的缓存单例（[vertices_manager.hpp](file:///workspace/src/lib/moo/vertices_manager.hpp)）：

```cpp
// vertices_manager.hpp:26
class VerticesManager : public Moo::DeviceCallback
{
public:
    typedef std::map< std::string, Vertices * > VerticesMap;
    static VerticesManager* instance();
    VerticesPtr get( const std::string& resourceID, int numNodes = 0 );
    virtual void deleteManagedObjects();
    virtual void createManagedObjects();
private:
    VerticesMap vertices_;
    SimpleMutex verticesLock_;
    bool enableMorphVertices_;   // graphics setting 控制 morph 开关
};
```

`enableMorphVertices_` 为 false 时把 `MorphVertices` 替换成普通 `Vertices`，节省内存。

#### 3.13.3 `VertexSnapshot`

延迟渲染时保存顶点状态的接口（vertices.hpp:264）：

```cpp
// vertices.hpp:264
class VertexSnapshot : public ReferenceCount
{
public:
    virtual bool getVertexDepths( uint32 startVertex, uint32 nVertices,
                                  float* pOutDepths ) = 0;
    virtual uint32 setVertices( uint32 startVertex, uint32 nVertices,
                                bool staticLighting ) = 0;
    virtual void fini() = 0;
};
```

`getVertexDepths` 算出每个顶点的 view space depth，用于在 sorted channel 里按深度排序；`setVertices` 在真正绘制时把对应的顶点 set 到 device。`fini` 释放引用。

`VertexSnapshotCache`（vertices.hpp:52）是一个对象池，按 refCount==1 判断 snapshot 是否空闲可复用，避免每帧 `new`。

### 3.14 `Primitive` / `PrimitiveManager`

#### 3.14.1 `Primitive`

索引缓冲 + primitive group（[primitive.hpp](file:///workspace/src/lib/moo/primitive.hpp)）：

```cpp
// primitive.hpp:33
class Primitive : public SafeReferenceCount
{
public:
    virtual HRESULT setPrimitives();             // set IB 到 device
    virtual HRESULT drawPrimitiveGroup( uint32 groupIndex );
    virtual HRESULT release( );
    virtual HRESULT load( );

    uint32 nPrimGroups() const;
    const PrimitiveGroup& primitiveGroup( uint32 i ) const;
    uint32 maxVertices() const;
    const std::string& resourceID() const;
    D3DPRIMITIVETYPE primType() const;
    const IndicesHolder& indices() const { return indices_; }
    const Vector3& origin( uint32 i ) const;       // 每个 group 的几何中心
    void calcGroupOrigins( const VerticesPtr verts );
protected:
    typedef std::vector< PrimitiveGroup > PrimGroupVector;
    PrimGroupVector primGroups_;
    std::vector<Vector3> groupOrigins_;
    uint32 nIndices_;
    uint32 maxVertices_;
    std::string resourceID_;
    D3DPRIMITIVETYPE primType_;
    IndicesHolder indices_;
    IndexBuffer indexBuffer_;
};
```

`PrimitiveGroup`（[primitive_file_structs.hpp](file:///workspace/src/lib/moo/primitive_file_structs.hpp):65）：

```cpp
// primitive_file_structs.hpp:65
struct PrimitiveGroup
{
    int startIndex_;
    int nPrimitives_;
    int startVertex_;
    int nVertices_;
};
```

`drawPrimitiveGroup(groupIndex)` 调用 `rc().drawIndexedPrimitive(primType_, startVertex, 0, nVertices, startIndex, nPrimitives)`。

#### 3.14.2 `PrimitiveManager`

缓存单例（[primitive_manager.hpp](file:///workspace/src/lib/moo/primitive_manager.hpp)），与 `VerticesManager` 类似：

```cpp
// primitive_manager.hpp:26
class PrimitiveManager : public Moo::DeviceCallback
{
public:
    typedef std::map< std::string, Primitive * > PrimitiveMap;
    static PrimitiveManager* instance();
    PrimitivePtr get( const std::string& resourceID );
    virtual void deleteManagedObjects();
    virtual void createManagedObjects();
private:
    PrimitiveMap primitives_;
    SimpleMutex primitivesLock_;
};
```

### 3.15 `.primitive` 文件格式

`primitive_file_structs.hpp` 定义了 `.primitive` 二进制文件的结构：

```cpp
// primitive_file_structs.hpp:21
struct VertexHeader
{
    char vertexFormat_[ 64 ];   // 如 "xyznuviiiwwtb"
    int  nVertices_;
};

struct MorphHeader
{
    int version_;
    int nMorphTargets_;
};

struct MorphTargetHeader
{
    char identifier_[ 64 ];    // morph target 名字
    int  channelIndex_;        // 对应的 morph channel 索引
    int  nVertices_;           // 这个 target 影响的顶点数
};

struct IndexHeader
{
    char indexFormat_[ 64 ];   // "index16" 或 "index32"
    int  nIndices_;
    int  nTriangleGroups_;
};

struct PrimitiveGroup    // 见上
{
    int startIndex_;
    int nPrimitives_;
    int startVertex_;
    int nVertices_;
};
```

`.primitive` 文件布局：
```
[VertexHeader][vertex bytes...]
[MorphHeader]([MorphTargetHeader][morph vertex bytes...])*
[IndexHeader][index bytes...]
[PrimitiveGroup * nTriangleGroups]
```

`.visual` 文件本身是 XML（DataSection），引用 `.primitive` 文件作为 geometry。

---

## 4. Effect / Shader 系统

### 4.1 `EffectManager`

全局 Effect 缓存与编译管理单例（[effect_manager.hpp](file:///workspace/src/lib/moo/effect_manager.hpp)）：

```cpp
// effect_manager.hpp:34
class EffectManager : public DeviceCallback, public Singleton< EffectManager >
{
public:
    class IListener
    {
    public:
        virtual void onSelectPSVersionCap(int psVerCap) = 0;
    };
    typedef std::map< std::string, std::string > StringStringMap;
    typedef std::vector< std::string >           IncludePaths;

    ManagedEffectPtr get( const std::string& resourceID );
    void deleteEffect( ManagedEffect* pEffect );

    void compileOnly( const std::string& resourceID );
    bool needRecompile( const std::string& resourceID );
    bool hashResource( const std::string& name, MD5::Digest& result );
    std::string resolveInclude( const std::string& name );
    bool finishEffectInits();

    void createUnmanagedObjects();
    void deleteUnmanagedObjects();

    const IncludePaths& includePaths() const;
    void addIncludePaths( const std::string& pathString );

    bool checkModified( const std::string& objectName,
                        const std::string& resName,
                        MD5::Digest* resDigest = NULL );
    bool registerGraphicsSettings();

    void setMacroDefinition( const std::string & key, const std::string & value );
    const StringStringMap & macros();
    const std::string & fxoInfix() const;       // 编译产物路径后缀
    void fxoInfix(const std::string & infix);

    int PSVersionCap() const;                   // 像素着色器版本上限
    void PSVersionCap( int psVersion );

    StateManager * pStateManager();
    EffectIncludes * pEffectIncludes();

    void addListener(IListener * listener);
    void delListener(IListener * listener);
private:
    typedef std::map< std::string, ManagedEffect* > Effects;
    typedef std::list< ManagedEffectPtr >            EffectList;
    Effects             effects_;       // 注意：裸指针，外部引用归零即销毁
    EffectList          effectInitQueue_;   // 待初始化队列，引用计数
    SimpleMutex         effectInitQueueMutex_;
    StrDigestMap        hashCache_;         // resourceID → MD5
    SmartPointer< StateManager >   pStateManager_;
    SmartPointer< EffectIncludes > pEffectIncludes_;
    IncludePaths        includePaths_;
    SimpleMutex         effectsLock_;
    StringStringMap     macros_;            // 编译宏定义
    std::string         fxoInfix_;          // .fxo 缓存子目录后缀
    ListenerVector      listeners_;
    GraphicsSettingPtr  psVerCapSettings_;  // PS 版本上限设置
    SimpleMutex         compileMutex_;
};
```

关键设计：
- `effects_` 用裸指针——若用智能指针会永远不释放（因为缓存自己持有引用）。`ManagedEffect` 自己 `SafeReferenceCount`，外部 `EffectMaterial` 持有 `ManagedEffectPtr`，外部全释放后自动销毁。
- `effectInitQueue_` 用智能指针——加载后异步初始化（编译 technique、注册 graphics setting），未完成的不能被销毁。
- `hashCache_` 用 MD5 判断 `.fx` 是否变化以决定是否重编译。
- `fxoInfix_` 是编译产物路径后缀，根据当前 PS 版本上限、宏定义等生成，不同配置编译产物分开缓存。
- `pStateManager_` 是全局唯一的 `ID3DXEffectStateManager`，所有 effect 共享，把状态变更路由到 `RenderContext` 的缓存。
- `pEffectIncludes_` 实现 `ID3DXInclude`，处理 `#include`。

### 4.2 `ManagedEffect`

单个 D3DX Effect 的包装（[managed_effect.hpp](file:///workspace/src/lib/moo/managed_effect.hpp)）：

```cpp
// managed_effect.hpp:172
class ManagedEffect :
    public SafeReferenceCount,
    public EffectTechniqueSetting::IListener,
    public EffectManager::IListener
{
public:
    typedef std::pair<D3DXHANDLE, EffectConstantValuePtr*>  MappedConstant;
    typedef std::vector<MappedConstant>                     MappedConstants;
    typedef std::vector<RecordedEffectConstantPtr>          RecordedEffectConstants;

    bool load( const std::string& resourceName );
    bool registerGraphicsSettings(const std::string & effectResource);

    ID3DXEffect* pEffect() { return pEffect_.pComObject(); }
    const std::string& resourceID() { return resourceID_; }

    EffectPropertyMappings& defaultProperties();
    void setAutoConstants();                        // 设置自动常量（matrix/time/...）
    void recordAutoConstants( RecordedEffectConstants& recordedList );

    BinaryPtr compile( const std::string& resName, D3DXMACRO * preProcessDefinition=NULL,
                       bool force=false, std::string* result=NULL );
    bool cache(BinaryPtr bin, D3DXMACRO * preProcessDefinition=NULL );

    bool validateAllTechniques();
    bool validateShaderVersion( ID3DXEffect * d3dxeffect, D3DXHANDLE hTechnique) const;
    bool getFirstValidTechnique(D3DXHANDLE & hT);

    EffectTechniqueSetting * graphicsSettingEntry();
    int  maxPSVersion(D3DXHANDLE hTechnique) const;

    INLINE D3DXHANDLE getCurrentTechnique() const;
    bool setCurrentTechnique( D3DXHANDLE hTec, bool setExplicit );
    const std::string currentTechniqueName();
    VisualChannelPtr getChannel( D3DXHANDLE techniqueOverride = NULL );

    bool finishInit();
    bool readyToUse() const { return initComplete_ && validated_; }

    bool skinned( D3DXHANDLE techniqueOverride = NULL );
    bool bumpMapped( D3DXHANDLE techniqueOverride = NULL );
    bool dualUV( D3DXHANDLE techniqueOverride = NULL );

    typedef std::vector< TechniqueInfo > TechniqueInfoCache;
    const TechniqueInfoCache& techniques() { return techniques_; }
private:
    ComObjectWrap<ID3DXEffect> pEffect_;
    D3DXHANDLE                 hCurrentTechnique_;
    bool                       techniqueExplicitlySet_;
    bool                       initComplete_;
    int                        settingsListenerId_;
    EffectPropertyMappings     defaultProperties_;
    MappedConstants            mappedConstants_;   // name → 自动常量 setter
    std::string                resourceID_;
    TechniqueInfoCache         techniques_;
    std::string                settingName_, settingDesc_;
    EffectTechniqueSettingPtr  settingEntry_;
    bool                       settingsAdded_;
    bool                       validated_;
    uint32                     firstValidTechniqueIndex_;
};
```

#### 4.2.1 `TechniqueInfo`

每个 technique 的元数据（managed_effect.hpp:78）：

```cpp
// managed_effect.hpp:78
struct TechniqueInfo
{
    D3DXHANDLE        handle_;
    bool              supported_;       // 当前 caps 是否支持
    std::string       name_;
    std::string       settingLabel_;    // graphics setting 选项 label
    std::string       settingDesc_;
    VisualChannelPtr  channel_;        // 该 technique 走哪个 visual channel
    int               psVersion_;       // 所需 PS 版本
    bool              skinned_;
    bool              bumpMapped_;
    bool              dualUV_;
};
```

`channel_` 决定该 technique 渲染到 solid / sorted / shimmer / distortion 哪个通道——effect 可以在 annotation 里指定 `<channel>SORTED</channel>`。

#### 4.2.2 `EffectTechniqueSetting`

把一个 effect 的多个 technique 暴露为 graphics setting 选项（managed_effect.hpp:128）。effect 文件通过 annotation 声明：

```hlsl
int graphicsSetting < string label = "PARALLAX_MAPPING"; >;

technique lighting_shader1 < string label = "SHADER_MODEL_1"; > { ... }
technique software_fallback < string label = "SHADER_MODEL_0"; > { ... }
```

`EffectTechniqueSetting` 继承 `GraphicsSetting`，每个 technique 是一个选项。用户在选项菜单切换时，`onSelectTechnique` 回调把 `hCurrentTechnique_` 切换到对应 handle。`setPSCapOption(psVersionCap)` 在 PS 版本上限变化时自动选最高支持的 technique。

#### 4.2.3 `EffectMacroSetting`

通过修改 effect 编译宏来切换路径的设置（managed_effect.hpp:282），需要重启生效。例如 `romp/sky_light_map.cpp` 用它来启用 / 禁用动态天空：

```cpp
// 注册示例
EffectMacroSetting setting( "SKY_LIGHTING", "Sky lighting quality",
                            "SKY_LIGHTING_QUALITY", &setupFunc );
setting.addOption( "HIGH",   "High quality",   true,  "1" );
setting.addOption( "LOW",    "Low quality",    true,  "0" );
```

`onOptionSelected` 修改 `macros_` 中的 `SKY_LIGHTING_QUALITY` 值，下次 effect 编译时该宏被传入 `D3DXCreateEffect`。

### 4.3 `EffectMaterial`

业务层最常用的材质类（[effect_material.hpp](file:///workspace/src/lib/moo/effect_material.hpp)），在 `ManagedEffect` 之上挂属性：

```cpp
// effect_material.hpp:38
class EffectMaterial : public SafeReferenceCount
{
public:
    typedef std::map< D3DXHANDLE, EffectPropertyPtr > Properties;

    EffectMaterial();
    explicit EffectMaterial( const EffectMaterial & other );
    EffectMaterial & operator=( const EffectMaterial & other );
    ~EffectMaterial();

    bool load( DataSectionPtr pSection, bool addDefault = true );  // 从 .mfm xml 加载
    void save( DataSectionPtr pSection );

    bool initFromEffect( const std::string& effect, const std::string& diffuseMap = "",
                         int doubleSided = -1 );   // 编程式创建

    bool begin();        // effect->Begin()
    bool end() const;    // effect->End()
    bool beginPass( uint32 pass ) const;
    bool endPass() const;
    bool commitChanges() const;

    StateRecorder* recordPass( uint32 pass ) const;  // 录制状态用于延迟渲染
    uint32 nPasses() const { return nPasses_; }

    const std::string& identifier() const;
    D3DXHANDLE hTechnique() const;
    bool hTechnique( D3DXHANDLE hTec );

    ManagedEffectPtr pEffect() { return pManagedEffect_; }

    bool boolProperty( bool & result, const std::string & name ) const;
    bool intProperty( int & result, const std::string & name ) const;
    bool floatProperty( float & result, const std::string & name ) const;
    bool vectorProperty( Vector4 & result, const std::string & name ) const;
    EffectPropertyPtr getProperty( const std::string & name );
    ConstEffectPropertyPtr getProperty( const std::string & name ) const;
    bool replaceProperty( const std::string & name, EffectPropertyPtr effectProperty );

    WorldTriangle::Flags getFlags( int objectMaterialKind ) const;   // 物理碰撞 flags

    Properties & properties() { return properties_; }
    bool skinned() const;
    bool bumpMapped() const;
    bool dualUV() const;
    bool readyToUse() const
    { return pManagedEffect_ ? pManagedEffect_->readyToUse() : false; }
    VisualChannelPtr channel() const;
    bool isDefault( EffectPropertyPtr pProp );
    void replaceDefaults();
#ifdef EDITOR_ENABLED
    int materialKind() const;
    int collisionFlags() const;
    bool bspModified_;
#endif
private:
    bool loadInternal( DataSectionPtr pSection, EffectPropertyMappings& outProperties );
    ManagedEffectPtr  pManagedEffect_;
    UINT              nPasses_;
    Properties        properties_;
    VisualChannelPtr  channelOverride_;
    D3DXHANDLE        hOverriddenTechnique_;
    std::string       identifier_;
    int               materialKind_;       // 物理材质类型
    int               collisionFlags_;
    static EffectMaterialPtr s_curMaterial;
};
```

`.mfm` 文件是 XML：

```xml
<material>
    <identifier>MyMaterial</identifier>
    <fx>shaders/my.fx</fx>
    <property name="diffuseMap" type="Texture">textures/my.dds</property>
    <property name="colour" type="Vector4">1 1 1 1</property>
    <property name="doubleSided" type="Bool">false</property>
</material>
```

`load()` 解析 xml，`initFromEffect` 内部调用 `EffectManager::get(effectName)` 拿到 `ManagedEffectPtr`，然后对每个 `<property>` 创建对应 `EffectProperty`。`begin()` / `beginPass()` / `commitChanges()` 直接转发到 `ID3DXEffect`。`recordPass(pass)` 用 `StateRecorder` 录制一次 pass 的所有状态变更，用于 visual channel 延迟渲染时回放。

### 4.4 `EffectProperty`

effect 参数的抽象基类（managed_effect.hpp:47）：

```cpp
// managed_effect.hpp:47
class EffectProperty : public SafeReferenceCount
{
public:
    virtual EffectProperty* clone() const = 0;
    virtual bool apply( ID3DXEffect* pEffect, D3DXHANDLE hProperty ) = 0;
    virtual bool be( const bool & b ) { return false; }
    virtual bool be( const float & f ) { return false; }
    virtual bool be( const int & i ) { return false; }
    virtual bool be( const Matrix & m ) { return false; }
    virtual bool be( const Vector4 & v ) { return false; }
    virtual bool be( const BaseTexturePtr pTex ) { return false; }
    virtual bool be( const std::string & s ) { return false; }
    virtual bool getBool( bool & b ) const { return false; };
    virtual bool getInt( int & i ) const { return false; };
    virtual bool getFloat( float & f ) const { return false; };
    virtual bool getVector( Vector4 & v ) const { return false; };
    virtual bool getMatrix( Matrix & m ) const { return false; };
    virtual bool getResourceID( std::string & s ) const { return false; };
    virtual void asVector4( Vector4 & v ) const { getVector( v ); }
    virtual void setParent( const EffectProperty* pParent ) {};
    virtual void save( DataSectionPtr pDS ) = 0;
};
```

`be(...)` 系列是设置值（"be this value"），`getXxx(...)` 是读取值。`apply(effect, handle)` 把值真正下发到 effect。具体派生类有 `EffectConstantValue`（自动常量）、纹理属性、布尔属性、向量属性等，通过 `EffectPropertyFunctor`（managed_effect.hpp:330）注册到 `g_effectPropertyProcessors` map，按类型字符串创建。

### 4.5 `EffectConstantValue`

自动常量基类（[effect_constant_value.hpp](file:///workspace/src/lib/moo/effect_constant_value.hpp)）：

```cpp
// effect_constant_value.hpp:28
class EffectConstantValue : public ReferenceCount
{
public:
    virtual bool operator()(ID3DXEffect* pEffect, D3DXHANDLE constantHandle) = 0;
    static void fini();
    static EffectConstantValuePtr* get( const std::string& identifier,
                                         bool createIfMissing = true );
    static void set( const std::string& identifier, EffectConstantValue* pEffectConstantValue );
    virtual class RecordedEffectConstant* record(ID3DXEffect* pEffect,
                                                  D3DXHANDLE constantHandle) { return NULL; }
private:
    typedef std::map<std::string, EffectConstantValuePtr*> Mappings;
    static Mappings mappings_;
};
```

`get(identifier)` 按 effect 文件中的 semantic 名（如 `"WorldViewProjection"`、`"Time"`、`"FogColour"`）查找已注册的 setter。`operator()(effect, handle)` 把值 set 到 effect 的对应 handle。`ManagedEffect::setAutoConstants()` 遍历 `mappedConstants_`，对每个调用 `(*value)(pEffect_, handle)`。

常见自动常量 setter（在 effect_constant_value.cpp 等文件中定义）：
- 矩阵类：`World` / `View` / `Projection` / `WorldView` / `WorldViewProjection` / `ViewProjection` / `InvView` / `LastViewProjection`
- 时间类：`Time` / `FrameTimestamp` / `Fps`
- 相机类：`CameraPosition` / `CameraPlanes`（near/far）
- 光照类：`AmbientColour` / `DirectionalLights` / `OmniLights` / `SpotLights`（由 `EffectLightingSetter` 触发）
- 雾类：`FogColour` / `FogStart` / `FogEnd` / `FogEnabled`
- 屏幕类：`HalfPixelOffset` / `ViewportSize` / `ScreenSize`
- 调试类：`ObjectID`（用于 picking）

### 4.6 `EffectStateManager` 与状态录制

#### 4.6.1 `StateManager`

把 effect 内部的状态变更路由到 `RenderContext` 缓存（[effect_state_manager.hpp](file:///workspace/src/lib/moo/effect_state_manager.hpp)）：

```cpp
// effect_state_manager.hpp:30
class StateManager : public ID3DXEffectStateManager, public SafeReferenceCount
{
public:
    STDMETHOD(QueryInterface)(THIS_ REFIID iid, LPVOID *ppv);
    STDMETHOD_(ULONG, AddRef)(THIS);
    STDMETHOD_(ULONG, Release)(THIS);

    HRESULT __stdcall LightEnable( DWORD index, BOOL enable );
    HRESULT __stdcall SetFVF( DWORD fvf );
    HRESULT __stdcall SetLight( DWORD index, CONST D3DLIGHT9* pLight );
    HRESULT __stdcall SetMaterial( CONST D3DMATERIAL9* pMaterial );
    HRESULT __stdcall SetNPatchMode( FLOAT nSegments );
    HRESULT __stdcall SetPixelShader( LPDIRECT3DPIXELSHADER9 pShader );
    HRESULT __stdcall SetPixelShaderConstantB/F/I( ... );
    HRESULT __stdcall SetVertexShader( LPDIRECT3DVERTEXSHADER9 pShader );
    HRESULT __stdcall SetVertexShaderConstantB/F/I( ... );
    HRESULT __stdcall SetRenderState( D3DRENDERSTATETYPE state, DWORD value );
    HRESULT __stdcall SetSamplerState( DWORD sampler, D3DSAMPLERSTATETYPE type, DWORD value );
    HRESULT __stdcall SetTexture( DWORD stage, LPDIRECT3DBASETEXTURE9 pTexture);
    HRESULT __stdcall SetTextureStageState( DWORD stage, D3DTEXTURESTAGESTATETYPE type, DWORD value );
    HRESULT __stdcall SetTransform( D3DTRANSFORMSTATETYPE state, CONST D3DMATRIX* pMatrix );
};
```

每个方法都转发到 `rc().setXxx(...)`。D3DX Effect 调用 `ID3DXEffect::SetStateManager(stateManager)` 后，`effect->Begin()` / `CommitChanges()` 时不再直接调 device，而是通过这个回调——这样 effect 设置的状态也享受 RenderContext 的缓存。

#### 4.6.2 `QuickAlloc` / `ConstantAllocator`

每帧分配的内存池（effect_state_manager.hpp:65 / 146），避免堆分配：

```cpp
// effect_state_manager.hpp:65
template <int ElementSize>
class QuickAlloc : public RenderContextCallback
{
public:
    void* alloc();
    void reset();
    static QuickAlloc& instance();
    virtual void renderContextFini();
private:
    std::vector< char* > memTable_;
    uint32 offset_;
};

template <class ElemType>
class ConstantAllocator
{
public:
    class Allocation { ... };
    Allocation* init( const ElemType* data, uint32 nConstants );
    void reset();
    const ElementPool& pool() const;
private:
    ElementPool pool_;  uint32 offset_;
};
```

`alloc()` 在 1024 个元素的块里线性分配，跨块就 `new char[]`。每帧 `reset()` 重置 `offset_` 但不释放内存——下一帧复用。`ConstantAllocator<Vector4>` 用于批量 set shader 常量（一次 `SetVertexShaderConstantF` 设置一整组）。

#### 4.6.3 `StateRecorder` / `WrapperStateRecorder`

录制一次 effect pass 的所有状态，之后回放（effect_state_manager.hpp:265）：

```cpp
// effect_state_manager.hpp:265
class StateRecorder : public StateManager
{
public:
    // 所有 SetXxx 都录入对应的 vector
    HRESULT __stdcall SetRenderState( D3DRENDERSTATETYPE state, DWORD value );
    // ...
    virtual void setStates();     // 回放所有录制的状态
    void init();
    static StateRecorder* get();
    static void clear();
protected:
    void setRenderStates();  void setTextureStageStates();  void setSamplerStates();
    void setTransforms();  void setTextures();  void setLights();
    typedef ConstantAllocator<Vector4>::Allocation F4Constant;
    typedef std::pair< UINT, F4Constant* > F4Mapping;
    typedef VectorNoDestructor<F4Mapping> F4Mappings;
    F4Mappings vertexShaderConstantsF_;
    // ... I/B 常量、render state、transform、texture、light 等
    static uint32 s_nextAlloc_;
    static std::vector< SmartPointer<StateRecorder> > s_stateRecorders_;
};

class WrapperStateRecorder : public StateRecorder   // 用于 wrapper 模式
{
public:
    static void retireRecorders();   // 把录制好的推到 retire 列表等 GPU 完成
    static WrapperStateRecorder* get();
    static void put( WrapperStateRecorder* recorder );
    HRESULT __stdcall SetRenderState( D3DRENDERSTATETYPE state, DWORD value );
    virtual void setStates();
    void setStatesReal();
private:
    bool needsToRetire_;
    static SimpleMutex s_mutex_;
    static std::vector< SmartPointer<WrapperStateRecorder> > s_allRecorders_;
    static std::list< WrapperStateRecorder* > s_freeRecorders_;
    static std::vector< WrapperStateRecorder* > s_retireList_;
};
```

`StateRecorder::get()` 从 `s_stateRecorders_` 池取一个空闲的，`init()` 清空录制内容，effect pass 开始时把它 set 为 state manager，pass 结束后 effect 的所有状态都录到了 vector 里。`setStates()` 回放这些状态。`EffectMaterial::recordPass(pass)` 就是这么用的——visual channel 把状态录制下来，延迟到通道统一绘制时回放。

### 4.7 `EffectLightingSetter`

高效批量设置光照常量（[effect_lighting_setter.hpp](file:///workspace/src/lib/moo/effect_lighting_setter.hpp)）：

```cpp
// effect_lighting_setter.hpp:68
class EffectLightingSetter
{
public:
    typedef int32 BatchCookie;
    EffectLightingSetter( ManagedEffectPtr pEffect );
    void begin();                  // 缓存 semantic → handle 映射
    void resetBatchCookie() { lastBatch_ = NO_BATCH; }
    void apply( LightContainerPtr pDiffuse, LightContainerPtr pSpecular,
                bool commitChanges = true,
                BatchCookie batch = NO_BATCHING );   // 批量应用光照
    static const uint32 NO_BATCHING = 0xC00CEE00;
private:
    void addSemantic( const std::string& name );
    Semantics   semantics_;        // {name, handle, value setter}
    ManagedEffectPtr pEffect_;
    BatchCookie lastBatch_;
    static const uint32 NO_BATCH = 0xC00CEEEE;
};
```

用法示例（注释里给的）：
```cpp
EffectLightingSetter::begin( pMat );
pMat->begin();
pMat->beginPass();
for (uint32 i=0; i<500; i++)
{
    Moo::rc().lightContainer( lightcontainer_[i] );
    EffectLightingSetter::apply( pMat );
    draw( object_[i] );
}
pMat->endPass();
pMat->end();
```

`begin()` 一次性把所有光照相关 semantic 的 handle 缓存好，`apply()` 在循环里只 set 变化的值。`BatchCookie` 用于跨 frame 跳过相同光照（同一 batch 内不重复 set）。

### 4.8 `EffectIncludes` / `EffectVisualContext`

#### 4.8.1 `EffectIncludes`

`ID3DXInclude` 实现（[effect_includes.hpp](file:///workspace/src/lib/moo/effect_includes.hpp)）：

```cpp
// effect_includes.hpp:24
class EffectIncludes : public ID3DXInclude, public ReferenceCount
{
public:
    HRESULT __stdcall Open( D3DXINCLUDE_TYPE IncludeType, LPCSTR pFileName,
                            LPCVOID pParentData, LPCVOID *ppData, UINT *pBytes );
    HRESULT __stdcall Close( LPCVOID pData );
    void resetDependencies() { includesNames_.clear(); }
    std::list< std::string >& dependencies() { return includesNames_; }
    void currentPath( const std::string& currentPath );
    const std::string& currentPath( ) const;
private:
    typedef std::pair<LPCVOID, BinaryPtr> IncludeFile;
    std::vector< IncludeFile > includes_;
    std::list< std::string > includesNames_;   // 用于依赖追踪
    std::string currentPath_;
};
```

`Open` 通过 `BWResource::openSection` 读 include 文件，记录到 `includesNames_` 用于 hash 校验（决定是否需要重编译 effect）。`currentPath_` 处理相对路径 include。

#### 4.8.2 `EffectVisualContext`

visual 渲染期间的临时上下文（[effect_visual_context.hpp](file:///workspace/src/lib/moo/effect_visual_context.hpp)）：

```cpp
// effect_visual_context.hpp:26
class EffectVisualContext : public Singleton< EffectVisualContext >
{
public:
    EffectVisualContext();
    void initConstants();
    void pRenderSet( Visual::RenderSet* pRenderSet );
    Visual::RenderSet* pRenderSet() const;
    const Matrix& invWorld();
    float invWorldScale();
    DX::CubeTexture* pNormalisationMap();   // 法线归一化 cube map
    void tick( float dTime );
    void staticLighting( bool value );
    bool staticLighting( ) const;
    void isOutside( bool value );
    bool isOutside( ) const;
    void overrideConstants(bool value);
    bool overrideConstants() const;
private:
    Visual::RenderSet* pRenderSet_;
    Matrix invWorld_;
    float  invWorldScale_;
    bool   staticLighting_;
    bool   isOutside_;
    bool   overrideConstants_;
    bool   inited_;
    ComObjectWrap<DX::CubeTexture> pNormalisationMap_;
    void createNormalisationMap();
    typedef std::map< EffectConstantValuePtr*, EffectConstantValuePtr > ConstantMappings;
    ConstantMappings constantMappings_;
};

class EffectVisualContextSetter   // RAII
{
public:
    EffectVisualContextSetter( Visual::RenderSet* pRenderSet )
    { EffectVisualContext::instance().pRenderSet( pRenderSet ); }
    ~EffectVisualContextSetter()
    { EffectVisualContext::instance().pRenderSet( NULL ); }
};
```

`Visual::draw` 遍历 renderSet 时用 `EffectVisualContextSetter` 把当前 renderSet 推入 context，effect 中的 `"InvWorld"` / `"InvWorldScale"` / `"StaticLighting"` 等自动常量就能取到正确值。`pNormalisationMap_` 是预生成的归一化 cube map（把任意方向向量归一化），存于显存避免每像素 normalize。

### 4.9 `ShaderSet` / `ShaderManager`

固定函数时代的 vertex shader 选择器（[shader_set.hpp](file:///workspace/src/lib/moo/shader_set.hpp)、[shader_manager.hpp](file:///workspace/src/lib/moo/shader_manager.hpp)）：

```cpp
// shader_set.hpp:25
class ShaderSet : public Moo::DeviceCallback, public ReferenceCount
{
public:
    typedef uint32 ShaderID;
    typedef IDirect3DVertexShader9* ShaderHandle;
    typedef std::map< ShaderID, ShaderHandle > ShaderMap;

    ShaderSet( DataSectionPtr shaderSetSection, const std::string & vertexFormat,
               const std::string & shaderType );
    static ShaderID shaderID( char nDirectionalLights, char nPointLights, char nSpotLights );
    ShaderHandle shader( char nDirectionalLights, char nPointLights, char nSpotLights,
                         bool hardwareVP = false );
    void deleteUnmanagedObjects( );
    void preloadAll();
private:
    ShaderMap shaders_;     // 软件处理
    ShaderMap hwShaders_;  // 硬件处理
    DataSectionPtr shaderSetSection_;
    std::string vertexFormat_, shaderType_;
    bool preloading_;
};

// shader_manager.hpp:33
class ShaderManager : public InitSingleton< ShaderManager >
{
public:
    ShaderSetPtr shaderSet( const std::string& vertexFormat,
                            const std::string& shaderType,
                            const Moo::VertexDeclaration* pDecl = NULL );
private:
    ShaderSetPtr loadSet( const std::string& vertexFormat,
                          const std::string& shaderType,
                          const Moo::VertexDeclaration* pDecl = NULL );
    DataSectionPtr shaderRoot_;
    typedef StringMap< ShaderSetPtr > ShaderSetMap;
    typedef StringMap< ShaderSetMap > ShaderFormatMap;
    ShaderFormatMap sets_;
};
```

`ShaderID` 把 `(nDirLights, nPointLights, nSpotLights)` 编码成一个 uint32，作为 shader cache 的 key。固定函数路径下，按当前光源数选最匹配的 vertex shader——避免每帧重编译。Effect 时代这套路基本不用了，但 `Moo::Material` 的固定函数路径仍可能引用。

### 4.10 `Material`（固定函数材质）

固定函数管线的材质（[material.hpp](file:///workspace/src/lib/moo/material.hpp)）：

```cpp
// material.hpp:45
class Material
{
public:
    typedef enum UV2Generator_ { NONE=0, CHROME, PROJECTION, NORMAL, ROLLING_UV,
        TRANSFORM, LAST_UV2_GENERATOR } UV2Generator;
    typedef enum UV2Angle_ { TOP=0, SIDE, FRONT, LAST_UV2_ANGLE } UV2Angle;
    typedef enum BlendType_ { ZERO=1, ONE, SRC_COLOUR, INV_SRC_COLOUR, SRC_ALPHA,
        INV_SRC_ALPHA, DEST_ALPHA, INV_DEST_ALPHA, DEST_COLOUR, INV_DEST_COLOUR,
        SRC_ALPHA_SAT, BOTH_SRC_ALPHA, BOTH_INV_SRC_ALPHA,
        LAST_BELNDTYPE = 0xFFFFFFFF } BlendType;
    enum Channel { SOLID = 1<<0, SORTED = 1<<1, SHIMMER = 1<<2, FLARE = 1<<3 };

    const std::string& identifier() const;
    uint32 numTextureStages() const;
    TextureStage& textureStage( uint32 stageNum );
    void addTextureStage( const TextureStage &textureStage );

    bool alphaBlended() const;  BlendType srcBlend() const;  BlendType destBlend() const;
    const Colour& ambient() const;  const Colour& diffuse() const;  const Colour& specular() const;
    float selfIllum() const;
    uint32 alphaReference() const;  bool alphaTestEnable() const;
    bool zBufferRead() const;  bool zBufferWrite() const;
    bool solid() const;  bool sorted() const;  bool flare() const;  bool shimmer() const;
    bool doubleSided() const;
    uint8 channelFlags() const;
    UV2Generator uv2Generator() const;  UV2Angle uv2Angle() const;
    uint32 textureFactor() const;  bool fogged() const;
    uint32 collisionFlags() const;  uint8 materialKind() const;

    void set( bool vertexAlphaOverride = false ) const;   // 把所有状态 set 到 device
    static void setVertexColour();

    bool load( const std::string& resourceID );            // 从 .mfm xml 加载
    bool load( DataSectionPtr spMaterialSection );

    static bool disableMaterials;
    static bool shadowMaterials;
    static bool shimmerMaterials;
    static void tick( float dTime );
    static void channelMask( uint8 mask );
    static void channelOn( uint8 on );
    static void globalBump( bool bumpOn );
private:
    std::string identifier_;
    Colour ambient_, diffuse_, specular_;
    float selfIllum_;
    bool doAlphaBlend_;  BlendType srcBlend_, destBlend_;
    uint32 textureFactor_;
    bool alphaTestEnable_;  uint32 alphaReference_;
    bool zBufferWrite_, zBufferRead_, doubleSided_, fogged_;
    uint8 channelFlags_;
    UV2Generator uv2Generator_;  UV2Angle uv2Angle_;
    uint32 collisionFlags_;
    std::vector<TextureStage> textureStages_;
    Vector3 uvAnimation_;  Vector4 uvTransform_;
    uint32 frameTimestamp_;  uint64 lastTime_;
    bool bumpEnable_;  Colour specularColour_;
    BaseTexturePtr bumpTexture_, diffuseTexture_, glowTexture_;
    bool characterLighting_;
    static uint8 channelMask_, channelOn_;
    static bool globalBump_;
};
```

`.mfm` 文件既可以是固定函数（用 `Material`）也可以是 effect（用 `EffectMaterial`）。`Material::set()` 把所有状态（blend mode、texture stage、z buffer、cull、fog）下发到 device。`uv2Generator_` 决定第二套 UV 如何生成（CHROME = 球面反射投影、PROJECTION = 相机投影、NORMAL = 法线方向、ROLLING_UV = 滚动 UV、TRANSFORM = 自定义矩阵）。

### 4.11 `MaterialLoader` / `SectionProcessors`

`.mfm` xml 的 section 解析器注册表（[material_loader.hpp](file:///workspace/src/lib/moo/material_loader.hpp)）：

```cpp
// material_loader.hpp:25
class SectionProcessor
{
public:
    typedef bool (*Method)( Material& mat, DataSectionPtr pSect );
    SectionProcessor( Method m ) : method_( m ) {}
    Method operator *() { return method_; }
private:
    Method method_;
};
typedef std::pair<const char *, SectionProcessor::Method> SectProcEntry;

class SectionProcessors : public StringHashMap<SectionProcessor>
{
public:
    static SectionProcessors& instance();
};
```

每个 `<textureStage>` / `<uv2Generator>` / `<blendType>` 等 xml tag 对应一个 `SectionProcessor::Method`，按 tag 名查表分发。第三方模块可以注册自己的 processor 扩展 `.mfm` 格式。

### 4.12 `graphics_settings`

全局图形设置注册表（[graphics_settings.hpp](file:///workspace/src/lib/moo/graphics_settings.hpp)）：

```cpp
// graphics_settings.hpp:59
class GraphicsSetting : public ReferenceCount
{
public:
    typedef std::pair<std::string, std::pair<std::string, bool> > StringStringBool;
    typedef std::vector<StringStringBool> StringStringBoolVector;
    typedef SmartPointer<GraphicsSetting> GraphicsSettingPtr;
    typedef std::vector<GraphicsSettingPtr> GraphicsSettingVector;

    GraphicsSetting( const std::string & label, const std::string & desc,
                     int activeOption, bool delayed, bool needsRestart );
    virtual ~GraphicsSetting() = 0;

    const std::string & label() const;
    const std::string & desc() const;
    int addOption( const std::string & label, const std::string & desc, bool isSupported );
    const StringStringBoolVector & options() const;
    bool selectOption( int optionIndex, bool force = false );
    virtual void onOptionSelected( int optionIndex ) = 0;
    int activeOption() const;
    void needsRestart( bool a_needsRestart );
    void addSlave( GraphicsSettingPtr slave );

    static void init( DataSectionPtr section );
    static void write( DataSectionPtr section );
    static void add( GraphicsSettingPtr option );
    static const GraphicsSettingVector & settings();
    static bool hasPending();
    static bool isPending( GraphicsSettingPtr setting, int & o_pendingOption );
    static void commitPending();
    static void rollbackPending();
    static bool needsRestart();
    static GraphicsSettingPtr getFromLabel( const std::string & settingDesc );
private:
    std::string label_, desc_;
    StringStringBoolVector options_;
    int activeOption_;
    bool delayed_;       // 延迟生效（用户确认后再 commit）
    bool needsRestart_;  // 需要重启
    GraphicsSettingVector slaves_;   // 联动从设置
    static GraphicsSettingVector s_settings;
    static SettingIndexVector s_pending;
    static StringIntMap s_latentSettings;
    static bool s_needsRestart;
    static std::string s_filename;
};

template<typename ClassT>
class CallbackGraphicsSetting : public GraphicsSetting   // 成员函数回调
{
public:
    typedef void (ClassT::*Function)(int);
    CallbackGraphicsSetting( const std::string & label, const std::string & desc,
                              ClassT & instance, Function function,
                              int activeOption, bool delayed, bool needsRestart );
    virtual void onOptionSelected(int optionIndex);
};

class StaticCallbackGraphicsSetting : public GraphicsSetting   // 静态函数回调
{ /* 类似上面 */ };

template<typename ClassT>
GraphicsSetting::GraphicsSettingPtr
    makeCallbackGraphicsSetting( ... );   // 便捷工厂
```

每个 setting 有多个 option，`selectOption` 切换。`delayed=true` 的设置先入 `s_pending`，UI 调 `commitPending()` 后才真正触发 `onOptionSelected`（用户在选项菜单点 OK 才生效）。`needsRestart=true` 切换后置全局 `s_needsRestart`，UI 提示重启。`addSlave` 让一个设置带动另一个（如纹理质量档位联动过滤质量）。

`graphics_settings_picker`（graphics_settings_picker.hpp）根据硬件 caps 自动选最佳默认 option。

---

## 5. 渲染对象（Visual / Node / Animation）

### 5.1 `Visual`

moo 的核心网格资产类（[visual.hpp](file:///workspace/src/lib/moo/visual.hpp)），代表一个 `.visual` 文件：

```cpp
// visual.hpp:138
class Visual : public SafeReferenceCount
{
public:
    enum { LOAD_ALL = 0, LOAD_GEOMETRY_ONLY = 1 } LOADING_FLAGS;

    typedef EffectMaterial Material;   // 兼容 VisualLoader

    class BSPCache { /* BSP 树缓存 */ };
    class BSPTreePtr : public BSPProxyPtr { /* BSPTree* 自动转 BSPProxyPtr */ };
    class Materials : public std::vector<EffectMaterialPtr>
    {
    public:
        const EffectMaterialPtr find( const std::string& identifier ) const;
    };

    struct PrimitiveGroup
    {
        uint32 groupIndex_;          // Primitive::primGroups_ 中的索引
        EffectMaterialPtr material_;
    };
    typedef std::vector< PrimitiveGroup > PrimitiveGroupVector;

    struct Geometry
    {
        VerticesPtr          vertices_;
        PrimitivePtr         primitives_;
        PrimitiveGroupVector primitiveGroups_;
        uint32 nTriangles() const;
    };
    typedef std::vector< Geometry > GeometryVector;

    class RenderSet : public Aligned
    {
    public:
        bool            treatAsWorldSpaceObject_;   // 是否世界空间（不蒙皮）
        NodePtrVector   transformNodes_;             // 影响本 renderSet 的骨骼
        GeometryVector  geometry_;
        Matrix          firstNodeStaticWorldTransform_;
    };
    typedef std::avector< RenderSet > RenderSetVector;
    typedef std::vector< AnimationPtr > AnimationVector;

    class DrawOverride   // 自定义 draw 行为
    {
    public:
        virtual HRESULT draw( Visual* pVisual, bool ignoreBoundingBox = false ) = 0;
    };

    explicit Visual( const std::string& resourceID, bool validateNodes = true,
                     uint32 loadingFlags = LOAD_ALL );
    HRESULT draw( bool ignoreBoundingBox = false, bool useDefaultPose = true );
    HRESULT batch( bool ignoreBoundingBox = false, bool useDefaultPose = true );
    static HRESULT drawBatches();
    void justDrawPrimitives();

    uint32 nVertices() const;
    uint32 nTriangles() const;
    NodePtr rootNode() const;
    AnimationPtr animation( uint32 i ) const;
    uint32 nAnimations() const;
    void addAnimation( AnimationPtr animation );
    AnimationPtr findAnimation( const std::string& identifier ) const;

    const BoundingBox& boundingBox() const;
    void boundingBox( const BoundingBox& bb );

    uint32 nPortals( void ) const;
    const PortalData& portal( uint32 i ) const;
    const BSPTree* pBSPTree() const;

    void overrideMaterial( const std::string& materialIdentifier, const EffectMaterial& mat );
    int gatherMaterials( const std::string & materialIdentifier,
                          std::vector< PrimitiveGroup * > & primGroups,
                          ConstEffectMaterialPtr * ppOriginal = NULL );
    int collateOriginalMaterials( std::vector< EffectMaterialPtr > & rppMaterial );

    void useExistingNodes( NodeCatalogue & nodeCatalogue );
    void staticVertexColours( StaticVertexColoursPtr staticVertexColours );

    const std::string & resourceID() const;
    RenderSetVector& renderSets() { return renderSets_; }
    bool isOK() const { return isOK_; }

    static void disableBatching( bool val );
    static BSPProxyPtr loadBSPVisual( const std::string& resourceID );
    static DrawOverride* s_pDrawOverride;
    static uint32 drawCount_;
private:
    HRESULT drawBatch();
    void addLightsInModelSpace( const Matrix& invWorld );
    void addLightsInWorldSpace( );
    bool isInVisualManager_;
    AnimationVector animations_;
    NodePtr rootNode_;
    RenderSetVector renderSets_;
    std::vector<PortalData> portals_;
    BoundingBox bb_;
    Materials ownMaterials_;
    StaticVertexColoursPtr staticVertexColours_;
    BSPProxyPtr pBSP_;
    bool validateNodes_;
    class VisualBatcher* pBatcher_;
    std::string resourceID_;
    bool isOK_;
    static VectorNoDestructor< Visual* > s_batches_;
    static bool disableBatching_;
};
```

#### 5.1.1 `RenderSet` 设计

`RenderSet` 把**共享同一组骨骼**的 geometry 分到一起——这样 GPU vertex shader 常量里的骨骼矩阵只需要 set 一次，多个 geometry 共享。一个 Visual 可能有多个 RenderSet（如角色身体 / 武器 / 帽子分别用不同骨骼子集）。`treatAsWorldSpaceObject_=true` 表示该 renderSet 不做蒙皮（如 flora 的草），顶点直接在世界空间，`setTransformedVertices` 把顶点预变换到世界空间。

#### 5.1.2 `PortalData`

chunk 间可见性传送（visual.hpp:43）：

```cpp
// visual.hpp:43
class PortalData : public std::vector<Vector3>
{
public:
    PortalData() : flags_( 0 ) { }
    int flags() const;
    void flags( int f );
    bool isInvasive() const { return !!(flags_ & 1); }   // 是否"侵入"式 portal
private:
    int flags_;
};
```

`isInvasive()` 表示该 portal 可以穿越看到对面（chunk 间 hole）。chunk 模块的 `ChunkBoundary::Portal` 引用这些数据。

#### 5.1.3 `StaticVertexColours`

静态顶点色（烘焙光照）的抽象（visual.hpp:116）：

```cpp
// visual.hpp:116
class StaticVertexColours : public SafeReferenceCount
{
public:
    static const UINT STREAM_NUMBER = 1;   // 第二个 stream
    virtual bool readyToRender() const = 0;
    virtual void unset() = 0;
    virtual void set() = 0;
};
typedef SmartPointer<StaticVertexColours> StaticVertexColoursPtr;
```

chunk 模块的 `ChunkLighting` 实现这个接口，把烘焙的顶点色作为 stream 1 提供。`Visual::draw` 检测到 `staticVertexColours_` 存在且 `readyToRender()` 就 set 它，effect 通过 `DIFFUSE` 语义读取。

#### 5.1.4 `draw` 流程

`Visual::draw`（visual.cpp:872）是典型调用流：

```cpp
// visual.cpp:872
HRESULT Visual::draw( bool ignoreBoundingBox, bool useDefaultPose )
{
    if (s_pDrawOverride) return s_pDrawOverride->draw( this, ignoreBoundingBox );
    static VisualHelper helper;
    if (helper.shouldDraw(ignoreBoundingBox, bb_))
    {
        drawCount_++;
        if (useDefaultPose) { rootNode_->traverse(); rootNode_->loadIntoCatalogue(); }
        bool useStaticLighting = staticVertexColours_.exists()
                                  && staticVertexColours_->readyToRender();
        helper.start( useStaticLighting, bb_ );
        EffectVisualContext::instance().initConstants();
        EffectVisualContext::instance().staticLighting( useStaticLighting );
        if (useStaticLighting) staticVertexColours_->set();
        // 遍历 renderSets
        for (auto & renderSet : renderSets_)
        {
            EffectVisualContextSetter setter( &renderSet );   // RAII 推入
            for (auto & geometry : renderSet.geometry_)
            {
                geometry.vertices_->setVertices( rc().mixedVertexProcessing(), useStaticLighting );
                geometry.primitives_->setPrimitives();
                bool transformedVertsSet = false;
                for (auto & primitiveGroup : geometry.primitiveGroups_)
                {
                    EffectMaterialPtr pMat = primitiveGroup.material_;
                    if (!pMat->readyToUse()) continue;
                    if (!pMat->channel())   // 立即渲染路径
                    {
                        // 世界空间对象且不蒙皮 → 预变换顶点
                        if (!pMat->skinned() && renderSet.treatAsWorldSpaceObject_)
                        {
                            if (!transformedVertsSet) {
                                geometry.vertices_->setTransformedVertices(
                                    pMat->bumpMapped(), renderSet.transformNodes_ );
                                transformedVertsSet = true;
                            }
                        }
                        else if (transformedVertsSet) {   // 切回普通顶点
                            geometry.vertices_->setVertices(...);
                            transformedVertsSet = false;
                        }
                        if (pMat->begin())
                        {
                            for (uint32 i = 0; i < pMat->nPasses(); i++) {
                                pMat->beginPass( i );
                                geometry.primitives_->drawPrimitiveGroup( primitiveGroup.groupIndex_ );
                                pMat->endPass();
                            }
                            pMat->end();
                        }
                    }
                    else   // 延迟渲染路径：放入 visual channel
                    {
                        VertexSnapshotPtr pVSS = geometry.vertices_->getSnapshot(
                            renderSet.transformNodes_, pMat->skinned(), pMat->bumpMapped() );
                        float distance = ...;   // 算 z 距离用于排序
                        pMat->channel()->addItem( pVSS, geometry.primitives_, pMat,
                            primitiveGroup.groupIndex_, distance, staticVertexColours_ );
                    }
                }
            }
        }
        if (useStaticLighting) staticVertexColours_->unset();
        VertexBuffer::reset( Moo::UV2Stream::STREAM_NUMBER );
        VertexBuffer::reset( Moo::ColourStream::STREAM_NUMBER );
        helper.fini();
    }
    return S_OK;
}
```

两条路径：
1. **立即渲染**：`pMat->channel()` 为空时直接 `begin` / `beginPass` / `drawPrimitiveGroup` / `endPass` / `end`。用于 solid 通道。
2. **延迟渲染**：`pMat->channel()` 非空时，把 `VertexSnapshot + Primitive + Material` 加入对应 visual channel，等到通道统一绘制时再按 z 排序渲染。用于 sorted / shimmer / distortion 通道。

`batch()` 与 `draw()` 类似，但只把对象放入 `s_batches_`，`drawBatches()` 才真正绘制——允许跨多个 visual 合并相同材质的 draw call。

### 5.2 `VisualManager`

Visual 缓存单例（[visual_manager.hpp](file:///workspace/src/lib/moo/visual_manager.hpp)）：

```cpp
// visual_manager.hpp:28
class VisualManager
{
public:
    typedef std::map< std::string, Visual * > VisualMap;
    static VisualManager* instance();
    static void init();
    static void fini();
    VisualPtr get( const std::string& resourceID, bool loadIfMissing = true );
    void fullHouse( bool noMoreEntries = true );
    void add( Visual * pVisual, const std::string & resourceID );
private:
    VisualMap visuals_;
    SimpleMutex visualsLock_;
    bool fullHouse_;
    static VisualManager* pInstance_;
    friend Visual::~Visual();
};
```

### 5.3 `VisualCompound`

实例合并优化（[visual_compound.hpp](file:///workspace/src/lib/moo/visual_compound.hpp)）——把同一 visual 的多个实例合并成一个大 draw call，用于 flora、props 等大量重复物件：

```cpp
// visual_compound.hpp:38
class VisualCompound : public DeviceCallback, public BackgroundTask
{
public:
    VisualCompound();
    bool init( const std::string& visualResourceName );
    TransformHolder* addTransform( const Matrix& transform, uint32 batchCookie );
    bool draw( EffectMaterialPtr pMaterialOverride = NULL );
    void updateDrawCookie();
    bool valid() const;
    void deleteManagedObjects();
    SimpleMutex& mutex();

    class Batch   // 按 batchCookie（通常是 chunk）分组
    {
    public:
        void draw( uint32 sequence );
        LightContainerPtr pLightContainer();
        LightContainerPtr pSpecLContainer();
        TransformHolder* add( const Matrix& transform );
        void del( TransformHolder* transformHolder );
        typedef std::vector< PrimitiveGroup > PrimitiveGroups;
        const PrimitiveGroups& primitiveGroups() const;
        VertexBuffer& vertexBuffer();
        IndexBuffer& indexBuffer();
        const BoundingBox& boundingBox() const;
        typedef std::vector<uint8> Sequences;
        const TransformHolders& transformHolders() const;
        void invalidate();
        bool preloaded();
        void preload();
    };

    static VisualCompound* get( const std::string& visualName );
    static TransformHolder* add( const std::string& visualName, const Matrix& transform, uint32 batchCookie );
    static void drawAll( EffectMaterialPtr pMaterialOverride = NULL );
    static void fini();
    static void grabDelMutex();
    static void giveDelMutex();
    static void disable( bool val );
private:
    VerticesHolder   sourceVerts_;       // 源顶点（VertexXYZNUVTBPC）
    IndicesHolder    sourceIndices_;
    uint32           nSourceVerts_;
    uint32           nSourceIndices_;
    std::vector< PrimitiveGroup > sourcePrimitiveGroups_;
    BoundingBox      sourceBB_;
    BoundingBox      bb_;
    VertexDeclaration* pDecl_;
    typedef std::map< uint32, Batch* > BatchMap;
    BatchMap         batchMap_;          // batchCookie → Batch
    bool valid_;
    std::vector< EffectMaterialPtr > materials_;
    std::vector< EffectLightingSetter* > lightingSetters_;
    typedef std::vector<Batch*> BatchVector;
    BatchVector renderBatches_;
    BatchVector dirtyBatches_;
    uint32 drawCookie_, updateCookie_;
    void doBackgroundTask( BgTaskManager & mgr );   // 后台线程合并
    bool update();
    Batch* getNextDirtyBatch();
    void invalidate();
    std::string resourceName_;
    bool taskAdded_;
    SimpleMutex mutex_;
    static bool disable_;
};
```

工作流：
1. `VisualCompound::get(visualName)` 创建 / 取得某 visual 的 compound。
2. chunk 加载时 `VisualCompound::add(visualName, transform, batchCookie)` 把每个实例的 transform 加入对应 Batch。
3. `dirty(batch)` 标记该 batch 需要重算，`doBackgroundTask` 在后台线程把所有实例的顶点变换后合并到一个大 VB / IB。
4. `drawAll()` 主线程绘制所有 batch，每个 batch 一个 draw call（而不是每实例一个）。

`TransformHolder`（visual_compound.hpp:189）持有单个实例的 transform 和它在合并后 VB 中的 sequence（偏移）。

### 5.4 `VisualSplitter` / `VisualChannels`

#### 5.4.1 `VisualSplitter`

按骨骼数量拆分 renderSet（[visual_splitter.hpp](file:///workspace/src/lib/moo/visual_splitter.hpp)）：

```cpp
// visual_splitter.hpp:25
class VisualSplitter
{
public:
    VisualSplitter(uint32 nodeLimit = 17);   // 单 renderSet 最多 17 个骨骼
    void open( const std::string& resourceName );
    bool split();
    void save( const std::string& resourceName );
    class RenderSet : public ReferenceCount
    {
    public:
        virtual bool compute() = 0;
        virtual void save( DataSectionPtr pVisualSection, DataSectionPtr pPrimitivesSection ) = 0;
    };
    static void copySections( DataSectionPtr pDest, DataSectionPtr pSrc );
private:
    uint32 nodeLimit_;
    std::string resourceName_;
    std::vector< RenderSetPtr > pRenderSets_;
    DataSectionPtr pSection_;
    NodePtr pNode_;
};
```

`split()` 把超过 `nodeLimit` 个骨骼的 renderSet 拆成多个子 renderSet，每个子 renderSet 的骨骼数 ≤ 限制（GPU vertex shader 常量寄存器有限，通常 17 个 4×3 矩阵）。这是离线工具，运行时不调用。

#### 5.4.2 `VisualChannels`

延迟渲染通道系统（[visual_channels.hpp](file:///workspace/src/lib/moo/visual_channels.hpp)）：

```cpp
// visual_channels.hpp:29
class VisualChannel : public SafeReferenceCount
{
public:
    virtual void addItem( VertexSnapshotPtr pVSS, PrimitivePtr pPrimitives,
        EffectMaterialPtr pMaterial, uint32 primitiveGroup, float zDistance,
        StaticVertexColoursPtr pStaticVertexColours ) = 0;
    virtual void clear() = 0;
    static void add( const std::string& name, VisualChannelPtr pChannel );
    static void remove( const std::string& name );
    static VisualChannelPtr get( const std::string& name );
    static void initChannels();
    static void finiChannels();
    static void clearChannelItems();
    static bool enabled();
    static void enabled(bool enabled);
private:
    static VisualChannels visualChannels_;   // name → channel
    static bool enabled_;
};

class ChannelDrawItem : public ReferenceCount
{
public:
    virtual void draw() = 0;
    virtual void fini() {};
    float distance() const;
    void alwaysVisible(bool val);
};

class VisualDrawItem : public ChannelDrawItem
{
public:
    void init( VertexSnapshotPtr pVSS, PrimitivePtr pPrimitives,
        SmartPointer<StateRecorder> pStateRecorder,
        uint32 primitiveGroup, float distance,
        StaticVertexColoursPtr pStaticVertexColours = NULL,
        bool sortInternal = false );
    virtual void draw();
    virtual void fini();
    static VisualDrawItemPtr get();   // 从对象池取
protected:
    void drawSorted();    // 内部三角形排序
    void drawUnsorted();
    VertexSnapshotPtr pVSS_;
    PrimitivePtr pPrimitives_;
    uint32 primitiveGroup_;
    Vector3 partialWorldView_;
    SmartPointer<StateRecorder> pStateRecorder_;   // 录制的状态
    StaticVertexColoursPtr pStaticVertexColours_;
    bool sortInternal_;
};
```

内置通道：
- **`SortedChannel`**：按 z 距离排序的半透明物体。`addItem` 把 VisualDrawItem 加入 `s_drawItems_`，`draw()` 排序后逐个调用 `draw()`。`push()` / `pop()` 支持嵌套（反射场景）。
- **`InternalSortedChannel`**：sorted channel 的变体，内部三角形也排序（用于水晶 / 玻璃等多层半透明）。
- **`ShimmerChannel`**：水波纹效果（屏幕级），渲染到 shimmer buffer 后合成。
- **`SortedShimmerChannel`**：同时渲染到 sorted 和 shimmer。
- **`DistortionChannel`**：每材质自定义扭曲贴图的"哈哈镜"效果，比 shimmer 灵活。`DistortionDrawItem` 多了一个 `maskPass_` 用于绘制扭曲 mask。

通道顺序（固定）：
```
1. Solid（立即渲染，Visual::draw 内部直接 draw）
2. Shimmer
3. Distortion
4. Sorted（含 InternalSorted、SortedShimmer）
```

`initChannels()` 注册所有内置通道，`clearChannelItems()` 每帧开始清空，`draw()` 在 `EnviroMinder::draw` 中按顺序调用。

### 5.5 `Node` / `NodeCatalogue`

#### 5.5.1 `Node`

骨骼节点（[node.hpp](file:///workspace/src/lib/moo/node.hpp)）：

```cpp
// node.hpp:39
class Node : public SafeReferenceCount, public Aligned
{
public:
    Node();
    void traverse( );                    // 遍历计算 worldTransform_
    void loadIntoCatalogue( );
    void visitSelf( const Matrix& parent );

    void addChild( NodePtr node );
    void removeFromParent( );
    void removeChild( NodePtr node );

    Matrix& transform( );                 // 局部 transform
    const Matrix& transform( ) const;
    void transform( const Matrix& m );

    const Matrix& worldTransform( ) const;   // 世界 transform（traverse 后才有）
    void worldTransform( const Matrix& );

    NodePtr parent( ) const;               // 注意：parent_ 是裸指针，避免循环引用
    uint32 nChildren( ) const;
    NodePtr child( uint32 i );

    const std::string& identifier( ) const;
    NodePtr find( const std::string& identifier );
    uint32 countDescendants( ) const;
    void loadRecursive( DataSectionPtr nodeSection );

    float blend( int blendCookie );           // 取 blend 比例
    void blend( int blendCookie, float blendRatio );
    void blendClobber( int blendCookie, const Matrix & transform );  // 覆盖式 blend
    BlendTransform & blendTransform();
private:
    Matrix   transform_;        // 局部
    Matrix   worldTransform_;   // 世界（traverse 计算）
    int      blendCookie_;      // 防止重复 blend
    float    blendRatio_;
    BlendTransform blendTransform_;   // 双四元数 blend
    bool     transformInBlended_;
    Node*    parent_;           // 裸指针
    NodePtrVector children_;
    std::string identifier_;
public:
    static int s_blendCookie_;   // 全局 blend cookie，每次动画 tick 自增
};
```

`traverse()` 递归从根到叶，把父节点的 `worldTransform_` 与子节点 `transform_` 相乘得到子节点 `worldTransform_`。`blend(cookie, ratio)` 用 `blendCookie_` 跟踪当前动画帧，避免同一帧被多个动画重复 blend。`blendClobber` 是覆盖式（不 blend，直接设），用于位移覆盖。`BlendTransform`（math 库）是双四元数表示，避免线性矩阵插值的体积变化问题。

#### 5.5.2 `NodeCatalogue`

全局节点共享单例（[node_catalogue.hpp](file:///workspace/src/lib/moo/node_catalogue.hpp)）：

```cpp
// node_catalogue.hpp:36
class NodeCatalogue : public StringHashMap< NodePtr >,
                      public Singleton< NodeCatalogue >
{
public:
    static Node * findOrAdd( NodePtr pNode );
    static NodePtr find( const char * identifier );
    static void grab();
    static void give();
private:
    SimpleMutex nodeCatalogueLock_;
};
```

所有同名 node 共享一份存储——多个模型用同一套骨骼时，只动画一次，所有模型共享 `worldTransform_`。代价是 `PyModelNodes` 不能直接引用 node 状态，需要每帧 copy 出来。`grab()` / `give()` 引用计数管理，所有模型释放后才清空 catalogue。

### 5.6 `Animation` / `AnimationChannel` / `AnimationManager`

#### 5.6.1 `AnimationChannel`

单个骨骼的动画通道基类（[animation_channel.hpp](file:///workspace/src/lib/moo/animation_channel.hpp)）：

```cpp
// animation_channel.hpp:32
class AnimationChannel : public SafeReferenceCount
{
public:
    AnimationChannel();
    AnimationChannel( const AnimationChannel & other );

    virtual Matrix result( float time ) const;             // 直接返回 Matrix
    virtual void  result( float time, Matrix& out ) const;
    virtual void  result( float time, BlendTransform& out ) const;  // 双四元数

    virtual void  nodeless( float time, float blendRatio ) const;  // 不绑 node 的事件（音效 cue）
    virtual bool  wantTick() const;
    virtual void  tick( float dtime, float otime, float ntime,
                        float btime, float etime ) const;

    const std::string& identifier( ) const;
    virtual bool load( BinaryFile & bf );
    virtual bool save( BinaryFile & bf ) const;

    virtual void preCombine ( const AnimationChannel & rOther ) = 0;
    virtual void postCombine( const AnimationChannel & rOther ) = 0;
    virtual int  type() const = 0;
    virtual AnimationChannel * duplicate() const = 0;

    static AnimationChannel * construct( int type );   // 工厂，按 type 创建
protected:
    typedef AnimationChannel * (*Constructor)();
    static void registerChannelType( int type, Constructor cons );
    class TypeRegisterer { /* 注册器 */ };
private:
    std::string identifier_;
    static ChannelTypes * s_channelTypes_;
};

class ChannelBinder   // 把 channel 绑到具体 node
{
private:
    AnimationChannelPtr channel_;
    NodePtr node_;
public:
    ChannelBinder( AnimationChannelPtr channel, NodePtr node );
    inline void animate( float time ) const
    {
        if (!channel_) return;
        if (node_) channel_->result( time, node_->transform() );
        else        channel_->nodeless( time, 1.f );
    }
};
typedef std::vector< ChannelBinder > ChannelBinderVector;
```

派生类（按 `type()` 区分）：

| 派生类 | 文件 | 特点 |
|----|----|----|
| `InterpolatedAnimationChannel` | [interpolated_animation_channel.hpp](file:///workspace/src/lib/moo/interpolated_animation_channel.hpp) | 关键帧线性插值（最常用），存关键帧时刻 + Matrix，`result(time)` 找两帧线性插值 |
| `DiscreteAnimationChannel` | [discrete_animation_channel.hpp](file:///workspace/src/lib/moo/discrete_animation_channel.hpp) | 离散关键帧（不插值），逐帧切换，用于 stylized 动画 |
| `MorphAnimationChannel` | [morph_animation_channel.hpp](file:///workspace/src/lib/moo/morph_animation_channel.hpp) | 形变通道，触发 `MorphVertices` 的某 morph target，`nodeless(time, blendRatio)` 把 blend 值写入 `MorphVertices::addMorphValue` |
| `StreamedAnimationChannel` | [streamed_animation_channel.hpp](file:///workspace/src/lib/moo/streamed_animation_channel.hpp) | 流式动画，从 `.anca` 文件按需加载关键帧，`tick` 触发预加载 |
| `CueChannel` | [cue_channel.hpp](file:///workspace/src/lib/moo/cue_channel.hpp) | 事件 cue，到达某时刻触发回调（如脚步声、特效），`nodeless(time, blend)` 触发 |
| `TranslationOverrideChannel` | [translation_override_channel.hpp](file:///workspace/src/lib/moo/translation_override_channel.hpp) | 只覆盖位移，旋转 / 缩放保留 base 动画——用于脚步贴地（rotation 走 walk 动画，translation 走 IK） |

`preCombine` / `postCombine` 用于动画合并（如上半身 + 下半身动画合并）。

#### 5.6.2 `Animation`

动画集合（[animation.hpp](file:///workspace/src/lib/moo/animation.hpp)）：

```cpp
// animation.hpp:40
class Animation : public ReferenceCount
{
public:
    Animation( Animation * anim, NodePtr root );
    Animation( Animation * anim );
    Animation();

    void animate( float time ) const;                          // 直接播放
    static void animate( const BlendedAnimations & list );     // 混合多个
    static void animate( BlendedAnimations::const_iterator first,
                         BlendedAnimations::const_iterator final );
    void animate( int blendCookie, float frame, float blendRatio,
                  const NodeAlphas * pAlphas = NULL );
    void tick( float dtime, float oframe, float nframe,
               float bframe, float eframe );

    uint32 nChannelBinders( ) const;
    const ChannelBinder& channelBinder( uint32 i ) const;
    ChannelBinder& channelBinder( uint32 i );
    void addChannelBinder( const ChannelBinder& binder );

    AnimationChannelPtr findChannel( NodePtr node ) const;
    AnimationChannelPtr findChannel( const std::string& identifier );

    ChannelBinder * itinerantRoot( ) const;       // 流浪根（动态绑定的额外 node）

    float totalTime( ) const;
    const std::string identifier( ) const;
    const std::string internalIdentifier( ) const;
    bool load( const std::string & resourceID );
    bool save( const std::string & resourceID, uint64 useModifiedTime = 0 );

    void translationOverrideAnim( AnimationPtr pBase, AnimationPtr pTranslationReference,
                                   const std::vector< std::string >& noOverrideChannels );
private:
    float               totalTime_;
    ChannelBinderVector channelBinders_;
    std::string         identifier_;
    std::string         internalIdentifier_;
    mutable ChannelBinder * pItinerantRoot_;
    StreamedAnimationPtr pStreamer_;
    AnimationPtr       pMother_;      // 母动画（用于共享数据）
    AnimationChannelVector tickers_;   // wantTick() 为 true 的 channel
};

struct BlendedAnimation
{
    Animation *animation_;
    const NodeAlphas *alphas_;        // 每骨骼 alpha（部分 blend）
    float frame_;
    float blendRatio_;
};
```

`animate(blendCookie, frame, blendRatio, pAlphas)` 是核心：遍历所有 `ChannelBinder`，调用 `channel->result(frame, node->transform())`，按 `blendRatio` 与上一帧结果 blend。`NodeAlphas` 控制每骨骼的 blend 权重（上半身用动画 A，下半身用动画 B）。`itinerantRoot` 是动态绑定的额外根 node，用于"挂载"外部骨骼（武器握把）。

#### 5.6.3 `AnimationManager`

动画缓存单例（[animation_manager.hpp](file:///workspace/src/lib/moo/animation_manager.hpp)）：

```cpp
// animation_manager.hpp:33
class AnimationManager : public Singleton< AnimationManager >
{
public:
    AnimationManager();
    AnimationPtr get( const std::string& resourceID, NodePtr rootNode );
    AnimationPtr get( const std::string& resourceID );
    AnimationPtr find( const std::string& resourceID );
    std::string resourceID( Animation * pAnim );
    void fullHouse( bool noMoreEntries = true );
    typedef std::map< std::string, Animation * > AnimationMap;
private:
    AnimationMap animations_;
    SimpleMutex animationsLock_;
    bool fullHouse_;
};
```

`get(resourceID, rootNode)` 复用缓存的 channel 数据，但为每个 rootNode 创建独立的 `ChannelBinder`——多个模型可以共享一份动画数据，但各自的骨骼独立动画。

### 5.7 `SoftwareSkinner` / `MorphVertices`

#### 5.7.1 CPU 蒙皮

[software_skinner.hpp](file:///workspace/src/lib/moo/software_skinner.hpp) 定义 CPU 蒙皮的顶点类型与接口：

```cpp
// software_skinner.hpp:35
class RigidSkinVertex       // 单骨骼刚体顶点
{
public:
    Vector3 position, normal;  Vector2 uv;  int index;
    typedef VertexXYZNUVI SecondaryType;     // GPU 蒙皮格式
    typedef VertexXYZNUVI PrimaryType;        // CPU 蒙皮输出格式
    void output( VertexXYZNUV& vertex, const NodePtrVector& nodes );
    void output( VertexXYZNUVTBPC& vertex, const NodePtrVector& nodes );
    // ... 多种 output 重载
};

class RigidSkinBumpVertex : public RigidSkinVertex   // + tangent/binormal
{
    Vector3 tangent, binormal;
    typedef VertexXYZNUVITB SecondaryType;
    typedef VertexXYZNUVITBPC PrimaryType;
};

class SoftSkinVertex         // 3 权重软蒙皮
{
public:
    Vector3 position, normal;  Vector2 uv;
    int index1, index2, index3;
    float weight1, weight2, weight3;
    typedef VertexXYZNUVIIIWW SecondaryType;
    typedef VertexXYZNUVIIIWWPC PrimaryType;
    void output( VertexXYZNUV& vertex, const NodePtrVector& nodes )
    {
        const Matrix& wTrans1 = nodes[index1]->worldTransform();
        const Matrix& wTrans2 = nodes[index2]->worldTransform();
        const Matrix& wTrans3 = nodes[index3]->worldTransform();
        vertex.pos_ = wTrans1.applyPoint( position ) * weight1 +
                      wTrans2.applyPoint( position ) * weight2 +
                      wTrans3.applyPoint( position ) * weight3;
        vertex.normal_ = wTrans1.applyVector( normal );
        vertex.uv_ = uv;
    }
};

class SoftSkinBumpVertex : public SoftSkinVertex   // + tangent/binormal
{ Vector3 tangent, binormal; };

class BaseSoftwareSkinner : public ReferenceCount   // 接口
{
public:
    virtual void transformVertices( VertexXYZNUV* pDestVertices,
        int startVertex, int nVertices, const NodePtrVector& nodes,
        const Vector3* pPositionOverrides = NULL ) = 0;
    virtual void transformVertices( VertexXYZNUVTBPC* pDestVertices, ... ) = 0;
    virtual void transformVertices( VertexXYZNUV* pDestVertices, ...,
                                    const Matrix* pTransforms, ... ) = 0;
    virtual void outputDepths( float* pDestDepths, int startVertex, int nVertices,
                                const Vector4* partialWVP,
                                const Vector3* pPositionOverrides = NULL ) = 0;
    virtual void transformPositions( Vector3* pDestPos, int startVertex,
        int nVertices, const NodePtrVector& nodes,
        bool simplified = true, uint32 vertexSkip = 0 ) = 0;
};

template <class VertexType>
class SoftwareSkinner : public BaseSoftwareSkinner   // 模板实现
{
public:
    typedef std::vector< VertexType > Vertices;
    template<typename VT> void init( const typename VT* pVertices, int vertexCount );
    void transformVertices( VertexXYZNUV* pDestVertices, ... ) { /* 遍历 + output */ }
    // ...
private:
    Vertices vertices_;
};
```

CPU 蒙皮用于：1）固定函数硬件（无 vertex shader）；2）需要预计算顶点深度用于排序（sorted channel）；3）软件 vertex processing 模式。`output` 方法把蒙皮后的顶点写入 `VertexXYZNUV` 或 `VertexXYZNUVTBPC`（含 tangent/binormal），可以接受 `positionOverride` 用于 IK 修改顶点位置。

#### 5.7.2 `MorphVertices`（形变顶点）

`MorphVertices` 继承 `Vertices`，为带 morph target 的网格提供"顶点形变 + 蒙皮快照"双能力。文件见 [morph_vertices.hpp](file:///workspace/src/lib/moo/morph_vertices.hpp)。其内部结构由三层组成：

```
MorphVertices : public Vertices
├── MorphTargetBase : public ReferenceCount     // 单个 morph target
│     ├── MorphVertex { Vector3 delta_; int index_; }   // 增量顶点
│     └── apply(void* vertices, float amount) = 0
├── MorphTarget<VertexType> : MorphTargetBase   // 模板化 apply 实现
└── VertexListBase : public ReferenceCount      // 引用计数的顶点列表
      └── vertices() / nVertices()
```

`MorphTargetBase::applyPosOnly` 是一个非虚内联方法，只更新顶点位置（用于深度排序时不需要法线 / UV）：

```cpp
// morph_vertices.hpp:54
void applyPosOnly( Vector3* pos, float amount )
{
    BW_GUARD;
    MVVector::iterator it = morphVertices_.begin();
    MVVector::iterator end = morphVertices_.end();
    while (it != end)
    {
        const MorphVertex& mv = *it++;
        pos[mv.index_] += mv.delta_ * amount;
    }
}
```

模板化的 `MorphTarget<VertexType>::apply` 做同样的事，但写入完整顶点结构（`verts[mv.index_].pos_ += mv.delta_ * amount`）。这样一份 morph 数据可以同时驱动多种顶点格式（`VertexXYZNUV` / `VertexXYZNUVIIIWW` / `VertexXYZNUVTBPC`），由编译期模板特化分发。

##### Morph 值的全局状态机

`MorphVertices` 暴露一组静态接口供 `Model` 层在动画 tick 时调用：

```cpp
// morph_vertices.hpp:134
static void addMorphValue( const std::string& channelName, float value, float blend )
{
    morphValues_[channelName].addValue( value, blend, globalCookie_ );
}
static void incrementCookie() { globalCookie_++; }
static void globalBlend( float blend ) { globalBlend_ = blend; }
static float morphValue( const std::string& channelName )
{
    return morphValues_[channelName].value(globalCookie_);
}
```

- `globalCookie_` 是每帧自增的"代际戳"。每帧开始时 `Model` 调一次 `incrementCookie()`，所有 `morphValues_` 中缓存值就被视作过期。
- `addMorphValue(channel, value, blend)` 由各 morph 动画通道在 `tick` 时累加，`MorphValue::addValue` 内部检查 `cookie_ != cookie` 则重置 `value_`/`totalBlend_`，再以加权方式累加 `value_ += value * blend`。多个动画叠加同一 morph 通道即自然混合。
- `morphValue(channel)` 返回最终归一化值（`value_ / totalBlend_`），供 `setVerticesType` 在锁 VB 时应用。

##### setVerticesType 模板：运行时合成顶点

```cpp
// morph_vertices.hpp:163
template<class VertexType>
HRESULT setVerticesType( bool software )
{
    BW_GUARD;
    static std::vector< VertexType > verts;
    verts.assign( (VertexType*) vertexList_->vertices(),
                  (VertexType*) vertexList_->vertices() + nVertices_ );
    MorphTargetVector::iterator it = morphTargets_.begin();
    MorphTargetVector::iterator end = morphTargets_.end();
    while (it != end)
    {
        float val = morphValues_[(*it)->identifier()].value( globalCookie_);
        if (val > 0.001f)
            (*it)->apply( &verts.front(), val );
        ++it;
    }
    // ... 锁 DynamicVertexBuffer 并 memcpy
}
```

注意几个细节：
1. `static std::vector<VertexType> verts` —— 函数级静态缓冲，避免每帧分配，但意味着同一类型的 morph vertices 不能并发渲染（moo 本身是单线程渲染，所以没问题）。
2. 只对 `val > 0.001f` 的 morph target 调用 `apply`，避免零权重下做无意义的内存写。
3. 检查 `DevCaps2 & D3DDEVCAPS2_STREAMOFFSET`：如果硬件支持流偏移（stream offset），用 `vb.lock2(nVertices_)` 在 ring buffer 中拿到一个对齐偏移；否则用 `vb.lock(nVertices_)` 从 ring 头开始。后者会强制 ring buffer 切换到新段，影响吞吐。
4. `software == true` 时走 `DynamicSoftwareVertexBuffer`，即用 `D3DPOOL_SCRATCH` 的内存顶点缓冲，配合 software vertex processing。这是为软件 vertex shader / CPU 蒙皮路径准备的。

##### VertexSnapshot 缓存

`getSnapshot(nodes, skinned, bumpMapped)` 是 `Visual::draw` 在 sorted channel 排序时的关键调用：它返回一个"已蒙皮 + 已 morph"的顶点快照，附带生成该快照的 `NodePtrVector` 哈希。如果同一帧多个 Visual 实例引用同一 `MorphVertices` 且骨骼变换相同，就复用缓存的快照，避免重复 CPU 蒙皮。缓存分两套：`skinnedSnapshotCache_`（蒙皮后）和 `rigidSnapshotCache_`（无骨骼刚体），由 `skinned` 参数选择。

##### .morph 文件二进制布局

`.primitive` 文件中 morph 数据段遵循 [primitive_file_structs.hpp](file:///workspace/src/lib/moo/primitive_file_structs.hpp) 中的布局：

```
MorphHeader  { int version_; int nMorphTargets_; }
MorphTargetHeader[nMorphTargets_]
   { char identifier_[64]; int channelIndex_; int nVertices_; }
MorphVertex[nMorphTargets_][nVertices_]
   { Vector3 delta_; int index_; }
```

每个 `MorphTargetHeader` 后紧接 `nVertices_` 个 `MorphVertex`。`identifier_`（最多 64 字符）对应动画通道名（如 "mouth_open"），由 `MorphAnimationChannel` 在 `tick` 时通过 `MorphVertices::addMorphValue(identifier, value, blend)` 注入。

### 5.8 光照

moo 的光照系统分四块：光源类型（`OmniLight` / `SpotLight` / `DirectionalLight`）、光照桶（`LightContainer`）、固定函数管线提交（`commitToFixedFunctionPipeline`）、地形光照预计算（`getTerrainLight`）。雾作为状态由 `RenderContext` 持有，由 `FogHelper` 命名空间统一设置。

#### 5.8.1 光源类型层级

三种光源都继承 `SafeReferenceCount`，都通过 `typedef SmartPointer<X> XPtr` 暴露智能指针，集合类型用 `VectorNoDestructor<XPtr>`（无析构的 vector，避免引用计数原子操作开销）。

```cpp
// omni_light.hpp:33
class OmniLight : public SafeReferenceCount
{
public:
    OmniLight( const D3DXCOLOR& colour, const Vector3& position,
               float innerRadius, float outerRadius );
    void worldTransform( const Matrix& transform );   // 由 LightContainer 调用
    const Vector3&  worldPosition() const;
    float worldInnerRadius() const;
    float worldOuterRadius() const;
    bool  intersects( const BoundingBox& worldSpaceBB ) const;
    float attenuation( const BoundingBox& worldSpaceBB ) const;
    Vector4* getTerrainLight( uint32 timestamp, float lightScale );
    bool dynamic() const;     void dynamic( bool );
    int  priority() const;   void priority(int);
private:
    Vector3 position_;       float innerRadius_, outerRadius_;
    Colour  colour_;
    Vector3 worldPosition_;  float worldInnerRadius_, worldOuterRadius_;
    bool    dynamic_;
    uint32  terrainTimestamp_;
    Vector4 terrainLight_[3];   // 用于地形着色器常量
};
```

设计要点：
- **模型空间 vs 世界空间**：`position_` / `direction_` 是光源自身的局部属性（在 `LightContainer::addToSelf` 时被复制为原始值），`worldXxx` 是经 `worldTransform(matrix)` 应用变换后的世界空间值。这样同一光源可以挂到不同空间（角色持火把跟随、场景固定光源）。
- **双半径衰减**：`innerRadius_` 内完全强度，`outerRadius_` 外为零，中间线性插值。`attenuation(bb)` 给定 AABB 返回"该 box 内最坏衰减"，用于光源剔除——若 `attenuation < 阈值` 则不参与该对象的着色。
- **priority / dynamic**：`priority` 是手工优先级（高优先级先填入固定函数有限的 8 个光源槽），`dynamic` 标记光源是否动态（动态光源在 terrain 光照预计算中要重新生成）。
- **EDITOR_ENABLED 下的 multiplier**：编辑器中可设置光照强度倍数（运行时为 1.0f）。

`SpotLight` 在 OmniLight 基础上增加 `direction_` / `cosConeAngle_`（圆锥半角的余弦）和 `lightView_` / `lightBounds_`（用于阴影投影矩阵）。`DirectionalLight` 最简单，只有 `direction_` 和 `colour_`，无位置与衰减。

#### 5.8.2 `LightContainer`（光照桶）

[light_container.hpp](file:///workspace/src/lib/moo/light_container.hpp) 定义。它是引用计数的"光照集合"，按 Omni/Spot/Directional 三类分别存储：

```cpp
// light_container.hpp:34
class LightContainer : public SafeReferenceCount
{
public:
    LightContainer( const LightContainerPtr & pLC, const BoundingBox& bb,
                    bool limitToRenderable = false, bool dynamicOnly = false );
    void assign( const LightContainerPtr & pLC );
    void init( const LightContainerPtr & pLC, const BoundingBox& bb,
               bool limitToRenderable = false, bool dynamicOnly = false );
    void addToSelf( const LightContainerPtr & pLC,
                    bool addOmnis=true, bool addSpots=true, bool addDirectionals=true );

    const Colour& ambientColour() const;
    const DirectionalLightVector& directionals() const;
    const OmniLightVector& omnis() const;
    const SpotLightVector& spots() const;

    void addExtraOmnisInWorldSpace() const;
    void addExtraOmnisInModelSpace( const Matrix& invWorld ) const;
    static void addLightsInWorldSpace();
    static void addLightsInModelSpace( const Matrix& invWorld );
    void commitToFixedFunctionPipeline();
private:
    Colour                  ambientColour_;
    DirectionalLightVector  directionalLights_;
    OmniLightVector         omniLights_;
    SpotLightVector         spotLights_;
};
```

构造函数的 `bb` 参数是当前要渲染对象的包围盒，`limitToRenderable` 表示是否用 `Light::intersects(bb)` 过滤掉不影响该对象的光源，`dynamicOnly` 表示只取 `dynamic()==true` 的光源。`VisualHelper::start` 会用这些参数构造一个"对象可见光照子集"，避免把场景里全部 100+ 光源塞进固定函数管线的 8 槽。

##### addExtraOmnisInWorldSpace / ModelSpace

这两个 const 方法是静态全局光照注入点：`VisualHelper` 在每个 `Visual::draw` 开始时把它们推到当前 `LightContainer`，使"额外光源"（如附件在角色身上的火把）能进入光照计算。`addExtraOmnisInModelSpace(invWorld)` 把光源位置经 `invWorld` 转到模型空间，便于着色器直接用模型空间法线 dot 光向。

##### commitToFixedFunctionPipeline

把容器内的 ambient + 前 N 个 directional/omni/spot 通过 `device->SetLight` / `LightEnable` 提交到 D3D9 固定函数管线。N 由 `MaxSimLights`（caps 决定，通常 8）限制，按 `priority` 排序后取前 N。Effect 路径不走这里，而是由 `EffectLightingSetter` 把光照转成 shader 常量（见 §4.7）。

#### 5.8.3 地形光照预计算

`OmniLight::getTerrainLight` / `SpotLight::getTerrainLight` 把光源压成 `Vector4[3]` 或 `Vector4[4]` 的着色器常量形式（位置/方向 + 颜色 + 衰减参数打包到 Vector4 的 xyzw），由 `Terrain::Renderer` 收集所有影响当前 block 的光源后批量提交到地形 effect。`terrainTimestamp_` 是缓存戳——同一光源在世界变换未变时只计算一次地形常量。

#### 5.8.4 雾：`FogHelper` 与 `RenderContext` 雾状态

`RenderContext` 持有四个雾状态：`fogEnabled_` / `fogColour_` / `fogNear_` / `fogFar_`，通过 getter/setter 暴露。`FogHelper` 命名空间负责实际下发到 D3D，关键在于**硬件能力探测**：

```cpp
// fog_helper.hpp:23
namespace FogHelper
{
    void setFog( float start, float end,
                D3DRENDERSTATETYPE fogState, D3DFOGMODE mode );
    void setFogStart( float start );
    void setFogEnd( float end );
    void setFogVertexMode(D3DFOGMODE mode);
    void setFogTableMode(D3DFOGMODE mode);
    bool tableModeSupported();      // 探测 D3DPRASTERCAPS_FOGTABLE
    void setFogColour( uint32 colour );
    void setFogEnable( bool state );
};
```

如果硬件不支持 table fog（`D3DPRASTERCAPS_FOGTABLE` 不在 `RasterCaps` 中），`setFogTableMode` 自动降级为 vertex fog。`FogHelper::setFog(start, end, fogState, mode)` 是组合 setter，根据 `tableModeSupported()` 选择 `D3DRS_FOGTABLEMODE` 还是 `D3DRS_FOGVERTEXMODE`。这是 BigWorld 兼容旧硬件（如 GeForce2 / Radeon 7000）的关键 workaround。

### 5.9 渲染状态设置器

moo 提供 4 个 RAII 风格的 scoped setter，构造时保存旧状态、设置新状态，析构时恢复。它们都是栈上对象，开销极小（不分配堆）。

#### 5.9.1 `CameraPlanesSetter`

[camera_planes_setter.hpp](file:///workspace/src/lib/moo/camera_planes_setter.hpp)：临时修改 `RenderContext::camera()` 的 near/far 平面，常用于：
- **粒子 / 透明物体的近距离渲染**：把 near 推到 0.01 防止粒子被近平面裁切。
- **shadow map 渲染**：用紧密的 near/far 提高深度精度。
- **天空盒渲染**：far 设为 1e10 让天空永远不被远裁切。

```cpp
// camera_planes_setter.hpp:24
class CameraPlanesSetter
{
public:
    CameraPlanesSetter( float nearPlane, float farPlane );
    ~CameraPlanesSetter();
private:
    float savedNearPlane_;
    float savedFarPlane_;
};
```

#### 5.9.2 `ScissorsSetter`

[scissors_setter.hpp](file:///workspace/src/lib/moo/scissors_setter.hpp)：封装 `SetScissorRect`。`isAvailable()` 检查 `D3DPRASTERCAPS_SCISSORTEST` caps。用于：
- **GUI 局部重绘**：只在脏矩形内渲染。
- **光晕 / lens flare**：只在屏幕中心区域绘制。
- **性能优化**：把粒子绘制限制在屏幕可见区域。

```cpp
// scissors_setter.hpp:24
class ScissorsSetter
{
public:
    ScissorsSetter();
    ScissorsSetter( uint32 x, uint32 y, uint32 width, uint32 height );
    ~ScissorsSetter();
    static bool isAvailable();
private:
    RECT oldRect_;
    RECT newRect_;
};
```

#### 5.9.3 `ViewportSetter`

[viewport_setter.hpp](file:///workspace/src/lib/moo/viewport_setter.hpp)：封装 `SetViewport`。提供 4 个构造重载——空构造（用 backbuffer 尺寸）、(w,h)、(x,y,w,h)、完整带 minZ/maxZ 版本。用于：
- **小地图 / HUD** 渲染到屏幕一角。
- **split-screen 多人**。
- **Render-to-texture**：设置与 RT 尺寸匹配的 viewport。

```cpp
// viewport_setter.hpp:24
class ViewportSetter
{
public:
    ViewportSetter( uint32 width, uint32 height, uint32 x, uint32 y,
                    float minZ, float maxZ );
    ~ViewportSetter();
private:
    void apply( DX::Viewport& v );
    DX::Viewport oldViewport_;
    DX::Viewport newViewport_;
};
```

#### 5.9.4 `RenderContextDebug`

[render_context_debug.hpp](file:///workspace/src/lib/moo/render_context_debug.hpp) 只在 `_DEBUG` 下编译，提供 `printRenderContextState(out, rc)` 把当前 RenderContext 的所有状态（device / 矩阵 / 雾 / 光照 / 缓存命中数）输出到流。调试渲染异常时调用，是 moo 的"状态快照"工具。

### 5.10 资源加载上下文与流式数据

#### 5.10.1 `ResourceLoadContext`

[resource_load_context.hpp](file:///workspace/src/lib/moo/resource_load_context.hpp) 极简——只维护一个线程局部字符串栈：

```cpp
// resource_load_context.hpp:23
class ResourceLoadContext
{
public:
    static void push( const std::string & requester );
    static void pop();
    static std::string formattedRequesterString();
};

class ScopedResourceLoadContext
{
public:
    ScopedResourceLoadContext( const std::string & requester )
    { ResourceLoadContext::push( requester ); }
    ~ScopedResourceLoadContext()
    { ResourceLoadContext::pop(); }
};
```

`formattedRequesterString()` 返回形如 `"models/player/char.visual -> materials/skin.mat -> textures/skin.dds"` 的链式字符串。当纹理 / visual / material 加载失败或断言触发时，错误消息会带上这个串，帮助定位是哪个上层资源导致了下层加载问题。`Visual` 构造函数典型用法：

```cpp
// visual.cpp:478
Moo::ScopedResourceLoadContext resLoadCtx( BWResource::getFilename( resourceID ) );
```

#### 5.10.2 `StreamedDataCache`（.anca 流式动画缓存）

[streamed_data_cache.hpp](file:///workspace/src/lib/moo/streamed_data_cache.hpp) 处理 `.anca`（animation cache）文件的流式加载。`.anca` 是把多个 `.animation` 文件打包后预计算出来的二进制缓存，目的是让大型动画（如过场动画）可以分块流式加载，而不是一开始就全部读入内存。

```cpp
// streamed_data_cache.hpp:23
class StreamedDataCache
{
public:
    static const int CACHE_VERSION;
    StreamedDataCache( const std::string & resourceID, bool createIfAbsent );

    struct EntryInfo       // .anca 文件中每条动画的目录项
    {
        uint32 preloadSize_;    // 预加载大小
        uint32 streamSize_;     // 流式部分大小
        uint32 fileOffset_;      // 在 .anca 中的偏移
        uint32 version_;        // 数据版本
        uint64 modifiedTime_;   // 源动画修改时间
    };

    class Tracker : public SafeReferenceCount   // 加载请求状态
    {
    public:
        int offset_;
        int finished_;   // 0=未读, 1=成功, -1=失败
    };

    const EntryInfo* findEntryData( const std::string& name,
                                     uint32 minVersion, uint64 minModified );
    TrackerPtr load( uint32 offset, BinaryPtr pData );
    TrackerPtr save( uint32 offset, uint32 size, const void* data );
    bool fileAccessDone( TrackerPtr tracker );
};
```

##### .anca 文件布局

```
[文件头 / 目录 EntryInfo[n]]
[preload 数据块 1][streamed 数据块 1]    <- 每条动画
[preload 数据块 2][streamed 数据块 2]
...
```

每条动画数据本身的内部结构（见 streamed_data_cache.hpp 注释）：

```
uint32   preload data size (含 streamed sizes)
......   preload data (标准 animation 格式)
uint32*  streamed data size for each block in the anim
------   (preload 数据结束)
......   streamed data
```

- **preload**：包含骨骼层级、通道元数据、关键帧索引等小数据，启动时一次性读入。
- **streamed**：包含实际关键帧采样数据，按需在动画播放到对应时间点时由 `StreamedAnimationChannel` 触发 `load(offset, pData)` 异步读取。
- `Tracker` 是异步加载句柄，`finished_` 字段由后台 IO 线程置位，主线程通过 `fileAccessDone(tracker)` 轮询。
- `findEntryData(name, minVersion, minModified)` 检查缓存是否过期（源 `.animation` 比缓存新则需重建），返回 `EntryInfo*` 或 NULL。

`deleteOnClose(true)` 让 `StreamedDataCache` 析构时删除 `.anca` 文件，用于编辑器中"重建缓存"流程。

---

## 6. 与其它模块的协作

moo 是客户端的"渲染基座"，几乎所有视觉相关模块都直接调用它。下面列出六个最重要的协作面。

### 6.1 与 `resmgr`（资源管理）

所有 moo 的磁盘资源都通过 [BWResource](file:///workspace/src/lib/resmgr/bwresource.hpp) 加载：

| moo 类 | 加载的资源 | resmgr 接口 |
|---|---|---|
| `TextureManager` | `.dds` / `.tga` / `.png` | `BWResource::openSection(path)` → `BinaryPtr` |
| `VisualManager` | `.visual`（XML）+ `.primitive`（二进制） | `BWResource::openSection` + `PrimitiveFile` |
| `EffectManager` | `.fx`（D3DX Effect） | `BWResource::openSection` → `BinaryPtr` 喂 `D3DXCreateEffectFromFile` |
| `AnimationManager` | `.animation` / `.anca` | `BWResource::openSection` + `StreamedDataCache` |
| `NodeCatalogue` | `.node` | `BWResource::openSection` |
| `Material` (legacy) | `.xml` 材质定义 | `DataSection` 树 |

`BWResource` 提供 `DataSectionPtr`（XML 抽象）和 `BinaryPtr`（原始字节）。moo 不直接调用 `fopen` / `CreateFile`，所有 IO 都走 resmgr，这样 resmgr 可以：
- 从 ZIP 包加载（`BWResource::instance().addPath("data.zip")`）。
- 后台异步预读（`BgTaskManager`）。
- 跨平台路径规整化。

[primitive_file.hpp](file:///workspace/src/lib/resmgr/primitive_file.hpp) 专门处理 `.primitive` 二进制文件，提供 `PrimitiveFile::open(BinaryPtr)` 返回结构化访问器，moo 的 `Primitive` / `Vertices` 通过它读取顶点 / 索引数据。

### 6.2 与 `model`（模型系统）

[model](file:///workspace/src/lib/model/) 是 moo 之上的一层"角色 / 装备"抽象：

- `SuperModel` 持有多个 `Model`，每个 `Model` 持有一个 `Visual` + 一组 `Animation`。
- `Model::tick(time)` 计算骨骼姿态（`Node::blend`），更新 `worldTransform_`。
- `Model::draw()` 调用 `Visual::draw(ignoreBB, useDefaultPose=false)`，传入已计算的骨骼。
- 装备（武器、护甲）通过 `ModelMutator` 替换 `Visual` 的 `RenderSet` / `PrimitiveGroup` 的 `EffectMaterial`（`Visual::overrideMaterial`）。
- Morph 动画通过 `MorphVertices::addMorphValue` 注入，`Model::tick` 每帧调一次 `MorphVertices::incrementCookie()`。
- `Model` 的 LOD 由 `RenderContext::lodValue()` 决定，`lodFar` / `lodPower` 控制衰减曲线。

`Model` 也直接使用 `LightContainer`：角色身上的"自发光"装备（如发光剑）会通过 `LightContainer::addOmni` 注入到场景光照桶。

### 6.3 与 `chunk`（场景分块）

[chunk](file:///workspace/src/lib/chunk/) 把世界切成 64×64 米的方块，每块内嵌静态几何：

- `ChunkModel` 持有 `VisualPtr`，在 `Chunk::draw` 时遍历可见 chunk 调用 `Visual::draw`。
- `ChunkManager::draw` 是 moo 渲染管线的"场景遍历入口"（见 §8）。
- chunk 内的静态光照通过 `LightContainer` 预计算并存到 `chunk.localLights`，draw 时压入 RenderContext。
- chunk 间的 portal 可见性用 `Visual::PortalData` 描述——portal 是导出的多边形顶点环，标记"穿过此多边形可见邻 chunk"。
- flora（草地）通过 `LOAD_GEOMETRY_ONLY` 标志只加载 Visual 的几何，跳过 BSP / 材质，大幅减少内存。
- `ChunkManager` 维护"待 batch 的 Visual 列表"，每帧调用 `Visual::drawBatches()` 一次性提交所有同 material 实例（见 §5.1.4）。

### 6.4 与 `terrain`（地形）

[terrain](file:///workspace/src/lib/terrain/) 完全基于 moo 的 Effect / RenderTarget / VertexBuffer：

- 地形 effect 是普通 `.fx`，由 `EffectManager::load` 加载。
- 地形材质属性（地表类型、纹理层）通过 `EffectMaterial::setProperty` 设置。
- 地形 normal map / splat map 通过 `RenderTarget` 在 GPU 生成。
- 地形 LOD 用 `RenderContext::lodValue` 配合 `Terrain::Renderer` 自己的 LOD 算法。
- 地形光照：`OmniLight::getTerrainLight` / `SpotLight::getTerrainLight` 把光源压成 shader 常量，`Terrain::Block::draw` 收集所有影响当前 block 的光源后批量提交。
- `HoleMap`（地形打洞）用 `RenderTarget` 渲染 mask 纹理。
- 地形阴影贴图：定向光经 `SpotLight::worldTransform` 计算光照视图矩阵，渲染到 `RenderTarget`，下帧作为 shader 输入。

### 6.5 与 `romp`（环境与后处理）

[romp](file:///workspace/src/lib/romp/) 是"Render Output / Management Pipeline"——环境与后处理：

- `EnviroMinder` 是 moo 渲染管线的总指挥（见 §8）：每帧先设置 `RenderContext` 的雾、光照、天空、远剪裁面，再触发 `ChunkManager::draw`。
- `EnviroMinder` 持有 `LightContainerPtr`（场景主光照）和 `DirectionalLightPtr`（太阳光），通过 `RenderContext::lightContainer` 提交。
- `SkyDome` / `Cloud` 用 `Visual` + 自定义 `EffectMaterial` 渲染。
- 后处理：`PostProcessing::Chain` 持有 `RenderTarget` 链，每帧 `draw` 调用 `RenderContext::pushRenderTarget` / `popRenderTarget` 切换渲染目标，把场景 RT 作为输入纹理做模糊 / bloom / 色调映射。
- `Rain` / `Snow` 粒子用 `EffectMaterial` + 动态 `DynamicVertexBuffer<VertexXYZDTV>`，每帧 CPU 更新位置。
- `TimeOfDay` 控制太阳方向与色温，通过 `DirectionalLight::direction` / `colour` 实时改光照。
- `romp` 也负责 device lost 时的恢复（`EnviroMinder::draw` 会检查 `RenderContext::checkDevice`）。

### 6.6 与 `particle` / `gui` / `physics`

- **particle**：`ParticleSystem` 用 `DynamicVertexBuffer<VertexXYZDTV>` 流式上传粒子，`EffectMaterial` 渲染。粒子排序走 `VisualChannels::sorted`。
- **gui**：`GUI::Manager` 用固定函数管线（`Material` 而非 `EffectMaterial`），`DynamicVertexBuffer<VertexXYZUV>` 渲染四边形。`ViewportSetter` 限制绘制区域到控件 rect。
- **physics**：`Visual` 的 `pBSPTree()` 返回 `BSPTree*`，由 `physics2` 模块用于碰撞检测。`BSPProxy` 是引用计数代理，避免 `BSPTree` 被多处共享时重复析构。`Visual::loadBSPVisual` 加载 `.bsp.visual`（只含 BSP 的特殊 visual）。

---

## 7. moo 初始化流程

moo 的初始化分两阶段：**库初始化**（`Moo::init()`）和**设备创建**（`RenderContext::createDevice`）。前者只创建 `RenderContext` 单例，后者才真正创建 D3D device。

### 7.1 库初始化

[init.cpp](file:///workspace/src/lib/moo/init.cpp)：

```cpp
// init.cpp:24
RenderContext*  g_RC = NULL;

bool init()
{
    BW_GUARD;
    if ( !s_initialised )
    {
        MF_ASSERT_DEV( g_RC == NULL );
        if( g_RC == NULL )
            g_RC = new RenderContext();
        s_initialised = g_RC->init();
        return s_initialised;
    }
    return true;
}

void fini()
{
    BW_GUARD;
    if ( s_initialised )
    {
        g_RC->fini();
        delete g_RC;
        g_RC = NULL;
        s_initialised = false;
    }
}
```

`RenderContext::init()` 主要做：
1. `d3d_ = Direct3DCreate9(D3D_SDK_VERSION)` —— 创建 `IDirect3D9`。
2. 枚举所有 adapter，填充 `devices_` vector（每个含 `D3DCAPS9` / 显示模式列表 / `compatibilityFlags_`）。
3. 初始化 `TextureManager` / `EffectManager` / `VisualManager` / `AnimationManager` / `VerticesManager` / `PrimitiveManager` / `NodeCatalogue` 等单例。
4. 注册 `RenderContextCallback`（`EffectManager` 等需要 device lost 通知的对象）。
5. `initRenderStates()` —— 把 `rsCache_` / `tssCache_` / `sampCache_` / `textureCache_` 的 `Id` 全置为 `cacheValidityId_ + 1`，强制下次 `setXxx` 真正下发到 D3D。

此时**没有 D3D device**，不能创建 GPU 资源。资源 manager 只是注册了，可以接受 `get(resourceID)` 调用，但 `load()` 会延迟到 device 创建后才执行（或返回未就绪的资源，由调用者轮询 `readyToRender`）。

### 7.2 设备创建

`RenderContext::createDevice(hWnd, deviceIndex, modeIndex, windowed, wantStencil, windowedSize, hideCursor, forceRef, useWrapper)` 是真正的设备创建入口。流程：

```
createDevice
├── 1. 选择 adapter / mode
├── 2. fillPresentationParameters()           // 填 D3DPRESENT_PARAMETERS
│     ├── BackBufferWidth/Height
│     ├── BackBufferFormat
│     ├── BackBufferCount (1 或 2 for tripleBuffering)
│     ├── MultiSampleType / Quality
│     ├── SwapEffect (D3DSWAPEFFECT_DISCARD)
│     ├── hDeviceWindow
│     ├── Windowed
│     ├── EnableAutoDepthStencil = true
│     ├── AutoDepthStencilFormat (经 getMatchingZBufferFormat 选择)
│     ├── Flags (D3DPRESENTFLAG_LOCKABLE_BACKBUFFER 等)
│     ├── FullScreen_RefreshRateInHz
│     └── PresentationInterval (D3DPRESENT_INTERVAL_ONE if waitForVBL)
├── 3. 选择 BehaviorFlags
│     ├── D3DCREATE_HARDWARE_VERTEXPROCESSING  (默认)
│     ├── D3DCREATE_MIXED_VERTEXPROCESSING     (mixedVertexProcessing_ = true)
│     └── D3DCREATE_SOFTWARE_VERTEXPROCESSING (forceRef 或硬件不足)
├── 4. d3d_->CreateDevice(...)
│     └── 如果 useWrapper：DX::createDeviceWrapper(deviceD3D)
│           返回 DeviceWrapper 包装对象（见 §2.5）
├── 5. 创建 backbuffer surface 引用 (backBufferDesc_)
├── 6. createSecondSurface()                  // MRT 第二 RT 纹理
├── 7. updateDeviceInfo()                     // 填充 caps / psVersion / vsVersion
├── 8. initRenderStates()                     // 重置缓存 Id
├── 9. DeviceCallback::createAllUnmanaged()   // 通知所有资源重建 unmanaged
├── 10. DeviceCallback::createAllManaged()    // 通知所有资源重建 managed
└── 11. setGammaCorrection()                  // 应用 Gamma 校正
```

##### PresentationParameters 的关键决策

- **Z buffer 格式**：`getMatchingZBufferFormat(colourFmt, stencilWanted, &stencilAvailable)` 查表 + 查 caps。`stencilWanted=true` 时优先 `D3DFMT_D24S8`，否则 `D3DFMT_D24X8`（节省 8 位 stencil 带宽）。若硬件不支持 stencil，回退到 `D3DFMT_D16` 并设 `stencilAvailable_ = false`。
- **`mixedVertexProcessing_`**：当硬件 caps 不足（如 `MaxVertexShaderConst < 32`）时启用，允许部分顶点处理走软件路径。`Visual::draw` 会检查 `rc().mixedVertexProcessing()` 决定 VB 用硬件还是软件池。
- **`useWrapper`**：开启 [moo_dx.hpp](file:///workspace/src/lib/moo/moo_dx.hpp) 中的 `DeviceWrapper`，把所有 D3D 调用记录到 `CommandBuffer`，由独立 D3D 线程异步执行。这是 BigWorld 的多线程渲染基础。

### 7.3 设备丢失与恢复

设备丢失（device lost）发生在：ALT+TAB 切换、屏幕保护、电源管理、显示器分辨率改变。`RenderContext::checkDevice(reset*)` 每帧开头被调用，检查 `device_->TestCooperativeLevel()`：

- 返回 `D3DERR_DEVICELOST`：设备已丢失，跳过本帧渲染（仅 `present`）。
- 返回 `D3DERR_DEVICENOTRESET`：可以恢复，调用 `resetDevice()`。

`resetDevice()` 流程：

```
resetDevice
├── 1. DeviceCallback::deleteAllUnmanaged()    // 释放所有 D3DPOOL_DEFAULT 资源
├── 2. device_->Reset(&presentParameters_)     // D3D Reset
├── 3. DeviceCallback::createAllUnmanaged()    // 重建 unmanaged 资源
└── 4. invalidateStateCache()                 // cacheValidityId_++，强制状态全重设
```

注意 `D3DPOOL_MANAGED` 资源由 D3D runtime 自动管理，不需要 delete/create。`DeviceCallback` 主要服务 `RenderTarget` / `DynamicVertexBuffer` / `DynamicIndexBuffer` / `VertexDeclaration`（部分情况下）等 default pool 资源。

### 7.4 关闭流程

```
Moo::fini
├── g_RC->fini()
│     ├── DeviceCallback::deleteAllUnmanaged()
│     ├── DeviceCallback::deleteAllManaged()
│     ├── releaseDevice()                    // device_.release(), d3d_.release()
│     └── RenderContextCallback::fini()      // 通知所有 callback 释放
├── delete g_RC
└── g_RC = NULL
```

各资源 manager 的 `fini()` 通常由 `RenderContextCallback::fini()` 触发，而不是显式调用——它们在构造时注册为 `RenderContextCallback` 派生类，析构时被统一通知。

---

## 8. 一帧的渲染管线

下面是一帧的完整渲染流水线，从 `romp` 入口到 `present`：

### 8.1 高层流程（ASCII 时序图）

```
[主线程]                                  [D3D线程（若用Wrapper）]
App::tick
│
├─ Physics::tick                          │
├─ Model::tick (骨骼 / 动画)              │
│    └─ Node::blendTransform              │
│    └─ MorphVertices::addMorphValue      │
├─ Chunk::tick (可见性 / 流式加载)         │
├─ EnviroMinder::tick (天气 / 时间)       │
│
EnviroMinder::draw ─────────────────────────┐
│                                          │
│  RenderContext::beginScene               │
│  ├─ checkDevice                          │
│  ├─ device->BeginScene (或 wrapper 队列) │──▶ CommandBuffer::BeginScene
│  └─ clearBindings / invalidateStateCache │
│                                          │
│  // 1. 远景：天空 + 云                   │
│  Sky::draw  → Visual::draw               │
│  Cloud::draw → VertexShader + RT         │
│                                          │
│  // 2. 主场景                            │
│  setView / setProjection / setWorld      │
│  LightContainer::commitToFFPipeline     │
│  FogHelper::setFog(...)                  │
│                                          │
│  ChunkManager::draw                      │
│  ├─ for each visible chunk:              │
│  │   └─ ChunkModel::Visual::draw         │
│  │       └─ Visual::batch                │──▶ CommandBuffer::DrawIndexedPrim
│  ├─ Visual::drawBatches                  │   （延迟到此处一次性提交）
│  ├─ Flora::draw                           │
│  └─ Water::draw → RenderTarget 折射      │
│                                          │
│  // 3. 动态对象                          │
│  for each entity:                        │
│     SuperModel::draw → Visual::draw      │
│                                          │
│  // 4. 粒子 / 后处理通道                 │
│  VisualChannels::draw                     │
│  ├─ solid channel: 不透明               │
│  ├─ sorted channel: 半透明（按深度排序） │
│  ├─ shimmer channel: 闪光                │
│  └─ distortion channel: 折射 / 热扭曲    │
│                                          │
│  // 5. 后处理                            │
│  PostProcessing::Chain::draw             │
│  ├─ pushRenderTarget(RT1)                │
│  ├─ drawFullscreenQuad(bloom shader)     │
│  ├─ popRenderTarget                     │
│  └─ ... (motion blur / DoF / tone map)  │
│                                          │
│  // 6. GUI                               │
│  GUI::Manager::draw                      │
│                                          │
│  RenderContext::endScene                 │
│  RenderContext::present                  │──▶ D3D Present
│                                          │
│  RenderContext::nextFrame                │
│  ├─ primitiveGroupCount_ = 0             │
│  ├─ drawCount_ = 0                       │
│  ├─ MorphVertices::incrementCookie       │
│  ├─ lastFrameProfilingData_ = live      │
│  └─ currentFrame_++                      │
│                                          │
│  // 7. 资源加载（如果 GPU 空闲）         │
│  preloadDeviceResources(timeLimitMs)     │
└──────────────────────────────────────────┘
```

### 8.2 `RenderContext` 在一帧中的角色

`RenderContext` 是状态机，一帧中状态变化时序：

| 阶段 | RenderContext 状态变化 | 缓存命中 |
|---|---|---|
| beginScene | `beginSceneCount_++`，开始记录 | 状态全 miss（cacheValidityId 未变但状态可能被外部改） |
| Sky draw | `view/projection/world` 不变，仅切换 material effect | effect state 全命中 |
| Terrain draw | `world = identity`，`lightContainer` 切换 | 光照常量 miss，其余命中 |
| Chunk draw | `world` 每实例变，`view/proj` 不变 | world 矩阵 miss（每实例必下发），其余命中 |
| Model draw | `world` 每角色变，骨骼矩阵走 shader 常量 | 同上 |
| Particles | `world = identity`，`blendMode` 频繁切换 | blend state miss，其余命中 |
| PostFX | `pushRenderTarget` 切 RT，`viewport` 变 | RT + viewport miss，shader 大切换 |
| GUI | `viewport` 切到控件 rect，固定函数管线 | shader miss（切回 FFP），其余命中 |
| endScene / present | `presentParameters_` 用到，`currentFrame_++` | — |

### 8.3 State Cache 命中分析

`setRenderState` 的缓存逻辑（伪代码）：

```cpp
uint32 RenderContext::setRenderState(D3DRENDERSTATETYPE state, uint32 value)
{
    RSCacheEntry& entry = rsCache_[state];
    if (entry.Id == cacheValidityId_ && entry.currentValue == value)
        return 0;                          // 命中，跳过 D3D 调用
    entry.Id = cacheValidityId_;
    entry.currentValue = value;
    device_->SetRenderState(state, value);  // 真正下发
    ++liveProfilingData_.nDrawcalls_;       // 仅统计性计数
    return 1;
}
```

典型一帧中，`setRenderState` 调用次数可达 **几十万次**，但实际下发 D3D 的只有几千次——这是 moo 性能的关键。`EffectStateManager` 进一步把 D3DX Effect 内部的状态变更也走这条缓存路径，避免 Effect `CommitChanges` 时盲目重设所有状态。

### 8.4 `Visual::draw` 内部流程

```
Visual::draw(ignoreBB, useDefaultPose)
│
├─ 1. shouldDraw 检查（VisualHelper）
│     ├─ 包围盒可见性（视锥剔除）
│     ├─ LOD 距离检查（rc().lodValue()）
│     └─ drawCount_ 限制（每帧最大绘制数）
│
├─ 2. VisualHelper::start
│     ├─ 计算 worldSpaceBB
│     ├─ 保存 RenderContext 当前 lightContainer
│     ├─ 构造子 LightContainer（limitToRenderable=true）
│     ├─ addLightsInWorldSpace / addLightsInModelSpace(invWorld)
│     └─ 计算 worldViewProjection 矩阵
│
├─ 3. for each RenderSet:
│     ├─ 设置 transformNodes_ 的世界矩阵（骨骼）
│     ├─ EffectVisualContextSetter（设置 effect 的 bone matrices 常量）
│     ├─ for each Geometry:
│     │   ├─ vertices_->setVertices(software)        // 绑定 VB
│     │   ├─ primitives_->setPrimitives()             // 绑定 IB
│     │   └─ for each PrimitiveGroup:
│     │       ├─ material_->begin()                   // Effect Begin
│     │       ├─ for each pass:
│     │       │   ├─ material_->beginPass(i)
│     │       │   ├─ material_->commitChanges()       // 提交属性
│     │       │   ├─ drawPrimitiveGroup(groupIndex_)  // DrawIndexedPrimitive
│     │       │   └─ material_->endPass()
│     │       └─ material_->end()
│     └─ VisualHelper::fini（恢复 LightContainer）
│
└─ 4. 如果 batch 模式：记录到 s_batches_，延迟到 drawBatches 提交
```

### 8.5 `VisualChannels` 延迟通道

`VisualChannels`（[visual_channels.hpp](file:///workspace/src/lib/moo/visual_channels.hpp)）是 moo 的"透明物体渲染调度器"。4 个通道：

| 通道 | 用途 | 排序 | 深度写 |
|---|---|---|---|
| `solid` | 不透明物体 | 无 | 写 |
| `sorted` | 半透明（玻璃、水、粒子） | 按视距降序（远→近） | 不写 |
| `shimmer` | 闪光（镜头光晕、 bloom 源） | 无 | 不写 |
| `distortion` | 折射（热扭曲、玻璃扭曲） | 无 | 不写 |

调用 `Visual::draw` 时，如果 `material` 标记为某通道，`Visual` 不立即绘制，而是把"绘制命令 + 当前 world 矩阵 + material 快照"压入对应通道队列。`VisualChannels::draw(channel)` 在主渲染流程后统一执行：sorted 通道会按 `worldViewPosition.z` 排序所有实例，再依次绘制。

这是 moo 处理透明物体的核心机制——保证半透明物体正确混合（后绘制近的覆盖远的），同时减少 `RenderContext` 状态切换（同通道内 material 相近的批量绘制）。

---

## 9. `.visual` 文件格式详解

`.visual` 是 BigWorld 的网格资产格式，由模型导出工具（3ds Max / Maya 插件）生成。它由两部分组成：
1. **`.visual`**：XML 文件，描述结构（rendersets / geometry / materials / portals / BSP 引用）。
2. **`.primitive`**：二进制文件，包含顶点 / 索引 / morph 数据。

加载时 `VisualManager::get("foo.visual")` 同时打开这两个文件，由 `VisualLoader<Visual>` 模板类驱动解析。

### 9.1 `.visual` XML 结构

```xml
<!-- 例: models/player/char.visual -->
<root>
  <boundingBox>
    <min x="-0.5" y="-0.5" z="0.0"/>
    <max x="0.5"  y="0.5"  z="1.8"/>
  </boundingBox>

  <node>  <!-- 引用 .node 文件（骨骼层级） -->
    <label>root</label>
    <file>char.node</file>
  </node>

  <renderSet>  <!-- 一组按骨骼分组的几何 -->
    <treatAsWorldSpaceObject>false</treatAsWorldSpaceObject>
    <node>  <!-- 此 renderset 涉及的骨骼 -->
      <label>Bip01_Spine</label>
      <file>char.node</file>
    </node>
    <node>...</node>

    <geometry>  <!-- 一份顶点 + 索引 + 材质组 -->
      <vertices>
        <resourceID>char_primitive</resourceID>
        <format>xyznuviiiwwtb</format>  <!-- 顶点格式串 -->
        <nVertices>1234</nVertices>
      </vertices>
      <primitive>
        <resourceID>char_primitive</resourceID>
        <nPrimGroups>3</nPrimGroups>
      </primitive>
      <primitiveGroup>
        <material>m_skin</material>     <!-- material 标识符 -->
        <groupIndex>0</groupIndex>        <!-- Primitive::PrimGroup 索引 -->
      </primitiveGroup>
      <primitiveGroup>...</primitiveGroup>
    </geometry>
    <geometry>...</geometry>
  </renderSet>
  <renderSet>...</renderSet>

  <portal>
    <flags>0</flags>
    <vertex x="..." y="..." z="..."/>
    <vertex>...</vertex>
  </portal>

  <bsp>  <!-- 可见性 / 碰撞 BSP -->
    <file>char_bsp.visual</file>
  </bsp>
</root>
```

字段说明：

| 标签 | 含义 |
|---|---|
| `boundingBox` | 模型局部空间 AABB，用于剔除与碰撞 |
| `node` | 骨骼节点引用，`label` 是节点名，`file` 指向 `.node` 文件（多个 visual 可共享同一 `.node`） |
| `renderSet` | 一组共享骨骼的几何（见 §5.1.1） |
| `treatAsWorldSpaceObject` | true 时几何已在世界空间，不应用 world 矩阵（用于 chunk 静态物） |
| `geometry` | 顶点 + 索引 + 材质组三元组 |
| `vertices` / `primitive` | `resourceID` 都指向同一 `.primitive` 文件（共享二进制） |
| `primitiveGroup` | `material` 是材质标识符，到 `Materials::find` 查找对应 `EffectMaterial`；`groupIndex` 是 `.primitive` 内 `PrimitiveGroup` 数组的下标 |
| `portal` | chunk 间可见性多边形 |
| `bsp` | 引用独立的 `.bsp.visual`，只含 BSP 树，用于碰撞 |

### 9.2 `.primitive` 二进制布局

[primitive_file_structs.hpp](file:///workspace/src/lib/moo/primitive_file_structs.hpp) 定义头部结构。整体布局：

```
┌────────────────────────────────────────────────────┐
│ File Header (固定，由 PrimitiveFile 解析)          │
├────────────────────────────────────────────────────┤
│ VertexHeader                                       │
│   char vertexFormat_[64];   // 例 "xyznuviiiwwtb"  │
│   int   nVertices_;                                  │
├────────────────────────────────────────────────────┤
│ VertexData[nVertices_]    // 由 vertexFormat_ 解析 │
│   每个顶点结构由 vertex_formats.hpp 的对应类型决定  │
│   例: VertexXYZNUV (28B) / VertexXYZNUVIIIWW (40B) │
├────────────────────────────────────────────────────┤
│ IndexHeader                                        │
│   char indexFormat_[64];    // "list" / "strip"     │
│   int   nIndices_;                                  │
│   int   nTriangleGroups_;                           │
├────────────────────────────────────────────────────┤
│ IndexData[nIndices_]        // uint16 / uint32     │
├────────────────────────────────────────────────────┤
│ PrimitiveGroup[nTriangleGroups_]                   │
│   见 primitive_file_structs.hpp:65                 │
│   struct PrimitiveGroup {                          │
│     int startIndex_;                                │
│     int nPrimitives_;                               │
│     int startVertex_;                               │
│     int nVertices_;                                  │
│   }                                                 │
├────────────────────────────────────────────────────┤
│ MorphHeader (可选)                                │
│   int version_;   int nMorphTargets_;              │
├────────────────────────────────────────────────────┤
│ MorphTargetHeader[nMorphTargets_] × ...            │
│   { char identifier_[64]; int channelIndex_;       │
│     int nVertices_; }                              │
├────────────────────────────────────────────────────┤
│ MorphVertex[nMorphTargets_][nVertices_]            │
│   { Vector3 delta_; int index_; }                  │
└────────────────────────────────────────────────────┘
```

##### vertexFormat_ 串约定

`vertexFormat_` 是 64 字节字符串，命名 moo 顶点类型的缩写：

| 串 | 对应类型 | 含义 |
|---|---|---|
| `xyznuv` | `VertexXYZNUV` | 位置/法线/UV（无骨骼） |
| `xyznuvtb` | `VertexXYZNUVTB` | + tangent/binormal（bump map） |
| `xyznuviiiww` | `VertexXYZNUVIIIWW` | + 3 骨骼索引 + 3 权重 |
| `xyznuviiiwwtb` | `VertexXYZNUVIIIWWTB` | + tangent/binormal（带骨骼 bump） |
| `xyznuviiiwwpc` | `VertexXYZNUVIIIWWPC` | + packed colour（静态光照） |
| `xyznduv` | `VertexXYZNDUV` | 位置/法线/diffuse/UV（固定函数） |
| `xyzl` | `VertexXYZL` | 位置 + 顶点色（仅 BSP 渲染用） |

解析时 `Vertices::load` 读 `vertexFormat_` 串，匹配到对应类型，按其 `sizeof` 计算每顶点字节数，从 `VertexData` 区读出顶点数组。`MorphVertices::load` 额外读 `MorphHeader` 与后续 morph 数据。

### 9.3 `PrimitiveGroup` 与 `Visual::PrimitiveGroup` 的关系

注意 [primitive_file_structs.hpp](file:///workspace/src/lib/moo/primitive_file_structs.hpp) 中的 `Moo::PrimitiveGroup`（二进制文件中的）与 [visual.hpp](file:///workspace/src/lib/moo/visual.hpp) 中的 `Visual::PrimitiveGroup`（运行时的）是**两个不同结构**：

```cpp
// primitive_file_structs.hpp:65 - 文件中的
struct PrimitiveGroup
{
    int startIndex_;      // IB 中的起始索引
    int nPrimitives_;     // 三角形数
    int startVertex_;     // VB 中的起始顶点
    int nVertices_;       // 顶点数
};

// visual.hpp:197 - 运行时的
struct PrimitiveGroup
{
    uint32 groupIndex_;            // 指向文件中 PrimitiveGroup 的下标
    EffectMaterialPtr material_;   // 该组的材质
};
```

`.primitive` 文件描述"几何怎么分组"（按索引范围），`.visual` 文件描述"每组用什么材质"。运行时 `Visual::PrimitiveGroup::groupIndex_` 索引到 `Primitive::primGroups_[groupIndex_]` 拿到索引范围，配合 `material_` 一起提交到 `drawIndexedPrimitive`。

### 9.4 加载流程

```
VisualManager::get("char.visual")
│
├─ VisualMap 查找：命中则返回 VisualPtr
│
└─ 未命中：new Visual("char.visual")
   │
   ├─ VisualLoader<Visual> loader(baseName="char")
   │
   ├─ loader.getRootDataSection()
   │   └─ BWResource::openSection("char.visual")  → DataSectionPtr
   │
   ├─ ScopedResourceLoadContext("char.visual")
   │
   ├─ loader.loadBSPTree(pFoundBSP, bspMaterialIDs)
   │   └─ 若有 <bsp> 节点，loadBSPVisual("char_bsp.visual")
   │       └─ VisualManager::get("char_bsp.visual") 递归（只含 BSP）
   │
   ├─ loader.loadPrimitives("char")
   │   └─ PrimitiveFile::open(BWResource::openSection("char.primitive"))
   │       ├─ 读 VertexHeader → 选 VertexType
   │       ├─ 读 VertexData → VertexListBase
   │       ├─ 读 IndexHeader + IndexData → IndexBuffer
   │       └─ 读 PrimitiveGroup[] → Primitive::PrimGroup[]
   │
   ├─ loader.loadVertices(pVertices, vertexFormat)
   │   └─ 若有 MorphHeader → new MorphVertices
   │       else → new Vertices (普通模板实例)
   │
   ├─ loader.loadNodes(rootNode_)
   │   └─ NodeCatalogue::get("char.node")
   │       └─ new Node（递归建子节点树）
   │
   ├─ for each <renderSet> in .visual:
   │   ├─ RenderSet rs
   │   ├─ rs.treatAsWorldSpaceObject_ = ...
   │   ├─ for each <node> in renderSet:
   │   │   └─ rs.transformNodes_.push_back( rootNode_->find(label) )
   │   ├─ for each <geometry>:
   │   │   ├─ Geometry g
   │   │   ├─ g.vertices_ = VerticesManager::get(...)
   │   │   ├─ g.primitives_ = PrimitiveManager::get(...)
   │   │   └─ for each <primitiveGroup>:
   │   │       ├─ PrimitiveGroup pg
   │   │       ├─ pg.groupIndex_ = ...
   │   │       └─ pg.material_ = loadMaterial(materialIdentifier)
   │   │           └─ EffectMaterial::load(DataSection) 或 Material::load
   │   └─ rs.geometry_.push_back(g)
   │
   ├─ Visual::portals_ = loader.loadPortals()
   ├─ Visual::bb_ = loader.loadBoundingBox()
   └─ Visual::animations_ = loader.loadAnimations()  (可选)
```

`loadMaterial` 是关键：根据材质标识符查 `<materials>` 节，若 `<material>` 子节点带 `fx` 属性则加载 `EffectMaterial`，否则走 legacy `Material`（固定函数）。同一 `.visual` 多次引用相同 material 标识符时，`Materials::find` 命中复用。

---

## 10. 性能、调试与可观察性

### 10.1 `DogWatch` / `Profiler`

moo 大量使用 `cstdmf` 的 `DogWatch` 做分块性能计量。`Visual::draw` 顶部：

```cpp
// visual.cpp:56
PROFILER_DECLARE( Visual_draw, "Visual Draw" );
```

典型 DogWatch 标签：`"Visual Draw"` / `"PrimGB"`（PrimitiveGroup batcher）/ `"RecordAutos"`（记录自动常量）/ `"CommitChanges"` / `"Draw"`。它们在 `DogWatchManager` 中聚合，可通过 watcher 查看。

### 10.2 Watcher（`MF_WATCH`）

`RenderContext` 和各 manager 注册了大量 watcher 到 `Watcher` 模块，运行时通过控制台查看：

```cpp
// visual.cpp:75
MF_WATCH("Render/Visual/premadeBSP_Count", *this,
    &WatchOwner::premadeBSPCount, "The size of the premade BSP Cache");
```

典型 watcher 命名空间：`Render/Visual/*` / `Render/Texture/*` / `Render/Effect/*`。客户端运行时输入 `watch Render/Visual/premadeBSP_Count` 即可查看缓存大小。

### 10.3 `RenderContext` 绘制统计

`DrawcallProfilingData` 结构（[render_context.hpp](file:///workspace/src/lib/moo/render_context.hpp):106）记录每帧 drawcall 数与 primitive 数：

```cpp
struct DrawcallProfilingData
{
    uint32 nDrawcalls_;
    uint32 nPrimitives_;
};
// RenderContext 持有：
DrawcallProfilingData liveProfilingData_;        // 当前帧累加
DrawcallProfilingData lastFrameProfilingData_;   // 上一帧快照
```

`drawIndexedPrimitive` 等绘制函数内部 `++liveProfilingData_.nDrawcalls_`。`nextFrame()` 时 `lastFrame = live; live = 0`。通过 `lastFrameProfilingData()` 暴露给上层（HUD 显示"当前帧 X drawcalls / Y triangles"）。

### 10.4 `OcclusionQuery`

`RenderContext` 提供 `createOcclusionQuery()` / `beginQuery` / `endQuery` / `getQueryResult`，包装 `IDirect3DQuery9`（`D3DQUERYTYPE_OCCLUSION`）。典型用法：
- 在镜头看不到的区域绘制小代表几何，先 `beginQuery` / `draw` / `endQuery`。
- 下一帧 `getQueryResult(visiblePixels, query, false)` 拿到可见像素数，若为 0 则跳过该区域完整渲染。

这是 BigWorld 的"层次 Z 剔除"基础，用于大型场景中远距离 chunk 的可见性判断。

### 10.5 内存压力

`RenderContext::memoryCritical(bool)` 是显存压力开关。当 `getAvailableTextureMem()` 低于阈值时（由 `TextureManager` 监测），置为 true，触发：
- `TextureManager` 释放 LRU 纹理。
- `VisualManager` 释放 `fullHouse` 模式下的非引用 visual。
- 上层模块降级（如关闭高分辨率纹理 mip）。

### 10.6 资源预加载

`RenderContext::addPreloadResource(IDirect3DResource9*)` 把资源加入预加载队列，`preloadDeviceResources(timeLimitMs)` 在帧末尾空闲时调用 `resource->PreLoad()`（D3D9 的 `EvictManagedResources` 反操作），提前把纹理从磁盘搬到显存。这是避免渲染时"卡顿"（首次使用纹理时的 IO stall）的关键。

---

## 11. 关键设计要点回顾

### 11.1 缓存优先的设计哲学

moo 的核心性能策略是**"所有 D3D 调用先查缓存"**。`RenderContext` 持有：
- `rsCache_[D3DRS_MAX]`：210 个 render state。
- `tssCache_[D3DFFSTAGES_MAX][D3DTSS_MAX]`：8 stages × 33 states。
- `sampCache_[D3DSAMPSTAGES_MAX][D3DSAMP_MAX]`：261 sampler stages × 14 states（含 vertex texture sampler）。
- `textureCache_[D3DSAMPSTAGES_MAX]`：每 stage 一个 `BaseTexture*`。

每条记录带 `Id` 字段，与 `cacheValidityId_` 比对——device reset 时 `cacheValidityId_++` 一次让全表失效，避免逐条清零。`EffectStateManager` 把 D3DX Effect 内部状态变更也走同一路径，保证 Effect 切换不会盲目重设状态。

### 11.2 资源生命周期与引用计数

moo 的所有 GPU 资源都继承 `SafeReferenceCount`（线程安全的引用计数）或 `ReferenceCount`（非线程安全）。`SmartPointer<T>` 是侵入式智能指针。资源 manager（`TextureManager` / `VisualManager` 等）持有一个"主引用"，业务模块持附加引用——只有所有引用释放后资源才真正析构。`DeviceCallback` 是另一条独立的生命周期线，处理 device lost 时的 unmanaged 资源重建。

### 11.3 模板与运行时多态的混合

moo 大量使用模板处理"类型不同但逻辑相同"的场景：
- `DynamicVertexBuffer<VertexType>`：每个顶点类型一个 ring buffer 单例。
- `MorphTarget<VertexType>`：每种顶点格式独立的 morph apply。
- `VisualLoader<Visual>`：泛化 visual 加载，`VisualCompound` 复用同套机制。
- `SoftwareSkinner<VertexType>`：CPU 蒙皮按顶点类型特化。

而跨类型的接口（如 `BaseTexture::pTexture()` / `BaseSoftwareSkinner::transformVertices`）用虚函数。这种"模板内部 + 虚接口边界"是 C++ 渲染引擎的经典模式。

### 11.4 DeviceWrapper 与多线程渲染

`moo_dx.hpp` 中的 `DeviceWrapper` 是 BigWorld 多线程渲染的核心：它实现完整的 `IDirect3DDevice9` 接口，但每个方法把调用序列化到 `CommandBuffer`，由独立 D3D 线程异步执行。主线程不阻塞在 D3D 调用上，可以继续做逻辑 / 物理 / 动画 tick。`DX::flush(true)` 用于强制等待（如 device reset 前）。`WRAPPER_FLAG_*` 控制不同行为（immediate lock / deferred lock / zero texture lock / query issue flush）。

但 wrapper 也带来复杂性：所有 COM 对象都要 wrapper 化（`TextureWrapper` / `VertexBufferWrapper` / ...），引用计数要双重管理（wrapper 与底层 D3D 对象）。`WrapperStateManager` 是 wrapper 版本的 Effect state manager。这套机制默认关闭，仅在多线程渲染配置下启用。

### 11.5 兼容性标志

`CompatibilityFlag` 枚举（[render_context.hpp](file:///workspace/src/lib/moo/render_context.hpp):42）标记不同显卡厂商的怪癖：

```cpp
enum CompatibilityFlag
{
    COMPATIBILITYFLAG_NOOVERWRITE = 1 << 0,  // 不支持 D3DLOCK_NOOVERWRITE
    COMPATIBILITYFLAG_NVIDIA      = 1 << 1,
    COMPATIBILITYFLAG_ATI         = 1 << 2,
};
```

`DeviceInfo::compatibilityFlags_` 由 `updateDeviceInfo` 根据 `identifier_.VendorId` 设置。`DynamicVertexBuffer::lock` 检查 `NOOVERWRITE` 标志，决定是否使用 ring buffer（不支持则每次锁都创建新 VB，性能差但兼容）。这是 BigWorld 兼容 2005-2010 年代各种显卡的关键。

---

## 12. 文件索引

下表列出 moo 库的核心文件及其作用，便于按图索骥：

| 文件 | 作用 |
|---|---|
| [init.hpp](file:///workspace/src/lib/moo/init.hpp) / [init.cpp](file:///workspace/src/lib/moo/init.cpp) | 库初始化入口 |
| [render_context.hpp](file:///workspace/src/lib/moo/render_context.hpp) / .cpp / .ipp | 全局渲染上下文 |
| [render_context_callback.hpp](file:///workspace/src/lib/moo/render_context_callback.hpp) | RC 生命周期回调 |
| [render_context_debug.hpp](file:///workspace/src/lib/moo/render_context_debug.hpp) | 调试状态打印 |
| [moo_dx.hpp](file:///workspace/src/lib/moo/moo_dx.hpp) / .cpp | D3D 包装层 + DeviceWrapper |
| [device_callback.hpp](file:///workspace/src/lib/moo/device_callback.hpp) / .ipp | 设备事件回调接口 |
| [com_object_wrap.hpp](file:///workspace/src/lib/moo/com_object_wrap.hpp) | COM 智能指针 |
| [base_texture.hpp](file:///workspace/src/lib/moo/base_texture.hpp) / .ipp | 纹理基类 |
| [managed_texture.hpp](file:///workspace/src/lib/moo/managed_texture.hpp) / .ipp | 托管纹理 |
| [image_texture.hpp](file:///workspace/src/lib/moo/image_texture.hpp) | 内存纹理 |
| [animating_texture.hpp](file:///workspace/src/lib/moo/animating_texture.hpp) | 动画纹理 |
| [texture_manager.hpp](file:///workspace/src/lib/moo/texture_manager.hpp) / .ipp | 纹理管理器 |
| [texture_compressor.hpp](file:///workspace/src/lib/moo/texture_compressor.hpp) / .ipp | 纹理压缩 |
| [texture_aggregator.hpp](file:///workspace/src/lib/moo/texture_aggregator.hpp) | 纹理聚合 |
| [render_target.hpp](file:///workspace/src/lib/moo/render_target.hpp) / .cpp | 渲染目标 |
| [cube_render_target.hpp](file:///workspace/src/lib/moo/cube_render_target.hpp) / .cpp | 立方体 RT |
| [mrt_support.hpp](file:///workspace/src/lib/moo/mrt_support.hpp) | 多目标渲染 |
| [texturestage.hpp](file:///workspace/src/lib/moo/texturestage.hpp) / .ipp | 纹理阶段 |
| [vertex_buffer.hpp](file:///workspace/src/lib/moo/vertex_buffer.hpp) / .ipp | 顶点缓冲 |
| [index_buffer.hpp](file:///workspace/src/lib/moo/index_buffer.hpp) / .ipp | 索引缓冲 |
| [dynamic_index_buffer.hpp](file:///workspace/src/lib/moo/dynamic_index_buffer.hpp) | 动态 IB |
| [dynamic_vertex_buffer.hpp](file:///workspace/src/lib/moo/dynamic_vertex_buffer.hpp) | 动态 VB（ring buffer） |
| [vertex_declaration.hpp](file:///workspace/src/lib/moo/vertex_declaration.hpp) | 顶点声明 |
| [vertex_streams.hpp](file:///workspace/src/lib/moo/vertex_streams.hpp) | 顶点流 |
| [vertex_formats.hpp](file:///workspace/src/lib/moo/vertex_formats.hpp) | 顶点格式定义 |
| [vertices.hpp](file:///workspace/src/lib/moo/vertices.hpp) / .ipp | 顶点集合抽象 |
| [vertices_manager.hpp](file:///workspace/src/lib/moo/vertices_manager.hpp) | 顶点管理器 |
| [morph_vertices.hpp](file:///workspace/src/lib/moo/morph_vertices.hpp) | 形变顶点 |
| [primitive.hpp](file:///workspace/src/lib/moo/primitive.hpp) / .ipp | D3D primitive 抽象 |
| [primitive_manager.hpp](file:///workspace/src/lib/moo/primitive_manager.hpp) | primitive 管理器 |
| [primitive_file_structs.hpp](file:///workspace/src/lib/moo/primitive_file_structs.hpp) | .primitive 二进制结构 |
| [effect_manager.hpp](file:///workspace/src/lib/moo/effect_manager.hpp) / .ipp | Effect 管理器 |
| [managed_effect.hpp](file:///workspace/src/lib/moo/managed_effect.hpp) / .ipp / .cpp | D3DX Effect 包装 |
| [effect_material.hpp](file:///workspace/src/lib/moo/effect_material.hpp) / .ipp / .cpp | Effect 材质 |
| [material.hpp](file:///workspace/src/lib/moo/material.hpp) / .ipp / .cpp | 固定函数材质 |
| [material_loader.hpp](file:///workspace/src/lib/moo/material_loader.hpp) | 材质加载 |
| [effect_state_manager.hpp](file:///workspace/src/lib/moo/effect_state_manager.hpp) / .cpp | Effect 状态管理 |
| [effect_constant_value.hpp](file:///workspace/src/lib/moo/effect_constant_value.hpp) / .cpp | Effect 常量 |
| [effect_lighting_setter.hpp](file:///workspace/src/lib/moo/effect_lighting_setter.hpp) / .cpp | 光照常量注入 |
| [effect_visual_context.hpp](file:///workspace/src/lib/moo/effect_visual_context.hpp) / .cpp | 渲染集上下文 |
| [effect_includes.hpp](file:///workspace/src/lib/moo/effect_includes.hpp) | Effect include 处理 |
| [shader_set.hpp](file:///workspace/src/lib/moo/shader_set.hpp) / .ipp | Shader 集合 |
| [shader_manager.hpp](file:///workspace/src/lib/moo/shader_manager.hpp) / .ipp | Shader 管理器 |
| [graphics_settings.hpp](file:///workspace/src/lib/moo/graphics_settings.hpp) / .cpp | 图形设置 |
| [graphics_settings_picker.hpp](file:///workspace/src/lib/moo/graphics_settings_picker.hpp) | 设置选择器 |
| [visual.hpp](file:///workspace/src/lib/moo/visual.hpp) / .cpp / .ipp | 网格资产 |
| [visual_manager.hpp](file:///workspace/src/lib/moo/visual_manager.hpp) / .ipp / .cpp | Visual 管理器 |
| [visual_compound.hpp](file:///workspace/src/lib/moo/visual_compound.hpp) / .cpp | Visual 复合 |
| [visual_splitter.hpp](file:///workspace/src/lib/moo/visual_splitter.hpp) / .cpp | RenderSet 切分 |
| [visual_channels.hpp](file:///workspace/src/lib/moo/visual_channels.hpp) / .cpp | 延迟通道 |
| [visual_common.hpp](file:///workspace/src/lib/moo/visual_common.hpp) | Visual 公共 |
| [node.hpp](file:///workspace/src/lib/moo/node.hpp) / .ipp | 骨骼节点 |
| [node_catalogue.hpp](file:///workspace/src/lib/moo/node_catalogue.hpp) / .ipp | 节点目录 |
| [animation.hpp](file:///workspace/src/lib/moo/animation.hpp) / .ipp / .cpp | 动画 |
| [animation_channel.hpp](file:///workspace/src/lib/moo/animation_channel.hpp) / .ipp | 动画通道 |
| [animation_manager.hpp](file:///workspace/src/lib/moo/animation_manager.hpp) / .ipp | 动画管理器 |
| [streamed_animation_channel.hpp](file:///workspace/src/lib/moo/streamed_animation_channel.hpp) | 流式动画 |
| [software_skinner.hpp](file:///workspace/src/lib/moo/software_skinner.hpp) | CPU 蒙皮 |
| [light_container.hpp](file:///workspace/src/lib/moo/light_container.hpp) / .ipp | 光照桶 |
| [omni_light.hpp](file:///workspace/src/lib/moo/omni_light.hpp) / .ipp | 全向光 |
| [spot_light.hpp](file:///workspace/src/lib/moo/spot_light.hpp) / .ipp | 聚光灯 |
| [directional_light.hpp](file:///workspace/src/lib/moo/directional_light.hpp) / .ipp | 定向光 |
| [fog_helper.hpp](file:///workspace/src/lib/moo/fog_helper.hpp) / .cpp | 雾辅助 |
| [camera.hpp](file:///workspace/src/lib/moo/camera.hpp) | 相机 |
| [camera_planes_setter.hpp](file:///workspace/src/lib/moo/camera_planes_setter.hpp) / .cpp | 近远平面 setter |
| [scissors_setter.hpp](file:///workspace/src/lib/moo/scissors_setter.hpp) / .cpp | 剪刀矩形 setter |
| [viewport_setter.hpp](file:///workspace/src/lib/moo/viewport_setter.hpp) / .cpp | 视口 setter |
| [resource_load_context.hpp](file:///workspace/src/lib/moo/resource_load_context.hpp) / .cpp | 资源加载上下文 |
| [streamed_data_cache.hpp](file:///workspace/src/lib/moo/streamed_data_cache.hpp) / .cpp | 流式数据缓存 |

---

## 13. 总结

moo 库是 BigWorld 客户端渲染的基石，其设计可以归纳为五个层次：

1. **D3D 封装层**（`DX` namespace / `moo_dx.hpp`）：把 `IDirect3DDevice9` 包装为可缓存的 `RenderContext`，可选地再包装为多线程 `DeviceWrapper`。
2. **资源管理层**（`*Manager` + `DeviceCallback`）：纹理、Visual、Effect、Animation 等资源的引用计数、缓存、device lost 恢复。
3. **抽象层**（`BaseTexture` / `Vertices` / `Primitive` / `ManagedEffect` / `LightContainer`）：把 D3D 对象抽象为面向游戏的接口，隐藏硬件细节。
4. **资产层**（`Visual` / `Node` / `Animation` / `Material`）：把抽象资源组合成"可绘制的东西"，处理骨骼、动画、材质、BSP。
5. **调度层**（`VisualChannels` / `VisualHelper` / `RenderContext` 状态机）：在一帧内调度渲染顺序、状态切换、透明物体排序。

关键设计取舍：
- **状态缓存优先于灵活**：所有状态走 `RenderContext::setXxx`，看似啰嗦但带来 10× 性能提升。
- **模板优先于虚函数**：性能敏感路径（顶点处理、morph）用模板特化，只在边界用虚函数。
- **延迟渲染优先于立即**：透明物体走 `VisualChannels` 延迟通道，避免状态抖动。
- **兼容性优先于现代**：支持 D3D9 fixed function（`Material`）+ programmable（`EffectMaterial`）双路径，兼容 2005 年代的硬件。
- **单线程渲染优先于多线程**：默认单线程，`DeviceWrapper` 作为可选多线程路径，降低复杂度。

理解 moo 是理解整个 BigWorld 客户端渲染的基础——`terrain`、`chunk`、`model`、`romp`、`particle` 都建立在它之上。掌握 `RenderContext` 的状态机、`Visual` 的渲染集设计、`ManagedEffect` 的常量自动绑定这三个核心机制，就能在源码中快速定位任何渲染相关问题。

---

> **参见**：
> - 上层模型系统：[13-模型系统-model.md](file:///workspace/study-docs/13-模型系统-model.md)
> - 地形渲染：[05-地形系统-terrain.md](file:///workspace/study-docs/05-地形系统-terrain.md)
> - 粒子与后处理：[14-粒子与后处理-particle-postfx.md](file:///workspace/study-docs/14-粒子与后处理-particle-postfx.md)
> - 资源管理：[02-资源管理-resmgr.md](file:///workspace/study-docs/02-资源管理-resmgr.md)
> - 场景分块：[06-场景分块-chunk.md](file:///workspace/study-docs/06-场景分块-chunk.md)
