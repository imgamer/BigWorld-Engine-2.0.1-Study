# 17. 客户端 GUI 与 UI 栈：ashes / guimanager / guitabs / Scaleform / web_render / input

> 本文分析 BigWorld Engine 2.0.1 客户端「在游戏画面之上叠加 UI」的完整技术栈。BigWorld 的 UI 并非一个单一的库，而是按职责分成了若干个相互正交的子模块：底层 `ashes` 提供「把带纹理的矩形/文字/Flash 绘制到屏幕」的能力；`guimanager` 在 MFC 编辑器侧提供「菜单/工具条/命令项」的抽象；`guitabs` 提供「可停靠/可撕离的分页面板」；`scaleform` 把 Scaleform GFx Flash 引擎接入 `ashes`；`web_render` 把 Mozilla/Gecko 渲染的网页变成一张可供 `ashes` 使用的纹理；`input` 把键盘/鼠标/手柄/IME 事件喂给以上所有消费者。

## 0. 目录结构与职责定位

需要先澄清一个易混淆点：源码 `src/lib/controls/` 目录里**并不是** `GUIComponent`/`GUIShader` 那一族类（任务描述把它们归到了 `controls/`，但实际代码并非如此）。`controls/` 实际是编辑器侧的 **Win32/MFC 控件集**（`slider.hpp`、`image_button.hpp`、`edit_numeric.hpp`、`search_field.hpp`、`sizer.hpp`、`memdc.hpp` 等），服务于 WorldEditor、UAL 等基于 MFC 的工具对话框。

