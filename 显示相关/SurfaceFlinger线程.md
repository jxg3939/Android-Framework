# SurfaceFlinger进程的线程

## 1 进程入口与初始化顺序

SurfaceFlinger的可执行文件是`surfaceflinger`,入口在`main_surfaceflinger.cpp`。所有线程都从`main()`开始派生,先看它:

```cpp
// frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp
int main(int, char**) {
    signal(SIGPIPE, SIG_IGN);

    // HIDL的RPC线程池,1个线程,给旧的HIDL gralloc allocator服务用
    hardware::configureRpcThreadpool(1 /* maxThreads */, false /* callerWillJoin */);

    startGraphicsAllocatorService();

    // When SF is launched in its own process, limit the number of
    // binder threads to 4.
    ProcessState::self()->setThreadPoolMaxThreadCount(4);

    // 让所有线程带上uclamp.min,重点是RenderEngine这类关键线程
    if (SurfaceFlinger::setSchedAttr(true) != NO_ERROR) {
        ALOGW("Failed to set uclamp.min during boot: %s", strerror(errno));
    }

    // 先把当前线程设成SCHED_FIFO优先级1,这样startThreadPool起的binder线程
    // 会继承这个调度策略和优先级;池建好后再把主线程优先级恢复
    int newPriority = 0;
    int origPolicy = sched_getscheduler(0);
    struct sched_param origSchedParam;
    int errorInPriorityModification = sched_getparam(0, &origSchedParam);
    if (errorInPriorityModification == 0) {
        int policy = SCHED_FIFO;
        newPriority = sched_get_priority_min(policy);
        struct sched_param param;
        param.sched_priority = newPriority;
        errorInPriorityModification = sched_setscheduler(0, policy, &param);
    }

    // 启动binder线程池
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // 恢复主线程原调度策略
    if (errorInPriorityModification == 0) {
        errorInPriorityModification = sched_setscheduler(0, origPolicy, &origSchedParam);
    }

    // 实例化SurfaceFlinger
    sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger();

    // SF节点的最小调度策略设为SCHED_FIFO优先级1,任何优先级更低的线程至少跑在这之上
    if (errorInPriorityModification == 0) {
        flinger->setMinSchedulerPolicy(SCHED_FIFO, newPriority);
    }

    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);

    // 客户端能连之前先完成初始化
    flinger->init();

    // 注册两个服务:旧的ISurfaceComposer和新的AIDL接口SurfaceFlingerAIDL
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false, ...);
    sp<SurfaceComposerAIDL> composerAIDL = sp<SurfaceComposerAIDL>::make(flinger);
    ...
    sm->addService(String16("SurfaceFlingerAIDL"), composerAIDL, false, ...);

    startDisplayService();

    // 主线程正式设为SCHED_FIFO
    if (SurfaceFlinger::setSchedFifo(true) != NO_ERROR) {
        ALOGW("Failed to set SCHED_FIFO during boot: %s", strerror(errno));
    }

    // 主线程进入运行循环,不再返回
    flinger->run();
    return 0;
}
```

