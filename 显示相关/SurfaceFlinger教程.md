# SurfaceFlinger教程

## 1 SurfaceFlinger的定位与职责

SurfaceFlinger是Android的**系统合成器(system compositor)**,跑在独立的native进程`surfaceflinger`里。它不产生任何画面,只负责把别的进程交上来的画面**按正确的层级和位置合成成一块屏幕内容**,再交给硬件上屏。

它的输入输出可以概括成一句话:**输入是buffer+元数据,输出是屏幕上的像素**。

| 输入 | 来源 | 含义 |
|---|---|---|
| buffer | app/WMS(经BlastBufferQueue) | 一帧像素数据,连同acquire fence |
| Transaction | WMS/app(经ISurfaceComposer) | layer的几何、z序、透明度、裁剪、buffer等一组原子变更 |
| 硬件vsync | HWC | 驱动整个合成节奏的时钟信号 |

| 输出 | 去向 | 含义 |
|---|---|---|
| 合成结果 | HWC→屏幕 | client合成画好的client target + device合成layer交给HWC上屏 |
| vsync分发 | 各app的Choreographer | 告诉app什么时候该开始画下一帧 |
| release/present fence | 生产者 | buffer用完了、这一帧真正上屏了的通知 |

SurfaceFlinger在整个图形栈里的位置:

| 组件 | 角色 |
|---|---|
| app/RenderThread | 生产者,画buffer |
| WindowManagerService | 用SurfaceControl发Transaction控制窗口几何与层级 |
| BlastBufferQueue | app侧管理buffer池,把buffer+事务原子地提交给SF |
| SurfaceFlinger | 唯一的合成器,接buffer、应用事务、合成、上屏 |
| HWC/composer HAL | 硬件合成器,做device合成与最终上屏 |
| Choreographer | app侧接收vsync,编排UI帧 |

## 2 进程启动与初始化

启动细节在[`SurfaceFlinger线程模型.md`](SurfaceFlinger线程模型.md)第1节,这里只列结论性的初始化顺序:

1. `configureRpcThreadpool(1)`——给旧的HIDL allocator服务用。
2. `ProcessState::self()->setThreadPoolMaxThreadCount(4)`——binder线程池最多4个。
3. `startThreadPool()`——起binder线程池。
4. `flinger->init()`——按顺序建RenderEngine、HWComposer、Scheduler(含EventThread)。
5. 向ServiceManager注册两个服务:`ISurfaceComposer`(旧接口)和`SurfaceFlingerAIDL`(新AIDL接口)。
6. `flinger->run()`——主线程进入`Scheduler::run()`的消息循环,不再返回。

两个服务注册对应两套客户端入口:旧的`ISurfaceComposer`走C++的`SurfaceComposerClient`,新的AIDL接口走`SurfaceComposerAIDL`。最终都落到`SurfaceFlinger`的同一个方法上,例如`SurfaceComposerAIDL::scheduleCommit()`调`mFlinger->sfdo_scheduleCommit()`。

## 3 核心数据模型

SurfaceFlinger内部的数据模型可以分成**三个层次**:客户端拿到的`SurfaceControl`、服务端的`Layer`、以及每帧交给合成引擎的`LayerSnapshot`快照。

### 3.1 三个层次

| 层次 | 类 | 谁持有 | 生命周期 | 作用 |
|---|---|---|---|---|
| 客户端 | `SurfaceControl` | WMS/app | 客户端强引用 | 描述一个可合成对象,是事务操作的目标 |
| 服务端 | `Layer` | SurfaceFlinger | 随客户端handle存亡 | 保存服务端状态、接buffer、做latch |
| 快照 | `LayerSnapshot` | 每帧重建 | 单帧 | 合成引擎所需的全部几何/内容数据,扁平z序列表 |