真正承载「游戏内 2D UI 渲染」的 `*GUIComponent*` / `*GUIShader*` 类族位于 [src/lib/ashes/](file:///workspace/src/lib/ashes/)（见第 1 节）。下表是各模块的真实归属：

| 目录 | 模块性质 | 关键基类/Singleton | 服务对象 |
|------|----------|--------------------|----------|
| [ashes/](file:///workspace/src/lib/ashes/) | 游戏内 2D UI 渲染库（moo 的轻量 GDI 包装 + GUI 组件/着色器） | `SimpleGUI`、`SimpleGUIComponent`、`GUIShader` | 运行时客户端（也供 tools） |
| [controls/](file:///workspace/src/lib/controls/) | MFC 编辑器控件集 | `slider`、`image_button`、`search_field`、`sizer` | WorldEditor / UAL 等工具对话框 |
| [guimanager/](file:///workspace/src/lib/guimanager/) | 编辑器菜单/工具条/命令项抽象（namespace `GUI`） | `GUI::Manager`、`GUI::Item`、`GUI::Functor` | MFC 编辑器（菜单、工具条） |
| [guitabs/](file:///workspace/src/lib/guitabs/) | 可停靠/撕离分页面板框架（namespace `GUITABS`） | `GUITABS::Manager`、`Dock`、`Panel` | 编辑器主框架（属性页、UAL 面板） |
| [scaleform/](file:///workspace/src/lib/scaleform/) | Scaleform GFx (Flash) 集成（受 `SCALEFORM_SUPPORT` 编译开关保护） | `Scaleform::Manager`、`FlashGUIComponent` | 客户端（可选） |
| [web_render/](file:///workspace/src/lib/web_render/) | Mozilla/Gecko 网页渲染为纹理 | `WebPage`、`MozillaWebPageManager` | 客户端（商店/新闻/公告） |
| [input/](file:///workspace/src/lib/input/) | 输入设备 + IME | `InputDevices`、`InputHandler`、`IME` | 全体 UI 消费者 |

### 模块依赖图

```
                         ┌──────────────┐
                         │   input/     │  InputDevices(Singleton)
                         │ InputHandler │  KeyEvent/MouseEvent/AxisEvent
                         │   IME        │  IMEEvent
                         └──────┬───────┘
                                │ processEvents(handler)
                ┌───────────────┼────────────────────────┐
                ▼               ▼                        ▼
         ┌──────────┐    ┌──────────────┐         ┌──────────────┐
         │ ashes/   │    │ guimanager/  │         │  scaleform/  │
         │SimpleGUI │    │ GUI::Manager │         │   Manager    │
         │(InputHdl)│    │  (MFC菜单)   │         │ onKeyEvent/  │
         └────┬─────┘    └──────┬───────┘         │  onMouse     │
              │                 │                  └──────┬───────┘
              │ update/draw     │ act()/update()          │ FlashGUIComponent
              ▼                 ▼                         ▼
         ┌────────────────────────────────────────────────────┐
         │                 moo/ (D3D9 设备/EffectMaterial)      │
         └────────────────────────────────────────────────────┘
              ▲                                 ▲
              │ WebPage.texture (BaseTexture)   │ PyTextureProvider
         ┌────┴──────────────┐
         │   web_render/     │  MozillaWebPageManager(后台线程)
         │ WebPageProvider   │  → 纹理 → SimpleGUIComponent.texture
         └───────────────────┘
```

`ashes` 是运行时 UI 的核心；`scaleform` 和 `web_render` 都通过「提供纹理 / 提供 SimpleGUIComponent 子类」的方式接入 `ashes`；`guimanager` 与 `guitabs` 则是 MFC 编辑器侧的 UI 框架，与运行时 `ashes` 共享 `input` 与 `moo`，但本身不直接消费 `SimpleGUIComponent`。

---

## 1. ashes：游戏内 2D UI 渲染库

`ashes`（`src/lib/ashes/`）是 BigWorld 客户端「在 3D 画面之上画 UI」的基础库。它的命名暗示了「尘埃／余烬」——是 `moo`（图形底层）之上的一层薄包装。它同时承担两个职责：

1. **GUI 组件体系**：`SimpleGUIComponent` 及其派生类，构成一棵可脚本化的组件树；
2. **GDI 风格的简单绘制**：`simple_gui.hpp` / `mouse_cursor.hpp` 等提供屏幕坐标、像素↔裁剪空间换算、鼠标光标管理。

### 1.1 SimpleGUI —— 组件树管理器（Singleton + InputHandler）

[simple_gui.hpp](file:///workspace/src/lib/ashes/simple_gui.hpp) 中的 `SimpleGUI` 是整个 GUI 子系统的入口：

```cpp
class SimpleGUI : public InputHandler, public Singleton< SimpleGUI >
{
public:
    typedef std::vector< SimpleGUIComponent* > FocusList;
    typedef std::vector< SimpleGUIComponent* > Components;

    void addSimpleComponent( SimpleGUIComponent& c );
    void removeSimpleComponent( SimpleGUIComponent& c );
    void update( float dTime );
    void draw();

    // 输入焦点列表（每类一份，按添加顺序遍历）
    void addInputFocus( SimpleGUIComponent* c );
    void addMouseButtonFocus( SimpleGUIComponent* c );
    void addMouseCrossFocus( SimpleGUIComponent* c );
    void addMouseMoveFocus( SimpleGUIComponent* c );
    void addMouseDragFocus( SimpleGUIComponent* c );
    void addMouseDropFocus( SimpleGUIComponent* c );

    bool handleKeyEvent( const KeyEvent & event );
    bool handleMouseEvent( const MouseEvent & event );
    bool handleAxisEvent( const AxisEvent & event );

    bool processClickKey( const KeyEvent & event );
    bool processDragKey( const KeyEvent & event );
    bool processMouseMove( const MouseEvent & event );
    bool processDragMove( const MouseEvent & event );

    // 裁剪区栈（PC 端用 viewport + 视矩阵实现）
    bool pushClipRegion( SimpleGUIComponent& c );
    bool pushClipRegion( const Vector4& );
    void popClipRegion();
    const Vector4& clipRegion();

    // 像素/裁剪空间换算
    void pixelRangesToClip( float w, float h, float* retW, float* retH );
    void clipRangesToPixel( float w, float h, float* retW, float* retH );
    float screenWidth() const;
    float screenHeight() const;
    Vector2 pixelToClip() const;
    // ...
};
```

关键设计点：

- **`SimpleGUI` 本身是一个 `InputHandler`**（见第 6 节），客户端把它注册到 `InputDevices` 后，输入事件由它派发给当前各焦点列表中的组件。
- **六类输入焦点列表**：`mouseButtonFocusList_`、`crossFocusList_`（鼠标穿越/进出）、`dragFocusList_`、`dropFocusList_` 等。组件通过 `focus(true)` / `mouseButtonFocus(true)` 等属性把自己加入对应列表。事件派发使用「最近命中测试」`closestHitTest()`，按 Z 序（`drawOrder_`）挑选最上层组件。
- **裁剪区栈 `clipStack_`**：在 PC 端因为用 viewport + view matrix 实现裁剪，所以保留了 `originalView_` 与 `clipFixer_` 矩阵；`commitClipRegion()` 把栈顶区域真正下发到设备。
- **分辨率变更**：`hasResolutionChanged()` 与 `realResolutionCounter_` 让组件能在分辨率切换后重算布局（例如 `TextGUIComponent` 的字体大小匹配）。
- **拖拽**：`DragInfoPtr dragInfo_` + `dragDistanceSqr_` 记录拖拽阈值，超过阈值才触发 `handleDragStartEvent`。

Python 侧通过模块 `GUI` 暴露：`GUI.addRoot` / `GUI.delRoot` / `GUI.reSort` / `GUI.roots`（`PY_MODULE_STATIC_METHOD_DECLARE`）。

### 1.2 SimpleGUIComponent —— 所有 GUI 组件的基类

[simple_gui_component.hpp](file:///workspace/src/lib/ashes/simple_gui_component.hpp) 是整个 UI 栈最重要的类。注意：BigWorld 中**没有**一个叫 `GUIComponent` 的独立基类——任务描述里提到的「`GUIComponent` 基类」实际上就是 `SimpleGUIComponent` 本身（所有其它 GUI 组件都直接派生自它）。它同时继承 `PyObjectPlusWithWeakReference`（脚本化 + 弱引用）与 `Aligned`（SIMD 对齐）：

```cpp
class SimpleGUIComponent : public PyObjectPlusWithWeakReference, public Aligned
{
    Py_Header( SimpleGUIComponent, PyObjectPlusWithWeakReference )
public:
    SimpleGUIComponent( const std::string& textureName, PyTypePlus * pType = &s_type_ );
    static void init( DataSectionPtr config );
    static void fini();

    // 渲染主循环三件套
    virtual void update( float dTime, float relParentWidth, float relParentHeight );
    virtual void applyShaders( float dTime );
    virtual void draw( bool reallyDraw, bool overlay = true );
    virtual void addAsSortedDrawItem( const Matrix& worldTransform );

    // 子树
    bool addChild( const std::string & name, SimpleGUIComponent* child );
    bool removeChild( SimpleGUIComponent* child );
    SimpleGUIComponentPtr child( const std::string& name );
    SimpleGUIComponentPtr parent();

    // 输入（全是 virtual，默认实现转发到 Python 脚本对象）
    virtual bool handleKeyEvent( const KeyEvent & event );
    virtual bool handleMouseButtonEvent( const KeyEvent & event );
    virtual bool handleMouseEvent( const MouseEvent & event );
    virtual bool handleAxisEvent( const AxisEvent & event );
    virtual bool handleMouseEnterEvent / handleMouseLeaveEvent / handleMouseClickEvent
                 / handleDragStartEvent / handleDragStopEvent
                 / handleDragEnterEvent / handleDragLeaveEvent / handleDropEvent( ... );
    bool hitTest( const Vector2 & mousePosition ) const;

    // 着色器管线
    void addShader( const std::string & name, GUIShader* shader );
    void removeShader( GUIShader* shader );
    virtual void applyShader( GUIShader& shader, float dTime );
    GUIVertex* vertices( int* numVertices );  // 给 shader 用

    // 持久化
    virtual bool load( DataSectionPtr pSect, const std::string& ownerName, LoadBindings & bindings );
    virtual void save( DataSectionPtr pSect, SaveBindings & bindings );
    virtual void bound();
    // ...
};
```

#### 1.2.1 工厂宏与组件注册

`SimpleGUIComponent` 用 `NamedObject` 实现了一个类型工厂，使子类能通过 `.gui` DataSection 反序列化时按类名构造：

```cpp
typedef NamedObject<SimpleGUIComponent * (*)()> GUIComponentFactory;

#define COMPONENT_FACTORY_DECLARE( CONSTRUCT )                  \
    static GUIComponentFactory s_factory;                       \
    virtual GUIComponentFactory & factory() { return s_factory; } \
    static SimpleGUIComponent * s_create() { return new CONSTRUCT; }

#define COMPONENT_FACTORY( CLASS )                              \
    GUIComponentFactory CLASS::s_factory( #CLASS, CLASS::s_create );
```

每个子类在头文件里写 `COMPONENT_FACTORY_DECLARE(...)`，在 cpp 里写 `COMPONENT_FACTORY(Class)`，即可被 `load()` 按字符串类名重建。

#### 1.2.2 位置/尺寸的「模式」枚举

这是 `SimpleGUIComponent` 最容易被忽略但最关键的设计——位置和尺寸各有三种模式：

```cpp
typedef enum { POSITION_MODE_CLIP, POSITION_MODE_PIXEL, POSITION_MODE_LEGACY } ePositionMode;
typedef enum { SIZE_MODE_CLIP,     SIZE_MODE_PIXEL,     SIZE_MODE_LEGACY }     eSizeMode;
```

- `CLIP`：以裁剪空间（-1..1）表示，相对父组件中心；
- `PIXEL`：以像素表示，相对父组件左上角；
- `LEGACY`：旧式「相对屏幕」语义，保留用于兼容旧资源。

锚点 `eHAnchor`（LEFT/CENTER/RIGHT）、`eVAnchor`（TOP/CENTER/BOTTOM）决定组件如何「挂」在 position 上。

#### 1.2.3 材质 FX 与过滤

```cpp
typedef enum {
    FX_ADD, FX_BLEND, FX_BLEND_COLOUR, FX_BLEND_INVERSE_COLOUR, FX_SOLID,
    FX_MODULATE2X, FX_ALPHA_TEST, FX_BLEND_INVERSE_ALPHA, FX_BLEND2X, FX_ADD_SIGNED
} eMaterialFX;
typedef enum { FT_NONE, FT_POINT, FT_LINEAR } eFilterType;
```

这些值在 `buildMaterial()` 里映射到 `Moo::EffectMaterial` 的 technique（`s_techniqueTable` 缓存 technique handle）。

#### 1.2.4 关键内部状态

| 成员 | 作用 |
|------|------|
| `texture_` / `textureProvider_` | `Moo::BaseTexturePtr` 静态纹理，或 `PyTextureProviderPtr` 动态纹理（动画/网页/渲染目标） |
| `blueprint_` / `vertices_` / `indices_` | 顶点缓冲蓝图与运行时副本（shader 会修改 `vertices_`） |
| `material_` | `Moo::EffectMaterialPtr`，由 `buildMaterial()` 创建 |
| `shaders_` | `StringMap<GUIShaderPtr>`，按名查找 |
| `children_` / `childOrder_` | 子组件表 + 绘制顺序索引（`DepthCompare` 排序） |
| `runTimeColour_` / `runTimeTransform_` | 每帧由 colour/matrix shader 修改的动态值；`runTimeTransform_` 绘制后存最终 world-view-projection |
| `runTimeClipRegion_` | 绘制时算出的动态裁剪区，留作下一帧 hit-test |
| `guiTreeCookie_` | 树结构变更标记，用于缓存失效 |
| `focus_` / `mouseButtonFocus_` / `moveFocus_` / `crossFocus_` / `dragFocus_` / `dropFocus_` | 六类输入焦点开关 |

#### 1.2.5 渲染三阶段：update → applyShaders → draw

```
SimpleGUI::update(dTime)
  └─ 对每个 root: update(dTime, relW, relH)
       └─ layout() 计算 x,y,w,h（按 position/size mode + anchor）
       └─ buildMesh() / tile() 重建顶点（脏时）
       └─ updateChildren(dTime, relW, relH)

SimpleGUI::draw()
  └─ 对每个 root: draw(reallyDraw, overlay)
       └─ drawSelf(): applyShaders(dTime) → 提交 material/顶点到 Moo
       └─ drawChildren(reallyDraw, overlay)（按 childOrder_ 排序）
```

`applyShaders()` 遍历 `shaders_`，对每个 shader 调 `shader.processComponent(*this, dTime)`；shader 可修改 `vertices_`（顶点位置/UV/颜色）或 `runTimeColour_`/`runTimeTransform_`。

### 1.3 GUIVertexFormat

[gui_vertex_format.hpp](file:///workspace/src/lib/ashes/gui_vertex_format.hpp) 极短，仅一行实质内容：

```cpp
#include "moo/vertex_formats.hpp"
typedef Moo::VertexXYZDUV2	GUIVertex;
```

即 GUI 顶点 = 「位置 XYZ + Diffuse 颜色 + 两套 UV」。第二套 UV 用于 tile/matrix shader 等高级效果。`TextGUIComponent` 用 `CustomMesh<GUIVertex>` 持有文字三角网。

### 1.4 GUIShader 体系（着色器管线）

GUI「着色器」并非 GPU shader，而是**每帧在 CPU 端修改组件顶点/颜色/矩阵的过滤器**。基类 [gui_shader.hpp](file:///workspace/src/lib/ashes/gui_shader.hpp)：

```cpp
class GUIShader : public PyObjectPlus
{
    Py_Header( GUIShader, PyObjectPlus )
public:
    // 返回 true 继续处理子组件，false 中止
    virtual bool processComponent( SimpleGUIComponent& component, float dTime );
    virtual void value( float v ) = 0;   // 所有 shader 至少有一个 value
    virtual bool load( DataSectionPtr pSect ) { return true; }
    virtual void save( DataSectionPtr pSect ) { }
};

// 工厂宏，与 COMPONENT_FACTORY 同构
typedef NamedObject<GUIShader * (*)()> GUIShaderFactory;
#define SHADER_FACTORY_DECLARE( CONSTRUCT ) ...
#define SHADER_FACTORY( CLASS ) ...
```

派生着色器一览（均位于 `ashes/`）：

| 类 | 文件 | 作用 | 关键属性 |
|----|------|------|----------|
| `ClipGUIShader` | [clip_gui_shader.hpp](file:///workspace/src/lib/ashes/clip_gui_shader.hpp) | 把组件按某方向裁掉一部分（血条/进度条） | `mode`(RIGHT/UP/DOWN/LEFT)、`value`、`speed`、`delay`、`slant`、`dynValue`(`Vector4Provider`) |
| `AlphaGUIShader` | [alpha_gui_shader.hpp](file:///workspace/src/lib/ashes/alpha_gui_shader.hpp) | 修改 alpha，可从一边渐变到另一边 | `mode`(FADE_ALL/RIGHT/UP/DOWN/LEFT)、`start`/`stop`/`alpha`/`speed` |
| `ColourGUIShader` | [colour_gui_shader.hpp](file:///workspace/src/lib/ashes/colour_gui_shader.hpp) | 在 start/middle/end 三色间按 `value` 插值，或由 `colourProvider`(`Vector4Provider`) 提供 | `start`/`middle`/`end`、`value`、`speed`、`colourProvider`、`LinearAnimation<Vector4>` |
| `MatrixGUIShader` | [matrix_gui_shader.hpp](file:///workspace/src/lib/ashes/matrix_gui_shader.hpp) | 用矩阵变换组件四角，支持 `BlendTransform` 平滑过渡 | `target`(`MatrixProvider`)、`eta`、`blend` |

`ClipGUIShader` 的常量数组语义很有代表性：

```cpp
//constants for clip shader :
// 1 ) the desired T-value ( the amount of the vertices left after clipping )
// 2 ) the time ( the time required to move T-value to new value )
const float* getConstants( void ) const;
void constants( float param, float paramPerSeconds = 1.f );
void value( float v );
void speed( float t );
void delay( float t );
```

即设 `value` 后，shader 会在 `speed` 秒内把当前 T 值线性插值到目标，实现「血条平滑减少」这类效果。

### 1.5 具体 GUI 组件一览

下表是 `ashes` 里所有 `*GUIComponent`（外加 `src/client/latency_gui_component`）的派生关系：

```
PyObjectPlusWithWeakReference, Aligned
└── SimpleGUIComponent                       （基类：纹理矩形）
    ├── TextGUIComponent                     （文字）
    ├── FrameGUIComponent                    （边框：4 角 + 4 边 + 背景）
    ├── FrameGUIComponent2                   （单次 draw call 的 9-slice 边框）
    ├── WindowGUIComponent                   （窗口：层次变换 + 裁剪 + 滚动）
    ├── ConsoleGUIComponent                  （已弃用：可编辑控制台）
    ├── MeshGUIAdaptor                       （把 PyAttachment 当 GUI 画）
    ├── LatencyGUIComponent  (src/client)    （掉线指示）
    ├── bounding_box_gui_component           （画 BoundingBox）
    ├── gobo_component                       （gobo/遮片）
    └── graph_gui_component                  （数据曲线图）
```

#### 1.5.1 TextGUIComponent

[text_gui_component.hpp](file:///workspace/src/lib/ashes/text_gui_component.hpp)：用 `romp/font.hpp` + `font_metrics.hpp` 把 `std::wstring label_` 转成 `CustomMesh<GUIVertex>`。支持 `multiline`、`colourFormatting`（颜色标记）、`stringWidth`/`stringDimensions`。`textureName` 被显式禁用（`virtual void textureName(...)` 屏蔽）。`recalculate()` 在文本/字体/分辨率变化时重建 mesh。

#### 1.5.2 FrameGUIComponent

[frame_gui_component.hpp](file:///workspace/src/lib/ashes/frame_gui_component.hpp)：经典「九宫格」边框的早期实现，但用的是**三张纹理 + 八个子组件**：

```cpp
SimpleGUIComponent* corners_[4];  // 角纹理四等分
SimpleGUIComponent* edges_[4];    // 边纹理（底边正向，其余镜像/旋转）
```

构造需 `backgroundTextureName`、`frameTextureName`、`edgeTextureName` + `tileWidth/tileHeight`。

#### 1.5.3 FrameGUIComponent2

[frame_gui_component2.hpp](file:///workspace/src/lib/ashes/frame_gui_component2.hpp)：**单次 draw call** 的边框，要求纹理按特定布局排布（参见 CCM 文档）。它重写 `buildMesh()`，用 `setQuad()` 一次性生成所有四边形：

```cpp
virtual void buildMesh();
void updateVertices( GUIVertex* v, float relativeParentWidth, float relativeParentHeight );
GUIVertex* setQuad( GUIVertex* pFirstVert,
    float x1,float y1, float x2,float y2, float x3,float y3, float x4,float y4,
    float u1,float v1, float u2,float v2, float u3,float v3, float u4,float v4 );
```

相比 `FrameGUIComponent` 的 9 个子组件 9 次 draw call，这是为性能优化的版本。

#### 1.5.4 WindowGUIComponent

[window_gui_component.hpp](file:///workspace/src/lib/ashes/window_gui_component.hpp)：真正的「窗口」——支持层次变换、裁剪、滚动：

```cpp
Matrix scrollTransform_;
Matrix anchorTransform_;
Vector2 scroll_;
Vector2 scrollMin_;
Vector2 scrollMax_;
```

子组件若 position mode 为 `CLIP`，则以本窗口为裁剪空间（(-1,-1) 为左下、(1,1) 为右上）；若为 `PIXEL`，则相对窗口左上角。`nearestRelativeParent()` 让子组件找到最近的「相对父」以计算尺寸。

#### 1.5.5 ConsoleGUIComponent

[console_gui_component.hpp](file:///workspace/src/lib/ashes/console_gui_component.hpp)：已弃用（注释明说「deprecated in favour of script based solutions that take advantage of multiline TextGUIComponents」）。包装 `CallbackConsole`，提供可编辑行、闪烁光标、字体大小匹配（按分辨率从 `resource.xml` 的 `system/consoleFonts` 选字体）。

#### 1.5.6 MeshGUIAdaptor

[mesh_gui_adaptor.hpp](file:///workspace/src/lib/ashes/mesh_gui_adaptor.hpp)：把任意 `PyAttachment`（模型/粒子等）当作 GUI 组件来画。同时实现 `MatrixLiaison`（供 attachment 取世界矩阵）：

```cpp
class MeshGUIAdaptor : public SimpleGUIComponent, MatrixLiaison
{
    SmartPointer<PyAttachment> attachment_;
    MatrixProviderPtr transform_;
    static Moo::LightContainerPtr s_lighting_;
    // ...
};
```

**重要警告**（来自类注释）：它在 GUI 时段绘制，**Z-Buffer 不可用**，只适合画「扁平、加性/alpha 混合」的自定义 mesh（如异形 GUI 边框）。要画角色应改用 `PyModelRenderer`/`PySceneRenderer`。

#### 1.5.7 LatencyGUIComponent

[src/client/latency_gui_component.hpp](file:///workspace/src/client/latency_gui_component.hpp)：客户端专用，断线时显示 "Offline"。内部持有一个 `TextGUIComponent* txt_`，根据连接状态切换 `visible`。

### 1.6 SimpleGui / mouse_cursor / gui_attachment

- [simple_gui.hpp](file:///workspace/src/lib/ashes/simple_gui.hpp) 内含 `extern int GUI_token;`（模块初始化令牌）与 `SimpleGUI`（见 1.1）。
- [mouse_cursor.hpp](file:///workspace/src/lib/ashes/mouse_cursor.hpp)：`MouseCursor` 维护光标位置/可见性，`SimpleGUI::internalMouseCursor()` 惰性创建。
- [gui_attachment.hpp](file:///workspace/src/lib/ashes/gui_attachment.hpp)：`GUIAttachment`，把 GUI 组件作为一种 attachment 挂到场景对象上（较少用）。

---

## 2. guimanager：编辑器菜单/工具条/命令抽象

[guimanager/](file:///workspace/src/lib/guimanager/) 是 **MFC 编辑器侧**的命令抽象层（namespace `GUI`）。它把「菜单项 / 工具条按钮 / 快捷键」统一抽象为 `Item` 树，命令的执行/更新/文本通过可插拔的 `Functor` 体系分发到 C++ / Python / 选项表 / DataSection。

> 注意：此模块与运行时 `ashes` **不直接交互**——它面向 WorldEditor 等 MFC 工具，用 Win32 `HMENU`/`HIMAGELIST` 而非 `moo` 绘制。但二者共享 `input` 与 `resmgr`。

### 2.1 Manager（Singleton）

[gui_manager.hpp](file:///workspace/src/lib/guimanager/gui_manager.hpp)：

```cpp
class Manager : public InitSingleton<Manager>, public Item
{
    std::map<std::string,std::set<SubscriberPtr>> subscribers_;
public:
    using Item::add;
    void add( SubscriberPtr subscriber );
    void remove( SubscriberPtr subscriber );
    virtual ItemPtr operator()( const std::string& path );  // 按路径找 Item

    void act( unsigned short commandID );     // 执行命令
    void update( unsigned short commandID );  // 更新命令状态
    void update();
    void changed( ItemPtr item );             // 通知订阅者

    FunctorManager& functors();
    CppFunctor& cppFunctor();
    OptionFunctor& optionFunctor();
    PythonFunctor& pythonFunctor();
    DataSectionFunctor& dataSectionFunctor();
    BitmapManager& bitmaps();
private:
    FunctorManager functorManager_;
    CppFunctor cppFunctor_;
    OptionFunctor optionFunctor_;
    PythonFunctor pythonFunctor_;
    DataSectionFunctor dataSectionFunctor_;
    BitmapManager bitmapManager_;
};
```

`Manager` 自己也是一个 `Item`（根节点），`Subscriber` 订阅某 root 路径，`changed()` 回调通知 UI 刷新。

### 2.2 Item —— 命令树节点

[gui_item.hpp](file:///workspace/src/lib/guimanager/gui_item.hpp)：

```cpp
class Item : public SafeReferenceCount
{
    std::string type_, name_, displayName_, description_, icon_,
                shortcutKey_, action_, updater_, imports_;
    std::map<std::string,std::string> values_;
    std::vector<Item*> parents_;       // 不持有（避免环）
    std::vector<ItemPtr> subitems_;
    unsigned short commandID_;          // Win32 命令 ID
public:
    void add( ItemPtr item );
    ItemPtr operator[]( const std::string& name );  // 按 values 名取值
    unsigned int update();   // 调用 updater functor
    bool act();              // 调用 action functor
    bool processInput( InputDevice* inputDevice );  // 快捷键
    ItemPtr findByCommandID( unsigned short commandID );
    static void registerType( ItemTypePtr itemType );
};
```

`ItemType` 是「类型策略」：`act`/`update`/`shortcutPressed`。`BasicItemType` 是默认实现。注册新类型后，菜单项的勾选/单选行为可由类型决定（如 `updateToggle`、`updateChoice`）。

### 2.3 Functor 体系

[gui_functor.hpp](file:///workspace/src/lib/guimanager/gui_functor.hpp) 定义抽象 `Functor`，含四类操作：

```cpp
class Functor
{
public:
    virtual const std::string& name() const = 0;
    virtual bool text( const std::string& textor, ItemPtr item, std::string& result ) = 0;
    virtual bool update( const std::string& updater, ItemPtr item, unsigned int& result ) = 0;
    virtual DataSectionPtr import( const std::string& importer, ItemPtr item ) = 0;
    virtual bool act( const std::string& action, ItemPtr item, bool& result ) = 0;
};
```

`FunctorManager` 用 `name` 字符串路由调用。四种内置实现：

| Functor | 文件 | 后端 |
|---------|------|------|
| `CppFunctor` | [gui_functor_cpp.hpp](file:///workspace/src/lib/guimanager/gui_functor_cpp.hpp) | C++ 成员函数（通过 `ActionMaker`/`TextorMaker`/`UpdaterMaker` 注册） |
| `PythonFunctor` | [gui_functor_python.hpp](file:///workspace/src/lib/guimanager/gui_functor_python.hpp) | Python 模块（默认 `UIExt`），按名字调用函数 |
| `OptionFunctor` | [gui_functor_option.hpp](file:///workspace/src/lib/guimanager/gui_functor_option.hpp) | `OptionMap`（键值对选项表） |
| `DataSectionFunctor` | [gui_functor_datasection.hpp](file:///workspace/src/lib/guimanager/gui_functor_datasection.hpp) | DataSection 节点 |

### 2.4 三个 Maker 模板

任务描述提到的 `GUIActionMaker`/`GUITextorMaker`/`GUIUpdaterMaker` 实际是这三个模板（命名前缀 `GUI` 是 namespace，不是 `GUITickler` 那种类名——代码中**没有** `GUITickler` 类）：

[gui_action_maker.hpp](file:///workspace/src/lib/guimanager/gui_action_maker.hpp)：

```cpp
template< typename T, int index = 0 >
struct ActionMaker: public Action
{
    bool (T::*func_)( ItemPtr item);
    ActionMaker( std::string names, bool (T::*func)( ItemPtr item ) )
        : func_( func )
    {
        // names 可用分隔符串多个名字，全部注册到 cppFunctor
        while (!names.empty()) {
            GUI::Manager::instance().cppFunctor().set( name, this );
            // ...
        }
    }
    virtual bool act( ItemPtr item ) { return (((T*)this)->*func_)( item ); }
};
```

[gui_textor_maker.hpp](file:///workspace/src/lib/guimanager/gui_textor_maker.hpp) 与 [gui_updater_maker.hpp](file:///workspace/src/lib/guimanager/gui_updater_maker.hpp) 结构相同，分别封装「返回 `std::string` 的文本 functor」和「返回 `unsigned int` 状态的更新 functor」。它们让 MFC 工具类用一行成员变量即可把方法注册为命令回调：

```cpp
class MyApp {
    GUI::ActionMaker<MyApp>  m_refresh_{ "ual.refresh", &MyApp::guiActionRefresh };
    GUI::UpdaterMaker<MyApp> m_layout_{  "ual.layout",  &MyApp::guiActionLayout };
};
```

### 2.5 Menu / Toolbar / Bitmap

- [gui_menu.hpp](file:///workspace/src/lib/guimanager/gui_menu.hpp)：`Menu : public Subscriber`，把 `Item` 子树渲染成 `HMENU`。按 item type 分派：`updateSeparator`/`updateAction`/`updateToggle`/`updateChoice`/`updateExpandedChoice`/`updateGroup`/`updateUnknownItem`。`buildMenuText` 拼接显示名 + 快捷键。
- [gui_toolbar.hpp](file:///workspace/src/lib/guimanager/gui_toolbar.hpp)：`Toolbar : public Subscriber` + `CGUIToolBar : public CToolBar`。维护三套 `HIMAGELIST`（`disabledImageList_`/`hotImageList_`/`normalImageList_`），按 item 填 `TBBUTTONINFO`。`createToolbars()` 静态方法从 DataSection 一次创建多个工具条。
- [gui_bitmap.hpp](file:///workspace/src/lib/guimanager/gui_bitmap.hpp)：`Bitmap` 持有 `HBITMAP`，类型用字符串（`"NORMAL"`/`"HOVER"`/`"DISABLED"`，可自由扩展）。`BitmapManager` 按 `name+type+size` 缓存。

### 2.6 GUIInputHandler / PyItem

- [gui_input_handler.hpp](file:///workspace/src/lib/guimanager/gui_input_handler.hpp)：`BigworldInputDevice`（用 BigWorld `keyDownTable`）、`Win32InputDevice`（用 Win32 `GetAsyncKeyState`）实现 `GUI::InputDevice::isKeyDown`，供 `Item::processInput` 做快捷键匹配。
- [py_gui_item.hpp](file:///workspace/src/lib/guimanager/py_gui_item.hpp)：`PyItem : public PyObjectPlus`，把 `GUI::ItemPtr` 暴露给 Python，支持 `child`/`childName`/`keys`/`values`/`items`/`createItem`/`deleteItem`/`copy`，可像字典一样 `item["name"]` 取值。

---

## 3. guitabs：可停靠/撕离分页面板框架

[guitabs/](file:///workspace/src/lib/guitabs/)（namespace `GUITABS`）是编辑器主框架用的「VS 风格」可停靠、可撕离、可分页的面板系统。WorldEditor 的属性页、UAL 面板等都作为 `Content` 注册进来。

[manager.hpp](file:///workspace/src/lib/guitabs/manager.hpp)：

```cpp
class Manager : public Singleton<Manager>, public ReferenceCount
{
public:
    bool registerFactory( ContentFactoryPtr factory );
    bool insertDock( CFrameWnd* mainFrame, CWnd* mainView );  // 把 dock 嵌入主框架
    void removeDock();
    PanelHandle insertPanel( const std::wstring & contentID, InsertAt insertAt, PanelHandle destPanel = 0 );
    bool removePanel( PanelHandle panel );
    void showPanel( ... );
    bool load( const std::wstring & fname = L"" );   // 加载布局
    bool save( const std::wstring & fname = L"" );   // 保存布局
    PanelHandle clone( PanelHandle content, int x, int y );
private:
    DockPtr dock_;
    DragManagerPtr dragMgr_;
    std::list<ContentFactoryPtr> factoryList_;
};
```

文件清单与职责：

| 文件 | 作用 |
|------|------|
| [manager.hpp](file:///workspace/src/lib/guitabs/manager.hpp) | 唯一对外入口，管理 dock 与工厂 |
| `content.hpp` / `content_container.hpp` / `content_factory.hpp` | 面板内容抽象基类、容器、工厂 |
| `dock.hpp` / `dock_node.hpp` / `docked_panel_node.hpp` / `main_view_node.hpp` / `splitter_node.hpp` | dock 树节点（splitter / panel / main view） |
| `panel.hpp` / `tab.hpp` / `tab_ctrl.hpp` | 面板与分页标签 |
| `floater.hpp` | 撕离后的浮动窗口 |
| `drag_manager.hpp` | 拖拽管理（撕离/停靠） |
| `nice_splitter_wnd.hpp` | 自绘分隔条 |

`Content` 子类（如 `UalDialog`）实现 `getContentID`/`getCWnd`/`clone`/`load`/`save`/`onClose` 等方法即可被 dock 接纳。

---

## 4. scaleform：Scaleform GFx (Flash) 集成

[scaleform/](file:///workspace/src/lib/scaleform/) 把 Scaleform 公司的 GFx Flash 引擎接入 BigWorld。整个模块受 [config.hpp](file:///workspace/src/lib/scaleform/config.hpp) 的编译开关保护：

```cpp
// 默认关闭，需购买 Scaleform 授权后开启
#define SCALEFORM_SUPPORT 0
#define SCALEFORM_IME 0   // 启用后会禁用 BigWorld 内置 IME
```

所有头文件都用 `#if SCALEFORM_SUPPORT` 包裹。下文涉及 `GFx*`/`GPtr`/`GRendererD3D9` 等均为 Scaleform SDK 类型。

### 4.1 Manager（Singleton + DeviceCallback）

[manager.hpp](file:///workspace/src/lib/scaleform/manager.hpp)：

```cpp
class Manager : public InitSingleton< Manager >, public Moo::DeviceCallback
{
public:
    virtual bool doInit();
    virtual bool doFini();
    // Moo::DeviceCallback：设备丢失/重建时回收/重建非托管资源
    void deleteUnmanagedObjects();
    void createUnmanagedObjects();
    void deleteManagedObjects();
    void createManagedObjects();

    void tick(float elapsedTime, uint32 frameCatchUpCount = 2);
    void draw();

    void createRenderer(DX::Device*, D3DPRESENT_PARAMETERS*, bool noSceneCalls, HWND hwnd);
    void resetDevice(DX::Device*, D3DPRESENT_PARAMETERS*);
    void destroyRenderer();
    GPtr<GRendererD3D9> pRenderer();
    Loader* pLoader();
    GPtr<Log> pLogger();
    GPtr<GFxDrawTextManager> pTextManager();

    void addMovieView(PyMovieView* obj);
    void removeMovieView(PyMovieView* obj);

    // 输入
    bool onKeyEvent(const KeyEvent &event);
    bool onMouse(int buttons, int nMouseWheelDelta, int xPos, int yPos);
    void enablePygfxMouseHandle(bool state);
    // ...
private:
    GFxSystem system_;
    Loader* pLoader_;
    GPtr<GRendererD3D9> pRenderer_;
    GPtr<GFxRenderConfig> pRenderConfig_;
    GPtr<GFxRenderStats> pRenderStats_;
    GPtr<GFxDrawTextManager> pDrawTextManager_;
    GPtr<Log> pLogger_;
    std::list<PyMovieView*> list_;   // 当前所有 movie view
};
```

`Manager` 实现 `Moo::DeviceCallback`，因此设备丢失（alt-tab、分辨率切换）时能正确回收 `GRendererD3D9` 的非托管资源。`tick` 推进所有 movie，`draw` 绘制所有 movie。

### 4.2 Loader —— .gfx/.swf 加载

[loader.hpp](file:///workspace/src/lib/scaleform/loader.hpp)：

```cpp
class FileOpener : public GFxFileOpener {
    virtual GFile* OpenFile(const char *pfilename, SInt flags, SInt mode);
};

class BWToGFxImageLoader : public GFxImageLoader {
    GImageInfoBase* LoadImage(const char *purl);
};

// 关键：派生 GMemoryFile，把 BinaryPtr 挂在文件实例上，
// 避免 FileOpener 单例在并发加载时覆盖缓冲
class BWGMemoryFile : public GMemoryFile {
public:
    BWGMemoryFile(const char *pFileName, BinaryPtr ptr)
        : GMemoryFile(pFileName, (const UByte*)ptr->cdata(), ptr->len()),
          swfFile_(ptr) {}
private:
    BinaryPtr swfFile_;
};

class Loader : public GFxLoader {
public:
    GPtr<GFxFileOpener> pFileOpener_;
};
```

`BWGMemoryFile` 是这里最巧妙的设计——Scaleform 的 `GFxFileOpener` 是单例，若把 `BinaryPtr` 存在单例上，并发加载第二个文件会让第一个文件的内存缓冲失效。把 `BinaryPtr` 挂在 `GMemoryFile` 实例上即可解决。

### 4.3 FlashGUIComponent / FlashTextGUIComponent

这两个类把 Flash 接入 `ashes` 的组件树（都派生自 `SimpleGUIComponent`）：

[flash_gui_component.hpp](file:///workspace/src/lib/scaleform/flash_gui_component.hpp)：

```cpp
class FlashGUIComponent : public SimpleGUIComponent
{
    Py_Header( FlashGUIComponent, SimpleGUIComponent )
public:
    FlashGUIComponent( PyMovieView* pMovieView, PyTypePlus * pType = &s_type_ );
    void update( float dTime, float relParentWidth, float relParentHeight );
    void drawSelf( bool reallyDraw, bool overlay );
    bool handleKeyEvent( const KeyEvent & event );
    bool handleMouseButtonEvent( const KeyEvent & event );
    bool handleMouseEvent( const MouseEvent & event );
    PY_RO_ATTRIBUTE_DECLARE( pMovieView_.getObject(), movie )
protected:
    PyMovieViewPtr pMovieView_;
};
```

它把绘制与输入全部委托给所持 `PyMovieView`，自身只负责在组件树中的位置/尺寸。典型用法（取自类注释）：

```python
mvDef = Scaleform.createMovie( "scaleform/flash_ui.swf" )
g = GUI.Flash( mvDef.createInstance() )
GUI.addRoot( g )
setattr( g.movie, "_root.actionScriptVariable", "hello world" )
g.movie.invoke( "_root.OpenMenuLevel2" )
```

[flash_text_gui_component.hpp](file:///workspace/src/lib/scaleform/flash_text_gui_component.hpp) 用 Scaleform 的 **DrawText API**（而非 Flash 影片）画文字：

```cpp
class FlashTextGUIComponent : public SimpleGUIComponent
{
    GPtr<GFxDrawText> gfxText_;
    GFxDrawTextManager::TextParams textParams_;  // FontSize/Underline/WordWrap/Multiline
    Vector2 corners_[4];
    GRectF stringRect_;
    // ...
};
```

它提供 `fontSize`/`underline`/`wordWrap`/`multiline` 等属性，是 `TextGUIComponent` 的 Scaleform 版替代品（字体由 Scaleform 字体库提供，而非 `romp/font`）。

### 4.4 IME

[ime.hpp](file:///workspace/src/lib/scaleform/ime.hpp)（受 `SCALEFORM_IME` 保护）：

```cpp
class IME
{
public:
    static bool init( const std::string& imeMovie );
    static bool handleIMMMessage( HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam );
    static bool handleIMEMessage( HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam );
    static void setFocussedMovie( class PyMovieView* m );
    static PyMovieView* pFocussedMovie();
    static void onDeleteMovieView( class PyMovieView* m );
    static class PyMovieView * s_pFocussed;
};
```

启用 Scaleform IME 后，Win32 IME 消息会被路由到当前焦点 movie，由 Scaleform 自己的 IME 影片渲染候选窗。

### 4.5 Python 绑定：PyMovieView / PyMovieDef / PyExternalInterface / PyFSCommandHandler

- [py_movie_view.hpp](file:///workspace/src/lib/scaleform/py_movie_view.hpp)：包装 `GFxMovieView`（影片实例视图）。提供 `invoke`（调 ActionScript 函数）、`setFSCommandCallback`、`setExternalInterfaceCallback`、`scaleMode`(`SM_NoScale`/`SM_ShowAll`/`SM_ExactFit`/`SM_NoBorder`)、`gotoFrame`/`gotoLabeledFrame`、`visible`/`backgroundAlpha`/`restart`/`setPause`/`setFocussed`/`showAsGlobalMovie`，以及 `handleKeyEvent`/`handleMouseButtonEvent`/`handleMouseEvent`。
- [py_movie_def.hpp](file:///workspace/src/lib/scaleform/py_movie_def.hpp)：包装 `GFxMovieDef`（影片定义/资源）。`createInstance()` 创建视图，`setAsFontMovie()`/`addToFontLibrary()` 把影片注册为字体来源。
- [py_external_interface.hpp](file:///workspace/src/lib/scaleform/py_external_interface.hpp)：实现 `GFxExternalInterface`，把 ActionScript `ExternalInterface.call` 回调到 Python `PyObjectPtr pCallback_`。
- [py_fs_command_handler.hpp](file:///workspace/src/lib/scaleform/py_fs_command_handler.hpp)：实现 `GFxFSCommandHandler`，把 `fscommand` 回调到 Python。

### 4.6 Log / Util / Config

- [log.hpp](file:///workspace/src/lib/scaleform/log.hpp)：`Log` 实现 Scaleform 日志接口，转发到 BigWorld 日志系统。
- [util.hpp](file:///workspace/src/lib/scaleform/util.hpp)：工具函数（颜色/矩形换算等）。
- [config.hpp](file:///workspace/src/lib/scaleform/config.hpp)：编译开关（见 4 节开头）。

### 4.7 Scaleform 与原生 GUI 对比

| 维度 | 原生 ashes GUI | Scaleform |
|------|----------------|-----------|
| 资源格式 | `.gui`（XML/DataSection）+ 纹理 `.dds/.tga` | `.gfx`/`.swf`（Flash） |
| 排版/动画 | 手写 Python + GUIShader 插值 | Flash 时间轴 + ActionScript |
| 文字 | `TextGUIComponent`（`romp/font`） | `FlashTextGUIComponent`（GFxDrawText）或 Flash 内置文字 |
| 输入 | `SimpleGUI` 派发 KeyEvent/MouseEvent | `Manager::onKeyEvent`/`onMouse` 转发到 movie |
| 绘制 | `Moo::EffectMaterial` + `GUIVertex` | `GRendererD3D9`（Scaleform 自管 D3D 状态） |
| 与组件树集成 | 原生 | 通过 `FlashGUIComponent` 伪装成 `SimpleGUIComponent` |
| 设备丢失 | `moo` 统一处理 | `Manager` 实现 `DeviceCallback` 单独处理 |
| 授权 | 内置 | 需 Scaleform 商业授权（默认 `SCALEFORM_SUPPORT 0`） |
| IME | `input/IME`（见第 6 节） | `scaleform/IME`（启用后禁用内置 IME） |

---

## 5. web_render：Mozilla/Gecko 网页渲染为纹理

[web_render/](file:///workspace/src/lib/web_render/) 把 Mozilla/Gecko（通过第三方 `LLMozlib2`）渲染的网页变成一张 D3D 纹理，供 `SimpleGUIComponent.texture` 显示。典型用途：游戏内商店、新闻、公告、登录页。

### 5.1 WebPage —— 抽象基类

[web_page.hpp](file:///workspace/src/lib/web_render/web_page.hpp)：

```cpp
class WebPage : public Moo::DeviceCallback, public Moo::BaseTexture
{
public:
    virtual void navigate( const std::wstring& url ) = 0;
    virtual void update() = 0;
    virtual void updateBrowser() = 0;
    virtual void handleMouseButtonEvent(const Vector2& pos, bool down) = 0;
    virtual void handleMouseMove(const Vector2& pos) = 0;
    virtual void handleKeyboardEvent(uint32 keyCodeIn) = 0;
    virtual void handleUnicodeInput(uint32 keyCodeIn) = 0;
    virtual bool enableProxy( bool, const std::string&, uint32 ) = 0;
    virtual bool enableCookies( bool value) = 0;
    virtual bool navigateBack() / navigateForward() / navigateReload() = 0;
    virtual bool setSize(uint32 w, uint32 h, uint32 exactW, uint32 exactH) = 0;
    virtual void focusBrowser(bool focus, bool enable) = 0;
    virtual void set404Redirect( const std::wstring& url ) = 0;
    virtual std::string evaluateJavaScript( const std::string & script ) = 0;
    virtual void updateTexture() = 0;
    // ...
    typedef enum { GRAPHICS_OPTION_LOW, GRAPHICS_OPTION_MEDIUM, GRAPHICS_OPTION_HIGH } eGraphicsOption;
    static eGraphicsOption graphicsOption_;
};
```

注意 `WebPage` 既是 `Moo::DeviceCallback`（设备丢失处理）又是 `Moo::BaseTexture`（可直接当纹理用）。

### 5.2 WebPageProvider —— Python 包装

同一文件中的 `WebPageProvider : public PyObjectPlusWithWeakReference` 是给 Python 用的包装：

```cpp
class WebPageProvider : public PyObjectPlusWithWeakReference
{
    SmartPointer<WebPage> webPage_;
public:
    PyTextureProviderPtr pTexture();   // 返回给 SimpleGUIComponent.texture
    void navigate( const std::wstring& url );
    void handleMouseButtonEvent(const Vector2& pos, bool down);
    // ...
    typedef enum { GS_EFFECT_QUALITY, GS_EFFECT_SCROLLING } eGraphicsSettingsBehaviour;
};
```

用法（取自类注释）：

```python
webPage = BigWorld.WebPageProvider(800,600, True, True, u"", "EFFECT_QUALITY")
webPage.navigate("http://www.google.com")
component.texture = webPage.texture()
```

`EFFECT_QUALITY` 表示按画质缩放网页；`EFFECT_SCROLLING` 表示在低画质下显示网页的较小局部。

### 5.3 MozillaWebPageManager —— 后台线程

[mozilla_web_page.hpp](file:///workspace/src/lib/web_render/mozilla_web_page.hpp) 是最复杂的部分。`MozillaWebPageManager` 是 `Singleton + SimpleThread`，**在后台线程跑 Mozilla**，通过命令/回调两个 FIFO 与主线程通信：

```cpp
class MozillaWebPageManager : public Singleton< MozillaWebPageManager >, public SimpleThread
{
    mutable SimpleMutex managerMutex_;
    SafeFifo<MozillaWebPageCommandPtr > commandFifo_;   // 主线程 → 后台
    SafeFifo<MozillaWebPageCommandPtr > callbackFifo_;  // 后台 → 主线程
    MapIntMozillaWebPage mapIntMozillaWebPage_;         // key → MozillaWebPage
    MapIntMozillaWebPageInterface mapIntMozillaWebPageInterface_;  // key → Interface
    // ...
    static const char* ON_NAVIGATION_BEGIN;
    static const char* ON_NAVIGATION_COMPLETE;
    static const char* ON_STATUS_TEXT_CHANGED;
};
```

内嵌类 `MozillaWebPage` 继承 `LLEmbeddedBrowserWindowObserver`，实现 `onPageChanged`/`onNavigateBegin`/`onNavigateComplete`/`onUpdateProgress`/`onStatusTextChange`/`onLocationChange`/`onClickLinkHref`/`onClickLinkNoFollow` 等浏览器观察者回调，并把像素拷贝到 `pixelsCopy_`（带 `SimpleMutex` 保护，主线程通过 `updateTextureDirect` 拷到 D3D 纹理）。

### 5.4 MozillaWebPageInterface —— 主线程门面

[mozilla_web_page_interface.hpp](file:///workspace/src/lib/web_render/mozilla_web_page_interface.hpp) 是 `WebPage` 的 Mozilla 实现，**主线程侧**：

```cpp
class MozillaWebPageInterface : public WebPage
{
    int key_;
    ComObjectWrap<DX::Texture> pTexture_;
    MD5::Digest textureChkSum_;   // 用 MD5 判断像素是否变化，避免无谓上传
    bool pageChanged_;
    PyObjectPtr pListener_;       // Python 回调对象
    float ratePerSecond_;         // 刷新频率
    bool constantRefreshDuringNavigation_;
    // ...
    class SizeInfo {  // 受保护的尺寸信息
        int webPageWidth_, webPageHeight_, textureWidth_, textureHeight_;
        bool texturePow2_, textureSquared_;
    } sizeInfo;
};
```

它把所有操作打包成 `MozillaWebPageCommand` 推入 `commandFifo_`，由后台 `MozillaWebPageManager` 执行；回调则从 `callbackFifo_` 取出，通过 `pListener_` 调 Python 方法（`ON_NAVIGATION_BEGIN` 等）。

`MD5::Digest textureChkSum_` 是个性能优化：只有像素 MD5 变化时才更新 D3D 纹理，避免静态网页每帧上传。

---

## 6. input：输入系统与 IME

[input/](file:///workspace/src/lib/input/) 是所有 UI 消费者的输入来源。

### 6.1 事件类型

[input.hpp](file:///workspace/src/lib/input/input.hpp) 定义三类事件 + 一个联合：

```cpp
class KeyEvent {            // 键/鼠标按钮
    KeyCode::Key key_; int32 state_; uint32 modifiers_;
    wchar_t character_[CHAR_MAX_SIZE];  // UTF-16 字符
    float cursorX_, cursorY_;
    static KeyEvent make( KeyCode::Key key, int32 state, uint32 mods, const Vector2& cursorPos );
    bool isKeyDown() const { return state_ >= 0; }
    int32 repeatCount() const { return state_; }   // 0=首次按下, -1=释放, >0=自动重复
    bool isMouseButton() const;
    const wchar_t* utf16Char() const;
    bool utf8Char( std::string& output ) const;
    bool isShiftDown() / isCtrlDown() / isAltDown();
    Vector2 cursorPosition() const;
};

class MouseEvent {          // 鼠标移动
    long dx_, dy_, dz_;     // dz = 滚轮
    float cursorX_, cursorY_;
};

class AxisEvent {           // 手柄摇杆
    enum Axis { AXIS_LX, AXIS_LY, AXIS_RX, AXIS_RY, NUM_AXES };
    Axis axis_; float value_; float dTime_;
};

class InputEvent {          // 统一包装
    enum Type { KEY, MOUSE, AXIS, UNDEFINED } type_;
    union { KeyEvent key_; MouseEvent mouse_; AxisEvent axis_; };
    uint32 seqId_;          // 用于 consumeInput
};
```

`KeyStates` 维护每键的按下/自动重复状态；`state_` 字段巧妙地用 `-1`=释放、`0`=首次按下、`>0`=重复次数同时表达「是否按下」与「重复计数」。

### 6.2 InputHandler —— 抽象基类

```cpp
class InputHandler
{
public:
    virtual bool handleKeyEvent( const KeyEvent & event );
    virtual bool handleInputLangChangeEvent();
    virtual bool handleIMEEvent( const IMEEvent & event );
    virtual bool handleMouseEvent( const MouseEvent & event );
    virtual bool handleAxisEvent( const AxisEvent & event );
};
```

`SimpleGUI`、`Scaleform::Manager`（通过 `onKeyEvent`/`onMouse`）、`Tool`（gizmo 模块）等都实现/持有 `InputHandler`。

### 6.3 InputDevices（Singleton）+ Joystick

```cpp
class InputDevices : public Singleton< InputDevices >
{
public:
    bool init( void * hInst, void * hWnd );
    static bool processEvents( InputHandler & handler );
    static bool handleWindowsMessage( HWND, UINT, WPARAM&, LPARAM&, LRESULT& );
    static uint32 modifiers();
    static bool isKeyDown( KeyCode::Key key );
    static KeyStates & keyStates();
    static Joystick & joystick();
    static void setFocus( bool state, InputHandler * handler );
    static void consumeInput();
    static void pushIMECharEvent( wchar_t* chs );
private:
    EventQueue eventQueue_;        // Win32 消息累积成事件入队
    Joystick joystick_;
    KeyStates keyStates_, keyStatesInternal_;
    // ...
};
```

`Joystick`（同文件）封装 `IDirectInputDevice8`，把摇杆轴映射为 `AxisEvent`，把方向键映射为 `KeyEvent`。`processEvents` 把队列里的事件依次喂给 `InputHandler`，返回值决定是否「已处理」。

文件清单：
- [key_code.hpp](file:///workspace/src/lib/input/key_code.hpp) / `key_code.cpp`：`KeyCode::Key` 枚举（`KEY_NONE`...`NUM_KEYS`）。
- `vk_map.hpp/cpp`：Win32 虚拟键码 ↔ `KeyCode::Key` 映射。
- `input_cursor.hpp/cpp`：光标位置维护。
- `xbinput.cpp`：Xbox 输入（旧）。
- `py_input.hpp/cpp`：Python 绑定。

### 6.4 IME —— 输入法

[input/ime.hpp](file:///workspace/src/lib/input/ime.hpp) 是 BigWorld **内置** IME（与 Scaleform IME 二选一）：

```cpp
class IME : public InitSingleton<IME>
{
public:
    enum Language { LANGUAGE_NON_IME, LANGUAGE_CHS, LANGUAGE_CHT, LANGUAGE_JAPANESE, LANGUAGE_KOREAN };
    enum State    { STATE_OFF, STATE_ON, STATE_ENGLISH };

    bool initIME( HWND hWnd );
    void update();
    void processEvents( InputHandler & handler );   // 产生 IMEEvent 喂给 InputHandler
    bool handleWindowsMessage( HWND, UINT, WPARAM&, LPARAM&, LRESULT& );
    void onInputLangChange();
    void enabled( bool enable, bool finalise=true );

    // 当前 IME 状态（供 UI 绘制候选窗/组合串）
    std::wstring composition();
    const AttrArray& compositionAttr();      // 组合串各字符属性（已转换/未转换/选中）
    int compositionCursorPosition();
    const std::wstring& reading();           // 阅读串（日文）
    bool readingVisible(), readingVertical();
    const WStringArray& candidates();        // 候选词列表
    bool candidatesVisible(), candidatesVertical();
    int selectedCandidate();
};
```

[ime_event.hpp](file:///workspace/src/lib/input/ime_event.hpp) 的 `IMEEvent` 用一组 bool 标记「什么变了」：

```cpp
class IMEEvent {
    bool stateChanged_, candidatesVisibilityChanged_, candidatesChanged_,
         selectedCandidateChanged_, compositionChanged_,
         compositionCursorPositionChanged_, readingVisibilityChanged_, readingChanged_;
    bool dirty() const;   // 任一为真
};
```

UI 侧（如 `TextGUIComponent` 的编辑模式，或 Scaleform 的 IME movie）根据这些标记决定重绘哪部分。`ImeUi.cpp/h` 是来自 DXUT 示例的 IME UI 参考实现。

### 6.5 input 与 appmgr / controls / ashes 的协作

```
Win32 消息循环
   │
   ▼
InputDevices::handleWindowsMessage  ──→  IME::handleWindowsMessage (IME 消息拦截)
   │                                       │
   ▼                                       ▼
InputDevices::processEvents(handler)    IME::processEvents(handler)
   │                                       │
   ├─→ handler.handleKeyEvent/MouseEvent/AxisEvent
   │
   ▼
InputHandler 实现者：
   - SimpleGUI (ashes)        → 派发给焦点 GUI 组件
   - Scaleform::Manager       → onKeyEvent/onMouse → movie
   - Tool (gizmo)             → 工具快捷键
   - appmgr (dev_menu 等)     → 开发菜单
```

`controls/`（MFC 控件）不直接消费 `InputHandler`——它走 MFC 的消息映射，但 `GUI::BigworldInputDevice` 会读 `InputDevices::keyStates` 来做快捷键命中。

---

## 7. GUI 完整渲染流程

把以上各节串起来，一帧 GUI 的完整流程是：

```
1. Win32 消息泵
   InputDevices::handleWindowsMessage()
     ├─ IME::handleWindowsMessage()  (若启用内置 IME，拦截 IME 消息)
     └─ 把键/鼠标消息转成 InputEvent 入 eventQueue_

2. 客户端主循环 update 阶段
   InputDevices::processEvents( simpleGUI )     // SimpleGUI 是 InputHandler
     └─ simpleGUI.handleKeyEvent/MouseEvent/AxisEvent
          └─ processClickKey/processMouseMove/processDragMove
               └─ closestHitTest( focusList, cursorPos )  按 drawOrder 挑中
                    └─ component.handleMouse*Event()      → 转发到 Python 脚本
   IME::processEvents( simpleGUI )  (若有 IME 事件)
     └─ simpleGUI.handleIMEEvent() → 焦点文本组件

   Scaleform::Manager::onKeyEvent/onMouse  (若启用 Scaleform)
     └─ 转发到 focussed PyMovieView

3. 客户端主循环 GUI update 阶段
   SimpleGUI::update(dTime)
     └─ 对每个 root component:
          update(dTime, relParentW, relParentH)
            ├─ layout()  按 position/size mode + anchor 算 x,y,w,h
            ├─ buildMesh()/tile()  (脏时重建顶点)
            ├─ applyShaders(dTime)
            │    └─ for each GUIShader: shader.processComponent(*this, dTime)
            │         ├─ ClipGUIShader 改 vertices 的位置/UV
            │         ├─ AlphaGUIShader 改 vertices 的 diffuse alpha
            │         ├─ ColourGUIShader 改 runTimeColour_
            │         └─ MatrixGUIShader 改 runTimeTransform_
            └─ updateChildren(...)

   Scaleform::Manager::tick(elapsed, frameCatchUp=2)
     └─ 推进所有 movie 的 ActionScript 时间轴

   MozillaWebPageManager::tick(dTime)  (后台线程已渲染网页像素)
     └─ 主线程把 pixelsCopy 拷到 D3D 纹理 (updateTextureDirect)

4. 客户端主循环 draw 阶段
   SimpleGUI::draw()
     └─ pushClipRegion( root )  → 设 viewport/视矩阵
          root.draw(reallyDraw, overlay)
            ├─ drawSelf():
            │    ├─ material_->hTechnique = s_techniqueTable[materialFX_]
            │    ├─ setConstants(runTimeColour_, pixelSnap_)
            │    ├─ setVertexShaderConstant( runTimeTransform_ )
            │    └─ Moo::EffectMaterial::commit() → DrawPrimitive
            └─ drawChildren()  (按 childOrder_ 排序，递归)
          popClipRegion()

   Scaleform::Manager::draw()
     └─ GRendererD3D9 把每个 movie 的显示列表画到后台缓冲
        (FlashGUIComponent.drawSelf 会调 pMovieView_->Display)

5. Moo present → 屏幕
```

### 关键调用流：一次鼠标点击穿透到 Python

```
Win32 WM_LBUTTONDOWN
  → InputDevices::handleWindowsMessage
     → pushMouseButtonEvent( KEY_LEFTMOUSE, true, mods, cursorPos )
  → InputDevices::processEvents( simpleGUI )
     → simpleGUI.handleKeyEvent( KeyEvent )
        → simpleGUI.processClickKey( event )
           → closestHitTest( mouseButtonFocusList_, cursorPos )
              → 命中 component X (hitTest 返回 true 且在 clipRegion 内)
                 → X.handleMouseButtonEvent( event )
                    → invokeKeyEventHandler( pScriptObject_, "handleMouseClickEvent", ... )
                       → PyObject_CallMethod( script, "handleMouseClickEvent", ... )
```

---

## 8. 总结与设计要点

1. **分层清晰**：`input`（事件源）→ `ashes`（2D 渲染）→ `SimpleGUIComponent`（组件树）→ `GUIShader`（每帧变形/着色）→ `moo`（D3D 提交）。`scaleform`/`web_render` 通过「提供纹理 / 派生 SimpleGUIComponent」平行接入。
2. **脚本化优先**：`SimpleGUIComponent` 既是 C++ 类也是 Python 类（`PyObjectPlusWithWeakReference`），几乎所有属性、事件都暴露给 Python，游戏逻辑用 `.gui` XML + Python 脚本组合 UI。
3. **工厂模式贯穿**：`COMPONENT_FACTORY_DECLARE`/`SHADER_FACTORY_DECLARE` 让 `.gui` 资源按类名反序列化；`DECLARE_THUMBNAIL_PROVIDER`（UAL 侧）同理。
4. **设备丢失处理**：`Moo::DeviceCallback` 被 `Scaleform::Manager`、`WebPage` 等实现，确保 alt-tab 后资源正确重建。
5. **编辑器与运行时分离**：`guimanager`/`guitabs`/`controls` 是 MFC 编辑器侧，与运行时 `ashes` 共享 `input`/`moo`/`resmgr` 但不互相依赖。
6. **IME 双轨**：内置 `input/IME` 与 `scaleform/IME` 二选一（由 `SCALEFORM_IME` 开关决定），后者启用会禁用前者。
7. **Web 渲染的后台线程化**：`MozillaWebPageManager` 用命令/回调 FIFO + MD5 校验，把 Mozilla 的重活放到后台线程，主线程只做纹理上传，避免阻塞帧。
8. **任务描述勘误**：`GUIComponent`/`GUITickler`/`py_gui_component` 等名称在代码中不存在；`*GUIComponent`/`*GUIShader` 类族实际位于 `ashes/` 而非 `controls/`；`GUIActionMaker`/`GUITextorMaker`/`GUIUpdaterMaker` 实际是 `guimanager/` 中的 `ActionMaker`/`TextorMaker`/`UpdaterMaker` 模板（`GUI` 是 namespace）。