`flinger->init()`是初始化的核心,它按顺序建起各个子系统:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::init() {
    ...
    // 1. 先建RenderEngine(合成用的GPU引擎),backend和是否threaded由属性决定
    auto builder = renderengine::RenderEngineCreationArgs::Builder()
                           .setPixelFormat(...)
                           .setImageCacheSize(...)
                           .setContextPriority(...);
    chooseRenderEngineType(builder);
    mRenderEngine = renderengine::RenderEngine::create(builder.build());
    mCompositionEngine->setRenderEngine(mRenderEngine.get());

    // 2. 建完RenderEngine再设主线程task profile
    if (!SetTaskProfiles(0, {"SFMainPolicy"})) { ... }

    // 3. 建HWComposer(HWC HAL的封装),并把SF自己设为回调
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(mHwcServiceName));
    auto& composer = mCompositionEngine->getHwComposer();
    composer.setCallback(*this);

    // 4. 建Scheduler(消息队列 + VSYNC调度 + EventThread)
    ...
    initScheduler(display);
    ...
}
```

其中`initScheduler()`是Scheduler及内部线程的创建点,第4节会展开。到这里可以先得出进程启动的先后:先限binder线程数→起binder池→建RenderEngine→建HWComposer→建Scheduler(含EventThread)→最后`run()`进入主循环。

## 2 线程总览

按最新源码,SurfaceFlinger进程里的线程是这些:

| 线程 | 数量 | 创建位置 | 主要功能 |
|---|---|---|---|
| 主线程 | 1 | `main()`本身 | 跑`Scheduler::run()`的Looper循环,响应vsync-sf执行commit与composite |
| binder线程池 | 最多4 | `ProcessState::startThreadPool()` | 处理`ISurfaceComposer`的binder请求(建Layer、提交Transaction等) |
| 两个EventThread | 2 | `Scheduler::createEventThread()` | 分发vsync给app的Choreographer连接("app"和"appSf"两个相位) |
| VSyncDispatch定时器线程 | 1 | `VsyncSchedule`内部 | 在计算出的唤醒时间触发"sf"/"app"/"appSf"三个回调 |
| RenderEngine线程 | 1 | `RenderEngineThreaded`构造 | GPU合成(client合成),仅threaded模式 |
| 其他辅助线程 | 若干 | 各处 | TouchTimer、DisplayPowerTimer、RegionSamplingThread、FpsReporter等 |

上一版里按旧架构写的"sf的EventThread"在最新源码里已经不存在了(改成了VSyncDispatch回调),`HwcAsyncWorker`则换了位置和职责(见第7节),下面逐条说明现状。

## 3 主线程:Looper循环里的commit与composite

### 3.1 进入主循环

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::run() {
    mScheduler->run();
}

// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::run() {
    while (true) {
        waitMessage();
    }
}
```

`Scheduler`继承自`android::impl::MessageQueue`,所以它既管VSYNC调度,又提供主线程的消息循环。`run()`最后落到`waitMessage()`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
int MessageQueue::waitMessage() {
    do {
        // 把当前线程待发送的Binder命令写入binder驱动，避免马上进入pollOnce阻塞
        IPCThreadState::self()->flushCommands();
        int32_t ret = mLooper->pollOnce(-1);
        switch (ret) {
            case Looper::POLL_WAKE:
            case Looper::POLL_CALLBACK:
                continue;
            case Looper::POLL_ERROR:
                ALOGE("Looper::POLL_ERROR");
                continue;
            case Looper::POLL_TIMEOUT:
            default:
                continue;
        }
    } while (true);
}
```

主线程平时阻塞在`mLooper->pollOnce(-1)`上,靠`MessageQueue`里的`Looper`+`Handler`驱动:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
MessageQueue::MessageQueue(ICompositor& compositor, sp<Handler> handler)
      : mCompositor(compositor),
        mLooper(sp<Looper>::make(true)),
        mHandler(std::move(handler)) {}
```

`mCompositor`是一个`ICompositor&`,实现在第3.4节用到。

### 3.2 "sf"的vsync回调如何唤醒主线程

第3.1节里主线程最后停在`waitMessage()`的`mLooper->pollOnce(-1)`,平时一直睡在这里。这一节只回答一个问题:**到底什么东西能让主线程从`pollOnce`里醒过来?**答案是Looper收到一条消息。沿着这条消息往上追,就能把SF主线程的整个唤醒机制看清楚。

先看这条消息到达后发生了什么。主线程醒来后,Looper会调用消息对应的`Handler::handleMessage()`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

`handleMessage`只做两件事:把`mFramePending`这个标志清成false(它的作用马上讲到),然后调`onFrameSignal()`。`onFrameSignal`才是真正干活的地方——commit和composite都在里面,那是第3.3节的内容。这里先别急着进去,继续追下一个问题:这条消息是**谁**发出来的?