`SurfaceControl`与`Layer`的对应关系:客户端`nativeCreate`会跨进程让SF建一个`Layer`,并把Layer的句柄(`LayerHandle`)返回给客户端,`SurfaceControl`持有这个handle。客户端每改一次`SurfaceControl`(setPosition/setBuffer/setZOrder...),最终都是一条发往SF的事务,落在对应`Layer`的状态上。

服务端这层在最新源码里做了一个重要重构:原来分散的`BufferLayer`/`BufferStateLayer`/`ColorLayer`/`EffectLayer`被合并成一个统一的`Layer`类,类型信息放进创建参数里:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
Layer::Layer(const surfaceflinger::LayerCreationArgs& args)
      : sequence(args.sequence),
        mFlinger(sp<SurfaceFlinger>::fromExisting(args.flinger)),
        mName(base::StringPrintf("%s#%d", args.name.c_str(), sequence)),
        mWindowType(static_cast<WindowInfo::Type>(
                args.metadata.getInt32(gui::METADATA_WINDOW_TYPE, 0))) {
    ...
    mDrawingState.crop = {0, 0, -1, -1};
    mDrawingState.acquireFence = sp<Fence>::make(-1);
    ...
    mLayerFEs.emplace_back(frontend::LayerHierarchy::TraversalPath{static_cast<uint32_t>(sequence)},
                           args.flinger->getFactory().createLayerFE(mName, this));
}
```

### 3.2 Layer的构成

一个`Layer`对象大致由三块组成:

| 成员 | 类型 | 作用 |
|---|---|---|
| `mDrawingState` | `State` | 客户端提交的"目标状态":buffer、crop、transform、alpha、z序等 |
| `mBufferInfo` | `BufferInfo` | 当前真正被合成的那块buffer及其几何信息 |
| `mLayerFEs` | `LayerFE`列表 | 合成引擎的前端接口,一个Layer可出现在多个display上,每个display对应一个LayerFE |

`mDrawingState`和`mBufferInfo`的分离是理解buffer接住(latch)机制的关键:`setBuffer`先把新buffer写进`mDrawingState`,等acquire fence信号后`latchBufferImpl`才把它提升(promote)到`mBufferInfo`——也就是"提交"和"真正拿去合成"是两个时间点。

### 3.3 Layer树与z序

所有`Layer`组成一棵树(层级类似场景图scene graph)。子layer继承父layer的部分属性,这样SystemServer、SysUI、app可以在不同层级维护自己的策略而不必理解整棵树。绘制顺序由**中序遍历+相对z值**决定:

```
负z的子layer画在父layer下面;非负z的子layer画在父layer上面;
z相同的按创建顺序,后创建的在上。
```

最新源码在`FrontEnd/readme.md`里给了遍历伪代码:

```
fn traverseBottomToTop(root):
  for each child node including relative children,
    sorted by z then layer id, with z less than 0:
          traverseBottomToTop(childNode)

  visit(root)

  for each child node including relative children,
    sorted by z then layer id, with z greater than or equal to 0:
          traverseBottomToTop(childNode)
