# BigWorld Engine 2.0.1 地形系统（terrain）

> 源码目录：`/workspace/src/lib/terrain/`
> 模块定位：BigWorld 客户端 + 编辑器 + 服务端**共享**的"大世界地形"渲染与查询库。一个 terrain block 对应 chunk 网格中的 100m×100m 区域（一个 outdoor chunk），提供可流式加载的**高度图 / 洞图 / 法线图 / 阴影图 / 主导纹理图 / 多层纹理混合 / LOD 纹理**，并支持基于四叉树 cell 的几何碰撞、基于顶点 LOD 的几何变形（geomorphing）、基于相机距离的纹理 LOD 混合。
> 模块边界：上层通过 `Terrain::Manager` 单例驱动；下层 GPU 资源全部走 [moo](file:///workspace/src/lib/moo/)，纹理 / visual 通过 [resmgr](file:///workspace/src/lib/resmgr/) 的 `BWResource` 加载，背景任务走 `cstdmf` 的 `BgTaskManager`。chunk 模块用 `ChunkTerrain` 把 `BaseTerrainBlock` 嵌入 [chunk](file:///workspace/src/lib/chunk/)。
> 编号：05（接 `04-场景分块-chunk.md`，与 `06-网络层-Mercury.md` 并列；地形与 `12-渲染核心-moo.md` 关联紧密）。

---

## 1. 模块定位

terrain 库解决七件事：

1. **多版本共存**：同时维护 v1（旧 terrain1，固定 vertex buffer，无 LOD）和 v2（新 terrain2，含顶点 LOD、纹理 LOD、流式加载、geomorphing）。两套实现共用同一组抽象基类。
2. **大世界分块**：单个地形块大小固定 `BLOCK_SIZE_METRES = 100.0f`，与一个 outdoor chunk 一一对应（见 [terrain_data.hpp](file:///workspace/src/lib/terrain/terrain_data.hpp)）。
3. **可流式资源**：高度图、顶点 LOD、纹理 blends、法线图都按相机距离**按需加载 / 卸载**，由 `Resource<>` 模板 + `BgTaskManager` 异步完成。
4. **多 LOD 渲染**：远距离使用单 pass LOD 纹理 + 低分辨率法线，近距离使用完整 blends 多 pass 渲染；顶点 LOD 通过 GeomipMapping 在不同 vertex grid 之间做形态变换（morph）。
5. **几何碰撞**：用 `TerrainQuadTreeCell` 把高度图划分成四叉树，对线段 / 棱柱（prism）做裁剪查询，仅检测相交的网格三角形。
6. **主导纹理图**：把多层混合纹理压缩成一张 "dominant texture map"（按材质索引存储），用于足音、粒子表面、AI 寻路等只需知道"当前是什么材质"的逻辑。
7. **服务端裁剪**：通过 `MF_SERVER` 编译宏，服务端只保留碰撞、高度查询、材质查询等接口，去掉所有 D3D / Effect 相关代码——服务端 `BaseTerrainBlock` 直接继承 `SafeReferenceCount`，客户端则继承 `BaseRenderTerrainBlock`。

### 1.1 模块依赖关系图

```
                     ┌──────────────────────────────┐
                     │         上层调用方           │
   cellapp (server)  │   客户端 App / Editor         │
        ▲            │   (camera, tick, draw)       │
        │            └───────────────┬──────────────┘
        │                            │
        │                            ▼
   ┌────┴────────────────────────────────────────────────┐
   │                ChunkTerrain (chunk 模块)            │
   │             持有 BaseTerrainBlockPtr                 │
   └──────────────────────┬──────────────────────────────┘
                          │
                          ▼
        ┌─────────────────────────────────────────────────┐
        │              Terrain::Manager (singleton)        │
        │   currentRenderer_ / lodController_              │
        │   graphicsOptions_ / pTerrain2Defaults_          │
        └───────┬───────────────────────┬──────────────────┘
                │                       │
                ▼                       ▼
   ┌────────────────────┐    ┌───────────────────────────┐
   │ BaseTerrainRenderer│    │  BaseTerrainBlock         │
   │ (v1 / v2 / Null)   │    │  loadBlock 工厂方法        │
   └─────────┬──────────┘    └────────┬──────────────────┘
             │                        │
             ▼                        ▼
   ┌──────────────────┐      ┌────────────────────────────┐
   │ TerrainRenderer1│      │ TerrainBlock1 / 2          │
   │ TerrainRenderer2│      │ CommonTerrainBlock1 / 2    │
   │  addBlock/       │      │ EditorTerrainBlock1 / 2    │
   │  drawAll/        │      │  + HeightMap/HoleMap/...   │
   │  drawSingle      │      └────────────────────────────┘
   └──────────────────┘
             │
             ▼
        ┌──────────────────────────────────────────┐
        │  Moo::RenderContext / EffectMaterial /    │
        │  VertexBuffer / IndexBuffer / Texture    │
        └──────────────────────────────────────────┘
                  │
                  ▼
              D3D9 device
```

### 1.2 编译宏与版本切换

terrain 库通过四个宏切换行为：

| 宏 | 含义 |
|----|------|
| `MF_SERVER` | 编译为服务端版本，剔除所有 D3D / Effect 相关代码 |
| `EDITOR_ENABLED` | 编译为编辑器版本，启用 `EditorBaseTerrainBlock` / `EditorVertexLodManager` / `TTL2Cache` |
| `TERRAINBLOCK1` | 一个宏别名：编辑器态展开为 `EditorTerrainBlock1`，服务端态展开为 `CommonTerrainBlock1`，客户端态展开为 `TerrainBlock1`（见 [editor_terrain_block1.hpp](file:///workspace/src/lib/terrain/terrain1/editor_terrain_block1.hpp):17–82） |
| `TERRAINBLOCK2` | 类似上面，用于 v2（在 `editor_terrain_block2.hpp` 中定义） |

服务端与客户端的 `BaseTerrainBlock` 基类不同，靠 `#ifdef MF_SERVER` 在 [base_terrain_block.hpp](file:///workspace/src/lib/terrain/base_terrain_block.hpp):36–40 切换：

```cpp
// base_terrain_block.hpp:36
#ifdef MF_SERVER
    typedef SafeReferenceCount         BaseTerrainBlockBase;
#else
    typedef BaseRenderTerrainBlock      BaseTerrainBlockBase;
#endif
```

### 1.3 版本探测与工厂方法

`BaseTerrainBlock::loadBlock` 是地形块的统一入口，根据 resource 路径末尾是 `/terrain` 还是 `/terrain2`，以及实际是否存在子 section，决定走 v1 还是 v2：

```cpp
// base_terrain_block.cpp:32
uint32 BaseTerrainBlock::terrainVersion( std::string& resource )
{
    if ( resource.substr( resource.length() - 8 ) == "/terrain" &&
        BWResource::openSection( resource + '2' ) != NULL )
    {
        resource += '2';                 // 自动升级到 v2
        return 200;
    }
    else if ( /* ... */ BWResource::openSection( resource ) != NULL )
        return 100;
    else if ( /* /terrain2 */ BWResource::openSection( resource ) != NULL )
        return 200;
    return 0;
}
```

返回 100 → new `TERRAINBLOCK1()`；返回 200 → new `TERRAINBLOCK2(pSettings)`。`pSettings` 是 `TerrainSettingsPtr`，从 `space.settings` 中读取。

---

## 2. 公共基类与接口

### 2.1 BaseTerrainBlock

[base_terrain_block.hpp](file:///workspace/src/lib/terrain/base_terrain_block.hpp) 是所有地形块的抽象基类，定义了以下纯虚接口：

| 接口 | 语义 |
|------|------|
| `load(filename, worldTransform, cameraPosition, error)` | 从 `.cdata` 加载地形 |
| `heightMap() / holeMap() / dominantTextureMap()` | 取子图（高度 / 洞 / 主导纹理） |
| `boundingBox()` | 块本地包围盒 |
| `collide(start, end, callback)` | 线段 / 棱柱碰撞 |
| `heightAt(x, z) / normalAt(x, z)` | 取高度 / 法线（已考虑洞） |
| `dataSectionName()` | 返回 `"terrain"` 或 `"terrain2"` |
| `doingBackgroundTask()` | 是否正在后台加载 / 卸载 |

它还提供两个静态工具：

```cpp
// base_terrain_block.hpp:197
static TerrainFinder::Details findOutsideBlock(Vector3 const &pos);
static float getHeight(float x, float z);            // 全局取高度，无地形返回 NO_TERRAIN
static void setTerrainFinder(TerrainFinder & terrainFinder);
```

`TerrainFinder` 由 chunk 模块实现（`ChunkManager` 注入），把世界坐标翻译成 `(pBlock, pMatrix, pInvMatrix)` 三元组（见 [terrain_finder.hpp](file:///workspace/src/lib/terrain/terrain_finder.hpp):28–46）。

`NO_TERRAIN = -1000000.f` 是"该位置无地形"哨兵值，定义在 [base_terrain_block.cpp](file:///workspace/src/lib/terrain/base_terrain_block.cpp):27。

### 2.2 BaseRenderTerrainBlock

[base_render_terrain_block.hpp](file:///workspace/src/lib/terrain/base_render_terrain_block.hpp) 是**仅客户端**的渲染层基类（被 `#ifndef MF_SERVER` 包裹），在 `BaseTerrainBlock` 之上加了三个纯虚：

```cpp
// base_render_terrain_block.hpp:39
class BaseRenderTerrainBlock : public SafeReferenceCount
{
public:
    virtual bool draw( Moo::EffectMaterialPtr pMaterial ) = 0;
    virtual HorizonShadowMap &shadowMap() = 0;            // 地平线阴影图
    virtual void createUMBRAMesh( UMBRAMesh& umbraMesh ) const = 0;   // Umbra 遮挡几何
    virtual void cacheCurrentLighting( bool cacheSpecular = true ) = 0;
    // ...
};
```

`UMBRAMesh` 内部分 `testVertices_` / `writeVertices_` 两组顶点，分别用于 Umbra 可见性测试和写入遮挡 buffer（参考 [Umbra 文档](file:///workspace/doc/api_cpp/client/classChunkTerrain-members.html)）。

### 2.3 EditorBaseTerrainBlock

[editor_base_terrain_block.hpp](file:///workspace/src/lib/terrain/editor_base_terrain_block.hpp) 在编辑器构建中加入保存 / 修改纹理层 / 重建法线图等接口：

| 接口 | 语义 |
|------|------|
| `save(dataSection) / saveToCData / save(filename)` | 三种保存路径 |
| `numberTextureLayers() / maxNumberTextureLayers()` | 当前/最大纹理层数 |
| `insertTextureLayer / removeTextureLayer / textureLayer(idx)` | 增删改纹理层 |
| `drawIgnoringHoles(setTextures)` | 编辑器中忽略洞绘制 |
| `setHeightMapDirty() / rebuildNormalMap(quality) / rebuildLodTexture(transform)` | 高度图变化后的连锁刷新 |
| `rebuildCombinedLayers(compress, genDominantTextureMap)` | 重建合并后的纹理层 |
| `findLayer / findWeakestLayer / optimiseLayers` | 层管理工具 |

常量：

```cpp
// editor_base_terrain_block.hpp:36
static const uint32 BLENDS_MIN_VALUE_THRESHOLD = 8;       // 小于此阈值的 blend 视为空
static const size_t INVALID_LAYER             = (size_t)-1;
```

### 2.4 BaseTerrainRenderer

[base_terrain_renderer.hpp](file:///workspace/src/lib/terrain/base_terrain_renderer.hpp) 是渲染器单例的抽象基类，由 `TerrainRenderer1` / `TerrainRenderer2` / `NullTerrainRenderer`（在 [manager.cpp](file:///workspace/src/lib/terrain/manager.cpp):32 匿名命名空间里）实现：

```cpp
// base_terrain_renderer.hpp:62
virtual void addBlock( BaseRenderTerrainBlock* pBlock, const Matrix& transform ) = 0;
virtual void drawAll( Moo::EffectMaterialPtr pOverride = NULL, bool clearList = true ) = 0;
virtual void drawSingle( BaseRenderTerrainBlock* pBlock, const Matrix& transform,
                         Moo::EffectMaterialPtr altMat = NULL,
                         bool useCachedLighting = false ) = 0;
virtual void clearBlocks() = 0;
virtual bool canSeeTerrain() const = 0;
virtual uint32 version() = 0;       // 100 或 200
```

它通过静态 `instance()` / `instance(newInstance)` 切换当前实现：

```cpp
// base_terrain_renderer.cpp（实现位置）：
static BaseTerrainRenderer* BaseTerrainRenderer::instance()
{
    if (current_ == NULL)
        return &dummy_;     // 啥都不做的 NullTerrainRenderer，避免空指针
    return current_;
}
```

`enableSpecular / enableHoleMap / isVisible` 三个开关由编辑器的 `ChunkPhotographer` 切换，决定材质 specular 和 hole map 是否生效。

---

## 3. 子图接口（TerrainMap 体系）

terrain 的所有"图"——高度图、洞图、阴影图、纹理层 blend、主导纹理图——都共享一个 `TerrainMap<T>` 模板基类，提供 lock/unlock/image/iterator 的统一接口。

### 3.1 TerrainMap<T> 模板

[terrain_map.hpp](file:///workspace/src/lib/terrain/terrain_map.hpp)：

```cpp
// terrain_map.hpp:31
template<typename TYPE>
class TerrainMap : public SafeReferenceCount
{
public:
    typedef Moo::Image<TYPE>                       ImageType;
    typedef TYPE                                   PixelType;
    typedef TerrainMapIter< TerrainMap<TYPE> >     Iterator;

    virtual uint32 width() const = 0;
    virtual uint32 height() const = 0;

#ifdef EDITOR_ENABLED
    virtual bool lock( bool readOnly ) = 0;        // 加锁以写入
    virtual ImageType &image() = 0;                 // 取可写图像（仅在 lock 后）
    virtual bool unlock() = 0;                      // 解锁，可能触发 D3D 重建
    virtual bool save( DataSectionPtr ) const = 0;
#endif
    virtual ImageType const &image() const = 0;     // 只读图像（不需要 lock）
    virtual Iterator iterator(int x, int y);
    virtual bool load(DataSectionPtr dataSection, std::string *error = NULL) = 0;
};
```

底层图像用 `Moo::Image<TYPE>` 存储，TYPE 即像素类型：

| Map | PixelType | 文件 |
|-----|-----------|------|
| TerrainHeightMap | `float` | [terrain_height_map.hpp](file:///workspace/src/lib/terrain/terrain_height_map.hpp) |
| TerrainHoleMap | `bool` | [terrain_hole_map.hpp](file:///workspace/src/lib/terrain/terrain_hole_map.hpp) |
| HorizonShadowMap | `HorizonShadowPixel{uint16 east, west}` | [horizon_shadow_map.hpp](file:///workspace/src/lib/terrain/horizon_shadow_map.hpp) |
| TerrainTextureLayer | `uint8`（blend mask） | [terrain_texture_layer.hpp](file:///workspace/src/lib/terrain/terrain_texture_layer.hpp) |
| DominantTextureMap | `MaterialIndex = uint8` | [dominant_texture_map.hpp](file:///workspace/src/lib/terrain/dominant_texture_map.hpp) |

### 3.2 TerrainMapIter 与 TerrainMapHolder

[terrain_map_iter.hpp](file:///workspace/src/lib/terrain/terrain_map_iter.hpp) 是一个轻量的二维迭代器，包装 `(x, y, dx, dy)`，提供 `get/set/move`：

```cpp
// terrain_map_iter.hpp:154
template<typename TERRAINMAP>
inline typename TerrainMapIter<TERRAINMAP>::PixelType
Terrain::TerrainMapIter<TERRAINMAP>::get() const
{
    return terrainMap_->image().get(x_ + dx_, y_ + dy_);
}
```

[terrain_map_holder.hpp](file:///workspace/src/lib/terrain/terrain_map_holder.hpp) 是 RAII 包装器：构造时 `lock`，析构时 `unlock`，避免忘了解锁：

```cpp
// terrain_map_holder.hpp:46
template<typename TERRAINMAP>
inline Terrain::TerrainMapHolder<TERRAINMAP>::TerrainMapHolder(
    TERRAINMAP *terrainMap, bool readOnly)
    : terrainMap_(terrainMap), readOnly_(readOnly)
{
    if (terrainMap_ != NULL) terrainMap_->lock(readOnly);
}
```

### 3.3 TerrainHeightMap 接口

[terrain_height_map.hpp](file:///workspace/src/lib/terrain/terrain_height_map.hpp) 在 `TerrainMap<float>` 上扩展了高度图专属接口：

```cpp
// terrain_height_map.hpp:29
class TerrainHeightMap : public TerrainMap<float>
{
public:
    virtual float spacingX() const = 0;       // X 方向采样间距（米）
    virtual float spacingZ() const = 0;
    virtual uint32 blocksWidth() const = 0;    // 可见区域宽度（块数）
    virtual uint32 verticesWidth() const = 0;  // 可见区域顶点数
    virtual uint32 polesWidth() const = 0;     // 含不可见边界的总顶点数
    virtual uint32 xVisibleOffset() const = 0; // 可见区域 X 偏移
    virtual float minHeight() const = 0;
    virtual float maxHeight() const = 0;
    virtual float heightAt(int x, int z) const = 0;
    virtual float heightAt(float x, float z) const = 0;    // 浮点采样（双线性插值）
    virtual Vector3 normalAt(int x, int z) const = 0;
    virtual Vector3 normalAt(float x, float z) const = 0;
    float slopeAt(int x, int z) const;         // 坡度（度）
    // ...
};
```

`DEFAULT_UNIT_SCALE = 0.01f` 是默认高度精度（1cm），用于把浮点高度转 `uint16` 存储：

```cpp
// terrain_height_map.hpp:222
inline float TerrainHeightMap::convertHeight2Float( int32 height ) const
{ return float(height) * unitScale_; }

inline int32 TerrainHeightMap::convertFloat2Height( float height ) const
{ return int32( floorf( ( height + 0.5f * unitScale_ ) / unitScale_ ) ); }
```

### 3.4 TerrainHoleMap / HorizonShadowMap / TerrainTextureLayer

- **TerrainHoleMap**：`TerrainMap<bool>`，`true` 表示该格有洞。提供 `noHoles() / allHoles() / holeAt(x,z) / holeAt(xs,zs,xe,ze)`（见 [terrain_hole_map.hpp](file:///workspace/src/lib/terrain/terrain_hole_map.hpp)）。
- **HorizonShadowMap**：`TerrainMap<HorizonShadowPixel>`，每像素两个 `uint16`，分别记录东、西方向的地平线角（用于地形自阴影）。`shadowAt(x,z)` 返回该位置的地平线值（见 [horizon_shadow_map.hpp](file:///workspace/src/lib/terrain/horizon_shadow_map.hpp)）。
- **TerrainTextureLayer**：`TerrainMap<uint8>`，每像素是该层 blend mask 的不透明度（0~255）。每层关联一个 detail texture 和 u/v 投影向量：

```cpp
// terrain_texture_layer.hpp:37
class TerrainTextureLayer : public TerrainMap<uint8>
{
public:
    virtual std::string textureName() const = 0;
    virtual bool textureName(std::string const &filename) = 0;
    virtual Moo::BaseTexturePtr texture() const = 0;
    virtual bool hasUVProjections() const = 0;
    virtual Vector4 const &uProjection() const = 0;
    virtual Vector4 const &vProjection() const = 0;
    // 静态工具：把最多 4 层 blend 成一张 D3D 纹理
    static ComObjectWrap<DX::Texture> createBlendTexture(...);
    static void defaultUVProjections(Vector4 &u, Vector4 &v);
};
```

### 3.5 DominantTextureMap

[dominant_texture_map.hpp](file:///workspace/src/lib/terrain/dominant_texture_map.hpp) 把多 blend 层压缩成一张 `Image<MaterialIndex>`，每像素存储"主导"层的索引：

```cpp
// dominant_texture_map.hpp:51
class DominantTextureMap : public TerrainMap<MaterialIndex>
{
public:
    DominantTextureMap(TextureLayers& layerData, float sizeMultiplier = 0.5f);
    uint32 materialKind( float x, float z ) const;       // 取 material_kinds（用于足音等）
    const std::string& texture( float x, float z ) const; // 取材质纹理名
    // ...
};
```

由于 `MaterialIndex` 是离散索引，对它做 bicubic 插值没意义，所以 [dominant_texture_map.hpp](file:///workspace/src/lib/terrain/dominant_texture_map.hpp):33–43 特化了 `Moo::Image<MaterialIndex>::getBicubic` 直接退化为 `get(int(x), int(y))`。

### 3.6 TerrainSettings 与 TerrainGraphicsOptions

[TerrainSettings](file:///workspace/src/lib/terrain/terrain_settings.hpp) 是从 `space.settings` 读取的、所有 block 共享的配置对象：

| 字段 | 含义 |
|------|------|
| `version_` | 100 / 200 |
| `heightMapSize_ / normalMapSize_ / holeMapSize_ / shadowMapSize_ / blendMapSize_` | 各子图分辨率 |
| `vertexLod_` (TerrainVertexLod) | 顶点 LOD 距离表 |
| `numVertexLods_` | LOD 层数 |
| `lodTextureStart_ / lodTextureDistance_ / blendPreloadDistance_` | LOD 纹理混合的起止距离 + 预加载缓冲 |
| `lodNormalStart_ / lodNormalDistance_ / normalPreloadDistance_` | 法线图 LOD 距离 + 预加载缓冲 |
| `defaultHeightMapLod_ / detailHeightMapDistance_` | 默认高度图 LOD + 何时加载 detail 高度图 |
| `directSoundOcclusion_ / reverbSoundOcclusion_` | 声音遮挡系数 |
| `pRenderer_` | 当前 renderer 实例 |

静态全局开关：

```cpp
// terrain_settings.hpp:103
static uint32 topVertexLod();        // 限制全局最高 LOD（性能档位）
static bool   useLodTexture();       // 是否启用 LOD 纹理
static bool   constantLod();         // 强制所有 block 同 LOD（编辑器调试用）
static bool   doBlockSplit();        // 是否对子块做 LOD 分割
```

`absoluteBlendPreloadDistance()` = `lodTextureStart + lodTextureDistance + blendPreloadDistance`，是"需要预加载 blends 的总距离阈值"。

[TerrainGraphicsOptions](file:///workspace/src/lib/terrain/terrain_graphics_options.hpp) 在客户端注册两个 GraphicsSetting 选项（LOD 距离档位、最高 LOD 档位），运行时通过 `lodModifier_` 修改所有 `TerrainSettings` 的距离。

---

## 4. Manager 与缓存

### 4.1 Terrain::Manager

[manager.hpp](file:///workspace/src/lib/terrain/manager.hpp) 是 terrain 库的入口单例，继承 `InitSingleton<Manager>`：

```cpp
// manager.hpp:49
class Manager : public InitSingleton<Manager>
{
public:
    const DataSectionPtr pTerrain2Defaults() const;
    BaseTerrainRenderer* currentRenderer();
    void currentRenderer( BaseTerrainRenderer* newInstance );
    BasicTerrainLodController& lodController();
    TerrainGraphicsOptions& graphicsOptions();
    void wireFrame( bool wireFrame );
    bool wireFrame() const;
#ifdef EDITOR_ENABLED
    TTL2Cache& ttl2Cache();
#endif
protected:
    bool doInit();
    bool doFini();
    // ...
};
```

`doInit()`（[manager.cpp](file:///workspace/src/lib/terrain/manager.cpp):93）做的事：

1. 初始化 `MaterialKinds`（`DominantTextureMap` 依赖）
2. 加载 `system/terrain2` 配置（AutoConfigString `g_terrain2Settings`）
3. 初始化 `TerrainGraphicsOptions`

`currentRenderer()` 在没有 renderer 时返回 `NullTerrainRenderer` 的静态实例，避免渲染时空指针：

```cpp
// manager.cpp:171
BaseTerrainRenderer* Manager::currentRenderer()
{
    if ( currentRenderer_ == NULL )
    {
        static NullTerrainRenderer nullRenderer;
        return &nullRenderer;
    }
    return currentRenderer_;
}
```

### 4.2 TerrainBlockCache

[terrain_block_cache.hpp](file:///workspace/src/lib/terrain/terrain_block_cache.hpp) 是服务端用于**多 space 共享地形几何**的缓存，按 resource ID 为 key：

```cpp
// terrain_block_cache.hpp:59
class TerrainBlockCache
{
public:
    static TerrainBlockCache& instance();
    TerrainBlockCacheEntryPtr findOrLoad( const std::string& resourceID,
                                          TerrainSettingsPtr pSettings );
    // ...
private:
    typedef std::map< std::string, TerrainBlockCacheEntry* > Map;
    Map map_;
    mutable SimpleMutex lock_;
};
```

`TerrainBlockCacheEntry` 把 cache 的引用计数与 `BaseTerrainBlockPtr` 的引用计数**分开**——前者持有 mutex，后者在 client 之间自由传递，避免每次 incRef/decRef 都要锁 cache。

### 4.3 TerrainFinder 与 TerrainCollisionCallback

- [TerrainFinder](file:///workspace/src/lib/terrain/terrain_finder.hpp) 是"按世界坐标找 block"的接口，由 `ChunkManager` 实现。返回的 `Details` 含 `(pBlock, pMatrix, pInvMatrix)` 三元组。
- [TerrainCollisionCallback](file:///workspace/src/lib/terrain/terrain_collision_callback.hpp) 是碰撞回调接口，每命中一个三角形就调用 `collide(triangle, tValue)`，返回 `true` 表示接受、`false` 表示继续找。

---

## 5. 高度图压缩

[height_map_compress.hpp](file:///workspace/src/lib/terrain/height_map_compress.hpp) / [.cpp](file:///workspace/src/lib/terrain/height_map_compress.cpp) 提供两个函数：

```cpp
// height_map_compress.hpp:31
BinaryPtr compressHeightMap(Moo::Image<float> const &heightMap);
bool      decompressHeightMap(BinaryPtr data, Moo::Image<float> &heightMap);
```

压缩策略：**量化到 1mm 网格 → 转 `int32` → PNG 32bpp 压缩**，魔数 `0x71706e67`（"qpng"）：

```cpp
// height_map_compress.cpp:39
inline int32 quantise(float h)
{
    return (int32)(h/QUANTISATION_LEVEL + 0.5f);  // QUANTISATION_LEVEL = 0.001f
}
```

注释里说"典型压缩比 8:1"，并且反复压缩-解压是稳定的（量化误差被吸收）。版本格式 `VERSION_REL_UINT16_PNG = 3`（见 [terrain_data.hpp](file:///workspace/src/lib/terrain/terrain_data.hpp):38）记录在 `HeightMapHeader.version_`。

---

## 6. terrain1（v1）实现

terrain1 是上一代实现，**无 LOD、无流式**，固定一个 vertex buffer + index buffer 直接画。

### 6.1 类层级

```
BaseTerrainBlock
   ▲
   │
CommonTerrainBlock1Base  (EditorBaseTerrainBlock 或 BaseTerrainBlock)
   ▲
   │
CommonTerrainBlock1     ← 加载 / 碰撞 / 高度查询
   ▲
   │
TerrainBlock1           ← 客户端：vertex buffer / lighting cache / DeviceCallback
   ▲
   │
EditorTerrainBlock1     ← 编辑器：保存 / 增删纹理层
```

### 6.2 CommonTerrainBlock1

[common_terrain_block1.hpp](file:///workspace/src/lib/terrain/terrain1/common_terrain_block1.hpp) 持有 v1 block 的核心数据：

```cpp
// common_terrain_block1.hpp:136
private:
    float spacing_;
    uint32 width_, height_;                    // 子图尺寸
    uint32 blocksWidth_, blocksHeight_;        // 可见块尺寸
    uint32 verticesWidth_, verticesHeight_;    // 顶点数
    uint32 detailWidth_, detailHeight_;        // detail 纹理尺寸
    uint32 numMapElements_, numTextures_;
    uint32 textureNameSize_;

    TerrainHeightMap1Ptr        pHeightMap_;
    TerrainHoleMap1Ptr          pHoleMap_;
    DominantTextureMapPtr        pDominantTextureMap_;
    mutable BoundingBox         bb_;
```

它还定义了一个 64 字的 header（`#pragma pack(push, 1)`）：

```cpp
// common_terrain_block1.hpp:110
struct TerrainBlockHeader
{
    uint32  version_;
    uint32  heightMapWidth_;
    uint32  heightMapHeight_;
    float   spacing_;
    uint32  nTextures_;
    uint32  textureNameSize_;
    uint32  detailWidth_;
    uint32  detailHeight_;
    uint32  padding_[64 - 8];        // 预留 64 个 uint32 给将来扩展
};
```

### 6.3 TerrainBlock1（客户端）

[terrain_block1.hpp](file:///workspace/src/lib/terrain/terrain1/terrain_block1.hpp) 在 `CommonTerrainBlock1` 上加了渲染资源：

```cpp
// terrain_block1.hpp:78
private:
    TextureLayers                textureLayers_;     // 多层 blend
    HorizonShadowMap1Ptr        pHorizonMap_;
    Moo::LightContainerPtr       diffLights_;       // 缓存的漫反射光
    Moo::LightContainerPtr       specLights_;       // 缓存的高光
    Moo::VertexBuffer            vertexBuffer_;     // 一个大 VB
    Moo::IndexBuffer             indexBuffer_;      // 一个大 IB
    uint32                       nVertices_;
    uint32                       nIndices_;
    uint32                       nPrimitives_;
```

继承 `Moo::DeviceCallback` 是为了 device lost/reset 时重建 VB/IB。

### 6.4 TerrainHeightMap1 / TerrainHoleMap1 / HorizonShadowMap1 / TerrainTextureLayer1

- [TerrainHeightMap1](file:///workspace/src/lib/terrain/terrain1/terrain_height_map1.hpp)：内部存 `Moo::Image<float> heights_`，minHeight/maxHeight/diagonalDistanceX4 缓存以加速 normalAt。提供 `collide` / `hmCollide` 系列碰撞方法。
- [TerrainHoleMap1](file:///workspace/src/lib/terrain/terrain1/terrain_hole_map1.hpp)：`Moo::Image<bool> image_`，持有 back-pointer 到 `CommonTerrainBlock1`。
- [HorizonShadowMap1](file:///workspace/src/lib/terrain/terrain1/horizon_shadow_map1.hpp)：`HorizonShadowImage image_`，按太阳角度预算地平线。
- [TerrainTextureLayer1](file:///workspace/src/lib/terrain/terrain1/terrain_texture_layer1.hpp)：每层 `Moo::Image<uint8> blends_` + `uProjection_/vProjection_` + `Moo::BaseTexturePtr pTexture_`。

### 6.5 TerrainRenderer1

[terrain_renderer1.hpp](file:///workspace/src/lib/terrain/terrain1/terrain_renderer1.hpp)：

```cpp
// terrain_renderer1.hpp:33
class TerrainRenderer1 : public BaseTerrainRenderer
{
private:
    typedef std::pair<Matrix, TerrainBlock1 *>  BlockInPlace;
    typedef AVectorNoDestructor<BlockInPlace>   BlockVector;

    class Renderer : public ReferenceCount { /* draw 接口 */ };
    class EffectFileRenderer : public Renderer
    {
        Moo::VertexDeclaration*        pDecl_;
        Moo::EffectMaterialPtr         material_;
        EffectFileTextureSetter*       textureSetter_;
        Moo::EffectConstantValuePtr   sunAngleSetter_;
        Moo::EffectConstantValuePtr   penumbraSizeSetter_;
        // ...
    };

    BlockVector              blocks_;
    SmartPointer<Renderer>   renderer_;
};
```

`version()` 返回 100，`addBlock` 把 block + 变换矩阵加入 `blocks_`，`drawAll` 调用 `renderer_->draw(blocks_)`。`EffectFileRenderer` 持有一个 `EffectMaterial`（fx 文件），并在 `setInitialRenderStates` / `sortBlocks` / `renderBlocks` 中完成绘制。

### 6.6 TerrainTextureSetter / EffectFileTextureSetter

[terrain_texture_setter.hpp](file:///workspace/src/lib/terrain/terrain1/terrain_texture_setter.hpp) 是 v1 给 effect 注入多层纹理的小工具：

```cpp
// terrain_texture_setter.hpp:51
void effect( ComObjectWrap<ID3DXEffect> pEffect )
{
    pEffect_ = pEffect;
    handles_.clear();
    handles_.push_back( pEffect->GetParameterByName(NULL,"Layer0") );
    handles_.push_back( pEffect->GetParameterByName(NULL,"Layer1") );
    handles_.push_back( pEffect->GetParameterByName(NULL,"Layer2") );
    handles_.push_back( pEffect->GetParameterByName(NULL,"Layer3") );
}
```

最多支持 4 层纹理（与 fx 文件中的 `Layer0~Layer3` sampler 对应）。

### 6.7 EditorTerrainBlock1

[editor_terrain_block1.hpp](file:///workspace/src/lib/terrain/terrain1/editor_terrain_block1.hpp) 在 `TerrainBlock1` 上加保存 / 增删层等接口，并维护 `std::vector<bool> isLayerEmpty_` 跟踪哪些层实际为空（便于"找最弱层"等优化）。

---

## 7. terrain2（v2）实现（重点）

terrain2 是当前主推实现，**全套可流式 + 顶点 LOD + 纹理 LOD + geomorphing**。

### 7.1 类层级

```
CommonTerrainBlock2Base (EditorBaseTerrainBlock / BaseTerrainBlock)
   ▲
   │
CommonTerrainBlock2     ← 加载 / 碰撞 / 高度查询
   ▲
   │
TerrainBlock2           ← 客户端：streaming / preDraw / draw / DeviceCallback
   ▲
   │
EditorTerrainBlock2     ← 编辑器：保存 / 增删层 / rebuildLodTexture
```

### 7.2 CommonTerrainBlock2

[common_terrain_block2.hpp](file:///workspace/src/lib/terrain/terrain2/common_terrain_block2.hpp)：

```cpp
// common_terrain_block2.hpp:43
class CommonTerrainBlock2 : public CommonTerrainBlock2Base
{
private:
    TerrainSettingsPtr          settings_;          // 配置指针
    TerrainHeightMap2Ptr        pHeightMap_;
    TerrainHoleMap2Ptr          pHoleMap_;
    DominantTextureMap2Ptr      pDominantTextureMap_;
    mutable BoundingBox         bb_;
};
```

`dataSectionName()` 返回 `"terrain2"`。`internalLoad` 接受一个 LOD 参数，决定加载哪一档高度图。

### 7.3 TerrainBlock2（客户端核心）

[terrain_block2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_block2.hpp) 是 v2 的灵魂。它持有所有可流式资源：

```cpp
// terrain_block2.hpp:183
protected:
    VerticesResourcePtr          pVerticesResource_;        // VertexLodManager
    TerrainBlendsResourcePtr     pBlendsResource_;          // 多层 blend
    HeightMapResourcePtr         pDetailHeightMapResource_; // detail 高度图

    TerrainNormalMap2Ptr         pNormalMap_;
    HorizonShadowMap2Ptr         pHorizonMap_;
    TerrainLodMap2Ptr            pLodMap_;

    Moo::LightContainerPtr       pDiffLights_;
    Moo::LightContainerPtr       pSpecLights_;

    DistanceInfo                 distanceInfo_;
    LodRenderInfo                lodRenderInfo_;

    uint32                       depthPassMark_;
    uint32                       preDrawMark_;
    std::string                  fileName_;
    DrawState                    currentDrawState_;
    friend class TerrainRenderer2;
```

`DistanceInfo` 与 `LodRenderInfo` 是两个关键结构：

```cpp
// terrain_block2.hpp:114
struct DistanceInfo
{
    Vector3  relativeCameraPos_;       // 相机相对块原点的位置
    float    minDistanceToCamera_;     // 块四角到相机的最近距离
    float    maxDistanceToCamera_;
    uint32   currentVertexLod_;        // 当前帧使用的顶点 LOD
    uint32   nextVertexLod_;           // morph 目标 LOD
};

struct LodRenderInfo
{
    MorphRanges     morphRanges_;      // 形态变换区间（main + subblock）
    NeighbourMasks  neighbourMasks_;   // 邻居 LOD 差（用于退化三角形）
    uint8           subBlockMask_;     // 哪些子块用低 LOD
    uint8           renderTextureMask_; // 该帧要画哪些纹理（RTM_DrawLOD 等）
};
```

`DrawState` 是一份"渲染期间不能变"的快照，防止流式加载中途换资源：

```cpp
// terrain_block2.hpp:174
struct DrawState
{
    VertexLodEntryPtr     currentVertexLodPtr_;
    VertexLodEntryPtr     nextVertexLodPtr_;
    TerrainBlendsPtr      blendsPtr_;
    bool                  blendsRendered_;
    bool                  lodsRendered_;
};
```

### 7.4 TerrainHeightMap2 与 AliasedHeightMap

[TerrainHeightMap2](file:///workspace/src/lib/terrain/terrain2/terrain_height_map2.hpp) 在 `TerrainHeightMap` 上加了：

- **`visibleOffset_`**：可见区域相对总图的偏移（默认 `DEFAULT_VISIBLE_OFFSET = 2`），即每边多 2 个不可见顶点用于跨块插值。
- **`lodLevel_`**：当前高度图的 LOD 等级（0 最高）。
- **`quadTree_`**：可变的 `TerrainQuadTreeCell`，懒加载。
- **`UnlockCallback`**：unlock 时通知（编辑器用它来重建 normal map）。

`refreshInternalDimensions` 计算 spacing / blocksWidth / 缓存 `diagonalDistanceX4` 以避免 `normalAt` 里的 `sqrtf`：

```cpp
// terrain_height_map2.hpp:230
inline void TerrainHeightMap2::refreshInternalDimensions()
{
    internalBlocksWidth_  = heights_.width()  - ( internalVisibleOffset() * 2 ) - 1;
    internalBlocksHeight_ = heights_.height() - ( internalVisibleOffset() * 2 ) - 1;
    internalSpacingX_ = BLOCK_SIZE_METRES / internalBlocksWidth_;
    internalSpacingZ_ = BLOCK_SIZE_METRES / internalBlocksHeight_;
    diagonalDistanceX4_ = sqrtf( internalSpacingX_ * internalSpacingZ_ * 2 ) * 4;
}
```

[AliasedHeightMap](file:///workspace/src/lib/terrain/terrain2/aliased_height_map.hpp) 是高度图的"多重采样抗锯齿"——它不存储自己的图，而是在采样时按 `level` 偏移到父图的对应位置：

```cpp
// aliased_height_map.cpp:43
float AliasedHeightMap::height( uint32 x, uint32 z ) const
{
    return pParent_->heightAt((int)(x << level_), (int)(z << level_));
}
```

`level` 即 mipmap 等级，`level=1` 时每 2 个采样跳 1 个，等价于把高分辨率高度图"降采样"成低分辨率——但内存里只有一份。`VertexLodEntry` 用 `AliasedHeightMap` 生成不同 LOD 的顶点 buffer。

### 7.5 TerrainQuadTreeCell：四叉树碰撞

[terrain_quad_tree_cell.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_quad_tree_cell.hpp) / [.cpp](file:///workspace/src/lib/terrain/terrain2/terrain_quad_tree_cell.cpp) 是**碰撞查询**用的四叉树（不是渲染 LOD 用的）：

```cpp
// terrain_quad_tree_cell.hpp:32
class TerrainQuadTreeCell
{
public:
    void init( const TerrainHeightMap2* map,
        uint32 xOffset, uint32 zOffset, uint32 xSize, uint32 zSize,
        float xMin, float zMin, float xMax, float zMax, float minCellSize );

    bool collide( const Vector3& source, const Vector3& extent,
        const TerrainHeightMap2* pOwner, TerrainCollisionCallback* pCallback ) const;
    bool collide( const WorldTriangle& source, const Vector3& extent,
        const TerrainHeightMap2* pOwner, TerrainCollisionCallback* pCallback ) const;
private:
    BoundingBox                        boundingBox_;
    std::vector<TerrainQuadTreeCell>   children_;
};
```

构造算法（递归二分）：

```cpp
// terrain_quad_tree_cell.cpp:337
void TerrainQuadTreeCell::init( const TerrainHeightMap2* map,
    uint32 xOffset, uint32 zOffset, uint32 xSize, uint32 zSize,
    float xMin, float zMin, float xMax, float zMax, float minCellSize )
{
    if (xSize > minCellSize && zSize > minCellSize)
    {
        children_.resize(4);
        // 4 个子 cell 分别覆盖四个象限
        children_[0].init( map, xOffset,        zOffset,        halfXSize, halfZSize,
                           xMin,                zMin,                xMax-halfXDiff, zMax-halfZDiff, minCellSize);
        children_[1].init( map, xOffset+halfXSize, zOffset,        halfXSize, halfZSize,
                           xMin+halfXDiff,      zMin,                xMax,           zMax-halfZDiff, minCellSize);
        children_[2].init( map, xOffset,        zOffset+halfZSize, halfXSize, halfZSize,
                           xMin,                zMin+halfZDiff,      xMax-halfXDiff, zMax,           minCellSize);
        children_[3].init( map, xOffset+halfXSize, zOffset+halfZSize, halfXSize, halfZSize,
                           xMin+halfXDiff,      zMin+halfZDiff,      xMax,           zMax,           minCellSize);
        // 父 boundingBox = 4 个子的并集
        boundingBox_ = children_[0].boundingBox_;
        boundingBox_.addBounds( children_[1].boundingBox_ );
        boundingBox_.addBounds( children_[2].boundingBox_ );
        boundingBox_.addBounds( children_[3].boundingBox_ );
    }
    else
    {
        // 叶子：直接扫所有顶点算 boundingBox
        for (uint32 z = zOffset; z < (zOffset + zSize + 1); z++)
            for (uint32 x = xOffset; x < (xOffset + xSize + 1); x++)
                boundingBox_.addYBounds( map->heightAt((int)x, (int)z) );
    }
}
```

碰撞算法：

1. **线段碰撞**：先用 `boundingBox_.clip(s, e)` 把线段裁到 cell 内；如果当前是叶子，调用 `pOwner->hmCollide(...)` 做网格三角形扫描；否则按线段端点所在象限决定先访问哪个子 cell，避免全树遍历：

```cpp
// terrain_quad_tree_cell.cpp:433
uint32 startQuadrant = (sNormalised.x > 0.5f ? 1 : 0) + (sNormalised.z > 0.5f ? 2 : 0);
uint32 endQuadrant   = (eNormalised.x > 0.5f ? 1 : 0) + (eNormalised.z > 0.5f ? 2 : 0);

if (startQuadrant == endQuadrant)
    return children_[startQuadrant].collide(...);
else if ((startQuadrant & endQuadrant) == 0)
    // 不相邻，扫 startQuadrant, startQuadrant^1, endQuadrant^1, endQuadrant
else
    // 相邻，只扫 startQuadrant, endQuadrant
```

2. **棱柱碰撞**（`WorldTriangle` 起点 + `Vector3` 终点的 prism）：`clipPrism` 用 6 面 outcode 把三角形投影到 bounding box 边界，再做子 cell 递归。这处理"三角形扫过体积"的碰撞（角色移动、AOE 范围等）。

阈值 `quadtreeThreshold()` = `internalSpacingX_ * 4`，即子 cell 小于 4 个 spacing 时停止分裂。

### 7.6 TerrainHoleMap2 / HorizonShadowMap2 / DominantTextureMap2 / TerrainNormalMap2 / TerrainLodMap2

| 类 | 文件 | 职责 |
|----|------|------|
| TerrainHoleMap2 | [terrain_hole_map2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_hole_map2.hpp) | 持 `ComObjectWrap<DX::Texture> texture_`，在 `unlock()` 时上传到 D3D 纹理；`holeAt(x,z)` 用于碰撞、阴影剔除 |
| HorizonShadowMap2 | [horizon_shadow_map2.hpp](file:///workspace/src/lib/terrain/terrain2/horizon_shadow_map2.hpp) | 直接持 `DX::Texture`，由编辑器 `create(size)` 生成 |
| DominantTextureMap2 | [dominant_texture_map2.hpp](file:///workspace/src/lib/terrain/terrain2/dominant_texture_map2.hpp) | 在 `DominantTextureMap` 上加 `load / save` |
| TerrainNormalMap2 | [terrain_normal_map2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_normal_map2.hpp) | **流式**：`pNormalMap_`（高质量）+ `pLodNormalMap_`（低质量，永远在内存），后台 `NormalMapTask` 加载高质量 |
| TerrainLodMap2 | [terrain_lod_map2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_lod_map2.hpp) | LOD 纹理（一张预渲染好的 block 概览图），`load/save` 走 DataSection |

`TerrainNormalMap2` 的双图设计很关键：

```cpp
// terrain_normal_map2.hpp:58
ComObjectWrap<DX::Texture> pMap() const
{
    return pNormalMap_.hasComObject() ? pNormalMap_ : pLodNormalMap_;
}
```

近距离显示高质量法线（`pNormalMap_`，可能还在后台加载），远距离或还在加载时退回到 `pLodNormalMap_`（永远在内存）。

### 7.7 顶点 LOD 体系

#### 7.7.1 TerrainVertexLod（距离表）

[terrain_vertex_lod.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_vertex_lod.hpp) 是一个 `std::vector<float>`——一串距离阈值，每项代表"该 LOD 适用的最远距离"：

```cpp
// terrain_vertex_lod.hpp:23
class TerrainVertexLod : public std::vector<float>
{
public:
    uint32  calculateLodLevel( float dist ) const;
    void    calculateMasks( const TerrainBlock2::DistanceInfo& distanceInfo,
                            TerrainBlock2::LodRenderInfo& lodRenderInfo ) const;
    float   distance( uint32 lod ) const;
    void    distance( uint32 lod, float value );
    float   startBias() const;
    float   endBias() const;
    static float xzDistance( const Vector3& chunkCorner, float blockSize,
                             const Vector3& cameraPosition );
    static void minMaxXZDistance( const Vector3& relativeCameraPos,
        const float blockSize, float& minDistance, float& maxDistance );
private:
    void internalSubBlockTests( NeighbourMasks& neighbourMasks, uint8 subBlockMask ) const;
    void externalSubBlockTests( uint32 mainBlockLod, uint8 subBlockMask,
        const Vector3& cameraPosition, NeighbourMasks& neighbourMasks ) const;
    Vector2 calcMorphRanges( uint32 lodLevel ) const;
    float startBias_, endBias_;
};
```

`startBias_ / endBias_` 控制 geomorphing 在 LOD 切换区间的起始/结束比例（默认 0.3 / 0.7）。

`calculateLodLevel(dist)` 简单遍历距离表找到第一个 `dist <= distance_[lod]` 的 LOD。

`calculateMasks` 算出 `LodRenderInfo`：当前 / 下一 LOD、4 个邻居各自的 LOD 差（用于在边界生成退化三角形）、4 个子块各自的 LOD、形态变换区间。

#### 7.7.2 VertexLodEntry（单 LOD 的 VB+IB）

[vertex_lod_entry.hpp](file:///workspace/src/lib/terrain/terrain2/vertex_lod_entry.hpp)：

```cpp
// vertex_lod_entry.hpp:45
class VertexLodEntry : public SafeReferenceCount
{
public:
    bool init( const AliasedHeightMap* baseMap,
               const AliasedHeightMap* previousMap,
               uint32                  gridSize );

    bool load( DataSectionPtr pTerrain, uint32 level,
               BinaryPtr& pVertices, uint32& gridSize );

    bool draw( Moo::EffectMaterialPtr pMaterial,
               const Vector2&          morphRanges,
               const NeighbourMasks&   neighbourMasks,
               uint8                   subBlockMask );
private:
    TerrainIndexBufferPtr    pIndexes_;       // 索引 buffer
    TerrainVertexBufferPtr   pVertices_;      // 顶点 buffer
};
```

`init` 用 `AliasedHeightMap` 生成顶点，每个顶点存两个高度：当前 LOD 和下一 LOD，shader 据此做 geomorphing。

`draw` 根据 `neighbourMasks` 选择哪个子块画、`subBlockMask` 选择主块还是子块、`morphRanges` 控制形态变换进度。

#### 7.7.3 VertexLodManager（管理一个 block 的所有 LOD）

[vertex_lod_manager.hpp](file:///workspace/src/lib/terrain/terrain2/vertex_lod_manager.hpp)：

```cpp
// vertex_lod_manager.hpp:42
class VertexLodManager : public Resource< VertexLodArray >
{
public:
    struct WorkingSet
    {
        uint32 start_, end_;
        bool IsWithin( const WorkingSet & other );
        bool IsOverlapping( const WorkingSet & other );
    };

    VertexLodManager( TerrainBlock2& owner, uint32 numLods );
    void evaluate( uint32 lod, uint32 topLod );
    virtual void stream( ResourceStreamType streamType = RST_Asyncronous );
    VertexLodEntryPtr getLod( uint32 level, bool doSubstitution = false );
    void getCurrentWorkingSet( WorkingSet& workingSet ) const;
    void getRequestedWorkingSet( WorkingSet& workingSet ) const;
    static inline uint32 getLodSize( uint32 lodLevel, uint32 numLods )
    { return 1 << ( numLods - lodLevel ); }       // LOD 0 → 2^(N-1), LOD N-1 → 1
protected:
    void getTargetWorkingSet( uint32 lod, WorkingSet & workingSet, uint32 topLod ) const;
    void evictNotInSet( const WorkingSet & workingSet, bool preserveOne );
    virtual bool generate( uint32 level );

    TerrainBlock2&  owner_;
    WorkingSet      currentWorkingSet_;      // 当前已加载
    WorkingSet      requestedWorkingSet_;    // 期望加载
    WorkingSet      loadingWorkingSet_;      // 正在加载
```

关键概念：**WorkingSet**——一组连续的 LOD 索引。`getTargetWorkingSet` 返回 `[lod-1, lod+1]`（边界裁剪），即同时加载当前 LOD + 上一 LOD（用于 morph）+ 下一 LOD（用于预判）。`evaluate(lod, topLod)` 计算出 `requestedWorkingSet_`，`stream()` 异步加载缺失的 LOD、卸载多余的：

```cpp
// vertex_lod_manager.hpp:157
inline void VertexLodManager::getTargetWorkingSet(
    uint32 lod, WorkingSet& workingSet, uint32 topLod ) const
{
    workingSet.start_ = lod;
    if ( workingSet.start_ > topLod ) workingSet.start_--;
    workingSet.end_ = lod;
    if ( workingSet.end_ < object_->size() - 1) workingSet.end_++;
}
```

`getLod(level, doSubstitution)` 返回该 LOD 的 `VertexLodEntry`，如果它还在加载中且 `doSubstitution=true`，则用最近的可用的 LOD 替代（避免渲染卡顿）。

`getLodSize` 是几何级数：`numLods=8` 时 LOD 0 是 128x128 网格、LOD 7 是 1x1。注释里说 "LOD 0 is highest"。

#### 7.7.4 EditorVertexLodManager

[editor_vertex_lod_manager.hpp](file:///workspace/src/lib/terrain/terrain2/editor_vertex_lod_manager.hpp) 在编辑器里**强制同步**重新生成所有 LOD（`isDirty_` 标记），因为编辑器修改高度图后必须立即可见，不允许流式延迟。

### 7.8 GridVertexBuffer / TerrainVertexBuffer / TerrainIndexBuffer

#### 7.8.1 GridVertexBuffer

[grid_vertex_buffer.hpp](file:///workspace/src/lib/terrain/terrain2/grid_vertex_buffer.hpp) 是一个**全局共享**的网格顶点 buffer，按 `(resolutionX, resolutionZ)` 为 key 缓存：

```cpp
// grid_vertex_buffer.hpp:26
class GridVertexBuffer : public SafeReferenceCount
{
public:
    static GridVertexBuffer* get( uint16 resolutionX, uint16 resolutionZ );
    Moo::VertexBuffer pBuffer() const { return pVertexBuffer_; };
private:
    static std::map< uint32, GridVertexBuffer* > s_gridVertexBuffers_;
    static SimpleMutex s_mutex_;
};
```

它的顶点数据是 `[0,1]×[0,1]` 的网格 UV，shader 用 `mul(gridUV, blockSize)` 算出世界坐标——所有同样分辨率的 block 共用一个 GridVertexBuffer，节省显存。

#### 7.8.2 TerrainVertexBuffer

[terrain_vertex_buffer.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_vertex_buffer.hpp) 是单 LOD 的顶点 buffer，每个顶点存 `(x_height, z_height, x_nextHeight, z_nextHeight)` 两个高度（current + next LOD），用于 geomorphing：

```cpp
// terrain_vertex_buffer.hpp:30
class TerrainVertexBuffer : public SafeReferenceCount
{
public:
    static void generate( const AliasedHeightMap* hm, const AliasedHeightMap* previousLOD,
                         uint32 resolutionX, uint32 resolutionZ,
                         std::vector< Vector2 > & vertices );
    bool init( const Vector2 * vertices, uint32 resolutionX, uint32 resolutionZ, uint32 usage );
    bool set();
    Moo::VertexBuffer getBuffer() { return pVertexBuffer_; }
private:
    Moo::VertexBuffer                pVertexBuffer_;
    SmartPointer<GridVertexBuffer>   pGridBuffer_;       // 复用 GridVB 的 UV 部分
};
```

#### 7.8.3 TerrainIndexBuffer

[terrain_index_buffer.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_index_buffer.hpp) 是按 `(quadCountX, quadCountZ)` 全局共享的索引 buffer：

```cpp
// terrain_index_buffer.hpp:59
class TerrainIndexBuffer : public Moo::DeviceCallback, public SafeReferenceCount
{
public:
    enum {
        DIRECTION_POSITIVEX = 0x01,
        DIRECTION_NEGATIVEX = 0x02,
        DIRECTION_POSITIVEZ = 0x04,
        DIRECTION_NEGATIVEZ = 0x08
    };
    static TerrainIndexBufferPtr get( uint32 quadCountX, uint32 quadCountZ );
    void draw( Moo::EffectMaterialPtr pMaterial, const Vector2& morphRanges,
               const NeighbourMasks& neighbourMasks, uint8 subBlockMask );
private:
    typedef uint32 IndexType;          // 32 位索引
    enum TriangleListOrder { TLO_TILES_ROWS = 0, TLO_TILES_SWIZZLED };

    void generateIndices(TriangleListOrder order, IndexType* pOut, uint32 bufferSize) const;
    void generateDegenerates(IndexType xStart, IndexType zStart,
                             IndexType xEnd,  IndexType zEnd,
                             IndexType rowSize, IndexType*& pBuffer) const;

    uint32 quadCountX_, quadCountZ_;
    uint32 indexCount_, subBlockDegIndexCount_;
    Moo::IndexBuffer pIndexBuffer_;
    static IndexBufferMap s_buffers_;       // (qx,qz) → buffer
    static SimpleMutex    s_mutex_;
};
```

`generateDegenerates` 在 LOD 边界生成退化三角形（共线三角形不画像素但保持 topology 连续），这就是经典的 **Geometry Clipmap / Geomipmapping** 技巧——邻居 LOD 差 1 时，沿边界 stitch 一行退化三角形避免 T-junction 漏洞。

索引顺序可选 `TLO_TILES_ROWS`（普通行主序）或 `TLO_TILES_SWIZZLED`（Morton/Z 曲线序，提高 cache 命中率）。

### 7.9 TerrainBlends / CombinedLayer / TerrainTextureLayer2 / TTL2Cache

#### 7.9.1 TerrainBlends

[terrain_blends.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_blends.hpp)：

```cpp
// terrain_blends.hpp:32
struct CombinedLayer
{
    uint32                       width_, height_;
    ComObjectWrap<DX::Texture>   pBlendTexture_;     // 合并后的 blend 纹理
    TextureLayers                textureLayers_;     // 该合并层包含的源层
    bool                         smallBlended_;      // 是否压缩成 8/16 bit
};

struct TerrainBlends : SafeReferenceCount
{
    bool init( TerrainBlock2& owner );
    void createCombinedLayers( bool compressTextures,
                               DominantTextureMap2Ptr* newDominantTexture);

    TextureLayers                 textureLayers_;     // 所有源层
    std::vector<CombinedLayer>    combinedLayers_;    // 合并后的层（最多 4 层/张）
};
```

合并策略：把 N 个源层每 4 个一组打包成一张 `CombinedLayer`，调用 `TerrainTextureLayer::createBlendTexture` 生成 D3D 纹理。如果硬件支持 `D3DFMT_L8 / A8L8`，可以打成 8/16 bit 节省显存（`smallBlended_=true`）。

#### 7.9.2 TerrainBlendsResource

`TerrainBlendsResource : public Resource<TerrainBlends>`：流式加载 blends，`evaluate(renderTextureMask)` 根据 renderer 标志决定是否需要：

```cpp
// terrain_blends.hpp:82
inline ResourceRequired TerrainBlendsResource::evaluate( uint8 renderTextureMask )
{
    required_ = RR_No;
    if (renderTextureMask & (TerrainRenderer2::RTM_PreLoadBlend | TerrainRenderer2::RTM_DrawBlend))
        required_ = RR_Yes;
    return required_;
}
```

#### 7.9.3 TerrainTextureLayer2 / TTL2Cache

[terrain_texture_layer2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_texture_layer2.hpp) 在编辑器构建里加了 `compressedBlend_` 字段，blend 数据可存为压缩或解压形态。

[TTL2Cache](file:///workspace/src/lib/terrain/terrain2/ttl2cache.hpp) 是编辑器专用的 LRU 缓存，保留最近 1024 个解压后的 `TerrainTextureLayer2`，避免反复解压：

```cpp
// ttl2cache.hpp:28
class TTL2Cache
{
public:
    static const uint32 CACHE_SIZE = 1024;
    static TTL2Cache *instance();
    void onLock(TerrainTextureLayer2 *layer, bool readOnly);
    void onUnlock(TerrainTextureLayer2 *layer);
    void delTextureLayer(TerrainTextureLayer2 *layer);
private:
    typedef std::list<TerrainTextureLayer2*> LayerList;
    LayerList layers_;
};
```

### 7.10 TerrainLodTexture / TerrainPhotographer

#### 7.10.1 TerrainLodTexture

[terrain_lod_texture.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_lod_texture.hpp) 是一个**纹理 atlas**，把多个小块纹理拼接成一张大纹理，但允许纹理之间互相"渗透"（bleed），用于远距离的整块地形纹理：

```cpp
// terrain_lod_texture.hpp:28
class TerrainLodTexture : public ReferenceCount, public Moo::DeviceCallback
{
public:
    bool init( uint32 textureSize, uint32 lodSize, D3DFORMAT textureFormat );
    TerrainLodTextureEntryPtr addTextureTile( DX::Texture* pTexture,
                                              uint32 uTile, uint32 vTile );
    DX::Texture* pTexture() { return pTexture_.pComObject(); }
private:
    uint32 lodItemSize_;        // 单块尺寸
    uint32 textureSize_;       // 整张尺寸
    uint32 rowSize_;            // 一行能放几块
    float  lodItemFraction_;
    D3DFORMAT textureFormat_;
    ComObjectWrap<DX::Texture> pTexture_;
};
```

#### 7.10.2 TerrainPhotographer

[terrain_photographer.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_photographer.hpp) 用于**把一个 block 渲染成一张 LOD 纹理**（"截图"）——离线烘焙时用：

```cpp
// terrain_photographer.hpp:26
class TerrainPhotographer
{
public:
    bool init( uint32 basePhotoSize );
    bool photographBlock( BaseTerrainBlock* pBlock, const Matrix& transform );
    bool output( ComObjectWrap<DX::Texture>& pDestTexture, D3DFORMAT destImageFormat );
private:
    Moo::RenderTargetPtr pBasePhoto_;
};
```

它持一个 `Moo::RenderTarget`，把 block 用 textured effect 画到 RT 上，最后 `output` 拷出纹理。仅 `EDITOR_ENABLED` 时存在。

### 7.11 TerrainLodController / BasicTerrainLodController

[terrain_lod_controller.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_lod_controller.hpp) 是两个 LOD 控制器：

- `TerrainLodController`：按相机位置 + 块网格坐标管理"焦点 block 窗口"（旧实现，似乎未实际使用）。
- `BasicTerrainLodController`：当前使用的版本，全局单例，按相机位置 tick 所有 block 的 `evaluate + stream`：

```cpp
// terrain_lod_controller.hpp:56
class BasicTerrainLodController
{
public:
    void setCameraPosition( const Vector3& position );
    void addBlock( TerrainBlock2* pBlock, const Matrix& worldTransform );
    bool delBlock( TerrainBlock2* pBlock );
    float closestUnstreamedBlock() const;
    static BasicTerrainLodController& instance();
private:
    float closestUnstreamedBlock_;             // 最近未流式完成的块距离（FLT_MAX 表示无）
    typedef std::pair< Matrix, TerrainBlock2* > BlockEntry;
    typedef std::avector< BlockEntry >         BlockContainer;
    BlockContainer  blocks_;
    SimpleMutex     accessMutex_;
};
```

`Manager::lodController()` 返回它的实例（见 [manager.hpp](file:///workspace/src/lib/terrain/manager.hpp):62）。

### 7.12 TerrainRenderer2

[terrain_renderer2.hpp](file:///workspace/src/lib/terrain/terrain2/terrain_renderer2.hpp) 是 v2 的渲染器，包含 4 个 material 子类：

```cpp
// terrain_renderer2.hpp:48
class BaseMaterial
{
public:
    Moo::EffectMaterialPtr  pEffect_;
    D3DXHANDLE  viewProj_, world_, cameraPosition_;
    D3DXHANDLE  normalMap_, normalMapSize_;
    D3DXHANDLE  horizonMap_, horizonMapSize_;
    D3DXHANDLE  holesMap_, holesMapSize_, holesSize_;
    D3DXHANDLE  layer_[4], layerUProjection_[4], layerVProjection_[4];
    D3DXHANDLE  blendMap_, blendMapSize_, layerMask_;
    D3DXHANDLE  lodTextureStart_, lodTextureDistance_;
    D3DXHANDLE  useMultipassBlending_, hasHoles_;
};

class TexturedMaterial : public BaseMaterial { /* specular 参数 */ };
class LodTextureMaterial : public BaseMaterial { /* LOD 纹理专用 */ };
class ZPassMaterial : public BaseMaterial { /* Z pre-pass 专用 */ };
```

`RenderTextureMask` 是核心调度位：

```cpp
// terrain_renderer2.hpp:159
enum RenderTextureMask
{
    RTM_None           = 0,
    RTM_DrawLOD        = 1,         // 画 LOD 纹理
    RTM_DrawBlend      = 1 << 1,    // 画 blend
    RTM_PreLoadBlend   = 1 << 2,    // 预加载 blend
    RTM_DrawLODNormals = 1 << 3,   // 用低分辨率法线 + LOD 纹理
    RTM_PreloadNormals = 1 << 4    // 预加载高质量法线
};
```

`getLoadFlag` / `getDrawFlag` 把距离翻译成 mask：

```cpp
// terrain_renderer2.hpp:268
inline uint8 TerrainRenderer2::getLoadFlag(float minDistance,
        float textureBlendsPreloadDistance, float normalPreloadDistance)
{
    if ( minDistance <= textureBlendsPreloadDistance )
        return RTM_PreLoadBlend | RTM_PreloadNormals;
    if ( minDistance <= normalPreloadDistance )
        return RTM_PreloadNormals;
    return RTM_DrawLODNormals;
}
```

`DrawMethod` 描述实际绘制路径：

```cpp
// terrain_renderer2.hpp:171
enum DrawMethod
{
    DM_Z_ONLY,          // Z pre-pass（处理洞时让 stencil 正确）
    DM_SINGLE_LOD,      // 单 pass LOD 纹理
    DM_SINGLE_OVERRIDE, // 单 pass 用 override material（编辑器拾取等）
    DM_BLEND,           // 多 pass 完整 blends
    DM_LOD_NORMALS      // 低分辨率法线 + LOD 纹理
};
```

`drawSingleInternal` 的核心调度逻辑（见 [terrain_renderer2.cpp](file:///workspace/src/lib/terrain/terrain2/terrain_renderer2.cpp):510）：

1. 设 world transform
2. `tb2->preDraw(useCachedLighting, true)` 让 block 把当前 LOD、blends、normal map 拷到 `currentDrawState_`，并替换为可用的 LOD（找不到则失败，画 magenta placeholder）
3. 设 vertex declaration
4. 取 `currentDrawState_.blendsPtr_`
5. 根据 `lodRenderInfo_.renderTextureMask_` 决定：
   - 如果有 holes 且未做过 Z pass → 先 `DM_Z_ONLY`
   - `RTM_DrawBlend` → `drawTerrainMaterial(tb2, &texturedMaterial_, DM_BLEND)`
   - `RTM_DrawLOD` → `drawTerrainMaterial(tb2, &lodTextureMaterial_, DM_SINGLE_LOD)`
   - `RTM_DrawLODNormals` → `drawTerrainMaterial(tb2, &lodTextureMaterial_, DM_LOD_NORMALS)`

`drawTerrainMaterial` 内部按 CombinedLayer 数量循环多 pass，每 pass 设 4 个 layer 纹理 + 对应的 u/v 投影 + blend texture + layerMask，调用 `VertexLodEntry::draw`。

### 7.13 Resource<> 模板（流式加载基础设施）

[resource.hpp](file:///workspace/src/lib/terrain/terrain2/resource.hpp) 是所有可流式资源的基类模板：

```cpp
// resource.hpp:74
template < typename O >
class Resource : public SafeReferenceCount
{
public:
    typedef O                         ObjectType;
    typedef SmartPointer<ObjectType>  ObjectTypePtr;

    class ResourceTask : public BackgroundTask
    {
    public:
        virtual void doBackgroundTask( BgTaskManager & mgr )
        {
            resource_->load();              // 后台线程：load
            mgr.addMainThreadTask( this );  // 完成后回主线程
        }
        virtual void doMainThreadTask( BgTaskManager & mgr )
        {
            resource_->postAsyncLoad();     // 主线程：上传 D3D 资源
        }
    };

    ResourceRequired evaluate( bool required = false );
    virtual void stream( ResourceStreamType streamType = RST_Asyncronous );
    inline ObjectTypePtr getObject();
    inline bool         isRequired() const;
    virtual ResourceState getState() const;
protected:
    virtual bool load() = 0;
    virtual void unload();
    virtual void preAsyncLoad() {}
    virtual void startAsyncTask();
    virtual void postAsyncLoad() {}

    ResourceRequired   required_;
    ObjectTypePtr      object_;
    ResourceStreamType streamType_;
private:
    ResourceTask*      task_;
};
```

状态机：

```
              evaluate()
   RS_Unloaded ───────────► required_=RR_Yes
        │                         │
        │                         │ stream()
        │                         ▼
        │                   startAsyncTask()
        │                         │
        │                         ▼
        │                  RS_Loading (task_ != NULL)
        │                         │
        │                  doBackgroundTask → load()
        │                         │
        │                  doMainThreadTask → postAsyncLoad()
        │                         │
        │                         ▼
        │                   RS_Loaded (object_ != NULL)
        │                         │
        │ evaluate(false)          │
        │ + stream()               │
        └─────────────────────────► unload() → object_=NULL → RS_Unloaded
```

派生类只需实现 `load()` 与自定义的 `evaluate(...)`，例如 `HeightMapResource::evaluate(requiredVertexGridSize, standardHeightMapSize, detailHeightMapDistance, blockDistance, topLodLevel)` 决定是否需要 detail 高度图。

---

## 8. 数据布局

### 8.1 内存中的 TerrainBlock2 数据布局

```
TerrainBlock2 (sizeof ≈ 数百字节)
├─ CommonTerrainBlock2 部分
│  ├─ TerrainSettingsPtr          settings_            (4B 指针 + 引用计数)
│  ├─ TerrainHeightMap2Ptr        pHeightMap_          (smart pointer)
│  ├─ TerrainHoleMap2Ptr           pHoleMap_
│  ├─ DominantTextureMap2Ptr       pDominantTextureMap_
│  └─ mutable BoundingBox          bb_                  (28B)
│
├─ TerrainBlock2 自己
│  ├─ VerticesResourcePtr          pVerticesResource_  → VertexLodManager
│  │                                                      ├─ owner_
│  │                                                      ├─ WorkingSet current_ / requested_ / loading_
│  │                                                      └─ VertexLodArray object_
│  │                                                           └─ 每项 VertexLodEntryPtr
│  │                                                                ├─ TerrainVertexBufferPtr  pVertices_
│  │                                                                │    ├─ Moo::VertexBuffer
│  │                                                                │    └─ SmartPointer<GridVertexBuffer> pGridBuffer_
│  │                                                                └─ TerrainIndexBufferPtr pIndexes_
│  │                                                                     └─ Moo::IndexBuffer
│  │
│  ├─ TerrainBlendsResourcePtr     pBlendsResource_    → TerrainBlendsResource
│  │                                                      └─ object_ = TerrainBlends
│  │                                                           ├─ TextureLayers textureLayers_
│  │                                                           └─ vector<CombinedLayer> combinedLayers_
│  │                                                                └─ 每项 ComObjectWrap<DX::Texture> pBlendTexture_
│  │
│  ├─ HeightMapResourcePtr         pDetailHeightMapResource_ → HeightMapResource
│  │                                                          └─ TerrainHeightMap2 (detail LOD)
│  │
│  ├─ TerrainNormalMap2Ptr         pNormalMap_
│  │   ├─ ComObjectWrap<DX::Texture>  pNormalMap_      (高质量，流式)
│  │   ├─ ComObjectWrap<DX::Texture>  pLodNormalMap_   (低质量，常驻)
│  │   └─ NormalMapTask*              pTask_
│  │
│  ├─ HorizonShadowMap2Ptr         pHorizonMap_
│  │   ├─ ComObjectWrap<DX::Texture>  texture_
│  │   └─ HorizonShadowImage          image_
│  │
│  ├─ TerrainLodMap2Ptr             pLodMap_           (一张预渲染纹理)
│  │
│  ├─ Moo::LightContainerPtr       pDiffLights_       (缓存的环境光 + 方向光)
│  ├─ Moo::LightContainerPtr       pSpecLights_       (缓存的高光，可空)
│  │
│  ├─ DistanceInfo                  distanceInfo_
│  ├─ LodRenderInfo                 lodRenderInfo_
│  ├─ uint32                        depthPassMark_
│  ├─ uint32                        preDrawMark_
│  ├─ std::string                   fileName_
│  └─ DrawState                     currentDrawState_   (一帧渲染期间不变)
```

### 8.2 磁盘上的 .cdata 数据布局

地形数据存在 `.cdata` 文件的 `terrain` 或 `terrain2` 子 section 下，由 [terrain_data.hpp](file:///workspace/src/lib/terrain/terrain_data.hpp) 中的 headers 描述：

| Header | magic | 含义 |
|--------|-------|------|
| `HeightMapHeader` | `0x00706d68` ("hmp\0") | 高度图：`width_ / height_ / compression_ / version_ / minHeight_ / maxHeight_`，version 4 = `VERSION_ABS_QFLOAT`（量化浮点） |
| `BlendHeader` | `0x00646c62` ("bld\0") | 单 blend 层：`width_ / height_ / bpp_ / uProjection_ / vProjection_ / version_` |
| `ShadowHeader` | `0x00646873` ("shd\0") | 地平线阴影图 |
| `HolesHeader` | `0x006c6f68` ("hol\0") | 洞图 |
| `VertexLODHeader` | `0x00726576` ("ver\0") | 顶点 LOD 数据：`version_ / gridSize_`，version 2 = zip 压缩 |
| `TerrainBlock1Header` | (无 magic) | v1 总 header，64 字 |
| `TerrainNormalMapHeader` | `0x006d726e` ("nrm\0") | 法线图，version 1 = `VERSION_16_BIT_PNG` |
| `DominantTextureMapHeader` | `0x0074616d` ("mat\0") | 主导纹理图：`numTextures_ / textureNameSize_ / width_ / height_`，version 1 = zip |

全局常量：

```cpp
// terrain_data.hpp:17
const float BLOCK_SIZE_METRES      = 100.0f;
const float SUB_BLOCK_SIZE_METRES  = BLOCK_SIZE_METRES / 2.0f;     // 50m，子块 LOD
```

---

## 9. LOD 选择算法

### 9.1 顶点 LOD 选择

伪代码（基于 [TerrainVertexLod](file:///workspace/src/lib/terrain/terrain2/terrain_vertex_lod.hpp) 与 `TerrainBlock2::evaluate`）：

```
输入：cameraPosition（世界）, blockWorldTransform
输出：DistanceInfo, LodRenderInfo

1. relativeCameraPos = invWorldTransform.applyPoint(cameraPosition)
2. minMaxXZDistance(relativeCameraPos, BLOCK_SIZE_METRES,
                   out minDistance, out maxDistance)
   - minDistance = max(0, max(|relativeCameraPos.x|, |relativeCameraPos.z|) - BLOCK_SIZE_METRES/2)
   - maxDistance = sqrt(relativeCameraPos.x^2 + relativeCameraPos.z^2) + 半对角线
3. currentVertexLod = vertexLod.calculateLodLevel(minDistance)
4. nextVertexLod    = vertexLod.calculateLodLevel(maxDistance)
5. morphRanges      = vertexLod.calcMorphRanges(currentVertexLod)
6. subBlockMask    = (maxDistance < threshold) ? 全 1 : 0
                     // 子块 LOD：如果块内距离差大，部分子块用低 LOD
7. neighbourMasks   = internalSubBlockTests + externalSubBlockTests
                     // 比较邻居 LOD 决定边界 stitch 方向
8. renderTextureMask = TerrainRenderer2::getLoadFlag(minDistance, ...)
                     | TerrainRenderer2::getDrawFlag(partial, min, max, settings)
```

`getDrawFlag` 把 LOD 决策结果转换为渲染动作：

```cpp
// terrain_renderer2.hpp:143 (声明)
static uint8 getDrawFlag( uint8 partialResult,
                          float minDistance, float maxDistance,
                          const TerrainSettingsPtr pSettings );
```

返回 `RTM_DrawBlend` / `RTM_DrawLOD` / `RTM_DrawLODNormals` 等。

### 9.2 纹理 LOD 选择（getLoadFlag）

```
if minDistance <= absoluteBlendPreloadDistance():
    flag = RTM_PreLoadBlend | RTM_PreloadNormals
elif minDistance <= absoluteNormalPreloadDistance():
    flag = RTM_PreloadNormals
else:
    flag = RTM_DrawLODNormals
```

### 9.3 几何形态变换（geomorphing）

shader 里 `vertex.position.y = lerp(currentLOD.height, nextLOD.height, morphFactor)`，`morphFactor` 由 `morphRanges` 决定：

- `morphRanges.main_.x` = 形变开始（距离 LOD 边界）
- `morphRanges.main_.y` = 形变结束（应该恰好等于下一 LOD）
- `morphFactor = clamp((distance - morphRanges.x) / (morphRanges.y - morphRanges.x), 0, 1)`

顶点 buffer 里同时存了两个高度（current 和 next LOD），shader 根据距离插值——这就是 `TerrainVertexBuffer::generate` 生成 `vector<Vector2>` 的原因（每顶点两高度）。

### 9.4 边界 stitch（退化三角形）

邻居 A 用 LOD 0（128x128）、邻居 B 用 LOD 1（64x64）时，B 的一个顶点对应 A 的两个顶点，直接拼会有 T-junction。`TerrainIndexBuffer::generateDegenerates` 在 B 的边界添加共线三角形（退化三角形），让 GPU 顶点着色器跑过这些"零面积"三角形，但光栅化阶段不画像素——只填补 topology。

`NeighbourMasks` 是 5 元素 `StaticArray<uint8, 5>`：[0] = 主块，[1..4] = 4 个子块。每个 byte 4 bit 分别代表 +X/-X/+Z/-Z 方向是否需要 stitch。

---

## 10. 与 chunk 模块的集成

### 10.1 ChunkTerrain

[chunk_terrain.hpp](file:///workspace/src/lib/chunk/chunk_terrain.hpp) 把 `BaseTerrainBlock` 包装成一个 `ChunkItem`：

```cpp
// chunk_terrain.hpp:40
class ChunkTerrain : public ChunkItem
{
    DECLARE_CHUNK_ITEM( ChunkTerrain )
public:
    virtual void toss( Chunk * pChunk );       // 加入/移出 chunk
    virtual void draw();
    virtual uint32 typeFlags() const;

    Terrain::BaseTerrainBlockPtr block()        { return block_; }
    const Terrain::BaseTerrainBlockPtr block() const { return block_; }
    const BoundingBox & bb() const              { return bb_; }
    void calculateBB();

    static bool outsideChunkIDToGrid( const std::string& chunkID,
                                        int32& x, int32& z );
private:
    Terrain::BaseTerrainBlockPtr    block_;
    BoundingBox                     bb_;
#if UMBRA_ENABLE
    Terrain::BaseRenderTerrainBlock::UMBRAMesh umbraMesh_;
    bool                                        umbraHasHoles_;
    UmbraModelProxyPtr                          pUmbraWriteModel_;
#endif
#if FMOD_SUPPORT
    SoundOccluder                 soundOccluder_;
#endif
};
```

`draw()` 的实现典型流程（参见 [chunk_terrain.cpp](file:///workspace/src/lib/chunk/chunk_terrain.cpp)）：

```cpp
void ChunkTerrain::draw()
{
    if (block_) {
        Matrix world = pChunk_->transform();
        Terrain::BaseTerrainRenderer::instance()->addBlock(
            dynamic_cast<Terrain::BaseRenderTerrainBlock*>(block_.get()),
            world);
    }
}
```

注意 `addBlock` 只是入队，实际渲染发生在 `ChunkManager::draw` 末尾调用 `BaseTerrainRenderer::drawAll()` 时——这样所有可见 block 可以一次性按材质 / 距离排序，减少 state 切换。

### 10.2 ChunkTerrainCache

[chunk_terrain.hpp](file:///workspace/src/lib/chunk/chunk_terrain.hpp):111 中的 `ChunkTerrainCache` 是 chunk 级别的缓存（每个 chunk 一个），用于快速访问该 chunk 的地形，并添加碰撞 obstacle：

```cpp
class ChunkTerrainCache : public ChunkCache
{
public:
    virtual int focus();     // chunk 被聚焦时调用，触发流式加载
    void pTerrain( ChunkTerrain * pT );
    ChunkTerrain * pTerrain()       { return pTerrain_; }
    static Instance<ChunkTerrainCache> instance;
private:
    Chunk * pChunk_;
    ChunkTerrain * pTerrain_;
    SmartPointer<ChunkTerrainObstacle> pObstacle_;   // 碰撞 obstacle
};
```

### 10.3 TerrainFinder 的注入

`BaseTerrainBlock::setTerrainFinder()` 由 chunk 模块在初始化时调用，传入一个 chunk 的实现（`ChunkManager` 或其内部 helper）。这样 `BaseTerrainBlock::findOutsideBlock(pos)` / `BaseTerrainBlock::getHeight(x,z)` 才能跨 chunk 找到对应的地形块。

---

## 11. 渲染管线位置

terrain 处于 [moo](file:///workspace/src/lib/moo/) 的 RenderContext 提交链路中，具体位置：

```
一帧渲染管线：

  EnviroMinder::draw(romp)
       │
       │  设置雾、天空盒、太阳方向 → RenderContext
       ▼
  ChunkManager::draw
       │
       ├─ 遍历可见 chunk
       │   ├─ ChunkTerrain::draw  →  BaseTerrainRenderer::addBlock
       │   ├─ ChunkModel::draw     →  Moo::VisualChannel
       │   ├─ ChunkLight::draw     →  光源收集
       │   └─ ...
       │
       ▼
  BaseTerrainRenderer::drawAll()
       │
       │  TerrainRenderer2::drawAll
       │   ├─ 遍历 blocks_
       │   ├─ 每个 block 调用 drawSingle
       │   │   ├─ setTransformConstants (viewProj / world / cameraPos)
       │   │   ├─ setNormalMapConstants (pNormalMap_ + size)
       │   │   ├─ setHorizonMapConstants (pHorizonMap + size)
       │   │   ├─ setHolesMapConstants (pHolessMap + size + holesSize)
       │   │   ├─ setTextureLayerConstants (4 layer textures + u/v proj + blend texture + layerMask)
       │   │   └─ setLodTextureConstants (lodTextureStart + lodTextureDistance)
       │   └─ drawTerrainMaterial → VertexLodEntry::draw → Moo::EffectMaterial::draw
       │
       ▼
  Moo::RenderContext 提交 → D3D9 device
```

### 11.1 EffectMaterial 与 RenderContext 的交互

每个 `BaseMaterial` 持 `Moo::EffectMaterialPtr pEffect_`，在 `TerrainRenderer2::initInternal` 里从 fx 文件加载：

```cpp
// terrain_renderer2.cpp:261
texturedMaterial_.pEffect_ = new Moo::EffectMaterial;
texturedMaterial_.pEffect_->initFromEffect( texturedEffect );
texturedMaterial_.getHandles();
```

`getHandles()` 缓存 `GetParameterByName` 的 `D3DXHANDLE`，避免每帧字符串查找。绘制时通过 `pMaterial->SetParam(handle, value)` 设值，最终 `EffectMaterial::draw` 调用 `ID3DXEffect::Begin/BeginPass/SetTexture/SetFloat/EndPass/End`。

`SunAngle` 与 `PenumbraSize` 通过 `Moo::EffectConstantValue::get("SunAngle")` 注册成全局常量 setter，每帧 shader 查询该常量时调用 setter 重新计算（见 [terrain_renderer2.cpp](file:///workspace/src/lib/terrain/terrain2/terrain_renderer2.cpp):140 的 `SunAngleConstantSetter`）。

### 11.2 VertexDeclaration

```cpp
// terrain_renderer2.cpp:273
pDecl_ = Moo::VertexDeclaration::get( "terrain" );
```

`"terrain"` 顶点声明定义在 fx 文件或 `vertex_formats.hpp`，典型包含：position、blendindices、texcoord。terrain2 不存 position.x/z（因为是规则的网格），只存 y（高度），shader 用 `gridUV × blockSize + blockOffset` 反推 x/z。

---

## 12. unit_test 目录

unit_test 目录包含两组测试：

### 12.1 test_resource.cpp

[test_resource.cpp](file:///workspace/src/lib/terrain/unit_test/test_resource.cpp) 测试 `Resource<>` 模板：

- `DummyTextObject` 模拟一个加载对象（`doLoad()` 里 `mySleep(100)` 模拟 IO）
- `DummyTextResource` 在 `evaluate(threshold > 10)` 时标记为 required
- 测试用例覆盖：
  - `Resource_testConstruction`：刚构造时 `getObject()==NULL`、`getState()==RS_Unloaded`、`isRequired()==false`
  - `Resource_testEvaluate`：阈值在 10 上下来回切换时 `required_` 翻转
  - `Resource_testStream`：`evaluate(true)` + `stream()` 后状态从 `RS_Unloaded` → `RS_Loading` → `RS_Loaded`，且 `getObject()` 非空
  - 异步版本通过 `BgTaskManager::instance().startThreads(1)` 起后台线程

### 12.2 test_vertex_lod_manager.cpp

[test_vertex_lod_manager.cpp](file:///workspace/src/lib/terrain/unit_test/test_vertex_lod_manager.cpp) 测试 `VertexLodManager`：

- `DummyTerrainBlock2`：继承 `TerrainBlock2`，跳过实际加载只设一个 129x129 的高度图
- `SlowVertexLodManager`：覆盖 `generate()` 加 `mySleep(100)` 模拟异步加载延迟
- 测试用例：
  - `VertexLodManager_testConstruction`：构造后 object 已存在、状态为 `RS_Loaded`（构造时同步预创建空数组）
  - `VertexLodManager_testWorkingSet`：`WorkingSet::IsWithin` / `IsOverlapping` 集合关系
  - `VertexLodManager_testEvaluate`：`evaluate(lod, topLod)` 后 `requestedWorkingSet_` 正确
  - `VertexLodManager_testStream`：异步流式后 LOD 数组按 WorkingSet 加载完成

测试框架用 CppUnitLite2（`TEST_F` 宏），`Fixture` 构造时 `BgTaskManager::startThreads(1)`，析构时 `stopAll(false, false)` 等所有任务结束。

---

## 13. 关键流程总结

### 13.1 加载流程时序

```
ChunkTerrain::load (chunk 模块)
    │
    ▼
BaseTerrainBlock::loadBlock(filename, worldXform, cameraPos, pSettings)
    │
    ├─ terrainVersion(resource) → 100 / 200
    │
    ├─ new TERRAINBLOCK2(pSettings) → TerrainBlock2
    │
    ▼
TerrainBlock2::load(filename, worldXform, cameraPos, error)
    │
    ├─ CommonTerrainBlock2::internalLoad(...)
    │   ├─ 打开 .cdata
    │   ├─ 读 heightMap header → 创建 TerrainHeightMap2（默认 LOD）
    │   ├─ 读 holesHeader    → 创建 TerrainHoleMap2
    │   └─ 读 dominantTextureMapHeader → 创建 DominantTextureMap2
    │
    ├─ initVerticesResource(error)
    │   └─ new VertexLodManager(*this, numLods)
    │       └─ load() 同步加载 VertexLodArray
    │
    ├─ pBlendsResource_ = new TerrainBlendsResource(*this)
    │   └─ evaluate → required_=RR_No（远距离不加载）
    │
    ├─ pDetailHeightMapResource_ = new HeightMapResource(...)
    │   └─ evaluate → 远距离不加载 detail
    │
    ├─ pNormalMap_    = new TerrainNormalMap2
    │   └─ init(terrainResource) → 加载 lodNormalMap_
    │
    ├─ pHorizonMap_   = new HorizonShadowMap2(*this)
    └─ pLodMap_       = new TerrainLodMap2
```

### 13.2 渲染流程时序

```
ChunkManager::draw
    │
    ▼
ChunkTerrain::draw
    │
    ▼
BaseTerrainRenderer::instance()->addBlock(block, worldXform)
    │
    │  ...其他 chunk item...
    │
    ▼
BaseTerrainRenderer::drawAll()
    │
    ▼
TerrainRenderer2::drawAll
    │
    ├─ 遍历 blocks_
    │   │
    │   ▼
    │   drawSingleInternal(block, transform, false, NULL, useCachedLighting)
    │       │
    │       ├─ tb2->preDraw(useCachedLighting, true)
    │       │   ├─ 从 VertexLodManager 取 currentLodPtr / nextLodPtr
    │       │   ├─ 从 TerrainBlendsResource 取 blendsPtr
    │       │   ├─ 缓存到 currentDrawState_
    │       │   └─ 失败 → 返回 false，画 magenta placeholder
    │       │
    │       ├─ setVertexDeclaration(pDecl_)
    │       │
    │       ├─ 取 currentDrawState_.blendsPtr_
    │       │
    │       ├─ 若有 holes 且 depthPassMark_ 过期 → drawTerrainMaterial(zPassMaterial_, DM_Z_ONLY)
    │       │
    │       ├─ 若 renderTextureMask_ & RTM_DrawBlend
    │       │   └─ drawTerrainMaterial(texturedMaterial_, DM_BLEND)
    │       │       ├─ setTransformConstants
    │       │       ├─ setNormalMapConstants
    │       │       ├─ setHorizonMapConstants
    │       │       ├─ setHolesMapConstants
    │       │       ├─ 遍历 combinedLayers_ (每 4 层一 pass)
    │       │       │   ├─ setTextureLayerConstants(pBlend, material, layer, blended)
    │       │       │   └─ VertexLodEntry::draw(material, morphRanges, neighbourMasks, subBlockMask)
    │       │       │       └─ TerrainIndexBuffer::draw → EffectMaterial::draw → D3D draw
    │       │       └─ updateConstants (SunAngle / PenumbraSize)
    │       │
    │       └─ 若 renderTextureMask_ & RTM_DrawLOD
    │           └─ drawTerrainMaterial(lodTextureMaterial_, DM_SINGLE_LOD)
    │               └─ 用 pLodMap_ 一张纹理搞定
```

### 13.3 流式 tick 时序

```
BasicTerrainLodController (在 chunk tick 中调用)
    │
    ▼
setCameraPosition(cameraPos)
    │
    ▼
遍历 blocks_
    │
    ▼
TerrainBlock2::evaluate(relativeCameraPos)
    │
    ├─ TerrainVertexLod::minMaxXZDistance(...)
    │   → distanceInfo_.minDistanceToCamera_, maxDistanceToCamera_
    │
    ├─ vertexLod.calculateLodLevel(minDistance) → currentVertexLod_
    │   vertexLod.calculateLodLevel(maxDistance) → nextVertexLod_
    │
    ├─ vertexLod.calculateMasks(distanceInfo_, lodRenderInfo_)
    │   → lodRenderInfo_.morphRanges_, neighbourMasks_, subBlockMask_
    │
    ├─ TerrainRenderer2::getLoadFlag(minDistance, ...) → renderTextureMask_ 的高位
    ├─ TerrainRenderer2::getDrawFlag(partial, min, max, settings) → renderTextureMask_ 的低位
    │
    ├─ pVerticesResource_->evaluate(currentVertexLod_, topLod)
    │   → 设置 requestedWorkingSet_
    ├─ pBlendsResource_->evaluate(renderTextureMask_) → required_
    ├─ pDetailHeightMapResource_->evaluate(gridSize, stdSize, detailDist, blockDist, topLod)
    │
    ▼
TerrainBlock2::stream()
    │
    ├─ pVerticesResource_->stream()  (异步)
    │   ├─ Resource::stream() 检查 required_ vs state
    │   ├─ RS_Unloaded + required → startAsyncTask → BgTaskManager::addBackgroundTask
    │   ├─ 后台线程 doBackgroundTask → load()
    │   └─ 主线程 doMainThreadTask → postAsyncLoad() (上传 D3D 资源)
    │
    ├─ pBlendsResource_->stream()    (异步)
    ├─ pDetailHeightMapResource_->stream()
    └─ pNormalMap_->stream()         (异步，调 TerrainNormalMap2::NormalMapTask)
```

---

## 14. 性能与调试

### 14.1 Watcher 监控项

`TerrainRenderer2::initInternal` 注册的 Watcher（[terrain_renderer2.cpp](file:///workspace/src/lib/terrain/terrain2/terrain_renderer2.cpp):293）：

| Watcher 路径 | 含义 |
|--------------|------|
| `Render/Terrain/Terrain2/RenderedBlocks` | 当前帧渲染的 block 数 |
| `Render/Terrain/Terrain2/VertexLODSizeMB` | 所有 VertexLodEntry 总显存占用 |
| `Render/Terrain/Terrain2/specular power/multiplier/fresnelConstant/fresnelExp` | 实时可调的 specular 参数 |
| `Render/Performance/Enable Terrain Draw` | 全局开关 terrain 绘制（`g_drawTerrain`） |
| `Render/Terrain/Terrain2/Draw Heightmap` | 用 LineHelper 画高度线框 |
| `Render/Terrain/Terrain2/Draw Placeholders` | 画 magenta placeholder（加载失败的 block） |
| `Render/Terrain/Terrain2/QuadtreeSize` | 四叉树总内存占用 |

### 14.2 Profiler

```cpp
// terrain_renderer2.cpp:37
PROFILER_DECLARE( TerrainRenderer2_updateConstants, "TerrainRenderer2 UpdateConstants" );
PROFILER_DECLARE( TerrainRenderer2_drawTerrainMaterial, "TerrainRenderer2 DrawTerrainMaterial" );
```

### 14.3 内存追踪

`VertexLodEntry::s_totalMB_` 累加所有 LOD 顶点 buffer 占用，通过 `WATCH_TERRAIN_VERTEX_LODS` 宏开关（见 [vertex_lod_entry.hpp](file:///workspace/src/lib/terrain/terrain2/vertex_lod_entry.hpp):24）。

`TerrainQuadTreeCell` 通过 `ENABLE_RESOURCE_COUNTERS` 宏追踪四叉树内存，见 [terrain_quad_tree_cell.cpp](file:///workspace/src/lib/terrain/terrain2/terrain_quad_tree_cell.cpp):268。

---

## 15. 与其它模块的协作

| 模块 | 协作点 |
|------|--------|
| [moo](file:///workspace/src/lib/moo/) | 所有 GPU 资源（`VertexBuffer`/`IndexBuffer`/`BaseTexture`/`EffectMaterial`/`RenderTarget`/`VertexDeclaration`）通过 moo 创建；`RenderContext` 提供 camera / view / projection / light / fog |
| [resmgr](file:///workspace/src/lib/resmgr/) | `.cdata` 通过 `BWResource::openSection` 加载；`AutoConfigString g_terrain2Settings("system/terrain2")` 加载默认配置 |
| [chunk](file:///workspace/src/lib/chunk/) | `ChunkTerrain` 把 block 嵌入 chunk；`ChunkTerrainCache` 缓存 per-chunk；`ChunkManager` 实现 `TerrainFinder` |
| [romp](file:///workspace/src/lib/romp/) | `EnviroMinder` 设置 RenderContext 的雾、太阳方向；`LineHelper` 用于 placeholder 绘制 |
| [physics2](file:///workspace/src/lib/physics2/) | `MaterialKinds` 由 `physics2/material_kinds.hpp` 提供，`DominantTextureMap::materialKind()` 返回 `MaterialKind`，用于足音、脚步粒子 |
| [cstdmf](file:///workspace/src/lib/cstdmf/) | `BgTaskManager` 异步加载；`SimpleMutex` 保护缓存；`SmartPointer`/`SafeReferenceCount` 引用计数；`InitSingleton` 单例基类；`Watcher` 调试；`Profiler` 性能分析 |
| [model](file:///workspace/src/lib/model/) | 间接：模型在 terrain 上行走时通过 `BaseTerrainBlock::getHeight` 取高度 |
| [speedtree] | 间接：植被在 terrain 上放置时取高度 / 法线 |

---

## 16. 小结

terrain 库是 BigWorld"大世界"愿景的核心组件之一，其设计有几个亮点：

1. **抽象分层清晰**：`BaseTerrainBlock` / `BaseRenderTerrainBlock` / `EditorBaseTerrainBlock` 三层抽象让服务端 / 客户端 / 编辑器共享同一份核心代码，差异通过 `#ifdef MF_SERVER / EDITOR_ENABLED` 在编译期裁剪。
2. **v1/v2 共存**：`terrainVersion()` 工厂方法让两个版本的地形数据可以共存于同一 space，便于逐步升级内容。
3. **流式基础设施通用**：`Resource<>` 模板被高度图、顶点 LOD、blends、法线图复用，统一的 `evaluate / stream / load / unload` 状态机简化了所有可流式资源的生命周期管理。
4. **GPU 资源全局共享**：`GridVertexBuffer` 和 `TerrainIndexBuffer` 按 `(resolution, quadCount)` 全局缓存，所有同规格的 block 共用一份 VB/IB，大幅减少显存。
5. **Geomipmapping + Geomorphing**：通过 `AliasedHeightMap` 在 CPU 端生成多 LOD 顶点（双高度），shader 端按距离插值，避免视觉跳变。
6. **四叉树碰撞裁剪**：`TerrainQuadTreeCell` 仅用于碰撞查询（不是渲染 LOD），按 outcode 快速剔除不相交的子区域，远比线性扫描高效。
7. **可调试性强**：大量 Watcher、Profiler、内存计数器，可以运行时观察 LOD 决策、显存占用、block 数量。

v1/v2 共存的代价是抽象层有些重复，且 v1 的 `TerrainBlock1` 直接持 VB/IB 而不走 streaming，未来若 v1 完全废弃可以删除大量代码。`Resource<>` 模板的状态机虽然清晰，但 `preAsyncLoad / postAsyncLoad` 钩子的使用比较隐晦，需要仔细阅读派生类才能理解整个加载时序。