发消息的是同一个`Handler`的另一个方法`dispatchFrame()`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::Handler::dispatchFrame(VsyncId vsyncId, TimePoint expectedVsyncTime) {
    if (!mFramePending.exchange(true)) {
        mVsyncId = vsyncId;
        mExpectedVsyncTime = expectedVsyncTime;
        mQueue.mLooper->sendMessage(sp<MessageHandler>::fromExisting(this), Message());
    }
}
```

`dispatchFrame`先做`mFramePending.exchange(true)`,这是"把true写进去、同时返回旧值"的原子操作。如果旧值已经是true,说明上一帧的消息还没被`handleMessage`处理掉,`exchange`返回true、`!true`为假,于是**这次直接丢弃、不再发第二条消息**。这个标志保证了主线程Looper里最多只有一条待处理的帧消息:SF这一帧还没赶完时,新来的vsync-sf不会被越堆越多,而是被合并掉。反过来,如果旧值是false(当前没有pending的帧),就把`vsyncId`和`expectedVsyncTime`存下来,再`sendMessage`叫醒主线程。

那又是谁在调`dispatchFrame`?是`MessageQueue::vsyncCallback()`,也就是这一节标题里那个名叫"sf"的回调:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    // Trace VSYNC-sf
    mVsync.value = (mVsync.value + 1) % 2;
    const auto expectedVsyncTime = TimePoint::fromNs(vsyncTime);
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.lastCallbackTime = expectedVsyncTime;
        mVsync.scheduledFrameTimeOpt.reset();
    }
    const auto vsyncId = VsyncId{mVsync.tokenManager->generateTokenForPredictions(
            {targetWakeupTime, readyTime, vsyncTime})};
    mHandler->dispatchFrame(vsyncId, expectedVsyncTime);
}
```

`vsyncCallback`就做三件事:翻一下trace计数(方便抓trace时看SF醒没醒)、把`vsyncTime`记成`lastCallbackTime`(供下次schedule当基准)、用`TokenManager`给这一帧生成一个`vsyncId`(这个编号会一路带进commit/composite,供后续做帧预测和丢帧判定)。最后调`dispatchFrame`。注意它**只负责发信号,不做任何合成**。

接着往下追:`vsyncCallback`是被谁、在什么时机调用的?它不是SF自己起的某个线程在轮询,而是被**注册**进`VSyncDispatch`(第5节要讲的vsync分发器)的一个普通回调,由第5节的定时器线程到点触发。注册的地方是`MessageQueue::onNewVsyncScheduleLocked()`,名字就写死成"sf":

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    ...
    mVsync.registration = std::make_unique<scheduler::VSyncCallbackRegistration>(
            std::move(dispatch),
            std::bind(&MessageQueue::vsyncCallback, this,
                      std::placeholders::_1, std::placeholders::_2, std::placeholders::_3),
            "sf");
    ...
}
```

这一步发生在SF初始化时的`Scheduler::initVsync()`里,`Scheduler`把从主显示的`VsyncSchedule`里拿到的`VSyncDispatch`传进来,注册才算完成。`VSyncCallbackRegistration`就是注册后拿到的一个句柄,拿着它才能对这个回调做schedule/cancel操作。

最后还有一个容易漏掉的点:注册了**不等于**就会一直被触发。这个回调要先被`schedule()`(预约)一次,定时器才会在算好的时间点去调它。谁来预约?是`MessageQueue::scheduleFrame()`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::scheduleFrame(Duration workDurationSlack) {
    std::lock_guard lock(mVsync.mutex);
    const auto workDuration = Duration(mVsync.workDuration.get() - workDurationSlack);
    mVsync.scheduledFrameTimeOpt =
            mVsync.registration->schedule({.workDuration = workDuration.ns(),
                                           .readyDuration = 0,
                                           .lastVsync = mVsync.lastCallbackTime.ns()});
}
```

`schedule`的含义是"在某个vsync事件之前`workDuration`纳秒叫醒我",`workDuration`就是SF这一帧的合成预算(`sfWorkDuration`),这样SF能赶在目标vsync之前把commit+composite做完。

`scheduleFrame()`在`SurfaceFlinger.cpp`里一共有这么几处调用,归纳起来都是"SF下一帧有活要干":

| 调用点 | 触发场景 |
|---|---|
| `SurfaceFlinger::scheduleCommit()` | 主入口,被`setTransactionFlags`、`scheduleComposite`、`onChoreographerAttached`等调用 |
| `SurfaceFlinger::commit()` 里mode set pending | 显示模式切换还没落地,下一帧再来 |
| `notifyExpectedPresentIfRequired()` | VRR的expected present hint需要跟着帧发 |
| `kernelTimerChanged()` | 内核idle timer状态变了,刷新率悬浮层要重画 |
| `vrrDisplayIdle()` | VRR idle状态变了,刷新率悬浮层要重画 |