```

实际实现里,这棵树被`LayerHierarchyBuilder`建成一张**图(graph)**:mirror(镜像)layer在图中是同一个节点、有多个父节点,从而不用克隆Layer就能实现镜像。z序的扁平化由`LayerSnapshotBuilder`完成,产出一个按z排序的`LayerSnapshot`列表。

### 3.4 DisplayDevice与DisplayInfo

Display这一侧,`DisplayDevice`封装一块物理显示器的输出能力与状态(power mode、是否secure、支持哪些HDR类型、亮度等),`FrontEnd/DisplayInfo.h`则给FrontEnd提供display的几何信息。每个display有一个`DisplayDevice`,合成时SF按display逐个`present`。VSYNC、刷新率选择都以display为单位,其中有一个被选为**pacesetter**(领跑display),决定整个合成节奏。

## 4 事务(Transaction)系统

Transaction是SurfaceFlinger的核心API,一切对layer的改动(几何、层级、buffer)都以事务为单位**原子地**提交。

### 4.1 客户端Transaction与merge

客户端通过`SurfaceComposerClient::Transaction`攒一组对`SurfaceControl`的操作,再`apply()`。事务支持merge,merge满足结合律(怎么分组结果一样)但不满足交换律(顺序有意义):

```cpp
Transaction a; a.setAlpha(sc, 2);
Transaction b; b.setAlpha(sc, 4);
// a.merge(b) 与 b.merge(a) 结果不同
```

`SurfaceComposerClient::Transaction::apply()`最终跨进程调用SF的`setTransactionState`,把攒好的操作一次性发过去。

### 4.2 binder线程入口:setTransactionState

`setTransactionState`跑在binder线程上,是事务进入SF的大门:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
status_t SurfaceFlinger::setTransactionState(
        const FrameTimelineInfo& frameTimelineInfo, Vector<ComposerState>& states,
        Vector<DisplayState>& displays, uint32_t flags, const sp<IBinder>& applyToken,
        InputWindowCommands inputWindowCommands, int64_t desiredPresentTime, bool isAutoTimestamp,
        const std::vector<client_cache_t>& uncacheBuffers, bool hasListenerCallbacks,
        const std::vector<ListenerCallbacks>& listenerCallbacks, uint64_t transactionId,
        const std::vector<uint64_t>& mergedTransactionIds) {
    // 1. 按调用者pid/uid做权限sanitize
    uint32_t permissions = LayerStatePermissions::getTransactionPermissions(originPid, originUid);
    for (auto& composerState : states) {
        composerState.state.sanitize(permissions);
    }
    ...
    // 2. 把ComposerState解析成ResolvedComposerState:
    //    从SurfaceControl handle拿到layerId、parentId,把bufferData转成ExternalTexture
    std::vector<ResolvedComposerState> resolvedStates;
    for (auto& state : states) {
        resolvedStates.emplace_back(std::move(state));
        resolvedState.layerId = LayerHandle::getLayerId(resolvedState.state.surface);
        if (resolvedState.state.hasBufferChanges() && ...) {
            resolvedState.externalTexture =
                    getExternalTextureFromBufferData(*resolvedState.state.bufferData, ...);
        }
        ...
    }
    // 3. 组装成TransactionState
    TransactionState state{frameTimelineInfo, resolvedStates, displays, flags, applyToken,
                           ..., desiredPresentTime, isAutoTimestamp, ..., transactionId, ...};
    // 4. 进lockless队列(不要求在主线程执行)
    mTransactionHandler.queueTransaction(std::move(state));
    // 5. 置eTransactionFlushNeeded标志,排一帧commit
    setTransactionFlags(eTransactionFlushNeeded, schedule, applyToken, frameHint);
    return NO_ERROR;
}
```

关键点:**binder线程在这里只做解析和入队,不做任何合成、不碰Layer状态**。真正应用事务要等主线程下一次commit。

### 4.3 LocklessQueue与ApplyToken

入队用的是`LocklessQueue<TransactionState>`(无锁队列),所以binder线程入队不需要拿SF的状态锁:

```cpp
// frameworks/native/services/surfaceflinger/FrontEnd/TransactionHandler.h
void TransactionHandler::queueTransaction(TransactionState&&);
...
LocklessQueue<TransactionState> mLocklessTransactionQueue;
```

顺序保证依赖**ApplyToken**:每个进程、每个buffer生产者默认各有一个唯一ApplyToken,事务按ApplyToken排队,所以**只有同一ApplyToken下的事务顺序才是有保证的**。不同客户端之间互不阻塞、互不影响。

### 4.4 主线程commit:collect/flush/apply

