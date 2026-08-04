# 18. 工具与编辑器子系统：UAL / Gizmo / Duplo / Ashes / OpenAutomate

> 本文承接 [17-GUI与UI栈-gui-scaleform-web.md](file:///workspace/study-docs/17-GUI与UI栈-gui-scaleform-web.md)，专门分析 BigWorld Engine 2.0.1 中「支撑 World Editor / Model Editor / Particle Editor 等工具」的那一层公共库。这部分代码不是运行时客户端的核心，却是「美术与策划每天打开就用的编辑器」的基石。它由五个相互正交但彼此咬合的模块组成：
>
> - **UAL (Universal Asset Locator)**：跨文件系统 / XML 数据源的资产浏览器，带缩略图后台生成、历史记录、收藏夹、拖放目标管理；
> - **Gizmo**：3D 视口里的位置/旋转/缩放/半径/角度操纵器，以及配套的属性系统（`GeneralEditor`/`GeneralProperty`）、工具栈（`ToolManager`）、撤销重做（`UndoRedo`）、吸附（`SnapProvider`）；
> - **Duplo**：编辑器与运行时共用的「可附加对象 + 马达 + 模型渲染到纹理 + 阴影投射」层，是 `pymodel` 在工具侧的承载体；
> - **Ashes**：游戏内 2D UI 渲染库（详见文档 17），工具侧通过 `PyModelRenderer` 把模型渲染成纹理喂给 `SimpleGUIComponent`，故在此仅做衔接说明；
> - **OpenAutomate**：把客户端接入 Futuremark OpenAutomate 自动化测试框架的 Python 包装层。

## 0. 目录结构与职责定位

| 目录 | 模块性质 | 关键基类/Singleton | 服务对象 |
|------|----------|--------------------|----------|
| [ual/](file:///workspace/src/lib/ual/) | 资产浏览器（MFC + GUITABS 面板） | `UalManager`、`UalDialog`、`ListProvider`、`VFolderProvider`、`ThumbnailManager` | WorldEditor / ModelEditor / ParticleEditor |
| [gizmo/](file:///workspace/src/lib/gizmo/) | 3D 操纵器 + 属性编辑框架 | `GizmoManager`、`Gizmo`、`GeneralEditor`、`GeneralProperty`、`ToolManager`、`UndoRedo` | 所有 3D 视口编辑器 |
| [duplo/](file:///workspace/src/lib/duplo/) | 附加对象 / 马达 / 模型渲染 / 阴影 | `PyModel`、`PyAttachment`、`Motor`、`PyModelRenderer`、`ShadowCaster` | 运行时客户端 + 工具 |
| [ashes/](file:///workspace/src/lib/ashes/) | 2D UI 渲染库（详见文档 17） | `SimpleGUI`、`SimpleGUIComponent` | 运行时 + 工具（缩略图/图标） |
| [open_automate/](file:///workspace/src/lib/open_automate/) | OpenAutomate 自动化测试包装 | `BWOpenAutomate`、`OACommandWrapper`、`OANamedOptionWrapper` | 客户端（基准/兼容性测试） |

### 模块依赖图

```
        ┌─────────────────────────────────────────────────────────────┐
        │                       World Editor App                       │
        │            (MFC 主框架 + GUITABS 可停靠面板体系)              │
        └───────┬────────────────────────┬───────────────────┬────────┘
                │                        │                   │
                ▼                        ▼                   ▼
        ┌───────────────┐      ┌─────────────────┐   ┌──────────────┐
        │   ual/        │      │    gizmo/       │   │ open_automate│
        │ UalManager(S) │◀─────│ GizmoManager(S) │   │ BWOpenAutomate│
        │ UalDialog     │ 拖放  │ GeneralEditor   │   │ OA*Wrapper   │
        │ ListProvider  │ 资产  │ ToolManager(S)  │   └──────┬───────┘
        │ ThumbnailMgr  │  ↓    │ UndoRedo        │          │ -openautomate
        └──────┬────────┘      └────────┬────────┘          ▼
               │                        │             ┌──────────────┐
               │ 缩略图渲染               │ 模型可视化    │ 外部         │
               ▼                        ▼             │ OpenAutomate │
        ┌──────────────────────────────────────────┐  │  测试主机    │
        │                duplo/                    │  └──────────────┘
        │ PyModel  PyAttachment  ChunkAttachment   │
        │ Motor(Homer/Orbitor/Oscillator...)       │
        │ PyModelRenderer ──texture──▶ ashes/      │
        │ ShadowCaster  *DrawOverride              │
        └─────────────────────┬────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   moo/ + chunk/  │  (D3D9 设备、场景分块)
                    └──────────────────┘
```

关键耦合点：

1. **UAL → Duplo/Ashes**：缩略图由 `ThumbnailManager` 调用各类 `ThumbnailProvider`（图片/模型/XML）生成，模型缩略图走 `duplo` 的 `PyModelRenderer` 把模型渲染到一张纹理，再保存为磁盘文件。
2. **Gizmo → UAL**：编辑器从 UAL 拖资产到 3D 视口，触发 `Gizmo` 创建/选中对象，并通过 `GeneralProperty` 把对象的可编辑属性暴露给属性面板。
3. **Duplo → Ashes**：`PyModelRenderer` 实现了 `TextureRenderer` 接口，把 `PyModel` 画到一张 `PyTextureProvider`，最终挂到 `SimpleGUIComponent.texture` 上显示。
4. **OpenAutomate → 客户端**：客户端以 `-openautomate` 参数启动后，`BWOpenAutomate` 把命令/选项包装成 Python 对象，由测试脚本驱动渲染基准与兼容性用例。

---

## 1. UAL：通用资产浏览器

UAL 是 BigWorld 工具链里复用度最高的组件之一。它本身是一个 MFC `CDialog`，同时实现 `GUITABS::Content` 接口，因此可以作为一个可停靠/可撕离的面板嵌入任何基于 GUITABS 的编辑器主框架。多个 UAL 实例可以共存（克隆），由 `UalManager` 统一调度。

### 1.1 UalManager —— 单例调度器

[ual_manager.hpp](file:///workspace/src/lib/ual/ual_manager.hpp) 中的 `UalManager` 是整个 UAL 子系统的入口，它同时是 `Singleton<UalManager>` 和 `GUI::ActionMaker<UalManager>`（即把 Refresh / Layout 等动作注册到 `GUI::Manager` 菜单/工具条）：

```cpp
class UalManager :
    public Singleton<UalManager>,
    public GUI::ActionMaker<UalManager>
{
public:
    UalManager();
    ~UalManager();

    // 应用程序添加默认搜索路径
    void addPath( const std::wstring& path );
    // 设置配置文件路径（必须在创建任何 ual dialog 之前调用）
    void setConfigFile( std::wstring config );
    const std::wstring getConfigFile();

    // History / Favourites 访问器
    UalHistory& history() { return history_; }
    UalFavourites& favourites() { return favourites_; }

    // ThumbnailManager / DropManager 访问器
    ThumbnailManager& thumbnailManager() { return *thumbnailManager_; }
    UalDropManager& dropManager() { return dropManager_; }

    // 一组回调注入点：单击/双击/右键菜单/拖放起止/焦点/错误
    void setItemClickCallback( UalItemCallback* callback );
    void setItemDblClickCallback( UalItemCallback* callback );
    void setPopupMenuCallbacks( UalStartPopupMenuCallback* c1, UalEndPopupMenuCallback* c2 );
    void setStartDragCallback( UalItemCallback* callback );
    void setUpdateDragCallback( UalItemCallback* callback );
    void setEndDragCallback( UalItemCallback* callback );
    void setFocusCallback( UalFocusCallback* callback );
    void setErrorCallback( UalErrorCallback* callback );

    UalDialog* getActiveDialog();
    void updateItem( const std::wstring& longText );     // 强制刷新某个条目
    void refreshAllDialogs();
    void showItem( const std::wstring& vfolder, const std::wstring& longText );
    void fini();
private:
    std::vector<std::wstring> paths_;
    std::vector<UalDialog*> dialogs_;
    std::wstring configFile_;
    UalHistory history_;
    UalFavourites favourites_;
    ThumbnailManagerPtr thumbnailManager_;
    UalDropManager dropManager_;
    UINT_PTR timerID_;
    /* 一组 SmartPointer<UalXxxCallback> ... */
};
```

设计要点：

- **回调全部用 functor 注入**：UAL 不直接知道宿主编辑器的类型，所有「用户双击了一个 model」「用户把 texture 拖到了视口」之类的事件，都通过 `UalItemCallback` 等 functor 回调上抛。宿主用 `UalFunctor1<YourClass, UalItemInfo*>` 把成员函数包成回调塞进来。
- **History / Favourites 是全局共享的**：它们挂在 `UalManager` 上而不是单个 `UalDialog` 上，所以「在 UAL_A 里访问过的资产，在 UAL_B 的历史里也能看到」。
- **`thumbnailManager_` 是引用计数对象**：所有 `UalDialog` 共享同一个 `ThumbnailManager`，避免重复生成缩略图。
- **定时器 `onTimer`**：用一个 Win32 定时器周期性驱动 `ThumbnailManager::tick()`，把后台线程生成的缩略图回写到主线程的列表控件。

### 1.2 AssetInfo / UalItemInfo —— 资产元数据与事件载体

UAL 里流动的最小信息单元是 `AssetInfo`，定义在 [asset_info.hpp](file:///workspace/src/lib/ual/asset_info.hpp)：

```cpp
class AssetInfo : public ReferenceCount
{
public:
    AssetInfo() {};
    AssetInfo( const std::wstring& type, const std::wstring& text,
               const std::wstring& longText,
               const std::wstring& thumbnail = L"",
               const std::wstring& description = L"" );
    AssetInfo( DataSectionPtr sec );   // 也可从 XML DataSection 直接构造

    bool empty() const;
    bool equalTo( const AssetInfo& other ) const;
    const std::wstring& type() const;        // 例如 L"file", L"xml"
    const std::wstring& text() const;        // 显示名
    const std::wstring& longText() const;    // 全路径 / 唯一标识
    const std::wstring& thumbnail() const;   // 缩略图路径
    const std::wstring& description() const;
private:
    std::wstring type_, text_, longText_, thumbnail_, description_;
};
typedef SmartPointer<AssetInfo> AssetInfoPtr;
```

`AssetInfo` 只是数据容器；真正在回调里传递的是 [ual_callback.hpp](file:///workspace/src/lib/ual/ual_callback.hpp) 中的 `UalItemInfo`，它在 `AssetInfo` 之上附加了「来源对话框 / 鼠标坐标 / 是否文件夹 / 链式 next 指针」：

```cpp
class UalItemInfo
{
public:
    UalItemInfo( UalDialog* dialog, AssetInfo assetInfo, int x, int y,
                 bool isFolder = false, void* folderData = 0 );
    const UalDialog* dialog() const;
    const AssetInfo& assetInfo() const;
    int x() const; int y() const;
    bool isFolder() const;
    UalItemInfo* getNext() const;   // 支持多选拖放：链表串起多个条目
private:
    UalDialog* dialog_;
    AssetInfo  assetInfo_;
    int x_, y_;
    bool isFolder_;
    void* folderExtraData_;
    UalItemInfo* next_;             // 多选时由 UalDialog 串成链表
};
```

回调类型全部基于 `cstdmf/bw_functor.hpp` 的 `BWBaseFunctorN`：

```cpp
typedef BWBaseFunctor1< UalItemInfo* >              UalItemCallback;          // 单击/双击/拖放
typedef BWBaseFunctor2< UalItemInfo*, UalPopupMenuItems& > UalStartPopupMenuCallback;
typedef BWBaseFunctor2< UalItemInfo*, int >         UalEndPopupMenuCallback;
typedef BWBaseFunctor1< const std::string& >        UalErrorCallback;
typedef BWBaseFunctor2< UalDialog*, bool >          UalFocusCallback;
#define UalFunctor1  BWFunctor1   // 带 instance 的成员函数绑定别名
```

### 1.3 UalDialog —— MFC 对话框 + GUITABS 面板

[ual_dialog.hpp](file:///workspace/src/lib/ual/ual_dialog.hpp) 中的 `UalDialog` 是 UAL 的视觉主体，多重继承自 `CDialog`、`GUITABS::Content` 以及三个事件处理器接口：

```cpp
class UalDialog :
    public CDialog,
    public GUITABS::Content,
    public FolderTreeEventHandler,
    public SmartListCtrlEventHandler,
    public FiltersCtrlEventHandler
{
public:
    // ---- GUITABS::Content 接口 ----
    static const std::wstring contentID;
    std::wstring getContentID() { return contentID; }
    std::wstring getDisplayString() { return dlgLongCaption_; }
    HICON getIcon() { return hicon_; }
    CWnd* getCWnd() { return this; }
    bool isClonable() { return true; }        // 允许 GUITABS 克隆出多个实例
    GUITABS::ContentPtr clone();
    OnCloseAction onClose( bool isLastContent )
    { return isLastContent ? CONTENT_HIDE : CONTENT_DESTROY; }

    bool loadConfig( const std::wstring fname = L"" );
    void saveConfig();
    void setListStyle( SmartListCtrl::ViewStyle style );
    void setLayout( bool vertical, bool resetLastSize = false );
    void showItem( const std::wstring& vfolder, const std::wstring& longText );

    // ---- 三个子控件的事件回调 ----
    void folderTreeSelect( VFolderItemData* data );
    void folderTreeStartDrag( VFolderItemData* data );
    void listDoubleClick( int index );
    void listStartDrag( int index );
    void listItemRightClick( int index );
    void filterClicked( const wchar_t* name, bool pushed, void* data );
    /* ... */
private:
    CToolBarCtrl    toolbar_;
    FolderTree      folderTree_;     // 左侧虚拟文件夹树
    SmartListCtrl   smartList_;      // 右侧资产列表（虚拟列表）
    SearchField     search_;         // 顶部搜索框（来自 controls/）
    CStatic         statusBar_;
    FiltersCtrl     filtersCtrl_;    // 过滤器条
    NiceSplitterWnd* splitterBar_;

    // 每个 UalDialog 各自持有一组 provider
    ListFileProviderPtr    fileListProvider_;
    ListXmlProviderPtr     xmlListProvider_;
    ListXmlProviderPtr     historyListProvider_;
    ListXmlProviderPtr     favouritesListProvider_;
    VFolderXmlProviderPtr  historyFolderProvider_;
    VFolderXmlProviderPtr  favouritesFolderProvider_;
};
```

`UalDialog` 的布局是经典的「三栏 + 工具条」：

```
┌─────────────────────────────────────────────────────┐
│ [搜索框] [刷新] [布局切换] [大图标/小图标/列表]      │  toolbar_
├──────────────┬──────────────────────────────────────┤
│ 虚拟文件夹树  │            资产列表 (SmartListCtrl)   │
│ (FolderTree) │                                        │
│              │                                        │
│  + models    │   [缩略图]  [缩略图]  [缩略图]         │
│  + textures  │                                        │
│  + history   │                                        │
│  + favourites│                                        │
├──────────────┴──────────────────────────────────────┤
│ [过滤器: *.model | *.tga | *.xml | ...]              │  filtersCtrl_
├─────────────────────────────────────────────────────┤
│ 状态栏: "123 items"                                  │  statusBar_
└─────────────────────────────────────────────────────┘
```

配置加载由一组 `loadXxx` 私有方法完成：`loadMain` 读主框架、`loadToolbar` 读工具条按钮、`loadFilters` 读过滤器、`loadVFolders` 读虚拟文件夹定义。虚拟文件夹可以「基于文件系统」也可以「基于 XML」，由不同的 `VFolderProvider` 实现。

`UalDialogFactory` 是给 GUITABS 用的工厂：

```cpp
class UalDialogFactory : public GUITABS::ContentFactory
{
public:
    GUITABS::ContentPtr create()
    { return createUal( UalManager::instance().getConfigFile() ); }
    UalDialog* createUal( const std::wstring& configFile )
    {
        UalDialog* newUal = new UalDialog( configFile );
        newUal->Create( UalDialog::IDD );
        return newUal;
    }
};
```

### 1.4 Provider 体系：ListProvider 与 VFolderProvider

UAL 的「数据来源」被抽象成两层 provider，分别对应「右侧资产列表」和「左侧文件夹树」。

#### ListProvider（资产列表数据源）

基类定义在 [smart_list_ctrl.hpp](file:///workspace/src/lib/ual/smart_list_ctrl.hpp)：

```cpp
class ListProvider : public ReferenceCount
{
public:
    virtual void setFilterHolder( FilterHolder* filterHolder );
    virtual void refresh() = 0;            // 重新扫描数据源
    virtual bool finished() = 0;           // 后台扫描是否完成
    virtual int getNumItems() = 0;
    virtual const AssetInfo getAssetInfo( int index ) = 0;
    virtual void getThumbnail( ThumbnailManager& manager,
                                int index, CImage& img, int w, int h,
                                ThumbnailUpdater* updater ) = 0;
    virtual void filterItems() = 0;        // 按当前过滤器筛选
protected:
    FilterHolder* filterHolder_;
};
typedef SmartPointer<ListProvider> ListProviderPtr;
```

`ListProvider` 的类层级：

```
ListProvider (ReferenceCount)
├── ListFileProvider   ← 扫描文件系统（带后台线程）
├── ListXmlProvider    ← 从 XML 列表读取
└── ListMultiProvider  ← 聚合多个 provider
```

`ListFileProvider`（[list_file_provider.hpp](file:///workspace/src/lib/ual/list_file_provider.hpp)）是最常用、也最复杂的一个，它把「扫描目录」放在后台 `SimpleThread` 里做，主线程只负责消费结果：

```cpp
class ListFileProvider : public ListProvider
{
public:
    enum ListFileProvFlags
    {
        LISTFILEPROV_DEFAULT      = 0,
        LISTFILEPROV_DONTRECURSE  = 1,   // 不递归子目录
        LISTFILEPROV_DONTFILTERDDS = 2    // 不自动过滤掉有同名 tga/jpg/png 的 dds
    };
    virtual void init( const std::wstring& type,
                       const std::wstring& paths,
                       const std::wstring& extensions,
                       const std::wstring& includeFolders,
                       const std::wstring& excludeFolders,
                       int flags = LISTFILEPROV_DEFAULT );
    virtual void refresh();
    virtual bool finished();
    virtual int getNumItems();
    virtual const AssetInfo getAssetInfo( int index );
private:
    static const int VECTOR_BLOCK_SIZE = 1000;   // 分块扩容，避免频繁 realloc
    class ListItem : public SafeReferenceCount   // 单个文件条目
    {
    public:
        std::wstring fileName;   // 全路径
        std::wstring dissolved;  // 解析后的名字
        std::wstring title;      // 仅文件名
    };
    std::vector<ListItemPtr> items_;
    std::vector<ListItemPtr> searchResults_;      // 搜索后的子集
    // 后台线程
    SimpleThread* thread_;
    SimpleMutex mutex_;
    SimpleMutex threadMutex_;
    bool threadWorking_;
    std::vector<ListItemPtr> threadItems_;       // 线程产出，主线程消费
    std::vector<ListItemPtr> tempItems_;
    clock_t flushClock_;
    int threadFlushMsec_;
    int threadYieldMsec_;                         // 每 msec 让线程 sleep 50ms
    int threadPriority_;                          // >0 高于正常，<0 低于正常
};
```

关键设计：扫描结果以 `VECTOR_BLOCK_SIZE = 1000` 为粒度分块追加，并通过 `flushThreadBuf()` 周期性地把 `threadItems_` 合并到 `items_`，这样大目录（几万个文件）下 UI 不会卡死，且 `SmartListCtrl` 能在扫描中途就显示已发现的条目。`setThreadYieldMsec` / `setThreadPriority` 让宿主能调低扫描线程的优先级，避免抢占主线程。

`ListXmlProvider`（[list_xml_provider.hpp](file:///workspace/src/lib/ual/list_xml_provider.hpp)）则把一组 `AssetInfo` 存在 XML `DataSection` 里，用于历史记录、收藏夹这类「条目数有限、需要持久化」的场景。

#### VFolderProvider（虚拟文件夹树数据源）

左侧 `FolderTree` 的每个节点由一个 `VFolderProvider` 提供子节点。基类 `VFolderProvider` 与 `VFolderItemData` 定义在 [folder_tree.hpp](file:///workspace/src/lib/ual/folder_tree.hpp)（参见该文件）。`VFolderXmlProvider`（[vfolder_xml_provider.hpp](file:///workspace/src/lib/ual/vfolder_xml_provider.hpp)）是 XML 驱动的实现：

```cpp
class VFolderXmlProvider : public VFolderProvider
{
public:
    virtual void init( const std::wstring& path );
    virtual bool startEnumChildren( const VFolderItemDataPtr parent );
    virtual VFolderItemDataPtr getNextChild( ThumbnailManager& thumbnailManager, CImage& img );
    virtual void getThumbnail( ThumbnailManager& thumbnailManager,
                                VFolderItemDataPtr data, CImage& img );
    virtual const std::wstring getDescriptiveText( VFolderItemDataPtr data,
                                                    int numItems, bool finished );
    virtual bool getListProviderInfo( VFolderItemDataPtr data,
                                       std::wstring& retInitIdString,
                                       ListProviderPtr& retListProvider,
                                       bool& retItemClicked );
private:
    std::wstring path_;
    bool sort_;
    std::vector<ItemPtr> items_;
    std::vector<ItemPtr>::iterator itemsItr_;
};
```

`getListProviderInfo` 是 VFolder 与 List 之间的桥梁：当用户在左侧树点中一个虚拟文件夹时，`UalDialog` 调用此方法拿到对应的 `ListProvider`，再把它喂给右侧 `SmartListCtrl`。文件系统驱动的对应实现是 `VFolderFileProvider`，聚合多个的是 `VFolderMultiProvider`。

### 1.5 ThumbnailManager —— 后台线程缩略图生成

缩略图是 UAL 性能上最敏感的部分。一个美术目录里可能躺着上万个 `.model` / `.tga` / `.png`，全部同步生成缩略图会冻结 UI。`ThumbnailManager`（[thumbnail_manager.hpp](file:///workspace/src/lib/ual/thumbnail_manager.hpp)）采用「后台线程 prepare + 主线程 render + 磁盘缓存」的三段式：

```cpp
class ThumbnailManager : public Aligned, public ReferenceCount
{
public:
    static void registerProvider( ThumbnailProviderPtr provider );

    void create( const std::wstring& file, CImage& img, int w, int h,
                 ThumbnailUpdater* updater, bool loadDirectly = false );
    void tick();    // 由 UalManager 的定时器驱动

    std::wstring postfix() const;   // 缩略图文件名后缀，默认 ".thumb.bmp"
    std::wstring folder() const;    // 缓存目录
    int size() const;               // 缩略图边长（像素）
    COLORREF backColour() const;    // 背景色
private:
    static std::vector<ThumbnailProviderPtr> * s_providers_;

    SimpleThread* thread_;
    SimpleMutex mutex_;
    ThreadDataPtr renderData_;          // 传给主线程渲染的数据
    Moo::RenderTarget renderRT_;        // 主线程渲染用的 RT
    bool renderRequested_;
    int renderSize_;
    std::list<ThreadDataPtr> pending_;   // 待处理队列
    SimpleMutex pendingMutex_;
    std::list<ThreadResultPtr> results_; // 线程产出
    SimpleMutex resultsMutex_;
    std::list<ThreadResultPtr> ready_;   // 已就绪可被 UI 取走
    std::set<std::wstring> errorFiles_;  // 失败名单，避免重试
    bool stopThreadRequested_;
    Moo::LightContainerPtr pNewLights_;  // 渲染模型缩略图用的默认灯光
};
```

`ThumbnailProvider` 是策略接口，每种资产类型一个实现：

```cpp
class ThumbnailProvider : public ReferenceCount
{
public:
    virtual bool isValid( const ThumbnailManager& manager, const std::wstring& file ) = 0;
    // 在后台线程调用：加载/准备资产（不能碰 D3D 设备）
    virtual bool prepare( const ThumbnailManager& manager, const std::wstring& file ) = 0;
    // 在主线程调用：把准备好的资产画到传入的 RenderTarget
    virtual bool render( const ThumbnailManager& manager,
                         const std::wstring& file, Moo::RenderTarget* rt ) = 0;
    virtual bool needsCreate( const ThumbnailManager& manager, const std::wstring& file,
                              std::wstring& thumb, int& size );
};
```

UAL 自带的 provider 包括：

- `ImageThumbProv`（[image_thumb_prov.hpp](file:///workspace/src/lib/ual/image_thumb_prov.hpp)）：tga/jpg/png/bmp/dds 等图片，直接缩放；
- `ModelThumbProv`（[model_thumb_prov.hpp](file:///workspace/src/lib/ual/model_thumb_prov.hpp)）：`.model`，借助 `duplo` 的 `PyModelRenderer` 把模型渲染到 RT；
- `IconThumbProv`（[icon_thumb_prov.hpp](file:///workspace/src/lib/ual/icon_thumb_prov.hpp)）：从文件关联图标取缩略图；
- `XmlThumbProv`（[xml_thumb_prov.hpp](file:///workspace/src/lib/ual/xml_thumb_prov.hpp)）：XML 条目自定义缩略图。

注册用宏完成，本质是在静态初始化期往 `s_providers_` 推一个实例：

```cpp
#define DECLARE_THUMBNAIL_PROVIDER()  static ThumbProvFactory s_factory_;
#define IMPLEMENT_THUMBNAIL_PROVIDER( CLASS ) \
    ThumbProvFactory CLASS::s_factory_( new CLASS() );
```

典型时序（生成一个 model 缩略图）：

```
SmartListCtrl::OnGetDispInfo
   └─ ThumbnailManager::create(file, img, w, h, updater)
        ├─ needsCreate?  读磁盘上已有的 .thumb.bmp → 命中则直接返回
        └─ 入队 pending_ （持锁）
                │
                ▼ （后台 SimpleThread）
        ThumbnailProvider::prepare(file)
          ModelThumbProv: 加载 .model → SuperModel（不碰 D3D）
                │
                ▼ （主线程 tick）
        renderRequested_ = true
        ThumbnailProvider::render(file, rt)
          ModelThumbProv: PyModelRenderer 画一帧到 renderRT_
        保存 renderRT_ → 磁盘 .thumb.bmp
        入队 ready_
                │
                ▼ （下一帧 tick）
        ThumbnailUpdater::thumbManagerUpdate(longText)
          SmartListCtrl 收到通知 → 重绘对应行
```

### 1.6 History / Favourites / DropManager

这三者都是 `UalManager` 上的附属子系统：

- **`UalHistory`**（[ual_history.hpp](file:///workspace/src/lib/ual/ual_history.hpp)）继承自 `XmlItemList`，维护一个有时戳、有上限的最近访问列表。为了支持「拖放失败时回滚」，它采用两阶段提交：

  ```cpp
  void prepareItem( const XmlItem& item );   // EndDrag 时调用
  bool addPreparedItem();                     // 应用确认成功后再真正加入
  void discardPreparedItem();                 // 失败则丢弃
  ```

- **`UalFavourites`**（[ual_favourites.hpp](file:///workspace/src/lib/ual/ual_favourites.hpp)）：用户收藏的资产，同样基于 `XmlItemList`，可分组。
- **`UalDropManager`**（[ual_drop_manager.hpp](file:///workspace/src/lib/ual/ual_drop_manager.hpp)）：管理「拖放目标区域」。宿主编辑器把自己的某个 `HWND`（或控件 ID）注册成一个 drop target，附带一个 `UalDropCallback`；UAL 在拖动过程中遍历所有注册的窗口，命中时高亮其边框，松手时调用 `execute`：

  ```cpp
  class UalDropManager
  {
  public:
      void add( SmartPointer< UalDropCallback > dropping, bool useHighlighting = true );
      void start( const std::string& ext);
      SmartPointer< UalDropCallback > test( HWND hwnd, UalItemInfo* ii );
      SmartPointer< UalDropCallback > test( UalItemInfo* ii );
      bool end( UalItemInfo* ii );
      static CRect HIT_TEST_NONE;       // 不命中
      static CRect HIT_TEST_MISS;       // 命中窗口但不在有效区
      static CRect HIT_TEST_OK_NO_RECT; // 命中但不需要画高亮框
  private:
      DropMap droppings_;               // HWND → 多个 callback
      std::set< HWND > dontHighlightHwnds_;
      CPen pen_;
      CRect highlightRect_;
      HWND lastHighlightWnd_;
  };

  template< class C >
  class UalDropFunctor: public UalDropCallback   // 把成员函数包成 callback
  {
      typedef bool (C::*Method)( UalItemInfo* ii );
      typedef CRect (C::*Method2)( UalItemInfo* ii );  // 可选的命中测试
      /* ... */
  };
  ```

### 1.7 SmartListCtrl —— 虚拟列表控件

[smart_list_ctrl.hpp](file:///workspace/src/lib/ual/smart_list_ctrl.hpp) 中的 `SmartListCtrl` 继承自 MFC `CListCtrl` 并实现 `ThumbnailUpdater`，是一棵「按需请求数据」的虚拟列表：

```cpp
class SmartListCtrl : public CListCtrl, public ThumbnailUpdater
{
public:
    enum ViewStyle { LIST, SMALLICONS, BIGICONS };
    explicit SmartListCtrl( ThumbnailManagerPtr thumbnailManager );
    void init( ListProviderPtr provider, XmlItemVec* customItems = 0,
               bool clearSelection = true );
    void setMaxCache( int maxItems );
    void refresh();
    const AssetInfo getAssetInfo( int index );
    void updateItem( int index, bool removeFromCache = true );
    bool showItem( const AssetInfo& assetInfo );
    void setSearchText( std::string searchText );
    void updateFilters();
    // 拖放
    bool isDragging();
    void updateDrag( int x, int y );
    void endDrag();
    // ThumbnailUpdater
    void thumbManagerUpdate( const std::wstring& longText );
private:
    ViewStyle style_;
    ListProviderPtr provider_;
    ThumbnailManagerPtr thumbnailManager_;
    ListCache listCacheBig_;        // 大图标缓存
    ListCache listCacheSmall_;      // 小图标缓存
    CImageList imgListBig_, imgListSmall_;
    controls::DragImage * dragImage_;
    bool dragging_;
    int thumbWidth_, thumbHeight_;            // 大图标尺寸
    int thumbWidthSmall_, thumbHeightSmall_;  // 小图标尺寸
    int maxItems_;
};
```

它用 MFC 的 owner-data 模式（`OnGetDispInfo` 回调）按行请求数据，再用 `ListCache` 缓存最近访问的若干行的缩略图，避免滚动时反复生成。三个定时器分别负责：选择通知去抖（`SMARTLIST_SELTIMER_MSEC = 50`）、加载进度刷新（`SMARTLIST_LOADTIMER_MSEC = 100`）、重绘请求（`SMARTLIST_REDRAWTIMER_MSEC = 200`）。

---

## 2. Gizmo：3D 操纵器与属性编辑框架

`gizmo/` 是 BigWorld 编辑器的「交互核心」。它解决两类问题：①在 3D 视口里用鼠标直接拖动对象的位置/旋转/缩放；②把对象的可编辑属性统一暴露给属性面板和 Python 脚本。这两件事通过 `GeneralProperty` 这套视图工厂机制紧密耦合。

### 2.1 GizmoManager —— 操纵器调度与命中测试

[gizmo_manager.hpp](file:///workspace/src/lib/gizmo/gizmo_manager.hpp) 定义了操纵器的抽象基类和调度器：

```cpp
class Gizmo : public ReferenceCount
{
public:
    const static int ALWAYS_ENABLED = -1;
    virtual void drawZBufferedStuff( bool force ) {};
    virtual bool draw( bool force ) = 0;
    virtual bool intersects( const Vector3& origin, const Vector3& direction,
                             float& t, bool force ) = 0;
    virtual void click( const Vector3& origin, const Vector3& direction ){};
    virtual void rollOver( const Vector3& origin, const Vector3& direction ){};
    virtual void setVisualOffsetMatrixProxy( MatrixProxyPtr matrix ){};
protected:
    virtual Matrix objectTransform() const = 0;
    virtual Matrix gizmoTransform() const;
    Matrix getCoordModifier() const;       // 世界/本地坐标系变换
    virtual Matrix objectCoord() const;
};
typedef SmartPointer< Gizmo > GizmoPtr;
typedef std::vector< GizmoPtr > GizmoVector;

class GizmoManager
{
public:
    static GizmoManager & instance();
    void draw();
    bool update( const Vector3& worldRay );   // 鼠标移动：更新命中的 gizmo
    bool click();
    bool rollOver();
    void addGizmo( GizmoPtr pGizmo );
    void removeGizmo( GizmoPtr pGizmo );
    void removeAllGizmo();
    void forceGizmoSet( GizmoSetPtr gizmoSet );  // 临时强制只显示一组
    GizmoSetPtr forceGizmoSet();
    const Vector3& getLastCameraPosition();
    PY_MODULE_STATIC_METHOD_DECLARE( py_gizmoUpdate )
    PY_MODULE_STATIC_METHOD_DECLARE( py_gizmoClick )
private:
    GizmoVector gizmos_;
    Vector3 lastWorldRay_;
    Vector3 lastWorldOrigin_;
    GizmoPtr intersectedGizmo_;     // 当前命中的 gizmo
    GizmoSetPtr forcedGizmoSet_;
};
```

`GizmoManager` 不是 `Singleton` 模板派生类，而是手写的 `instance()` 静态方法（私有构造 + 删除拷贝）。`update(worldRay)` 每帧用鼠标射线对所有 gizmo 做命中测试，把命中的那个记到 `intersectedGizmo_`；`click()` / `rollOver()` 把事件转给当前命中的 gizmo。`forceGizmoSet` 用于「临时只显示某组 gizmo 并强制 `force=true`」的场景（例如正在拖动时锁定到当前 gizmo，避免鼠标飘到别的 gizmo 上）。

### 2.2 Gizmo 的类层级

```
Gizmo (ReferenceCount)
├── PositionGizmo     ← 平移：3 轴 + 3 平面 + 自由
├── RotationGizmo     ← 旋转：3 轴圆环
├── ScaleGizmo        ← 缩放：圆环（ALT 键）
├── RadiusGizmo       ← 半径：单轴（用于光源/粒子半径）
├── AngleGizmo        ← 角度：圆弧
├── TileGizmo         ← 地形 tile 选择
├── LinkGizmo         ← 链接：对象间连线
└── CombinationGizmos ← 组合多个 gizmo
```

#### PositionGizmo

[position_gizmo.hpp](file:///workspace/src/lib/gizmo/position_gizmo.hpp) 实现平移操纵器，它有 6 个可交互区域：X/Y/Z 三条轴 + XY/XZ/YZ 三个平面，外加一个「自由移动」中心。每个区域用一个 `PositionShapePart` 描述：

```cpp
class PositionShapePart : public ShapePart
{
public:
    PositionShapePart( const Moo::Colour& col, int axis )   // 平面（轴向 axis）
        : colour_( col ), isFree_( false ), isPlane_( true )
    {
        Vector3 normal( 0, 0, 0 );
        normal[axis] = 1.f;
        planeEq_ = PlaneEq( normal, 0 );
    }
    PositionShapePart( const Moo::Colour& col, const Vector3& direction ) // 轴
        : colour_( col ), isPlane_( false ), isFree_( false ), direction_( direction ) {}
    PositionShapePart( const Moo::Colour& col )             // 自由中心
        : colour_( col ), isFree_( true ) {}
    bool isFree() const; bool isPlane() const;
    const PlaneEq& plane() const; const Vector3& direction() const;
private:
    bool isFree_, isPlane_;
    Moo::Colour colour_;
    PlaneEq planeEq_;
    Vector3 direction_;
};

class PositionGizmo : public Gizmo, public Aligned
{
public:
    PositionGizmo( int disablerModifiers = MODIFIER_SHIFT | MODIFIER_CTRL | MODIFIER_ALT,
                   MatrixProxyPtr matrix = NULL,
                   MatrixProxyPtr visualOffsetMatrix = NULL,
                   float radius = 0.2f,
                   bool isPlanar = false );
    bool draw( bool force );
    bool intersects( const Vector3& origin, const Vector3& direction, float& t, bool force );
    void click( const Vector3& origin, const Vector3& direction );
    void rollOver( const Vector3& origin, const Vector3& direction );
protected:
    virtual void rebuildMesh( bool force );
    Matrix objectTransform() const;
    bool snapToObstableEnabled() const;
    bool snapToTerrainEnabled() const;
    SolidShapeMesh selectionMesh_;          // 命中测试用的几何体
    Moo::VisualPtr drawMesh_[3];            // 三轴可视化
    Moo::VisualPtr pDrawMesh_;              // 平面/自由中心可视化
    PositionShapePart* currentPart_;
    MatrixProxyPtr matrixProxy_;            // 被操纵的矩阵（可空→用 current properties）
    MatrixProxyPtr visualOffsetMatrix_;     // 视觉上偏移到别处
    SnapProvider::SnapMode snapMode_;
    float radius_;
    int disablerModifiers_;
    bool isPlanar_;
};
```

`disablerModifiers` 表示「按下这些修饰键时本 gizmo 失效」，默认 Shift+Ctrl+Alt 全按下时禁用——这是为了在多个 gizmo 同时存在时让用户通过修饰键切换。`matrixProxy_` 为 `NULL` 时，gizmo 不直接操纵某个对象，而是通过 `CurrentPositionProperties`（见 2.5）操纵当前选中的所有对象的 position 属性，从而支持多选平移。

#### RotationGizmo 与 ScaleGizmo

[rotation_gizmo.hpp](file:///workspace/src/lib/gizmo/rotation_gizmo.hpp)：

```cpp
class RotationGizmo : public Gizmo, public Aligned
{
public:
    RotationGizmo( MatrixProxyPtr pMatrix,
                   int enablerModifier = MODIFIER_SHIFT,                  // 默认按 SHIFT 才出现
                   int disablerModifier = MODIFIER_CTRL | MODIFIER_ALT );
    /* draw / intersects / click / rollOver 同上 */
    Matrix getCoordModifier() const;
protected:
    SolidShapeMesh selectionMesh_;
    Moo::VisualPtr drawMesh_;
    RotationShapePart* currentPart_;
    MatrixProxyPtr pMatrix_;
    Moo::Colour lightColour_;
    int enablerModifier_, disablerModifier_;
};
```

[scale_gizmo.hpp](file:///workspace/src/lib/gizmo/scale_gizmo.hpp)：

```cpp
class ScaleGizmo : public Gizmo, public Aligned
{
public:
    enum Type
    {
        SCALE_XYZ,   // 任意方向缩放
        SCALE_UV     // 只在 uv（x-z）平面缩放
    };
    ScaleGizmo( MatrixProxyPtr pMatrix,
                int enablerModifier = MODIFIER_ALT,                       // 默认按 ALT 才出现
                float scaleSpeedFactor = 1.0f,
                Type type = SCALE_XYZ );
    /* ... */
private:
    ScaleFloatProxyPtr scaleXFloatProxy_, scaleYFloatProxy_, scaleZFloatProxy_;
    ScaleMatrixProxyPtr scaleMatrixProxy_;
    Type type_;
    float scaleSpeedFactor_;
};
```

三者的修饰键约定是 BigWorld 编辑器的「肌肉记忆」：**Shift = 旋转，Alt = 缩放，无修饰键 = 平移**。

### 2.3 GeneralEditor / GeneralProperty —— 属性系统的核心

这是 `gizmo/` 里最具架构美感的一块。任意一个可被编辑器编辑的对象（chunk item、model、粒子发射器……）都通过 `GeneralEditor` 暴露一组 `GeneralProperty`，而每种 `GeneralProperty` 又能生成多种「视图」（Gizmo 视图、文本视图、Borland 控件视图、Python 视图……），视图之间用「视图工厂 + view kind id」解耦。

[general_editor.hpp](file:///workspace/src/lib/gizmo/general_editor.hpp)：

```cpp
class GeneralEditor : public PyObjectPlus
{
    Py_Header( GeneralEditor, PyObjectPlus )
public:
    GeneralEditor( PyTypePlus * pType = &s_type_ );
    virtual void addProperty( GeneralProperty * pProp );

    void elect();      // 把自己设为「当前编辑器」，触发所有 property 的视图 elect
    void expel();      // 反向

    typedef std::vector< GeneralEditorPtr > Editors;
    static const Editors & currentEditors( void );                // 当前选中的编辑器列表
    static bool settingMultipleEditors() { return s_settingMultipleEditors_; }
    static void currentEditors( const Editors & editors );        // 多选时设置多个
    static void createViews( bool doCreate ) { s_createViews_ = doCreate; }
protected:
    typedef std::vector< GeneralProperty * > PropList;
    PropList properties_;
private:
    static Editors s_currentEditors_;
    static bool s_settingMultipleEditors_;
    static bool s_createViews_;
};
```

`GeneralProperty` 是所有属性的基类，内嵌「视图」抽象与视图工厂宏：

```cpp
class GeneralProperty
{
public:
    GeneralProperty( const std::string & name, const std::wstring & uiname = L"" );
    void WBEditable( bool editable );
    void UIName( const std::wstring& name );
    void UIDesc( const std::wstring& name );
    void setGroup( const std::wstring & groupName );

    virtual const ValueType & valueType() const { RETURN_VALUETYPE( UNKNOWN ); }
    virtual PyObject * EDCALL pyGet();
    virtual int EDCALL pySet( PyObject * value, bool transient = false,
                                               bool addBarrier = true );
    virtual void elect();
    virtual void elected();
    virtual void expel();
    virtual void select();

    class View
    {
    public:
        virtual void EDCALL elect() = 0;
        virtual void EDCALL expel() = 0;
        virtual void EDCALL select() = 0;
        virtual bool isEditorView() const { return false; }
        virtual void EDCALL lastElected() { ERROR_MSG( "..." ); }
    };
    static int nextViewKindID();
protected:
    Views views_;        // 按 viewKindID 索引的视图数组
    GENPROPERTY_VIEW_FACTORY_DECLARE( GeneralProperty )
};
```

视图工厂用一组宏实现「每种属性类有一张 `ViewFactory` 函数指针表，按 viewKindID 索引」：

```cpp
#define GENPROPERTY_VIEW_FACTORY_DECLARE( PROPCLASS )                    \
    public:                                                              \
        typedef View * (*ViewFactory)( PROPCLASS & prop );               \
        static void registerViewFactory( int vkid, ViewFactory fn );    \
    private:                                                             \
        static std::vector<ViewFactory> * viewFactories_;

#define GENPROPERTY_VIEW_FACTORY( PROPCLASS )                            \
    std::vector<PROPCLASS::ViewFactory> * PROPCLASS::viewFactories_ = NULL; \
    void PROPCLASS::registerViewFactory( int vkid, ViewFactory fn )      \
    {                                                                    \
        if (viewFactories_ == NULL) viewFactories_ = new std::vector<ViewFactory>; \
        while (int(viewFactories_->size()) <= vkid) viewFactories_->push_back( NULL ); \
        (*viewFactories_)[ vkid ] = fn;                                  \
    }
```

每个属性子类在自己的构造函数里调用 `GENPROPERTY_MAKE_VIEWS()`，遍历自己的 `viewFactories_`，为每个非空工厂创建一个 `View` 实例并塞进 `views_`。一个具体的 View（比如 `GenPositionProperty` 的 gizmo 视图）在 `elect()` 时把自己注册到全局的「当前属性集合」里，gizmo 据此知道该操纵哪些属性。

### 2.4 GeneralProperty 的具体子类与 MatrixProxy

[general_properties.hpp](file:///workspace/src/lib/gizmo/general_properties.hpp) 实现了一组常用属性，它们都围绕 `MatrixProxy` 这个「矩阵读写代理」展开：

```cpp
class MatrixProxy : public ReferenceCount
{
public:
    typedef Matrix Data;
    virtual void EDCALL getMatrix( Matrix & m, bool world = true ) = 0;
    virtual void EDCALL getMatrixContext( Matrix & m ) = 0;
    virtual void EDCALL getMatrixContextInverse( Matrix & m ) = 0;
    virtual bool EDCALL setMatrix( const Matrix & m ) = 0;
    virtual void EDCALL recordState() = 0;                       // 拖动开始：记下初始状态
    virtual bool EDCALL commitState( bool revertToRecord = false,
                                      bool addUndoBarrier = true ) = 0;  // 拖动结束：提交/回滚
    virtual bool EDCALL hasChanged() = 0;
    static MatrixProxyPtr getChunkItemDefault( ChunkItemPtr pItem );
};

class GenPositionProperty : public GeneralProperty
{
public:
    GenPositionProperty( const std::string & name, MatrixProxyPtr pMatrix, float size = 1000000.f );
    virtual const ValueType & valueType() const { RETURN_VALUETYPE( VECTOR ); }
    MatrixProxyPtr pMatrix() { return pMatrix_; }
private:
    MatrixProxyPtr pMatrix_;
    float size_;
    GENPROPERTY_VIEW_FACTORY_DECLARE( GenPositionProperty )
};
// 类似的还有 GenRotationProperty / GenScaleProperty / GenMatrixProperty
```

`MatrixProxy` 把「如何修改一个对象的世界矩阵」这件事完全抽象出来：`recordState` / `commitState` 是为撤销重做服务的——拖动开始时记录初值，拖动结束时提交一个 `UndoRedo::Operation`。`getChunkItemDefault` 是个工厂方法，为普通的 `ChunkItem` 生成默认的 `MatrixProxy`。

`ValueType`（[value_type.hpp](file:///workspace/src/lib/gizmo/value_type.hpp)）则是一个枚举式类型描述符，用于属性面板选择合适的编辑控件：

```cpp
namespace ValueTypeDesc
{
    enum Desc { UNKNOWN, BOOL, INT, FLOAT, STRING, VECTOR, MATRIX,
                COLOUR, FILEPATH, DATE_STRING };
}
class ValueType   // 包装器，持有一个 BaseValueType*
{
public:
    bool isNumeric() const;   // INT 或 FLOAT
    bool isValid() const;
    bool toString( int v, std::string & ret ) const;
    bool fromString( const std::string & v, int & ret ) const;
};
```

### 2.5 CurrentGeneralProperties —— 多选属性聚合

多选时，多个对象的同类属性需要被「合并」成一个虚拟属性供 gizmo 操纵。[current_general_properties.hpp](file:///workspace/src/lib/gizmo/current_general_properties.hpp) 用模板 `PropertyCollator` 实现这一点：

```cpp
template<class PROPERTY>
class PropertyCollator : public GeneralProperty::View
{
public:
    PropertyCollator( PROPERTY& prop ) : prop_( prop ) {}
    virtual void EDCALL elect()  { properties_.push_back( &prop_ ); }
    virtual void EDCALL expel()  { properties_.erase( std::find( properties_.begin(),
                                                                 properties_.end(), &prop_ ) ); }
    virtual void EDCALL select() {}
    static std::vector<PROPERTY*> properties() { return properties_; }
protected:
    static std::vector<PROPERTY*> properties_;
private:
    PROPERTY& prop_;
};

class CurrentPositionProperties : public PropertyCollator<GenPositionProperty>
{
public:
    CurrentPositionProperties( GenPositionProperty& prop ) : PropertyCollator<GenPositionProperty>( prop ) {}
    static GeneralProperty::View * create( GenPositionProperty & prop )
    { return new CurrentPositionProperties( prop ); }
    static Vector3 averageOrigin();   // 所有选中对象的平均原点（gizmo 摆放位置）
private:
    static struct ViewEnroller
    {
        ViewEnroller()
        {
            GenPositionProperty_registerViewFactory(
                GeneralProperty::nextViewKindID(), &create );
        }
    } s_viewEnroller;
};
// CurrentRotationProperties / CurrentScaleProperties 同理
```

精妙之处在于 `ViewEnroller`：它是一个静态嵌套结构体，其构造函数在模块加载时自动执行，把 `create` 注册为 `GenPositionProperty` 的一个 view factory。于是只要 `gizmo` 库被链接进来，「位置属性 → 当前位置属性集合」这条视图链路就自动接通了，无需任何手动注册代码。`averageOrigin` 给 gizmo 提供摆放位置——多选时 gizmo 画在所有对象平均原点处。

### 2.6 Tool / ToolManager —— 工具栈

除了「对象上的 gizmo」，编辑器还有一类「全局工具」：地形抬升刷子、选择工具、涂色工具等。这些走 [tool.hpp](file:///workspace/src/lib/gizmo/tool.hpp) 的 `Tool` 体系：

```cpp
class Tool : public InputHandler, public PyObjectPlus
{
    Py_Header( Tool, PyObjectPlus )
public:
    Tool( ToolLocatorPtr locator, ToolViewPtr view,
          ToolFunctorPtr functor, PyTypePlus * pType = &s_type_ );
    virtual void onPush();    // 入栈时回调
    virtual void onPop();
    virtual void onBeginUsing();
    virtual void onEndUsing();
    virtual void calculatePosition( const Vector3& worldRay );
    virtual void update( float dTime );
    virtual void render();
    virtual bool handleKeyEvent( const KeyEvent & event );
    virtual bool handleMouseEvent( const MouseEvent & event );
    virtual bool handleContextMenu();
    virtual bool applying() const;
    void findRelevantChunks( float buffer = 0.f );   // 找出影响范围内的 chunk
private:
    float size_, strength_;
    ChunkPtr currentChunk_;
    ChunkPtrVector relevantChunks_;
    ToolViewPtrs view_;            // 多个命名视图
    ToolLocatorPtr locator_;       // 决定「作用点在哪」
    ToolFunctorPtr functor_;       // 决定「做什么」
};
typedef SmartPointer<Tool> ToolPtr;
```

`Tool` 是「Locator + View + Functor」三件套的组合：Locator 把鼠标射线换算成世界坐标/作用点，View 负责画刷子光标，Functor 负责实际改变场景数据。`ToolManager`（[tool_manager.hpp](file:///workspace/src/lib/gizmo/tool_manager.hpp)）用一个栈管理当前工具：

```cpp
class ToolManager
{
public:
    static ToolManager & instance();
    void pushTool( ToolPtr tool );   // 新工具入栈（如进入「绘制模式」）
    void popTool();
    ToolPtr tool();
    void changeSpace( const Vector3& worldRay );
private:
    typedef std::vector<ToolPtr> ToolStack;
    ToolStack tools_;
};
```

栈式设计支持「临时切换工具」：比如在地形编辑中按住某键临时切到选择工具，松开后再 `popTool` 回到地形刷。

### 2.7 SnapProvider / CoordModeProvider —— 吸附与坐标系

[snap_provider.hpp](file:///workspace/src/lib/gizmo/snap_provider.hpp) 定义吸附策略接口（手写单例）：

```cpp
class SnapProvider
{
public:
    enum SnapMode { SNAPMODE_XYZ, SNAPMODE_TERRAIN, SNAPMODE_OBSTACLE };
    virtual SnapMode snapMode() const { return SNAPMODE_XYZ; }
    virtual bool snapPosition( Vector3& v ) { return true; }          // 吸附绝对位置（网格）
    virtual Vector3 snapNormal( const Vector3& v ) { return Vector3( 0, 1, 0 ); }
    virtual void snapPositionDelta( Vector3& v ) {}                   // 吸附位移增量
    virtual void snapAngles( Matrix& v ) {}                           // 吸附旋转角
    virtual float angleSnapAmount() { return 0.f; }
    static SnapProvider* instance();
    static void instance( SnapProvider* sp );   // 注入自定义实现
private:
    static SnapProvider* s_instance_;
    static bool s_internal_;
};
```

`instance(sp)` 允许宿主编辑器注入自己的 `SnapProvider` 子类（例如 WorldEditor 注入「吸附到地形」的实现），否则用默认空实现。`CoordModeProvider`（[coord_mode_provider.hpp](file:///workspace/src/lib/gizmo/coord_mode_provider.hpp)）类似，决定 gizmo 用世界坐标还是对象本地坐标——`Gizmo::getCoordModifier()` 会读它。

### 2.8 UndoRedo —— 撤销/重做

[undoredo.hpp](file:///workspace/src/lib/gizmo/undoredo.hpp) 实现编辑器的撤销重做栈。核心是 `Operation` 抽象与「barrier（屏障）」概念：

```cpp
class UndoRedo
{
public:
    bool canUndo() const;
    bool canRedo() const;
    void setSavePoint() { savePoint_ = undoList_.size(); }   // 标记「已保存」位置
    bool needsSave() const { return (savePoint_ != undoList_.size()); }

    class Environment   // 用于记录操作发生时的环境（如选中集）
    {
    public:
        class Operation : public ReferenceCount { virtual void exec() = 0; };
        static Operations record();
        static void replay( const Operations& ops );
    };
    class Operation     // 一次撤销操作（两个 barrier 之间）
    {
    public:
        Operation( int kind ) : kind_( kind ) { env_ = Environment::record(); }
        virtual void undo() = 0;
        virtual bool iseq( const Operation & oth ) const = 0;  // 判断是否「同一对象同一数据」
        bool operator==( const Operation & oth ) const;
    private:
        int kind_;
        Environment::Operations env_;
    };
};
```

「barrier」是把多个 `Operation` 打包成一个撤销单元的机制——比如「批量移动 10 个对象」对应 10 个 `Operation`，但用户按一次 Ctrl+Z 应该全部撤销，所以会用一个 barrier 把它们包起来。`MatrixProxy::commitState(addUndoBarrier)` 的最后一个参数就是控制提交时是否加 barrier。

---

## 3. Duplo：附加对象、马达与模型渲染

`duplo/` 这个名字（乐高得宝）暗示了它的定位：把「可附加到场景/模型的动态对象」像积木一样拼起来。它既是运行时客户端模型系统的实现，也是工具侧（缩略图、模型预览）的基础。

### 3.1 PyAttachment / ChunkEmbodiment / ChunkAttachment 体系

`PyAttachment`（[py_attachment.hpp](file:///workspace/src/lib/duplo/py_attachment.hpp)）是所有「可被附加到某物上的脚本对象」的基类；`ChunkEmbodiment`（[chunk_embodiment.hpp](file:///workspace/src/lib/duplo/chunk_embodiment.hpp)）是「在 chunk 里有一个实体化身」的抽象。`ChunkAttachment`（[chunk_attachment.hpp](file:///workspace/src/lib/duplo/chunk_attachment.hpp)）把这两者桥接起来：

```cpp
class ChunkAttachment : public ChunkDynamicEmbodiment, public MatrixLiaison,
    public Aligned
{
public:
    ChunkAttachment();
    ChunkAttachment( PyAttachmentPtr pAttachment );
    virtual void tick( float dTime );
    virtual void draw();
    virtual void toss( Chunk * pChunk );          // 被扔进/扔出某个 chunk

    void enterSpace( ChunkSpacePtr pSpace, bool transient );
    void leaveSpace( bool transient );
    virtual void move( float dTime );

    virtual const Matrix & worldTransform() const { return worldTransform_; }
    virtual const Matrix & getMatrix() const       { return worldTransform_; }
    virtual bool setMatrix( const Matrix & m );

    virtual void localBoundingBox( BoundingBox & bb, bool skinny = false ) const;
    virtual void worldBoundingBox( BoundingBox & bb, const Matrix& world, bool skinny = false ) const;

    PyAttachment * pAttachment() const { return (PyAttachment*)&*pPyObject_; }
    static int convert( PyObject * pObj, ChunkEmbodimentPtr & pNew, const char * varName );
private:
    bool needsSync_;
    Matrix worldTransform_;
    bool inited_;
};
```

`MatrixLiaison` 是「矩阵联络人」接口——`PyAttachment` 通过它读写自己的世界矩阵。`ChunkAttachment` 把一个 `PyAttachment`（可能是 `PyModel`、粒子系统等）变成 chunk 里的一个动态条目，负责 `tick`/`draw`/`toss` 的生命周期，并维护 `worldTransform_`。

### 3.2 PyModel —— 脚本可见的模型

[pymodel.hpp](file:///workspace/src/lib/duplo/pymodel.hpp) 中的 `PyModel` 是 duplo 最核心的类，它既是 `PyAttachment`（可被附加到实体或 chunk），又是脚本里 `BigWorld.Model(...)` 的返回类型：

```cpp
class PyModel : public PyAttachment, public Aligned
{
    Py_Header( PyModel, PyAttachment )
public:
    // PyAttachment overrides
    void tick( float dTime );
    void draw( const Matrix & worldTransform, float lod );
    virtual void tossed( bool outside );

    // 脚本属性
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, visible, visible )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, outsideOnly, outsideOnly )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, stipple, stipple )      // 闪烁（半透明）
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, shimmer, shimmer )      // 扭曲（热气效果）
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, moveScale, moveScale )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, actionScale, actionScale )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, height, height )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( float, yaw, yaw )

    // Motor 管理
    PY_METHOD_DECLARE_WITH_DOC( py_addMotor, "add a Motor" );
    PY_METHOD_DECLARE_WITH_DOC( py_delMotor, "remove a Motor" );
    PyObject * pyGet_motors();
    int pySet_motors( PyObject * value );
    Motor * motorZero() { return motors_.size() > 0 ? motors_[0] : NULL; }

    // 几何辅助
    void straighten();
    void rotate( float angle, const Vector3 & vAxis, const Vector3 & vCentre = Vector3::zero() );
    void alignTriangle( const Vector3 & v0, const Vector3 & v1, const Vector3 & v2, bool randomYaw = true );
    Vector3 reflectOffTriangle( const Vector3 & v0, const Vector3 & v1, const Vector3 & v2, const Vector3 & fwd );
    void zoomExtents( bool upperFrontFlushness, float xzMutiplier = 1.f );

    // 节点/hardpoint
    PyObject * node( const std::string & nodeName, MatrixProviderPtr local = NULL );
    PY_RO_ATTRIBUTE_DECLARE( SmartPointer<PyObject>( this->node(""), true ), root );
private:
    SuperModel * pSuperModel_;          // 底层模型资源
    ActionQueue actionQueue_;           // 动作队列
    BoundingBox localBoundingBox_, localVisibilityBox_;
    Fashions fashions_;                 // PyDye / ModelAligner 等
    PyModelNodes knownNodes_;           // 缓存的节点
    Matrix unitTransform_;
    Vector3 unitOffset_;
    float unitRotation_;
    typedef std::vector<Motor*> Motors;
    Motors motors_;                     // 马达列表（每帧 rev 一次）
    Matrix initialItinerantContext_, initialItinerantContextInverse_;
    SmartPointer<PyObject> tossCallback_;
    static PyModel *s_pCurrent_;
};
```

关键点：

- **SuperModel 是底层资源**：`PyModel` 持有一个 `SuperModel*`（来自 `model/` 库），后者是多个 `.model` 文件拼成的「超级模型」。`BigWorld.Model("body.model","head.model")` 会构造一个 `SuperModel`。
- **Motor 列表驱动运动**：每帧 `PyModel::tick` 遍历 `motors_`，调用 `Motor::rev(dTime)`，马达在此修改模型的位置/朝向/动画。默认马达是 `ActionMatcher`（在客户端 entity 代码里设置），它根据实体运动速度自动选 Idle/Walk/Run 动作。
- **Fashion 系统**：`fashions_` 是 `StringHashMap<SmartPointer<PyFashion>>`，按名字索引染料（`PyDye`）、对齐器（`ModelAligner`）等。`model.Chest = "Red"` 这种脚本写法就是往 `fashions_` 里塞一个 `PyDye`。
- **后台加载**：`ScheduledModelLoader`（同文件）继承 `BackgroundTask`，在后台线程加载模型资源，加载完再切回主线程构造 `PyModel`，避免大模型卡住主线程。
- **`zoomExtents`**：把模型缩放到 1 立方米内，这正是 `PyModelRenderer` 用来摆放预览模型的方法。

### 3.3 Motor 马达族

[motor.hpp](file:///workspace/src/lib/duplo/motor.hpp) 是抽象基类：

```cpp
class Motor : public PyObjectPlus
{
    Py_Header( Motor, PyObjectPlus )
public:
    Motor( PyTypePlus * pType ) : PyObjectPlus( pType ), pOwner_( NULL ) {}
    virtual void attach( PyModel * pOwner ) { pOwner_ = pOwner; this->attached(); }
    virtual void detach() { pOwner_ = NULL; this->detached(); }
    virtual void rev( float dTime ) = 0;   // 每帧调用，驱动模型
    PyModel * pOwner() { return pOwner_; }
protected:
    PyModel * pOwner_;   // 不持有引用，避免循环引用
};
```

注意 `pOwner_` 是裸指针且不增引用计数——因为 `PyModel` 持有 `Motor`，若 `Motor` 反过来持有 `PyModel` 就会循环引用。`duplo` 自带一组马达：

| 马达类 | 头文件 | 行为 |
|--------|--------|------|
| `Homer` | [homer.hpp](file:///workspace/src/lib/duplo/homer.hpp) | 朝目标 `MatrixProvider` 直线追踪，可设 proximity 回调 |
| `Orbitor` | [orbitor.hpp](file:///workspace/src/lib/duplo/orbitor.hpp) | 接近目标后进入环绕轨道，带 wobble 抖动 |
| `LinearHomer` | [linear_homer.hpp](file:///workspace/src/lib/duplo/linear_homer.hpp) | 线性匀速追踪 |
| `Oscillator` | [oscillator.hpp](file:///workspace/src/lib/duplo/oscillator.hpp) | 绕轴正弦摆动 |
| `Bouncer` | [bouncer.hpp](file:///workspace/src/lib/duplo/bouncer.hpp) | 弹跳 |
| `Tracker` | [tracker.hpp](file:///workspace/src/lib/duplo/tracker.hpp) | 跟踪另一个对象朝向 |
| `Propellor` | [propellor.hpp](file:///workspace/src/lib/duplo/propellor.hpp) | 螺旋桨旋转 |
| `Servo` | [servo.hpp](file:///workspace/src/lib/duplo/servo.hpp) | 伺服朝向目标 |

以 `Homer` 为例：

```cpp
class Homer : public Motor
{
    Py_Header( Homer, Motor )
public:
    virtual void rev( float dTime );
    PY_RW_ATTRIBUTE_DECLARE( target_, target )            // MatrixProvider 目标
    PY_RW_ATTRIBUTE_DECLARE( speed_, speed )
    PY_RW_ATTRIBUTE_DECLARE( turnRate_, turnRate )        // 弧度/秒
    PY_RW_ATTRIBUTE_DECLARE( turnAxis_, turnAxis )
    PY_RW_ATTRIBUTE_DECLARE( tripTime_, tripTime )        // 启动后多久开始减速
    PY_RW_ATTRIBUTE_DECLARE( goneTime_, goneTime )
    PY_RW_ATTRIBUTE_DECLARE( zenithed_, zenithed )
    PY_RW_ATTRIBUTE_DECLARE( proximity_, proximity )      // 进入此距离触发回调
    PY_RW_ATTRIBUTE_DECLARE( proximityCallback_, proximityCallback )
    PY_RW_ATTRIBUTE_DECLARE( arrivalCountLimit_, arrivalCountLimit )
    PY_RW_ATTRIBUTE_DECLARE( scale_, scale )              // 接近时缩放
    PY_RW_ATTRIBUTE_DECLARE( scaleTime_, scaleTime )
private:
    MatrixProviderPtr target_;
    float speed_, turnRate_, tripTime_, goneTime_, proximity_, scale_, scaleTime_;
    Vector3 turnAxis_;
    bool zenithed_;
    PyObject * proximityCallback_;
    int arrivalCount_, arrivalCountLimit_;
    void makePCBHappen();   // 触发 proximity 回调
};
```

`rev(dTime)` 的典型逻辑：从 `target_` 取目标矩阵，计算到自身的方向与距离，按 `speed_` 朝目标移动一段，按 `turnRate_` 转向，若距离 < `proximity_` 则调用 `makePCBHappen` 触发 Python 回调。`Orbitor` 在此基础上多了「进入 `spinRadius_` 后改为环绕」的状态机，并带 `wobbleFreq_`/`wobbleMax_` 做正弦抖动。

### 3.4 PyModelRenderer —— 模型渲染到纹理

[py_model_renderer.hpp](file:///workspace/src/lib/duplo/py_model_renderer.hpp) 是 duplo 与 ashes 的桥梁，它把 `PyModel` 画到一张 `PyTextureProvider`，供 `SimpleGUIComponent` 显示：

```cpp
class PyModelRenderer : public PyObjectPlusWithWeakReference, public TextureRenderer
{
    Py_Header( PyModelRenderer, PyObjectPlusWithWeakReference )
public:
    class ModelHolder : public PySTLSequenceHolder<ChunkEmbodiments>
    {
    public:
        ModelHolder( ChunkEmbodiments & seq, PyObject * pOwnerIn, bool writableIn );
        int insert( PyObject * pItem );
    };

    PyModelRenderer( int width, int height, PyTypePlus * pType = &s_type_ );
    virtual void renderSelf( float dTime );

    bool dynamic() const            { return dynamic_; }
    void dynamic( bool b );

    PY_AUTO_METHOD_DECLARE( RETVOID, render, OPTARG( float, 0.f, END ) )
    PY_DEFERRED_ATTRIBUTE_DECLARE( models )
    PyObject * pyGet_texture();
    PY_RO_ATTRIBUTE_SET( texture )
    PY_RW_ACCESSOR_ATTRIBUTE_DECLARE( bool, dynamic, dynamic )
private:
    ChunkEmbodiments models_;
    ModelHolder modelsHolder_;
    bool dynamic_;
public:
    static ChunkSpacePtr s_pSpace_;                  // 渲染用的独立 chunk space
    static Moo::LightContainerPtr s_lighting_;
    static Moo::LightContainerPtr s_specularLighting_;
};
```

头文件 docstring 给出了典型用法，值得逐字引用：

```cpp
# create a new model to render
mod = BigWorld.Model( "objects/models/characters/biped.model" )
# create the model renderer
mr = BigWorld.PyModelRenderer( 256, 256 )
mr.models = [ mod ]
# the pyModelRenderer camera is setup well to look at a unit cube.
mod.zoomExtents()
# dynamic=1 每帧更新（动画/马达）；dynamic=0 需手动 render()
mr.dynamic = 0
mr.render()
# 用 SimpleGUIComponent 显示
cmp = GUI.Simple( "somedummytexture.bmp" )
cmp.texture = mr.texture
GUI.addRoot( cmp )
```

要点：

- **相机固定**：相机在 `(0,1,-1)` 朝 `(0,-1,1)` 看，专门为「单位立方体」设计，所以脚本要先 `zoomExtents()` 把模型缩放到 1 立方米。
- **`dynamic` 模式**：`true` 时每帧 `renderSelf` 被调用（因为有马达/动画），`false` 时只在显式 `render()` 时画一次，省 GPU。
- **静态 chunk space**：`s_pSpace_` 是一个独立的 `ChunkSpace`，模型不进入真实世界，而是在这个离屏 space 里渲染，因此 docstring 说「模型必须未被附加到任何东西」。
- **用途**：①游戏内角色头像、物品图标；②工具侧的模型缩略图（`ModelThumbProv` 走类似路径）。

### 3.5 ShadowCaster —— 阴影缓冲

[shadow_caster.hpp](file:///workspace/src/lib/duplo/shadow_caster.hpp) 实现浮点阴影缓冲（shadow buffer，区别于 stencil shadow volume）：

```cpp
class ShadowCaster : public ReferenceCount, public Aligned
{
public:
    ShadowCaster();
    bool init( ShadowCasterCommon* pCommon, bool halfRes, uint32 idx,
               Moo::RenderTarget* pDepthParent );
    void begin( const BoundingBox& worldSpaceBB, const Vector3& lightDirection );
    void end();
    void beginReceive( bool useTerrain = true );
    void endReceive();
    void setupMatrices( const BoundingBox& worldSpaceBB, const Vector3& lightDirection );
    void angleIntensity( float angleIntensity ) { angleIntensity_ = angleIntensity; }
    const BoundingBox& aaShadowVolume() const { return aaShadowVolume_; }
    Moo::RenderTarget* shadowTarget() { return pShadowTarget_.getObject(); }
    void setupConstants( Moo::EffectMaterial& effect );   // 把阴影矩阵注入 effect
private:
    void pickReceivers();
    Matrix view_, projection_, viewport_, shadowProjection_, viewProjection_;
    BoundingBox aaShadowVolume_;       // 阴影影响范围 AABB
    RECT scissorRect_;                 // 裁剪矩形，限制渲染区域
    ShadowCasterCommon* pCommon_;
    typedef std::vector<ChunkItemPtr> ShadowItems;
    ShadowItems shadowItems_;          // 参与投射/接收的 chunk item
    Moo::RenderTargetPtr pShadowTarget_;
    float angleIntensity_;
    uint32 shadowBufferSize_;
};
```

`begin`/`end` 包裹「从光源视角渲染深度」的 pass，`beginReceive`/`endReceive` 包裹「用阴影纹理给场景着色」的 pass。`setupMatrices` 根据光源方向和场景 AABB 计算光源视图/投影矩阵，`scissorRect_` 把渲染裁剪到只覆盖阴影区域，提高性能。`ShadowCasterCommon`（[shadow_caster_common.hpp](file:///workspace/src/lib/duplo/shadow_caster_common.hpp)）是多个 `ShadowCaster` 共享的资源池（如共用深度父 RT）。这套机制既用于运行时，也用于编辑器的阴影预览。

### 3.6 DrawOverride —— 渲染覆盖层

duplo 提供一组 `*DrawOverride` 来在不改模型材质的前提下「整体改写渲染方式」，用于编辑器的高亮/特殊显示：

| 类 | 头文件 | 作用 |
|----|--------|------|
| `StippleDrawOverride` | [stipple_draw_override.hpp](file:///workspace/src/lib/duplo/stipple_draw_override.hpp) | 闪烁半透明（选中态/不可放置提示） |
| `ShimmerDrawOverride` | [shimmer_draw_override.hpp](file:///workspace/src/lib/duplo/shimmer_draw_override.hpp) | 热气扭曲 |
| `MaterialDrawOverride` | [material_draw_override.hpp](file:///workspace/src/lib/duplo/material_draw_override.hpp) | 替换材质（如全白/全红高亮） |

它们都实现一个 `draw` 接口，在 `PyModel::draw` 流程里被查询。

### 3.7 其他 duplo 组件

- **`SkeletonCollider`**（[skeleton_collider.hpp](file:///workspace/src/lib/duplo/skeleton_collider.hpp)）：把骨骼当成胶囊体做碰撞，用于角色与场景的物理交互。
- **`FootPrintRenderer` / `FootTrigger`**（[foot_print_renderer.hpp](file:///workspace/src/lib/duplo/foot_print_renderer.hpp)、[foot_trigger.hpp](file:///workspace/src/lib/duplo/foot_trigger.hpp)）：脚印渲染与脚步触发（脚步声、扬尘）。
- **`Decal`**（[decal.hpp](file:///workspace/src/lib/duplo/decal.hpp)）：弹孔、血迹等贴花。
- **`PySplodge`**（[py_splodge.hpp](file:///workspace/src/lib/duplo/py_splodge.hpp)）：地面湿/干痕迹。
- **`PyLoft`**（[py_loft.hpp](file:///workspace/src/lib/duplo/py_loft.hpp)）：样条放样生成几何。
- **`PyMorphControl`**（[py_morph_control.hpp](file:///workspace/src/lib/duplo/py_morph_control.hpp)）：形态键控制。
- **`ChunkDynamicObstacle`**（[chunk_dynamic_obstacle.hpp](file:///workspace/src/lib/duplo/chunk_dynamic_obstacle.hpp)）：动态物体的碰撞体注册到 chunk。
- **`BoxAttachment`**（[box_attachment.hpp](file:///workspace/src/lib/duplo/box_attachment.hpp)）：把一个 AABB 附加到节点上，用于简单碰撞/触发。

---

## 4. Ashes：与工具的衔接

`ashes/` 库本身已在 [17-GUI与UI栈](file:///workspace/study-docs/17-GUI与UI栈-gui-scaleform-web.md) 详细分析，这里只重述它与工具子系统的关系：

1. **缩略图回显**：`ThumbnailManager` 生成的缩略图最终以 `CImage` 形式交给 `SmartListCtrl` 显示，但模型缩略图的「渲染到 RT」这一步走的是 `moo` 的 `RenderTarget`，与 `ashes` 不直接相关。
2. **模型预览面板**：编辑器里「选中一个 model，右侧出现旋转预览」的面板，本质是一个 `SimpleGUIComponent`，其 `texture` 来自 `PyModelRenderer.texture`（一个 `PyTextureProvider`）。这条链路是 `duplo → ashes` 的典型依赖。
3. **GDI 风格绘制**：`ashes` 的 `simple_gui.hpp` 提供的像素↔裁剪空间换算、裁剪区栈，被工具的某些自绘控件复用。

因此，工具侧对 `ashes` 的依赖是「以纹理形式接入运行时 UI 渲染」，而不是直接调用 `SimpleGUIComponent` 的全部 API。

---

## 5. OpenAutomate：自动化测试包装

`open_automate/` 是把 BigWorld 客户端接入 Futuremark OpenAutomate（一个显卡基准/兼容性测试框架）的薄包装层。客户端以 `-openautomate` 命令行参数启动后，进入被外部测试主机驱动的模式。

### 5.1 BWOpenAutomate —— 枚举与全局状态

[open_automate_wrapper.hpp](file:///workspace/src/lib/open_automate/open_automate_wrapper.hpp) 中的 `BWOpenAutomate` 是一组枚举与全局标志的容器：

```cpp
class BWOpenAutomate
{
public:
    static const std::string OPEN_AUTOMATE_ARGUMENT;        // "-openautomate"
    static const std::string OPEN_AUTOMATE_TEST_ARGUMENT;   // "INTERNAL_TEST"

    typedef enum
    {
        BWOA_CMD_EXIT                = 0,
        BWOA_CMD_RUN                 = 1,
        BWOA_CMD_GET_ALL_OPTIONS     = 2,
        BWOA_CMD_GET_CURRENT_OPTIONS = 3,
        BWOA_CMD_SET_OPTIONS         = 4,
        BWOA_CMD_GET_BENCHMARKS      = 5,
        BWOA_CMD_RUN_BENCHMARK       = 6,
        BWOA_CMD_RESTART             = 10000,
        BWOA_CMD_AUTO_DETECT         = 10001,
    } eOACommandType;
    PY_BEGIN_ENUM_MAP( eOACommandType, BWOA_ )
        PY_ENUM_VALUE( BWOA_CMD_EXIT )
        PY_ENUM_VALUE( BWOA_CMD_RUN )
        /* ... */
    PY_END_ENUM_MAP()

    typedef enum
    {
        BWOA_TYPE_INVALID = 0, BWOA_TYPE_STRING = 1, BWOA_TYPE_INT = 2,
        BWOA_TYPE_FLOAT = 3, BWOA_TYPE_ENUM = 4, BWOA_TYPE_BOOL = 5
    } eOAOptionDataType;

    typedef enum
    {
        BWOA_COMP_OP_INVALID = 0, BWOA_COMP_OP_EQUAL = 1, BWOA_COMP_OP_NOT_EQUAL = 2,
        BWOA_COMP_OP_GREATER = 3, BWOA_COMP_OP_LESS = 4,
        BWOA_COMP_OP_GREATER_OR_EQUAL = 5, BWOA_COMP_OP_LESS_OR_EQUAL = 6
    } eOAComparisonOpType;

    static bool s_runningBenchmark;   // 是否正在跑 benchmark
    static bool s_doingExit;          // 是否正在退出
};
```

三个枚举分别描述「命令类型」「选项数据类型」「选项比较运算符」。`PY_ENUM_MAP` / `PY_ENUM_CONVERTERS_*` 宏把它们注册到 Python，使脚本可以 `OpenAutomateWrapper.BWOA_CMD_RUN` 这样访问。命令语义：

| 命令 | 含义 |
|------|------|
| `BWOA_CMD_EXIT` | 退出客户端 |
| `BWOA_CMD_RUN` | 运行一个测试用例 |
| `BWOA_CMD_GET_ALL_OPTIONS` | 返回所有可配置选项 |
| `BWOA_CMD_GET_CURRENT_OPTIONS` | 返回当前选项值 |
| `BWOA_CMD_SET_OPTIONS` | 设置选项 |
| `BWOA_CMD_GET_BENCHMARKS` | 列出可用的 benchmark |
| `BWOA_CMD_RUN_BENCHMARK` | 运行某个 benchmark |
| `BWOA_CMD_RESTART` | 重启客户端（特殊值 10000，避开 OpenAutomate 保留区段） |
| `BWOA_CMD_AUTO_DETECT` | 自动检测硬件能力 |

### 5.2 Wrapper 类族

`BWOpenAutomate` 只放枚举，真正的「Python 可见对象」是三个 Wrapper 类，它们包装 OpenAutomate C API 的结构体：

```cpp
class OAVersionStructWrapper : public PyObjectPlus   // 包装 oaVersionStruct
{
    Py_Header( OAVersionStructWrapper, PyObjectPlus )
public:
    void fromOAVersion( oaVersionStruct& tempVersion );
    PY_RW_ATTRIBUTE_DECLARE( major_, major )
    PY_RW_ATTRIBUTE_DECLARE( minor_, minor )
    PY_RW_ATTRIBUTE_DECLARE( custom_, custom )
    PY_RW_ATTRIBUTE_DECLARE( build_, build )
public:
    uint32 major_, minor_, custom_, build_;
};

class OACommandWrapper : public PyObjectPlus          // 包装 oaCommand
{
    Py_Header( OACommandWrapper, PyObjectPlus )
public:
    PyObject * pyGet_benchmark();
    PY_RO_ATTRIBUTE_SET( benchmark )
public:
    oaCommand internalCommand_;   // 直接持有原生结构体
};

class OANamedOptionWrapper : public PyObjectPlus      // 包装 oaNamedOptionStruct
{
    Py_Header( OANamedOptionWrapper, PyObjectPlus )
public:
    OANamedOptionWrapper( oaNamedOptionStruct* option, PyTypePlus * pType = &s_type_ );
    void toNamedOption( oaNamedOptionStruct& option ) const;   // 回写到原生结构体
    PY_DEFERRED_ATTRIBUTE_DECLARE( dataType )
    PY_DEFERRED_ATTRIBUTE_DECLARE( name )
    PY_DEFERRED_ATTRIBUTE_DECLARE( enumValue )
    PY_DEFERRED_ATTRIBUTE_DECLARE( minValueInt )
    PY_DEFERRED_ATTRIBUTE_DECLARE( minValueFloat )
    PY_DEFERRED_ATTRIBUTE_DECLARE( maxValueInt )
    PY_DEFERRED_ATTRIBUTE_DECLARE( maxValueFloat )
    PY_DEFERRED_ATTRIBUTE_DECLARE( numSteps )
    PY_DEFERRED_ATTRIBUTE_DECLARE( parentName )
    PY_DEFERRED_ATTRIBUTE_DECLARE( comparisonOp )
    PY_AUTO_METHOD_DECLARE( RETVOID, setComparison,
        ARG( BWOpenAutomate::eOAOptionDataType, ARG(PyObjectPtr, END ) ) )
public:
    oaNamedOptionStruct internalOption_;
    std::string name_, enumValue_, parentName_, comparisonVal_;
};
```

[open_automate_wrapper.cpp](file:///workspace/src/lib/open_automate/open_automate_wrapper.cpp) 把这些类注册到 Python 模块 `OpenAutomateWrapper`：

```cpp
const std::string BWOpenAutomate::OPEN_AUTOMATE_ARGUMENT = "-openautomate";
const std::string BWOpenAutomate::OPEN_AUTOMATE_TEST_ARGUMENT = "INTERNAL_TEST";

std::string getOpenAutomateArgument()    { return BWOpenAutomate::OPEN_AUTOMATE_ARGUMENT; }
std::string getOpenAutomateTestArgument(){ return BWOpenAutomate::OPEN_AUTOMATE_TEST_ARGUMENT; }
PY_AUTO_MODULE_FUNCTION( RETDATA, getOpenAutomateArgument, END, OpenAutomateWrapper )
PY_AUTO_MODULE_FUNCTION( RETDATA, getOpenAutomateTestArgument, END, OpenAutomateWrapper )

PY_ENUM_MAP( BWOpenAutomate::eOACommandType );
PY_ENUM_CONVERTERS_SCATTERED( BWOpenAutomate::eOACommandType );     // 枚举值不连续 → scattered
PY_ENUM_MAP( BWOpenAutomate::eOAOptionDataType );
PY_ENUM_CONVERTERS_CONTIGUOUS( BWOpenAutomate::eOAOptionDataType );// 枚举值连续 → contiguous
PY_ENUM_MAP( BWOpenAutomate::eOAComparisonOpType );
PY_ENUM_CONVERTERS_CONTIGUOUS( BWOpenAutomate::eOAComparisonOpType );

PY_FACTORY_NAMED( OAVersionStructWrapper, "CreateOAVersionStructWrapper", OpenAutomateWrapper )
```

注意两种枚举转换器的区别：`SCATTERED` 用于枚举值不连续（如 `BWOA_CMD_RESTART=10000`）的情况，内部用 map；`CONTIGUOUS` 用于连续枚举，内部用数组下标，更快。

### 5.3 testOAInit —— 内部测试钩子

[open_automate_tester.hpp](file:///workspace/src/lib/open_automate/open_automate_tester.hpp) 只暴露一个函数：

```cpp
oaBool testOAInit(const oaChar *init_str, oaVersion *version);
```

注释说明它的用途：「在 `testOAInit` 期间覆盖 OpenAutomate 的函数表，用于内部测试 FantasyDemo」。换言之，正常接入时客户端把一组回调函数指针交给 OpenAutomate 主机；而 `testOAInit` 在「自我测试」模式下，把这组函数表替换成内部 stub，让客户端在没有外部主机时也能跑通流程。注释里也坦承了局限：基于 proxy 的测试（跨进程 RPC）尚未实现，因为客户端 `RESTART` 时难以在模块内持久化状态。

### 5.4 典型调用流

```
客户端启动:
  main( "-openautomate" )
    └─ BWOpenAutomate::OPEN_AUTOMATE_ARGUMENT 命中
        └─ bwOARegisterApp()  注册到 OpenAutomate 主机
            └─ 主机发 BWOA_CMD_GET_ALL_OPTIONS
                └─ 客户端返回一组 OANamedOptionWrapper
                   (分辨率/画质/抗锯齿等)
            └─ 主机发 BWOA_CMD_SET_OPTIONS
                └─ OANamedOptionWrapper::toNamedOption 回写
                   → 客户端应用设置
            └─ 主机发 BWOA_CMD_RUN_BENCHMARK
                └─ BWOpenAutomate::s_runningBenchmark = true
                   → 客户端跑预定场景，统计 FPS
                └─ 完成后 s_runningBenchmark = false
            └─ 主机发 BWOA_CMD_EXIT
                └─ BWOpenAutomate::s_doingExit = true → 退出
```

---

## 6. 典型调用流汇总

### 6.1 从 UAL 拖一个 model 到视口

```
用户在 SmartListCtrl 上按下鼠标
  └─ SmartListCtrl::OnBeginDrag
       └─ UalDialog::listStartDrag(index)
            └─ UalManager::startDragCallback_(UalItemInfo*)
                 │  (宿主编辑器接到回调，记录起始 asset)
                 ▼
鼠标移动 (MFC 拖动循环 dragLoop)
  └─ UalDialog::handleDragMouseMove
       └─ UalManager::updateDrag(itemInfo, endDrag=false)
            └─ UalDropManager::test(hwnd, ii)
                 ├─ 命中视口 HWND?  → 高亮边框
                 └─ 不命中 → 取消高亮
鼠标松开
  └─ UalManager::endDragCallback_(UalItemInfo*)
       └─ UalDropManager::end(ii)
            └─ UalDropCallback::execute(ii)
                 │  (宿主: 在视口里创建 ChunkItem)
                 ▼
            └─ UalHistory::prepareItem + addPreparedItem
                 └─ 写入 history XML
```

### 6.2 用 PositionGizmo 平移一个对象

```
用户选中一个 ChunkItem
  └─ GeneralEditor::currentEditors({editor})
       └─ 每个 GenPositionProperty 的视图 elect()
            └─ CurrentPositionProperties::elect()
                 └─ properties_.push_back(&prop)
GizmoManager::draw()
  └─ PositionGizmo::draw(force=false)
       └─ 用 CurrentPositionProperties::averageOrigin() 摆放 gizmo
鼠标移动
  └─ GizmoManager::update(worldRay)
       └─ PositionGizmo::intersects(...) 命中 X 轴
            └─ intersectedGizmo_ = this
用户按下并拖动
  └─ PositionGizmo::click → MatrixProxy::recordState()
  └─ 拖动中: MatrixProxy::setMatrix(平移后矩阵, transient=true)
       └─ CurrentPositionProperties 遍历所有 prop → 各自 pMatrix_->setMatrix
松开
  └─ MatrixProxy::commitState(addUndoBarrier=true)
       └─ UndoRedo 记录一个 Operation
            └─ Ctrl+Z 时 Operation::undo() 回滚
```

### 6.3 生成一个 model 缩略图

```
SmartListCtrl 需要显示 model 文件缩略图
  └─ ListFileProvider::getThumbnail(manager, index, img, w, h, updater)
       └─ ThumbnailManager::create(file, img, w, h, updater)
            └─ needsCreate? 查磁盘 .thumb.bmp → 未命中
            └─ 遍历 s_providers_ 找 ModelThumbProv (isValid=true)
                 └─ 入队 pending_
                      │ (后台 SimpleThread)
                      ▼
                 ModelThumbProv::prepare(file)
                   └─ 加载 SuperModel (不碰 D3D)
                      │ (主线程 tick)
                      ▼
                 ModelThumbProv::render(file, rt)
                   └─ 用 PyModelRenderer 画一帧到 renderRT_
                      └─ PyModelRenderer::renderSelf
                           └─ PyModel::draw (在 s_pSpace_ 内)
                 保存 renderRT_ → 磁盘 .thumb.bmp
                 入队 ready_
                      │ (下一帧)
                      ▼
                 ThumbnailUpdater::thumbManagerUpdate(longText)
                   └─ SmartListCtrl 重绘该行
```

---

## 7. 设计要点小结

1. **UAL 的 provider 抽象**：`ListProvider` / `VFolderProvider` 把数据源（文件系统/XML/聚合）与视图（树/列表）彻底分离，新增一种数据源只需加一个 provider 子类。`ListFileProvider` 的后台线程 + 分块 flush 是大目录不卡 UI 的关键。
2. **ThumbnailManager 的三段式**：`prepare`（后台线程，不碰 D3D）→ `render`（主线程，用 RT）→ 磁盘缓存。既保证线程安全，又复用 D3D 设备。
3. **Gizmo 的视图工厂**：`GeneralProperty` + `View` + `ViewFactory` + `ViewEnroller` 这套宏，让「属性 → 可视化」的连接在静态初始化期自动完成，新增一种属性或视图无需改任何现有代码。`PropertyCollator` 模板优雅地解决了多选聚合。
4. **MatrixProxy 与 UndoRedo 的协作**：`recordState` / `commitState` 把「拖动是一组连续 setMatrix」收敛成「一次 undo 操作」，barrier 机制再支持「批量操作合并」。
5. **Duplo 的 Motor 不持有 owner 引用**：`Motor::pOwner_` 是裸指针，避免 `PyModel ↔ Motor` 循环引用，这是 BigWorld 脚本对象管理的常见模式。
6. **PyModelRenderer 的独立 chunk space**：模型预览不进真实世界，而在 `s_pSpace_` 离屏渲染，与游戏场景完全隔离。
7. **OpenAutomate 的薄包装**：`BWOpenAutomate` 只是把 C API 的结构体包成 `PyObjectPlus`，并用 `SCATTERED`/`CONTIGUOUS` 两种枚举转换器兼顾不连续与连续枚举。`testOAInit` 用函数表覆盖实现无主机的自我测试。

这些模块共同构成了 BigWorld 工具链的「积木箱」：UAL 解决「找资产」，Gizmo 解决「改资产」，Duplo 提供「资产在工具里的可视化与动画」，Ashes 提供「画到屏幕」，OpenAutomate 提供「自动化验证」。理解它们的协作关系，是阅读 WorldEditor / ModelEditor 等具体编辑器代码的前提。

---

## 附：本文引用的核心文件清单

| 模块 | 文件 |
|------|------|
| UAL | [ual_manager.hpp](file:///workspace/src/lib/ual/ual_manager.hpp)、[ual_dialog.hpp](file:///workspace/src/lib/ual/ual_dialog.hpp)、[asset_info.hpp](file:///workspace/src/lib/ual/asset_info.hpp)、[ual_callback.hpp](file:///workspace/src/lib/ual/ual_callback.hpp)、[ual_history.hpp](file:///workspace/src/lib/ual/ual_history.hpp)、[ual_drop_manager.hpp](file:///workspace/src/lib/ual/ual_drop_manager.hpp)、[thumbnail_manager.hpp](file:///workspace/src/lib/ual/thumbnail_manager.hpp)、[smart_list_ctrl.hpp](file:///workspace/src/lib/ual/smart_list_ctrl.hpp)、[list_file_provider.hpp](file:///workspace/src/lib/ual/list_file_provider.hpp)、[vfolder_xml_provider.hpp](file:///workspace/src/lib/ual/vfolder_xml_provider.hpp) |
| Gizmo | [gizmo_manager.hpp](file:///workspace/src/lib/gizmo/gizmo_manager.hpp)、[position_gizmo.hpp](file:///workspace/src/lib/gizmo/position_gizmo.hpp)、[rotation_gizmo.hpp](file:///workspace/src/lib/gizmo/rotation_gizmo.hpp)、[scale_gizmo.hpp](file:///workspace/src/lib/gizmo/scale_gizmo.hpp)、[general_editor.hpp](file:///workspace/src/lib/gizmo/general_editor.hpp)、[general_properties.hpp](file:///workspace/src/lib/gizmo/general_properties.hpp)、[current_general_properties.hpp](file:///workspace/src/lib/gizmo/current_general_properties.hpp)、[value_type.hpp](file:///workspace/src/lib/gizmo/value_type.hpp)、[tool.hpp](file:///workspace/src/lib/gizmo/tool.hpp)、[tool_manager.hpp](file:///workspace/src/lib/gizmo/tool_manager.hpp)、[snap_provider.hpp](file:///workspace/src/lib/gizmo/snap_provider.hpp)、[undoredo.hpp](file:///workspace/src/lib/gizmo/undoredo.hpp) |
| Duplo | [pymodel.hpp](file:///workspace/src/lib/duplo/pymodel.hpp)、[motor.hpp](file:///workspace/src/lib/duplo/motor.hpp)、[homer.hpp](file:///workspace/src/lib/duplo/homer.hpp)、[orbitor.hpp](file:///workspace/src/lib/duplo/orbitor.hpp)、[chunk_attachment.hpp](file:///workspace/src/lib/duplo/chunk_attachment.hpp)、[py_model_renderer.hpp](file:///workspace/src/lib/duplo/py_model_renderer.hpp)、[shadow_caster.hpp](file:///workspace/src/lib/duplo/shadow_caster.hpp) |
| OpenAutomate | [open_automate_wrapper.hpp](file:///workspace/src/lib/open_automate/open_automate_wrapper.hpp)、[open_automate_wrapper.cpp](file:///workspace/src/lib/open_automate/open_automate_wrapper.cpp)、[open_automate_tester.hpp](file:///workspace/src/lib/open_automate/open_automate_tester.hpp) |
