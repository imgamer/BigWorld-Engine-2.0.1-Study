# 15. 环境与氛围系统（romp）

> BigWorld Engine 2.0.1 源码学习文档
> 覆盖目录：`src/lib/romp/`
> romp = Render-Oriented Miscellaneous Pieces，BigWorld 客户端的“环境/氛围/控制台/字体/调试可视化”杂烩库
> 关键词：EnviroMinder、TimeOfDay、SkyGradientDome、Weather、FogController、Flora、Water、LensEffectManager、XConsole、Font

---

## 0. 概览

`romp` 是 BigWorld 客户端最大的“杂项渲染库”。它不专注于某一类对象（如模型、粒子、地形），而是把“让世界看起来有氛围”的所有零散子系统收拢在一起：

- **环境管理**：`EnviroMinder` 是中心，聚合天空、时间、天气、雾、植被、水、海等子系统；
- **时间与天空**：`TimeOfDay`、`SunAndMoon`、`SkyGradientDome`、`StarDome`、`SkyDomeOccluder`、`SkyDomeShadows`、`SkyLightMap`；
- **天气**：`Weather`、`Rain`、`Lightning`；
- **雾**：`FogController`（全局单例）；
- **光照与光效**：`LightMap`/`EffectLightMap`、`LensEffect`/`LensEffectManager`、`FlashBangEffect`、`EnvironmentCubeMap`、`PhotonOccluder`、`PyChunkLight`/`PyChunkSpotLight`；
- **植被（Flora）**：`Flora`、`FloraBlock`、`FloraRenderer`、`FloraLightMap`、`FloraTexture`、`FloraVertexType`、`Ecotype`、`EcotypeGenerator`；
- **水**：`Water`、`WaterCell`、`Waters`、`WaterSimulation`、`WaterSplash`、`Sea`、`WaterSceneRenderer`、`PyWaterVolume`；
- **地形 occluder**：`TerrainOccluder`、`ZBufferOccluder`、`ZAttenuationOccluder`；
- **控制台与字体**：`XConsole`、`ConsoleManager`、`Font`、`FontManager`、`FontMetrics`、`GlyphCache`、`GlyphReferenceHolder`；
- **UI/工具**：`Progress`/`GUIProgress`/`SuperModelProgress`、`AnimationGrid`、`CustomMesh`、`DebugGeometry`、`LineHelper`、`LineEditor`、`Geometrics`、`HandlePool`、`OcclusionQueryHelper`、`HistogramProvider`；
- **杂项**：`Dapple`、`Distortion`、`FrameLogger`、`EngineStatistics`、`TextureFeeds`、`Watermark`、`LODSettings` 及一批 `Py*` 绑定。

### 0.1 模块依赖图

```
                          ┌─────────────────────────────┐
                          │       客户端主循环           │
                          │  (App::tick / draw 流程)     │
                          └──────────────┬──────────────┘
                                         │ 持有/调用
                                         ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │                         EnviroMinder                              │
   │  (每个 ChunkSpace 一个，聚合以下子系统)                          │
   │                                                                   │
   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│
   │  │TimeOfDay │ │ Weather  │ │SkyGradi- │ │SunAndMoon│ │  Rain   ││
   │  │          │ │          │ │entDome   │ │          │ │         ││
   │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬────┘│
   │       │            │            │            │            │      │
   │  ┌────▼───┐ ┌──────▼─────┐ ┌───▼────┐ ┌─────▼────┐ ┌────▼────┐ │
   │  │ Flora  │ │SkyLightMap │ │  Sea   │ │Environment│ │ SkyDome │ │
   │  │        │ │(cloud shadow)│ (Waters)│ │CubeMap   │ │Occluder │ │
   │  └────────┘ └────────────┘ └────────┘ └──────────┘ └─────────┘ │
   └──────────────────────────────┬───────────────────────────────────┘
                                  │ 设置全局状态
                                  ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  FogController (单例)   LensEffectManager (单例)   Waters (单例)  │
   │  TextureFeeds (单例)    Distortion (单例)          ConsoleManager │
   └──────────────────────────────┬───────────────────────────────────┘
                                  │ 共同依赖
                                  ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │                          moo / chunk / terrain                   │
   │  RenderContext / EffectMaterial / RenderTarget / Visual /        │
   │  CubeRenderTarget / ChunkSpace / BaseTerrainBlock                │
   └──────────────────────────────────────────────────────────────────┘
```

---

## 1. 模块定位

`romp` 的定位由其名“Render-Oriented Miscellaneous Pieces”决定：它是**所有不便归入其它库的渲染相关功能的集散地**。与 `model`（模型）、`particle`（粒子）、`terrain`（地形）、`chunk`（分块场景）平行，但内容更杂：

| 子系统 | 职责 | 是否单例 |
|---|---|---|
| EnviroMinder | 每 space 一个环境管理器，聚合天空/时间/天气/雾/植被/水 | 否（per-space） |
| FogController | 全局雾状态 + 雾发射器 | 是 |
| LensEffectManager | 镜头光晕的注册/衰减/绘制 + 光线遮挡查询 | 是 |
| Waters | 所有水面 `Water` 的集合 + 共享资源 | 是 |
| TextureFeeds | 命名纹理流（脚本可注入动态纹理） | 是 |
| Distortion | 屏幕热扭曲/水扭曲后处理 | 是 |
| ConsoleManager | 控制台栈管理 + 输入分发 | 是 |
| Flora | 每 space 一个植被系统 | 否（per-space，由 EnviroMinder 持有） |

---

## 2. EnviroMinder —— 环境管理器（romp 的中心）

