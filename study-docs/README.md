# BigWorld Engine 2.0.1 源码学习文档

> 本目录是 BigWorld Technology 2.0.1 引擎源码（位于 [/workspace](file:///workspace)）的系统分析文档集。
> 文档从 **代码模块** 与 **功能纵切** 两条主线展开，覆盖引擎的每个核心子系统。
> 共 **24 份文档**，约 **41,000 行 / 1.9 MB**。

---

## 阅读顺序建议

### 路线 A：服务端工程师
1. [00-架构总览](file:///workspace/study-docs/00-架构总览.md) ← 先看
2. [08-服务端框架-server](file:///workspace/study-docs/08-服务端框架-server.md)
3. [07-实体定义-entitydef](file:///workspace/study-docs/07-实体定义-entitydef.md)
4. [06-网络层-Mercury](file:///workspace/study-docs/06-网络层-Mercury.md)
5. [10-服务端进程-cellapp](file:///workspace/study-docs/10-服务端进程-cellapp.md)
6. [09-服务端进程-baseapp](file:///workspace/study-docs/09-服务端进程-baseapp.md)
7. [11-服务端进程-mgr-dbmgr-loginapp-reviver](file:///workspace/study-docs/11-服务端进程-mgr-dbmgr-loginapp-reviver.md)
8. [19-功能纵切-AOI与可见性](file:///workspace/study-docs/19-功能纵切-AOI与可见性.md)
9. [20-功能纵切-持久化与备份恢复](file:///workspace/study-docs/20-功能纵切-持久化与备份恢复.md)
10. [21-功能纵切-脚本与对象模型](file:///workspace/study-docs/21-功能纵切-脚本与对象模型.md)
11. [22-功能纵切-构建与部署](file:///workspace/study-docs/22-功能纵切-构建与部署.md)

### 路线 B：客户端工程师
1. [00-架构总览](file:///workspace/study-docs/00-架构总览.md) ← 先看
2. [01-基础库-cstdmf](file:///workspace/study-docs/01-基础库-cstdmf.md)
3. [02-数学库-math](file:///workspace/study-docs/02-数学库-math.md)
4. [03-资源管理-resmgr](file:///workspace/study-docs/03-资源管理-resmgr.md)
5. [12-渲染核心-moo](file:///workspace/study-docs/12-渲染核心-moo.md)
6. [13-模型系统-model](file:///workspace/study-docs/13-模型系统-model.md)
7. [04-场景分块-chunk](file:///workspace/study-docs/04-场景分块-chunk.md)
8. [05-地形系统-terrain](file:///workspace/study-docs/05-地形系统-terrain.md)
9. [15-环境与氛围-romp](file:///workspace/study-docs/15-环境与氛围-romp.md)
10. [14-粒子与后处理-particle-postfx](file:///workspace/study-docs/14-粒子与后处理-particle-postfx.md)
11. [16b-客户端框架-appmgr-input-camera-entity](file:///workspace/study-docs/16b-客户端框架-appmgr-input-camera-entity.md)
12. [16-客户端框架-连接层-connection](file:///workspace/study-docs/16-客户端框架-连接层-connection.md)
13. [17-GUI与UI栈-gui-scaleform-web](file:///workspace/study-docs/17-GUI与UI栈-gui-scaleform-web.md)

### 路线 C：工具/编辑器工程师
1. [00-架构总览](file:///workspace/study-docs/00-架构总览.md)
2. [03-资源管理-resmgr](file:///workspace/study-docs/03-资源管理-resmgr.md)
3. [04-场景分块-chunk](file:///workspace/study-docs/04-场景分块-chunk.md)
4. [17-GUI与UI栈-gui-scaleform-web](file:///workspace/study-docs/17-GUI与UI栈-gui-scaleform-web.md)
5. [18-工具与编辑器-ual-gizmo-duplo](file:///workspace/study-docs/18-工具与编辑器-ual-gizmo-duplo.md)
6. [附录A-job_system-zip-png](file:///workspace/study-docs/附录A-job_system-zip-png.md)

### 路线 D：网络/协议工程师
1. [06-网络层-Mercury](file:///workspace/study-docs/06-网络层-Mercury.md) ← 核心
2. [07-实体定义-entitydef](file:///workspace/study-docs/07-实体定义-entitydef.md)（序列化）
3. [16-客户端框架-连接层-connection](file:///workspace/study-docs/16-客户端框架-连接层-connection.md)（登录协议）
4. [11-服务端进程-mgr-dbmgr-loginapp-reviver](file:///workspace/study-docs/11-服务端进程-mgr-dbmgr-loginapp-reviver.md)（machined 协议）
5. [22-功能纵切-构建与部署](file:///workspace/study-docs/22-功能纵切-构建与部署.md)（machined 守护）

### 路线 E：想看跨模块"一个移动请求如何传播"
1. [19-功能纵切-AOI与可见性](file:///workspace/study-docs/19-功能纵切-AOI与可见性.md)（含完整时序图）
2. [21-功能纵切-脚本与对象模型](file:///workspace/study-docs/21-功能纵切-脚本与对象模型.md)（Python ↔ C++ 桥接）

---

## 文档总览

### 第 0 部分：总览（1 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [00-架构总览.md](file:///workspace/study-docs/00-架构总览.md) | 267 | 引擎定位、目录结构、库全景、进程拓扑、关键抽象、文档索引 |

### 第 1 部分：基础库（3 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [01-基础库-cstdmf.md](file:///workspace/study-docs/01-基础库-cstdmf.md) | 1596 | 容器 / 智能指针 / 单例 / 内存诊断 / 时间 / Watcher 系统 / 后台任务 |
| [02-数学库-math.md](file:///workspace/study-docs/02-数学库-math.md) | 1216 | Vector2/3/4 / Matrix / Quaternion / 包围盒 / 噪声 / 插值 |
| [附录A-job_system-zip-png.md](file:///workspace/study-docs/附录A-job_system-zip-png.md) | 721 | JobSystem（CommandBuffer 双缓冲） / zlib / libpng |

### 第 2 部分：资源与场景（3 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [03-资源管理-resmgr.md](file:///workspace/study-docs/03-资源管理-resmgr.md) | 913 | BWResource / DataSection / 多文件系统 / packed/zip/xml/dir |
| [04-场景分块-chunk.md](file:///workspace/study-docs/04-场景分块-chunk.md) | 1870 | ChunkManager / ChunkSpace / GeometryMapping / ChunkItem / Umbra / StationGraph |
| [05-地形系统-terrain.md](file:///workspace/study-docs/05-地形系统-terrain.md) | 1904 | Terrain v1/v2 / LOD / HorizonShadow / QuadTree / 顶点 LOD |

### 第 3 部分：网络层（2 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [06-网络层-Mercury.md](file:///workspace/study-docs/06-网络层-Mercury.md) | 2298 | NetworkInterface / Channel / Bundle / Packet / IDL / 加密 / machined 协议 |
| [16-客户端框架-连接层-connection.md](file:///workspace/study-docs/16-客户端框架-连接层-connection.md) | 1691 | ServerConnection / LoginHandler / RSA 登录 / 数据下载 |

### 第 4 部分：实体与服务端（5 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [07-实体定义-entitydef.md](file:///workspace/study-docs/07-实体定义-entitydef.md) | 1138 | EntityDescription / DataType / MethodDescription / MailBoxBase / BitReader |
| [08-服务端框架-server.md](file:///workspace/study-docs/08-服务端框架-server.md) | 1012 | ServerApp / EntityApp / ManagerApp / TimeKeeper / BackupHash / Watcher 转发 |
| [09-服务端进程-baseapp.md](file:///workspace/study-docs/09-服务端进程-baseapp.md) | 2054 | BaseApp / Proxy / 备份归档 / SQLite 二级库 / 客户端 mailbox |
| [10-服务端进程-cellapp.md](file:///workspace/study-docs/10-服务端进程-cellapp.md) | 3796 | Cell / RangeList / Witness / Controller 体系 / Ghost / BufferedGhostMessage |
| [11-服务端进程-mgr-dbmgr-loginapp-reviver.md](file:///workspace/study-docs/11-服务端进程-mgr-dbmgr-loginapp-reviver.md) | 3071 | baseappmgr / cellappmgr / dbmgr / loginapp / reviver / 受控停机 |

### 第 5 部分：渲染（5 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [12-渲染核心-moo.md](file:///workspace/study-docs/12-渲染核心-moo.md) | 4145 | D3D 封装 / RenderContext / Effect / Texture / Visual / Animation / Light |
| [13-模型系统-model.md](file:///workspace/study-docs/13-模型系统-model.md) | 749 | SuperModel / NodefullModel / Dye / Fashion / ModelFactory |
| [14-粒子与后处理-particle-postfx.md](file:///workspace/study-docs/14-粒子与后处理-particle-postfx.md) | 1764 | ParticleSystem / PSA / PostFX Manager / Phase / FilterQuad |
| [15-环境与氛围-romp.md](file:///workspace/study-docs/15-环境与氛围-romp.md) | 2057 | EnviroMinder / TimeOfDay / Weather / Water / Flora / Font / XConsole |
| [16b-客户端框架-appmgr-input-camera-entity.md](file:///workspace/study-docs/16b-客户端框架-appmgr-input-camera-entity.md) | 966 | App / Module / Input / Camera / Player / AvatarFilter |

### 第 6 部分：UI 与工具（2 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [17-GUI与UI栈-gui-scaleform-web.md](file:///workspace/study-docs/17-GUI与UI栈-gui-scaleform-web.md) | 1127 | controls / guimanager / scaleform / web_render / input |
| [18-工具与编辑器-ual-gizmo-duplo.md](file:///workspace/study-docs/18-工具与编辑器-ual-gizmo-duplo.md) | 1652 | UAL / Gizmo / Duplo / Ashes / OpenAutomate |

### 第 7 部分：功能纵切（4 份）

| 文件 | 行数 | 主题 |
| --- | --- | --- |
| [19-功能纵切-AOI与可见性.md](file:///workspace/study-docs/19-功能纵切-AOI与可见性.md) | 1032 | Witness / RangeList / ProximityController / Ghost / 跨 cell 边界 |
| [20-功能纵切-持久化与备份恢复.md](file:///workspace/study-docs/20-功能纵切-持久化与备份恢复.md) | 942 | 持久化分层 / 备份哈希环 / 崩溃恢复 / 二级库 |
| [21-功能纵切-脚本与对象模型.md](file:///workspace/study-docs/21-功能纵切-脚本与对象模型.md) | 1483 | PyObjectPlus / Entity 三态 / MailBox / ScriptTimers / 热重载 |
| [22-功能纵切-构建与部署.md](file:///workspace/study-docs/22-功能纵切-构建与部署.md) | 1608 | VS/Make 构建 / machined 守护 / 进程编排 / 受控停机 |

---

## 关键概念速查

| 概念 | 出现文档 | 一句话定义 |
| --- | --- | --- |
| **Mercury** | [06](file:///workspace/study-docs/06-网络层-Mercury.md) | BigWorld 自研的 UDP 可靠传输 + RPC 框架，所有进程间通信的基础 |
| **Entity / Base / Cell portion** | [07](file:///workspace/study-docs/07-实体定义-entitydef.md), [09](file:///workspace/study-docs/09-服务端进程-baseapp.md), [10](file:///workspace/study-docs/10-服务端进程-cellapp.md) | 一个逻辑 Entity 在 BigWorld 中由 1~3 个 portion 组成，分布在不同进程 |
| **Mailbox** | [07](file:///workspace/study-docs/07-实体定义-entitydef.md) | 指向远程 Entity 的引用，让跨进程调用在 Python 中像本地调用 |
| **Witness** | [19](file:///workspace/study-docs/19-功能纵切-AOI与可见性.md) | 玩家的"目击者"，决定把哪些 Entity 推给客户端 |
| **AOI** | [19](file:///workspace/study-docs/19-功能纵切-AOI与可见性.md) | Area of Interest，客户端只接收"它能看到"的 Entity 的更新 |
| **Ghost Entity** | [10](file:///workspace/study-docs/10-服务端进程-cellapp.md) | 跨 cell 边界的 Entity 的"代理副本"，由 BufferedGhostMessage 缓冲 |
| **Chunk** | [04](file:///workspace/study-docs/04-场景分块-chunk.md) | 场景的网格化分块，与 cell 边界对齐，按需流式加载 |
| **BackupHash 环** | [20](file:///workspace/study-docs/20-功能纵切-持久化与备份恢复.md), [08](file:///workspace/study-docs/08-服务端框架-server.md) | baseapp 间环形备份分布，节点死亡时邻居接管 |
| **SuperModel** | [13](file:///workspace/study-docs/13-模型系统-model.md) | 多个 Model 的聚合体，外观一致 |
| **PyObjectPlus** | [21](file:///workspace/study-docs/21-功能纵切-脚本与对象模型.md) | 所有暴露到 Python 的 C++ 对象的基类 |
| **machined** | [22](file:///workspace/study-docs/22-功能纵切-构建与部署.md) | 每台物理机器一个守护进程，启停 server、健康检查、watcher 转发 |
| **Watcher** | [01](file:///workspace/study-docs/01-基础库-cstdmf.md), [08](file:///workspace/study-docs/08-服务端框架-server.md) | BigWorld 标志性的运行时变量观察/修改机制 |

---

## 文档统计

- **文档总数**：24 份（22 份正文 + 1 份总览 + 1 份附录）
- **总行数**：41,072 行
- **总大小**：约 1.93 MB
- **平均每份**：约 1,710 行 / 81 KB
- **最长**：[12-渲染核心-moo.md](file:///workspace/study-docs/12-渲染核心-moo.md)（4,145 行）
- **最短**：[00-架构总览.md](file:///workspace/study-docs/00-架构总览.md)（267 行）

---

## 引用约定

- 所有源码引用均使用 `[文件名](file:///workspace/绝对路径)` 格式的可点击链接
- 代码块统一使用 ```cpp 标注（Python 代码用 ```python，构建脚本用 ```bash / ```make / ```xml）
- 类层级图、模块依赖图、时序图均为 ASCII 文本，便于在终端与 Markdown 渲染器中查看
- 文档间互引使用相对链接（如 `[详见 19](19-功能纵切-AOI与可见性.md)`）

---

## 源码参考

- 引擎源码：[/workspace/src/](file:///workspace/src/)
- API 文档（Doxygen 生成）：[/workspace/doc/api_cpp/](file:///workspace/doc/api_cpp/)
- 引擎版本：BigWorld Technology 2.0.1
- 版权：BigWorld Pty, Ltd.（Commercial in confidence）

---

## 局限与已知缺失

以下主题在源码中存在但因篇幅或优先级未深入：
- `speedtree` / `speedtreert`（SpeedTree 植被，第三方库源码未随仓库提供）
- `fmod` / `fmodsound` / `xactsnd`（音频后端，第三方库）
- `machined` 守护进程的 C++ 源码（不在本仓库，仅 `machine_guard.hpp` 协议可用）
- `cellappmgr` 类实现（仅 `cellappmgr_interface.hpp` 随源码提供，类实现未开放）
- MySQL 后端实现细节（`dbmgr_mysql` 目录的内部行为）
- 部分依赖（Scaleform、Umbra、Mozilla）以预编译 `.lib` 形式给出，源码不在本仓库
