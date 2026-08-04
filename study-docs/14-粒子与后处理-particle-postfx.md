# 14. 粒子系统与后处理流水线（particle / post_processing）

> BigWorld Engine 2.0.1 源码学习文档
> 覆盖目录：`src/lib/particle/` 与 `src/lib/post_processing/`
> 关键词：ParticleSystem、MetaParticleSystem、PSA、ParticleSystemRenderer、PostProcessing::Manager、Effect、Phase、FilterQuad

---

## 0. 概览

BigWorld 客户端把“特效类”渲染拆成两个相互独立的库：

| 库 | 目录 | 定位 | 构建基础 | Python 模块名 |
|---|---|---|---|---|
| 粒子系统 | `src/lib/particle/` | 客户端粒子层，构建在 `moo` 之上，提供粒子生成、运动、染色、碰撞、渲染等抽象 | `moo::Visual` / `moo::Material` / `moo::EffectMaterial` | `Pixie` |
| 后处理 | `src/lib/post_processing/` | 渲染后处理流水线，构建在 `moo::RenderTarget` + `moo::EffectMaterial` 之上 | `moo::RenderTarget` / `moo::EffectMaterial` / `moo::MRTSupport` | `PostProcessing` |

二者都通过 `pyscript` 暴露给 Python 脚本：粒子系统挂在 `Pixie` 模块下（见 [pixie.mpp](file:///workspace/src/lib/particle/pixie.mpp)），后处理挂在 `PostProcessing` 模块下（见 [manager.cpp](file:///workspace/src/lib/post_processing/manager.cpp)）。

### 0.1 模块依赖图

```
                         ┌──────────────────────────────┐
                         │           pyscript            │  (Py* 包装层)
                         └──────────┬─────────┬──────────┘
                                    │         │
                ┌───────────────────▼─┐     ┌─▼──────────────────────┐
                │   particle (Pixie)  │     │  post_processing        │
                │  ParticleSystem /   │     │  Manager / Effect /     │
                │  MetaParticleSystem │     │  Phase / FilterQuad     │
                │  PSA / Renderer     │     │                         │
                └──────────┬──────────┘     └────────────┬────────────┘
                           │                             │
                           │   共同依赖                  │
                           ▼                             ▼
        ┌──────────────────────────────────────────────────────┐
        │                        moo                            │
        │   Visual / Material / EffectMaterial / RenderTarget   │
        │   RenderContext / MRTSupport / TextureManager          │
        └──────────────────────────────────────────────────────┘
                           ▲                ▲
                           │                │
        ┌──────────────────┴──────┐  ┌──────┴──────────────────┐
        │        romp             │  │        chunk             │
        │  RompCollider /         │  │  ChunkParticles          │
        │  LensEffectManager /    │  │  (ChunkItem 集成入口)    │
        │  Geometrics / LodSettings│  │                          │
        └─────────────────────────┘  └──────────────────────────┘
```

粒子系统的 `toss/enterWorld/leaveWorld` 通过 `ChunkParticles`（chunk 库）接入分块场景；后处理则由客户端主循环在场景渲染完成后整体调用 `Manager::draw()`。

---

## 第一部分：粒子系统（particle）

## 1. 模块定位

粒子系统是 BigWorld 客户端的“特效层”。它的职责是把一组“可大量复制、生命周期短、需独立物理模拟”的视觉元素（粒子）组织起来：

- 生成（SourcePSA）、销毁（SinkPSA）、受力（ForcePSA）、碰撞（CollidePSA）、染色（TintShaderPSA）等行为由 **PSA（Particle System Action）** 描述；
- 粒子如何画到屏幕上由 **ParticleSystemRenderer** 描述（sprite / mesh / visual / trail / point_sprite / blur / amp）；
- 多个粒子系统组合成一个对象由 **MetaParticleSystem** 完成；
- 整个粒子库的全局状态（风、活跃开关、共享 effect）由 **ParticleSystemManager** 单例管理。

粒子系统在源码中以 [particle_system.hpp](file:///workspace/src/lib/particle/particle_system.hpp) 中的注释明确表达其抽象意图：

```cpp
// particle_system.hpp:32
/**
 *	This class is the abstract picture of the particles that is invariant
 *	to a renderer.
 */
class ParticleSystem : public ReferenceCount
```

即 `ParticleSystem` 本身与渲染器解耦——同样的粒子数据可被不同渲染器绘制。

## 2. 核心抽象类层级

```
ReferenceCount
   │
   ├── ParticleSystem            (粒子系统：粒子容器 + 动作集 + 渲染器)
   │      └── 持有 ParticlesPtr / Actions / ParticleSystemRendererPtr
   │
   ├── MetaParticleSystem        (多个 ParticleSystem 的组合容器)
   │
   ├── ParticleSystemAction      (PSA 基类，所有粒子动作的接口)
   │      ├── SourcePSA          (生成粒子)
   │      ├── SinkPSA            (销毁粒子)
   │      ├── ForcePSA           (恒定加速度)
   │      ├── CollidePSA         (碰撞场景)
   │      ├── TintShaderPSA      (按年龄染色)
   │      ├── StreamPSA          (气流)
   │      ├── JitterPSA          (抖动)
   │      ├── ScalerPSA          (缩放)
   │      ├── OrbitorPSA         (环绕)
   │      ├── SplatPSA           (贴附)
   │      ├── NodeClampPSA       (节点钳制)
   │      ├── MagnetPSA          (磁吸)
   │      ├── BarrierPSA         (屏障)
   │      ├── FlarePSA           (光晕)
   │      └── MatrixSwarmPSA     (矩阵群集)
   │
   ├── ParticleSystemRenderer    (渲染器基类)
   │      ├── SpriteParticleRenderer       (公告板)
   │      ├── VisualParticleRenderer       (每粒子一个 Visual)
   │      ├── MeshParticleRenderer         (网格粒子)
   │      ├── PointSpriteParticleRenderer  (硬件点精灵)
   │      ├── TrailParticleRenderer        (拖尾)
   │      ├── BlurParticleRenderer         (模糊)
   │      └── AmpParticleRenderer          (放大)
   │
   └── Particles (容器基类, AVectorNoDestructor<Particle>)
          ├── ContiguousParticles   (紧凑数组，高效，粒子位置不固定)
          └── FixedIndexParticles   (固定下标，用于 trail/mesh)
```

PyAttachment 体系（duplo 库）把上述 C++ 对象包成 Python 可挂接对象：

```
PyAttachment (duplo)
   ├── PyMetaParticleSystem   →  持有 MetaParticleSystemPtr
   └── PyParticleSystem       →  持有 ParticleSystemPtr
                                  (PyParticleSystem 继承自 PyAttachment，
                                   可挂到模型节点 / 实体上)
```

## 3. 粒子数据结构

### 3.1 `Particle` —— 单个粒子

定义见 [particle.hpp](file:///workspace/src/lib/particle/particle.hpp)。设计上极度紧凑，使用 `#pragma pack(push,1)` 按字节对齐，以节省内存并利于批量上传 GPU。

```cpp
// particle.hpp:44
#pragma pack(push, 1 )
class Particle
{
public:
    struct PositionColour
    {
        Vector3   position_;   ///< Position, units in metres.   [12 bytes]
        uint32    colour_;     ///< Shade/Tint with R,G,B and Alpha. [4 bytes]
    };
    // ...
private:
    struct MeshData {                  // 网格粒子专用
        uint16 meshSpinAge_;
        uint8  spinSpeed_;
        uint8  spinAxisX_, spinAxisY_, spinAxisZ_;
    };
    struct NonMeshData {               // 非网格粒子专用
        uint16 size_;
        uint16 idx_;
        uint16 distance_;
    };
    union ExtraData {
        MeshData    meshData_;
        NonMeshData nonMeshData_;
    };

    PositionColour positionColour_;   // [16 bytes]
    int16   xVel_, yVel_, zVel_;       ///< 256 m/s 量化的速度  [6 bytes]
    uint16  age_;                      ///< 量化后的年龄        [2 bytes]
    uint16  pitchYaw_;                 ///< 旋转                [2 bytes]
    ExtraData extraData_;              ///< idx/distance 或 spin [6 bytes]
};
#pragma pack( pop )
```

关键设计点：

| 字段 | 含义 | 设计取舍 |
|---|---|---|
| `positionColour_` | 位置 + 颜色 | 16 字节对齐，可直接作为顶点流 |
| `xVel_/yVel_/zVel_` | int16 量化速度 | 用 256 m/s 增量压缩，省内存 |
| `age_` | uint16 量化年龄 | `Particle::nAgeIncrements()` 把浮点 dt 量化为离散增量，避免累积误差 |
| `pitchYaw_` | 旋转打包 | 反弹时 256 方向 pitch/yaw |
| `ExtraData` union | 网格/非网格复用 | 同一结构服务两类粒子 |

文件顶部注释解释了网格粒子上限的由来——受顶点着色器常量寄存器限制：

```cpp
// particle.hpp:37
const int PARTICLE_MAX_MESHES = 15;   //limited by number of constants in vshader.
```

### 3.2 `Particles` —— 粒子容器

定义见 [particles.hpp](file:///workspace/src/lib/particle/particles.hpp)。它继承自 `AVectorNoDestructor<Particle>`（避免析构开销的向量），并提供两种策略：

```cpp
// particles.hpp:30
class Particles : public AVectorNoDestructor<Particle>, public ReferenceCount
{
public:
    virtual void addParticle( const Particle &particle, bool isMesh ) = 0;
    virtual iterator removeParticle( iterator particle ) = 0;
    virtual size_t index( iterator particle ) = 0;
    // ...
protected:
    Particles::iterator lastParticleAdded_;
    size_t capacity_;
};
```

| 容器类 | 特点 | 适用场景 |
|---|---|---|
| `ContiguousParticles` | 紧凑数组，删粒子和末尾交换，迭代/插入/删除快，但粒子下标会变 | 大多数 sprite 粒子 |
| `FixedIndexParticles` | 粒子下标固定，可能产生碎片，迭代较慢 | trail、mesh 粒子（需要稳定下标） |

渲染器通过 `createParticleContainer()` 决定使用哪种容器，例如 [visual_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/visual_particle_renderer.hpp)：

```cpp
// visual_particle_renderer.hpp:62
virtual ParticlesPtr createParticleContainer() const
{
    // TODO: Improve performance with Contiguous but need to
    // still be able to calculate particle index (Flex. Part. Formats)
    return new FixedIndexParticles;
}
```

## 4. `ParticleSystem` 详解

声明见 [particle_system.hpp](file:///workspace/src/lib/particle/particle_system.hpp)。它是粒子的“抽象画面”，独立于渲染器。

### 4.1 关键成员

```cpp
// particle_system.hpp:36
class ParticleSystem : public ReferenceCount
{
public:
    typedef std::vector<ParticleSystemActionPtr> Actions;
    // ...
private:
    float    windFactor_;      ///< 风对粒子的影响系数
    bool     enabled_;
    bool     doUpdate_;        ///< 是否在下次 tick 时强制更新
    bool     firstUpdate_;
    ParticlesPtr particles_;   ///< 粒子集合
    Actions  actions_;         ///< 行为集合（PSA）
    RompColliderPtr pGS_;      ///< 地面 specifier
    float    fixedFrameRate_;  ///< 固定帧率模拟（-1 表示用真实 dt）
    float    framesLeftOver_;  ///< 固定帧率剩余时间
    Vector3  windEffect_;      ///< 缓存的本帧风向量
    // ...PyAttachment 仿真字段
    MatrixLiaison * pOwnWorld_;
    bool attached_;
    bool inWorld_;
    ParticleSystemRendererPtr pRenderer_;  ///< 渲染器
    // ...
};
```

注意 `ParticleSystem` 并不直接继承 `PyAttachment`，而是“模拟”了 attachment 接口（`attach/detach/enterWorld/leaveWorld/tick/draw/move`），由外层 `PyParticleSystem` 真正继承 `PyAttachment` 并转发。

### 4.2 容量与上限

```cpp
// particle_system.hpp:30
#define MAX_CAPACITY 65536
```

构造函数把初始容量裁剪到该上限内：

```cpp
// particle_system.cpp:70
ParticleSystem::ParticleSystem( int initialCapacity ) :
    windFactor_( 0.f ), enabled_( true ), /* ... */
    particles_( new ContiguousParticles ),
    counter_( s_counter_++ % 10 ),
    // ...
{
    BW_GUARD;
    this->capacity( min(initialCapacity,MAX_CAPACITY) );
}
```

`counter_` 初值取 `s_counter_++ % 10`，是为了让多个粒子系统的包围盒重算时刻错开（避免同一帧所有系统一起重算 BB，造成帧抖动）。

## 5. PSA（Particle System Action）

### 5.1 PSA 基类与类型 ID

定义见 [particle_system_action.hpp](file:///workspace/src/lib/particle/actions/particle_system_action.hpp)。所有 PSA 共享 `delay` / `minimumAge` / `enabled` 三个属性，核心接口是 `execute`：

```cpp
// particle_system_action.hpp:61
class ParticleSystemAction : public ReferenceCount
{
public:
    virtual void execute( ParticleSystem &particleSystem, float dTime )=0;
    virtual int typeID( void ) const=0;
    virtual std::string nameID( void ) const=0;
    // ...
    static int nameToType(const std::string & name);
    static const std::string & typeToName(int type);
    static ParticleSystemActionPtr createActionOfType(int type);
protected:
    float delay_;       ///< 激活前最小等待时间（秒）
    float minimumAge_;  ///< 受影响粒子的最小年龄（秒）
    float age_;         ///< 已经过时间
    bool  enabled_;
};
```

类型 ID 在同一文件中以宏集中定义，便于序列化与工厂创建：

```cpp
// particle_system_action.hpp:33
#define PSA_SOURCE_TYPE_ID          1
#define PSA_SINK_TYPE_ID             2
#define PSA_BARRIER_TYPE_ID          3
#define PSA_FORCE_TYPE_ID            4
#define PSA_STREAM_TYPE_ID           5
#define PSA_JITTER_TYPE_ID           6
#define PSA_SCALAR_TYPE_ID           7
#define PSA_TINT_SHADER_TYPE_ID      8
#define PSA_NODE_CLAMP_TYPE_ID       9
#define PSA_ORBITOR_TYPE_ID         10
#define PSA_FLARE_TYPE_ID           11
#define PSA_COLLIDE_TYPE_ID         12
#define PSA_MATRIX_SWARM_TYPE_ID    13
#define PSA_MAGNET_TYPE_ID          14
#define PSA_SPLAT_TYPE_ID           15
#define PSA_TYPE_ID_MAX             16
```

### 5.2 各 PSA 一览

| PSA | 类型 ID | 文件 | 职责 |
|---|---|---|---|
| `SourcePSA` | 1 | [source_psa.hpp](file:///workspace/src/lib/particle/actions/source_psa.hpp) | 生成粒子：按速率、按移动、按需；支持周期 active/sleep |
| `SinkPSA` | 2 | [sink_psa.hpp](file:///workspace/src/lib/particle/actions/sink_psa.hpp) | 销毁粒子：超龄、低速、`outsideOnly` 杀掉进入室内的粒子 |
| `BarrierPSA` | 3 | barrier_psa.hpp | 屏障阻挡 |
| `ForcePSA` | 4 | [force_psa.hpp](file:///workspace/src/lib/particle/actions/force_psa.hpp) | 牛顿力（恒定加速度向量） |
| `StreamPSA` | 5 | stream_psa.hpp | 气流（方向 + 噪声） |
| `JitterPSA` | 6 | jitter_psa.hpp | 随机抖动 |
| `ScalerPSA` | 7 | scaler_psa.hpp | 按年龄缩放尺寸 |
| `TintShaderPSA` | 8 | [tint_shader_psa.hpp](file:///workspace/src/lib/particle/actions/tint_shader_psa.hpp) | 按年龄在多关键帧色调间线性插值 |
| `NodeClampPSA` | 9 | node_clamp_psa.hpp | 钳制到节点 |
| `OrbitorPSA` | 10 | orbitor_psa.hpp | 围绕点环绕 |
| `FlarePSA` | 11 | flare_psa.hpp | 生成镜头光晕 ID |
| `CollidePSA` | 12 | [collide_psa.hpp](file:///workspace/src/lib/particle/actions/collide_psa.hpp) | 与碰撞场景碰撞，支持弹性、旋转、声音 |
| `MatrixSwarmPSA` | 13 | matrix_swarm_psa.hpp | 矩阵群集运动 |
| `MagnetPSA` | 14 | magnet_psa.hpp | 磁吸到目标 |
| `SplatPSA` | 15 | splat_psa.hpp | 贴附到碰撞面 |

### 5.3 `SourcePSA` 详解

[source_psa.hpp](file:///workspace/src/lib/particle/actions/source_psa.hpp) 是最复杂的 PSA，支持三种触发方式（可同时启用）：

```cpp
// source_psa.hpp:33
class SourcePSA : public ParticleSystemAction
{
    // ...
private:
    bool motionTriggered_;     ///< 移动触发（按移动距离生成）
    bool timeTriggered_;       ///< 时间触发（按 rate 粒子/秒）
    bool grounded_;            ///< 落到地面
    float rate_;               ///< 时间触发生成速率
    float sensitivity_;        ///< 移动触发灵敏度
    float maxSpeed_;           ///< 时间触发粒子速度上限
    float activePeriod_;       ///< 活跃期长度
    float sleepPeriod_;        ///< 休眠期长度
    float sleepPeriodMax_;     ///< 休眠期最大值（随机）
    float minimumSize_, maximumSize_;
    int   forcedUnitSize_;     ///< 强制生成时的单位大小
    uint64 allowedTime_;       ///< 每帧允许生成时间上限
    VectorGenerator *pPositionSrc_;  ///< 位置生成器
    VectorGenerator *pVelocitySrc_;  ///< 速度生成器
    // ...
};
```

`activePeriod` / `sleepPeriod` / `sleepPeriodMax` 实现周期性喷射（如间歇喷泉）：活跃期生成，休眠期停止，休眠时长在 `[sleepPeriod_, sleepPeriodMax_]` 间随机。

### 5.4 `SinkPSA` 详解

[sink_psa.hpp](file:///workspace/src/lib/particle/actions/sink_psa.hpp) 销毁粒子的条件可叠加：

```cpp
// sink_psa.hpp:26
class SinkPSA : public ParticleSystemAction
{
    // ...
private:
    float maximumAge_;      ///< 最大年龄（秒），-1 表示不限
    float minimumSpeed_;    ///< 最小速度（m/s），-1 表示不限
    bool  outsideOnly_;     ///< 杀掉进入室内的粒子
    void cacheHullInfo(const BoundingBox& wvbb);   ///< outsideOnly 优化
    std::set<class Chunk*> shells_;                ///< 缓存的重叠 chunk 外壳
};
```

`outsideOnly_` 用于户外粒子（雨、雪）：粒子一旦进入室内 chunk 就被销毁。`cacheHullInfo` 缓存外壳信息以避免每帧重新查询。

### 5.5 `ForcePSA` 详解

[force_psa.hpp](file:///workspace/src/lib/particle/actions/force_psa.hpp) 极简，只是一个加速度向量：

```cpp
// force_psa.hpp:35
class ForcePSA : public ParticleSystemAction
{
public:
    ForcePSA( float x = 0.0f, float y = 0.0f, float z = 0.0f );
    ForcePSA( const Vector3 &newVector );
    const Vector3 &vector( void ) const;
    void vector( const Vector3 &newVector );
    void execute( ParticleSystem &particleSystem, float dTime );
    // ...
private:
    Vector3 vector_;   ///< 描述该力的向量（加速度）
};
```

典型用法：`vector_ = (0,-9.8,0)` 模拟重力。

### 5.6 `CollidePSA` 详解

[collide_psa.hpp](file:///workspace/src/lib/particle/actions/collide_psa.hpp) 让粒子与碰撞场景交互：

```cpp
// collide_psa.hpp:23
class CollidePSA : public ParticleSystemAction
{
    // ...
private:
    bool spriteBased_;          ///< 是否按 sprite 处理
    float elasticity_;          ///< 弹性系数
    float minAddedRotation_;    ///< 碰撞附加最小旋转
    float maxAddedRotation_;    ///< 碰撞附加最大旋转
    int   entityID_;
    std::string soundTag_;      ///< 声音标签
    bool  soundEnabled_;
    int   soundSrcIdx_;
    std::string soundProject_, soundGroup_, soundName_;
};
```

碰撞时可触发声音（`soundEnabled_` + `soundProject/Group/Name`），并给粒子附加随机旋转（`minAddedRotation_` ~ `maxAddedRotation_`）。

### 5.7 `TintShaderPSA` 详解

[tint_shader_psa.hpp](file:///workspace/src/lib/particle/actions/tint_shader_psa.hpp) 按年龄染色，使用 `std::map<float, Vector4>` 存关键帧：

```cpp
// tint_shader_psa.hpp:24
class TintShaderPSA : public ParticleSystemAction
{
public:
    typedef std::map<float, Vector4> Tints;
    void addTintAt( float time, const Vector4 &tint );
    void clearAllTints( void );
    // ...
private:
    Tints tints_;              ///< 年龄→色调 映射
    bool  repeat_;             ///< 是否循环
    float period_;             ///< 循环周期
    float fogAmount_;          ///< 雾混合量
    Vector4ProviderPtr modulator_;  ///< 全局调制器
};
```

初始颜色为 `(0.5,0.5,0.5,1.0)`，在相邻关键帧间线性插值；`modulator_`（Vector4Provider）可整体调制，`fogAmount_` 控制与场景雾的混合。

## 6. 渲染器（ParticleSystemRenderer）

### 6.1 基类接口

定义见 [particle_system_renderer.hpp](file:///workspace/src/lib/particle/renderers/particle_system_renderer.hpp)：

```cpp
// particle_system_renderer.hpp:34
class ParticleSystemRenderer : public ReferenceCount
{
public:
    virtual void update( Particles::iterator beg,
        Particles::iterator end, float dTime ) {}
    virtual void draw( const Matrix & worldTransform,
        Particles::iterator beg,
        Particles::iterator end,
        const BoundingBox & bb ) = 0;
    // Callback fn for ParticleSystemDrawItem sorted draw
    virtual void realDraw( const Matrix & worldTransform,
        Particles::iterator beg,
        Particles::iterator end ) = 0;
    bool viewDependent() const;
    bool local() const;
    virtual bool isMeshStyle() const { return false; }
    virtual ParticlesPtr createParticleContainer() const = 0;
    static ParticleSystemRenderer * createRendererOfType(const std::string type, DataSectionPtr ds = NULL);
    // ...
};
```

注意 `draw` 与 `realDraw` 的区分：`draw` 负责把粒子提交到 `Moo::SortedChannel`（按深度/材质排序的渲染通道），`realDraw` 是排序后真正绘制的回调。这一分离使粒子能正确参与场景的透明排序。

### 6.2 各渲染器

| 渲染器 | 文件 | 粒子形态 | 容器 |
|---|---|---|---|
| `SpriteParticleRenderer` | [sprite_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/sprite_particle_renderer.hpp) | 公告板四边形 | `ContiguousParticles` |
| `VisualParticleRenderer` | [visual_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/visual_particle_renderer.hpp) | 每粒子一个 `Moo::Visual` | `FixedIndexParticles` |
| `MeshParticleRenderer` | mesh_particle_renderer.hpp | 网格粒子（受 VS 常量上限约束，最多 15 个） | `FixedIndexParticles` |
| `PointSpriteParticleRenderer` | point_sprite_particle_renderer.hpp | 硬件点精灵 | — |
| `TrailParticleRenderer` | trail_particle_renderer.hpp | 拖尾 | `FixedIndexParticles` |
| `BlurParticleRenderer` | blur_particle_renderer.hpp | 模糊 | — |
| `AmpParticleRenderer` | amp_particle_renderer.hpp | 放大 | — |

### 6.3 `SpriteParticleRenderer` 详解

[sprite_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/sprite_particle_renderer.hpp) 是最常用的渲染器。它定义了 8 种材质效果：

```cpp
// sprite_particle_renderer.hpp:22
class SpriteParticleRenderer : public ParticleSystemRenderer
{
public:
    enum MaterialFX
    {
        FX_ADDITIVE,
        FX_ADDITIVE_ALPHA,
        FX_BLENDED,
        FX_BLENDED_COLOUR,
        FX_BLENDED_INVERSE_COLOUR,
        FX_SOLID,
        FX_SHIMMER,
        FX_SOURCE_ALPHA,
        FX_MAX
    };
    // ...
private:
    MaterialFX materialFX_;
    std::string textureName_;
    bool useFog_;
    Moo::Material material_;
    bool materialSettingsChanged_;
    int   frameCount_;      ///< 纹理动画帧数
    float frameRate_;       ///< 帧率
    Vector3 explicitOrientation_;
    bool rotated_;
    Moo::VertexTDSUV2 sprite_[4];   ///< 公告板四顶点
    ParticleSystemDrawItem sortedDrawItem_;
};
```

`MaterialFX` 决定混合模式（加色、混合、固体、闪烁等）；`frameCount_/frameRate_` 支持序列帧动画纹理；`sprite_[4]` 是公告板模板，绘制时按粒子位置/旋转实例化。

### 6.4 后台纹理加载

`particle_system_renderer.hpp` 内嵌一个模板辅助类 `BGUpdateData`，把纹理加载放到后台线程：

```cpp
// particle_system_renderer.hpp:99
template <typename RenderT>
struct BGUpdateData
{
    static void loadTexture(void * data)
    {
        BGUpdateData * updateData = static_cast<BGUpdateData *>(data);
        updateData->texture_ = Moo::TextureManager::instance()->get(
            updateData->spr_->textureName(), true, true, true, "texture/particle");
    }
    static void updateMaterial(void * data)
    {
        BGUpdateData * updateData = static_cast<BGUpdateData *>(data);
        updateData->spr_->updateMaterial(updateData->texture_);
        delete updateData;
    }
    RenderT * spr_;
    Moo::BaseTexturePtr texture_;
};
```

`SpriteParticleRenderer` 把 `BGUpdateData<SpriteParticleRenderer>` 声明为友元，实现纹理异步加载 + 主线程材质更新。

## 7. `MetaParticleSystem` —— 组合容器

定义见 [meta_particle_system.hpp](file:///workspace/src/lib/particle/meta_particle_system.hpp)。它把多个 `ParticleSystem` 组成一个逻辑整体，对外像单个 attachment：

```cpp
// meta_particle_system.hpp:35
class MetaParticleSystem : public ReferenceCount
{
public:
    typedef std::vector<ParticleSystemPtr> ParticleSystems;
    // ...
    MetaParticleSystemPtr clone() const;
    static void prerequisites( DataSectionPtr pSection, std::set<std::string>& output );
    static bool isParticleSystem( const std::string& file );

    bool tick( float dTime );
    void draw( const Matrix & worldTransform, float lod );
    virtual bool attach( MatrixLiaison * pOwnWorld );
    virtual void detach();
    virtual void enterWorld();
    virtual void leaveWorld();
    // ...
    void addSystem( ParticleSystemPtr newSystem );
    void removeSystem( ParticleSystemPtr system );
    ParticleSystemPtr system( const char * name );
    // ...
private:
    ParticleSystems systems_;
    MatrixLiaison * pOwnWorld_;
    bool attached_;
    bool inWorld_;
#ifdef EDITOR_ENABLED
    MetaData::MetaData metaData_;
#endif
};
```

`clone()` 深拷贝所有子系统但不复制粒子当前状态（见 [meta_particle_system.cpp](file:///workspace/src/lib/particle/meta_particle_system.cpp)）：

```cpp
// meta_particle_system.cpp:107
MetaParticleSystemPtr MetaParticleSystem::clone() const
{
    BW_GUARD;
    MetaParticleSystemPtr result = new MetaParticleSystem();
    for (size_t i = 0; i < systems_.size(); ++i)
    {
        result->addSystem(systems_[i]->clone());
    }
#ifdef EDITOR_ENABLED
    result->metaData() = metaData();
#endif
    return result;
}
```

`addSystem` 会自动把新系统挂到当前 attachment 上（若已 attached）：

```cpp
// meta_particle_system.cpp:372
void MetaParticleSystem::addSystem(ParticleSystemPtr system)
{
    BW_GUARD;
    if (attached_)
    {
        system->detach();
        system->attach(pOwnWorld_);
    }
    systems_.push_back(system);
}
```

## 8. `ParticleSystemManager` —— 单例

定义见 [particle_system_manager.hpp](file:///workspace/src/lib/particle/particle_system_manager.hpp)。它是粒子库的全局单例，同时是 `Moo::DeviceCallback`（设备丢失/恢复时重建资源）：

```cpp
// particle_system_manager.hpp:25
class ParticleSystemManager : public InitSingleton<ParticleSystemManager>,
                              public Moo::DeviceCallback
{
public:
    const std::string & meshSolidEffect() const;
    const std::string & meshAdditiveEffect() const;
    const std::string & meshBlendedEffect() const;
    void createUnmanagedObjects();
    void deleteUnmanagedObjects();
    DX::VertexShader* pPointSpriteVertexShader();
    Moo::VertexDeclaration* pPointSpriteVertexDeclaration();
    const bool active( void ) const { return active_; }
    void active( bool flag ) { active_ = flag; }
    const Vector3 & windVelocity( void ) const { return windVelocity_; }
    void windVelocity( const Vector3 & newWindVelocity ) { windVelocity_ = newWindVelocity; }
    SmartPointer<RompSound> rompSoundDefault();
private:
    Moo::ManagedEffectPtr meshSolidManagedEffect_, meshAdditiveManagedEffect_, meshBlendedManagedEffect_;
    DX::VertexShader* pPointSpriteVertexShader_;
    Moo::VertexDeclaration* pPointSpriteVertexDeclaration_;
    SmartPointer<RompSound> rompSoundDefault_;
    Vector3 windVelocity_;
    bool active_;
};
```

职责：

| 职责 | 说明 |
|---|---|
| 共享 effect | 为 `MeshParticleRenderer` 提供三种 ManagedEffect（solid/additive/blended） |
| 共享 VS/声明 | 为 `PointSpriteParticleRenderer` 提供顶点着色器与顶点声明 |
| 全局风 | `windVelocity_` 由游戏每帧更新，所有粒子通过 `windFactor_` 乘以该向量受力 |
| 全局开关 | `active_` 为 false 时所有粒子停止 tick/draw |
| 默认声音 | `CollidePSA` 的默认 `RompSound` |
| 设备回调 | 设备丢失时删非托管资源，恢复时重建 |

`ParticleSystem::tick` 第一行就检查全局开关：

```cpp
// particle_system.cpp:870
bool ParticleSystem::tick( float dTime )
{
    BW_GUARD;
    if (!ParticleSystemManager::instance().active())
        return false;
    // ...
}
```

## 9. Python 绑定（Pixie 模块）

### 9.1 模块定义

[pixie.mpp](file:///workspace/src/lib/particle/pixie.mpp) 是 Pixie 模块的工厂声明文件：

```cpp
// pixie.mpp:12
// Pixie Factory - documented as a module (used via import Pixie)

/*~ module Pixie
 *	Pixie provides an interface to the BigWorld client's particle
 *	systems. It exposes factory functions that provide instances
 *	of the client's base particle system and instances of modifier classes that
 *	affect the manner in which the particle system evolves over time.
 *
 *	The modifiers that affect the ParticleSystem are all derived from
 *	the ParticleSystemAction abstract base class.
 */
```

### 9.2 `PyParticleSystem`

定义见 [py_particle_system.hpp](file:///workspace/src/lib/particle/py_particle_system.hpp)。它继承 `PyAttachment`（来自 duplo 库），可挂到模型节点或实体上：

```cpp
// py_particle_system.hpp:41
class PyParticleSystem : public PyAttachment
{
    Py_Header( PyParticleSystem, PyAttachment )
public:
    typedef std::vector<ParticleSystemActionPtr> PyActions;
    virtual void tick( float dTime );
    virtual void draw( const Matrix & worldTransform, float lod );
    virtual void localBoundingBox( BoundingBox & bb, bool skinny = false );
    virtual bool attach( MatrixLiaison * pOwnWorld );
    virtual void detach();
    virtual void enterWorld();
    virtual void leaveWorld();
    // ...
    PY_METHOD_DECLARE( py_addAction )
    PY_METHOD_DECLARE( py_removeAction )
    PY_METHOD_DECLARE( py_clear )
    PY_METHOD_DECLARE( py_update )
    PY_METHOD_DECLARE( py_render )
    PY_METHOD_DECLARE( py_load )
    PY_METHOD_DECLARE( py_save )
    PY_METHOD_DECLARE( py_size )
    PY_METHOD_DECLARE( py_force )
    PY_RW_ATTRIBUTE_DECLARE( actionsHolder_, actions )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, fixedFrameRate, fixedFrameRate )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( int, capacity, capacity )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, windFactor, windFactor )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( PyParticleSystemRendererPtr, pRenderer, renderer )
    // ...
private:
    ParticleSystemPtr pSystem_;
    PySTLSequenceHolder<ParticleSystem::Actions> actionsHolder_;
};
```

Python 端用法示例（概念性）：

```python
import Pixie
ps = Pixie.ParticleSystem()
ps.capacity = 200
ps.renderer = Pixie.SpriteRenderer("textures/smoke.dds")
ps.addAction(Pixie.SourcePSA())
ps.addAction(Pixie.ForcePSA())
ps.addAction(Pixie.SinkPSA())
```

### 9.3 `PyMetaParticleSystem`

定义见 [py_meta_particle_system.hpp](file:///workspace/src/lib/particle/py_meta_particle_system.hpp)。除了包装 `MetaParticleSystem`，还提供两种创建方式：

```cpp
// py_meta_particle_system.hpp:32
class PyMetaParticleSystem : public PyAttachment
{
    // ...
    static PyObject* create( const std::string& filename );
    static PyObject* createBG( const std::string& filename, PyObjectPtr callback );
    // ...
};

class MPSBackgroundTask : public BackgroundTask
{
public:
    MPSBackgroundTask( const std::string& filename, PyObject * callback );
    void doBackgroundTask( BgTaskManager & mgr );
    void doMainThreadTask( BgTaskManager & mgr );
private:
    std::string filename_;
    MetaParticleSystemPtr pSystem_;
    SmartPointer<PyObject> pCallback_;
};
```

`createBG` 通过 `MPSBackgroundTask` 在后台线程加载粒子文件，加载完成后在主线程回调 Python callback——这是为了避免大粒子文件加载造成卡顿。

## 10. Chunk 集成：`ChunkParticles`

定义见 [chunk_particles.hpp](file:///workspace/src/lib/particle/chunk_particles.hpp)，实现见 [chunk_particles.cpp](file:///workspace/src/lib/particle/chunk_particles.cpp)。它是 `ChunkItem` + `MatrixLiaison`，把粒子系统作为场景静态物品接入分块系统：

```cpp
// chunk_particles.hpp:24
class ChunkParticles : public ChunkItem, public MatrixLiaison
{
public:
    ChunkParticles();
    ~ChunkParticles();
    bool load( DataSectionPtr pSection );
    virtual void draw();
    virtual void toss( Chunk * pChunk );
    virtual void tick( float dTime );
    virtual const Matrix & getMatrix() const { return worldTransform_; }
    virtual bool setMatrix( const Matrix & m ) { worldTransform_ = m; return true; }
protected:
    std::string resourceID_;
    DataSectionPtr resourceDS_;
    Matrix localTransform_;
    Matrix worldTransform_;
    MetaParticleSystemPtr system_;
    uint32 staggerIdx_;
    float seedTime_;
    bool reflectionVisible_;
private:
    static ChunkItemFactory::Result create( Chunk * pChunk, DataSectionPtr pSection );
    static ChunkItemFactory factory_;
};
```

### 10.1 工厂注册

```cpp
// chunk_particles.cpp:374
ChunkItemFactory ChunkParticles::factory_(
    "particles", 0, ChunkParticles::create );
```

`chunk` 目录在加载 `.chunk` 文件时遇到 `particles` 节点会调用 `ChunkParticles::create`。

### 10.2 加载流程

```cpp
// chunk_particles.cpp:197
bool ChunkParticles::load( DataSectionPtr pSection )
{
    localTransform_ = pSection->readMatrix34( "transform", Matrix::identity );
    resourceID_ = pSection->readString( "resource" );
    if (resourceID_.empty())
        return false;
    // Particles used to not have a path, support that by defaulting to particles/
    if (resourceID_.find( '/' ) == std::string::npos)
        resourceID_ = "particles/" + resourceID_;
    resourceDS_ = BWResource::openSection( resourceID_ );
    if (!resourceDS_)
        return false;
    seedTime_ = resourceDS_->readFloat( "seedTime", seedTime_ );
    reflectionVisible_ = pSection->readBool( "reflectionVisible", reflectionVisible_ );
    system_ = new MetaParticleSystem();
    system_->load( resourceDS_, resourceID_ );
    system_->isStatic(true);
    system_->attach(this);
    return true;
}
```

注意 `system_->isStatic(true)`：场景粒子标记为静态，只有可见或刚加载后才更新（见 §11.2）。

### 10.3 stagger 错峰播种

为避免所有静态粒子同时启动造成帧抖动，`tick` 用 `staggerIdx_` 错峰：

```cpp
// chunk_particles.cpp:77
void ChunkParticles::tick( float dTime )
{
    BW_GUARD_PROFILER( ChunkParticles_tick );
    if ( !system_ ) return;
    if (seedTime_ > 0.f)
    {
        if ((staggerIdx_++ % s_staggerInterval.value()) == 0)
        {
            system_->setDoUpdate();
            seedTime_ -= dTime;
        }
    }
    bool updated = system_->tick( dTime );
    // ...更新 umbra / chunk 可见性包围盒
}
```

`s_staggerInterval` 由配置项 `environment/chunkParticleStagger`（默认 50）控制。

### 10.4 toss 与进出世界

```cpp
// chunk_particles.cpp:273
void ChunkParticles::toss( Chunk * pChunk )
{
    Chunk * oldChunk = pChunk_;
    bool wasNowhere = pChunk_ == NULL;
    this->ChunkItem::toss( pChunk );
    Matrix m = localTransform_;
    if (pChunk_ != NULL)
    {
        if (wasNowhere) {
            m.postMultiply( pChunk_->transform() );
            this->setMatrix(m);
            if ( system_ ) system_->enterWorld();
        } else {
            if ( system_ ) system_->leaveWorld();
            m.postMultiply( pChunk_->transform() );
            this->setMatrix(m);
            if ( system_ ) system_->enterWorld();
        }
    }
    else if (!wasNowhere) {
        if ( system_ ) system_->leaveWorld();
    }
}
```

`toss` 是 chunk 物品在不同 chunk 间迁移的钩子；这里同步把世界矩阵（chunk 变换 × 局部变换）传给粒子系统，并触发进出世界。

## 11. 粒子系统完整生命周期

### 11.1 加载流程（.particle 文件）

`MetaParticleSystem::load` 同时支持旧版单系统格式和新版多系统格式（见 [meta_particle_system.cpp](file:///workspace/src/lib/particle/meta_particle_system.cpp)）：

```cpp
// meta_particle_system.cpp:185
bool MetaParticleSystem::load(DataSectionPtr pDS, const std::string & name)
{
    if (!pDS) return false;
    bool ok = true;
    DiaryEntryPtr de = Diary::instance().add( "particle " + name );
    Moo::ScopedResourceLoadContext resLoadCtx( "particle " + BWResource::getFilename( name ) );
    removeAllSystems();
    if (pDS->openSection( "serialiseVersionData" ))
    {
        // 旧格式：单系统
        ParticleSystemPtr newSystem = new ParticleSystem();
        ok = newSystem->load(pDS);
        newSystem->name(name);
        addSystem( newSystem );
    }
    else
    {
        // 新格式：多系统，每个子节点是一个 ParticleSystem
        for(DataSectionIterator it = pDS->begin(); it != pDS->end(); it++)
        {
            DataSectionPtr pSystemDS = *it;
            std::string systemName = pDS->unsanitise(pSystemDS->sectionName());
            if (systemName != "seedTime" && systemName != METADATA_SECTION_NAME)
            {
                ParticleSystemPtr newSystem = new ParticleSystem;
                ok &= newSystem->serialise(pSystemDS, true, systemName);
                addSystem( newSystem );
            }
        }
    }
    // 若已 attached，重新挂接子系统
    MatrixLiaison* attachment = pOwnWorld_;
    if (attachment) { this->detach(); this->attach( attachment ); }
    de->stop();
    return ok;
}
```

`isParticleSystem` 静态方法通过查找 `seedTime` 或 `serialiseVersionData` 节点判断文件类型：

```cpp
// meta_particle_system.cpp:45
/*static*/ bool MetaParticleSystem::isParticleSystem( const std::string& file )
{
    DataSectionPtr ds = BWResource::openSection( file );
    if ( !ds ) return false;
    for( int i = 0; i < ds->countChildren(); ++i )
    {
        DataSectionPtr child = ds->openChild( i );
        if ( child->sectionName() == "seedTime" ||
            child->findChild( "serialiseVersionData" ) != NULL )
            return true;
        DataSectionPtr firstDS = child->openFirstSection();
        if ( firstDS != NULL &&
            firstDS->findChild( "serialiseVersionData" ) != NULL )
            return true;
    }
    return false;
}
```

### 11.2 tick 流程

`ParticleSystem::tick`（[particle_system.cpp:870](file:///workspace/src/lib/particle/particle_system.cpp)）决定是否更新，并周期性重算包围盒：

```cpp
// particle_system.cpp:870
bool ParticleSystem::tick( float dTime )
{
    if (!ParticleSystemManager::instance().active())
        return false;
    // 静态粒子只在可见或刚加载后更新
    if (!isStatic() || doUpdate_)
    {
        this->update( dTime );
        if (++counter_ == 10) { counter_ = 0; calcBoundingBox = true; }
        if (calcBoundingBox) updateBoundingBox();
        return true;
    }
    return false;
}
```

`update`（[particle_system.cpp:920](file:///workspace/src/lib/particle/particle_system.cpp)）是真正的物理推进，按固定帧率切片：

```cpp
// particle_system.cpp:920
void ParticleSystem::update( float dTime )
{
    BW_GUARD_PROFILER( ParticleSystem_update );
    if (!ParticleSystemManager::instance().active()) return;
    dTime += framesLeftOver_;
    if ( dTime > 1.f ) dTime = 1.f;
    // 预计算风向量
    windEffect_ = Math::clamp(0.0f, windFactor(), 1.0f) *
        ParticleSystemManager::instance().windVelocity();
    while ( dTime > 0.f )
    {
        float dt = ( fixedFrameRate_ > 0.f ) ? ( 1.f / fixedFrameRate_ ) : dTime;
        // 把 dt 量化为粒子年龄增量的整数倍，避免累积误差
        uint16 nAgeIncrements = Particle::nAgeIncrements( dt );
        if (nAgeIncrements == 0) break;
        dt = Particle::age( nAgeIncrements );
        if ( dt > dTime )
            if (almostEqual(dt,dTime)) dt = dTime; else break;
        dTime -= dt;
        // 1) 老化 pass：每个活粒子年龄 +=
        if (particles_) {
            Particles::iterator pIter = particles_->begin();
            while ( pIter != particles_->end() ) {
                if (pIter->isAlive()) {
                    uint16 age = pIter->ageAccurate();
                    if ( (uint32)age + (uint32)nAgeIncrements > (uint32)Particle::ageMax() )
                        pIter->ageAccurate( Particle::ageMax() );
                    else
                        pIter->ageAccurate( age + nAgeIncrements );
                }
                pIter++;
            }
            // 2) 执行每个 PSA
            Actions::iterator aIter = actions_.begin();
            while ( aIter != actions_.end() ) {
                if ( firstUpdate_ ) (*aIter)->setFirstUpdate();
                if ( (*aIter)->enabled() ) (*aIter)->execute( *this, dt );
                aIter++;
            }
            firstUpdate_ = false;
        }
        // 3) 移动 pass：把速度应用到位置
        if ( particles_ ) {
            Particles::iterator pIter = particles_->begin();
            while ( pIter != particles_->end() ) {
                if (pIter->isAlive())
                    this->predictPosition( *pIter, dt, pIter->position() );
                pIter++;
            }
        }
        // 4) 渲染器更新（如 trail 追加顶点）
        if (pRenderer_) pRenderer_->update( begin(), end(), dt );
    }
    framesLeftOver_ = dTime;
    doUpdate_ = false;
}
```

`predictPosition` 把风加到速度上再积分：

```cpp
// particle_system.cpp:1027
void ParticleSystem::predictPosition( const Particle& particle, float dt, Vector3& retPos )
{
    Vector3 oldVelocity;
    particle.getVelocity( oldVelocity );
    Vector3 newVelocity = oldVelocity + windEffect_;
    retPos = particle.position() + dt * newVelocity;
}
```

### 11.3 draw 流程

`ParticleSystem::draw`（[particle_system.cpp:1049](file:///workspace/src/lib/particle/particle_system.cpp)）很薄，主要委托给渲染器：

```cpp
// particle_system.cpp:1049
void ParticleSystem::draw( const Matrix &world, float lod )
{
    BW_GUARD;
    float maxLod = LodSettings::instance().applyLodBias(maxLod_);
    if (!ParticleSystemManager::instance().active() || (maxLod > 0.f && lod > maxLod))
        return;
    if (pRenderer_ != NULL)
    {
        Matrix w( world );
        w.preTranslateBy( localOffset_ );
        pRenderer_->draw( w, this->begin(), this->end(), this->boundingBox() );
    }
    // 可见粒子也标记需要 tick
    doUpdate_ = true;
}
```

注意结尾 `doUpdate_ = true`：可见粒子在下一帧会被 tick（即使标记为 static）。这是静态粒子“可见才更新”的实现机制。

### 11.4 完整时序图

```
ChunkManager          ChunkParticles        MetaParticleSystem      ParticleSystem          PSA[]              Renderer
     │                      │                      │                      │                    │                    │
     │ toss(chunk)          │                      │                      │                    │                    │
     ├─────────────────────>│                      │                      │                    │                    │
     │                      │ enterWorld()         │                      │                    │                    │
     │                      ├─────────────────────>│ enterWorld()         │                    │                    │
     │                      │                      ├─────────────────────>│                    │                    │
     │                      │                      │                      │                    │                    │
     │ tick(dTime)          │                      │                      │                    │                    │
     ├─────────────────────>│ tick(dTime)          │                      │                    │                    │
     │                      ├─────────────────────>│ for each system: tick│                    │                    │
     │                      │                      ├─────────────────────>│ update(dTime)      │                    │
     │                      │                      │                      │  老化 pass         │                    │
     │                      │                      │                      │  for each PSA:     │                    │
     │                      │                      │                      ├───────────────────>│ execute(dt)        │
     │                      │                      │                      │  移动 pass         │                    │
     │                      │                      │                      │  renderer.update() ├───────────────────>│
     │                      │                      │                      │                    │                    │
     │ draw                 │                      │                      │                    │                    │
     ├─────────────────────>│ draw()               │                      │                    │                    │
     │                      ├─────────────────────>│ draw(world,lod)      │                    │                    │
     │                      │                      ├─────────────────────>│ draw(world,lod)    │                    │
     │                      │                      │                      ├───────────────────┐│                    │
     │                      │                      │                      │ LOD 检查           ││ draw(world,particles,bb)
     │                      │                      │                      ├───────────────────┼┼───────────────────>│
     │                      │                      │                      │ doUpdate_=true    ││ 提交到 SortedChannel
     │                      │                      │                      │                    ││ realDraw() 回调
```

---

## 第二部分：后处理流水线（post_processing）

## 12. 模块定位

后处理库实现渲染完成后的屏幕空间特效链：景深、运动模糊、色彩校正、辉光、热扭曲等。它构建在 `moo::RenderTarget` + `moo::EffectMaterial` 之上，通过多 RenderTarget（MRT）和一串 Phase 串联实现。

核心设计是“Effect 链 + Phase + FilterQuad”三层：

- **Manager**：单例，持有一条 Effect 链，提供 `tick/draw`；
- **Effect**：一个后处理效果，由多个 Phase 组成，可整体 bypass；
- **Phase**：单次“用某 material 把某 FilterQuad 画到某 RenderTarget”的操作；
- **FilterQuad**：屏幕空间四边形（可多采样），决定如何采样源纹理。

## 13. 核心类层级

```
InitSingleton<Manager>                (cstdmf)
   └── PostProcessing::Manager        (单例，持 Effect 链)

PyObjectPlus
   ├── PostProcessing::Effect         (Py 对象，持 Phase 列表)
   ├── PostProcessing::Phase          (抽象基类)
   │      └── PyPhase                 (通用 Phase：RenderTarget + Material + FilterQuad)
   ├── PostProcessing::FilterQuad     (抽象基类)
   │      └── PyFilterQuad            (通用 FilterQuad：多样本四边形)
   └── PostProcessing::Debug          (调试可视化)

Singleton<PhaseCreators>              (Phase 工厂注册表)
   └── PhaseFactory                   (按名创建 Phase)
Singleton<FilterQuadCreators>         (FilterQuad 工厂注册表)
   └── FilterQuadFactory              (按名创建 FilterQuad)
```

## 14. `Manager` 详解

声明见 [manager.hpp](file:///workspace/src/lib/post_processing/manager.hpp)，实现见 [manager.cpp](file:///workspace/src/lib/post_processing/manager.cpp)。

```cpp
// manager.hpp:21
namespace PostProcessing
{
    class Manager : public InitSingleton< Manager >
    {
    public:
        void tick( float dTime );
        void draw();
        void debug( DebugPtr d ) { debug_ = d; }
        DebugPtr debug() const   { return debug_; }
        typedef std::vector<EffectPtr> Effects;
        const Effects& effects() const { return effects_; }
        PY_MODULE_STATIC_METHOD_DECLARE( py_chain )
        PY_MODULE_STATIC_METHOD_DECLARE( py_debug )
        PY_MODULE_STATIC_METHOD_DECLARE( py_profile )
        PY_MODULE_STATIC_METHOD_DECLARE( py_save )
        PY_MODULE_STATIC_METHOD_DECLARE( py_load )
    private:
        PyObject* getChain() const;
        PyObject* setChain( PyObject * args );
        Effects effects_;
        DebugPtr debug_;
    };
};
```

### 14.1 tick

```cpp
// manager.cpp:60
void Manager::tick( float dTime )
{
    BW_GUARD_PROFILER( PostProcessing_Tick );
    Effects::iterator it = effects_.begin();
    Effects::iterator end = effects_.end();
    for (; it != end; it++)
    {
        Effect* e = it->getObject();
        e->tick(dTime);
    }
}
```

### 14.2 draw

```cpp
// manager.cpp:76
void Manager::draw()
{
    BW_GUARD_PROFILER( PostProcessing_Draw );
    Moo::MRTSupport::instance().bind();
    Moo::rc().setRenderState(
        D3DRS_COLORWRITEENABLE,
        D3DCOLORWRITEENABLE_RED |
        D3DCOLORWRITEENABLE_GREEN |
        D3DCOLORWRITEENABLE_BLUE |
        D3DCOLORWRITEENABLE_ALPHA );
    if ( debug_.hasObject() )
        debug_->beginChain( effects_.size() );
    Effects::iterator it = effects_.begin();
    Effects::iterator end = effects_.end();
    for (; it != end; it++)
    {
        Effect* e = it->getObject();
        e->draw( debug_.getObject() );
    }
    Moo::MRTSupport::instance().unbind();
}
```

关键点：

- 绘制前绑定 MRT（多渲染目标），允许 Phase 同时输出多个纹理；
- 显式打开 RGBA 四通道写入；
- 若设置了 `Debug`，调用 `beginChain` 通知调试器开始一条新链；
- 顺序执行每个 Effect 的 `draw`。

### 14.3 Python 接口

| 方法 | 说明 |
|---|---|
| `PostProcessing.chain([seq])` | 无参返回当前 Effect 链；有参设置链 |
| `PostProcessing.load(dataSection)` | 从 DataSection 加载链（不自动设为全局链） |
| `PostProcessing.save(dataSection)` | 保存当前链 |
| `PostProcessing.debug([dbg])` | 获取/设置 Debug 对象 |
| `PostProcessing.profile([n])` | 用 D3D TIMESTAMP query 测平均 GPU 耗时 |

`py_profile` 用 D3D query 测 GPU 时间，是性能调优利器（见 [manager.cpp:348](file:///workspace/src/lib/post_processing/manager.cpp)）：

```cpp
// manager.cpp:348
PyObject* Manager::py_profile( PyObject * args )
{
    size_t nSamples = 100;
    // ...
    DX::ScopedWrapperFlags swf(
        DX::getWrapperFlags() & ~DX::WRAPPER_FLAG_QUERY_ISSUE_FLUSH );
    // TIMESTAMPFREQ query
    // ...
    pStartQuery->Issue(D3DISSUE_BEGIN);
    pStartQuery->Issue(D3DISSUE_END);
    for (size_t i=0; i<nSamples; i++)
        Manager::instance().draw();
    pEndQuery->Issue(D3DISSUE_BEGIN);
    pEndQuery->Issue(D3DISSUE_END);
    // 取回 startTime64/endTime64，除以频率得秒
    double timeMSec = (double)(endTime64 - startTime64) / (double)timerFreq64;
    return Script::getData( timeMSec / (double)nSamples );
}
```

## 15. `Effect` 详解

声明见 [effect.hpp](file:///workspace/src/lib/post_processing/effect.hpp)，实现见 [effect.cpp](file:///workspace/src/lib/post_processing/effect.cpp)。

```cpp
// effect.hpp:30
class Effect : public PyObjectPlus
{
    Py_Header( Effect, PyObjectPlus )
public:
    virtual void tick( float dTime );
    virtual void draw( class Debug* );
    virtual bool load( DataSectionPtr );
    virtual bool save( DataSectionPtr );
    typedef std::vector<PhasePtr> Phases;
    const Phases& phases() const { return phases_; }
    PY_RW_ATTRIBUTE_DECLARE( phasesHolder_, phases )
    PY_RW_ATTRIBUTE_DECLARE( name_, name )
    PY_RW_ATTRIBUTE_DECLARE( bypass_, bypass )
private:
    Phases phases_;
    PySTLSequenceHolder<Phases> phasesHolder_;
    std::string name_;
    Vector4ProviderPtr bypass_;   ///< bypass 信号提供器
};
```

### 15.1 bypass 机制

`bypass_` 是一个 `Vector4Provider`，当其 `w` 分量接近 0 时，整个 Effect 跳过绘制（自动优化）：

```cpp
// effect.cpp:121
void Effect::draw( Debug* debug )
{
    if ( debug ) debug->beginEffect( phases_.size() );
    bool bypass = false;
    if ( bypass_.hasObject() )
    {
        Vector4 b;
        bypass_->output(b);
        bypass = almostEqual( b.w, 0.f );
    }
    if (!bypass)
    {
        Phases::iterator it = phases_.begin();
        for(; it!=phases_.end(); it++)
            (*it)->draw(debug);
    }
    else if (debug)
    {
        for (size_t i=0; i < phases_.size(); i++)
            debug->drawDisabledPhase();
    }
}
```

注释给出典型场景：用一个 Vector4 的 alpha 控制转移相位的淡出，当 alpha 归零时整个 Effect 可跳过中间相位，实现自动优化。

### 15.2 load

```cpp
// effect.cpp:159
bool Effect::load( DataSectionPtr pDS )
{
    name_ = pDS->asString(name_);
    DataSectionPtr bSect = pDS->findChild("bypass");
    if (bSect.hasObject())
    {
        Vector4 bp = bSect->asVector4();
        this->bypass_ = Vector4ProviderPtr( new Vector4Basic(bp), true );
    }
    phases_.clear();
    DataSection::iterator it = pDS->begin();
    for(; it!=pDS->end(); it++)
    {
        DataSectionPtr phase = *it;
        if (phase->sectionName() == "bypass") continue;
        PhasePtr p( PhaseFactory::loadItem(phase), true );
        if (p.hasObject()) phases_.push_back(p);
        else ERROR_MSG( "Unknown phase type %s\n", phase->sectionName().c_str() );
    }
    return true;
}
```

`PhaseFactory::loadItem` 根据节点的 sectionName 查注册表创建对应 Phase 子类。

## 16. `Phase` 与工厂

### 16.1 Phase 抽象基类

声明见 [phase.hpp](file:///workspace/src/lib/post_processing/phase.hpp)：

```cpp
// phase.hpp:31
class Phase : public PyObjectPlus
{
    Py_Header( Phase, PyObjectPlus )
public:
    Phase( PyTypePlus *pType = &s_type_ );
    ~Phase();
    virtual void tick( float dTime ) = 0;
    virtual void draw( class Debug*, RECT* = NULL ) = 0;
    virtual bool load( DataSectionPtr ) = 0;
    virtual bool save( DataSectionPtr ) = 0;
#ifdef EDITOR_ENABLED
    typedef void (*PhaseChangeCallback)( bool needsReload );
    virtual void edChangeCallback( PhaseChangeCallback pCallback ) = 0;
    virtual void edEdit( GeneralEditor * editor ) = 0;
#endif
};
```

### 16.2 `PyPhase` —— 通用 Phase

[py_phase.hpp](file:///workspace/src/lib/post_processing/py_phase.hpp) 是默认实现，组合 RenderTarget + Material + FilterQuad：

```cpp
// py_phase.hpp:38
class PyPhase : public Phase
{
    Py_Header( PyPhase, Phase )
    DECLARE_PHASE( PyPhase )
public:
    void tick( float dTime );
    void draw( class Debug*, RECT* = NULL );
    bool load( DataSectionPtr );
    bool save( DataSectionPtr );
    // ...
    PY_RW_ATTRIBUTE_DECLARE( pRenderTarget_, renderTarget )
    PY_RW_ATTRIBUTE_DECLARE( pMaterial_, material )
    PY_RW_ATTRIBUTE_DECLARE( pFilterQuad_, filterQuad )
    PY_RW_ATTRIBUTE_DECLARE( clearRenderTarget_, clearRenderTarget )
    PY_RW_ATTRIBUTE_DECLARE( name_, name )
private:
    bool            clearRenderTarget_;
    FilterQuadPtr   pFilterQuad_;
    PyMaterialPtr   pMaterial_;
    PyRenderTargetPtr pRenderTarget_;
    std::string     name_;
};
```

一个 PyPhase 的语义即：“把 `pFilterQuad_` 用 `pMaterial_` 画到 `pRenderTarget_`，可选先清空”。

### 16.3 Phase 工厂

[phase_factory.hpp](file:///workspace/src/lib/post_processing/phase_factory.hpp) 用宏实现自注册工厂：

```cpp
// phase_factory.hpp:30
#define DECLARE_PHASE( CLASS )                                  \
    static Phase* create( DataSectionPtr pSection );            \
    static PhaseRegistrar s_registrar;

#define IMPLEMENT_PHASE( CLASS, LABEL )                         \
    Phase* CLASS::create( DataSectionPtr pDS )                  \
    {                                                           \
        CLASS * pPhase = new CLASS();                           \
        if (pPhase->load(pDS)) return pPhase;                   \
        delete pPhase;                                          \
        return NULL;                                            \
    };                                                          \
    PhaseRegistrar CLASS::s_registrar( #CLASS, CLASS::create );

typedef Phase* (*PhaseCreator)( DataSectionPtr pSection );
class PhaseCreators : public std::map<std::string,PhaseCreator>,
                      public Singleton<PhaseCreators> {};
class PhaseFactory
{
public:
    static void registerType( const std::string& label, PhaseCreator creator = NULL );
    Phase* create( DataSectionPtr );
    static Phase* loadItem( DataSectionPtr );
};
class PhaseRegistrar
{
public:
    PhaseRegistrar( const std::string& label, PhaseCreator c )
    { PhaseFactory::registerType( label, c ); };
};
```

`PhaseRegistrar` 在全局对象构造期把 `CLASS::create` 注册到 `PhaseCreators` 单例，`PhaseFactory::loadItem` 按名查表创建。这是一种典型的“自注册工厂”模式。

## 17. `FilterQuad` 与工厂

### 17.1 FilterQuad 抽象基类

[filter_quad.hpp](file:///workspace/src/lib/post_processing/filter_quad.hpp)：

```cpp
// filter_quad.hpp:31
class FilterQuad : public PyObjectPlus
{
    Py_Header( FilterQuad, PyObjectPlus )
public:
    virtual void preDraw( Moo::EffectMaterialPtr pMat ) = 0;
    virtual void draw() = 0;
    virtual bool save( DataSectionPtr ) = 0;
    virtual bool load( DataSectionPtr ) = 0;
#ifdef EDITOR_ENABLED
    typedef void (*FilterChangeCallback)( bool needsReload );
    virtual void edEdit( GeneralEditor * editor, FilterChangeCallback pCallback ) = 0;
    virtual const char * creatorName() const = 0;
#endif
};
```

`preDraw` 在材质 draw 前调用，用于设置采样器/常量；`draw` 真正绘制四边形。

### 17.2 `PyFilterQuad`

[py_filter_quad.hpp](file:///workspace/src/lib/post_processing/py_filter_quad.hpp) 是通用实现，支持多样本（4-tap / 8-tap）：

```cpp
// py_filter_quad.hpp:31
class PyFilterQuad : public FilterQuad
{
    Py_Header( PyFilterQuad, FilterQuad )
    DECLARE_FILTER_QUAD( PyFilterQuad )
public:
    typedef std::vector<Vector3> Samples;   // {u offset, v offset, weight}
    void preDraw( Moo::EffectMaterialPtr pMat );
    void draw();
    bool save( DataSectionPtr );
    bool load( DataSectionPtr );
    PY_RW_ATTRIBUTE_DECLARE( samplesHolder_, samples )
private:
    Samples samples_;
    PySTLSequenceHolder<Samples> samplesHolder_;
    void addVert( Moo::VertexXYZNUV2& v, FilterSample* rSample, size_t tapSize, const Vector2& srcDim );
    void buildMesh();
    CustomMesh< Moo::FourTapVertex > verts4tap_;
    CustomMesh< Moo::EightTapVertex > verts8tap_;
    Moo::VertexDeclaration* pDecl4tap_;
    Moo::VertexDeclaration* pDecl8tap_;
};
```

`Samples` 是 `{u偏移, v偏移, 权重}` 的列表，每个样本对应一次源纹理采样。4 个样本用 `FourTapVertex`，8 个用 `EightTapVertex`——用于高斯模糊等多次采样滤镜。

### 17.3 FilterQuad 工厂

[filter_quad_factory.hpp](file:///workspace/src/lib/post_processing/filter_quad_factory.hpp) 与 Phase 工厂同构，提供 `DECLARE_FILTER_QUAD` / `IMPLEMENT_FILTER_QUAD` 宏和 `FilterQuadFactory` / `FilterQuadCreators` 单例。

## 18. `Debug` —— 调试可视化

[debug.hpp](file:///workspace/src/lib/post_processing/debug.hpp)：

```cpp
// debug.hpp:20
class Debug : public PyObjectPlus
{
    Py_Header( Debug, PyObjectPlus )
public:
    bool enabled() const { return enabled_; }
    void enabled( bool e ){ enabled_ = e; }
    virtual void beginChain( uint32 nEffects );
    virtual void beginEffect( uint32 nPhases );
    void drawDisabledPhase();
    void copyPhaseOutput( Moo::RenderTargetPtr pRT );
    virtual RECT nextPhase();
    PyRenderTargetPtr pRT() const;
    virtual Vector4 phaseUV( uint32 effect, uint32 phase, uint32 nEffects, uint32 nPhases );
    PY_RW_ATTRIBUTE_DECLARE( pRT_, renderTarget );
    PY_FACTORY_DECLARE()
private:
    int32 nEffects_, nPhases_, effect_, phase_;
    PyRenderTargetPtr pRT_;
    bool enabled_;
};
```

Debug 把每个 Phase 的输出拷贝到自己的 RenderTarget，并按网格排列，便于在编辑器中查看整条链的中间结果。`beginChain/beginEffect/nextPhase` 维护当前 effect/phase 计数，`phaseUV` 计算每个相位在调试纹理中的 UV 区块。

## 19. 后处理流水线关系

### 19.1 Phase → Effect → Manager 层级

```
┌─────────────────────────────────────────────────────────────┐
│                     Manager (单例)                           │
│   effects_: [Effect_0, Effect_1, ..., Effect_n]             │
│   tick(): for each Effect → tick()                          │
│   draw():  bind MRT → for each Effect → draw(debug) → unbind│
└──────────────────────────┬──────────────────────────────────┘
                           │ 持有
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     Effect (Py 对象)                         │
│   phases_: [Phase_0, Phase_1, ..., Phase_m]                 │
│   bypass_: Vector4Provider  (w≈0 时跳过全部 phase)          │
│   tick(): for each Phase → tick()                           │
│   draw(): 检查 bypass → for each Phase → draw(debug)        │
└──────────────────────────┬──────────────────────────────────┘
                           │ 持有
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Phase (PyPhase 通用实现)                    │
│   pRenderTarget_: 目标 RT                                   │
│   pMaterial_:     EffectMaterial（含 shader）                │
│   pFilterQuad_:   屏幕空间四边形（多样本）                   │
│   clearRenderTarget_: 是否先清空                             │
│   draw(): setRT → clear? → material.draw(filterQuad)        │
└──────────────────────────┬──────────────────────────────────┘
                           │ 持有
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                FilterQuad (PyFilterQuad 通用实现)            │
│   samples_: [{u,v,weight}, ...]                             │
│   draw(): 用 4-tap/8-tap 顶点画全屏四边形                    │
└─────────────────────────────────────────────────────────────┘
```

### 19.2 后处理绘制时序图

```
客户端主循环        Manager              Effect              PyPhase             FilterQuad        Moo::rc
     │                │                     │                   │                    │                │
     │ (场景渲染完成) │                     │                   │                    │                │
     │ tick(dTime)    │                     │                   │                    │                │
     ├───────────────>│ for each Effect     │                   │                    │                │
     │                ├────────────────────>│ for each Phase    │                    │                │
     │                │                     ├──────────────────>│ tick(dTime)        │                │
     │                │                     │                   │                    │                │
     │ draw()         │                     │                   │                    │                │
     ├───────────────>│ MRTSupport.bind()   │                   │                    │                │
     │                │ COLORWRITEENABLE=RGBA                   │                    │                │
     │                │ debug.beginChain(n) │                   │                    │                │
     │                │ for each Effect     │                   │                    │                │
     │                ├────────────────────>│ bypass? 检查      │                    │                │
     │                │                     │ for each Phase    │                    │                │
     │                │                     ├──────────────────>│ setRenderTarget    │                │
     │                │                     │                   │ clearRenderTarget? ├───────────────>│
     │                │                     │                   │ material.draw()   │                │
     │                │                     │                   ├───────────────────>│ preDraw(mat)   │
     │                │                     │                   │                    │ draw()         │
     │                │                     │                   │                    ├───────────────>│
     │                │                     │                   │ debug.nextPhase() │                │
     │                │ MRTSupport.unbind() │                   │                    │                │
```

### 19.3 数据流（典型景深链）

```
   场景颜色 RT ──┐
                 │   ┌────────────┐    ┌────────────┐    ┌────────────┐
   深度/CC RT ───┼──>│ Phase 0    │───>│ Phase 1    │───>│ Phase 2    │───> BackBuffer
                 │   │ 降采样 +   │    │ 高斯模糊 H │    │ 合成       │
   CoC RT ───────┘   │ CoC 计算   │    │            │    │(color*CoC+ │
                     └────────────┘    └────────────┘    │ blurred)   │
                                                          └────────────┘
   Effect (景深) 由 3 个 Phase 串联，每个 Phase 读上游 RT、写下游 RT
```

---

## 20. 粒子与后处理的关系

粒子系统与后处理是**两个独立子系统**，但在客户端渲染管线上有上下游关系：

1. **粒子在场景渲染阶段绘制**：`ChunkParticles::draw`（或 attachment draw）把粒子通过 `Moo::SortedChannel` 提交，参与场景透明排序，写入场景颜色/深度 RT。
2. **后处理在场景渲染完成后执行**：客户端主循环调用 `PostProcessing::Manager::instance().draw()`，读取场景 RT（及深度、CC 等），经 Effect 链处理后写回 BackBuffer。
3. **粒子可被后处理采样**：如辉光（bloom）会采样粒子产生的亮像素；运动模糊会基于粒子速度模糊。
4. **共享 moo 基础设施**：二者都用 `moo::EffectMaterial` / `moo::RenderTarget` / `moo::RenderContext`，但通过不同 Python 模块（`Pixie` vs `PostProcessing`）暴露。
5. **粒子的 `PointSpriteTransferMesh`**：[py_point_sprite_transfer_mesh.hpp](file:///workspace/src/lib/post_processing/py_point_sprite_transfer_mesh.hpp) 这类后处理工具用于把点精灵粒子数据转移到后处理 mesh，是二者的少数直接耦合点。

二者均由客户端主循环驱动，时序上：

```
帧开始
  │
  ├── Camera/RenderContext 设置
  ├── EnviroMinder.tick/draw（天空、雾、光照）
  ├── ChunkManager.draw（场景，含 ChunkParticles::draw → 粒子绘制）
  ├── 实体/模型绘制（含 attachment 粒子）
  │
  ├── PostProcessing::Manager.tick(dTime)     ← 后处理 tick
  ├── PostProcessing::Manager.draw()          ← 后处理 draw（读场景 RT，写 BackBuffer）
  │
  └── Present
```

---

## 21. 文件清单与阅读建议

### 21.1 粒子库核心文件（已阅读）

| 文件 | 角色 |
|---|---|
| [particle.hpp](file:///workspace/src/lib/particle/particle.hpp) | 单粒子结构（紧凑打包） |
| [particles.hpp](file:///workspace/src/lib/particle/particles.hpp) | 粒子容器基类 + Contiguous/FixedIndex |
| [particle_system.hpp](file:///workspace/src/lib/particle/particle_system.hpp) | 粒子系统主类 |
| [particle_system.cpp](file:///workspace/src/lib/particle/particle_system.cpp) | tick/update/draw 实现 |
| [meta_particle_system.hpp](file:///workspace/src/lib/particle/meta_particle_system.hpp) | 多系统组合容器 |
| [meta_particle_system.cpp](file:///workspace/src/lib/particle/meta_particle_system.cpp) | load/clone/prerequisites |
| [particle_system_manager.hpp](file:///workspace/src/lib/particle/particle_system_manager.hpp) | 全局单例（风、effect、设备回调） |
| [actions/particle_system_action.hpp](file:///workspace/src/lib/particle/actions/particle_system_action.hpp) | PSA 基类 + 类型 ID |
| [actions/source_psa.hpp](file:///workspace/src/lib/particle/actions/source_psa.hpp) | 生成粒子 |
| [actions/sink_psa.hpp](file:///workspace/src/lib/particle/actions/sink_psa.hpp) | 销毁粒子 |
| [actions/force_psa.hpp](file:///workspace/src/lib/particle/actions/force_psa.hpp) | 恒定力 |
| [actions/collide_psa.hpp](file:///workspace/src/lib/particle/actions/collide_psa.hpp) | 碰撞 |
| [actions/tint_shader_psa.hpp](file:///workspace/src/lib/particle/actions/tint_shader_psa.hpp) | 按年龄染色 |
| [renderers/particle_system_renderer.hpp](file:///workspace/src/lib/particle/renderers/particle_system_renderer.hpp) | 渲染器基类 + BGUpdateData |
| [renderers/sprite_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/sprite_particle_renderer.hpp) | 公告板渲染器 |
| [renderers/visual_particle_renderer.hpp](file:///workspace/src/lib/particle/renderers/visual_particle_renderer.hpp) | Visual 粒子渲染器 |
| [chunk_particles.hpp](file:///workspace/src/lib/particle/chunk_particles.hpp) | Chunk 集成声明 |
| [chunk_particles.cpp](file:///workspace/src/lib/particle/chunk_particles.cpp) | Chunk 集成实现 |
| [py_particle_system.hpp](file:///workspace/src/lib/particle/py_particle_system.hpp) | Python 包装（PyAttachment） |
| [py_meta_particle_system.hpp](file:///workspace/src/lib/particle/py_meta_particle_system.hpp) | Python 包装 + 后台加载 |
| [pixie.mpp](file:///workspace/src/lib/particle/pixie.mpp) | Pixie 模块声明 |

### 21.2 后处理库核心文件（已阅读）

| 文件 | 角色 |
|---|---|
| [manager.hpp](file:///workspace/src/lib/post_processing/manager.hpp) | 单例声明 |
| [manager.cpp](file:///workspace/src/lib/post_processing/manager.cpp) | tick/draw/Python 接口/profile |
| [effect.hpp](file:///workspace/src/lib/post_processing/effect.hpp) | Effect 声明 |
| [effect.cpp](file:///workspace/src/lib/post_processing/effect.cpp) | tick/draw/load/bypass |
| [phase.hpp](file:///workspace/src/lib/post_processing/phase.hpp) | Phase 抽象基类 |
| [phase.cpp](file:///workspace/src/lib/post_processing/phase.cpp) | Phase 基类实现 |
| [phase_factory.hpp](file:///workspace/src/lib/post_processing/phase_factory.hpp) | Phase 自注册工厂 |
| [py_phase.hpp](file:///workspace/src/lib/post_processing/py_phase.hpp) | 通用 Phase（RT+Material+FilterQuad） |
| [filter_quad.hpp](file:///workspace/src/lib/post_processing/filter_quad.hpp) | FilterQuad 抽象基类 |
| [filter_quad_factory.hpp](file:///workspace/src/lib/post_processing/filter_quad_factory.hpp) | FilterQuad 自注册工厂 |
| [py_filter_quad.hpp](file:///workspace/src/lib/post_processing/py_filter_quad.hpp) | 通用 FilterQuad（4/8-tap） |
| [debug.hpp](file:///workspace/src/lib/post_processing/debug.hpp) | 调试可视化 |

### 21.3 阅读建议路径

1. **粒子**：`particle.hpp` → `particles.hpp` → `particle_system.hpp` → `particle_system.cpp`（tick/update/draw）→ `particle_system_action.hpp` → 各 PSA → `particle_system_renderer.hpp` → 各 renderer → `meta_particle_system.hpp` → `chunk_particles.hpp/cpp` → `py_*` 绑定。
2. **后处理**：`manager.hpp/cpp` → `effect.hpp/cpp` → `phase.hpp` + `phase_factory.hpp` → `py_phase.hpp` → `filter_quad.hpp` + `filter_quad_factory.hpp` → `py_filter_quad.hpp` → `debug.hpp`。

---

## 22. 设计要点总结

| 维度 | 粒子系统 | 后处理 |
|---|---|---|
| 抽象核心 | 数据（Particle）+ 行为（PSA）+ 视觉（Renderer）三解耦 | Effect/Phase/FilterQuad 三层 + 工厂自注册 |
| 性能策略 | 紧凑打包粒子、量化年龄/速度、固定帧率切片、错峰 BB 重算、静态粒子可见才更新、后台纹理加载 | MRT、bypass 自动跳过、profile GPU 时间、4/8-tap 复用顶点格式 |
| 扩展机制 | 新增 PSA/Renderer 只需继承基类 + 注册 typeID | 新增 Phase/FilterQuad 用 `DECLARE_/IMPLEMENT_` 宏自注册 |
| Python 暴露 | `Pixie` 模块，工厂函数创建 PSA/Renderer/System | `PostProcessing` 模块，`chain/load/save/debug/profile` 静态方法 |
| 场景集成 | `ChunkParticles` 作为 ChunkItem 接入分块 | 主循环在场景渲染后整体调用 `Manager::draw` |
| 全局状态 | `ParticleSystemManager`（风、active、共享 effect） | `Manager` 单例（Effect 链、Debug） |

二者共同体现了 BigWorld 客户端“数据/行为/视觉分离 + 工厂自注册 + Python 脚本驱动 + moo 基础设施复用”的设计哲学。
