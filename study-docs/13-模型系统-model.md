# BigWorld Engine 2.0.1 模型系统（model）

> 源码目录：`/workspace/src/lib/model/`
> 模块定位：客户端**模型层**——构建在 `moo::Visual` 之上，对外提供 **多模型聚合、离散 LOD、骨骼动画、动作（Action）、染色（Dye）、材质覆盖（Material Override）、静态光照** 等高级抽象。它把“一个游戏角色”从一堆 `.model` + `.visual` + `.animation` 文件，组装成可在场景中 tick/draw 的统一实体 `SuperModel`。
> 模块边界：上层由 [chunk](file:///workspace/src/lib/chunk/) 的 `ChunkModel`、客户端实体 `PyModel`、编辑器驱动；下层把真正的网格/材质/动画提交委托给 [moo](file:///workspace/src/lib/moo/)（`Moo::Visual` / `Moo::Node` / `Moo::Animation` / `Moo::EffectMaterial`），文件 IO 走 [resmgr](file:///workspace/src/lib/resmgr/) 的 `BWResource`。
> 编号：13（与 `04-场景分块-chunk.md` 中提到的 `ChunkModel`、`16-客户端框架-连接层-connection.md` 的 `PyModel` 衔接）。

---

## 1. 模块定位与核心思想

`model` 库要解决的核心问题是：**一个“看起来是一个整体”的角色，实际由多个 part 拼装、多个 LOD 级别串联、并能换装/换色/换动画**。设计文档 [supermodelinfo.txt](file:///workspace/src/lib/model/supermodelinfo.txt) 开篇即点明：

> a SuperModel is a collection of parts. Each part has a tree of nodes in it. Parts are connected to each other, and the nodes in each part are attached as a subtree to the connecting node to form a skeleton.

由此衍生出几个关键设计决策（均出自 `supermodelinfo.txt`）：

1. **没有 supermodel 文件**：`SuperModel` 是运行时实例类，由调用方传入一组 `modelIDs`（`.model` 资源 ID）构造，而非由磁盘上的某个“supermodel 描述文件”定义。多 part 的拼装信息存在于各 `.visual` 的节点连接关系里。
2. **`Model` 是静态单例**：每个 `.model` 文件在进程内只有一份 `Model` 实例（“only one instance of this class for each model file, as it represents only the static data”），动态数据（动画时间、染色状态）存在引用它的 `SuperModel` 里。见 [model.hpp](file:///workspace/src/lib/model/model.hpp)。
3. **继承链即 LOD 链**：`.model` 通过 `<parent>` 字段构成父子链，子模型覆盖父模型的 visual/animation，并通过 `<extent>`（可见距离）实现离散 LOD 切换。动画在链上**累积**，染色在链上**拷贝**。
4. **扁平命名空间**：所有动画名、染色 matter/tint 名、属性（property）名都是扁平的，跨 part / 跨 LOD 共享一个全局属性表 `Model::s_propCatalogue_`。

### 1.1 模块依赖关系图

```
        ┌─────────────────────────────────────────────┐
        │              上层调用方                      │
        │  ChunkModel (chunk)  /  PyModel (client)    │
        │  SuperModelProgress / ModelEditor (romp)    │
        └────────────────────┬────────────────────────┘
                             │ 构造 SuperModel(modelIDs)
                             │        tick / draw
                             ▼
   ┌────────────────────────────────────────────────────────────┐
   │                       SuperModel                           │  ← 运行时实例
   │   models_ (vector<ModelStuff>)  lod_ / lodNextUp_ / Down    │
   │   draw(): LOD 选择 → dress(Fashion) → soakDyes → draw      │
   │   getAnimation / getAction / getDye / getMatchableActions   │
   └───────┬────────────────────────────────────┬───────────────┘
           │ 持有 topModel / curModel            │ 持有 Fashion*
           ▼                                    ▼
   ┌──────────────────────────┐    ┌────────────────────────────┐
   │     Model (静态单例)      │    │        Fashion 体系         │
   │  parent_ / extent_        │    │  SuperModelAnimation        │
   │  animations_ / actions_   │    │  SuperModelAction           │
   │  matters_ (dye 定义)      │    │  SuperModelDye              │
   │  Model::get() / reload()  │    │  (dress/undress SuperModel) │
   └──┬──────────────┬─────────┘    └────────────────────────────┘
      │ 继承          │ 工厂
      ▼              ▼
   ┌─────────────────────────────────────────┐
   │   NodefullModel │ NodelessModel         │
   │   (有骨骼节点)   │ (SwitchedModel<Vis>)  │
   │   nodeTree_      │  bulk_/frameDress_    │
   │   bulk_(Visual)  │  逐帧切换 visual       │
   └────────┬────────────────┬───────────────┘
            │                │
            ▼                ▼
   ┌────────────────────────────────────────────────────────────┐
   │  ModelFactory / ModelMap / createModelFromFile(model_common)│
   │  NodeTree / ModelAction / ModelAnimation / MatchInfo        │
   │  Matter / Tint / DyeProperty / DyeSelection / ModelDye      │
   │  MaterialOverride / ModelStaticLighting                     │
   └────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
   ┌────────────────────────────────────────────────────────────┐
   │  下游：moo (Visual/Node/Animation/EffectMaterial/NodeCatalogue)│
   │       resmgr (BWResource/DataSection)  physics2 (BSPTree)    │
   │       romp (StaticLightValues / LodSettings)                │
   └────────────────────────────────────────────────────────────┘
```

对外的高频入口只有两个：`Model::get(resourceID)`（加载/缓存静态模型）与 `SuperModel::draw()`（一帧渲染）。其余 `getAnimation/getAction/getDye` 都是构造 `Fashion` 用的辅助接口。

---

## 2. 顶层抽象：SuperModel

`SuperModel` 是“一个可绘制角色”的运行时化身，定义在 [super_model.hpp](file:///workspace/src/lib/model/super_model.hpp)。它的注释自嘲式地概括了其野心：

```cpp
// super_model.hpp:45
/**
 *	This class defines a super model. It's not just a model,
 *	it's a whole lot more - discrete level of detail, billboards,
 *	continuous level of detail, static mesh animations, multi-part
 *	animated models - you name it, this baby's got it.
 */
class SuperModel
{
public:
	explicit SuperModel( const std::vector< std::string > & modelIDs );
	float draw( const FashionVector * pFashions = NULL,
		int nLateFashions = 0, float atLod = -1.f, bool doDraw = true );
	void dressInDefault();
	SuperModelAnimationPtr	getAnimation( const std::string & name );
	SuperModelActionPtr		getAction( const std::string & name );
	SuperModelDyePtr		getDye( const std::string & matter, const std::string & tint );
	// ...
private:
	struct ModelStuff
	{
		ModelPtr	  topModel;   // 最高 LOD 的模型（动画/action 来源）
		Model		* preModel;  // LOD 切换时的前一个模型
		Model		* curModel;  // 当前 LOD 选中的模型
	};
	std::vector<ModelStuff>		models_;
	int			nModels_;
	float		lod_;            // 当前 LOD（归一化距离）
	float		lodNextUp_;      // 下一次向上切 LOD 的阈值
	float		lodNextDown_;    // 下一次向下切 LOD 的阈值
	bool		isOK_;
};
```

### 2.1 构造：从 modelIDs 到 ModelStuff

构造函数逐个 `Model::get()` 加载 `.model`，每个成功的模型压成一个 `ModelStuff`，并把它的 `extent()` 取作初始 `lodNextDown_`（最小的可见距离作为整体下界）。任一模型加载失败只置 `isOK_=false`，不中断其余加载：

```cpp
// super_model.cpp:163
SuperModel::SuperModel( const std::vector< std::string > & modelIDs ) :
	lod_( 0.f ), nModels_( 0 ),
	lodNextUp_( -1.f ), lodNextDown_( 1000000.f ),
	checkBB_( true ), redress_( false ), isOK_( true )
{
	for (uint i = 0; i < modelIDs.size(); i++)
	{
		ModelPtr pModel = Model::get( modelIDs[i] );
		if (pModel == NULL) { isOK_ = false; continue; }
		ModelStuff ms;
		ms.topModel = pModel;
		ms.curModel = ms.topModel.getObject();
		ms.preModel = NULL;
		models_.push_back( ms );
		lodNextDown_ = min( lodNextDown_, ms.topModel->extent() );
	}
	nModels_ = models_.size();
}
```

注意 `topModel` 与 `curModel` 的区别：`topModel` 永远是最高 LOD（动画/动作查询从这里走），`curModel` 是按当前 `lod_` 选中的实际绘制模型。这与 `supermodelinfo.txt` 中“actions 的匹配信息总是取自最高 LOD，但实际播放的动画取自当前 LOD 模型”一致。

### 2.2 draw() 的六步流水线

`SuperModel::draw()` 是整个模块的心脏，[super_model.cpp:219](file:///workspace/src/lib/model/super_model.cpp) 把一次绘制拆成清晰的六步：

```cpp
// super_model.cpp:219 （摘录关键步骤）
float SuperModel::draw( const FashionVector * pFashions, int nLateFashions,
	float atLod, bool doDraw )
{
	// Step 2: 计算当前 LOD（基于相机距离 + yscale + zoomFactor + LodSettings 偏置）
	if (atLod < 0.f) {
		float distance = mooWorld.applyToOrigin().dotProduct(...) + mooView._43;
		float yscale = mooWorld.applyToUnitAxisVector(1).length();
		distance = LodSettings::instance().applyLodBias(distance);
		lod_ = (distance / max( 1.f, yscale )) * Moo::rc().zoomFactor();
	} else lod_ = atLod;

	// 仅当 lod 跨越阈值时才重算 curModel（惰性 LOD 切换）
	if ((lod_ > lodNextDown_) || (lod_ <= lodNextUp_)) {
		for (int i=0; i < nModels_; i++) {
			Model* model = models_[i].topModel.getObject();
			while ((model) && (model->extent() != -1.f) && (model->extent() < lod_)) {
				if ((model->extent() != 0.f) &&
					((models_[i].preModel == NULL) ||
					 (model->extent() > models_[i].preModel->extent())))
					models_[i].preModel = model;
				model = model->parent();   // 沿继承链向低 LOD 走
			}
			models_[i].curModel = model;
		}
		// 更新下一次需要重算的上下界
	}

	// Step 3: dress —— 先早 Fashion，再各模型自 dress，再晚 Fashion
	Model::incrementBlendCookie();
	Moo::MorphVertices::incrementCookie();
	// 早 fashion（动画/动作通常放前面）
	for (...fashIt != fashLate...) (*fashIt)->dress( *this );
	// 各模型自己 dress（遍历节点树计算世界变换）
	for (int i = 0; i < nModels_; i++)
		if ((pModel = models_[i].curModel) != NULL) pModel->dress();
	// 晚 fashion（dye 通常放后面，覆盖材质）
	for (; fashIt != fashEnd; fashIt++) (*fashIt)->dress( *this );

	// Step 4: draw —— 每个模型先 soakDyes（提交材质），再 draw
	for (int i = 0; i < nModels_; i++) {
		if ((pModel = models_[i].curModel) != NULL) {
			pModel->soakDyes();
			if (doDraw) pModel->draw(checkBB_);
		}
	}

	// Step 5: undress —— 逆序恢复 fashion 的副作用
	for (fashRIt = pFashions->rbegin(); ...) (*fashRIt)->undress( *this );
	// Step 6: 返回 lod_ 供调用方使用
	return lod_;
}
```

关键点：

- **`blendCookie`**：每次 `draw` 自增的帧戳（`Model::incrementBlendCookie()`，定义见 [model.cpp:835](file:///workspace/src/lib/model/model.cpp)）。动画/染色的“是否本帧已写入”都用它判断，避免重复计算——这是 SuperModel 把多个 Model 的动态状态合并到一个全局节点表里的核心机制。
- **惰性 LOD**：只有当 `lod_` 跨过 `lodNextUp_`/`lodNextDown_` 才重算 `curModel`，帧内 O(nModels) 而非 O(nModels × 链长)。
- **Fashion 早晚分区**：`nLateFashions` 把 `pFashions` 末尾 N 个标记为“晚 fashion”，在模型 `dress()` 之后才执行，这样 dye 可以覆盖模型 dress 阶段设定的默认材质。

### 2.3 SuperModel 的三个 Fashion 子类

`SuperModel` 不直接持有动画/动作/染色数据，而是按需构造三种 `Fashion`（见 §6）返回给调用方，调用方把它们塞进 `FashionVector` 再 `draw()`：

| 方法 | 返回类型 | 作用 | 文件 |
|------|----------|------|------|
| `getAnimation(name)` | `SuperModelAnimationPtr` | 在每个 topModel 里查同名动画索引，dress 时 `playAnimation` | [super_model_animation.hpp](file:///workspace/src/lib/model/super_model_animation.hpp) |
| `getAction(name)` | `SuperModelActionPtr` | 动作 = 动画 + 触发器/混合；dress 时按 blendRatio 播放 | [super_model_action.hpp](file:///workspace/src/lib/model/super_model_action.hpp) |
| `getDye(matter,tint)` | `SuperModelDyePtr` | 在每个 topModel 里查 matter/tint，dress 时写全局属性表 + `soak` | [super_model_dye.hpp](file:///workspace/src/lib/model/super_model_dye.hpp) |

注意 `SuperModelAnimation` / `SuperModelAction` 是**变长类**——`int index[1]` 末尾数组按 `nModels_` 扩展，构造时用 `new char[sizeof(T)+(nModels-1)*sizeof(int)]` 手工分配（[super_model.cpp:420](file:///workspace/src/lib/model/super_model.cpp)）。这是一种把“每模型一个索引”紧凑存放的早期优化：

```cpp
// super_model.cpp:415
SuperModelAnimationPtr SuperModel::getAnimation( const std::string & name )
{
	if (nModels_ == 0) return NULL;
	char * pBytes = new char[sizeof(SuperModelAnimation)+(nModels_-1)*sizeof(int)];
	SuperModelAnimation * pAnim = new ( pBytes ) SuperModelAnimation( *this, name );
	return pAnim;
}
```

`getMatchableActions()` 则遍历所有 topModel 的 action，把 `isMatchable_` 为真的去重收集，供实体动作匹配系统使用（[super_model.cpp:453](file:///workspace/src/lib/model/super_model.cpp)）。

---

## 3. Model 类型层级

所有具体模型都派生自抽象基类 `Model`，[model.hpp](file:///workspace/src/lib/model/model.hpp)。`Model` 继承 `SafeReferenceCount`（线程安全的引用计数），并提供静态加载/缓存、染色读取、动画/动作索引等通用能力。

### 3.1 Model 基类

```cpp
// model.hpp:81
class Model : public SafeReferenceCount
{
	friend ModelAction; friend ModelActionsIterator; friend ModelMap; friend class ModelPLCB;
public:
	static ModelPtr get( const std::string & resourceID, bool loadIfMissing = true );
	ModelPtr reload( DataSectionPtr pFile = NULL, bool reloadChildren = true ) const;

	virtual bool valid() const;
	virtual void dress() = 0;          // 纯虚：遍历节点树/设定状态
	virtual void draw( bool checkBB ) = 0;  // 纯虚：提交到 moo
	virtual const BoundingBox & boundingBox() const = 0;
	virtual const BoundingBox & visibilityBox() const = 0;

	int  getAnimation( const std::string & name ) const;  // 沿 parent 链查
	void tickAnimation( int index, float dtime, float otime, float ntime );
	void playAnimation( int index, float time, float blendRatio, int flags );
	const ModelAction * getAction( const std::string & name ) const;
	ModelDye getDye( const std::string & matterName, const std::string & tintName, Tint ** ppTint = NULL );
	virtual void soak( const ModelDye & dye );

	static int getPropInCatalogue( const char * name );   // 全局属性表
	static PropCatalogue & propCatalogueRaw();
protected:
	static ModelPtr load( const std::string & resourceID, DataSectionPtr pFile );
	Model( const std::string & resourceID, DataSectionPtr pFile );
	virtual int  initMatter( Matter & m ) { return 0; }
	virtual bool initTint( Tint & t, DataSectionPtr matSect ) { return false; }
	int  readDyes( DataSectionPtr pFile, bool multipleMaterials );
	void postLoad( DataSectionPtr pFile );

	std::string  resourceID_;
	ModelPtr     parent_;       // LOD 继承父
	float        extent_;       // 可见距离
	Animations   animations_;   // vector<ModelAnimationPtr>
	AnimationsIndex animationsIndex_;  // name -> index
	ModelAnimationPtr pDefaultAnimation_;  // “初始姿势”动画
	Actions      actions_;      // vector<ModelActionPtr>
	ActionsIndex actionsIndex_;
	Matters      matters_;      // vector<Matter*>  染色定义
};
```

#### 加载与缓存：Model::get / load / postLoad

`Model::get()` 先查全局 `ModelMap`（`s_pModelMap`，一个 `OneClassPtr<ModelMap>` 懒加载单例），命中则直接返回；未命中则 `BWResource::openSection()` 读 `.model`，进入 `PostLoadCallBack::enter()/leave()` 包裹的 `Model::load()`：

```cpp
// model.cpp:676
ModelPtr Model::load( const std::string & resourceID, DataSectionPtr pFile )
{
	Moo::ScopedResourceLoadContext resLoadCtx( BWResource::getFilename( resourceID ) );
	ModelFactory factory( resourceID, pFile );
	ModelPtr model = Moo::createModelFromFile( pFile, factory );  // 见 §4
	if (!model) { ERROR_MSG(...); return NULL; }
	PostLoadCallBack::run();   // 跑完所有挂起的 postLoad 回调
	if (!(model && model->valid())) { ERROR_MSG(...); return NULL; }
	return model;
}
```

构造函数 [model.cpp:211](file:///workspace/src/lib/model/model.cpp) 做三件事：读 `<parent>` 并递归 `Model::get(parent+".model")`、读 `<extent>`（默认 10000.f）、若有 parent 则**拷贝**其 `animations_` 与 `matters_`（深拷贝 Matter，重置 emulsion）。真正的 dye/action 读取被推迟到 `postLoad()`，通过 `PostLoadCallBack` 机制在“同线程所有模型都构造完”后统一执行——这避免了 dye 引用兄弟模型时尚未就绪的问题。

`postLoad()` [model.cpp:757](file:///workspace/src/lib/model/model.cpp) 读所有 `<action>` 段构造 `ModelAction`，校验后塞进 `actions_`/`actionsIndex_`。

#### blendCookie 机制

```cpp
// model.cpp:835
int Model::incrementBlendCookie()
{
	IF_NOT_MF_ASSERT_DEV(MainThreadTracker::isCurrentThreadMain())
	{ MF_EXIT( "incrementBlendCookie called, but not from the main thread" ); }
	Model::s_blendCookie_ = ((Model::s_blendCookie_ + 1) & 0x0fffffff);
	return Model::s_blendCookie_;
}
```

每帧 `SuperModel::draw` 开头自增一次（初值 `0x08000000`，掩码 28 位）。`Matter::emulsionCookie_`、`Moo::Node` 的 blend cookie 都用它判断“本次 dress 是否已写过本节点/本材质”，从而让多个 part 共享同一份全局节点世界变换表，避免重复矩阵乘法。

### 3.2 NodefullModel：有骨骼的完整模型

[nodefull_model.hpp](file:///workspace/src/lib/model/nodefull_model.hpp) / [nodefull_model.cpp](file:///workspace/src/lib/model/nodefull_model.cpp)。继承 `Model` 与 `Aligned`（16 字节对齐，因含 `Matrix`），`bulk_` 是一个 `Moo::VisualPtr`，并自建 `NodeTree nodeTree_`。

构造流程（[nodefull_model.cpp:34](file:///workspace/src/lib/model/nodefull_model.cpp)）：

1. 读 `<nodefullVisual>` + `.visual`，`Moo::VisualManager::instance()->get()` 加载。
2. 读可选 `<visibilityBox>`，否则用 visual 的 boundingBox。
3. **遍历 visual 的节点树**，把每个节点 `Moo::NodeCatalogue::findOrAdd()` 登记进全局节点目录，同时构建扁平的 `nodeTree_`（DFS 序，每项存 `pNode` + `nChildren`），并记录初始姿势 `initialPose`。
4. `readDyes(pFile, true)` 读染色（`multipleMaterials=true`，走 `initMatter`/`initTint`）。
5. 用 `initialPose` 构造默认动画 `NodefullModelDefaultAnimation`（“初始姿势”，无限时长）。
6. 若有 `<animation>` 段，为模型创建 `.anca`（streamed animation cache），逐段读 `<nodes>` 资源 + `.animation`，构造 `NodefullModelAnimation`，按名登记或覆盖父链同名动画。
7. `postLoad(pFile)` 读 action。

`dress()` 是 NodefullModel 的精华（[nodefull_model.cpp:342](file:///workspace/src/lib/model/nodefull_model.cpp)）：用一个固定大小 `NODE_STACK_SIZE=128` 的栈做**非递归 DFS**，沿 `nodeTree_` 顺序遍历，对每个节点：

- 若本帧尚未被 blend（`pNode->blend(blendCookie)==0`），用 `initialPose[i]` clobber 之（保证未动画的节点保持初始姿势）；
- 调 `pNode->visitSelf(*stack[sat].trans)` 计算世界变换；
- 有子节点则入栈，记录 pop 计数。

`draw()` 极简：push world 矩阵，按 `batched_` 走 `bulk_->draw()` 或 `bulk_->batch()`，pop。

### 3.3 NodelessModel：无骨骼的静态网格

[nodeless_model.hpp](file:///workspace/src/lib/model/nodeless_model.hpp) / [nodeless_model.cpp](file:///workspace/src/lib/model/nodeless_model.cpp)。继承自 `SwitchedModel<Moo::VisualPtr>`（见 §3.4），即“用动画切换 visual”的模型，但没有真正的骨骼节点。它的“动画”是**逐帧切换一整套静态 visual**（静态网格动画）。

构造时调 `wireSwitch(pFile, &loadVisual, "nodelessVisual", "visual")`：先读主 `<nodelessVisual>` 作为 `bulk_` 并生成默认动画，再逐个读 `<animation>` 段，每段含多个 `<visual>` 帧名，按帧率组成 `SwitchedModelAnimation`。`loadVisual()` 会优先尝试 `xxx.static.visual`，回退 `xxx.visual`，并把 visual 的根节点登记到 `NodeCatalogue`（静态 `s_sceneRootNode_`）。

`dress()` [nodeless_model.cpp:76](file:///workspace/src/lib/model/nodeless_model.cpp) 先调父类 `SwitchedModel::dress()`（把 `frameDress_` 提交给 `frameDraw_`），再用 `s_sceneRootNode_` 把当前 world 矩阵写进全局节点表（让附件能挂到它身上）。`draw()` 用 `frameDraw_->draw()/batch()` 提交当前帧。

NodelessModel 还支持静态光照（`getStaticLighting` 返回 `NodelessModelStaticLighting`，见 [nodeless_model_static_lighting.hpp](file:///workspace/src/lib/model/nodeless_model_static_lighting.hpp)）和遮挡器标记（`<umbraOccluder>`/`<dpvsOccluder>`，`occluder()` 返回 `occluder_`）。

### 3.4 SwitchedModel 模板基类

[switched_model.hpp](file:///workspace/src/lib/model/switched_model.hpp) 是个模板，参数 `BULK` 是“帧”的类型（`NodelessModel` 用 `Moo::VisualPtr`）。它管理 `bulk_`（主帧）、`frameDress_`（dress 阶段设定的帧）、`frameDraw_`（实际绘制帧），以及 blend cookie/ratio。

`wireSwitch()` 是核心初始化（[switched_model.hpp:94](file:///workspace/src/lib/model/switched_model.hpp)）：加载主 bulk、构造默认动画、逐个读 `<animation>` 段。帧名解析支持两种模式：显式 `<visual>` 列表，或 `<frameCount>` + 自动生成 `bulkName.animName.N` 命名。`dress()` 简单地把 `frameDress_` 交给 `frameDraw_`，若无则回退 `bulk_`。

注：`switched_model_animation.hpp` 定义 `SwitchedModelAnimation<BULK>`，是 `ModelAnimation` 的子类，按时间在多帧间选择/插值（参见 [switched_model_animation.hpp](file:///workspace/src/lib/model/switched_model_animation.hpp)）。

### 3.5 类型层级图

```
                  ReferenceCount
                       ▲
                       │
              SafeReferenceCount
                       ▲
                       │
            ┌──────────┴──────────┐
            │        Model        │  (model.hpp/cpp)
            │  get/load/reload    │
            │  readDyes/postLoad  │
            │  tick/playAnimation │
            │  blendCookie        │
            └──────┬──────────────┘
                   │
      ┌────────────┼─────────────────────────┐
      │            │                           │
      ▼            ▼                           ▼
 NodefullModel  SwitchedModel<BULK>      (billboard 已注释掉)
 (nodefull_model)   ▲  (switched_model.hpp 模板)
 │ Aligned          │
 │ bulk_: Visual    │
 │ nodeTree_        │
 │ NodefullModel-   │
 │   Animation      ├── NodelessModel : SwitchedModel<Moo::VisualPtr>
 │   DefaultAnimation  (nodeless_model.hpp/cpp)
 │                    │ bulk_/frameDraw_
 │                    │ NodelessModelStaticLighting
 ▼                    ▼
 Moo::Visual       Moo::Visual (静态)
 Moo::NodeCatalogue
```

> 说明：`model_common.hpp` 中 `billboardVisual` 分支被注释，故 2.0.1 客户端实际只有 Nodefull / Nodeless 两类。`ModelCompound`（组合模型）位于 romp 库 [model_compound.hpp](file:///workspace/src/lib/romp/model_compound.hpp)，是把多个 SuperModel 组合渲染的更高层封装，不在 model 库内。

---

## 4. 工厂与映射

### 4.1 createModelFromFile + ModelFactory

[model_common.hpp](file:///workspace/src/lib/model/model_common.hpp) 提供一个模板函数，根据 `.model` 文件里的 section 决定实例化哪种子类：

```cpp
// model_common.hpp:23
template <class FACTORY>
typename FACTORY::ModelBase* createModelFromFile( DataSectionPtr& pFile, FACTORY& factory )
{
	if (pFile->openSection( "nodefullVisual" ))  return factory.newNodefullModel();
	else if (pFile->openSection( "nodelessVisual" )) return factory.newNodelessModel();
	//else if (pFile->openSection( "billboardVisual" )) return factory.newBillboardModel();
	else return NULL;
}
```

注释解释了为何要走工厂而非直接 `new`：建立客户端/服务端的耦合点，“makes people think twice before modifying this code”。`ModelFactory`（[model_factory.hpp](file:///workspace/src/lib/model/model_factory.hpp) / [.cpp](file:///workspace/src/lib/model/model_factory.cpp)）是客户端实现，`newNodefullModel()` 会 `new NodefullModel` 并校验 `valid()`：

```cpp
// model_factory.cpp:36
NodefullModel * ModelFactory::newNodefullModel()
{
	NodefullModel * model = new NodefullModel( resourceID_, pFile_ );
	if (model && model->valid()) return model;
	else return NULL;
}
```

### 4.2 ModelMap：全局已加载模型表

[model_map.hpp](file:///workspace/src/lib/model/model_map.hpp) / [.cpp](file:///workspace/src/lib/model/model_map.cpp)。一个 `std::map<std::string,Model*>` + `SimpleMutex`，提供 `add/del/find/findChildren`。`find()` 返回 `FALLIBLE` 智能指针——若引用计数已为 0，说明有人正阻塞在 `sm_` 上等删除，调用方拿到空指针。

`Model::get()` 在 `load` 成功后、`PostLoadCallBack::leave()` 之前 `add` 进表，保证其他线程能立刻查到。`findChildren()` 用于 `reload()` 时递归重载所有以本模型为 parent 的子模型。

---

## 5. 节点树 NodeTree

[node_tree.hpp](file:///workspace/src/lib/model/node_tree.hpp)。为了支持非递归遍历与跨 part 共享，NodefullModel 把 visual 的节点树**拍平**成 `std::vector<NodeTreeData>`：

```cpp
// node_tree.hpp:28
struct NodeTreeData
{
	NodeTreeData( Moo::Node * pN, int nc ) : pNode( pN ), nChildren( nc ) {}
	Moo::Node	* pNode;
	int			nChildren;
};
typedef std::vector<NodeTreeData>	NodeTree;
```

DFS 序存储，`nChildren` 用于在遍历时知道何时出栈。`NodeTreeIterator` 是一个单向迭代器，内部带 `NODE_STACK_SIZE=128` 的变换矩阵栈，`operator++` 推进并维护 `pParentTransform`，供 `pymodel` 等外部代码线性遍历骨架（注释明示“See pymodel.cpp for an example”）。

`NodefullModel::nodeTreeBegin()/End()` 返回迭代器范围，使外部能不依赖递归地访问整个骨架。

---

## 6. 动作 / 动画 / 染色：Fashion 体系

`Fashion` 是 SuperModel draw 流水线里“可穿戴/可脱卸的修饰”的统一抽象，[fashion.hpp](file:///workspace/src/lib/model/fashion.hpp)：

```cpp
// fashion.hpp:31
class Fashion : public ReferenceCount
{
protected:
	virtual void dress( class SuperModel & superModel ) = 0;
	virtual void undress( class SuperModel & superModel );
	friend class SuperModel;
};
typedef std::vector<FashionPtr> FashionVector;
```

`SuperModel::draw()` 接收一个 `FashionVector*`，依次调 `dress()`，结束后逆序调 `undress()`。三个具体子类：

### 6.1 SuperModelAnimation

[super_model_animation.hpp](file:///workspace/src/lib/model/super_model_animation.hpp) / [.cpp](file:///workspace/src/lib/model/super_model_animation.cpp)。持有 `time`/`lastTime`/`blendRatio` 和变长 `index[]`（每模型一个动画索引）。构造时对每个 topModel 调 `getAnimation(name)`（沿 parent 链查），dress 时对每个 curModel 调 `playAnimation(index[i], time, blendRatio, flags)`。

### 6.2 SuperModelAction（动作 = 动画 + 触发器）

[super_model_action.hpp](file:///workspace/src/lib/model/super_model_action.hpp) / [.cpp](file:///workspace/src/lib/model/super_model_action.cpp)。一个“动作”由 [model_action.hpp](file:///workspace/src/lib/model/model_action.hpp) 的 `ModelAction` 描述：动画名、blendIn/blendOut 时间、track、filler、isMovement/isCoordinated/isImpacting 标志、`MatchInfo`（匹配约束）。

```cpp
// model_action.hpp:31
class ModelAction : public ReferenceCount
{
public:
	ModelAction( DataSectionPtr pSect );
	bool valid( const ::Model & model ) const;
	bool promoteMotion() const;
	std::string	name_;
	std::string	animation_;
	float		blendInTime_;
	float		blendOutTime_;
	int			track_;
	bool		filler_, isMovement_, isCoordinated_, isImpacting_;
	int			flagSum_;
	bool		isMatchable_;
	MatchInfo	matchInfo_;
};
```

`SuperModelAction` 构造时（[super_model_action.cpp:26](file:///workspace/src/lib/model/super_model_action.cpp)）取第一个 topModel 的 action 作为 `pSource_`，并为每个 topModel 解析其动画索引（优先用本模型自己的同名 action 的动画，否则回退到 pSource 的动画名）。`tick()` 推进 `passed_`/`played_`，`dress()` 按 `blendRatio()`（基于 blendIn/Out 时间的淡入淡出曲线）调 `playAnimation`：

```cpp
// super_model_action.cpp:113
float SuperModelAction::blendRatio() const
{
	float fsum = min( (passed_) / pSource_->blendInTime_,
		finish_ > 0.f ? (finish_ - passed_) / pSource_->blendOutTime_ : 1.f );
	return fsum >= 1.f ? 1.f : fsum > 0.f ? fsum : 0.f;
}
```

### 6.3 MatchInfo：动作匹配约束

[match_info.hpp](file:///workspace/src/lib/model/match_info.hpp)。`MatchInfo` 含 `trigger`/`cancel` 两组 `Constraints`，每组约束实体速度、aux1、模型 yaw、Capabilities 开关。`satisfies()` 用于动作匹配系统判断当前实体状态是否应触发/取消该动作：

```cpp
// match_info.hpp:42
bool satisfies( const Capabilities & caps, float speed, float yaw, float aux1 ) const
{
	return caps.match( capsOn, capsOff ) &&
		minEntitySpeed <= speed && speed <= maxEntitySpeed &&
		inRange( yaw, minModelYaw, maxModelYaw ) &&
		inRange( aux1, minEntityAux1, maxEntityAux1 );
}
```

### 6.4 染色（Dye）系统

染色是 model 库最复杂的子系统，分四层：

| 层 | 类 | 文件 | 含义 |
|----|----|------|------|
| 定义 | `Matter` | [matter.hpp](file:///workspace/src/lib/model/matter.hpp) | 一个“可染部位”，如 Pants。含 `name_`、`replaces_`（要替换的 visual 材质 ID）、`tints_`（候选色）、`primitiveGroups_`（命中的图元组） |
| 定义 | `Tint` | [tint.hpp](file:///workspace/src/lib/model/tint.hpp) | 一个具体色，如 Khaki。含 `effectMaterial_`（替换用的 EffectMaterial）、`properties_`（DyeProperty 列表）、`sourceDyes_`（billboard 用） |
| 定义 | `DyeProperty` | [dye_property.hpp](file:///workspace/src/lib/model/dye_property.hpp) | 一个可调材质属性：`index_`（全局表索引）、`controls_`/`mask_`/`future_`、`default_`（Vector4） |
| 索引 | `ModelDye` | [model_dye.hpp](file:///workspace/src/lib/model/model_dye.hpp) | 打包的 `(matterIndex, tintIndex)`，16+16 位，作为 `Model::soak()` 的入参 |
| 应用 | `SuperModelDye` | [super_model_dye.hpp](file:///workspace/src/lib/model/super_model_dye.hpp) | Fashion 子类，dress 时写全局属性表 + 对每个 curModel `soak` |
| 辅助 | `DyeSelection` / `DyePropSetting` | [dye_selection.hpp](file:///workspace/src/lib/model/dye_selection.hpp) / [dye_prop_setting.hpp](file:///workspace/src/lib/model/dye_prop_setting.hpp) | billboard 染色选择与属性设定 |

#### 染色的两阶段：emulsify（设定）+ soak（提交）

`Matter` 用 `emulsion_`/`emulsionCookie_` 缓存“当前选中的 tint”，仅在 `blendCookie` 匹配时生效：

```cpp
// matter.cpp:101
void Matter::emulsify( int tint )
{
	if (uint(tint) >= tints_.size()) tint = 0;
	emulsion_ = tint;
	emulsionCookie_ = Model::blendCookie();  // 标记本帧已设定
}
void Matter::soak()
{
	int tint = (emulsionCookie_ == Model::blendCookie()) ? emulsion_ : 0;  // 否则用 Default
	TintPtr rTint = tints_[ tint ];
	rTint->applyTint();
	for (uint i = 0; i < primitiveGroups_.size(); i++)
		primitiveGroups_[i]->material_ = rTint->effectMaterial_;  // 替换图元组的材质指针
}
```

`SuperModelDye::dress()` [super_model_dye.cpp:65](file:///workspace/src/lib/model/super_model_dye.cpp) 做两件事：先把各 tint 的属性默认值写进全局 `Model::propCatalogueRaw()`（加锁），再对每个 curModel `soak(modelDye)`——后者调 `Matter::emulsify` 设定选中 tint，等 `SuperModel::draw` Step 4 的 `soakDyes()` 真正提交材质指针。这种“两阶段”让多个 part 的属性先汇总到全局表，再统一生效，解决了“邻居模型无法在构造时知道彼此属性”的问题（`supermodelinfo.txt` 详述）。

#### readDyes：从 .model 读染色定义

`Model::readDyes()` [model.cpp:294](file:///workspace/src/lib/model/model.cpp) 遍历 `<dye>` 段：每个 dye 有 `<matter>` 名与 `<replaces>` 材质 ID，调虚函数 `initMatter()`（Nodefull/Nodeless 都委托 `initMatter_NewVisual`，用 `bulk_->gatherMaterials()` 在 visual 里找到对应图元组并保存原材质）。每个 `<tint>` 含 `<material>` 段（调 `initTint()` 加载 `EffectMaterial`）和若干 `<property>`（登记进全局 `s_propCatalogue_`，若 effect 有同名参数则直接绑定 `D3DXHANDLE`）。billboard 分支（`multipleMaterials=false`）走 `sourceDyes_` 路径，不替换材质而是组合纹理因子。

---

## 7. 材质覆盖与静态光照

### 7.1 MaterialOverride

[material_override.hpp](file:///workspace/src/lib/model/material_override.hpp)。允许脚本临时把某 identifier 的所有图元组材质换成指定 `EffectMaterial`，并保存原材质以便恢复：

```cpp
// material_override.hpp:27
class MaterialOverride
{
public:
	std::string identifier_;
	std::vector< Moo::Visual::PrimitiveGroup * > effectiveMaterials_;
	std::vector< Moo::EffectMaterialPtr > savedMaterials_;
	void update( Moo::EffectMaterialPtr newMat );
	void reverse();
};
```

`NodefullModel::overrideMaterial()` / `NodelessModel::overrideMaterial()` 用 `materialOverrides_`（`StringMap<MaterialOverride>`）缓存，首次调用时 `gatherMaterials` 收集图元组。`gatherMaterials()` 直接转发给 `bulk_->gatherMaterials()`。

### 7.2 ModelStaticLighting

[model_static_lighting.hpp](file:///workspace/src/lib/model/model_static_lighting.hpp) 定义抽象接口，`NodelessModel::getStaticLighting()` 返回 `NodelessModelStaticLighting`（[nodeless_model_static_lighting.hpp](file:///workspace/src/lib/model/nodeless_model_static_lighting.hpp)），把预烘焙的 `StaticLightValues`（来自 romp）应用到 visual 顶点。`Model::setStaticLighting()` 调 `pLighting->set()`。

---

## 8. 公共头与协作

### 8.1 forward_declarations.hpp

[forward_declarations.hpp](file:///workspace/src/lib/model/forward_declarations.hpp) 集中声明所有跨文件类与智能指针别名（`ModelPtr`/`SuperModelPtr`/`TintPtr` 等），供 `.ipp` 与外部库使用。

### 8.2 与 moo 的协作

- `Moo::Visual`：模型网格与材质载体，`bulk_->draw()/batch()` 提交渲染。
- `Moo::Node` / `Moo::NodeCatalogue`：全局节点目录，所有 part 的节点按名去重登记，`visitSelf()` 计算世界变换。
- `Moo::Animation` / `Moo::AnimationManager` / `Moo::StreamedDataCache`：动画资源与流式缓存（`.anca`）。
- `Moo::EffectMaterial`：染色 tint 的材质载体，`gatherMaterials`/`replaceProperty` 操作其参数。
- `Moo::MorphVertices::incrementCookie()`：draw 时同步递增 morph 动画 cookie。

### 8.3 与 chunk / romp 的协作

- `ChunkModel`（chunk 库）持有一个 `SuperModel*`，在 chunk draw 时调 `SuperModel::draw()`。
- `PyModel`（客户端）通过 `getAnimation/getAction/getDye` 构造 Fashion 驱动 SuperModel。
- `SuperModelProgress`（romp）包装 SuperModel 加载进度。
- `StaticLightValues` / `LodSettings`（romp）提供静态光照值与 LOD 偏置。

---

## 9. SuperModel 完整加载与绘制流程

### 9.1 ASCII 时序图：从 .model 到 draw

```
调用方                     SuperModel            Model::get()          ModelFactory/Model          moo
  │                            │                      │                       │                       │
  │ new SuperModel(modelIDs)   │                      │                       │                       │
  ├───────────────────────────>│                      │                       │                       │
  │                            │ for each id:         │                       │                       │
  │                            ├─────────────────────>│ Model::get(id)        │                       │
  │                            │                      │ s_pModelMap->find()   │                       │
  │                            │                      │  miss → openSection   │                       │
  │                            │                      │  load() ─────────────>│ createModelFromFile   │
  │                            │                      │                       │  newNodefullModel()   │
  │                            │                      │                       │  new NodefullModel    │
  │                            │                      │                       │   read visual ────────>│ VisualManager::get
  │                            │                      │                       │   build nodeTree ─────>│ NodeCatalogue::findOrAdd
  │                            │                      │                       │   readDyes            │
  │                            │                      │                       │   load .animation ────>│ Animation::load
  │                            │                      │                       │   postLoad (actions)  │
  │                            │                      │  s_pModelMap->add()   │  return ModelPtr      │
  │                            │<─────────────────────┤                       │                       │
  │                            │ models_.push_back    │                       │                       │
  │<───────────────────────────┤                      │                       │                       │
  │                            │                      │                       │                       │
  │ getAnimation/getAction/getDye  (构造 Fashion，存入 FashionVector)         │                       │
  │                            │                      │                       │                       │
  │ sm.draw(&fashions)         │                      │                       │                       │
  ├───────────────────────────>│                      │                       │                       │
  │                            │ Step2: 算 lod_，惰性选 curModel              │                       │
  │                            │ Step3: incrementBlendCookie                  │                       │
  │                            │   早 fashion->dress() (anim/action playAnimation)│                  │
  │                            │   curModel->dress() (NodefullModel: DFS nodeTree, visitSelf) ──────>│ Node worldTransform
  │                            │   晚 fashion->dress() (dye: 写 propCatalogue, soak/emulsify)        │
  │                            │ Step4: curModel->soakDyes() (Matter::soak 替换 material 指针)       │
  │                            │        curModel->draw() (bulk_->draw) ──────────────────────────────>│ Visual::draw
  │                            │ Step5: fashion->undress() 逆序恢复            │                       │
  │<───────── return lod_ ─────┤                      │                       │                       │
```

### 9.2 .model 文件格式

`supermodelinfo.txt` 给出权威格式（节选，标记 `?`=可选、`*`=0或多、`+`=1或多、`|`=或）：

```
<root>
    ?<parent>     MODEL_RESOURCE            </parent>      <!-- LOD 继承父 -->
    ?<extent>     .f                        </extent>      <!-- 可见距离 -->

    [<nodefullVisual>  VISUAL_RESOURCE  </nodefullVisual>   <!-- 二选一决定子类 -->
     |<nodelessVisual> VISUAL_RESOURCE  </nodelessVisual>]

    <! if nodeless !>
        ?<batched>  false  </batched>
    <! endif !>

    *<animation>                              <!-- 动画段 -->
        <! if nodefull !>
            ?<name>        ANIMATION_NAME    </name>
            <nodes>        ANIMATION_RESOURCE</nodes>       <!-- .animation 资源 -->
            ?<firstFrame>  0                 </firstFrame>
            ?<lastFrame>   -1                </lastFrame>
            ?<alpha>       *<NODE_NAME> 0.f  </alpha>
            ?<cognate>     ANIMATION_NAME    </cognate>
        <! else (nodeless) !>
            <name>         ANIMATION_NAME    </name>
            +<visual>      VISUAL_RESOURCE   </visual>      <!-- 逐帧 visual -->
        <! endif !>
        ?<frameRate>    30.f                 </frameRate>
    </animation>

    *<action>                                 <!-- 动作段 -->
        <name>          ACTION_NAME          </name>
        ?<animation>    ANIMATION_NAME       </animation>
        ?<blendInTime>  0.3f                 </blendInTime>
        ?<blendOutTime> 0.3f                 </blendOutTime>
        ?<track>        0                    </track>
        ?<filler>       false                </filler>
        ?<isMovement>   false                </isMovement>
        ?<isCoordinated>false                </isCoordinated>
        ?<isImpacting>  false                </isImpacting>
        ?<match>        (trigger/cancel Constraints, scalePlaybackSpeed, feetFollowDirection, oneShot, promoteMotion) </match>
    </action>

    <! if not billboard !>                    <!-- 染色定义 -->
    *<dye>
        <matter>    MATTER_NAME              </matter>
        <replaces>  VISUAL_MATERIAL_ID       </replaces>    <!-- 要替换的材质标识 -->
        *<tint>
            <name>    TINT_NAME              </name>
            <material> [std material sections] </material>
            *<property>
                <name>     PROPERTY_NAME    </name>
                ?<controls> 0                </controls>
                ?<mask>     0                </mask>
                ?<default>  0.f 0.f 0.f 0.f  </default>
            </property>
        </tint>
    </dye>
</root>
```

要点（出自 `supermodelinfo.txt`）：

- **`<parent>` 构成 LOD 继承链**：子模型的 extent 必须单调递减，否则加载时告警（“could cause the system to get caught in a local minimum”）。无 extent 视为“永远不显示，但其动画仍可用”。
- **动画扁平命名空间**：所有 LOD 的动画同名累加，`getAnimation(name)` 沿 parent 链查找，当前 LOD 没有则回退默认姿势。`<cognate>` 用于 coordinated 动作协调。
- **dye 的 matter/tint/property 三级**：matter 绑定到 visual 的材质 ID（`replaces`），tint 提供替换材质 + 可调属性，property 名进全局表供跨 part 通信。
- **billboard（已禁用）** 走 `sourceDyes_` 路径，纹理名按 `TEXTURE.?[ANIM.FRAME].*TINT.dds` 组合，按需从 source model 生成。

---

## 10. 关键设计小结

1. **静态 Model + 动态 SuperModel**：模型文件加载后只读共享，每实例的状态（动画时间、染色选择）由 SuperModel 持有的 Fashion 承载，内存友好。
2. **blendCookie 帧戳**：让多 part 共享一份全局节点世界变换表与材质状态，dress 阶段写、draw 阶段读，避免重复计算与脏读。
3. **继承链即 LOD 链**：parent 链同时承担动画累积与 LOD 切换，`extent` 是切换阈值，`topModel`（最高 LOD）始终是动画/action 查询入口。
4. **Fashion 早/晚分区**：动画先于模型 dress、染色后于模型 dress，保证染色能覆盖默认材质。
5. **两阶段染色（emulsify + soak）**：dye dress 只设定意图，`soakDyes` 在 draw 前统一提交材质指针，配合全局属性表实现跨 part 属性传递。
6. **变长 SuperModelAnimation/Action**：用 `int index[1]` + 手工 `new char[]` 把“每模型一个索引”紧凑存放，是早期 C++ 的内存优化技巧。

> 待深入：`NodefullModelAnimation` 的 `flagFactor`/`alternateItinerants_`（移动/冲击/协调标志如何驱动节点变换，参见 [nodefull_model_animation.hpp](file:///workspace/src/lib/model/nodefull_model_animation.hpp)）；`ModelCompound`（romp）如何组合多个 SuperModel；`.anca` 流式动画缓存的分块加载细节。