主线程在commit阶段,由`updateLayerSnapshots()`把攒下的事务一次性消化。这是整条事务流水线的核心:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
bool SurfaceFlinger::updateLayerSnapshots(VsyncId vsyncId, nsecs_t frameTimeNs,
                                          bool flushTransactions, bool& outTransactionsAreEmpty) {
    frontend::Update update;
    if (flushTransactions) {
        // 1. 从lockless队列搬到本地
        mTransactionHandler.collectTransactions();
        // 2. 收下这一帧新建的layer / 销毁的handle(在mCreatedLayersLock下)
        update.newLayers = std::move(mNewLayers);
        update.destroyedHandles = std::move(mDestroyedHandles);
        ...
        // 3. 交给LayerLifecycleManager:注册新layer、应用事务、处理销毁
        mLayerLifecycleManager.addLayers(std::move(update.newLayers));
        update.transactions = mTransactionHandler.flushTransactions();  // 过滤出"就绪"的事务
        mLayerLifecycleManager.applyTransactions(update.transactions);
        mLayerLifecycleManager.onHandlesDestroyed(update.destroyedHandles);
        ...
        // 4. 重建层级图
        mLayerHierarchyBuilder.update(mLayerLifecycleManager);
    }
    ...
    // 5. 应用display级状态
    mustComposite |= applyAndCommitDisplayTransactionStatesLocked(update.transactions);
    // 6. 生成扁平z序快照
    mLayerSnapshotBuilder.update(args);
    // 7. 落到legacy Layer对象上、latch新buffer
    applyTransactionsLocked(update.transactions);
    for (auto& layer : ...) layer->commitTransaction();
    for (auto& layer : ...) layer->latchBufferImpl(unused, latchTime, bgColorOnly);
    ...
    // 8. 收尾
    mLayerLifecycleManager.commitChanges();
    ...
    if (mustComposite) commitTransactions();
    return mustComposite;
}
```

`flushTransactions()`不是把所有事务一股脑应用,而是**过滤出就绪的事务**:fence没信号、present时间没到的会留到下一帧。过滤逻辑由`TransactionHandler`里的`mTransactionReadyFilters`承担,返回`Ready`/`NotReady`/`NotReadyBarrier`/`NotReadyUnsignaled`四种判定。

### 4.5 FrontEnd五阶段流水线

最新源码把事务处理抽象成`FrontEnd`模块,`FrontEnd/readme.md`明确把流水线拆成五段:

| 阶段 | 组件 | 职责 |
|---|---|---|
| 1 排队与过滤 | `TransactionHandler` | 收事务、按就绪条件过滤出可应用的事务 |
| 2 生命周期与状态 | `LayerLifecycleManager` + `RequestedLayerState` | 维护服务端layer状态、处理layer创建销毁、产出change flags |
| 3 建遍历树 | `LayerHierarchyBuilder` | 由RequestedLayerState生成LayerHierarchy图 |
| 4 生成快照 | `LayerSnapshotBuilder` | 由层级图生成扁平z序的LayerSnapshot列表 |
| 5 回拨 | `TransactionCallbackInvoker` | 把OnCommit/OnComplete回调发回客户端 |

其中`RequestedLayerState`是服务端layer状态的简单数据类,事务被merge进这个状态,类似客户端侧事务的merge。状态随时可由`LayerCreationArgs`+事务列表重建。`LayerSnapshot`自包含合成引擎与RenderEngine所需的全部数据,与FrontEnd解耦,可被克隆并自由消费(input、无障碍等WindowInfo监听也从这些快照更新)。

设计意图:状态生成在**热路径**上,所以要可预测、快、稳定——避免锁竞争和上下文切换,只做"把一个像素放到屏幕上"必要的事。

## 5 Buffer管理:从queueBuffer到latch

buffer的队列细节(槽位状态、fence语义)见[`SurfaceFlinger前置基础.md`](SurfaceFlinger前置基础.md),这里讲buffer进入SF之后的路径。

### 5.1 两条提交路径

buffer不是独立提交的,而是**作为事务的一部分**。BlastBufferQueue在app侧acquire完buffer后,把buffer连同acquire fence打包成一条`setBuffer`事务发给SF。所以"提交buffer"和"改layer属性"走的是一条Transaction管道,这也正是BBQ解决画面与窗口位置不同步的关键。

### 5.2 setBuffer:buffer进入drawing state

事务应用时,`Layer::setBuffer()`把buffer写进`mDrawingState`:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
bool Layer::setBuffer(std::shared_ptr<renderengine::ExternalTexture>& buffer,
                      const BufferData& bufferData, ...) {
    ...
    mDrawingState.frameNumber = frameNumber;
    mDrawingState.buffer = std::move(buffer);
    mDrawingState.acquireFence = bufferData.flags.test(BufferData::BufferDataChange::fenceChanged)
            ? bufferData.acquireFence
            : Fence::NO_FENCE;
    ...
    setTransactionFlags(eTransactionNeeded);
    ...
}
```

