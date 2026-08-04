# BigWorld Engine 2.0.1 客户端框架 - appmgr / input / camera / entity

> 源码位置：`src/lib/appmgr/`、`src/lib/input/`、`src/lib/camera/`、`src/client/`
> 模块定位：本文覆盖客户端「主循环、模块栈、输入、相机、entity/玩家/平滑 filter」这条主线。
> 与 [16-客户端框架-连接层-connection.md](16-客户端框架-连接层-connection.md) 互补：那一份讲客户端如何与服务端通信（`ServerConnection`），这一份讲通信之上客户端自身如何运转（App 主循环、Module 切换、输入分发、相机矩阵供给、客户端 entity 与位置平滑）。
> 关键区分：仓库里有两套 `App`。一个是 `src/lib/appmgr/app.hpp` 中的 `App`（库层、`PresentThread` + `ModuleManager` 的瘦壳），另一个是 `src/client/app.hpp` 中的 `App`（客户端实际主类，单例，持有相机、cursor、console、13 个 `MainLoopTask`）。本文会明确区分二者。

---

## 1. 全景：客户端三层结构

BigWorld 客户端在进程内部分三层：

```
┌──────────────────────────────────────────────────────────────────┐
│  入口层  winmain.cpp / bw_winmain.cpp                            │
│    BWWinMain() : 注册 WNDCLASS → CreateWindow → App::init         │
│    BWWndProc() : Windows 消息泵 → InputDevices / App::handleSetFocus│
└───────────────────────────┬──────────────────────────────────────┘
                            │ 构造并 init
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  客户端主类  App (src/client/app.hpp) —— 单例                      │
│   ├─ updateFrame(active) : 主循环，调用 MainLoopTasks::root()       │
│   ├─ handleKeyEvent/Mouse/Axis : 输入分发（含 keyRouting_ 表）      │
│   ├─ camera()/activeCursor() : 当前相机与输入光标                   │
│   └─ 持有 13 个 MainLoopTask（Device/VOIP/Web/Script/Camera/        │
│              Canvas/World/Flora/Facade/Lens/GUI/Debug/Finale）    │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 由 MainLoopTasks 调度
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  功能库层                                                         │
│  appmgr  : 库层 App（PresentThread）、Module/ModuleManager/        │
│            ScriptedModule、InputManager、Options、DevMenu          │
│  input   : InputDevices(DirectInput+RawInput)、KeyEvent/MouseEvent│
│            /AxisEvent、Joystick、IME、KeyCode                       │
│  camera  : BaseCamera→CursorCamera/FlexiCam/FreeCamera、           │
│            DirectionCursor、ProjectionAccess、CameraControl        │
│  duplo   : Motor/Tracker/Servo/Oscillator/Homer/Bouncer/Propellor │
│            /Orbitor/LinearHomer（模型驱动器，ActionMatcher 即一种）│
│  connection : ServerConnection（见 connection 文档）              │
└──────────────────────────────────────────────────────────────────┘
```

被依赖方（顶层）：

- `cstdmf`：`main_loop_task.hpp`（`MainLoopTask`/`MainLoopTasks`）、`singleton.hpp`、`debug.hpp`、`watcher.hpp`、`profiler.hpp`、`timestamp.hpp`、`diary.hpp`
- `pyscript`：`script.hpp`、`pyobject_plus.hpp`、`personality.hpp`（personality 脚本桥）
- `moo`：`render_context.hpp`（`Moo::rc()` D3D 设备封装）
- `resmgr`：`bwresource.hpp`、`datasection.hpp`、`auto_config.hpp`

---

## 2. 入口层：winmain 与消息泵

### 2.1 BWWinMain 主流程