其中`scheduleCommit()`是主干,它自己又被这些地方调用:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::scheduleCommit(FrameHint hint, Duration workDurationSlack) {
    if (hint == FrameHint::kActive) {
        mScheduler->resetIdleTimer();
    }
    mPowerAdvisor->notifyDisplayUpdateImminentAndCpuReset();
    mScheduler->scheduleFrame(workDurationSlack);
}
```

往上再走一层,客户端提交事务的路径是`setTransactionState()`末尾的`setTransactionFlags(eTransactionFlushNeeded, ...)`,而`setTransactionFlags`里只在**这一帧还没有被排过**时才排:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::setTransactionFlags(uint32_t mask, TransactionSchedule schedule,
                                         const sp<IBinder>& applyToken, FrameHint frameHint) {
    mScheduler->modulateVsync({}, &VsyncModulator::setTransactionSchedule, schedule, applyToken);
    uint32_t transactionFlags = mTransactionFlags.fetch_or(mask);

    if (const bool scheduled = transactionFlags & mask; !scheduled) {
        mScheduler->resync();
        scheduleCommit(frameHint);      // 这一帧还没排,才去排
    } else if (frameHint == FrameHint::kActive) {
        mScheduler->resetIdleTimer();
    }
}
```

另外`commit()`自己也会补排:显示模式切换未落地、或者HWC反压(backpressure)时,`commit`返回false并调`scheduleCommit`;`composite()`末尾如果`mCompositionEngine->needsAnotherUpdate()`也会补排一次。

现在把整条链正着串一遍:客户端提交事务 → `setTransactionFlags`(`eTransactionFlushNeeded`) → `scheduleCommit` → `scheduleFrame`给"sf"回调预约一次 → 定时器线程到点调用`vsyncCallback` → `dispatchFrame`发消息 → 主线程从`pollOnce`醒来 → `handleMessage` → `onFrameSignal`做commit+composite。反过来看,**没有事务、没有待合成帧的时候,根本没人去schedule这个回调,主线程就安安静静睡在`pollOnce`里**——这就是"sf"回调被称为"按需唤醒"的原因:它不是每60Hz固定醒一次,而是有活干才醒。

还要注意:**这个回调不会自动重排**。`MessageQueue::vsyncCallback`里没有任何重新`schedule`的动作,`VSyncDispatch`那边在触发前会先`executing()`把entry置回disarmed。所以每触发一次,就要有人再调一次`scheduleFrame()`(或`setDuration()`)把它重新武装起来。SF连续出帧时,是每一帧里新来的事务在反复arm它。

### 3.3 onFrameSignal:commit与composite

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
    ...
    // 先算好各display这一帧的FrameTarget(present时间、vsyncId等)
    pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);
    ...
    // commit:应用事务、latch buffer,失败则跳过本帧合成
    if (!compositor.commit(pacesetterPtr->displayId, targets)) {
        ...
        mSchedulerCallback.onCommitNotComposited();
        return;
    }
    ...
    // composite:真正合成并上屏
    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
    ...
}
```

`compositor`就是`SurfaceFlinger`自己(它以`ICompositor`的身份被传给`Scheduler`),所以commit/composite最终落在`SurfaceFlinger`的方法上:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
bool SurfaceFlinger::commit(PhysicalDisplayId pacesetterId,
                            const scheduler::FrameTargets& frameTargets);

CompositeResultsPerDisplay SurfaceFlinger::composite(
        PhysicalDisplayId pacesetterId, const scheduler::FrameTargeters& frameTargeters);
```

- `commit`在主线程串行地应用binder线程攒下的事务、latch各layer的新buffer、处理backpressure(如果上一帧的HWC还没消费完就跳过本帧,`scheduleCommit`重新排下一帧)。
- `composite`走`CompositionEngine::present()`,对每个display:client合成的layer交给`RenderEngine::drawLayers()`(可能在其线程,见第6节),再把所有layer连同client target交给HWC做device合成与上屏(`presentOrValidate`)。

关键结论:主线程是唯一改Layer状态、做合成决策的线程,所有事务应用、buffer latch、合成调度都串行在这个Looper循环里,不存在并发写Layer树的问题。

## 4 binder线程池:客户端请求的入口