声明见 [enviro_minder.hpp](file:///workspace/src/lib/romp/enviro_minder.hpp)，实现见 [enviro_minder.cpp](file:///workspace/src/lib/romp/enviro_minder.cpp)。它是每个 `ChunkSpace` 的环境大脑，把所有环境子系统聚合成一个对象。

### 2.1 类声明要点

```cpp
// enviro_minder.hpp:72
class EnviroMinder
{
public:
    EnviroMinder(ChunkSpaceID spaceID);
    ~EnviroMinder();

    static void init();
    static void fini();

    struct DrawSelection
    {
        enum
        {
            // bit values for drawSkyCtrl are sorted in drawing order
            sunAndMoon  = 0x0001,
            skyGradient = 0x0002,
            skyBoxes    = 0x0004,
            sunFlare    = 0x0008,
            all         = 0xffff
        };
        uint32 value;
        DrawSelection() : value( all ) {}
    };

    bool load( DataSectionPtr pDS, bool loadFromExternal = true );
    void tick( float dTime, bool outside,
        const WeatherSettings * pWeatherOverride = NULL );
    void allowUpdate(bool val) { allowUpdate_ = val; }
    void activate();
    void deactivate();

    float farPlane() const;
    void setFarPlane(float farPlane);

    void drawHind( float dTime, DrawSelection drawWhat = DrawSelection(), bool showWeather = true );
    void drawHindDelayed( float dTime, DrawSelection drawWhat = DrawSelection() );
    void drawFore( float dTime, bool showWeather = true,
                                bool showFlora = true,
                                bool showFloraShadowing = false,
                                bool drawOverLays = true,
                                bool drawObjects = true );
    void tickSkyDomes( float dTime );
    void drawSkyDomes( bool isOcclusionPass = false );
    void drawSkySunCloudsMoon( float dTime, DrawSelection drawWhat );

    // 子系统访问器
    TimeOfDay *         timeOfDay()         { return timeOfDay_; }
    Weather *           weather()           { return weather_; }
    SkyGradientDome *   skyGradientDome()   { return skyGradientDome_; }
    SunAndMoon *        sunAndMoon()        { return sunAndMoon_; }
    Seas *              seas()              { return seas_; }
    Rain *              rain()              { return rain_; }
    SkyLightMap *       skyLightMap()       { return skyLightMap_; }
    Flora *             flora()             { return flora_; }
    Decal *             decal()             { return decal_; }
    EnvironmentCubeMap *environmentCubeMap(){ return environmentCubeMap_; }

    // Vector4Provider 控制器（脚本可注入动态控制）
    std::vector<Vector4ProviderPtr> skyDomeControllers_;
    Vector4ProviderPtr weatherControl_;
    Vector4ProviderPtr sunlightControl_;
    Vector4ProviderPtr ambientControl_;
    Vector4ProviderPtr fogControl_;

private:
    void decideLightingAndFog();
    TimeOfDay          * timeOfDay_;
    Weather            * weather_;
    SkyGradientDome    * skyGradientDome_;
    SunAndMoon         * sunAndMoon_;
    Seas               * seas_;
    Rain               * rain_;
    SkyDomeShadows     * skyDomeShadows_;
    SkyDomeOccluder    * skyDomeOccluder_;
    ZBufferOccluder    * zBufferOccluder_;
    ChunkObstacleOccluder * chunkObstacleOccluder_;
    SkyLightMap        * skyLightMap_;
    Flora              * flora_;
    Decal              * decal_;
    EnvironmentCubeMap * environmentCubeMap_;
    std::vector<Moo::VisualPtr> skyDomes_;
    // ...
    static EnviroMinder* s_activatedEM_;   // 同一时刻只有一个激活
};
```

### 2.2 关键设计点

1. **per-space 实例 + 全局激活互斥**：`s_activatedEM_` 保证同一时刻只有一个 EnviroMinder 处于激活态（相机所在 space 的那个）。`activate()` 中有 `MF_ASSERT( !s_activatedEM_ )`。
2. **Vector4Provider 控制器**：`weatherControl_` / `sunlightControl_` / `ambientControl_` / `fogControl_` 都是 `Vector4ProviderPtr`，允许脚本通过动态 provider 实时调制天气/光照/雾——这是 BigWorld “脚本驱动渲染”的典型手法。
3. **DrawSelection 位掩码**：`drawHind`/`drawSkySunCloudsMoon` 用位掩码选择画哪些天空元素（太阳月亮/天空渐变/天空盒/太阳光晕），且位序即绘制顺序。
4. ** Hind / Fore 分离**：`drawHind` 画背景（天空、雾、环境立方体贴图），`drawFore` 画前景（雨、海、贴花、植被、闪电）。中间穿插 `ChunkManager::draw` 画场景物体。

### 2.3 `SkyBoxScopedSetup` —— 天空盒渲染作用域

[enviro_minder.hpp:286](file:///workspace/src/lib/romp/enviro_minder.hpp) 定义了一个 RAII 类，在画天空盒时临时改视口与远近平面：

```cpp
// enviro_minder.hpp:286
class SkyBoxScopedSetup
{
public:
    SkyBoxScopedSetup();
    ~SkyBoxScopedSetup();
private:
    D3DVIEWPORT9 oldVp_;
    Moo::CameraPlanesSetter cps_;
};
```

实现把视口 `MinZ/MaxZ` 推到远端（`MinZ=1.0, MaxZ=2.0`），并用 `CameraPlanesSetter` 设远平面到 2000，确保天空盒不被场景几何裁掉：

```cpp
// enviro_minder.cpp:108
SkyBoxScopedSetup::SkyBoxScopedSetup():
    cps_( 1.f, 2000.f )
{
    Moo::rc().getViewport( &oldVp_ );
    D3DVIEWPORT9 vp = oldVp_;
    vp.MinZ = 1.0f;
    vp.MaxZ = 2.0f;
    Moo::rc().setViewport( &vp );
    FogController::instance().commitFogToDevice();
}
```

### 2.4 `PlayerAttachments` —— 玩家挂接粒子

EnviroMinder 还持有 `PlayerAttachments`，用于把粒子系统（如角色身上的雨滴溅落）挂到玩家模型节点：

```cpp
// enviro_minder.hpp:51
struct PlayerAttachment
{
    PyMetaParticleSystem* pSystem;
    std::string onNode;
};
class PlayerAttachments : public std::vector<PlayerAttachment>
{
public:
    void add( PyMetaParticleSystem* pSys, const std::string & node );
};
```

---

## 3. EnviroMinder 在主循环中的位置

### 3.1 一帧的典型调用流

EnviroMinder 把一帧的渲染切成 `tick → drawHind → (场景物体) → drawHindDelayed → drawFore`。其中天空可前可后（取决于显卡是否支持 shader）。

```
客户端主循环            EnviroMinder              ChunkManager          各子系统
     │                      │                         │                    │
     │ tick(dTime,outside)  │                         │                    │
     ├─────────────────────>│ weather.tick            │                    │
     │                      │ timeOfDay.tick          │                    │
     │                      │ skyGradientDome.update  │                    │
     │                      │ decideLightingAndFog    │                    │
     │                      │ rain.tick               │                    │
     │                      │ tickSkyDomes            │                    │
     │                      │ flora.update            │                    │
     │                      │                         │                    │
     │ drawHind(dTime)      │                         │                    │
     ├─────────────────────>│ skyGradientDome.addFogEmitter              │
     │                      │ rain.addFogEmitter                         │
     │                      │ FogController.tick / commitFogToDevice     │
     │                      │ skyLightMap.update                         │
     │                      │ environmentCubeMap.update                 │
     │                      │ (老显卡) drawSkySunCloudsMoon             │
     │                      │                         │                    │
     │ (场景物体绘制)        │                         │                    │
     ├───────────────────────────────────────────────>│ chunk 物体绘制     │
     │                      │                         │ (模型/粒子/水面)   │
     │                      │                         │                    │
     │ drawHindDelayed      │                         │                    │
     ├─────────────────────>│ (新显卡) drawSkySunCloudsMoon             │
     │                      │                         │                    │
     │ drawFore(dTime)      │                         │                    │
     ├─────────────────────>│ rain.amount 调整        │                    │
     │                      │ seas.draw               │                    │
     │                      │ decal.draw / footPrint  │                    │
     │                      │ flora.draw              │                    │
     │                      │ (rain overlay / lightning)                 │
     │                      │                         │                    │
     │ LensEffectManager.draw (镜头光晕，含遮挡查询)  │                    │
     │ PostProcessing (后处理)                         │                    │
     │ Present                                          │                    │
```

### 3.2 `tick` 实现

[enviro_minder.cpp:521](file:///workspace/src/lib/romp/enviro_minder.cpp)：

```cpp
// enviro_minder.cpp:521
void EnviroMinder::tick( float dTime, bool outside,
    const WeatherSettings * pWeatherOverride )
{
    s_skyBoxController->value( Vector4( 0,0,0,1 ) );
    s_windAnimation->tick( dTime, weather_->wind() );

    // 把风传给粒子系统
    ParticleSystemManager::instance().windVelocity( Vector3(
        weather_->wind().x, 0, weather_->wind().y ) );

    weather_->tick( dTime );

    // 计算太阳方向/颜色，供云/雨/雪使用
    Vector3 sunDir =
        timeOfDay_->lighting().sunTransform.applyToUnitAxisVector(2);
    sunDir.x = -sunDir.x; sunDir.z = -sunDir.z;
    sunDir.normalise();
    uint32 sunCol = Colour::getUint32( timeOfDay_->lighting().sunColour );
    float sunAngle = 2.f * MATH_PI - DEG_TO_RAD( ( timeOfDay_->gameTime() / 24.f ) * 360 );
    rain_->update( *weather_, outside );

    // 更新太阳/月亮位置与光照颜色
    timeOfDay_->tick( dTime );
    skyGradientDome_->update( timeOfDay_->gameTime() );

    this->decideLightingAndFog();   // 决定光照与雾

    rain_->tick( dTime );
    this->tickSkyDomes( dTime );

    flora_->update( dTime, *this );
}
```

注意第一行就把 `weather_->wind()` 传给了 `ParticleSystemManager`——天气的风直接影响所有粒子。

### 3.3 `drawHind` —— 背景绘制

[enviro_minder.cpp:757](file:///workspace/src/lib/romp/enviro_minder.cpp)：

```cpp
// enviro_minder.cpp:757
void EnviroMinder::drawHind( float dTime, DrawSelection drawWhat, bool showWeather )
{
    // 1) 累加雾发射器（天空、雨）
    skyGradientDome_->addFogEmitter();
    if (showWeather) rain_->addFogEmitter();

    // 2) 更新并提交雾到设备
    FogController::instance().tick();
    FogController::instance().commitFogToDevice();

    // 3) 更新天空光照图（云阴影）
    if (skyLightMap_ && allowUpdate_)
    {
        float sunAngle = 2.f * MATH_PI - DEG_TO_RAD( ( timeOfDay_->gameTime() / 24.f ) * 360 );
        skyLightMap_->update( sunAngle, Vector2::zero() );
    }
    // Geometrics 函数会破坏雾状态，重新提交
    FogController::instance().commitFogToDevice();

    // 4) 更新环境立方体贴图
    environmentCubeMap_->update( dTime, true, 1, drawWhat.value );

    // 5) 老显卡在场景前画天空
    if (EnviroMinder::primitiveVideoCard())
        drawSkySunCloudsMoon( dTime, drawWhat );
}
```

`primitiveVideoCard()` 区分老显卡（不支持 shader）和新显卡：老显卡在场景前画天空（避免 Z 测试问题），新显卡在场景后画天空（省填充率，见 `drawHindDelayed`）。

### 3.4 `drawFore` —— 前景绘制

[enviro_minder.cpp:826](file:///workspace/src/lib/romp/enviro_minder.cpp)：绘制雨、海、贴花、脚印、植被、闪电等场景前/后景元素。其中雨量有“反迟滞”刹车，避免突然起雨/停雨：

```cpp
// enviro_minder.cpp:836
float wantRain = 0.f;
if (weatherControl_ && allowUpdate_) {
    Vector4 value; weatherControl_->output(value);
    wantRain += value[0];
}
float haveRain = rain_->amount();
if (drawObjects) {
    if (allowUpdate_) {
        if (weatherControl_) rain_->amount( wantRain );
        else rain_->amount( haveRain + Math::clamp(dTime*0.03f, wantRain - haveRain) );
    }
    seas_->draw( dTime, timeOfDay_->gameTime() );
    // 贴花/脚印用 ClipPlaneBias 防 Z-fighting
    beginClipPlaneBiasDraw( decalClipPlaneBias );
        decal_->draw();
        footPrintRenderer_->draw();
    endClipPlaneBiasDraw();
    if (showFlora) flora_->draw( dTime, *this );
    // ...
}
```

### 3.5 `decideLightingAndFog` —— 光照雾决策

[enviro_minder.cpp:961](file:///workspace/src/lib/romp/enviro_minder.cpp) 把 `TimeOfDay` 计算出的基础光照/雾色，与脚本注入的 `Vector4Provider` 控制器相乘，得到最终场景光照与雾：

```cpp
// enviro_minder.cpp:961
void EnviroMinder::decideLightingAndFog()
{
    Vector4 control(0,0,1,1), sunlightControl(1,1,1,1),
            ambientControl(1,1,1,1), fogControl(1,1,1,1);
    if (weatherControl_   != NULL) weatherControl_->output( control );
    if (sunlightControl_  != NULL) sunlightControl_->output( sunlightControl );
    if (ambientControl_   != NULL) ambientControl_->output( ambientControl );
    if (fogControl_       != NULL) fogControl_->output( fogControl );

    float dimBy = 1.f;
    Vector4 lightDimmer( dimBy, dimBy, dimBy, 1.f );
    OutsideLighting & outLight = timeOfDay_->lighting();
    outLight.sunColour = outLight.sunColour * lightDimmer * sunlightControl;
    outLight.ambientColour = outLight.ambientColour * lightDimmer * ambientControl;

    // 计算最终雾密度
    float fogDensity = fogControl.w;
    float extraFog = Math::clamp(0.f, fogDensity - 1.f, 1.f);
    Vector3 controlFog = Vector3((float*)(&fogControl.x));
    Vector3 modcol = (extraFog * controlFog) + ( (1.f-extraFog) * Vector3(1,1,1) );
    modcol = modcol * Vector3(255,255,255);

    skyGradientDome_->fogModulation( modcol, fogDensity );
    skyGradientDome_->farMultiplier( fogDensity );
}
```

---

## 4. 时间与天空

### 4.1 `TimeOfDay` —— 时间与基础光照

声明见 [time_of_day.hpp](file:///workspace/src/lib/romp/time_of_day.hpp)。它管理游戏内时间（0~24 小时），并用 `LinearAnimation` 在关键帧间插值出太阳颜色、环境光颜色、雾色。

```cpp
// time_of_day.hpp:26
class OutsideLighting : public Aligned
{
public:
    Matrix  sunTransform;      // 太阳变换（位置/方向）
    Matrix  moonTransform;     // 月亮变换
    Vector4 sunColour;         // 太阳颜色
    Vector4 ambientColour;     // 环境光颜色
    Vector4 fogColour;         // 雾色
    const Matrix& mainLightTransform() const {
        return useMoonColour_ ? moonTransform : sunTransform;
    }
    const Vector3& mainLightDir() const {
        return mainLightTransform().applyToUnitAxisVector(2);
    }
    void useMoonColour( bool u ) { useMoonColour_ = u; }
private:
    bool useMoonColour_;       // 夜晚用月亮作为主光源
};

class TimeOfDay : public Aligned
{
public:
    void tick( float dTime );
    void load( DataSectionPtr root, bool ignoreTimeChanged = false );
    float gameTime() const;          // 游戏时间（小时）
    void  gameTime( float t );
    float secondsPerGameHour() const; // 现实秒/游戏小时
    void  secondsPerGameHour( float t );
    const OutsideLighting & lighting() const;
    // ...
    class UpdateNotifier : public ReferenceCount
    {
    public:
        virtual void updated( const TimeOfDay & tod ) = 0;
    };
    void addUpdateNotifier( SmartPointer<UpdateNotifier> pUN );
private:
    OutsideLighting now_;
    float time_;                  // 游戏时间（小时）
    float startTime_;
    float secondsPerGameHour_;
    float sunAngle_, moonAngle_;
    LinearAnimation< Vector3 > sunAnimation_;     // 时间→太阳色
    LinearAnimation< Vector3 > ambientAnimation_; // 时间→环境光
    LinearAnimation< Vector3 > fogAnimation_;     // 时间→雾色
    std::vector< SmartPointer<UpdateNotifier> > updateNotifiers_;
};
```

#### 4.1.1 关键设计

- **`OutsideLighting`** 是“当前时刻的户外光照快照”，由 `TimeOfDay` 计算后被多处消费（`EnviroMinder`、`SkyGradientDome`、`Rain` 等）。
- **`useMoonColour_`**：夜晚切换主光源为月亮，`mainLightDir()` 自动返回月亮方向。
- **`UpdateNotifier`**：观察者模式，当时间被外部（脚本/编辑器）修改而非 tick 推进时，通知订阅者（如 `SkyGradientDome` 需要重算）。
- **`LinearAnimation<Vector3>`**：三组关键帧动画（太阳色/环境光/雾色），按 `gameTime` 插值。

### 4.2 一天时间循环的内部状态变化

```
gameTime: 0h(午夜) ──────── 6h(日出) ──────── 12h(正午) ──────── 18h(日落) ──────── 24h
   │                                                                                            │
   │  useMoonColour_=true            useMoonColour_→false           useMoonColour_→true          │
   │  (月亮为主光源)                  (太阳为主光源)                 (月亮为主光源)              │
   │                                                                                            │
   │  sunAnimation_:    暗蓝 → 橙红日出 → 亮白正午 → 橙红日落 → 暗蓝                              │
   │  ambientAnimation_: 冷蓝 → 暖橙 → 亮白 → 暖橙 → 冷蓝                                       │
   │  fogAnimation_:    深灰 → 浅橙 → 浅蓝 → 浅橙 → 深灰                                        │
   │                                                                                            │
   │  sunTransform:     太阳在地下 → 升起 → 天顶 → 落下 → 地下                                  │
   │  moonTransform:    月亮在天顶 → 落下 → 地下 → 升起 → 天顶                                  │
   │                                                                                            │
   │  sunAngle (用于 SkyLightMap): 2π - (gameTime/24)*360°                                       │
   │                                                                                            │
   │  SkyGradientDome.update(gameTime): 重算天空渐变纹理 alpha + 雾色                            │
   │  StarDome.draw(gameTime): 白天 alpha=0，夜晚 alpha=1                                       │
   │  Rain/Flora: 读取 lighting().sunColour / sunDir 调整自身                                    │
```

`tick` 推进 `time_`：

```
TimeOfDay::tick(dTime):
  time_ += dTime / secondsPerGameHour_
  if (time_ >= 24.f) time_ -= 24.f   // 循环
  // 用 time_ 在三组 LinearAnimation 上插值，更新 now_.sunColour/ambientColour/fogColour
  // 根据 sunAngle_/moonAngle_ 更新 sunTransform/moonTransform
  // 若太阳低于地平线，设 useMoonColour_=true
```

### 4.3 `SunAndMoon` —— 太阳与月亮

声明见 [sun_and_moon.hpp](file:///workspace/src/lib/romp/sun_and_moon.hpp)。用 `CustomMesh<VertexXYZNDUV>` 画两个公告板，并给太阳挂一个 `LensEffect`（镜头光晕）：

```cpp
// sun_and_moon.hpp:28
class SunAndMoon
{
public:
    void timeOfDay( TimeOfDay* timeOfDay ) { timeOfDay_ = timeOfDay; }
    void create( void );
    void destroy( void );
    void draw();
private:
    void createMoon( void );
    void createSun( void );
    CustomMesh<Moo::VertexXYZNDUV>* moon_;
    CustomMesh<Moo::VertexXYZNDUV>* sun_;
    Moo::EffectMaterialPtr moonMat_, sunMat_;
    TimeOfDay* timeOfDay_;
    LensEffect sunLensEffect_;   // 太阳的镜头光晕
};
```

`draw` 根据 `timeOfDay_->lighting().sunTransform/moonTransform` 定位太阳/月亮，并在太阳可见时注册 `sunLensEffect_` 到 `LensEffectManager`。

### 4.4 `SkyGradientDome` —— 天空渐变穹顶

声明见 [sky_gradient_dome.hpp](file:///workspace/src/lib/romp/sky_gradient_dome.hpp)。用基于物理的 Mie/Rayleigh 散射参数生成天空渐变，是 `Moo::DeviceCallback`（设备丢失/恢复时重建纹理）：

```cpp
// sky_gradient_dome.hpp:30
class SkyGradientDome : public Moo::DeviceCallback
{
public:
    void activate( const class EnviroMinder&, DataSectionPtr pSpaceSettings );
    void load( DataSectionPtr root );
    void fogModulation( const Vector3 & modulateColour, float fogMultiplier );
    void draw( TimeOfDay* timeOfDay );
    void update( float time );
    void addFogEmitter();        // 向 FogController 注册自己为雾源
    void remFogEmitter();
private:
    FogController::Emitter fogEmitter_;   // 自身作为雾发射器
    Moo::EffectMaterialPtr material_;
    Moo::VisualPtr skyDome_;
    // Mie 散射参数
    float mieEffect_, turbidityOffset_, turbidityFactor_;
    float vertexHeightEffect_, sunHeightEffect_, power_, effectiveTurbidity_;
    LinearAnimation< float >   textureAlphas_;
    LinearAnimation< Vector3 > fogAnimation_;
    Moo::BaseTexturePtr pRayleighMap_;     // Rayleigh 散射查找图
    EffectParameterCache parameters_;
};
```

关键点：
- 天空本身是一个**雾发射器**（`fogEmitter_`），天空颜色贡献到全局雾色；
- `fogModulation` 接收 `decideLightingAndFog` 计算的调制色与雾密度；
- `pRayleighMap_` 是预计算的 Rayleigh 散射查找纹理，配合 Mie 参数实时合成天空颜色。

### 4.5 `StarDome` —— 星空穹顶

声明见 [star_dome.hpp](file:///workspace/src/lib/romp/star_dome.hpp)。极简，一个 `Moo::Visual` + 材质，按 `timeOfDay` 调透明度（白天不可见）：

```cpp
// star_dome.hpp:20
class StarDome
{
public:
    void init( void );
    void draw( float timeOfDay );
private:
    Moo::VisualPtr      visual_;
    Moo::Material       mat_;
    Moo::BaseTexturePtr texture_;
};
```

### 4.6 `SkyDomeOccluder` —— 天空盒遮挡查询

声明见 [sky_dome_occluder.hpp](file:///workspace/src/lib/romp/sky_dome_occluder.hpp)。专门为太阳光晕做遮挡判断：构造一个“看向太阳”的视图矩阵，把所有天空遮挡物画到屏幕中央，用 scissor rect 裁剪到太阳大小，再用 D3D occlusion query 数可见像素：

```cpp
// sky_dome_occluder.hpp:41
class SkyDomeOccluder : public PhotonOccluder, public Moo::DeviceCallback
{
public:
    SkyDomeOccluder( class EnviroMinder& enviroMinder );
    virtual float collides(
            const Vector3 & photonSourcePosition,
            const Vector3 & cameraPosition,
            const LensEffect& le );
    virtual void beginOcclusionTests();
    virtual void endOcclusionTests();
private:
    void getLookAtSunViewMatrix( Matrix& out );
    uint32 drawOccluders( float size );
    UINT lastResult_;
    uint32 possiblePixels_;
    OcclusionQueryHelper helper_;
    class EnviroMinder& enviroMinder_;
};
```

注释解释了原理：

> It works by setting the "OcclusionTestEnable" flag... a view matrix is constructed that makes the camera look directly at the sun... A scissors rectangle is used to clip these results to the size of the sun, and directX occlusion queries are used to record how many pixels of sun are visible.

### 4.7 `SkyDomeShadows` —— 天空盒云阴影

声明见 [sky_dome_shadows.hpp](file:///workspace/src/lib/romp/sky_dome_shadows.hpp)。它是 `SkyLightMap::IContributor`，把天空盒用正交投影画进天空光照图，生成云阴影：

```cpp
// sky_dome_shadows.hpp:31
class SkyDomeShadows : public SkyLightMap::IContributor
{
public:
    SkyDomeShadows( class EnviroMinder& enviroMinder );
    bool needsUpdate() { return true; }
    void render(SkyLightMap* lightMap, Moo::EffectMaterialPtr material, float sunAngle);
private:
    class EnviroMinder& enviroMinder_;
};
```

### 4.8 `SkyLightMap` —— 天空光照图

声明见 [sky_light_map.hpp](file:///workspace/src/lib/romp/sky_light_map.hpp)。继承 `EffectLightMap`，是一张“云阴影 + 太阳方向”的 RenderTarget，提供给场景物体着色器采样：

```cpp
// sky_light_map.hpp:30
class SkyLightMap : public EffectLightMap
{
public:
    void activate( const EnviroMinder&, DataSectionPtr pSpaceSettings );
    void update( float sunAngle, const Vector2& strataMovement );
    /// 贡献者接口（如 SkyDomeShadows、Dapple）
    class IContributor
    {
    public:
        virtual bool needsUpdate() = 0;
        virtual void render( SkyLightMap*, Moo::EffectMaterialPtr, float sunAngle ) = 0;
    };
    void addContributor( IContributor& c );
    void delContributor( IContributor& c );
private:
    typedef std::vector<SkyLightMap::IContributor*> Contributors;
    Contributors contributors_;
    Vector2 texCoordOffset_;
    float sunAngleOffset_;
};
```

`IContributor` 是“贡献者”模式：`SkyDomeShadows` 和 `Dapple` 都实现此接口，把自己的阴影/光斑画进同一张光照图。

---

## 5. 天气

### 5.1 `Weather` —— 风与阵风

声明见 [weather.hpp](file:///workspace/src/lib/romp/weather.hpp)。是 `PyObjectPlus`，Python 可直接控制：

```cpp
// weather.hpp:29
class Weather : public PyObjectPlus
{
    Py_Header( Weather, PyObjectPlus )
public:
    void tick( float dTime );
    void windAverage( float xv, float yv ) { windVelX_ = xv; windVelY_ = yv; }
    const Vector2& wind() const { return wind_; }
    void windGustiness( float amount ) { windGustiness_ = amount; }
private:
    Vector2 wind_;        // 当前实际风（带阵风抖动）
    float windVelX_, windVelY_;   // 目标风速
    float windGustiness_;          // 阵风强度
};
```

风是 2D 向量（x/z 平面），`tick` 中根据 `windGustiness_` 给目标风速加随机抖动，得到实际 `wind_`。该值被 `EnviroMinder::tick` 传给 `ParticleSystemManager` 和 `s_windAnimation`（用于 shader 风动画常量）。

### 5.2 `Rain` —— 雨

声明见 [rain.hpp](file:///workspace/src/lib/romp/rain.hpp)。雨是 romp 中较复杂的子系统，包含雨线绘制、室内外区分、雨滴溅落粒子、雾发射：

```cpp
// rain.hpp:34
class Rain
{
public:
    void activate( const class EnviroMinder&, DataSectionPtr pSpaceSettings );
    void deactivate( const class EnviroMinder& );
    void draw();
    void drawInside( ChunkBoundary::Portal*, Chunk* );   // 室内雨
    void tick( float dt );
    void amount( float a );        // 雨量 0~1
    float amount() const;
    void update( const Weather& w, bool outside );
    void addAttachments( class PlayerAttachments & pa ); // 玩家身上雨溅
    void addFogEmitter();
    void remFogEmitter();
    static void disable( bool state );
private:
    Moo::Material material_;
    int   numRainBits_;
    float sphereSize_;
    int   minPseudoDepth_, maxPseudoDepth_;
    float velocitySensitivity_;
    Vector3 wind_;
    Vector3 lastCameraPos_;
    float amount_, maxFarFog_, maxNearFog_;
    PyMetaParticleSystem* rainSplashes_;                 // 雨滴溅落粒子
    std::vector<SourcePSA*>      rainSplashSources_;
    std::vector<TintShaderPSA*>  rainSplashTints_;
    Moo::NodePtr cameraNode_;
    FogController::Emitter emitter_;
    bool outside_, transiting_;
    float transitionFactor_;
    Moo::VertexXYZDUV verts_[4];
};
```

关键点：
- 雨本身是一个**雾发射器**（`emitter_`），雨天会增加雾密度；
- 雨滴溅落用粒子系统（`rainSplashes_`，复用 `Pixie` 的 `SourcePSA`/`TintShaderPSA`）；
- `drawInside` 处理穿过 portal 进入室内的雨；
- `amount_` 由 `EnviroMinder::drawFore` 用反迟滞刹车调整。

### 5.3 `Lightning` —— 闪电

声明见 [lightning.hpp](file:///workspace/src/lib/romp/lightning.hpp)。极简，决定是否闪电并返回一个 `Vector4`（闪电颜色/强度）：

```cpp
// lightning.hpp:15
class Lightning
{
public:
    void lightningStrike( const Vector3 & top );
    Vector4 decideLightning( float dTime );
private:
    float conflict_;
};
```

---

## 6. 雾：`FogController`

声明见 [fog_controller.hpp](file:///workspace/src/lib/romp/fog_controller.hpp)。全局单例，管理雾色、雾密度、多个局部雾发射器：

```cpp
// fog_controller.hpp:31
class FogController
{
public:
    static FogController & instance();
    class Emitter
    {
    public:
        Vector3 position_;
        uint32  colour_;
        float   maxMultiplier_;
        float   nearMultiplier_;
        mutable int id_;
        bool    localised_;
        void innerRadius( float radius );
        void outerRadius ( float radius );
        float outerRadiusSquared() const;
        float innerRadiusSquared() const;
        float falloffRange() const;
        // ...
    };
    typedef std::vector< Emitter > Emitters;

    void enable( bool state );
    int  addEmitter( const Emitter & emitter );
    void delEmitter( int emitterID );
    uint32 colour( void ) const;
    void   colour( uint32 c );
    float  multiplier( void ) const;       // 远裁面/倍率
    void   multiplier( float );
    float  nearMultiplier( void ) const;
    void   update( Emitter & emitter );
    void   commitFogToDevice();            // 把当前雾设置提交到设备
    uint32 farObjectTFactor( void ) const; // 远处非雾物体的 T 因子
    uint32 additiveFarObjectTFactor( void ) const;
    void   tick();                          // 相机移动后调用
private:
    uint32 colour_;
    Vector4 v4Colour_;
    float multiplier_;        // far clipping plane / multiplier
    float nearMultiplier_;    // -far clipping plane * multiplier
    float multiplierTotal_;   // 所有雾发射器倍率之和
    bool  enabled_;
    Emitters emitters_;
    int   global_emitter_id_;
    uint32 farObjectTFactor_;
    uint32 additiveFarObjectTFactor_;
};
```

### 6.1 雾发射器模型

`Emitter` 描述一个局部雾源（如天空、雨、火堆），有内半径/外半径，在内半径内全雾，内外之间线性衰减。`SkyGradientDome` 和 `Rain` 都通过 `addFogEmitter` 把自己注册为雾源。

### 6.2 `commitFogToDevice`

把计算好的雾色/密度写入 D3D 渲染状态（`D3DRS_FOGCOLOR`/`D3DRS_FOGSTART`/`D3DRS_FOGEND` 等）。`EnviroMinder` 在 `drawHind` 中调用两次（第二次因为 `Geometrics` 调用会破坏雾状态）。

---

## 7. 光照与光效

### 7.1 `LightMap` / `EffectLightMap` —— 光照图基类

声明见 [light_map.hpp](file:///workspace/src/lib/romp/light_map.hpp)。基于 RenderTarget 的光照图，把顶点着色器常量绑定到纹理与变换：

```cpp
// light_map.hpp:28
class LightMap : public Moo::DeviceCallback
{
public:
    LightMap( const std::string& className );
    void orthogonalProjection(float xExtent, float zExtent, Matrix& ret);
    virtual void activate();
    virtual void deactivate();
protected:
    virtual bool init(const DataSectionPtr pSection);
    bool needsUpdate(float gameTime);
    virtual void createLightMapSetter( const std::string& textureFeedName );
    virtual void createTransformSetter() = 0;   // 子类提供变换
    float lastTime_, timeTolerance_;            // 按时间容差更新
    int width_, height_;
    Moo::RenderTargetPtr pRT_;
    Moo::EffectConstantValuePtr transformSetter_, lightMapSetter_;
    PyTextureProviderPtr textureProvider_;
};

class EffectLightMap : public LightMap
{
protected:
    EffectLightMap( const std::string& className );
    bool init(const DataSectionPtr pSection);
    Moo::EffectMaterialPtr material_;   // 用 effect 材质生成光照图
};
```

`SkyLightMap` 继承 `EffectLightMap`，是云阴影光照图。

### 7.2 `LensEffect` —— 镜头光晕

声明见 [lens_effect.hpp](file:///workspace/src/lib/romp/lens_effect.hpp)。一个镜头光晕由多个 `FlareData`（光斑，可含次级光斑如日冕）组成，按遮挡程度分级：

```cpp
// lens_effect.hpp:30
class FlareData
{
public:
    typedef std::vector< FlareData > Flares;
    void load( DataSectionPtr pSection );
    uint32 colour() const;
    const std::string & material() const;
    float clipDepth() const;
    float width() const;
    float height() const;
    void draw( const Vector4 & clipPos, float alphaStrength, float scale,
                uint32 lensColour ) const;
    const Flares & secondaries() const;   // 次级光斑
private:
    uint32 colour_;
    std::string material_;
    float clipDepth_, width_, height_, age_;
    Flares secondaries_;
};

class LensEffect : public ReferenceCount
{
public:
    typedef std::map< float, class FlareData > OcclusionLevels;
    enum VisibilityType { VT_UNSET = 0, VT_TYPE1, VT_TYPE2 };
    virtual void tick( float dTime, float visibility );
    virtual void draw();
    float distanceToAlpha( float dist ) const;
    const OcclusionLevels & occlusionLevels() const;
private:
    uint32 id_;
    Vector3 position_;
    float maxDistance_, area_, fadeSpeed_, visibility_;
    bool clampToFarPlane_;
    mutable VisibilityType visibilityType_;
    uint32 colour_;
    OcclusionLevels occlusionLevels_;   // 遮挡比例→光斑
};
```

`OcclusionLevels` 是“遮挡程度→光斑”映射：不同可见度下用不同光斑（如完全可见用大日冕，半遮挡用小光斑）。

### 7.3 `LensEffectManager` —— 光晕管理器

声明见 [lens_effect_manager.hpp](file:///workspace/src/lib/romp/lens_effect_manager.hpp)。单例，管理所有镜头光晕的注册、衰减、遮挡查询、绘制：

```cpp
// lens_effect_manager.hpp:44
class LensEffectManager : public InitSingleton<LensEffectManager>
{
public:
    virtual void tick( float dTime );
    virtual void draw();
    Moo::EffectMaterialPtr getMaterial( const std::string& material );
    void add( uint32 id, const Vector3 & worldPosition, const LensEffect & le );
    void forget( uint32 id );
    void clear( void );
    void addPhotonOccluder( PhotonOccluder * occluder );
    void delPhotonOccluder( PhotonOccluder * occluder );
private:
    LensEffectsMap lensEffects_;     // id→光晕（风格1）
    LensEffectsMap lensEffects2_;    // id→光晕（风格2）
    Materials materials_;            // 光晕材质缓存
    typedef std::vector< PhotonOccluder * > PhotonOccluders;
    PhotonOccluders photonOccluders_;
    ZAttenuationOccluder zAttenuationHelper_;   // 风格2 用 Z 衰减
};
```

工作流（见类注释）：
1. 渲染期：物体调用 `add(id, pos, le)` 注册光晕；若已存在则刷新，新的淡入；
2. `tick`：衰减光晕年龄，剔除已不存在的；
3. `draw`：用 photon occluder 做光线遮挡检查，按可见度绘制。

### 7.4 `PhotonOccluder` —— 光子遮挡器接口

声明见 [photon_occluder.hpp](file:///workspace/src/lib/romp/photon_occluder.hpp)。所有“能否挡住光源”判断的基类：

```cpp
// photon_occluder.hpp:23
class PhotonOccluder
{
public:
    virtual float collides(
            const Vector3 & photonSourcePosition,
            const Vector3 & cameraPosition,
            const LensEffect& le ) = 0;
    virtual void beginOcclusionTests() {};
    virtual void endOcclusionTests() {};
};
```

`collides` 返回 0~1 的可见度。实现有：

| 实现 | 文件 | 算法 |
|---|---|---|
| `SkyDomeOccluder` | [sky_dome_occluder.hpp](file:///workspace/src/lib/romp/sky_dome_occluder.hpp) | 看向太阳的视图 + scissor + occlusion query |
| `ZBufferOccluder` | [z_buffer_occluder.hpp](file:///workspace/src/lib/romp/z_buffer_occluder.hpp) | Z-buffer + occlusion query |
| `TerrainOccluder` | [terrain_occluder.hpp](file:///workspace/src/lib/romp/terrain_occluder.hpp) | 地形高度 ray cast |
| `ChunkObstacleOccluder` | （chunk 库） | chunk 障碍 ray cast |
| `ZAttenuationOccluder` | [z_attenuation_occluder.hpp](file:///workspace/src/lib/romp/z_attenuation_occluder.hpp) | alpha 通道衰减，多帧累积淡入淡出 |

`EnviroMinder::activate` 按显卡能力选择 occluder：优先 `SkyDomeOccluder`，次之 `ZBufferOccluder`，最后 `ChunkObstacleOccluder`。

### 7.5 `ZAttenuationOccluder` —— Z 衰减遮挡器

声明见 [z_attenuation_occluder.hpp](file:///workspace/src/lib/romp/z_attenuation_occluder.hpp)。注释详尽描述了算法：用 staging 纹理累积遮挡结果，实现光晕淡入淡出而非突然出现/消失：

> 1. 每个屏幕可见光晕加一个 2x2 矩形，分配 staging 纹理位置；
> 2. Z-Read Disable：画到主缓冲 alpha 通道，全黑；
> 3. Z-Read Enable：再画一次，全白；
> 4. 把 alpha 结果拷到 staging 纹理；
> 5. 用 staging 纹理衰减绘制光晕。

### 7.6 `EnvironmentCubeMap` —— 环境立方体贴图

声明见 [environment_cube_map.hpp](file:///workspace/src/lib/romp/environment_cube_map.hpp)。用 `Moo::CubeRenderTarget` 周期性渲染环境，供反射/折射采样：

```cpp
// environment_cube_map.hpp:21
class EnvironmentCubeMap
{
public:
    EnvironmentCubeMap( uint16 textureSize = 0, uint8 numFacesPerFrame = 0 );
    void update( float dTime, bool defaultNumFaces = true, uint8 numberOfFaces = 1,
                 uint32 drawSelection = 0x80000000 );
    void numberOfFacesPerFrame( uint8 numberOfFaces );
    Moo::CubeRenderTargetPtr cubeRenderTarget() { return pCubeRenderTarget_; }
private:
    Moo::EffectConstantValuePtr pCubeRenderTargetEffectConstant_;
    Moo::CubeRenderTargetPtr pCubeRenderTarget_;
    uint8 environmentCubeMapFace_;
    uint8 numberOfFacesPerFrame_;   // 每帧只更新几个面，分摊开销
};
```

`numberOfFacesPerFrame_` 控制每帧更新几个面（6 面立方体的分摊更新），避免每帧渲染全环境。

### 7.7 `FlashBangEffect` —— 闪光弹效果

声明见 [flash_bang_effect.hpp](file:///workspace/src/lib/romp/flash_bang_effect.hpp)。全屏滤镜，模拟闪光弹致盲：

```cpp
// flash_bang_effect.hpp:30
class FlashBangEffect
{
public:
    void draw();
    const Vector4& fadeValues() const;
    void fadeValues( const Vector4& fadeValues );
private:
    Moo::Material blendMaterial_;
    Moo::RenderTarget* pRT_;     // 保存上一帧用于淡出
    bool haveLastFrame_;
    Vector4 fadeValues_;         // 淡出参数
};
```

### 7.8 `PyChunkLight` / `PyChunkSpotLight` —— 脚本光源

声明见 [py_chunk_light.hpp](file:///workspace/src/lib/romp/py_chunk_light.hpp)。脚本可控的 omni/spot 光，可静态设属性或挂动态 provider：

```cpp
// py_chunk_light.hpp:30 (注释)
/*~ class BigWorld.PyChunkLight
 *  This is a script controlled omni light which can be used as a diffuse 
 *  and/or specular light source for models and shells.
 *  The colour, radius, and position properties can be set to static values
 *  via the colour, innerRadius and outerRadius, and position attributes.
 *  Alternatively, They can be set to dynamic values via the source, bounds,
 *  and shader attributes.
 */
```

强度由 `innerRadius`/`outerRadius` 决定：内半径内满强度，内外之间线性衰减，外半径外为 0。

---

## 8. 植被（Flora）

### 8.1 `Flora` —— 植被系统主类

声明见 [flora.hpp](file:///workspace/src/lib/romp/flora.hpp)。每 space 一个，是 `Moo::DeviceCallback`，管理一组 `FloraBlock`（分块网格）和 `Ecotype`（生态类型）：

```cpp
// flora.hpp:42
class Flora : public Moo::DeviceCallback, public Aligned
{
public:
    bool init( DataSectionPtr pSection, uint32 terrainVersion );
    void activate();
    void deactivate();
    void update( float dTime, class EnviroMinder& enviro );
    void draw( float dTime, class EnviroMinder& enviro );
    void drawShadows( ShadowCasterPtr pShadowCaster );
    Ecotype& ecotypeAt( const Vector2& );
    Ecotype::ID generateEcotypeID( const Vector2& p );
    void seedOffsetTable( const Vector2& );   // 按世界位置种子随机数表
    const Vector2& nextOffset();              // 取下一个随机偏移
    float nextRotation();
    float nextRandomFloat();
    Terrain::BaseTerrainBlockPtr getTerrainBlock( const Vector3 & pos, Vector3 & relativePos, ... ) const;
    void resetBlockAt( float x, float z );    // 标记 block 脏
    static void floraReset();
    static void enabled( bool state ) { s_enabled_ = state; }
private:
    FloraBlock* blocks_[BLOCK_STRIDE][BLOCK_STRIDE];   // 网格分块
    Ecotype* ecotypes_[256];                            // 最多 256 种生态
    Ecotype  degenerateEcotype_;
    uint32 vbSize_, numVertices_;
    Vector2 offsets_[LUT_SIZE];                         // 随机偏移查找表
    float   randoms_[LUT_SIZE];                         // 随机数查找表
    int lutSeed_;
    Vector2 lastPos_;
    bool cameraTeleport_;
    class FloraRenderer* pRenderer_;
    int centerBlockX_, centerBlockZ_;                   // 中心 block 坐标
    class FloraTexture* pFloraTexture_;
};
```

#### 8.1.1 关键设计

- **`Ecotype` 生态类型**：每种植被（草、花、灌木）是一个 Ecotype，ID 烘焙在地形纹理里，`ecotypeAt(pos)` 按地形纹理查生态。
- **随机偏移 LUT**：`offsets_`/`randoms_` 是按世界位置种子的查找表，保证同一位置的植被分布稳定（相机来回移动不会变化）。
- **block 网格跟随相机**：`moveBlocks` 在相机移动时把身后 block 移到身前，`centerBlockX_/Z_` 跟踪中心。
- **`cameraTeleport_`**：远距离传送时重置整个网格。

### 8.2 `Ecotype` —— 生态类型

声明见 [ecotype.hpp](file:///workspace/src/lib/romp/ecotype.hpp)。每种生态有纹理、UV 偏移、生成器，支持后台线程加载：

```cpp
// ecotype.hpp:35
class Ecotype
{
public:
    typedef uint8 ID;
    static const uint32 ID_AUTO = 255;
    void init( DataSectionPtr allEcotypesSection, uint8 id );
    void offsetUV( Vector2& offset );
    uint32 generate( class FloraVertexContainer* pVerts, uint32 id, uint32 maxVerts,
                     const Matrix& objectToWorld, const Matrix& objectToChunk,
                     BoundingBox& bb );
    bool isEmpty() const;
    uint8 id() const { return id_; }
    class EcotypeGenerator* generator() const { return generator_; }
    void generator( class EcotypeGenerator* g ) { generator_ = g; }
private:
    struct BkLoadInfo { class BackgroundTask* loadingTask_; Ecotype* ecotype_; DataSectionPtr pSection_; };
    static void backgroundInit( void* );
    static void onBackgroundInitComplete( void* );
    std::string textureResource_;
    Moo::BaseTexturePtr pTexture_;
    Vector2 uvOffset_;
    uint8 id_;
    class EcotypeGenerator* generator_;
    class Flora* flora_;
    BkLoadInfo* loadInfo_;
    bool isLoading_, isInited_;
    static SimpleMutex s_deleteMutex_;
};
```

`generate` 在指定位置生成植被几何，写入 `FloraVertexContainer`。`EcotypeGenerator`（见 ecotype_generators.hpp）是生成策略（如随机散布、网格分布）。

### 8.3 `FloraBlock` / `FloraRenderer` / `FloraTexture`

- `FloraBlock`（flora_block.hpp）：一个网格分块，持有一批植被顶点，按距离排序绘制；
- `FloraRenderer`（flora_renderer.hpp）：把所有 block 的顶点合并到顶点缓冲绘制；
- `FloraTexture`（flora_texture.hpp）：把所有 Ecotype 的纹理拼成一张大图集（texture atlas），减少 draw call；
- `FloraLightMap`（flora_light_map.hpp）：植被专用的光照图；
- `FloraVertexType`（flora_vertex_type.hpp）：植被顶点格式。

### 8.4 类层级

```
Moo::DeviceCallback, Aligned
   └── Flora
         持有 FloraBlock[BLOCK_STRIDE][BLOCK_STRIDE]
         持有 Ecotype*[256]
               └── Ecotype (持有 EcotypeGenerator, FloraTexture 中的一个图集块)
         持有 FloraRenderer (合并顶点缓冲)
         持有 FloraTexture (图集)
```

---

## 9. 水

### 9.1 `Water` —— 水面

声明见 [water.hpp](file:///workspace/src/lib/romp/water.hpp)。是 `ReferenceCount` + `Moo::DeviceCallback`，一个水面由多个 `WaterCell` 组成，支持反射、折射、波动模拟：

```cpp
// water.hpp:143
class Water : public ReferenceCount, public Moo::DeviceCallback, public Aligned
{
public:
    enum { ALWAYS_VISIBLE, INSIDE_ONLY, OUTSIDE_ONLY };
    typedef struct _WaterState
    {
        Vector3 position_; Vector2 size_; float orientation_;
        float tessellation_, textureTessellation_, consistency_;
        float fresnelConstant_, fresnelExponent_;
        float reflectionScale_, refractionScale_;
        Vector2 scrollSpeed1_, scrollSpeed2_, waveScale_;
        float sunPower_, sunScale_, windVelocity_, depth_;
        Vector4 reflectionTint_, refractionTint_;
        float simCellSize_, smoothness_;
        bool useEdgeAlpha_, useSimulation_, useCubeMap_, reflectBottom_;
        Vector4 deepColour_; float fadeDepth_;
        float foamIntersection_, foamMultiplier_, foamTiling_;
        bool bypassDepth_;
        std::string foamTexture_, waveTexture_, reflectionTexture_, transparencyTable_;
    } WaterState;

    Water( const WaterState& config, RompColliderPtr pCollider = NULL );
    void rebuild( const WaterState& config );
    void tick();
    void draw( Waters & group, float dTime );
    void updateSimulation( float dTime );
    bool addMovement( const Vector3& from, const Vector3& to, const float diameter ); // 涟漪
    bool canReflect(float* retDist = 0) const;
    void updateVisibility();
private:
    WaterState config_;
    Matrix transform_, invTransform_;
    Vector3 camPos_;
    std::vector<VertexBufferPtr> vertexBuffers_;
    Moo::EffectMaterialPtr simulationEffect_;
    WaterCell::WaterCellVector cells_;
    WaterCell::SimulationCellPtrList activeSimulations_;   // 活跃模拟 cell
    WaterScene* waterScene_;                                // 反射/折射场景
    Moo::BaseTexturePtr normalTexture_, foamTexture_, reflectionTexture_;
    RompColliderPtr pCollider_;
};
```

`WaterState` 是水面的全部可调参数（Fresnel、波浪、反射、泡沫等）。`addMovement` 触发涟漪（如角色涉水）。

### 9.2 `WaterCell` —— 水面分块

```cpp
// water.hpp:81
class WaterCell : public SimulationCell
{
public:
    bool init( Water* water, Vector2 start, Vector2 size );
    void initEdge( int index, WaterCell* cell );    // 邻接关系
    void initSimulation( int size, float cellSize );
    virtual void bindNeighbour( Moo::EffectMaterialPtr effect, int edge );
    void checkActivity( SimulationCellPtrList& activeList, WaterCellPtrList& edgeList );
    Moo::RenderTargetPtr simTexture();    // 模拟纹理
    float cellDistance() const;
    void updateDistance(const Vector3& camPos);
private:
    Vector2 min_, max_, size_;
    WaterCell* edgeCells_[4];             // 四邻
    Moo::IndexBuffer indexBuffer_;
    Water* water_;
    float cellDistance_;
};
```

水面被切成 cell 网格，只有相机附近的 cell 激活波动模拟（`activeSimulations_`），按距离剔除省开销。

### 9.3 `Waters` —— 水面集合单例

```cpp
// water.hpp:415
class Waters : public std::vector< Water* >, public Moo::EffectManager::IListener
{
public:
    static Waters& instance();
    void tick( float dTime );
    void drawDrawList( float dTime );
    void updateSimulations( float dTime );
    void checkVolumes();        // 检查相机是否在水体内
    int simulationLevel() const;
    void rainAmount( float rainAmount );   // 雨影响水面
    static uint32 addWaterVolumeListener( MatrixProviderPtr, PyObjectPtr ); // 进出水回调
    static void delWaterVolumeListener( uint32 id );
private:
    bool insideWaterVolume_;    // 眼睛是否在水内
    float movementImpact_;      // 模拟冲击缩放
    float rainAmount_;
    float impactTimer_;         // 静止脉冲计时
    VolumeListeners listeners_; // 进出水监听器
};
```

`Waters` 管理共享 effect、质量档位、相机进出水体的回调。`rainAmount_` 接收雨量，影响水面涟漪。

### 9.4 `WaterSimulation` —— 水波模拟

声明见 [water_simulation.hpp](file:///workspace/src/lib/romp/water_simulation.hpp)。用 RenderTarget 存储水面高度场，`SimulationTextureBlock` 管理多张模拟纹理（ping-pong）：

```cpp
// water_simulation.hpp:52
class SimulationTextureBlock
{
public:
    void init( int width, int height );
    void tickSimulation();
    bool lock();
    Moo::RenderTargetPtr& operator()(int idx);   // 双缓冲
private:
    // ...
};
```

`SIM_BORDER_SIZE`（3）是模拟边界尺寸，与 shader `water_common.fxh` 联动。

### 9.5 `Sea` —— 大海

声明见 [sea.hpp](file:///workspace/src/lib/romp/sea.hpp)。简单的大海，参数化波浪/潮汐/颜色：

```cpp
// sea.hpp:23
class Sea : public ReferenceCount
{
public:
    void draw( float dTime, float timeOfDay );
    float seaLevel() const; void seaLevel( float f );
    float wavePeriod() const; void wavePeriod( float f );
    float waveExtent() const; void waveExtent( float f );
    float tidePeriod() const; void tidePeriod( float f );
    float tideExtent() const; void tideExtent( float f );
    uint32 surfaceTopColour() const; void surfaceTopColour( uint32 );
    uint32 surfaceBotColour() const; void surfaceBotColour( uint32 );
    uint32 underwaterColour() const; void underwaterColour( uint32 );
private:
    float seaLevel_, wavePeriod_, waveExtent_, tidePeriod_, tideExtent_;
    uint32 surfaceTopColour_, surfaceBotColour_, underwaterColour_;
    float waveTime_;
};
```

`Sea` 是 EnviroMinder 中 `Seas`（`std::vector<Sea*>`）的元素，在 `drawFore` 中绘制。

### 9.6 `WaterSplash` / `WaterSceneRenderer` / `PyWaterVolume`

- `WaterSplash`（water_splash.hpp）：水花溅落粒子；
- `WaterSceneRenderer`（water_scene_renderer.hpp）：水面反射/折射场景渲染器；
- `PyWaterVolume`（py_water_volume.hpp）：Python 水体体积绑定，进出水触发回调。

---

## 10. 地形 Occluder

### 10.1 `TerrainOccluder`

声明见 [terrain_occluder.hpp](file:///workspace/src/lib/romp/terrain_occluder.hpp)。用地形高度做光线投射，判断太阳是否被山遮挡：

```cpp
// terrain_occluder.hpp:22
class TerrainOccluder : public PhotonOccluder
{
public:
    virtual float collides(
            const Vector3 & photonSourcePosition,
            const Vector3 & cameraPosition,
            const LensEffect& le );
};
```

### 10.2 `ZBufferOccluder`

声明见 [z_buffer_occluder.hpp](file:///workspace/src/lib/romp/z_buffer_occluder.hpp)。用 Z-buffer + occlusion query 判断光晕可见性：

```cpp
// z_buffer_occluder.hpp:28
class ZBufferOccluder : public PhotonOccluder, public Moo::DeviceCallback
{
public:
    static bool isAvailable();
    virtual float collides( ... );
    virtual void beginOcclusionTests();
    virtual void endOcclusionTests();
protected:
    virtual void writePixel( const Vector3& source );
    virtual void writeArea( const Vector3& source, float size );
private:
    OcclusionQueryHelper helper_;
    OcclusionQueryHelper helperZBuffer_;
    Moo::EffectMaterialPtr mat_;
};
```

### 10.3 Occluder 选择策略

`EnviroMinder::activate`（[enviro_minder.cpp:614](file:///workspace/src/lib/romp/enviro_minder.cpp)）按硬件能力级联选择：

```cpp
// enviro_minder.cpp:614
if (SkyDomeOccluder::isAvailable()) {
    skyDomeOccluder_ = new SkyDomeOccluder( *this );
    LensEffectManager::instance().addPhotonOccluder( skyDomeOccluder_ );
}
if (ZBufferOccluder::isAvailable()) {
    zBufferOccluder_ = new ZBufferOccluder();
    LensEffectManager::instance().addPhotonOccluder( zBufferOccluder_ );
} else {
    chunkObstacleOccluder_ = new ChunkObstacleOccluder();
    LensEffectManager::instance().addPhotonOccluder( chunkObstacleOccluder_ );
}
```

---

## 11. 控制台与字体

### 11.1 `XConsole` —— 控制台基类

声明见 [xconsole.hpp](file:///workspace/src/lib/romp/xconsole.hpp)。游戏内控制台（如命令行、调试输出）的基类，是 `Moo::DeviceCallback`：

```cpp
// xconsole.hpp:29
class XConsole : Moo::DeviceCallback
{
public:
    virtual bool init();
    virtual void activate( bool isReactivate );
    virtual void deactivate() {}
    void print( const std::wstring &string );
    void formatPrint( const char * format, ... );
    void clear();
    virtual void draw( float dTime );
    void scrollDown(); void scrollUp();
    void setConsoleColour( uint32 colour );
    void lineColourOverride( int line, uint32 colour );
    FontPtr font() { return font_; }
    int visibleWidth() const; int visibleHeight() const;
    const wchar_t * line( int line ) const;
    void setCursor( uint8 cx, uint8 cy );
protected:
    FontPtr font_;
    int visibleWidth_, visibleHeight_;
    bool fitToScreen_;
    // ...
};

const int MAX_CONSOLE_WIDTH  = 200;
const int MAX_CONSOLE_HEIGHT = 100;
```

控制台是字符网格（最大 200×100），每行可单独设颜色（`lineColourOverride`）。

### 11.2 `ConsoleManager` —— 控制台管理器

声明见 [console_manager.hpp](file:///workspace/src/lib/romp/console_manager.hpp)。单例，管理多个控制台 + 输入分发：

```cpp
// console_manager.hpp:25
class ConsoleManager : public InputHandler
{
public:
    void add( XConsole * pConsole, const char * label,
        KeyCode::Key key = KeyCode::KEY_NONE, uint32 modifiers = 0 );
    void draw( float dTime );
    static ConsoleManager & instance();
    XConsole * pActiveConsole() const;
    XConsole * find( const char * name ) const;
    bool handleKeyEvent( const KeyEvent & event );
    void activate( XConsole * pConsole );
    void toggle( const char * label );
    void deactivate();
private:
    XConsole * pActiveConsole_;
    typedef std::map< uint32, XConsole *> KeyedConsoles;        // 按键绑定
    typedef StringHashMap< XConsole *> LabelledConsoles;        // 按名查找
    KeyedConsoles keyedConsoles_;
    LabelledConsoles labelledConsoles_;
    bool darkenBackground_;
};
```

每个控制台可绑定一个快捷键，`toggle` 按键切换激活。`ConsoleManager` 是 `InputHandler`，捕获键盘事件分发给活动控制台。

### 11.3 `Font` —— 字体

声明见 [font.hpp](file:///workspace/src/lib/romp/font.hpp)。可立即绘制或生成 mesh：

```cpp
// font.hpp:44
class Font : public ReferenceCount
{
public:
    // 生成 clip 坐标 mesh（不受 scale 影响）
    virtual float drawIntoMesh(
        VectorNoDestructor<Moo::VertexXYZDUV2>& mesh,
        const std::wstring& str,
        float clipX = 0.f, float clipY = 0.f,
        float* w = NULL, float* h = NULL,
        bool multiline = false, bool colourFormatting = false );
    // 在指定框内生成 mesh
    virtual void drawIntoMesh( ... float w, float h, ... );
    // 直接画到屏幕
    void drawConsoleString( const std::string& str, int col, int row, int x = 0, int y = 0 );
    virtual void drawString( const std::string& str, int x, int y );
    virtual int drawString( std::wstring wstr, int x, int y, int w, int h,
        int minHyphenWidth = 0, const std::wstring& wordBreak = L" ",
        const std::wstring& punctuation = ... );   // 自动换行
    void colour( uint32 col );
    void scale( const Vector2& s );
    Moo::BaseTexturePtr pTexture();
    void fitToScreen( bool state, const Vector2& numCharsXY );
    FontMetrics& metrics();
protected:
    Font( FontMetrics& fm );
    virtual float makeCharacter( Moo::VertexXYZDUV2* vert, wchar_t c, const Vector2& pos );
    FontMetrics& metrics_;
    Vector2 scale_;
    uint32 colour_;
};
```

关键点：
- `drawIntoMesh` 把文字变成顶点流（`VertexXYZDUV2`，双 UV 用于双纹理），可被复用；
- `colourFormatting` 支持内联颜色控制码；
- `drawString` 带自动换行/连字符/标点断词；
- 字形只保证存活一帧，长期持有需用 `CachedFont`（见注释）。

### 11.4 字体子系统类层级

```
ReferenceCount
   └── Font (持有 FontMetrics&)
         ├── drawIntoMesh / drawString / drawConsoleString
         └── 字形仅存活一帧

FontMetrics (font_metrics.hpp)  字体度量（字宽/字高/字距）
   └── GlyphCache (glyph_cache.hpp)  字形缓存（按需生成字形纹理）
         └── GlyphReferenceHolder (glyph_reference_holder.hpp)  引用计数持有字形

FontManager (font_manager.hpp)  字体单例，按名加载/缓存 Font
```

---

## 12. UI / 工具

### 12.1 `Progress` / `GUIProgress` / `SuperModelProgress` —— 加载进度

声明见 [progress.hpp](file:///workspace/src/lib/romp/progress.hpp)。`ProgressTask` 表示单个任务进度，`ProgressDisplay` 管理一组任务并显示：

```cpp
// progress.hpp:25
class ProgressTask
{
public:
    ProgressTask( class ProgressDisplay * pOwner, const std::string & name, float length = 0 );
    bool step( float progress = 1 );   // 累加进度
    bool set( float done = 0 );        // 设置进度
    void length( float length ) { length_ = length; }
private:
    class ProgressDisplay * pOwner_;
    float done_, length_;    // length<=0 表示不确定时长
};

class ProgressDisplay
{
public:
    typedef bool (*ProgressCallback)();
    ProgressDisplay( FontPtr pFont = NULL, ProgressCallback pCallback = NULL,
        uint32 colour = 0xFF26D1C7 );
    // ...
};
```

`GUIProgressDisplay`（gui_progress.hpp）用 GUI 绘制进度条；`SuperModelProgressDisplay`（super_model_progress.hpp）在模型加载时显示进度。

### 12.2 `CustomMesh` —— 自定义网格模板

声明见 [custom_mesh.hpp](file:///workspace/src/lib/romp/custom_mesh.hpp)。模板类，继承 `VectorNoDestructor<T>`，提供即时绘制：

```cpp
// custom_mesh.hpp:26
template <class T>
class CustomMesh : public VectorNoDestructor< T >
{
public:
    explicit CustomMesh( D3DPRIMITIVETYPE primitiveType = D3DPT_TRIANGLELIST );
    D3DPRIMITIVETYPE primitiveType() const { return primitiveType_; }
    int vertexFormat() const { return T::fvf(); }
    int nVerts() const { return size(); }
    HRESULT draw();
    HRESULT drawEffect( Moo::VertexDeclaration* decl = NULL );
    HRESULT drawRange( uint32 from, uint32 to );
private:
    D3DPRIMITIVETYPE primitiveType_;
};
```

`SunAndMoon`、`Distortion`、`ZAttenuationOccluder` 等都用 `CustomMesh<VertexXYZNDUV>` 等实例。

### 12.3 `Geometrics` —— 几何工具

声明见 [geometrics.hpp](file:///workspace/src/lib/romp/geometrics.hpp)。静态工具类，提供矩形、线、包围盒、球等基础绘制：

```cpp
// geometrics.hpp:34
class Geometrics
{
public:
    static Geometrics & instance();
    static void createRectMesh( const Vector2& topLeft, const Vector2& bottomRight,
        const Moo::Colour& colour, CustomMesh<Moo::VertexTL>& result, float depth = 0 );
    static void drawRect( const Vector2& topLeft, const Vector2& bottomRight,
        const Moo::Colour& colour, float depth = 0 );
    static void drawRect( const Vector2& topLeft, const Vector2& bottomRight,
        Moo::EffectMaterial& material, bool linearUV = false );
    static void drawShadowPoly( const Moo::Colour& colour );
    // wireBox, sphere, line, point 等
};
```

### 12.4 `LODSettings` —— LOD 设置

声明见 [lod_settings.hpp](file:///workspace/src/lib/romp/lod_settings.hpp)。单例，应用 LOD 偏移到距离：

```cpp
// lod_settings.hpp:25
class LodSettings
{
public:
    void init(DataSectionPtr resXML);
    float applyLodBias(float distance);              // 应用偏移到单个距离
    void applyLodBias(float & lodNear, float & lodFar); // 应用到近远 LOD
    static LodSettings & instance();
private:
    std::vector<float> lodOptions_;
    Moo::GraphicsSetting::GraphicsSettingPtr lodSettings_;
};
```

被 `ParticleSystem::draw` 等调用，按画质档位调整 LOD。

### 12.5 `AnimationGrid`

声明见 [animation_grid.hpp](file:///workspace/src/lib/romp/animation_grid.hpp)。植被等用的动画网格，按位置/时间生成摆动。

### 12.6 `DebugGeometry` / `LineHelper` / `LineEditor`

- `DebugGeometry`（debug_geometry.hpp）：调试几何绘制；
- `LineHelper`（line_helper.hpp）：线段绘制工具；
- `LineEditor`（lineeditor.hpp）：可编辑线段（编辑器用）。

### 12.7 `HandlePool` / `OcclusionQueryHelper` / `HistogramProvider`

- `HandlePool`（[handle_pool.hpp](file:///workspace/src/lib/romp/handle_pool.hpp)）：句柄池，把 ID 映射到固定数量句柄（uint16），用于 occlusion query 等限量资源：

```cpp
// handle_pool.hpp:17
class HandlePool
{
public:
    typedef uint16 Handle;
    static const Handle INVALID_HANDLE = 0xffff;
    HandlePool( uint16 numHandles );
    Handle handleFromId( uint32 id );
    void beginFrame(); void endFrame(); void reset();
private:
    HandleMap handleMap_;                  // id→{handle,used}
    std::stack<Handle> unusedHandles_;     // 空闲句柄栈
    uint16 numHandles_;
};
```

- `OcclusionQueryHelper`（[occlusion_query_helper.hpp](file:///workspace/src/lib/romp/occlusion_query_helper.hpp)）：异步 occlusion query 封装，多帧平均结果：

```cpp
// occlusion_query_helper.hpp:32
class OcclusionQueryHelper : public Moo::DeviceCallback
{
public:
    OcclusionQueryHelper( uint16 maxHandles, uint32 defaultValue = 0,
        uint8 numFrames = MAX_FRAME_LAG );
    void begin();
    bool beginQuery( HandlePool::Handle idx );
    void endQuery( HandlePool::Handle idx );
    void end();
    int avgResult( HandlePool::Handle idx );   // 最近 n 帧平均可见像素
private:
    bool *resultPending_;
};
```

- `HistogramProvider`（histogram_provider.hpp）：直方图数据提供器（如亮度直方图用于色调映射）。

---

## 13. 杂项

### 13.1 `Dapple` —— 斑驳光影

声明见 [dapple.hpp](file:///workspace/src/lib/romp/dapple.hpp)。`SkyLightMap::IContributor`，画树冠斑驳光斑到天空光照图：

```cpp
// dapple.hpp:20
class Dapple : public SkyLightMap::IContributor
{
public:
    void dapple(float sunAngle);
    bool dappleEnabled();
    void activate( const class EnviroMinder& enviro, DataSectionPtr pSpaceSettings, SkyLightMap* skyLightMap );
    bool needsUpdate();
    void render( SkyLightMap* lightMap, Moo::EffectMaterial* material, float sunAngle );
private:
    Moo::EffectMaterial* dappleMaterial_;
    Moo::BaseTexturePtr pDappleTex_;
    Vector2 spaceMin_, spaceMax_;
};
```

> 注：`enviro_minder.hpp` 中 `dapple_` 被注释为 “unsupported feature at the moment. to be finished.”，故实际未启用。

### 13.2 `Distortion` —— 屏幕扭曲

声明见 [distortion.hpp](file:///workspace/src/lib/romp/distortion.hpp)。单例，实现热扭曲/水扭曲后处理。注释详尽描述了 begin/end 流程：

```cpp
// distortion.hpp:68
class Distortion : public Moo::DeviceCallback
{
public:
    bool init();
    void finz();
    static bool isSupported();
    bool isEnabled();
    void tick( float dTime );
    bool begin();
    void end();
private:
    void drawScene();
    void drawMasks();               // 画水/扭曲物体遮罩到 alpha
    void drawDistortionChannel( bool clear = true );
    void copyBackBuffer();
    Moo::VisualPtr visual_;
    EffectParameterCache parameters_;
    Moo::EffectMaterialPtr effectMaterial_;
    static Moo::RenderTargetPtr s_pRenderTexture_;
};
```

`begin` 拷贝后缓冲到离屏纹理并 push 全屏 RT；`end` 画遮罩、绑 MRT depth、画实际水面与扭曲通道物体。

### 13.3 `FrameLogger` —— 帧日志

声明见 [frame_logger.hpp](file:///workspace/src/lib/romp/frame_logger.hpp)。极简，记录每帧统计到日志：

```cpp
// frame_logger.hpp:21
class FrameLogger
{
public:
    static void init();
};
```

通过 `Debug/FrameLogger` watcher 激活与定制。

### 13.4 `EngineStatistics` / `ResourceManagerStats` / `ResourceStatistics` / `ResourceRef`

- `EngineStatistics`（engine_statistics.hpp）：引擎统计（FPS、三角面数、draw call 等）；
- `ResourceManagerStats`（resource_manager_stats.hpp）：资源管理器统计；
- `ResourceStatistics`（resource_statistics.hpp）：资源统计；
- `ResourceRef`（resource_ref.hpp）：资源引用跟踪。

### 13.5 `TextureFeeds` / `TextureExplorer`

声明见 [texture_feeds.hpp](file:///workspace/src/lib/romp/texture_feeds.hpp)。单例，命名纹理流——脚本可注入动态纹理：

```cpp
// texture_feeds.hpp:27
class TextureFeeds : public InitSingleton< TextureFeeds >
{
public:
    static void addTextureFeed( const std::string& identifier, PyTextureProviderPtr texture );
    static void delTextureFeed( const std::string& identifier );
    static PyTextureProviderPtr getTextureFeed( const std::string& identifier );
    static Moo::BaseTexturePtr get( const std::string& identifier );
private:
    typedef StringHashMap< PyTextureProviderPtr > Feeds;
    Feeds feeds_;
};

class TextureFeed : public Moo::BaseTexture
{
    // 委托 pTexture() 到当前 feed 的 provider
};
```

`TextureFeed` 是一个“可变纹理”：shader 引用名字，运行时由脚本切换底层纹理。`TextureExplorer`（texture_explorer.hpp）是调试浏览工具。

### 13.6 `Watermark` —— 水印

声明见 [watermark.hpp](file:///workspace/src/lib/romp/watermark.hpp)。在纹理中编/解码水印数据（防泄露）：

```cpp
// watermark.hpp:15
std::string decodeWaterMark (const std::string& encodedData, int width, int height);
std::string encodeWaterMark(const char* data, size_t size);
```

### 13.7 `Py*` 绑定集合

romp 还包含大量 Python 绑定类：

| 类 | 文件 | 用途 |
|---|---|---|
| `PyD3DEnums` | py_d3d_enums.hpp | 暴露 D3D 枚举给 Python |
| `PyGraphicsSetting` | py_graphics_setting.hpp | 图形设置 |
| `PyMaterial` | py_material.hpp | 材质 |
| `PyRenderTarget` | py_render_target.hpp | RenderTarget |
| `PyResourceRefs` | py_resource_refs.hpp | 资源引用 |
| `PyShimmerCountProvider` | py_shimmer_count_provider.hpp | 闪烁计数 provider |
| `PyTextureProvider` | py_texture_provider.hpp | 纹理 provider（Vector4 输出） |
| `PyChunkLight` | [py_chunk_light.hpp](file:///workspace/src/lib/romp/py_chunk_light.hpp) | 脚本 omni 光 |
| `PyChunkSpotLight` | py_chunk_spot_light.hpp | 脚本 spot 光 |
| `PyWaterVolume` | py_water_volume.hpp | 水体体积 |

### 13.8 `romp_collider` / `romp_sound`

- `romp_collider.hpp`：碰撞查询接口（被 `Water`、`Rain` 等用于地面碰撞）；
- `romp_sound.hpp`：默认声音 provider（被 `ParticleSystemManager` 用于碰撞声）。

---

## 14. 类层级总览

```
EnviroMinder (per-space)
   ├── TimeOfDay : Aligned
   │     └── OutsideLighting : Aligned
   ├── Weather : PyObjectPlus
   ├── SkyGradientDome : Moo::DeviceCallback
   ├── SunAndMoon (持 LensEffect)
   ├── Rain (持 PyMetaParticleSystem*, FogController::Emitter)
   ├── SkyDomeShadows : SkyLightMap::IContributor
   ├── SkyDomeOccluder : PhotonOccluder, Moo::DeviceCallback
   ├── ZBufferOccluder : PhotonOccluder, Moo::DeviceCallback
   ├── ChunkObstacleOccluder : PhotonOccluder  (chunk 库)
   ├── SkyLightMap : EffectLightMap : LightMap : Moo::DeviceCallback
   ├── Flora : Moo::DeviceCallback, Aligned
   │     ├── FloraBlock[]
   │     ├── Ecotype*[256] (持 EcotypeGenerator)
   │     ├── FloraRenderer
   │     └── FloraTexture
   ├── Seas (std::vector<Sea*>)
   │     └── Sea : ReferenceCount
   ├── EnvironmentCubeMap
   ├── Decal / FootPrintRenderer (duplo 库)
   └── skyDomes_ : std::vector<Moo::VisualPtr>

PhotonOccluder (抽象)
   ├── SkyDomeOccluder
   ├── ZBufferOccluder
   ├── TerrainOccluder
   └── ChunkObstacleOccluder

LensEffectManager : InitSingleton<LensEffectManager>
   持 LensEffectsMap (id→LensEffect)
   持 PhotonOccluders[]
   持 ZAttenuationOccluder

Water : ReferenceCount, Moo::DeviceCallback, Aligned
   持 WaterCell[] : SimulationCell
   持 WaterScene (WaterSceneRenderer)
Waters : std::vector<Water*>, Moo::EffectManager::IListener  (单例)

FogController (单例)
   持 Emitters[] (FogController::Emitter)

XConsole : Moo::DeviceCallback
ConsoleManager : InputHandler (单例)
Font : ReferenceCount
   └── FontMetrics → GlyphCache → GlyphReferenceHolder
```

---

## 15. 典型调用流：一天时间循环 + 渲染

下面整合 `TimeOfDay`、`SkyGradientDome`、`FogController`、`LensEffectManager` 在一日循环中的协作：

```
游戏每帧 dTime
   │
   ▼
EnviroMinder::tick(dTime, outside)
   │
   ├─> weather_->tick(dTime)
   │      wind_ = 目标风速 + 阵风抖动
   │
   ├─> ParticleSystemManager::windVelocity(weather_->wind())
   │      （风传给所有粒子）
   │
   ├─> timeOfDay_->tick(dTime)
   │      time_ += dTime / secondsPerGameHour_
   │      if (time_ >= 24) time_ -= 24
   │      sunColour      ← sunAnimation_.interpolate(time_)
   │      ambientColour  ← ambientAnimation_.interpolate(time_)
   │      fogColour      ← fogAnimation_.interpolate(time_)
   │      sunTransform   ← 由 sunAngle_ + time_ 计算
   │      moonTransform  ← 由 moonAngle_ + time_ 计算
   │      useMoonColour_ ← (太阳低于地平线)
   │
   ├─> skyGradientDome_->update(gameTime)
   │      重算天空渐变 alpha + 雾色 (fogAnimation_)
   │
   ├─> decideLightingAndFog()
   │      outLight.sunColour     *= sunlightControl_  (脚本 Vector4Provider)
   │      outLight.ambientColour *= ambientControl_
   │      fogDensity = fogControl_.w
   │      skyGradientDome_->fogModulation(modcol, fogDensity)
   │
   ├─> rain_->tick(dTime)
   ├─> tickSkyDomes(dTime)
   └─> flora_->update(dTime, *this)
   │
   ▼
EnviroMinder::drawHind(dTime)
   │
   ├─> skyGradientDome_->addFogEmitter()      （天空作为雾源）
   ├─> rain_->addFogEmitter()                 （雨作为雾源）
   ├─> FogController::tick()                  （累加所有发射器，按相机位置算最终雾）
   ├─> FogController::commitFogToDevice()     （写 D3D 雾状态）
   ├─> skyLightMap_->update(sunAngle, ...)    （更新云阴影光照图）
   │      调用 SkyDomeShadows::render()       （天空盒画进光照图）
   │      调用 Dapple::render()               （斑驳光斑画进光照图，若启用）
   ├─> environmentCubeMap_->update(dTime)     （分摊更新环境立方体贴图）
   └─> (老显卡) drawSkySunCloudsMoon()
          SkyBoxScopedSetup (改视口/远平面)
          sunAndMoon_->draw()                 （太阳/月亮公告板）
          sunAndMoon 注册 sunLensEffect_ 到 LensEffectManager
          skyGradientDome_->draw(timeOfDay_)
          starDome_->draw(gameTime)           （夜晚才可见）
          skyDomes_ 绘制
   │
   ▼
(场景物体绘制: ChunkManager::draw → 模型/粒子/水面, 采样 skyLightMap / environmentCubeMap)
   │
   ▼
EnviroMinder::drawHindDelayed(dTime)
   └─> (新显卡) drawSkySunCloudsMoon()        （省填充率：天空在场景后画）
   │
   ▼
EnviroMinder::drawFore(dTime)
   ├─> rain_->amount 调整 (反迟滞)
   ├─> seas_->draw(dTime, gameTime)
   ├─> decal_->draw() / footPrintRenderer_->draw()  (ClipPlaneBias)
   ├─> flora_->draw(dTime, *this)
   └─> rain overlay / lightning
   │
   ▼
LensEffectManager::draw()
   ├─> beginOcclusionTests()  (各 PhotonOccluder)
   ├─> 对每个 LensEffect:
   │      visibility = occluder->collides(sunPos, camPos, le)
   │      le.tick(dTime, visibility)
   │      le.draw()   (按 OcclusionLevels 选光斑)
   └─> endOcclusionTests()
```

### 15.1 日出过渡的状态变化示例

以 `gameTime` 从 5.5h（黎明前）到 6.5h（日出后）为例：

| gameTime | sunColour | ambientColour | fogColour | useMoonColour_ | SkyGradientDome | StarDome | 太阳位置 |
|---|---|---|---|---|---|---|---|
| 5.5h | 深蓝 (0.1,0.1,0.3) | 冷蓝 (0.15,0.18,0.25) | 深灰 (0.2,0.2,0.25) | true（月亮主光） | 暗夜渐变 | alpha=1 | 地平线下 |
| 6.0h | 橙红 (0.8,0.4,0.2) | 暖橙 (0.3,0.25,0.2) | 浅橙 (0.5,0.4,0.35) | false（切换太阳） | 日出渐变 | alpha→0 | 地平线 |
| 6.5h | 亮白 (1.0,0.95,0.85) | 亮白 (0.5,0.5,0.5) | 浅蓝 (0.6,0.65,0.7) | false | 白天渐变 | alpha=0 | 升起 |

切换瞬间（`useMoonColour_` 由 true→false），`OutsideLighting::mainLightDir()` 从月亮方向切换到太阳方向，所有采样该值的子系统（云、雨、植被）自动跟随。

---

## 16. 文件清单（已阅读 42 个核心文件）

### 16.1 EnviroMinder 与核心

| 文件 | 角色 |
|---|---|
| [enviro_minder.hpp](file:///workspace/src/lib/romp/enviro_minder.hpp) | 环境管理器声明 |
| [enviro_minder.cpp](file:///workspace/src/lib/romp/enviro_minder.cpp) | tick/drawHind/drawFore/decideLightingAndFog/activate |

### 16.2 时间与天空

| 文件 | 角色 |
|---|---|
| [time_of_day.hpp](file:///workspace/src/lib/romp/time_of_day.hpp) | 时间 + OutsideLighting + UpdateNotifier |
| [sun_and_moon.hpp](file:///workspace/src/lib/romp/sun_and_moon.hpp) | 太阳月亮公告板 + 太阳光晕 |
| [sky_gradient_dome.hpp](file:///workspace/src/lib/romp/sky_gradient_dome.hpp) | Mie/Rayleigh 天空渐变穹顶 |
| [star_dome.hpp](file:///workspace/src/lib/romp/star_dome.hpp) | 星空穹顶 |
| [sky_dome_occluder.hpp](file:///workspace/src/lib/romp/sky_dome_occluder.hpp) | 太阳光晕遮挡查询 |
| [sky_dome_shadows.hpp](file:///workspace/src/lib/romp/sky_dome_shadows.hpp) | 天空盒云阴影 |
| [sky_light_map.hpp](file:///workspace/src/lib/romp/sky_light_map.hpp) | 天空光照图 + IContributor |

### 16.3 天气与雾

| 文件 | 角色 |
|---|---|
| [weather.hpp](file:///workspace/src/lib/romp/weather.hpp) | 风与阵风 |
| [rain.hpp](file:///workspace/src/lib/romp/rain.hpp) | 雨 + 雨溅粒子 + 雾发射 |
| [lightning.hpp](file:///workspace/src/lib/romp/lightning.hpp) | 闪电 |
| [fog_controller.hpp](file:///workspace/src/lib/romp/fog_controller.hpp) | 全局雾 + 雾发射器 |

### 16.4 光照与光效

| 文件 | 角色 |
|---|---|
| [light_map.hpp](file:///workspace/src/lib/romp/light_map.hpp) | 光照图基类 + EffectLightMap |
| [lens_effect.hpp](file:///workspace/src/lib/romp/lens_effect.hpp) | 镜头光晕 + FlareData |
| [lens_effect_manager.hpp](file:///workspace/src/lib/romp/lens_effect_manager.hpp) | 光晕管理器单例 |
| [photon_occluder.hpp](file:///workspace/src/lib/romp/photon_occluder.hpp) | 光子遮挡器接口 |
| [z_attenuation_occluder.hpp](file:///workspace/src/lib/romp/z_attenuation_occluder.hpp) | Z 衰减遮挡器 |
| [environment_cube_map.hpp](file:///workspace/src/lib/romp/environment_cube_map.hpp) | 环境立方体贴图 |
| [flash_bang_effect.hpp](file:///workspace/src/lib/romp/flash_bang_effect.hpp) | 闪光弹效果 |
| [py_chunk_light.hpp](file:///workspace/src/lib/romp/py_chunk_light.hpp) | 脚本 omni 光 |

### 16.5 植被与水

| 文件 | 角色 |
|---|---|
| [flora.hpp](file:///workspace/src/lib/romp/flora.hpp) | 植被系统主类 |
| [ecotype.hpp](file:///workspace/src/lib/romp/ecotype.hpp) | 生态类型 |
| [water.hpp](file:///workspace/src/lib/romp/water.hpp) | 水面 + WaterCell + Waters |
| [water_simulation.hpp](file:///workspace/src/lib/romp/water_simulation.hpp) | 水波模拟纹理 |
| [sea.hpp](file:///workspace/src/lib/romp/sea.hpp) | 大海 |

### 16.6 Occluder

| 文件 | 角色 |
|---|---|
| [terrain_occluder.hpp](file:///workspace/src/lib/romp/terrain_occluder.hpp) | 地形遮挡 |
| [z_buffer_occluder.hpp](file:///workspace/src/lib/romp/z_buffer_occluder.hpp) | Z-buffer 遮挡 |

### 16.7 控制台与字体

| 文件 | 角色 |
|---|---|
| [xconsole.hpp](file:///workspace/src/lib/romp/xconsole.hpp) | 控制台基类 |
| [console_manager.hpp](file:///workspace/src/lib/romp/console_manager.hpp) | 控制台管理器单例 |
| [font.hpp](file:///workspace/src/lib/romp/font.hpp) | 字体 |

### 16.8 UI/工具与杂项

| 文件 | 角色 |
|---|---|
| [progress.hpp](file:///workspace/src/lib/romp/progress.hpp) | 加载进度 |
| [custom_mesh.hpp](file:///workspace/src/lib/romp/custom_mesh.hpp) | 自定义网格模板 |
| [geometrics.hpp](file:///workspace/src/lib/romp/geometrics.hpp) | 几何工具 |
| [lod_settings.hpp](file:///workspace/src/lib/romp/lod_settings.hpp) | LOD 设置 |
| [handle_pool.hpp](file:///workspace/src/lib/romp/handle_pool.hpp) | 句柄池 |
| [occlusion_query_helper.hpp](file:///workspace/src/lib/romp/occlusion_query_helper.hpp) | 异步遮挡查询 |
| [distortion.hpp](file:///workspace/src/lib/romp/distortion.hpp) | 屏幕扭曲 |
| [texture_feeds.hpp](file:///workspace/src/lib/romp/texture_feeds.hpp) | 命名纹理流 |
| [frame_logger.hpp](file:///workspace/src/lib/romp/frame_logger.hpp) | 帧日志 |
| [dapple.hpp](file:///workspace/src/lib/romp/dapple.hpp) | 斑驳光影 |
| [watermark.hpp](file:///workspace/src/lib/romp/watermark.hpp) | 水印编解码 |

---

## 17. 设计要点总结

| 维度 | 设计 |
|---|---|
| **中心聚合** | `EnviroMinder` 把所有环境子系统聚合成 per-space 对象，提供统一 tick/draw 入口 |
| **脚本驱动** | `Vector4Provider` 控制器（天气/光照/雾）+ `PyObjectPlus` 子系统（Weather）+ 一批 `Py*` 绑定，让脚本实时调制氛围 |
| **单例 vs per-space** | 全局状态（雾、光晕、水、纹理流、扭曲、控制台）用单例；空间相关（EnviroMinder、Flora）per-space，激活互斥 |
| **硬件能力级联** | Occluder、天空绘制时机（前/后）、Distortion 支持都按显卡能力分级降级 |
| **贡献者模式** | `SkyLightMap::IContributor` 让 `SkyDomeShadows`、`Dapple` 等向同一光照图贡献内容 |
| **异步/分摊** | `EnvironmentCubeMap` 分摊更新立方体面、`OcclusionQueryHelper` 多帧平均、`Flora` 后台加载 Ecotype、`Water` 按距离激活 cell |
| **数据驱动** | `TimeOfDay` 用 `LinearAnimation` 关键帧、`Water` 用 `WaterState` 结构、`Sea` 全参数化，均可由 XML 配置 |
| **雾发射器** | `FogController::Emitter` 让天空、雨等动态贡献局部雾，`commitFogToDevice` 统一提交 |
| **RAII 作用域** | `SkyBoxScopedSetup`、`CameraPlanesSetter`、`beginClipPlaneBiasDraw` 保证渲染状态正确恢复 |

romp 体现了 BigWorld 客户端“环境氛围一体化管理 + 脚本可调制 + 硬件能力自适应”的设计哲学，是连接 `moo`（底层渲染）、`chunk`（场景）、`terrain`（地形）、`particle`（粒子）的渲染编排层。