入口在 [bw_winmain.cpp](file:///workspace/src/client/bw_winmain.cpp)。`BWWinMain` 由 `winmain.cpp` 中的真正 `WinMain` 调用（`winmain.cpp` 负责 `GetCommandLineW`、注册 `WNDCLASS`、调用 `BWWinMain`，本仓库保留二者）。核心流程：

```cpp
int BWWinMain( HINSTANCE hInstance, LPCTSTR lpCmdLine,
               int nCmdShow, LPCTSTR lpClassName, LPCTSTR lpWindowName )
{
    BWResource bwresource;                       // 资源根作用域
    ...
    if (!parseCommandLine( lpCmdLine ))  return FALSE;   // 解析 -c/--config 等
    if ( !Moo::init() )                  return FALSE;   // D3D 初始化

    App app( configFilename, compileTimeString );        // 客户端主类（单例在此构造）
    ...
    hWnd = CreateWindow( lpClassName, windowName.c_str(),
                         parent ? WS_CHILD|... : WS_OVERLAPPEDWINDOW, ... );

    if( !app.init( hInstance, hWnd ) ) { DestroyWindow(hWnd); return FALSE; }

    while( !g_bAppQuit )                              // 标准游戏循环
    {
        if( PeekMessage( &msg, NULL, 0, 0, PM_REMOVE ) )
        {
            if( msg.message == WM_QUIT ) break;
            TranslateMessage( &msg );
            DispatchMessage( &msg );
        }
        else
        {
            if (!app.updateFrame(g_bActive))  ::PostQuitMessage(0);
        }
    }
    Moo::fini();
    return msg.wParam;
}
```

要点：

1. **命令行解析** `parseCommandLine`（同文件）：识别 `-r/--res`、`--options`、`-c/--config`（指定配置文件名，存入 `configFilename`）、`-sa/--script-arg`（透传给 Python `sys.argv`）、`parentWindow=`（嵌入浏览器时父窗口句柄）。还把 `argv[0]` 写入 `Script::g_scriptArgv`，让 Python 脚本能拿到。
2. **`Moo::init()`** 初始化 D3D9 设备封装（`Moo::rc()` 即 `RenderContext`，全局 D3D 访问点）。
3. **`App app(configFilename, compileTimeString)`** 构造客户端单例；`app.init` 完成 13 个 `MainLoopTask` 的注册与初始化。
4. **消息泵**：有消息时 `TranslateMessage/DispatchMessage`（最终落到 `BWWndProc`）；无消息时 `app.updateFrame` 跑一帧。`g_bActive` 在 `WM_SIZE` 中按最小化/隐藏置假，传给 `updateFrame` 决定是 `tick` 还是 `inactiveTick`。

### 2.2 BWWndProc：窗口消息分发

[BWWndProc](file:///workspace/src/client/bw_winmain.cpp) 处理的关键消息：

| 消息 | 处理 |
|------|------|
| `WM_SETCURSOR` | 软件光标时按 `SimpleGUI::mouseCursor().visible()` 决定 `ShowCursor` |
| `WM_MOUSEMOVE` | 软件光标模式下 `SetCursorPosition` |
| `WM_GETMINMAXINFO` | 限制最小窗口 100×100 |
| `WM_PAINT` | 启动阶段（`!g_bAppStarted`）画 "Launching ..." |
| `WM_MOVE` / `WM_SIZE` / `WM_ENTERSIZEMOVE` / `WM_EXITSIZEMOVE` | 通知 `App::moveWindow/resizeWindow` |
| `WM_SYSCOMMAND(SC_CLOSE)` / `WM_CLOSE` | `handleCloseRequest` → `MozillaWebPageManager::s_fini` → `PostQuitMessage` |
| `WM_ACTIVATE`(非嵌入) / `WM_SETFOCUS,WM_KILLFOCUS`(嵌入) | `App::handleSetFocus(true/false)` |
| `WM_IME_*` / `WM_INPUTLANGCHANGE` | Scaleform IME 或默认处理 |
| 末尾兜底 | `InputDevices::handleWindowsMessage(...)` 让输入系统吃消息 |

注意 `WM_SYSKEYUP+VK_MENU` 与 `WM_CONTEXTMENU` 被显式 `return TRUE`，阻止 Alt 弹出系统菜单 / 右键菜单。

### 2.3 两个 App 类的区别（重要）

| 维度 | `appmgr/App`（[app.hpp](file:///workspace/src/lib/appmgr/app.hpp)） | `client/App`（[app.hpp](file:///workspace/src/client/app.hpp)） |
|------|-----|-----|
| 角色 | 库层"瘦壳"，只管 PresentThread + ModuleManager | 客户端真正主类，单例 |
| 主循环 | `updateFrame(bool tick=true)`：直接驱动 `ModuleManager::currentModule()` | `updateFrame(bool active)`：驱动 `MainLoopTasks::root().tick/draw` |
| 输入 | 持有 `InputManager inputHandler_`，`InputDevices::processEvents(inputHandler_)` | 自身实现 `InputHandler`，`handleKeyEvent` 内部按 `keyRouting_` 路由 |
| 设备 | 不直接管 D3D 设备，靠 `Moo::rc()` | 同样靠 `Moo::rc()`，但额外持有相机/cursor/console |
| 用途 | 编辑器/工具可复用的轻量主循环 | 仅客户端可执行文件使用 |

`client/App` 才是实际跑游戏的那个；`appmgr/App` 是更老的、基于 Module 栈的库层抽象（仍在 `appmgr` 内部使用，见下文 Module 一节）。本文后续「App」默认指 `client/App`。

---

## 3. appmgr：模块框架与库层 App

`src/lib/appmgr/` 提供了一套"模块栈"式的应用框架。核心类关系：

```
Factory<Module> ◄── ModuleManager : public Factory<Module>   （单例，模块栈）
                        │ 持有 std::stack<ModulePtr>
                        ▼
                  Module : public InputHandler, public ReferenceCount
                        ▲
                        │
              ┌─────────┴──────────┐
              │                    │
        FrameworkModule        ScriptedModule : public FrameworkModule
       (beginScene/endScene)   （把事件转给 Python 脚本对象）

App(appmgr)  ──holds──►  InputManager : public InputHandler  ──routes──►  currentModule()
   │                                                                   │
   └─holds──► ModuleManager (startup modules from resources/data/modules.xml)
```

### 3.1 Module / FrameworkModule

[module.hpp](file:///workspace/src/lib/appmgr/module.hpp) 定义模块接口：

```cpp
class Module : public InputHandler, public ReferenceCount
{
public:
    virtual bool init( DataSectionPtr pSection );     // 创建时
    virtual void onStart();                           // 进栈
    virtual int  onStop();                             // 出栈，返回 exitCode
    virtual void onPause();                           // 被新模块盖住
    virtual void onResume( int exitCode );            // 上层模块弹出后恢复
    virtual void updateFrame( float dTime ) = 0;     // 每帧逻辑（纯虚）
    virtual void render( float dTime ) = 0;          // 每帧渲染（纯虚）
};

class FrameworkModule : public Module
{
    virtual void updateFrame( float dTime );          // 调 updateState + 包 beginScene/endScene + 画 console
    virtual bool updateState( float dTime );          // 子类实现真正的状态更新
    virtual void render( float dTime );
};
```

`Module` 同时继承 `InputHandler`（见 §5），所以模块本身就是输入处理器。`FrameworkModule` 把"开 beginScene → 调子类 `updateState` → `render` → 画 ConsoleManager → endScene"这套样板封装好，子类只写 `updateState/render`。

### 3.2 ModuleManager：模块栈

[module_manager.hpp](file:///workspace/src/lib/appmgr/module_manager.hpp) / [module_manager.cpp](file:///workspace/src/lib/appmgr/module_manager.cpp)。`ModuleManager` 继承自 `Factory<Module>`，并维护 `std::stack<ModulePtr>`：

- `instance()`：单例（`s_instance_`），`fini()` 销毁。
- `push(ModulePtr)`：先对 `currentModule()` 调 `onPause()`，再 `push`，对新栈顶调 `onStart()`。
- `pop()`：对栈顶调 `onStop()` 得到 `exitCode`，`pop`，对新栈顶调 `onResume(exitCode)`。
- `push(const std::string& identifier)`：用 `Factory::create(identifier)` 按名字造模块（找不到会再做一次去空格重试），造出来后 `decRef`（因为 Factory 的 creator 多加了一次引用）。
- `popAll()`：循环 `currentModule()->onStop()` 再 `pop`，**不调 onResume**（快速优雅退出）。

#### 模块切换示例（登录 → 世界）

模块栈是"覆盖式"的。典型登录流（脚本侧编排，C++ 提供 push/pop 原语）：

```cpp
// 1) 启动时 App::init 从 resources/data/modules.xml 读 <startup> 列表，逐个 push
//    （见 appmgr/app.cpp 的 App::init）
DataSectionPtr spSection = BWResource::instance().openSection(
    "resources/data/modules.xml/startup" );
spSection->readStrings( "module", startupModules );
for (auto & m : startupModules)
    ModuleManager::instance().push( m );

// 2) 登录模块（LoginModule，通常 ScriptedModule）被 push 到栈顶，
//    原来的模块被 onPause
ModuleManager::instance().push( "LoginModule" );

// 3) 登录成功后，登录模块自己 pop，栈恢复，下层模块 onResume(exitCode)
ModuleManager::instance().pop();   // exitCode 传给下层 onResume

// 4) 进游戏后切到世界模块
ModuleManager::instance().push( "WorldModule" );
```

`ScriptedModule::onStop` 会调 Python 脚本对象的 `onStop` 并把返回值作为 `exitCode`，再交给下层的 `onResume`。

### 3.3 Factory / Creator / IMPLEMENT_CREATOR

[factory.hpp](file:///workspace/src/lib/appmgr/factory.hpp) 是一个泛型工厂模板：

```cpp
template <class TYPE>
class Creator { virtual TYPE * create( DataSectionPtr pSection ) = 0; };

template <class TYPE>
class Factory {
    void add( const char * name, Creator<TYPE> & creator );
    TYPE * create( const char * name, DataSectionPtr pSection = NULL ) const;
    StringHashMap< Creator<TYPE> *> creators_;   // 名字→creator
};
```

模块通过宏 `IMPLEMENT_CREATOR(NAME, TYPE)` 自注册：

```cpp
// scripted_module.cpp
typedef ModuleManager ModuleFactory;            // 注意：模块工厂就是 ModuleManager 本身
IMPLEMENT_CREATOR(ScriptedModule, Module);      // 生成 s_ScriptedModuleCreator 静态对象
```

`IMPLEMENT_CREATOR` 展开后会生成一个 `NAME##Creator : public StandardCreator<TYPE>`，在构造时把自己注册进 `TYPE##Factory::instance()`（对 Module 来说就是 `ModuleFactory`/`ModuleManager`）。这样 `ModuleManager::push("ScriptedModule")` 就能造出 `ScriptedModule`。

### 3.4 ScriptedModule：Python 驱动的模块

[scripted_module.hpp](file:///workspace/src/lib/appmgr/scripted_module.hpp) / [.cpp](file:///workspace/src/lib/appmgr/scripted_module.cpp)。`ScriptedModule` 持有一个 `PyObject * pScriptObject_`，把所有 `Module` 虚函数转调到 Python：

`init(DataSectionPtr)` 读 `<script><module>Modules.py</module><class>MyClass</class></script>`，`PyImport_ImportModule` + `PyObject_CallMethod(className, "")` 造出脚本对象。之后：

- `onStart` → `Script::call(pScriptObject_.onStart)`
- `onStop` → `Script::ask(pScriptObject_.onStop)`，返回值经 `Script::setAnswer` 取得 int exitCode
- `onPause/onResume` → 对应 `onPause`/`onResume(exitCode)`
- `updateState(dTime)` → `ask("updateState",(f,)dTime)`，若脚本返回 False（未处理）则回退到 `SimpleGUI::instance().update(dTime)`
- `render(dTime)` → `ask("render",(f,)dTime)`，未处理则 `Clear` + `SimpleGUI::draw()`
- `handleKeyEvent/handleMouseEvent` → `ask("onKeyEvent/onMouseEvent")`，未处理回退 `SimpleGUI::handleKeyEvent/handleMouseEvent`

`ScriptedModule::onStart/onStop` 还会切 `SimpleGUI::mouseCursor().visible()`，让鼠标光标在模块进出时自动显隐。

### 3.5 ScriptInstance

[script_instance.hpp](file:///workspace/src/lib/appmgr/script_instance.hpp) 是 `PyInstancePlus` 的子类，给"实例脚本对象"提供统一的 `init(DataSectionPtr, moduleName, defaultTypeName)` 模板方法，按 `<script>/<className>` 加载并实例化 Python 类。`ScriptedModule` 的脚本加载逻辑与它同源。

### 3.6 InputManager（appmgr 层）

[input_manager.hpp](file:///workspace/src/lib/appmgr/input_manager.hpp) / [.cpp](file:///workspace/src/lib/appmgr/input_manager.cpp)。库层的 `InputManager : public InputHandler` 是 appmgr 框架里的输入路由器，路由顺序：

```cpp
bool InputManager::handleKeyEvent( const KeyEvent & event )
{
    bool handled = ConsoleManager::instance().handleKeyEvent( event );   // 1) console 优先
    ModulePtr module = ModuleManager::instance().currentModule();
    if ( module && !handled )
        handled = module->handleKeyEvent( event );                       // 2) 当前模块
    if ( !handled )
        handled = ApplicationInput::handleKeyEvent( event );             // 3) 全局快捷键
    return handled;
}
```

鼠标事件同理（但 console 不参与）。**注意**：这是 `appmgr/App` 框架用的路径；`client/App` 不用 `InputManager`，而是自己实现了一套更复杂的 `handleKeyEvent`（见 §6.3）。

### 3.7 ApplicationInput：全局快捷键

[application_input.hpp](file:///workspace/src/lib/appmgr/application_input.hpp)。处理跨模块的系统级按键：

- `ALT+ENTER`：切换窗口/全屏模式（可被 `disableModeSwitch()` 禁用）
- `SHIFT+F4`：退出应用

### 3.8 Options：游戏选项

[options.hpp](file:///workspace/src/lib/appmgr/options.hpp)。`Options` 是单例，把一个 XML `DataResource` 暴露成"按路径取值"的键值存储，支持 `String/Int/Bool/Float/Vector2/3/4/Matrix34`。同时把所有这些以 `BigWorld.setOption/getOption` 暴露给 Python（`PY_MODULE_STATIC_METHOD_DECLARE`）。`App::init` 用它读 `app/maxTimeDelta`、`app/timeScale`、`consoles/darkenBackground`、`checkFileCase` 等。

### 3.9 DevMenu / ClosedCaptions / Commentary

- [dev_menu.hpp](file:///workspace/src/lib/appmgr/dev_menu.hpp)：`DevMenu : public FrameworkModule`，一个开发菜单模块（按钮 + 文字菜单项 + 水印），用作开发期模块切换器。
- [commentary.hpp](file:///workspace/src/lib/appmgr/commentary.hpp)：`Commentary` 单例，是引擎运行时调试信息的中枢，分 6 个 `LevelId`（COMMENT/CRITICAL/ERROR_LEVEL/WARNING/HACK/SCRIPT_ERROR），支持注册 `View` 回调（观察者模式）。
- [closed_captions.hpp](file:///workspace/src/lib/appmgr/closed_captions.hpp)：`ClosedCaptions : public Commentary::View, public PyObjectPlus`，把 `Commentary` 的消息以字幕形式画到屏幕（无声音卡或失聪辅助）。内部维护 `Captions` 环形缓冲 + `PendingMessages`，`parseEventQueue` 在子线程 `update` 时拉取（用 `SimpleMutex` 保护 pending）。

### 3.10 库层 App 与 PresentThread

[app.hpp](file:///workspace/src/lib/appmgr/app.hpp) / [app.cpp](file:///workspace/src/lib/appmgr/app.cpp) 中的 `App` 主循环：

```cpp
void App::updateFrame( bool tick )
{
    Profiler::instance().tick();
    float dTime = this->calculateFrameTime();
    if (!tick) dTime = 0.f;

    if ( InputDevices::hasFocus() )
    {
        if ( bInputFocusLastFrame_ )  InputDevices::processEvents( this->inputHandler_ );
        else { InputDevices::consumeInput(); bInputFocusLastFrame_ = true; }
    }
    ...
    ModulePtr pModule = ModuleManager::instance().currentModule();
    if (pModule && Moo::rc().checkDevice())
    {
        pModule->updateFrame( dTime );                    // 内含 beginScene/render/endScene
        HRESULT hr = Moo::rc().beginScene();
        if (SUCCEEDED(hr)) { ...; pModule->render(dTime); ...; Moo::rc().endScene(); }
        presentThread_.present();                         // 异步 present
    }
}
```

`calculateFrameTime`：用 `timestamp()` 算 dt，超过 `maxTimeDelta_`（默认 0.5s）截断，乘 `timeScale_`，`paused_` 时 dt=0。`pause()` 会在取消暂停时修正 `lastTime_`，避免"跳跃"。

**PresentThread**（内嵌结构体，同时继承 `SimpleSemaphore` 和 `SimpleThread`）：把 `D3D Present` 放到独立线程，主线程 `onPresent()` 里 `while(presenting()) Sleep(0)` 等待。Present 线程还会发一个 `IDirect3DQuery9(EVENT)` 强制 CPU/GPU 同步，避免鼠标抖动（除非 Umbra 已开启同步）。`fps()` 由 present 线程统计。

`App::init` 还会：`BWResource::addPath` 当前目录、`ConsoleManager::createInstance`、`AutoConfig::configureAllFrom("resources.xml")`、`SimpleGUI::init`、`ModuleManager::init(modules.xml)`、按 `<startup>` 列表 push 启动模块、初始化 `EnviroMinderSettings/FloraSettings`。

---

## 4. client/App：客户端真正的主类

### 4.1 MainLoopTask 机制

[main_loop_task.hpp](file:///workspace/src/lib/cstdmf/main_loop_task.hpp) 定义任务接口与组：

```cpp
class MainLoopTask {
    virtual bool init();  virtual void fini();
    virtual void tick( float dTime );  virtual void draw();
    virtual void inactiveTick( float dTime );   // 最小化时调
    bool enableDraw;
};

class MainLoopTasks : public MainLoopTask {
    void add( MainLoopTask * pTask, const char * name, ... /*rules, NULL*/ );
    // rules: ">TaskA" 表示必须在 TaskA 之后，"<TaskB" 表示必须在 TaskB 之前
    static MainLoopTasks & root();
};
```

`MainLoopTasks` 内部用 `StringMap<MainLoopTask*>` + `OrderList` 拓扑排序，按 `>`/`<` 规则排定 tick/draw 顺序。

### 4.2 13 个 MainLoopTask 的注册与依赖

[App::init](file:///workspace/src/client/app.cpp) 中显式注册（`>X` 表示依赖 X，即排在 X 之后）：

```cpp
MainLoopTasks::root().add( NULL, "Device",  NULL );
MainLoopTasks::root().add( NULL, "VOIP",    ">Device",  NULL );
MainLoopTasks::root().add( NULL, "Web",     ">VOIP",    NULL );
MainLoopTasks::root().add( NULL, "Script",  ">Web",     NULL );
MainLoopTasks::root().add( NULL, "Camera",  ">Script",  NULL );
MainLoopTasks::root().add( NULL, "Canvas",  ">Camera",  NULL );
MainLoopTasks::root().add( NULL, "World",   ">Canvas",  NULL );
MainLoopTasks::root().add( NULL, "Flora",   ">World",   NULL );
MainLoopTasks::root().add( NULL, "Facade",  ">Flora",   NULL );
MainLoopTasks::root().add( NULL, "Lens",    ">Facade",  NULL );
MainLoopTasks::root().add( NULL, "GUI",      ">Lens",    NULL );
MainLoopTasks::root().add( NULL, "Debug",    ">GUI",     NULL );
MainLoopTasks::root().add( NULL, "Finale",   ">Debug",   NULL );

bool ok = MainLoopTasks::root().init();   // 一次性 init 所有 task
```

每个 task 名字对应一个 `*App` 类（都是 `MainLoopTask` 的派生，文件内 `static XxxApp instance;` 单例 + 一个 `extern int XxxApp_token` 用于链接器保留）。它们都是 `client/App` 的"阶段"（phase），而非继承关系：

| Task | 类（均 `: public MainLoopTask`） | 职责 |
|------|------|------|
| Device | [DeviceApp](file:///workspace/src/client/device_app.hpp) | 设备/进度条/偏好保存/消息时间前缀 |
| VOIP | [VOIPApp](file:///workspace/src/client/voip_app.hpp) | 语音（bwvoip 封装） |
| Web | [WebApp](file:///workspace/src/client/web_app.hpp) | Mozilla/Scaleform Web 集成 |
| Script | [ScriptApp](file:///workspace/src/client/script_app.hpp) | Python 脚本 tick |
| Camera | [CameraApp](file:///workspace/src/client/camera_app.hpp) | 相机更新/光标/输入 |
| Canvas | [CanvasApp](file:///workspace/src/client/canvas_app.hpp) | 画布后处理、LOD 控制器、闪光/扭曲 |
| World | [WorldApp](file:///workspace/src/client/world_app.hpp) | 场景/地形/碰撞 tick & draw |
| Flora | （在 romp） | 植被 |
| Facade | [FacadeApp](file:///workspace/src/client/facade_app.hpp) | 外观层 |
| Lens | [LensApp](file:///workspace/src/client/lens_app.hpp) | 镜头特效 |
| GUI | [GUIApp](file:///workspace/src/client/gui_app.hpp) | GUI |
| Debug | [DebugApp](file:///workspace/src/client/debug_app.hpp) | 调试覆盖 |
| Finale | [FinaleApp](file:///workspace/src/client/finale_app.hpp) | 最终合成（继承 `MainLoopTask, InputHandler`） |

注意 `ProfilerApp`（[profiler_app.hpp](file:///workspace/src/client/profiler_app.hpp)）也存在，但未在 13 个 root task 之列（其 token 仍被引用以保留链接）。

### 4.3 主循环 updateFrame

[App::updateFrame](file:///workspace/src/client/app.cpp)（精简）：

```cpp
bool App::updateFrame(bool active)
{
    Profiler::instance().tick();
    DiaryScribe deAll(Diary::instance(), "Frame" );
    this->calculateFrameTime();
    MouseCursor::updateMouseClipping();

    if (dTime_ > 0.f)
    {
        if (active) {
            { PROFILER_SCOPED(App_Tick);  g_watchTick.start();
              DiaryEntryPtr deTick = Diary::instance().add( "Tick" );
              MainLoopTasks::root().tick( dTime_ );          // 按依赖顺序 tick 全部 task
              deTick->stop();  g_watchTick.stop(); }

            if (Moo::rc().checkDevice()) {
                PROFILER_SCOPED(App_Draw);  g_watchOutput.start();
                DiaryEntryPtr deDraw = Diary::instance().add( "Draw" );
                MainLoopTasks::root().draw();                // 按依赖顺序 draw 全部 task
                deDraw->stop();  g_watchOutput.stop(); }
        } else {
            MainLoopTasks::root().inactiveTick( dTime_ );    // 最小化时
        }

        // 帧率限制：minFrameTime_ 控制最大帧率，sleepTime_ 让出 CPU
        if (minFrameTime_ > 0 && frameEndTime < lastFrameEndTime_ + minFrameTime_)
            sleepTime = ...;
        if (sleepTime) { PROFILER_SCOPED(Sys_Sleep); ::Sleep( sleepTime ); }
        ...
    }
    return !g_bAppQuit;
}
```

`Diary` 是 BigWorld 的"日记"系统，按帧记录 tick/draw 区间，便于事后分析性能。`DogWatch`（如 `g_watchTick`）是细粒度计时器，聚合到 `DogWatchManager`。

### 4.4 AppConfig

[app_config.hpp](file:///workspace/src/client/app_config.hpp)：`AppConfig` 单例，`init(DataSectionPtr)` 读配置根，`pRoot()` 暴露给全局。`bw_winmain` 用它读 `appTitle`（窗口标题，拼接 `BUILD_CONFIGURATION`）。

---

## 5. input 库：输入系统

源码位置 `src/lib/input/`。

### 5.1 事件类型与 InputHandler

[input.hpp](file:///workspace/src/lib/input/input.hpp) 定义四类事件：

- **`KeyEvent`**：按键。`key()`（`KeyCode::Key`）、`isKeyDown/isKeyUp`、`repeatCount()`（0=刚按下，>0=自动重复，-1=keyup）、`utf16Char()/utf8Char()`、`modifiers()`（`MODIFIER_SHIFT/CTRL/ALT`）、`cursorPosition()`（clip 空间）。静态工厂 `KeyEvent::make(...)`。
- **`MouseEvent`**：`dx()/dy()/dz()`（滚轮）+ `cursorPosition()`。
- **`AxisEvent`**：摇杆轴，`AXIS_LX/LY/RX/RY`，`value()`、`dTime()`。
- **`InputEvent`**：上述三者的 union，`type_` 区分，带全局递增 `seqId_`。

```cpp
class InputHandler {
    virtual bool handleKeyEvent( const KeyEvent & event );
    virtual bool handleInputLangChangeEvent();
    virtual bool handleIMEEvent( const IMEEvent & event );
    virtual bool handleMouseEvent( const MouseEvent & event );
    virtual bool handleAxisEvent( const AxisEvent & event );
};
```

`KeyStates` 维护 `int32 keyStates_[NUM_KEYS]`：负值=未按，0=刚按，正值=重复次数。`Joystick`（同文件）封装 `IDirectInputDevice8`，`processEvents` 产生 key/axis 事件。

### 5.2 InputDevices：DirectInput + Raw Input

`InputDevices : public Singleton<InputDevices>` 是输入中枢。`privateInit`（[input.cpp](file:///workspace/src/lib/input/input.cpp)）：

```cpp
bool InputDevices::privateInit( void * _hInst, void * _hWnd )
{
    // 1) DirectInput8Create —— 仅用于摇杆
    hr = DirectInput8Create( hInst, DIRECTINPUT_VERSION,
                             IID_IDirectInput8, (LPVOID*)&pDirectInput_, NULL );
    if (SUCCEEDED(hr))  this->joystick_.init( pDirectInput_, _hWnd );

    // 2) 鼠标 + 键盘：用 Win32 Raw Input（不是 DirectInput）
    RAWINPUTDEVICE rawDevice;
    rawDevice.usUsagePage = HID_USAGE_PAGE_GENERIC;
    rawDevice.usUsage     = HID_USAGE_GENERIC_KEYBOARD;   // 先注册键盘
    rawDevice.hwndTarget  = hWnd_;
    RegisterRawInputDevices(&rawDevice, 1, sizeof(rawDevice));
    ... HID_USAGE_GENERIC_MOUSE ...                         // 再注册鼠标

    // 3) IME
    if ( !IME::instance().initIME( hWnd_ ) )  WARNING_MSG("Failed to initialise IME.");
    return true;
}
```

> 说明：任务描述提到「`EventPoller`（DX8/SDL/Win32 三种实现）」。在本 2.0.1 仓库中，`EventPoller` 类存在于 `src/lib/network/`（Mercury 网络库的 socket 事件轮询器），并非输入库。输入层这里直接用 DirectInput8（摇杆）+ Win32 Raw Input（键鼠）+ `IME`，没有"三实现 EventPoller"抽象。`xbinput.cpp` 是 Xbox 手柄输入。后续版本才抽象出 `EventPoller` 接口。

`InputDevices` 的关键静态接口：

- `processEvents(InputHandler&)`：把事件队列 `eventQueue_` 里的事件依次喂给 handler。
- `handleWindowsMessage(hWnd,msg,wParam,lParam,result)`：在 `BWWndProc` 末尾被调用，把 Win32 消息转成 KeyEvent/MouseEvent 推入队列（也处理 IME 消息）。
- `modifiers()/isKeyDown()/isAltDown()...`：当前修饰键状态。
- `setFocus(state, handler)/hasFocus()`：窗口焦点。失焦时 `consumeInput()` 清空队列，避免误触发。
- `consumeInput()`：丢弃到某个 `seqId` 为止的事件（用于失焦/设备丢失）。
- `pushIMECharEvent(wchar_t*)`：IME 组字完成时产生字符事件。

设备丢失用 `lostDataFlags`（`KEY_DATA_LOST/MOUSE_DATA_LOST/JOY_DATA_LOST`）位掩码标记，`handleLostData` 通知 handler。

### 5.3 IME 输入法

[ime.hpp](file:///workspace/src/lib/input/ime.hpp)：`IME : public InitSingleton<IME>`。支持 5 种语言（`LANGUAGE_NON_IME/CHS/CHT/JAPANESE/KOREAN`）和 3 种状态（`STATE_OFF/ON/ENGLISH`）。维护 composition string、候选词列表、reading 串，`processEvents(InputHandler&)` 把 `IMEEvent` 喂给 handler。`handleWindowsMessage` 处理 `WM_IME_*`。`allowEnable(bool)` 由 `client/App` 在 debug 键按下时禁用 IME。

`ImeUi.cpp/ImeUi.h` 是 IME 的渲染 UI（候选词窗口绘制）。`py_input.cpp` 把输入系统暴露给 Python（`BigWorld` 模块）。

### 5.4 KeyCode 与 vk_map

[key_code.hpp](file:///workspace/src/lib/input/key_code.hpp) 定义 `KeyCode::Key` 枚举（含 `KEY_DEBUG`、`KEY_NONE`、`NUM_KEYS` 等）。`vk_map.cpp` 把 Windows VK 码映射到 `KeyCode::Key`。

---

## 6. 输入事件流：从硬件到脚本

`client/App` 的输入路径比 `appmgr/InputManager` 更复杂，核心是 `keyRouting_` 表保证 keydown/keyup 配对一致。

### 6.1 事件来源

```
硬件 → Win32 Raw Input / DirectInput(摇杆)
     → BWWndProc → InputDevices::handleWindowsMessage
     → 推入 InputDevices::eventQueue_
     → App::updateFrame 每帧 → (在 CameraApp 或 App 内) InputDevices::processEvents(handler)
```

`App` 自身是 `InputHandler`，被传给 `processEvents`，于是 `handleKeyEvent/handleMouseEvent/handleAxisEvent` 被回调。

### 6.2 keyRouting_ 表

[app.hpp](file:///workspace/src/client/app.hpp) 中：

```cpp
enum eEventDestination {
    EVENT_SINK_NONE = 0, EVENT_SINK_CONSOLE, EVENT_SINK_SCRIPT,
    EVENT_SINK_APP, EVENT_SINK_DEBUG, EVENT_SINK_PERSONALITY
};
eEventDestination keyRouting_[KeyCode::NUM_KEYS];   // 每个键记录它的 keydown 去了哪
```

作用：keydown 被某个 sink 处理后，记录到 `keyRouting_[key]`；对应的 keyup 必须也路由给同一个 sink（否则会出现"按下被 A 处理、抬起被 B 处理"的错配）。`handleKeyEvent` 开头：若是 keyup，先取 `keySunk = keyRouting_[key]` 并清空，最后用它强制把 keyup 也送给同一个 sink。

### 6.3 handleKeyEvent 分发顺序

[App::handleKeyEvent](file:///workspace/src/client/app.cpp)（精简，顺序很重要）：

```cpp
bool App::handleKeyEvent(const KeyEvent & event)
{
    HandleKeyEventHolder holder(*this);   // handleKeyEventDepth_++，防重入

    // 0) Debug 组合键：若 debugKeyEnable_ 且本次按键 + 其它键都按下，合成 KEY_DEBUG 事件
    if ( debugKeyEnable_ && ... && checkDebugKeysState( event.key() ) ) {
        KeyEvent thisEvent = KeyEvent::make( KEY_DEBUG, event.isKeyDown(), ... );
        InputDevices::keyStates().setKeyStateNoRepeat( KEY_DEBUG, event.isKeyDown() );
        this->handleKeyEvent( thisEvent );                  // 递归处理 debug 键
    }

    bool handled = false;
    eEventDestination keySunk = EVENT_SINK_NONE;
    if (event.isKeyUp()) {                                  // keyup：取走之前的 sink 记录
        if (keyRouting_[event.key()] == EVENT_SINK_NONE) return true;  // 无配对，丢弃
        keySunk = keyRouting_[event.key()];
        keyRouting_[event.key()] = EVENT_SINK_NONE;
    }

    // 1) Debug 键被按下时，所有键先给 debug
    if (!handled && event.isKeyDown() && isKeyDown(KEY_DEBUG)) {
        handled = this->handleDebugKeyDown( event );
        if (handled) keyRouting_[event.key()] = EVENT_SINK_DEBUG;
    }
    if (keySunk == EVENT_SINK_DEBUG) handled = true;        // 配对 keyup

    // 2) Console（活动控制台 + ConsoleManager）
    if (!handled) {
        handled = ConsoleManager::instance().handleKeyEvent( event );
        if (handled && event.isKeyDown()) keyRouting_[event.key()] = EVENT_SINK_CONSOLE;
    }
    if (keySunk == EVENT_SINK_CONSOLE) handled = true;

    // 3) 脚本全局钩子（BigWorldClientScript::sinkKeyboardEvent）
    if (!handled) handled = BigWorldClientScript::sinkKeyboardEvent(ievent);

    // 3.5) Scaleform（若启用）
    #if SCALEFORM_SUPPORT
        if (!handled) handled = Scaleform::Manager::instance().onKeyEvent( event );
    #endif

    // 4) Personality 脚本（Personality::instance().handleKeyEvent）
    if (!handled) {
        PyObject * ret = Script::ask(
            PyObject_GetAttrString( Personality::instance(), "handleKeyEvent" ), ...);
        Script::setAnswer( ret, handled, ...);
        if (handled && event.isKeyDown()) keyRouting_[event.key()] = EVENT_SINK_PERSONALITY;
    }
    if (keySunk == EVENT_SINK_PERSONALITY) handled = true;

    // 5) App 自己（仅 keydown，handleKeyDown 处理截图/退出等系统键）
    if (!handled && event.isKeyDown()) {
        handled = this->handleKeyDown( event );
        if (handled) keyRouting_[event.key()] = EVENT_SINK_APP;
    }
    if (keySunk == EVENT_SINK_APP) handled = true;

    // 6) 最后兜底：Player 实体的 handleKeyEvent，并强制 sink 所有按键
    if (!handled) {
        if ( Player::entity() != NULL)
            Script::call( PyObject_GetAttrString( Player::entity(), "handleKeyEvent" ), ...);
        if (event.isKeyDown())      { keyRouting_[event.key()] = EVENT_SINK_SCRIPT; handled = true; }
        else if (event.isKeyUp())   { keyRouting_[event.key()] = EVENT_SINK_NONE;   handled = true; }
    }

    return handled;
}
```

完整优先级链：**Debug 键合成 → Console → 脚本全局钩子 → Scaleform → Personality → App(系统键) → Player 脚本（兜底 sink）**。

`handleMouseEvent/handleAxisEvent`（同文件 ~2750 行起）类似但更简单，主要把事件交给当前 cursor、相机、player 等。

### 6.4 InputCursor / DirectionCursor

[input_cursor.hpp](file:///workspace/src/lib/input/input_cursor.hpp) 抽象"光标"（相机/瞄准方向）。[direction_cursor.hpp](file:///workspace/src/lib/camera/direction_cursor.hpp)：

`DirectionCursor : public InputCursor` 是客户端默认光标，单例（`App::init` 末尾若脚本未设置则 `activeCursor_ = &DirectionCursor::instance()`）。它持 pitch/yaw/roll、鼠标灵敏度、mouseHVBias（水平/垂直偏置）、maxPitch/minPitch、lookSpring（视线回中）。通过 `SpeedProvider` 决定鼠标移动到方向变化的映射（线性 `SimpleSpeedProvider` 或客户端 `ClientSpeedProvider`，见 §7.4）。

---

## 7. camera 库：相机系统抽象

源码 `src/lib/camera/`。类层次：

```
PyObjectPlus, InputHandler, Aligned
        │
   BaseCamera  （view_/invView_ 矩阵、MatrixProvider 包装、spaceID）
        ▲
   ┌────┴─────────┐──────────┐
   │              │          │
CursorCamera  FlexiCam   FreeCamera
```

### 7.1 BaseCamera

[base_camera.hpp](file:///workspace/src/lib/camera/base_camera.hpp)。`BaseCamera` 是 Python 化的相机基类（`Py_Header`），同时是 `InputHandler`：

```cpp
class BaseCamera : public PyObjectPlus, public InputHandler, public Aligned {
    virtual void set( const Matrix & viewMatrix ) = 0;   // 直接设矩阵
    virtual void update( float dTime ) = 0;               // 更新 view_/invView_
    void render();                                        // 把矩阵交给 Moo::rc()

    const Matrix & view() const;
    const Matrix & invView() const;
    MatrixProviderPtr viewMatrixProvider() const;          // 暴露给 MatrixProvider
    MatrixProviderPtr invViewMatrixProvider() const;

    PY_RO_ATTRIBUTE_DECLARE( invView_.applyToOrigin(), position )      // 相机位置
    PY_RO_ATTRIBUTE_DECLARE( invView_.applyToUnitAxisVector(2), direction )  // 朝向
    ...
    SpaceID spaceID_;                                     // 相机所在空间
    static bool sceneCheck( ... );                        // 与场景几何的检测
};
```

`set(ConstSmartPointer<MatrixProvider>)` 是用 MatrixProvider 设源（延迟求值），`set(Matrix)` 是直接覆盖。`render()` 把 `view_` 写进 `Moo::RenderContext`，渲染管线据此变换。

### 7.2 三种相机

#### CursorCamera（[cursor_camera.hpp](file:///workspace/src/lib/camera/cursor_camera.hpp)）

"漫游相机"，跟踪 target（Entity/Node/MatrixProvider），按 pivot（枢轴偏移）+ maxDistanceFromPivot 摆位，再被场景几何裁剪保证视线清晰。关键属性：

- `pivotPosition` / `maxDistanceFromPivot` / `minDistanceFromPivot` / `minDistanceFromTerrain`
- `movementHalfLife` / `turningHalfLife`：位置和朝向的半衰期平滑（指数趋近，越近越慢）
- `limitVelocity` / `maxVelocity`：限速
- `target` / `source`：两个 `MatrixProvider`，target 是看的目标，source 决定朝向
- `firstPersonPerspective` / `reverseView`
- `shake(duration, amount)`：相机震动
- `inaccuracyProvider`：抖动噪声供给（`Vector4ProviderPtr`）

多 target 优先级：Entity > Node > matrix；`targetPlayer()` 强制查 `Player::entity()`。`update(dt)` 内做碰撞裁剪、半衰期插值、震动叠加。

#### FlexiCam（[flexicam.hpp](file:///workspace/src/lib/camera/flexicam.hpp)）

"弹性跟随相机"，有 preferredPos（期望位置）和 actualPos（实际位置），通过 `positionAcceleration` / `trackingAcceleration` 做加速度平滑。`viewOffset` 偏移视线，`uprightDir` 约束竖直。target 优先级同 CursorCam。适合第三人称跟随。

#### FreeCamera（[free_camera.hpp](file:///workspace/src/lib/camera/free_camera.hpp)）

"自由相机"，无视实体、不做物理/几何检测，可被鼠标 + 方向键随意移动（消耗所有鼠标事件 + 方向键事件）。调试/编辑用。有 `fixed`（是否锁定）和 `invViewProvider`。

### 7.3 MatrixProvider：相机矩阵供给

`MatrixProvider`（在 `pyscript/script_math.hpp`，被 camera/entity 共用）是一个"延迟矩阵"：`virtual void matrix(Matrix & m) const` 按需计算。相机通过 `BaseCamera::viewMatrixProvider()` 把自己的 `view_` 包成 `MatrixProvider` 给外部（如 entity 的 Tracker）。反方向也成立：`BaseCamera::set(MatrixProviderPtr)` 让相机跟随一个动态矩阵源。

客户端在 [matrix_providers.hpp](file:///workspace/src/client/matrix_providers.hpp) 提供几种方向供给：

| 类 | 作用 |
|----|------|
| `EntityDirProvider` | 取某 Entity 的朝向（可选 pitch/yaw index）作为矩阵 |
| `DiffDirProvider` | 从 source 矩阵指向 target 矩阵的方向 |
| `ScanDirProvider` | 绕 Y 轴振荡扫描方向（amplitude/period/offset，可回调触限） |

这些常和 `duplo/` 里的 Tracker 配合，让模型节点（如头）转向某方向/某实体。

### 7.4 ClientCamera / DirectionCursor / 速度供给

[client_camera.hpp](file:///workspace/src/client/client_camera.hpp)。`ClientCamera` 是单例，持有当前 `BaseCameraPtr` + `ProjectionAccess*` + `DirectionCursor*` + `ClientSpeedProvider&`。`update(dTime)` 每帧驱动它们。

围绕它还有几个 Python 暴露对象：

- **`AutoAim`**（`PyObjectPlus`）：自动瞄准参数（friction、forwardAdhesion、strafeAdhesion、turnAdhesion、各种角度/距离 falloff）。
- **`Targeting`**：目标选择（selectionAngle/deselectionAngle/selectionDistance、`hasAnAutoAimTarget(...)` 计算当前自动瞄准目标）。
- **`CameraSpeed`**：相机转动速度参数（lookAxisMin/Max、accelerateStart/Rate/End、cameraYawSpeed、cameraPitchSpeed、turningHalfLife、`lookTable` 非线性映射表）。
- **`ClientSpeedProvider : public SpeedProvider`**：综合 `Targeting` + `CameraSpeed`，实现非线性、加速、靠近目标有摩擦的方向控制；`value(AxisEvent)` 把鼠标轴事件转成速度，`adjustDirection(dTime,pitch,yaw,roll)` 每帧调整 `DirectionCursor` 朝向。

### 7.5 ProjectionAccess / CameraControl

- [projection_access.hpp](file:///workspace/src/lib/camera/projection_access.hpp)：`ProjectionAccess` 暴露投影矩阵，可调 `nearPlane/farPlane/fov`，`rampFov(val,time)` 平滑过渡 FOV。`App::projAccess()` 返回它。
- [camera_control.hpp](file:///workspace/src/lib/camera/camera_control.hpp)：`CameraControl` 静态类，封装 FreeCamera 的控制量（yawVelocity/pitchVelocity/velocity/nudge/axisNudge/cameraMass/strafeRate）和 `calculateDeltaMatrix(dTime)`，是 FreeCamera 行为的参数中心。

### 7.6 相机数据流

```
AxisEvent(鼠标) ──► ClientSpeedProvider::value()  ─► DirectionCursor::handleAxisEvent
                                                          │
                                                          ▼ pitch_/yaw_/roll_
                              DirectionCursor::tick(dTime)│ (lookSpring 回中)
                                                          ▼
                          MatrixProvider(provider()) ──► CursorCamera::source
                                                          │
BaseCamera::update(dTime) ◄──────────────────────────────┘
        │ 计算 view_/invView_，碰撞裁剪、半衰期平滑、震动
        ▼
BaseCamera::render() ──► Moo::rc().view(view_)  / ProjectionAccess::update ─► Moo::rc().projection(...)
        │
        ▼ 渲染管线（WorldApp.draw 等）使用 rc 的矩阵
```

### 7.7 duplo 里的 Motor 体系（与相机/模型联动）

任务提到的 `Motor/Servo/Oscillator/Tracker/Homer/Bouncer/Gauche/Pots` 实际位于 `src/lib/duplo/`（不在 `camera/`），是模型的"驱动器"。基类 [motor.hpp](file:///workspace/src/lib/duplo/motor.hpp)：

```cpp
class Motor : public PyObjectPlus {
    virtual void attach( PyModel * pOwner );   // 挂到模型
    virtual void rev( float dTime ) = 0;        // 每帧驱动模型位置/朝向/动画
};
```

- **Tracker**（[tracker.hpp](file:///workspace/src/lib/duplo/tracker.hpp)）：让模型节点（如头、眼睛）跟随某方向（`MatrixProvider`），有 yaw/pitch 限位、角速度、半衰期。`BaseNodeInfo/TrackerNodeInfo` 描述每个受影响节点。
- **Servo / Oscillator / Homer / LinearHomer / Bouncer / Propellor / Orbitor**：分别做伺服跟随、振荡、追踪目标、线性追踪、弹跳、推进、绕轨。
- **Pots**（[pots.hpp](file:///workspace/src/client/pots.hpp)）：客户端的"位置/朝向目标"集合，配合上述驱动器。
- **ActionMatcher**（[action_matcher.hpp](file:///workspace/src/client/action_matcher.hpp)）：一种 `Motor`，按 Entity 速度、Entity-Model 夹角、caps 位标志，从 `.model` 的 `<action>` 列表里挑动画并按 `scalePlaybackSpeed` 调速，`feetFollowDirection` 让脚跟 `DirectionCursor`。新建 PyModel 默认挂一个 ActionMatcher。

这些 Motor 与相机系统通过 `MatrixProvider`（相机/方向光标提供矩阵）解耦联动：Tracker 的目标可以是 `BaseCamera::invViewMatrixProvider()` 或 `DirectionCursor::provider()`，于是模型头部能跟随相机视线。

---

## 8. 客户端 Entity / Player / Filter

### 8.1 Entity（客户端 DEM 视图）

[entity.hpp](file:///workspace/src/client/entity.hpp)：`Entity : public PyInstancePlus`，是分布式实体模型（DEM）的客户端表示。构造：

```cpp
Entity( EntityType & pType, EntityID id, Vector3 & pos, float * pAuxVolatile,
        int enterCount, PyObject * pInitDict, Entity * pSister );
```

生命周期：

- `checkPrerequisites()` / `loadingPrerequisites()`：异步加载模型等资源（`PrerequisitesOrder : public BackgroundTask`，后台线程加载 `resourceIDs`，完成后 `loaded_`）。
- `enterWorld(SpaceID, EntityID vehicleID, bool transient)` / `leaveWorld(transient)`：进出世界，挂到 chunk。
- `destroy()`：销毁。

Entity 持有 `FilterPtr`（位置平滑器，见 §8.4）、`PyModel`（视觉）、`Physics`（物理控制器，[physics.hpp](file:///workspace/src/client/physics.hpp)）。它本身是 Python 类实例（`Py_InstanceHeader`），脚本可挂属性/方法。

### 8.2 EntityType / EntityManager

- [entity_type.hpp](file:///workspace/src/client/entity_type.hpp)：`EntityType` 是实体模板，从 entitydef 的 `EntityDescription` + Python 脚本类构造。`init()/reload()` 加载所有类型；`find(uint)/find(string)` 按 ID/名字查；`newEntity(...)` 按流内容（`BASE_PLAYER_DATA/CELL_PLAYER_DATA/...`）反序列化创建。`pClass()` 返回 Python 类对象，`pPlayerClass()` 返回玩家专用派生类。
- [entity_manager.hpp](file:///workspace/src/client/entity_manager.hpp)：`EntityManager : public ServerMessageHandler`，存储客户端所有 entity 并控制显示。它实现 `ServerMessageHandler` 的全部回调（见 connection 文档），把服务端消息转成 entity 操作：

| 回调 | 作用 |
|------|------|
| `onBasePlayerCreate/onCellPlayerCreate` | 创建玩家 entity（base/cell 两次） |
| `onEntityCreate` | 创建普通 entity |
| `onEntityProperties/onEntityProperty` | 属性更新 |
| `onEntityMethod` | 客户端方法调用 |
| `onEntityEnter/onEntityLeave` | 进出 space/vehicle |
| `onEntityMoveWithError` | 带 posError 的位置更新（喂给 Filter） |
| `onEntityControl` | 切换对该 entity 的控制权（玩家接管） |
| `spaceData/spaceGone` | space 元数据 |
| `onVoiceData` | 语音数据 |
| `onStreamComplete` | 资源流下载完成 |
| `onRestoreClient` | 断线重连恢复 |
| `onEntitiesReset` | 服务端要求重置（可保留 player） |

`EntityManager` 维护 `enteredEntities_`（已进世界的）和 `cachedEntities_`（缓存），以及对"未知 entity"的 `unknownEntities_` + `queuedMessages_`（entity 还没造出来时先排队消息，造好后再 `forwardQueuedMessages`）。`FIRST_CLIENT_ID = (1<<30)+1` 起的 ID 是客户端自造 entity（`isClientOnlyID` 判定），`nextClientID()` 分配。

### 8.3 Player 与连接控制

- [player.hpp](file:///workspace/src/client/player.hpp)：`Player : public InitSingleton<Player>`，单例，存当前玩家 entity 指针 + 玩家专属对象（`EntityPhotonOccluder` 用于镜头光晕遮挡）。`setPlayer(Entity*)` 切玩家；`camTarget()` 返回相机目标的 `MatrixProvider`；`findChunk()` 查玩家所在 chunk；`drawPlayer()` 绘制玩家；`poseUpdateNotification(Entity*)` 通知 pose 更新。`Player::entity()` 是全局快捷取玩家 entity。
- [connection_control.hpp](file:///workspace/src/client/connection_control.hpp)：`ConnectionControl` 单例，封装对 `ServerConnection` 的连接控制。`connect(server, loginParams, progressFn)` 启动登录（内部 `LoginHandler`），`disconnect()`，`probe(server, progressFn)` 探测，`tick()` 每帧推进。`progressFn` 回调码：0=初始检查、1=登录（成功会断开重连 baseapp）、2=数据、6=已断开。内部 `RSAStreamEncoder` 加密登录参数，`LoggerMessageForwarder`（启用 watchers 时）转发日志给 machined。

### 8.4 Filter 体系：客户端位置平滑

[filter.hpp](file:///workspace/src/client/filter.hpp)：`Filter : public PyObjectPlus` 抽象基类。Filter 处理 entity 的"volatile"成员（position/yaw/pitch/roll），这些成员服务端是"尽力传送"的，频率随距离变化，Filter 负责把它们变成视觉平滑的运动。

```cpp
class Filter : public PyObjectPlus {
    virtual void reset( double time );
    virtual void input( double time, SpaceID, EntityID vehicle,
                        const Position3D & pos, const Vector3 & posError,
                        float * auxFiltered );          // 服务端更新喂入
    virtual void output( double time ) = 0;              // 产出当前帧位置给 entity
    virtual void owner( Entity * pEntity );
    static bool isActive();                               // 全局开关
};
```

> 注：`pos.y ≈ -13000` 是"在地面"标记，需按 (x,z) 采样地形高度。`posError` 是位置误差椭球，用于平滑置信度。

各 Filter 子类：

| Filter | 文件 | 算法 |
|--------|------|------|
| `AvatarFilter` | [avatar_filter.hpp](file:///workspace/src/client/avatar_filter.hpp) | 标准 avatar 平滑：存最近 `NUM_STORED_INPUTS=8` 个输入，按 `latency` 偏移时间在历史中线性插值；`idealLatency` 向最近两次更新间隔（或 `2*minLatency`）靠拢，速度 `velLatency`。支持 Python `callback(whence, fn, extra, passMissedBy)` 在某时刻回调 |
| `PlayerAvatarFilter` | [player_avatar_filter.hpp](file:///workspace/src/client/player_avatar_filter.hpp) | 玩家专用（玩家自己延迟更低/无延迟补偿） |
| `AvatarDropFilter` | [avatar_drop_filter.hpp](file:///workspace/src/client/avatar_drop_filter.hpp) | 处理下落（垂直运动） |
| `BoidsFilter` | [boids_filter.hpp](file:///workspace/src/client/boids_filter.hpp) | 群体（boids）行为平滑 |
| `DumbFilter` | [dumb_filter.hpp](file:///workspace/src/client/dumb_filter.hpp) | 最简单：直接用最新输入（无插值） |
| 通用工具 | [filter_utility_functions.hpp](file:///workspace/src/client/filter_utility_functions.hpp) | 共用插值/方向计算 |

`AvatarFilter` 内部 `StoredInput`（time/spaceID/vehicleID/position/positionError/direction/onGround）环形数组，`Waypoint`（前后两个路点）做插值，`CallbackQueue`（按时间排序的优先队列）管理定时回调。`latency_`、`idealLatency_`、`velLatency_/minLatency_/latencyFrames_/latencyCurvePower_` 都是 Python 可调属性。

#### AvatarFilter 工作流

```
服务端 onEntityMoveWithError ─► EntityManager ─► entity->filter_->input(t, space, veh, pos, posError, aux)
                                                       │ 存入 storedInputs_[currentInputIndex_++ % 8]
                                                       │ idealLatency = (新输入间隔 or 2*minLatency)
                                                       ▼
每帧 entity tick ─► filter_->output( now )             │ latency 向 idealLatency 靠拢
                                                       │ chooseNextWaypoint(now) 选下一对 waypoint
                                                       │ extract(now, space, veh, pos, vel, dir)
                                                       ▼
                                              entity 位置/朝向更新 ─► PyModel ─► ActionMatcher 选动画
```

`app.cpp` 顶部 `filterTokenSet` 引用了所有 filter 的 `extern int XxxFilter_token`，确保这些 Python 类型被链接进来（token 机制是 BigWorld 的"强制引用"惯用法，避免被链接器优化掉）。

### 8.5 其它客户端 entity 相关

- [chunk_entity.hpp](file:///workspace/src/client/chunk_entity.hpp)：`ChunkEntity : public ChunkItem`，chunk 里保存的"静态/客户端 entity"（地图编辑器放的 NPC 等）。`ChunkEntityCache` 在 chunk bind 时才创建它们的 Python 对象。
- [py_chunk_model.hpp](file:///workspace/src/client/py_chunk_model.hpp)：把 chunk 里的模型暴露给 Python。
- [subspace.hpp](file:///workspace/src/client/subspace.hpp)：`SubSpace`，space 的子空间抽象（vehicle 内部等），entity 可挂在不同 subspace。
- [portal_state_changer.hpp](file:///workspace/src/client/portal_state_changer.hpp)：改 portal 状态（开关门/可见性）。
- [entity_picker.hpp](file:///workspace/src/client/entity_picker.hpp)：`EntityPicker` 单例，按 caps → 距离 → 视野角 → 视线 顺序挑可选中的 entity（瞄准系统）。
- [entity_flare_collider.hpp](file:///workspace/src/client/entity_flare_collider.hpp)：`EntityPhotonOccluder`，玩家镜头光晕的遮挡体。
- [py_entities.hpp](file:///workspace/src/client/py_entities.hpp)：把 `EntityManager::entities()` 暴露成 Python 可迭代对象。

### 8.6 杂项客户端模块

| 文件 | 作用 |
|------|------|
| [server_discovery.hpp](file:///workspace/src/client/server_discovery.hpp) | `ServerDiscovery : public MainLoopTask, PyObjectPlus`，LAN 内发现 BigWorld 服务器（广播/收应答） |
| [critical_handler.hpp](file:///workspace/src/client/critical_handler.hpp) | 致命错误处理（`CRITICAL_MSG` 触发的对话框/退出） |
| [shadow_manager.hpp](file:///workspace/src/client/shadow_manager.hpp) | 阴影管理 |
| [time_globals.hpp](file:///workspace/src/client/time_globals.hpp) | 全局时间（game time / real time） |
| [version_info.hpp](file:///workspace/src/client/version_info.hpp) | 版本信息（与 loginapp 握手时校验） |
| [bwvoip.hpp](file:///workspace/src/client/bwvoip.hpp) / [voip.hpp](file:///workspace/src/client/voip.hpp) | VoIP 封装 |
| [minimap.hpp](file:///workspace/src/client/minimap.hpp) | 小地图 |
| [latency_gui_component.hpp](file:///workspace/src/client/latency_gui_component.hpp) | 延迟显示 GUI |
| [frame_rate_graph.hpp](file:///workspace/src/client/frame_rate_graph.hpp) | 帧率图 |
| [message_time_prefix.hpp](file:///workspace/src/client/message_time_prefix.hpp) | 日志消息时间前缀 |
| [pathed_filename.hpp](file:///workspace/src/client/pathed_filename.hpp) | 带路径的偏好文件名 |
| [pots.hpp](file:///workspace/src/client/pots.hpp) | Position/Orientation Target Set（驱动器目标集合） |
| [python_server.hpp](file:///workspace/src/client/python_server.hpp) / [py_server.hpp](file:///workspace/src/client/py_server.hpp) | Python 远程交互（调试） |
| [script_bigworld.hpp](file:///workspace/src/client/script_bigworld.hpp) | `BigWorld` 模块的客户端绑定（`BigWorld.camera/player/dcursor/...`） |
| [script_app.hpp](file:///workspace/src/client/script_app.hpp) | ScriptApp task |
| [py_scene_renderer.hpp](file:///workspace/src/client/py_scene_renderer.hpp) | 场景渲染器 Python 绑定 |

---

## 9. 客户端启动时序

把上面所有片段串起来，从双击 exe 到进入主循环：

```
WinMain (winmain.cpp)
  │ GetCommandLineW, 注册 WNDCLASS
  ▼
BWWinMain (bw_winmain.cpp)
  │ 1. BWResource bwresource;                  // 资源根
  │ 2. parseCommandLine → BWResource::init      // BW_RES_PATH / paths.xml
  │ 3. Moo::init()                              // D3D9 设备
  │ 4. CreateWindow( lpClassName, ... )
  │ 5. App app( configFilename, compileTime );  // 客户端单例构造
  ▼
App::App 构造
  │ 读 AppConfig、设 CONFIG_STRING(debug/release/hybrid/...)
  ▼
App::init( hInstance, hWnd )
  │ 1. DeviceApp::s_hInstance_/s_hWnd_ 设入
  │ 2. ConsoleManager::createInstance()
  │ 3. 注册 13 个 MainLoopTask 及依赖（>Device/>VOIP/>Web/>Script/>Camera/>Canvas
  │    />World/>Flora/>Facade/>Lens/>GUI/>Debug/>Finale）
  │ 4. MainLoopTasks::root().init()  ← 这里依次 init 每个 task：
  │        DeviceApp::init   → Moo::rc() 设备、进度条、InputDevices::init(DirectInput+RawInput+IME)
  │        VOIPApp::init      → bwvoip 初始化
  │        WebApp::init        → Mozilla/Scaleform
  │        ScriptApp::init    → Python 解释器、加载 Personality 脚本、注册 BigWorld 模块
  │        CameraApp::init    → ClientCamera、DirectionCursor、ClientSpeedProvider
  │        CanvasApp::init     → 后处理、LOD 控制器
  │        WorldApp::init      → ChunkManager、ChunkSpace、terrain、collision scene
  │        Flora/Facade/Lens/GUI/Debug/Finale
  │ 5. RecreateDeviceCallback::createInstance()
  │ 6. MF_WATCH 注册 watcher（debugKeyEnable、activeConsole、sleepTime）
  │ 7. 若脚本未设 cursor，activeCursor_ = &DirectionCursor::instance()
  │ 8. freeLoadingScreen()、EffectVisualContext::initConstants()
  │ 9. lastTime_ = frameTimerValue()  // 重置，避免首帧 dt 含初始化时间
  ▼
进入 while(!g_bAppQuit) 主循环
  │ PeekMessage → BWWndProc（输入消息推入 InputDevices 队列）
  │ 无消息时 app.updateFrame(g_bActive):
  │    calculateFrameTime → MouseCursor::updateMouseClipping
  │    MainLoopTasks::root().tick(dTime)   // 13 task 依次 tick
  │      ├─ DeviceApp::tick
  │      ├─ ScriptApp::tick  → Personality.tick / Python 回调
  │      ├─ CameraApp::tick   → InputDevices::processEvents(App) → App::handleKeyEvent/...
  │      │                       ClientCamera::update → DirectionCursor::tick → BaseCamera::update
  │      ├─ WorldApp::tick    → ChunkManager::tick、EntityManager::tick、Player::tick、filter.output
  │      └─ ...
  │    MainLoopTasks::root().draw()        // 13 task 依次 draw
  │      └─ WorldApp::draw → drawWorld/drawScene → Moo::rc() 渲染
  │    帧率限制 Sleep
  ▼
WM_CLOSE → handleCloseRequest → PostQuitMessage → 退出循环
App::fini → MainLoopTasks::finiAll → Moo::fini
```

注意 `CameraApp::tick` 阶段才真正 `InputDevices::processEvents(App)`——这与 `appmgr/App` 在 `updateFrame` 开头就 processEvents 不同，是 `client/App` 的实现细节（输入在 Camera 这个 task 阶段被消费）。

---

## 10. 关键设计总结

1. **双 App**：`appmgr/App`（库层、PresentThread + Module 栈）与 `client/App`（客户端单例、MainLoopTask 链）。前者用于编辑器/工具，后者用于游戏可执行文件。
2. **MainLoopTask 拓扑**：13 个 task 用 `">X"` 规则声明依赖，`MainLoopTasks` 拓扑排序后统一 tick/draw，避免散落的 `update` 调用。每个 task 是单例 + token 强制链接。
3. **输入两级路由**：库层 `InputManager`（console → module → ApplicationInput）用于 appmgr 框架；客户端 `App::handleKeyEvent` 用 `keyRouting_` 表保证 keydown/keyup 配对，按 Debug → Console → 脚本钩子 → Scaleform → Personality → App → Player 兜底 的优先级分发。
4. **相机 MatrixProvider 解耦**：相机与模型/Tracker 通过 `MatrixProvider`（延迟矩阵）交互，相机可"跟随"某 MatrixProvider，也可把自己的 view 包成 MatrixProvider 给 Tracker。`DirectionCursor` 是视线源，`ClientSpeedProvider` 把鼠标轴事件非线性加速成方向变化。
5. **Filter 平滑**：服务端 volatile 数据"尽力传送"，客户端 `AvatarFilter` 存 8 个历史点、用 latency 偏移时间做插值，`latency` 自适应向 idealLatency 靠拢。`PlayerAvatarFilter` 给玩家更低延迟。
6. **token 强制链接**：`extern int Xxx_token;` + `static int x = A_token | B_token | ...;` 是 BigWorld 惯用法，把可选 Python 类型/ChunkItem/Filter/Motor 拉进链接，否则静态库会被裁掉。
7. **PresentThread 异步**：D3D Present 在独立线程，主线程 `onPresent` 自旋等待，并用 D3D query 强制 CPU/GPU 同步以避免鼠标抖动。

> 未在本文展开的部分：`connection` 层见 [16-客户端框架-连接层-connection.md](16-客户端框架-连接层-connection.md)；服务端 entity/cellapp/baseapp 见 09/10/11 各文档；构建与部署见 [22-功能纵切-构建与部署.md](22-功能纵切-构建与部署.md)。