### 4.1 初始化

binder池的初始化分两步,都在`main()`里:

```cpp
// frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp
// 第一步:限到4个
ProcessState::self()->setThreadPoolMaxThreadCount(4);
...
// 第二步:起池
sp<ProcessState> ps(ProcessState::self());
ps->startThreadPool();
```

```cpp
// frameworks/native/libs/binder/ProcessState.cpp
#define DEFAULT_MAX_BINDER_THREADS 15

void ProcessState::startThreadPool() {
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);   // 先起1个,之后按需增长
    }
}

status_t ProcessState::setThreadPoolMaxThreadCount(size_t maxThreads) {
    ...
    mMaxThreads = maxThreads;
    ...
}
```

`DEFAULT_MAX_BINDER_THREADS`本来是15,但SF在`main()`里显式`setThreadPoolMaxThreadCount(4)`,所以这个进程的binder线程**最多4个**,`startThreadPool()`先起1个、并发请求增多时按需`spawnPooledThread()`再起,到4为止。另一个细节:`startThreadPool()`之前`main()`把主线程临时设成SCHED_FIFO优先级1,所以binder线程继承了这个策略和优先级。

### 4.2 主要功能

binder线程跑`IPCThreadState::joinThreadPool()`,等客户端事务,转成`SurfaceFlinger`服务端的方法。客户端操作的入口几乎都在这里:

| 客户端操作 | 落在binder线程上的方法 | 作用 |
|---|---|---|
| 创建Layer | `createLayer` | 客户端`SurfaceControl`向SF申请一个Layer |
| 提交事务 | `setTransactionState` | 把一帧的layer属性/buffer变更送进来 |
| 请求Vsync | `createDisplayEventConnection` | 给Choreographer建一条vsync连接 |
| 查询显示配置 | `getDisplayConfigs`/`getActiveDisplayConfig` | 读分辨率、刷新率 |
| 截图 | `captureLayers`/`captureDisplay` | 抓当前合成结果 |

最关键的`setTransactionState`:binder线程收到事务后**不合成**,只加锁放进待处理队列,再通知主线程排一帧;真正应用事务在主线程下一次commit里。binder线程只入队、不碰合成状态,用锁保护队列本身即可。

## 5 Scheduler内部的VSYNC分发线程

这是最复杂的一块。`Scheduler`内部既有VSYNC的生成与调度,又有两个真正向app分发vsync的EventThread线程。

### 5.1 VSYNC的生成与调度

`Scheduler::registerDisplay()`给每个display建一个`VsyncSchedule`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::registerDisplay(...) {
    auto schedulePtr = std::make_shared<VsyncSchedule>(selectorPtr->getActiveMode().modePtr,
                                                        mFeatures,
                                                        [this](PhysicalDisplayId id, bool enable) {
                                                            onHardwareVsyncRequest(id, enable);
                                                        });
    ...
}
```

`VsyncSchedule`内部持有两个关键组件:

- `VSyncTracker`:根据硬件vsync时间戳估算显示器的vsync周期和相位。
- `VSyncDispatch`(实现是`VSyncDispatchTimerQueue`):一个定时器队列,按预测的唤醒时间触发已注册的回调。

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
// 用单个定时器队列分发回调,内部持有一个TimeKeeper定时器
class VSyncDispatchTimerQueue : public VSyncDispatch {
    ...
    CallbackToken registerCallback(Callback, std::string callbackName) final;
    std::optional<ScheduleResult> schedule(CallbackToken, ScheduleTiming) final;
    ...
private:
    void timerCallback();
    void setTimer(nsecs_t, nsecs_t);
    ...
    std::unique_ptr<TimeKeeper> const mTimeKeeper;
    ...
};
```

硬件vsync从HWC回调进来,入口是`SurfaceFlinger::onComposerHalVsync()`(它是`SurfaceFlinger`实现的`HWC2::ComposerCallback`接口方法),再经`Scheduler::addResyncSample()`→`VsyncSchedule`→`VsyncTracker`。tracker更新周期相位后,`VSyncDispatchTimerQueue`算出每个回调("sf"/"app"/"appSf")该在什么时间点唤醒,交给`TimeKeeper`(一个定时器线程)到时触发。所以VSYNC分发有三个消费者,相位各不相同:

| 回调名 | 注册者 | 触发后干什么 | 跑在哪个线程 |
|---|---|---|---|
| `sf` | `MessageQueue` | `dispatchFrame`发消息给主线程 | 定时器线程 |
| `app` | `EventThread`(Cycle::Render) | `onVsync`唤醒app的EventThread | 定时器线程 |
| `appSf` | `EventThread`(Cycle::LastComposite) | `onVsync`唤醒appSf的EventThread | 定时器线程 |

### 5.2 两个EventThread的初始化

`Scheduler::createEventThread()`创建两个EventThread,名字分别是"app"和"appSf":

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::createEventThread(Cycle cycle, frametimeline::TokenManager* tokenManager,
                                  std::chrono::nanoseconds workDuration,
                                  std::chrono::nanoseconds readyDuration) {
    auto eventThread = std::make_unique<android::impl::EventThread>(
            cycle == Cycle::Render ? "app" : "appSf",
            getVsyncSchedule(), tokenManager, *this, workDuration, readyDuration);
    if (cycle == Cycle::Render) {
        mRenderEventThread = std::move(eventThread);
    } else {
        mLastCompositeEventThread = std::move(eventThread);
    }
}
```

创建点在上面的`initScheduler()`里:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
mScheduler->createEventThread(scheduler::Cycle::Render, mFrameTimeline->getTokenManager(),
                              /* workDuration */ configs.late.appWorkDuration,
                              /* readyDuration */ configs.late.sfWorkDuration);
mScheduler->createEventThread(scheduler::Cycle::LastComposite, mFrameTimeline->getTokenManager(),
                              /* workDuration */ activeRefreshRate.getPeriod(),
                              /* readyDuration */ configs.late.sfWorkDuration);
```

两个EventThread的`workDuration`不同:Cycle::Render用`appWorkDuration`(app侧工作量预算),Cycle::LastComposite用整个刷新周期。这决定了它们分发的vsync相位不同——"app"是让app**开始渲染**的vsync(vsync-app),"appSf"是**对齐SF上次合成完成时刻**的vsync。