注意:此时buffer只是**记录在案**,还没真正成为待合成内容。同时记录的acquire fence表示"这块buffer生产者还没画完,要等这个信号"。

### 5.3 latchBufferImpl:真正把buffer设为待合成

每帧commit时,对每个有就绪buffer的layer调用`latchBufferImpl()`:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
bool Layer::latchBufferImpl(bool& recomputeVisibleRegions, nsecs_t latchTime, bool bgColorOnly) {
    ...
    // 1. acquire fence没信号就返回,下帧再试
    if (!fenceHasSignaled()) {
        mFlinger->onLayerUpdate();
        return false;
    }
    // 2. 把drawing state里的buffer提升为当前合成buffer
    updateTexImage(latchTime, bgColorOnly);
    // 3. 抓取crop/transform/dataspace等几何信息
    gatherBufferInfo();
    ...
    // 几何变化时要求重算可见区域
    if ((mBufferInfo.mCrop != oldBufferInfo.mCrop) || ...) {
        recomputeVisibleRegions = true;
    }
    return true;
}
```

`fenceHasSignaled()`是并行化的关键:如果GPU还没画完(acquire fence未信号),SF**跳过这个layer的latch、下帧再试**,而不是阻塞主线程等GPU。`updateTexImage()`才是真正的"acquireBuffer":把buffer从`mDrawingState.buffer`搬到`mBufferInfo`,此后这个buffer就是合成引擎要画的当前内容。

### 5.4 与生产者并行

整个buffer循环的完整闭环:生产者dequeue→画→queue(带acquire fence)→SF经Transaction收到→commit时latch(等acquire fence)→合成上屏→present fence信号→releaseBuffer(带release fence)→生产者下次dequeue时等release fence再写。这个闭环的并行度由槽位数量决定,槽位就是流水线的深度。

## 6 合成管线:commit与composite

主线程每被"sf"vsync唤醒一次,就做一次commit+composite(见[`SurfaceFlinger线程模型.md`](SurfaceFlinger线程模型.md)第3节)。

### 6.1 commit:应用事务与latch

`commit()`主要做:

| 步骤 | 内容 |
|---|---|
| 检查backpressure | 上一帧HWC还没消费完就跳过本帧,`scheduleCommit`排下一帧 |
| 处理mode set | 有pending的显示模式切换时等fence |
| `updateLayerSnapshots` | 第4/5节的整套:flush事务、应用、latch buffer |
| 选刷新率 | `chooseRefreshRateForContent`,决定这帧要不要升/降刷新率 |
| 发callback | `TransactionCallbackInvoker`发OnCommit回调 |

commit阶段结束时,这一帧要合成哪些内容、每块buffer的几何都已确定。

### 6.2 composite:真正合成上屏

`composite()`走`CompositionEngine::present()`,对每个display做一遍。`Output::present()`的主流程:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
ftl::Future<std::monostate> Output::present(
        const compositionengine::CompositionRefreshArgs& refreshArgs) {
    updateColorProfile(refreshArgs);
    updateCompositionState(refreshArgs);
    planComposition();                 // 决定每个layer用client还是device合成
    writeCompositionState(refreshArgs);
    setColorTransform(refreshArgs);
    beginFrame();
    if (canPredictCompositionStrategy(refreshArgs)) {
        result = prepareFrameAsync();  // 预测合成策略,validate与client合成并行
    } else {
        prepareFrame();
    }
    finishFrame(std::move(result));
    ...
}
```