`EventThread`构造函数里做了三件事:注册vsync回调、起线程、设优先级:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
EventThread::EventThread(const char* name, std::shared_ptr<scheduler::VsyncSchedule> vsyncSchedule,
                         android::frametimeline::TokenManager* tokenManager,
                         IEventThreadCallback& callback, std::chrono::nanoseconds workDuration,
                         std::chrono::nanoseconds readyDuration)
      : mThreadName(name),
        ...
        mVsyncSchedule(std::move(vsyncSchedule)),
        // 1. 向VSyncDispatch注册自己的vsync回调(名字就是"app"/"appSf")
        mVsyncRegistration(mVsyncSchedule->getDispatch(), createDispatchCallback(), name),
        ...
        mCallback(callback) {
    // 2. 起线程,跑threadMain
    mThread = std::thread([this]() {
        std::unique_lock<std::mutex> lock(mMutex);
        threadMain(lock);
    });
    pthread_setname_np(mThread.native_handle(), mThreadName);

    // 3. 设SCHED_FIFO优先级2,减少抖动
    constexpr int EVENT_THREAD_PRIORITY = 2;
    struct sched_param param = {0};
    param.sched_priority = EVENT_THREAD_PRIORITY;
    if (pthread_setschedparam(mThread.native_handle(), SCHED_FIFO, &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO for EventThread");
    }
    set_sched_policy(tid, SP_FOREGROUND);
}
```

这里的关键设计:`EventThread`自己有一个`std::thread`,但它不直接等硬件vsync,而是通过`mVsyncRegistration`向`VSyncDispatch`注册回调。定时器线程在算出的时间点调用这个回调:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
scheduler::VSyncDispatch::Callback EventThread::createDispatchCallback() {
    return [this](nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime) {
        onVsync(vsyncTime, wakeupTime, readyTime);
    };
}

void EventThread::onVsync(nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime) {
    std::lock_guard<std::mutex> lock(mMutex);
    mLastVsyncCallbackTime = TimePoint::fromNs(vsyncTime);
    mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), wakeupTime,
                                       ++mVSyncState->count, vsyncTime, readyTime));
    mCondition.notify_all();   // 唤醒自己的EventThread线程
}
```

`onVsync`在定时器线程上执行,只是把vsync事件塞进`mPendingEvents`并notify,真正分发在EventThread自己的线程里做。

### 5.3 EventThread的threadMain:分发到连接

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;
    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
            ...
        }

        bool vsyncRequested = false;
        // 遍历所有连接,找出该消费这个事件的、以及请求了vsync的
        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) {
                if (event && shouldConsumeEvent(*event, connection)) {
                    consumers.push_back(connection);
                }
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;
                ++it;
            } else {
                it = mDisplayEventConnections.erase(it);
            }
        }

        if (!consumers.empty()) {
            dispatchEvent(*event, consumers);
            consumers.clear();
        }

        if (mVSyncState && vsyncRequested) {
            mState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            mState = State::Idle;
        }

        if (mState == State::VSync) {
            // 有连接在等vsync,再向VSyncDispatch排下一次回调
            const auto scheduleResult = mVsyncRegistration.schedule(...);
            ...
        }
        ...
        mCondition.wait(lock);
    }
}
```

每条连接是一个`EventThreadConnection`,对应某个进程的一个Choreographer。`createDisplayEventConnection()`在连接被创建时把它的`vsyncRequest`置位并notify,threadMain醒来后向`VSyncDispatch`排一次回调;等回调来了,`onVsync`把事件塞进队列、notify,threadMain再把它写进各连接的BitTube。分发走socket pair,不经过binder(这部分细节见`Choreographer如何与SurfaceFlinger建立联系.md`)。

### 5.4 appSf与deprecate_vsync_sf

最新源码里有一个`deprecate_vsync_sf()`的flag,配合`Cycle::LastComposite`的EventThread("appSf"),这是SF的vsync从"独立EventThread"向"VSyncDispatch回调"迁移的结果:原来有个专门的`mSfEventThread`驱动SF主线程,现在这个职责由`MessageQueue`注册的"sf"回调承担;空出来的"sf侧"vsync改成了面向app的"appSf"相位,给需要对齐SF合成时刻的app用(例如按SF合成节奏做帧同步的场景)。

## 6 RenderEngine线程:GPU合成

### 6.1 两种执行模式

RenderEngine负责client合成(把多个layer用GPU画进一块client target)。创建时由backend决定是否多线程:

```cpp
// frameworks/native/libs/renderengine/RenderEngine.cpp
std::unique_ptr<RenderEngine> RenderEngine::create(const RenderEngineCreationArgs& args) {
    threaded::CreateInstanceFactory createInstanceFactory;
    // 根据graphicsApi选GL/VK的Skia实现
    ...
    if (args.threaded == Threaded::YES) {
        return threaded::RenderEngineThreaded::create(std::move(createInstanceFactory));
    }
    return createInstanceFactory();
}
```

- `Threaded::YES` → `RenderEngineThreaded`,有独立线程。
- `Threaded::NO` → `SkiaGLRenderEngine`/`GaneshVkRenderEngine`等,`drawLayers()`直接在主线程同步执行。

是否threaded由`SurfaceFlinger::init()`里的`chooseRenderEngineType(builder)`决定(读`debug.renderengine.backend`等属性)。

### 6.2 RenderEngineThreaded的初始化与threadMain

`RenderEngineThreaded`构造时就起线程:

```cpp
// frameworks/native/libs/renderengine/threaded/RenderEngineThreaded.cpp
RenderEngineThreaded::RenderEngineThreaded(CreateInstanceFactory factory)
      : RenderEngine(Threaded::YES) {
    std::lock_guard lockThread(mThreadMutex);
    mThread = std::thread(&RenderEngineThreaded::threadMain, this, factory);
}

void RenderEngineThreaded::threadMain(CreateInstanceFactory factory) {
    // 设task profile和SCHED_FIFO优先级2
    if (!SetTaskProfiles(0, {"SFRenderEnginePolicy"})) { ... }
    if (setSchedFifo(true) != NO_ERROR) { ... }

    // 关键:真正的Skia RenderEngine在这个线程上创建,GL/VK context绑定到这个线程
    mRenderEngine = factory();

    pthread_setname_np(pthread_self(), mThreadName);
    // 通知构造方初始化完成
    mIsInitialized = true;
    mInitializedCondition.notify_all();

    // 任务循环:从mFunctionCalls取任务并执行,然后等在条件变量上
    while (mRunning) {
        const auto task = getNextTask();
        if (task) {
            (*task)(*mRenderEngine);
        }
        std::unique_lock<std::mutex> lock(mThreadMutex);
        mCondition.wait(lock, [this]() { return !mRunning || !mFunctionCalls.empty(); });
    }
    mRenderEngine.reset();
}
```

主线程的`drawLayers()`等调用在threaded模式下只是把绘制任务push进`mFunctionCalls`、notify一下并返回一个`std::future`,实际GPU绘制由RenderEngine线程执行。这样GPU那部分耗时(提交命令、等GPU)可以和主线程的后续工作重叠,主线程不必同步卡在drawLayers上。

## 7 HWC的present路径与HwcAsyncWorker的变迁

合成链最后一步是把结果交给HWC HAL校验并上屏。最新源码里这一步的调用链是:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Error Display::presentOrValidate(nsecs_t expectedPresentTime, int32_t frameIntervalNs, ...) {
    ...
    return static_cast<Error>(
            mComposer.presentOrValidateDisplay(mId, expectedPresentTime, frameIntervalNs, &numTypes, ...));
}
```

`mComposer`是`Hwc2::Composer`,其实现是`ComposerHal`(文件`DisplayHardware/ComposerHal.cpp`/`.h`),内部再分派到`AidlComposerHal`或`HidlComposerHal`,本质是向composer HAL服务进程发binder调用。

旧版本(Android 12–15)这里有一类线程叫`HwcAsyncWorker`:一个单线程worker,内部是`std::thread`+任务队列+`std::future`,用于把HWC的`presentDisplay`放到独立线程执行、避免阻塞主线程。在最新主分支里它**并没有被移除**,而是换了位置和职责:它现在是`compositionengine::impl::HwcAsyncWorker`,由每个`Output`按需懒创建,不再是一个常驻的全局线程:

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

它的职责从"异步present"变成了**合成策略可预测时把HWC的validate放到实时线程执行、与client合成重叠**:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/HwcAsyncWorker.h
// HWC Validate call may take multiple milliseconds to complete and can account for
// a signification amount of time in the display hotpath. This helper class allows
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

即`Output::present()`里若`canPredictCompositionStrategy()`为真,就`prepareFrameAsync()`,让HWC的validate在HwcAsyncWorker线程上跑,同时按预测的策略先做client合成;预测对了就继续,预测错了就重做client合成。所以"异步调用HWC"这一项在最新源码里对应的是`HwcAsyncWorker`的validate预测卸载,而不是一个常驻的present线程。

## 8 一帧里各线程如何协作

把上面几类线程串起来,一次正常出帧的完整路径:

1. HWC产生硬件vsync,回调进`SurfaceFlinger::onComposerHalVsync()`,经`Scheduler::addResyncSample()`交给`VsyncTracker`更新周期相位。
2. `VSyncDispatchTimerQueue`根据预测的present时间与各消费者的workDuration/readyDuration,算出"sf"/"app"/"appSf"三个回调的唤醒时间,交给`TimeKeeper`定时器线程。
3. 定时器线程按时间触发回调:
   - "app"回调→`EventThread("app")::onVsync`→唤醒它的std::thread→把vsync-app写进各Choreographer连接的BitTube,app的UI线程醒来画一帧。
   - "sf"回调→`MessageQueue::vsyncCallback`→`dispatchFrame`给主线程Looper发消息。
4. app画完经BlastBufferQueue把buffer和事务提交回来。
5. binder线程池收到`setTransactionState`,加锁放进待处理队列,通知主线程。
6. 主线程被"sf"消息唤醒,`onFrameSignal()`里先`commit`(应用事务、latch buffer),再`composite`(client layer交给RenderEngine、device layer交给HWC,`presentOrValidate`上屏)。
7. 上屏后present fence回到SF,buffer随之释放回生产者,下一帧开始。

整个进程里只有主线程改Layer状态、做合成决策;binder线程负责收请求,EventThread负责把vsync按节奏发给app,RenderEngine线程负责执行主线程决定好的GPU合成,定时器线程负责在正确的时间点触发这些唤醒。