`Output::prepare()`先`rebuildLayerStacks()`(收集可见layer、算覆盖区域),`planComposition()`再调用`mPlanner`给每个可见layer指派合成类型。合成类型落在client合成与device合成两类:

| 合成类型 | 谁画 | 结果 |
|---|---|---|
| CLIENT | RenderEngine(GPU) | 画进一块client target buffer |
| DEVICE | HWC | HWC直接拿buffer做硬件合成 |
| SOLID_COLOR/CURSOR等 | 专用路径 | 不占GPU合成 |

### 6.3 client合成与device合成

client合成的layer由`RenderEngine::drawLayers()`用GPU画进client target(threaded模式下在RenderEngine线程执行,见[`SurfaceFlinger线程模型.md`](SurfaceFlinger线程模型.md)第6节)。最后SF把"client target + 所有device layer"一起交给HWC,由HWC做device合成与上屏。

### 6.4 present fence与buffer回收

HWC校验并上屏后,`present fence`回到SF。这个fence表示"这一帧已经真正显示在屏幕上",SF据此触发OnComplete回调、并最终把buffer连同release fence释放回生产者,开始下一轮。

## 7 CompositionEngine

CompositionEngine是SurfaceFlinger的合成引擎,核心抽象是`Output`、`LayerFE`、`OutputLayer`。

| 类 | 作用 |
|---|---|
| `Output` | 一个display的合成输出:管RenderSurface、管HWC、决定合成策略、present |
| `LayerFE` | layer在合成引擎里的前端接口,携带LayerSnapshot的几何/内容数据 |
| `OutputLayer` | 一个layer在一个Output上的合成状态(可见性、覆盖、合成类型) |

`Output`每个display一个,持有:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/Output.h(概念)
RenderSurface* mRenderSurface;       // client target,GPU合成画到这里
OutputLayer列表;                     // 按z排序的可见layer
Planner mPlanner;                    // 合成策略规划器,决定client/device
```

`collectVisibleLayers()`从后往前遍历layer,增量计算每层的覆盖信息与可见性;`planComposition()`由`mPlanner`根据layer的数量、属性、HWC能力决定哪些layer走client、哪些走device。这个决策还支持**策略预测**:如果预测到合成策略不变,就把HWC的validate放到独立线程、和client合成并行(见第8节)。

## 8 HWC交互(composer3 AIDL)

合成链最后一步是把结果交给HWC HAL校验并上屏:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Error Display::presentOrValidate(nsecs_t expectedPresentTime, int32_t frameIntervalNs, ...) {
    ...
    return static_cast<Error>(
            mComposer.presentOrValidateDisplay(mId, expectedPresentTime, frameIntervalNs, &numTypes, ...));
}
```

`mComposer`是`Hwc2::Composer`,实现是`ComposerHal`,内部再分派到`AidlComposerHal`或`HidlComposerHal`——本质是向composer HAL服务进程发binder调用。最新版本composer HAL已迁移到AIDL composer3,跑在独立进程。

### 8.1 HwcAsyncWorker的现状

这里要修正一个易错点:`HwcAsyncWorker`在最新源码里**并没有被移除**,只是位置和职责变了。它现在是`compositionengine::impl::HwcAsyncWorker`,由每个`Output`按需懒创建:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::updateHwcAsyncWorker() {
    if (mPredictCompositionStrategy || mOffloadPresent) {
        if (!mHwComposerAsyncWorker) {
            mHwComposerAsyncWorker = std::make_unique<HwcAsyncWorker>();
        }
    } else {
        mHwComposerAsyncWorker.reset(nullptr);
    }
}
```

它的职责从"把present放到独立线程"变成了**在可预测合成策略时,把HWC的validate放到实时线程执行**,与client合成重叠:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/HwcAsyncWorker.h
// HWC Validate call may take multiple milliseconds... This helper class allows
// us to run the hwc validate function on a real time thread if we can predict what
// the composition strategy will be and if composition includes client composition.
// While the hwc validate runs, client composition is kicked off with the prediction.
class HwcAsyncWorker final {
    ...
    std::future<bool> send(std::function<bool()>);
    ...
    std::thread mThread;
};
```

也就是:`Output::present()`里若`canPredictCompositionStrategy()`为真,就`prepareFrameAsync()`,让HWC的validate在HwcAsyncWorker的线程上跑,同时主线程(或RenderEngine线程)按预测的策略先做client合成;预测对了就继续,预测错了就重做client合成。它用`std::packaged_task`+`std::future`接收validate结果,是一个短生命周期、按需创建的worker线程,而不是常驻的全局线程。

## 9 一帧的生命周期

### 9.1 硬件vsync的回调入口

HWC HAL产生vsync后,回调进SF的入口是`SurfaceFlinger::onComposerHalVsync()`。它不是随便一个函数,而是`SurfaceFlinger`实现了`HWC2::ComposerCallback`接口的结果:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.h
struct ComposerCallback {
    virtual void onComposerHalHotplugEvent(hal::HWDisplayId, DisplayHotplugEvent) = 0;
    virtual void onComposerHalRefresh(hal::HWDisplayId) = 0;
    virtual void onComposerHalVsync(hal::HWDisplayId, nsecs_t timestamp,
                                    std::optional<hal::VsyncPeriodNanos>) = 0;
    virtual void onComposerHalVsyncPeriodTimingChanged(hal::HWDisplayId,
                                                       const hal::VsyncPeriodChangeTimeline&) = 0;
    virtual void onComposerHalSeamlessPossible(hal::HWDisplayId) = 0;
    virtual void onComposerHalVsyncIdle(hal::HWDisplayId) = 0;
    ...
};
```

注册发生在`init()`里(`composer.setCallback(*this)`,见第2节),即SF把自身注册为HWC的回调对象。注意这些方法名是`onComposerHal*`前缀:旧版本(Android 12/13)这里叫`onVsyncReceived`/`onHotplugReceived`,**在当前源码里这些老名字已经不存在**,按老名字是搜不到的。

实现体把vsync喂给Scheduler:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onComposerHalVsync(hal::HWDisplayId hwcDisplayId, int64_t timestamp,
                                        std::optional<hal::VsyncPeriodNanos> vsyncPeriod) {
    ...
    Mutex::Autolock lock(mStateLock);
    if (const auto displayIdOpt = getHwComposer().onVsync(hwcDisplayId, timestamp)) {
        if (mScheduler->addResyncSample(*displayIdOpt, timestamp, vsyncPeriod)) {
            // period flushed
            mScheduler->modulateVsync(displayIdOpt, &VsyncModulator::onRefreshRateChangeCompleted);
        }
    }
}
```

再往下,Scheduler把样本转给`VsyncSchedule`,最终喂给VSyncTracker:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
bool Scheduler::addResyncSample(PhysicalDisplayId id, nsecs_t timestamp,
                                std::optional<nsecs_t> hwcVsyncPeriodIn) {
    ...
    auto schedule = getVsyncSchedule(id);
    if (!schedule) { ... return false; }
    return schedule->addResyncSample(TimePoint::fromNs(timestamp), hwcVsyncPeriod);
}

// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
bool VsyncSchedule::addResyncSample(TimePoint timestamp, ftl::Optional<Period> hwcVsyncPeriod) {
    bool needsHwVsync = false;
    bool periodFlushed = false;
    {
        std::lock_guard<std::mutex> lock(mHwVsyncLock);
        if (mHwVsyncState == HwVsyncState::Enabled) {
            needsHwVsync = mController->addHwVsyncTimestamp(timestamp.ns(),
                                                            hwcVsyncPeriod.transform(&Period::ns),
                                                            &periodFlushed);
        }
    }
    ...
}
```

两个容易被忽略的点:

- 只有`mHwVsyncState == HwVsyncState::Enabled`时样本才会被送进tracker。**硬件vsync不是一直开着的**,由SF按需开关:`Scheduler::onHardwareVsyncRequest()`→`SurfaceFlinger::requestHardwareVsync()`→`getHwComposer().setVsyncEnabled(displayId, ENABLE/DISABLE)`。tracker把周期预测足够准之后就可以关掉硬件vsync,靠`TimeKeeper`定时器自己维持节奏,省电。
- vsync样本进的是`VsyncSchedule`再经`VsyncController`的`addHwVsyncTimestamp()`到tracker,不是`SurfaceFlinger`直接调tracker。

### 9.2 完整出帧路径

把各环节串起来,一次正常出帧的完整路径:

1. HWC产生硬件vsync,回调`SurfaceFlinger::onComposerHalVsync()`,经`Scheduler::addResyncSample()`→`VsyncSchedule::addResyncSample()`→`addHwVsyncTimestamp()`交给`VSyncTracker`更新周期相位。
2. `VSyncDispatchTimerQueue`算出"app"/"appSf"/"sf"三个回调的唤醒时间,交给`TimeKeeper`定时器线程。
3. 定时器线程触发:
   - "app"回调→`EventThread("app")::onVsync`→唤醒它的线程→把vsync-app写进各Choreographer连接的BitTube,app的UI线程醒来画一帧。
   - "sf"回调→`MessageQueue::vsyncCallback`→`dispatchFrame`给主线程Looper发消息。
4. app画完,经BlastBufferQueue把buffer和事务(`setBuffer`等)提交回来。
5. binder线程池收到`setTransactionState`,解析后`queueTransaction`进lockless队列,`setTransactionFlags(eTransactionFlushNeeded)`排一帧。
6. 主线程被"sf"消息唤醒,`onFrameSignal()`里先`commit`:`updateLayerSnapshots`flush并应用事务→`LayerLifecycleManager`/`LayerHierarchyBuilder`/`LayerSnapshotBuilder`重建快照→对就绪buffer做`latchBufferImpl`(等acquire fence、提升buffer)。
7. 再`composite`:`CompositionEngine::present()`对每个display:planComposition决定client/device→client layer交给RenderEngine画进client target→连同device layer一起交给HWC validate并present上屏。
8. present fence回到SF,buffer连同release fence释放回生产者,下一帧开始。

## 10 总结

- SurfaceFlinger是唯一的系统合成器,输入是buffer+事务,输出是屏幕像素,节奏由硬件vsync驱动。
- 数据模型分三层:`SurfaceControl`(客户端)→`Layer`(服务端)→`LayerSnapshot`(每帧快照),buffer通过"drawing state→mBufferInfo"的两段式latch接住。
- 事务走无锁队列:`setTransactionState`在binder线程只入队,主线程commit时才collect/flush/apply;FrontEnd把这条流水线拆成五段(排队过滤→生命周期→层级图→快照→回调)。
- 合成由CompositionEngine的`Output`驱动,`planComposition`把layer分成client合成(RenderEngine)与device合成(HWC)。
- 最终上屏经composer3 AIDL交给HWC;`HwcAsyncWorker`仍在,职责是合成策略可预测时把HWC的validate放到实时线程与client合成并行。
- 整个进程只有主线程改Layer状态、做合成决策,这是理解SurfaceFlinger一致性和无锁设计的锚点。
