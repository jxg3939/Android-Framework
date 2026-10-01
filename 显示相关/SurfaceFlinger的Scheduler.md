# SurfaceFlinger的Scheduler

Scheduler是SurfaceFlinger里负责"节奏"的子系统:它预测每个显示器的vsync周期与相位,决定SF主线程该在什么时刻做commit/composite,决定每个app的Choreographer该在什么时刻收到vsync信号,还负责根据内容、触摸、idle等信号动态挑选刷新率。本文单独把它拆出来讲透,涉及的文件都位于`frameworks/native/services/surfaceflinger/Scheduler/`目录下(除非另有标注)。

本文与`SurfaceFlinger线程.md`是互补关系:那篇按线程讲,这篇按类讲。线程篇里"Scheduler内部的VSYNC分发线程"一节是本文的摘要版,这里展开讲每个类的接口、成员、调用链与状态机。

---

## 1 Scheduler的定位与类结构

### 1.1 继承关系

Scheduler在最新的AOSP源码里继承了**两个**基类,这是理解它的第一把钥匙:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
class Scheduler : public IEventThreadCallback, android::impl::MessageQueue {
    using Impl = android::impl::MessageQueue;
    ...
};
```

两个基类各管一件事:

| 基类 | 作用 |
|---|---|
| `android::impl::MessageQueue` | 给Scheduler提供一个`Looper`+`Handler`消息循环,以及一个注册到`VSyncDispatch`上的名为`sf`的vsync回调。它让Scheduler的主线程能"睡在消息循环里,被vsync回调唤醒"。 |
| `IEventThreadCallback` | 让Scheduler作为两个EventThread("app"/"appSf")的回调方,回答它们的问题:某个uid要不要节流vsync、某个uid的vsync周期是多少、要不要resync等。 |

`IEventThreadCallback`定义在EventThread.h里,只有四个纯虚方法:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.h
struct IEventThreadCallback {
    virtual bool throttleVsync(TimePoint, uid_t) = 0;            // 该uid的这个vsync要不要被节流掉
    virtual Period getVsyncPeriod(uid_t) = 0;                    // 该uid的vsync周期
    virtual void resync() = 0;                                   // 请求重新对齐到硬件vsync
    virtual void onExpectedPresentTimePosted(TimePoint) = 0;     // 一个期望上屏时间被提交给了app
};
```

### 1.2 Cycle枚举:两种vsync相位

Scheduler同时维护**两类**vsync消费方,用`Cycle`枚举区分:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
enum class Cycle {
    Render,        // 面向app渲染:让app开始画下一帧
    LastComposite // 对齐"SF上次合成完成"时刻,超前显示合成一个刷新周期
};
```

| Cycle | EventThread名字 | workDuration | 语义 |
|---|---|---|---|
| `Render` | `"app"` | `appWorkDuration` | 发给app的vsync,让app在deadline前画完并提交buffer |
| `LastComposite` | `"appSf"` | 整个vsync周期 | 发给"需要对齐SF合成节奏"的app的vsync |

这两个EventThread的存在意义、以及`deprecate_vsync_sf()`这个flag的来龙去脉,在`SurfaceFlinger线程.md`第5节已经讲过,这里不再重复,下文只讲实现细节。

---

## 2 Scheduler的初始化

### 2.1 构造函数

Scheduler由SurfaceFlinger的工厂创建。构造函数只做"成员初始化",不干别的:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
Scheduler::Scheduler(ICompositor& compositor, ISchedulerCallback& callback, FeatureFlags features,
                     surfaceflinger::Factory& factory, Fps activeRefreshRate, TimeStats& timeStats)
      : android::impl::MessageQueue(compositor),
        mFeatures(features),
        mVsyncConfiguration(factory.createVsyncConfiguration(activeRefreshRate)),
        mVsyncModulator(sp<VsyncModulator>::make(mVsyncConfiguration->getCurrentConfigs())),
        mRefreshRateStats(std::make_unique<RefreshRateStats>(timeStats, activeRefreshRate)),
        mSchedulerCallback(callback) {}
```

要点:

1. `android::impl::MessageQueue(compositor)`:先把消息循环和`sf`回调的基础设施建起来(它内部new了一个`Looper`和一个`Handler`),并把`ICompositor`存下来。`ICompositor`是SurfaceFlinger实现的一个接口,`commit()`、`composite()`、`configure()`、`sample()`这些"真正干合成活"的入口都通过它回调给SurfaceFlinger。
2. `mVsyncConfiguration`:一套**相位参数表**,按刷新率给出`appWorkDuration`、`sfWorkDuration`、`hwcMinWorkDuration`等预算值。
3. `mVsyncModulator`:相位调制器,在事务/刷新率切换时临时**平移vsync相位**,避免切帧丢帧。
4. `mRefreshRateStats`、`mSchedulerCallback`:刷新率统计器和回调用SurfaceFlinger的接口。

注意构造函数**没有**建EventThread、没有建VsyncSchedule、没有起定时器。这些要等`initVsync()`、`startTimers()`、`registerDisplay()`逐步完成。

### 2.2 initVsync:接通"sf"回调

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::initVsync(frametimeline::TokenManager& tokenManager,
                          std::chrono::nanoseconds workDuration) {
    Impl::initVsyncInternal(getVsyncSchedule()->getDispatch(), tokenManager, workDuration);
}
```

它把**当前pacesetter(主显示)的`VSyncDispatch`**传给MessageQueue,MessageQueue据此注册名为`"sf"`的回调。`getVsyncSchedule()`不带参数时返回pacesetter的schedule。这里的`workDuration`就是`sfWorkDuration`,表示"SF做commit+composite需要留多少时间预算"。

`initVsyncInternal`的完整实现见下文第4节,它最终调用`onNewVsyncScheduleLocked`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::initVsyncInternal(std::shared_ptr<scheduler::VSyncDispatch> dispatch,
                                     frametimeline::TokenManager& tokenManager,
                                     std::chrono::nanoseconds workDuration) {
    ...
    mVsync.workDuration = workDuration;
    mVsync.tokenManager = &tokenManager;
    oldRegistration = onNewVsyncScheduleLocked(std::move(dispatch));
    ...
}
```

`onNewVsyncScheduleLocked`里`std::bind`的注册名就是字符串`"sf"`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
mVsync.registration = std::make_unique<
        scheduler::VSyncCallbackRegistration>(std::move(dispatch),
                                              std::bind(&MessageQueue::vsyncCallback, this,
                                                        std::placeholders::_1,
                                                        std::placeholders::_2,
                                                        std::placeholders::_3),
                                              "sf");
```

### 2.3 startTimers:起触摸/电源定时器

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::startTimers() {
    if (const int32_t millis = set_touch_timer_ms(defaultTouchTimerValue); millis > 0) {
        mTouchTimer.emplace("TouchTimer", std::chrono::milliseconds(millis),
                            [this] { touchTimerCallback(TimerState::Reset); },
                            [this] { touchTimerCallback(TimerState::Expired); });
        mTouchTimer->start();
    }
    if (const int64_t millis = set_display_power_timer_ms(0); millis > 0) {
        mDisplayPowerTimer.emplace("DisplayPowerTimer", std::chrono::milliseconds(millis),
                                   [this] { displayPowerTimerCallback(TimerState::Reset); },
                                   [this] { displayPowerTimerCallback(TimerState::Expired); });
        mDisplayPowerTimer->start();
    }
}
```

这两个是`OneShotTimer`(一个带超时的单次定时器):触摸/亮屏后的一段时间内,把刷新率策略往"性能"方向拉;超时后回落到"省电/idle"方向。它们通过`touchTimerCallback`、`displayPowerTimerCallback`最终调用`applyPolicy`(见第10节)。

### 2.4 createEventThread:建"app"/"appSf"两个线程

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

SurfaceFlinger侧调用它时传入的duration如下(`SurfaceFlinger线程.md`里也贴过):

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
mScheduler->createEventThread(scheduler::Cycle::Render, mFrameTimeline->getTokenManager(),
                              configs.late.appWorkDuration, configs.late.sfWorkDuration);
mScheduler->createEventThread(scheduler::Cycle::LastComposite, mFrameTimeline->getTokenManager(),
                              activeRefreshRate.getPeriod(), configs.late.sfWorkDuration);
```

### 2.5 registerDisplay:给每个显示建VsyncSchedule

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::registerDisplay(PhysicalDisplayId displayId, RefreshRateSelectorPtr selectorPtr,
                                PhysicalDisplayId activeDisplayId) {
    auto schedulePtr =
            std::make_shared<VsyncSchedule>(selectorPtr->getActiveMode().modePtr, mFeatures,
                                            [this](PhysicalDisplayId id, bool enable) {
                                                onHardwareVsyncRequest(id, enable);
                                            });
    registerDisplayInternal(displayId, std::move(selectorPtr), std::move(schedulePtr),
                            activeDisplayId);
}
```

每个物理显示都配三个东西,塞进一个`Display`结构:一个`RefreshRateSelector`(挑刷新率)、一个`VsyncSchedule`(编排vsync)、一个`FrameTargeter`(算帧目标)。构造`VsyncSchedule`时传的lambda是**硬件vsync开关的回调**:当tracker觉得需要更多硬件vsync样本来校准时会调用它。

`registerDisplayInternal`里,第一个注册的显示会被提升为pacesetter(主显示):

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::registerDisplayInternal(...) {
    const bool isPrimary = !mPacesetterDisplayId;
    ...
    std::scoped_lock lock(mDisplayLock);
    mDisplays.emplace_or_replace(displayId, displayId, std::move(selectorPtr),
                                 std::move(schedulePtr), mFeatures);
    auto pacesetterVsyncSchedule = promotePacesetterDisplayLocked(activeDisplayId, promotionParams);
    ...
    applyNewVsyncSchedule(std::move(pacesetterVsyncSchedule));
    ...
}
```

`applyNewVsyncSchedule`把新的`VSyncDispatch`推给MessageQueue和两个EventThread,让它们把vsync回调从旧schedule迁到新schedule上(见4.5节与9.7节)。

---

## 3 数据结构总览

Scheduler的头文件把成员变量分得很清楚。先看两个核心的内嵌结构:

### 3.1 Display:每屏一个的三件套

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
struct Display {
    Display(PhysicalDisplayId displayId, RefreshRateSelectorPtr selectorPtr,
            VsyncSchedulePtr schedulePtr, FeatureFlags features)
          : displayId(displayId),
            selectorPtr(std::move(selectorPtr)),
            schedulePtr(std::move(schedulePtr)),
            targeterPtr(std::make_unique<FrameTargeter>(displayId, features)) {}

    const PhysicalDisplayId displayId;
    RefreshRateSelectorPtr selectorPtr;   // 挑刷新率
    VsyncSchedulePtr schedulePtr;         // 编排vsync
    FrameTargeterPtr targeterPtr;         // 算帧目标(beginFrame/endFrame)
    hal::PowerMode powerMode = hal::PowerMode::OFF;
};

ui::PhysicalDisplayMap<PhysicalDisplayId, Display> mDisplays;
ftl::Optional<PhysicalDisplayId> mPacesetterDisplayId;
```

所有显示存在`mDisplays`里,`mPacesetterDisplayId`标记谁是**主显示(pacesetter)**。多显示时,pacesetter的vsync节奏是基准,follower显示对齐它的上屏时间(见`onFrameSignal`里对follower的处理)。

### 3.2 Policy:刷新率策略状态

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
struct Policy {
    LayerHistory::Summary contentRequirements;   // 内容检测出的帧率需求
    TimerState idleTimer = TimerState::Reset;
    TouchState touch = TouchState::Inactive;
    TimerState displayPowerTimer = TimerState::Expired;
    hal::PowerMode displayPowerMode = hal::PowerMode::ON;
    ftl::Optional<FrameRateMode> modeOpt;        // 当前选中的显示模式
    std::optional<FrameRateMode> emittedModeOpt; // 最近已下发(emit)的显示模式
} mPolicy GUARDED_BY(mPolicyLock);
```

这堆状态就是"刷新率该选多高"的全部输入:内容在动、有触摸、没idle、刚亮屏 → 刷新率往高拉;反之往低降。第10节讲它如何驱动选择。

### 3.3 成员变量一览

| 成员 | 类型 | 作用 |
|---|---|---|
| `mRenderEventThread` | `unique_ptr<EventThread>` | Cycle::Render的`"app"`线程 |
| `mLastCompositeEventThread` | `unique_ptr<EventThread>` | Cycle::LastComposite的`"appSf"`线程 |
| `mVsyncConfiguration` | `unique_ptr<VsyncConfiguration>` | 按刷新率存储相位/时长参数表 |
| `mVsyncModulator` | `sp<VsyncModulator>` | 事务/切帧时平移vsync相位 |
| `mRefreshRateStats` | `unique_ptr<RefreshRateStats>` | 刷新率统计 |
| `mLayerHistory` | `LayerHistory` | 记录各layer的帧率,内容检测用 |
| `mTouchTimer`/`mDisplayPowerTimer` | `ftl::Optional<OneShotTimer>` | 触摸/亮屏定时器 |
| `mSchedulerCallback` | `ISchedulerCallback&` | 回调SurfaceFlinger(要硬件vsync、切模式等) |
| `mPolicyLock`/`mDisplayLock` | `mutex` | 保护策略/显示数据 |
| `mDisplays` | `PhysicalDisplayMap<...>` | 所有显示的Display结构 |
| `mPacesetterDisplayId` | `Optional<PhysicalDisplayId>` | 主显示id |
| `mAttachedChoreographers` | `unordered_map<int32_t, ...>` | 按layer id记录附着的Choreographer连接 |
| `mFrameRateOverrideMappings` | `FrameRateOverrideMappings` | 按uid记录的帧率override |

---

## 4 MessageQueue:"sf"回调与主线程循环

MessageQueue是Scheduler"主线程节拍器"的实现。它不自己开线程,而是复用**调用`run()`的那个线程**——也就是SurfaceFlinger的主线程。这一节是理解"主线程如何被vsync唤醒"的关键,`SurfaceFlinger线程.md`3.2节讲的是这条链的逆推,这里按正向展开。

### 4.1 成员结构

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.h
namespace impl {
class MessageQueue : public android::MessageQueue {
protected:
    class Handler : public MessageHandler {
        MessageQueue& mQueue;
        std::atomic_bool mFramePending = false;         // 是否已有"待处理的一帧"在排队
        std::atomic<VsyncId> mVsyncId;                  // 这一帧的vsync id
        std::atomic<TimePoint> mExpectedVsyncTime;      // 这一帧期望上屏时间
    public:
        void handleMessage(const Message& message) override;
        virtual bool isFramePending() const;
        virtual void dispatchFrame(VsyncId, TimePoint expectedVsyncTime);
    };

    ICompositor& mCompositor;         // 回调给SurfaceFlinger做commit/composite
    const sp<Looper> mLooper;         // 主线程消息循环
    const sp<Handler> mHandler;       // 处理"到点该合成一帧"的消息

    struct Vsync {
        frametimeline::TokenManager* tokenManager = nullptr;
        mutable std::mutex mutex;
        std::unique_ptr<scheduler::VSyncCallbackRegistration> registration; // "sf"回调注册
        TracedOrdinal<std::chrono::nanoseconds> workDuration;  // sfWorkDuration,可trace
        TimePoint lastCallbackTime;                            // 上次"sf"回调的vsync时间
        std::optional<scheduler::ScheduleResult> scheduledFrameTimeOpt;  // 已排的帧时间
        TracedOrdinal<int> value = {"VSYNC-sf", 0};            // 用于trace的0/1翻转
    };
    Vsync mVsync;
    ...
};
}
```

`mLooper`用`Looper::make(kAllowNonCallbacks)`创建,`kAllowNonCallbacks`说明这个Looper**只处理消息,不跑fd回调**——SurfaceFlinger主线程就是`Looper::pollOnce`+消息队列,不做epoll回调。

### 4.2 waitMessage与run:主线程的循环

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::waitMessage() {
    do {
        IPCThreadState::self()->flushCommands();
        int32_t ret = mLooper->pollOnce(-1);
        switch (ret) {
            case Looper::POLL_WAKE:
            case Looper::POLL_CALLBACK:
                continue;
            ...
        }
    } while (true);
}
```

而Scheduler的`run()`就是无限调用它:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::run() {
    while (true) {
        waitMessage();
    }
}
```

主线程平时就阻塞在`pollOnce(-1)`上(-1表示无限等待),直到有消息(比如"该合成一帧了")把它唤醒。

### 4.3 vsyncCallback:定时器线程上执行

当`VSyncDispatch`的定时器线程判断"该让SF合成下一帧了",它会以三个参数调用`MessageQueue::vsyncCallback`。注意这个函数**跑在定时器线程上**,不是主线程:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    SFTRACE_CALL();
    mVsync.value = (mVsync.value + 1) % 2;          // 翻转trace值,方便看波形

    const auto expectedVsyncTime = TimePoint::fromNs(vsyncTime);
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.lastCallbackTime = expectedVsyncTime;
        mVsync.scheduledFrameTimeOpt.reset();
    }

    // 用tokenManager生成这一帧的vsync id,三个时间戳分别是唤醒时间/就绪时间/vsync时间
    const auto vsyncId = VsyncId{mVsync.tokenManager->generateTokenForPredictions(
            {targetWakeupTime, readyTime, vsyncTime})};

    mHandler->dispatchFrame(vsyncId, expectedVsyncTime);
}
```

三个参数的含义(来自`VSyncDispatch.h`里`Callback`的注释):

| 参数 | 含义 |
|---|---|
| `vsyncTime` | 这个回调对应的vsync的时间戳(即"这一帧的目标上屏vsync") |
| `targetWakeupTime` | 希望客户端被唤醒的时间点 |
| `readyTime` | 希望客户端在这个时间点前完成工作的deadline |

### 4.4 dispatchFrame与handleMessage:把活转交主线程

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

`mFramePending`的`exchange(true)`是关键防抖:如果已经有一个"待合成帧"没被主线程处理掉(返回的旧值是`true`),就不重复发消息;只有旧值是`false`(说明主线程已经把上一帧处理完了)才发消息。这样即使定时器线程连续触发,也最多只有**一帧**排队,不会堆积。

消息被Looper派发后,主线程执行`handleMessage`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

`onFrameSignal`是纯虚函数,由Scheduler实现。到这里,控制流正式从"定时器线程"切回"SF主线程",进入commit/composite。

### 4.5 scheduleFrame与setDuration:排下一帧

`scheduleFrame`是"主动请下一帧"的入口。SurfaceFlinger做完一次commit/composite后,如果还有内容要画,会调用它让`"sf"`回调继续排下一次:

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

`setDuration`则是更新`sfWorkDuration`预算,并立即用新预算重新算唤醒时间:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::setDuration(std::chrono::nanoseconds workDuration) {
    std::lock_guard lock(mVsync.mutex);
    mVsync.workDuration = workDuration;
    mVsync.scheduledFrameTimeOpt =
            mVsync.registration->update({.workDuration = mVsync.workDuration.get().count(),
                                         .readyDuration = 0,
                                         .lastVsync = mVsync.lastCallbackTime.ns()});
}
```

`"sf"`回调用的是`readyDuration = 0`,因为它是**内部**消费方,不需要给SF自己额外预留buffer处理时间;而`"app"`回调则会把`readyDuration`设成`sfWorkDuration`(见`VSyncDispatch.h`里`ScheduleTiming`的注释)。

### 4.6 onNewVsyncScheduleLocked:迁移回调

当pacesetter切换(比如折叠屏内外屏切换、外接显示器),MessageQueue要把它注册的`"sf"`回调从旧的`VSyncDispatch`迁到新的:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    const bool reschedule = mVsync.registration &&
            mVsync.registration->cancel() == scheduler::CancelResult::Cancelled;
    auto oldRegistration = std::move(mVsync.registration);
    mVsync.registration = std::make_unique<scheduler::VSyncCallbackRegistration>(
            std::move(dispatch),
            std::bind(&MessageQueue::vsyncCallback, this, std::placeholders::_1,
                      std::placeholders::_2, std::placeholders::_3),
            "sf");
    if (reschedule) {
        // 原来在等回调,迁移后继续排
        mVsync.scheduledFrameTimeOpt =
                mVsync.registration->schedule({.workDuration = mVsync.workDuration.get().count(),
                                               .readyDuration = 0,
                                               .lastVsync = mVsync.lastCallbackTime.ns()});
    }
    return oldRegistration;
}
```

旧registration被**返回并在锁外析构**,是为了避免死锁:定时器线程可能正在执行`vsyncCallback`,它要先锁`mVsync.mutex`,而析构`VSyncCallbackRegistration`会等待回调执行完(`ensureNotRunning`),若在持锁时析构就会互相等死(这段注释在源码里写得很清楚)。

### 4.7 onFrameSignal:commit与composite

这是Scheduler最核心的方法,主线程每次被`"sf"`回调唤醒后都走这里:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
    const FrameTargeter::BeginFrameArgs beginFrameArgs =
            {.frameBeginTime = SchedulerClock::now(),
             .vsyncId = vsyncId,
             .expectedVsyncTime = expectedVsyncTime,
             .sfWorkDuration = mVsyncModulator->getVsyncConfig().sfWorkDuration,
             .hwcMinWorkDuration = mVsyncConfiguration->getCurrentConfigs().hwcMinWorkDuration,
             .debugPresentTimeDelay = debugPresentDelay};

    ftl::NonNull<const Display*> pacesetterPtr = pacesetterPtrLocked();
    pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);

    {
        FrameTargets targets;
        targets.try_emplace(pacesetterPtr->displayId, &pacesetterPtr->targeterPtr->target());
        expectedVsyncTime = pacesetterPtr->targeterPtr->target().expectedPresentTime();

        // follower显示对齐pacesetter的vsync
        for (const auto& [id, display] : mDisplays) {
            if (id == pacesetterPtr->displayId) continue;
            auto followerBeginFrameArgs = beginFrameArgs;
            followerBeginFrameArgs.expectedVsyncTime =
                    display.schedulePtr->vsyncDeadlineAfter(expectedVsyncTime);
            display.targeterPtr->beginFrame(followerBeginFrameArgs, *display.schedulePtr);
            targets.try_emplace(id, &targeter.target());
        }

        if (!compositor.commit(pacesetterPtr->displayId, targets)) {
            mSchedulerCallback.onCommitNotComposited();
            return;
        }
    }

    FrameTargeters targeters;
    targeters.try_emplace(pacesetterPtr->displayId, pacesetterPtr->targeterPtr.get());
    for (auto& [id, display] : mDisplays) {
        if (id == pacesetterPtr->displayId) continue;
        targeters.try_emplace(id, &display.targeterPtr);
    }

    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
    compositor.sample();
    for (const auto& [id, targeter] : targeters) {
        targeter->endFrame(*resultsPerDisplay.get(id));
    }
}
```

流程分四步:

1. `beginFrame`:让`FrameTargeter`根据vsync时间算出这一帧的`FrameTarget`(期望上屏时间、frame duration等,见第12节)。
2. `commit`:把这一帧要画的layer集合与目标交给SurfaceFlinger做事务提交。返回`false`表示"没东西可合成"(commit未合成),于是回调`onCommitNotComposited`并直接返回,跳过composite。
3. `composite`:真正合成+上屏。返回每个显示的合成结果。
4. `endFrame`:把合成结果喂回`FrameTargeter`,让它学习这一帧的实际耗时,用于后续更准的预测。

`compositor`就是那个`ICompositor&`,它的`commit`/`composite`最终落到SurfaceFlinger的`SurfaceFlinger::commit()`/`composite()`(这两步的细节在`SurfaceFlinger教程.md`第6节讲过)。

---

## 5 VSyncSchedule:每屏一个的vsync编排单元

### 5.1 它把三个类组装起来

`VsyncSchedule`是"每显示一个"的对象,内部持有三个组件,分别负责vsync的**预测**、**校准**、**分发**。它们都是"接口+实现"两件套:

| 组件 | 接口文件 | 实现类 | 职责 |
|---|---|---|---|
| `VSyncTracker` | `VSyncTracker.h` | `VSyncPredictor` | 根据历史vsync时间戳**预测**未来vsync的时刻、周期、相位 |
| `VsyncController` | `VsyncController.h` | `VSyncReactor` | 用present fence和硬件vsync**校准**tracker,决定要不要开硬件vsync |
| `VSyncDispatch` | `VSyncDispatch.h` | `VSyncDispatchTimerQueue` | 一个**定时器队列**,按预测的唤醒时间触发已注册的回调 |

三个组件的具体实现类,由`VsyncSchedule`的三个工厂方法`createTracker`/`createDispatch`/`createController`创建:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
VsyncSchedule::TrackerPtr VsyncSchedule::createTracker(ftl::NonNull<DisplayModePtr> modePtr) {
    constexpr size_t kHistorySize = 20;                 // 保留多少个历史样本
    constexpr size_t kMinSamplesForPrediction = 6;      // 至少多少个样本才能预测
    constexpr uint32_t kDiscardOutlierPercent = 20;     // 丢弃多少比例的离群样本
    return std::make_unique<VSyncPredictor>(std::make_unique<SystemClock>(), modePtr, kHistorySize,
                                            kMinSamplesForPrediction, kDiscardOutlierPercent);
}

VsyncSchedule::DispatchPtr VsyncSchedule::createDispatch(TrackerPtr tracker) {
    constexpr std::chrono::nanoseconds kGroupDispatchWithin = 500us;   // timerSlack
    constexpr std::chrono::nanoseconds kSnapToSameVsyncWithin = 3ms;   // minVsyncDistance
    return std::make_unique<VSyncDispatchTimerQueue>(std::make_unique<Timer>(), std::move(tracker),
                                                     kGroupDispatchWithin.count(),
                                                     kSnapToSameVsyncWithin.count());
}

VsyncSchedule::ControllerPtr VsyncSchedule::createController(PhysicalDisplayId id,
                                                             VsyncTracker& tracker,
                                                             FeatureFlags features) {
    constexpr size_t kMaxPendingFences = 20;
    const bool hasKernelIdleTimer = features.test(Feature::kKernelIdleTimer);
    auto reactor = std::make_unique<VSyncReactor>(id, std::make_unique<SystemClock>(), tracker,
                                                  kMaxPendingFences, hasKernelIdleTimer);
    reactor->setIgnorePresentFences(!features.test(Feature::kPresentFences));
    return reactor;
}
```

这些常量直接决定了Scheduler的行为,值得记住:

| 常量 | 值 | 作用 |
|---|---|---|
| `kHistorySize` | 20 | `VSyncPredictor`保留的历史样本数 |
| `kMinSamplesForPrediction` | 6 | 至少6个样本才开始预测 |
| `kDiscardOutlierPercent` | 20 | 丢弃20%离群样本,抗抖动 |
| `kGroupDispatchWithin` | 500us | `timerSlack`:两个回调唤醒时间差在这个范围内就合并成一次唤醒 |
| `kSnapToSameVsyncWithin` | 3ms | `minVsyncDistance`:两个vsync估计算"同一个vsync"的最小距离 |
| `kMaxPendingFences` | 20 | `VSyncReactor`最多挂多少个未签名的present fence |

`VSyncPredictor`和`VSyncReactor`构造时都用了`SystemClock`(单调时钟封装),`VSyncDispatchTimerQueue`用的定时器是实现`TimeKeeper`接口的`Timer`类。

构造时传入当前显示模式、feature flag、以及硬件vsync开关回调:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp (registerDisplay里)
auto schedulePtr = std::make_shared<VsyncSchedule>(selectorPtr->getActiveMode().modePtr, mFeatures,
                                                   [this](PhysicalDisplayId id, bool enable) {
                                                       onHardwareVsyncRequest(id, enable);
                                                   });
```

`VsyncSchedule`对外的关键方法(从Scheduler.cpp、EventThread.cpp里的调用点归纳):

| 方法 | 作用 |
|---|---|
| `getDispatch()` | 返回`shared_ptr<VSyncDispatch>`,给MessageQueue/EventThread注册回调 |
| `getTracker()` | 返回`VSyncTracker&`,用于预测 |
| `getController()` | 返回`VsyncController&`,用于校准 |
| `period()` | 当前vsync周期 |
| `vsyncDeadlineAfter(TimePoint)` | 给定时刻之后的下一个vsync deadline |
| `addResyncSample(timestamp, hwcVsyncPeriod)` | 喂一个硬件vsync样本 |
| `onDisplayModeChanged(modePtr, force)` | 模式切换时让tracker/controller重新校准 |
| `enableHardwareVsync()` / `disableHardwareVsync(disallow)` | 开关硬件vsync |
| `setPendingHardwareVsyncState(bool)` | 记录"待处理的硬件vsync开关状态" |
| `getPhysicalDisplayId()` | 归属的显示id |

另外两点值得注意:一是`VsyncSchedule`本身继承了公共接口`IVsyncSource`(定义在`scheduler/IVsyncSource.h`),`period()`、`vsyncDeadlineAfter()`、`minFramePeriod()`这三个"查vsync时间线"的方法都是从它override来的,这也是上层(如FrameTargeter)能拿到"某个显示的下一个vsync deadline"的途径;二是头文件里有一对历史遗留的类型别名`using VsyncDispatch = VSyncDispatch; using VsyncTracker = VSyncTracker;`(小写v开头),代码里两种写法是同一个类,注释里标了TODO要删掉别名。

### 5.2 数据流:预测→校准→分发

三个组件的关系是一条流水线:

1. **tracker**吃进硬件vsync时间戳(或present fence时间),维护一个"vsync时间线模型"(周期+相位)。
2. **controller**判断模型是否缺样本、是否需要开硬件vsync来喂更多样本。
3. **dispatch**用tracker的`nextAnticipatedVSyncTimeFrom`算出每个回调该在什么时刻唤醒,交给定时器。

这套"tracker/dispatch"模型是从旧版的`DispSync`演进来的:旧版是`DispSyncThread`+`DispSyncSource`自己算周期,新版拆成了`VSyncTracker`(纯预测)和`VSyncDispatchTimerQueue`(纯调度),职责更清晰。

---

## 6 VSyncTracker:预测vsync周期与相位

`VSyncTracker`是一个**纯接口**,只回答"下一个vsync在什么时候",不负责真的定时。实现类是`VSyncPredictor`(定义在`frameworks/native/services/surfaceflinger/Scheduler/VSyncPredictor.h`),它维护一个历史样本窗口(见5.1节`kHistorySize`/`kMinSamplesForPrediction`/`kDiscardOutlierPercent`),用这些样本拟合出"周期+相位"的线性外推模型。

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncTracker.h
class VSyncTracker {
public:
    // 喂一个vsync时间戳(硬件vsync或present fence),返回是否与模型一致
    virtual bool addVsyncTimestamp(nsecs_t timestamp) = 0;
    // 求"不早于timePoint"的下一个vsync时间
    virtual nsecs_t nextAnticipatedVSyncTimeFrom(nsecs_t timePoint,
                                                 std::optional<nsecs_t> lastVsyncOpt = {}) = 0;
    // 当前vsync周期
    virtual nsecs_t currentPeriod() const = 0;
    // 帧能显示的最小周期(VRR相关)
    virtual Period minFramePeriod() const = 0;
    // 某个时间戳是否与某个帧率同相
    virtual bool isVSyncInPhase(nsecs_t timePoint, Fps frameRate) = 0;
    // 设置显示模式(周期变化时重新校准)
    virtual void setDisplayModePtr(ftl::NonNull<DisplayModePtr>) = 0;
    // 设置render rate(如120Hz屏跑60Hz渲染)
    virtual void setRenderRate(Fps, bool applyImmediately) = 0;
    virtual void onFrameBegin(TimePoint, FrameTime) = 0;
    virtual void onFrameMissed(TimePoint) = 0;
    virtual bool needsMoreSamples() const = 0;
    virtual void resetModel() = 0;
    ...
};
```

几个关键点:

- `addVsyncTimestamp`喂进的样本,既可以是硬件vsync时间戳(通过`VsyncController::addHwVsyncTimestamp`间接调它),也可以是present fence的签名时间(通过`VsyncController::addPresentFence`间接调它)。tracker对两种来源一视同仁,统一用来拟合"周期+相位"。
- `nextAnticipatedVSyncTimeFrom`是**被调用最频繁**的方法:dispatch每排一个回调都要用它算唤醒时间。它保证返回的时间`>= timePoint`。
- `lastVsyncOpt`参数是给VRR(可变刷新率)屏用的:客户端告诉tracker"我上次用的vsync是哪个",tracker据此避免跨过VRR的最小帧周期。
- `setRenderRate`处理"屏是120Hz但内容只跑60Hz"的情况:tracker继续按120Hz追踪硬件vsync时间线,但`nextAnticipatedVSyncTimeFrom`按60Hz的节奏返回。
- `isVSyncInPhase`是frame rate override(帧率锁定)的关键:EventThread分发vsync前用它判断"这个vsync是否落在该app锁定的帧率相位上",不在相位上就节流掉。

---

## 7 VsyncController:用fence与硬件vsync校准

`VsyncController`管"校准"这一面:它决定tracker的样本够不够,不够就申请开硬件vsync。实现类是`VSyncReactor`(定义在`frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.h`)。

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncController.h
class VsyncController {
public:
    // 喂一个present fence,controller把fence时间当vsync信号
    // 这里面会调用到tracker的addVsyncTimestamp来判断是否信号是否对齐，是否需要校准
    virtual bool addPresentFence(std::shared_ptr<FenceTime>) = 0;
    // 喂一个硬件vsync时间戳
    // 这里面同样会调用到tracker的addVsyncTimestamp来判断是否信号是否对齐，同时对软件模型进行校准
    virtual bool addHwVsyncTimestamp(nsecs_t timestamp, std::optional<nsecs_t> hwcVsyncPeriod,
                                     bool* periodFlushed) = 0;
    // 显示模式切换,controller重新校准
    virtual void onDisplayModeChanged(ftl::NonNull<DisplayModePtr>, bool force) = 0;
    // 忽略present fence(某些场景只用硬件vsync)
    virtual void setIgnorePresentFences(bool ignore) = 0;
    virtual void setDisplayPowerMode(hal::PowerMode powerMode) = 0;
    ...
};
```

两个`add*`方法都返回`bool`,含义是"**模型还需要更多vsync信号才能做出准确预测**"。这个返回值直接驱动硬件vsync的开关,看Scheduler里怎么用它:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp

// 在SF完成composite的末尾执行postComposition，在这里面进行Vsync软件模型的信号对齐
void Scheduler::addPresentFence(PhysicalDisplayId id, std::shared_ptr<FenceTime> fence) {
    ...
    const bool needMoreSignals = schedule->getController().addPresentFence(std::move(fence));
    if (needMoreSignals) {
        schedule->enableHardwareVsync();       // 样本不够,开硬件vsync补样本
    } else {
        // 这之后 addHwVsyncTimestamp() （应该是6次），让软件模型重新拟合
        schedule->disableHardwareVsync(false); // 样本够了,可以关硬件vsync省电
    }
}
```

这套"按需开关硬件vsync"的设计是省电的关键:显示器熄灭或长时间idle时,不需要每个硬件vsync都上报,用present fence的签名时间就能维持模型;只有在模型需要重新校准(比如刚亮屏、切模式)时才开硬件vsync。

`addHwVsyncTimestamp`的`periodFlushed`输出参数表示"周期变化是否已完成":当composer报告了新的vsync周期(比如VRR屏切换了周期),controller会把这次周期切换走完。

---

## 8 VSyncDispatch与VSyncDispatchTimerQueue:定时器队列

`VSyncDispatch`是回调分发的**接口**,`VSyncDispatchTimerQueue`是它唯一的实现。这是整个Scheduler里最"机械"也最精细的部分。

### 8.1 VSyncDispatch接口

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatch.h
class VSyncDispatch {
public:
    struct CallbackToken : ftl::DefaultConstructible<CallbackToken, size_t>, ... {};
    using Callback =
            std::function<void(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime)>;

    virtual CallbackToken registerCallback(Callback, std::string callbackName) = 0;
    virtual void unregisterCallback(CallbackToken token) = 0;
    virtual std::optional<ScheduleResult> schedule(CallbackToken, ScheduleTiming) = 0;
    virtual std::optional<ScheduleResult> update(CallbackToken, ScheduleTiming) = 0;
    virtual CancelResult cancel(CallbackToken) = 0;
    virtual void dump(std::string&) const = 0;
};
```

`ScheduleTiming`是"这次调度要什么节奏"的完整描述:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatch.h
struct ScheduleTiming {
    nsecs_t workDuration = 0;                  // 客户端干完活需要的时长
    nsecs_t readyDuration = 0;                 // 客户端要在vsync前多久就绪(deadline)
    nsecs_t lastVsync = 0;                     // 目标显示时间,会snap到它之后最近的vsync
    std::optional<nsecs_t> committedVsyncOpt;  // 已承诺给客户端的vsync时间
    ...
};
```

回调会在**`workDuration + readyDuration`个ns之前**于某个vsync被触发。注释里特别解释了`readyDuration`对内部/外部消费方的区别:

- **内部**消费方(如`"sf"`):`readyDuration = 0`,因为不需要给SF自己额外预留buffer处理时间。
- **外部**消费方(如`"app"`):`readyDuration = sfWorkDuration`,因为app的buffer要经过SF处理,得把SF的处理时间也算进端到端预算,才能给app一个合理的deadline。

### 8.2 VSyncDispatchTimerQueueEntry:每个回调一个entry

每个注册的回调对应一个`VSyncDispatchTimerQueueEntry`,它维护一个三态状态机:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
class VSyncDispatchTimerQueueEntry {
    // 三种状态:disarmed -> armed(被schedule时)
    //          armed -> running -> disarmed(定时器触发时)
    //          armed -> disarmed(被cancel时)
    ...
    ScheduleResult schedule(VSyncDispatch::ScheduleTiming, VSyncTracker&, nsecs_t now);
    void update(VSyncTracker&, nsecs_t now);
    std::optional<nsecs_t> wakeupTime() const;
    std::optional<nsecs_t> readyTime() const;
    std::optional<nsecs_t> targetVsync() const;
    void disarm();
    nsecs_t executing();
    void callback(nsecs_t vsyncTimestamp, nsecs_t wakeupTimestamp, nsecs_t deadlineTimestamp);
    void ensureNotRunning();
private:
    struct ArmingInfo {
        nsecs_t mActualWakeupTime;
        nsecs_t mActualVsyncTime;
        nsecs_t mActualReadyTime;
    };
    ...
};
```

`schedule`的核心逻辑是:用tracker算出下一个vsync时间,再倒推唤醒时间:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
ScheduleResult VSyncDispatchTimerQueueEntry::schedule(VSyncDispatch::ScheduleTiming timing,
                                                      VSyncTracker& tracker, nsecs_t now) {
    auto nextVsyncTime =
            tracker.nextAnticipatedVSyncTimeFrom(std::max(timing.lastVsync,
                                                          now + timing.workDuration +
                                                                  timing.readyDuration),
                                                 timing.committedVsyncOpt.value_or(
                                                         timing.lastVsync));
    auto nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
    ...
    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return ScheduleResult{TimePoint::fromNs(nextWakeupTime), TimePoint::fromNs(nextVsyncTime)};
}
```

三个时间的关系一目了然:`wakeupTime = vsyncTime - workDuration - readyDuration`,`readyTime = vsyncTime - readyDuration`。也就是"提前(干活时长+就绪时长)唤醒,提前(就绪时长)必须干完"。

### 8.3 VSyncDispatchTimerQueue:单一定时器队列

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
class VSyncDispatchTimerQueue : public VSyncDispatch {
    ...
    VSyncDispatchTimerQueue(std::unique_ptr<TimeKeeper>, VsyncSchedule::TrackerPtr,
                            nsecs_t timerSlack, nsecs_t minVsyncDistance);
    ...
private:
    void timerCallback();
    void setTimer(nsecs_t, nsecs_t);
    void rearmTimer(nsecs_t now);
    ...
    std::unique_ptr<TimeKeeper> const mTimeKeeper;   // 实际的定时器
    VsyncSchedule::TrackerPtr mTracker;              // 预测来源
    nsecs_t const mTimerSlack;                       // 相近回调合并成一个唤醒的阈值
    nsecs_t const mMinVsyncDistance;                 // 两个vsync估计算"同一vsync"的最小距离
    CallbackMap mCallbacks;
    nsecs_t mIntendedWakeupTime = kInvalidTime;
};
```

关键设计:

1. **单一定时器**:整个队列只有一个`TimeKeeper`定时器,永远只定"下一个最近的唤醒时间"。多个回调的唤醒时间接近时(`< mTimerSlack`)会合并成一次唤醒。
2. **`rearmTimer`**:遍历所有armed的entry,取最小的`wakeupTime`,若它早于当前`mIntendedWakeupTime`就`setTimer`重设定时器。
3. **`timerCallback`**:定时器到点后,收集所有"该醒了"的entry,在**锁外**逐个调用它们的`callback`(即"sf"/"app"/"appSf"的回调),然后`rearmTimer`排下一次。

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
void VSyncDispatchTimerQueue::timerCallback() {
    std::vector<Invocation> invocations;
    {
        std::lock_guard lock(mMutex);
        auto const now = mTimeKeeper->now();
        for (auto it = mCallbacks.begin(); it != mCallbacks.end(); it++) {
            auto& callback = it->second;
            auto const wakeupTime = callback->wakeupTime();
            if (!wakeupTime) continue;
            auto const readyTime = callback->readyTime();
            auto const lagAllowance = std::max(now - mIntendedWakeupTime, 0ns);
            if (*wakeupTime < mIntendedWakeupTime + mTimerSlack + lagAllowance) {
                callback->executing();
                invocations.emplace_back(Invocation{callback, *callback->lastExecutedVsyncTarget(),
                                                    *wakeupTime, *readyTime});
            }
        }
        mIntendedWakeupTime = kInvalidTime;
        rearmTimer(mTimeKeeper->now());
    }
    for (auto const& invocation : invocations) {
        invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                                      invocation.deadlineTimestamp);
    }
}
```

注意`lagAllowance`:如果定时器线程醒晚了(now已经超过预期唤醒时间),仍把"本应现在醒"的回调一起触发,避免因为抖动丢回调。

### 8.4 TimeKeeper:真正的定时器线程

`TimeKeeper`是一个抽象,`VSyncDispatchTimerQueue`通过四个方法使用它。它是Scheduler的内部公共头文件,`VSyncDispatchTimerQueue.cpp`里以`#include <scheduler/TimeKeeper.h>`引入,位于`frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/TimeKeeper.h`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/TimeKeeper.h
class Clock {
public:
    virtual ~Clock();
    // 返回SYSTEM_TIME_MONOTONIC,便于测试打桩
    virtual nsecs_t now() const = 0;
};

/*
 * TimeKeeper is the interface for a single-shot timer primitive.
 */
class TimeKeeper : public Clock {
public:
    virtual ~TimeKeeper();
    // 在time时刻触发callback。只有一个定时器,再次调用会重置回调和时间
    virtual void alarmAt(std::function<void()>, nsecs_t time) = 0;
    // 取消一个已挂起的回调
    virtual void alarmCancel() = 0;
    virtual void dump(std::string&) const = 0;
};
```

注意`now()`来自基类`Clock`,而`VSyncPredictor`/`VSyncReactor`构造时用的`SystemClock`也是这个`Clock`体系的实现——"读时间"这件事在整个Scheduler里是统一抽象、可打桩的。

对应`VSyncDispatchTimerQueue`里的调用:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
void VSyncDispatchTimerQueue::cancelTimer() {
    mIntendedWakeupTime = kInvalidTime;
    mTimeKeeper->alarmCancel();
}
void VSyncDispatchTimerQueue::setTimer(nsecs_t targetTime, nsecs_t) {
    mIntendedWakeupTime = targetTime;
    mTimeKeeper->alarmAt(std::bind(&VSyncDispatchTimerQueue::timerCallback, this),
                         mIntendedWakeupTime);
    mLastTimerSchedule = mTimeKeeper->now();
}
```

`TimeKeeper`的实现类是`Timer`(定义在`frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/Timer.h`,这也是5.1节`createDispatch`里`std::make_unique<Timer>()`的类型)。它内部确实有一个**自己的定时器线程**:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/Timer.h
class Timer : public TimeKeeper {
public:
    nsecs_t now() const final;
    void alarmAt(std::function<void()>, nsecs_t time) final;   // 线程安全
    void alarmCancel() final;                                  // 线程安全
    void dump(std::string&) const final;
private:
    int mTimerFd = -1;                    // timerfd
    std::array<int, 2> mPipes = {-1, -1}; // 用于唤醒epoll的自管道
    std::thread mDispatchThread;          // 定时器线程
    void threadMain();
    bool dispatch();
    std::function<void()> mCallback GUARDED_BY(mMutex);
};
```

实现方式是`timerfd`+`epoll`+一根自管道,由`mDispatchThread`这条线程跑`threadMain`等待超时。这就是`SurfaceFlinger线程.md`第5节里说的"定时器线程",到点后它执行注册进来的回调(`VSyncDispatchTimerQueue::timerCallback`)。它和EventThread、SF主线程是三类不同的线程。

`Timer`只提供**单次**定时能力(`single-shot timer primitive`),重复的节奏由`VSyncDispatchTimerQueue`自己在每次`timerCallback`末尾调`rearmTimer`重新设下一次来实现。

### 8.5 VSyncCallbackRegistration:RAII的注册句柄

`VSyncCallbackRegistration`把"注册+取消"包成一个RAII对象,持有它即代表"这个回调还活着",析构时自动`unregisterCallback`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatch.h
class VSyncCallbackRegistration {
public:
    VSyncCallbackRegistration(std::shared_ptr<VSyncDispatch>, VSyncDispatch::Callback,
                              std::string callbackName);
    ~VSyncCallbackRegistration();                 // 析构时自动unregister
    std::optional<ScheduleResult> schedule(VSyncDispatch::ScheduleTiming);
    std::optional<ScheduleResult> update(VSyncDispatch::ScheduleTiming);
    CancelResult cancel();
private:
    std::shared_ptr<VSyncDispatch> mDispatch;
    std::optional<VSyncDispatch::CallbackToken> mToken;
};
```

MessageQueue的`mVsync.registration`、EventThread的`mVsyncRegistration`都是这个类型。这就是为什么`onNewVsyncScheduleLocked`要"把旧registration返回并在锁外析构"——析构会触发`unregisterCallback`,进而`ensureNotRunning`等待可能正在执行的定时器回调。

---

## 9 EventThread:"app"和"appSf"

EventThread是把vsync**分发给app**的线程。`SurfaceFlinger线程.md`5.2/5.3节讲了它的创建和threadMain,这里补上状态机和细节。

### 9.1 VSyncRequest状态机

每个app的连接(一个Choreographer)有一个`vsyncRequest`,决定它要什么节奏的vsync:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.h
enum class VSyncRequest {
    None = -2,                 // 不要vsync
    Single = -1,               // 只要一次vsync(且要回调)
    SingleSuppressCallback = 0,// 只要一次,但这次不发回调
    Periodic = 1,              // 周期要(后续值即周期数)
};
```

app侧通过`Choreographer`调用`requestNextVsync`(对应`Single`),或`setVsyncRate`(对应`Periodic`/`None`)。`EventThreadConnection`是binder侧的连接对象:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
binder::Status EventThreadConnection::setVsyncRate(int rate) {
    mEventThread->setVsyncRate(static_cast<uint32_t>(rate),
                               sp<EventThreadConnection>::fromExisting(this));
    return binder::Status::ok();
}
binder::Status EventThreadConnection::requestNextVsync() {
    mEventThread->requestNextVsync(sp<EventThreadConnection>::fromExisting(this));
    return binder::Status::ok();
}
```

`requestNextVsync`会先`mCallback.resync()`(Scheduler::resync,见第13节)再置位:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::requestNextVsync(const sp<EventThreadConnection>& connection) {
    mCallback.resync();
    std::lock_guard<std::mutex> lock(mMutex);
    if (connection->vsyncRequest == VSyncRequest::None) {
        connection->vsyncRequest = VSyncRequest::Single;
        mCondition.notify_all();
    } else if (connection->vsyncRequest == VSyncRequest::SingleSuppressCallback) {
        connection->vsyncRequest = VSyncRequest::Single;
    }
}
```

### 9.2 构造函数:注册回调+起线程+设优先级

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
EventThread::EventThread(const char* name, std::shared_ptr<scheduler::VsyncSchedule> vsyncSchedule,
                         android::frametimeline::TokenManager* tokenManager,
                         IEventThreadCallback& callback, std::chrono::nanoseconds workDuration,
                         std::chrono::nanoseconds readyDuration)
      : mThreadName(name),
        mVsyncTracer(...),
        mWorkDuration(...),
        mReadyDuration(readyDuration),
        mVsyncSchedule(std::move(vsyncSchedule)),
        mVsyncRegistration(mVsyncSchedule->getDispatch(), createDispatchCallback(), name),
        mTokenManager(tokenManager),
        mCallback(callback) {
    mThread = std::thread([this]() {
        std::unique_lock<std::mutex> lock(mMutex);
        threadMain(lock);
    });
    pthread_setname_np(mThread.native_handle(), mThreadName);
    // SCHED_FIFO优先级2,减少抖动
    constexpr int EVENT_THREAD_PRIORITY = 2;
    struct sched_param param = {0};
    param.sched_priority = EVENT_THREAD_PRIORITY;
    pthread_setschedparam(mThread.native_handle(), SCHED_FIFO, &param);
    set_sched_policy(tid, SP_FOREGROUND);
}
```

三件事:向`VSyncDispatch`注册名为`name`(`"app"`或`"appSf"`)的回调;起自己的`std::thread`跑`threadMain`;设成SCHED_FIFO优先级2减少抖动。

### 9.3 onVsync:定时器线程塞事件

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
    mCondition.notify_all();
}
```

`onVsync`在**定时器线程**上执行,只是把vsync事件塞进`mPendingEvents`并notify,真正的分发在EventThread自己的线程里做。

### 9.4 threadMain:分发主循环

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;
    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
            // hotplug事件会初始化/清空mVSyncState
            ...
        }

        bool vsyncRequested = false;
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

        // 决定下一次的状态:有连接要vsync→VSync,否则Idle
        if (mVSyncState && vsyncRequested) {
            mState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            mState = State::Idle;
        }

        if (mState == State::VSync) {
            // 还有连接在等,继续向VSyncDispatch排下一次回调
            mVsyncRegistration.schedule({.workDuration = mWorkDuration.get().count(),
                                         .readyDuration = mReadyDuration.count(),
                                         .lastVsync = mLastVsyncCallbackTime.ns(),
                                         .committedVsyncOpt = mLastCommittedVsyncTime.ns()});
        } else {
            mVsyncRegistration.cancel();  // 没人要了,取消回调省资源
        }

        if (!mPendingEvents.empty()) continue;

        // 等事件或等连接注册/请求
        if (mState == State::Idle) {
            mCondition.wait(lock);
        } else {
            // 超时兜底:VSync状态等1s,合成vsync等16ms,超时就伪造一个vsync
            const std::chrono::nanoseconds timeout =
                    mState == State::SyntheticVSync ? 16ms : 1000ms;
            if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
                ...
                mPendingEvents.push_back(makeVSync(...));  // 伪造vsync,防驱动卡死
            }
        }
    }
    mVsyncRegistration.cancel();
}
```

几个要点:

- **按需排回调**:只有`vsyncRequested`(有连接要vsync)才`schedule`下一次;没人要就`cancel`,省掉定时器开销。
- **兜底伪造**:若驱动卡死、1s内没来硬件vsync,EventThread会自己伪造一个vsync事件,避免app永远等不到。
- **合成vsync**:当显示器熄灭,`mVSyncState->synthetic = true`,EventThread进入`SyntheticVSync`状态,以16ms(约60Hz)的节奏喂app假vsync,让app继续跑(见`enableSyntheticVsync`)。

### 9.5 shouldConsumeEvent:谁该收这个事件

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
bool EventThread::shouldConsumeEvent(const DisplayEventReceiver::Event& event,
                                     const sp<EventThreadConnection>& connection) const {
    const auto throttleVsync = [&]() {
        if (connection->frameRate.isValid()) {
            return !mVsyncSchedule->getTracker()
                            .isVSyncInPhase(vsyncData.preferredExpectedPresentationTime(),
                                            connection->frameRate);
        }
        return mCallback.throttleVsync(expectedPresentTime, connection->mOwnerUid);
    };

    switch (event.header.type) {
        case DISPLAY_EVENT_VSYNC:
            switch (connection->vsyncRequest) {
                case VSyncRequest::None:
                    return false;
                case VSyncRequest::SingleSuppressCallback:
                    connection->vsyncRequest = VSyncRequest::None;
                    return false;
                case VSyncRequest::Single:
                    if (throttleVsync()) return false;
                    connection->vsyncRequest = VSyncRequest::SingleSuppressCallback;
                    return true;
                case VSyncRequest::Periodic:
                    if (throttleVsync()) return false;
                    return true;
                default:
                    return event.vsync.count % vsyncPeriod(connection->vsyncRequest) == 0;
            }
        ...
    }
}
```

节流逻辑:`throttleVsync`最终调用Scheduler::throttleVsync(即`IEventThreadCallback`),它用`getFrameRateOverride(uid)`查该app有没有锁帧率,有就用`isVSyncInPhase`判断该vsync是否落在锁定的帧率相位上,不在就丢弃。

### 9.6 dispatchEvent与generateFrameTimeline

真正写管道时,每个consumer拿到一份**独立**的vsync数据,包含按它自己的帧率算出的frame timeline:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        DisplayEventReceiver::Event copy = event;
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
            const Period frameInterval = mCallback.getVsyncPeriod(consumer->mOwnerUid);
            copy.vsync.vsyncData.frameInterval = frameInterval.ns();
            generateFrameTimeline(copy.vsync.vsyncData, frameInterval.ns(), copy.header.timestamp,
                                  ...);
        }
        ...
        consumer->postEvent(copy);  // 写进BitTube(socket pair)
    }
}
```

`generateFrameTimeline`生成多个候选帧时间线(deadline+expectedPresentationTime的组合),并给每个生成一个vsync id:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
int64_t EventThread::generateToken(nsecs_t timestamp, nsecs_t deadlineTimestamp,
                                   nsecs_t expectedPresentationTime) const {
    if (mTokenManager != nullptr) {
        return mTokenManager->generateTokenForPredictions(
                {timestamp, deadlineTimestamp, expectedPresentationTime});
    }
    return FrameTimelineInfo::INVALID_VSYNC_ID;
}
```

vsync id由`frametimeline::TokenManager`生成,SF和app两侧都用同一个id追踪这一帧,这就是FrameTimeline机制(帧级追踪)的基础。

### 9.7 onNewVsyncSchedule:迁移回调

pacesetter切换时,EventThread也要把回调迁到新schedule,逻辑和MessageQueue的`onNewVsyncScheduleLocked`对称:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
scheduler::VSyncCallbackRegistration EventThread::onNewVsyncScheduleInternal(
        std::shared_ptr<scheduler::VsyncSchedule> schedule) {
    std::lock_guard<std::mutex> lock(mMutex);
    const bool reschedule = mVsyncRegistration.cancel() == scheduler::CancelResult::Cancelled;
    mVsyncSchedule = std::move(schedule);
    auto oldRegistration = std::exchange(mVsyncRegistration,
                                         scheduler::VSyncCallbackRegistration(
                                                 mVsyncSchedule->getDispatch(),
                                                 createDispatchCallback(), mThreadName));
    if (reschedule) {
        mVsyncRegistration.schedule({.workDuration = mWorkDuration.get().count(),
                                     .readyDuration = mReadyDuration.count(),
                                     .lastVsync = mLastVsyncCallbackTime.ns(),
                                     .committedVsyncOpt = mLastCommittedVsyncTime.ns()});
    }
    return oldRegistration;
}
```

---

## 10 刷新率选择:RefreshRateSelector与Policy

Scheduler的另一个大职责是**动态挑刷新率**。它不自己算,而是把一堆"信号"汇总成`GlobalSignals`,交给`RefreshRateSelector`排序。

### 10.1 策略状态与信号

第3.2节的`Policy`结构存的是策略状态。这些状态通过`applyPolicy`驱动选择:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
template <typename S, typename T>
auto Scheduler::applyPolicy(S Policy::*statePtr, T&& newState) -> GlobalSignals {
    std::vector<display::DisplayModeRequest> modeRequests;
    GlobalSignals consideredSignals;
    bool refreshRateChanged = false;

    {
        std::scoped_lock lock(mPolicyLock);
        auto& currentState = mPolicy.*statePtr;
        if (currentState == newState) return {};
        currentState = std::forward<T>(newState);

        DisplayModeChoiceMap modeChoices;
        ftl::Optional<FrameRateMode> modeOpt;
        {
            std::scoped_lock lock(mDisplayLock);
            modeChoices = chooseDisplayModes();
            ...
            std::tie(modeOpt, consideredSignals) =
                    modeChoices.get(*mPacesetterDisplayId).transform(...).value();
        }
        ...
        if (mPolicy.modeOpt != modeOpt) {
            mPolicy.modeOpt = modeOpt;
            refreshRateChanged = true;
        }
    }
    if (refreshRateChanged) {
        mSchedulerCallback.requestDisplayModes(std::move(modeRequests));
    }
    ...
    return consideredSignals;
}
```

它是个**模板方法**,`statePtr`指向`Policy`里某个字段。四种输入都会触发它:

| 触发点 | statePtr | 含义 |
|---|---|---|
| `chooseRefreshRateForContent` | `contentRequirements` | 内容检测(视频帧率等)出的需求 |
| `idleTimerCallback` | `idleTimer` | idle定时器到点/复位 |
| `touchTimerCallback` | `touch` | 触摸/离开触摸 |
| `displayPowerTimerCallback` | `displayPowerTimer` | 亮屏定时器 |

### 10.2 chooseDisplayModes:交给RefreshRateSelector

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
auto Scheduler::chooseDisplayModes() const -> DisplayModeChoiceMap {
    DisplayModeChoiceMap modeChoices;
    const auto globalSignals = makeGlobalSignals();

    const Fps pacesetterFps = [&]() {
        auto rankedFrameRates =
                pacesetterSelectorPtrLocked()->getRankedFrameRates(mPolicy.contentRequirements,
                                                                   globalSignals);
        const Fps pacesetterFps = rankedFrameRates.ranking.front().frameRateMode.fps;
        modeChoices.try_emplace(*mPacesetterDisplayId,
                                DisplayModeChoice::from(std::move(rankedFrameRates)));
        return pacesetterFps;
    }();

    // 给开机状态的follower显示也选一遍
    for (const auto& [id, display] : mDisplays) {
        if (id == *mPacesetterDisplayId) continue;
        if (display.powerMode != hal::PowerMode::ON) continue;
        auto rankedFrameRates =
                display.selectorPtr->getRankedFrameRates(mPolicy.contentRequirements, globalSignals,
                                                         pacesetterFps);
        modeChoices.try_emplace(id, DisplayModeChoice::from(std::move(rankedFrameRates)));
    }
    return modeChoices;
}
```

`getRankedFrameRates`是`RefreshRateSelector`的核心:它拿到内容需求+全局信号,返回一组**排序好的**候选帧率。排序考虑的因素包括:内容是否需要高帧率、触摸/亮屏是否要求性能、idle是否允许降帧率、省电、最低帧率约束等。最终`ranking.front()`就是最优选。

`makeGlobalSignals`把散落的定时器状态汇总成三个bool:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
GlobalSignals Scheduler::makeGlobalSignals() const {
    const bool powerOnImminent = mDisplayPowerTimer &&
            (mPolicy.displayPowerMode != hal::PowerMode::ON ||
             mPolicy.displayPowerTimer == TimerState::Reset);
    return {.touch = mTouchTimer && mPolicy.touch == TouchState::Active,
            .idle = mPolicy.idleTimer == TimerState::Expired,
            .powerOnImminent = powerOnImminent};
}
```

选出的`FrameRateMode`不会直接生效,而是通过`mSchedulerCallback.requestDisplayModes`交给SurfaceFlinger,由SurfaceFlinger去HWC切模式;模式真正切好后再回调`Scheduler::onDisplayModeChanged`,进而`onModeChanged`通知app(走EventThread)。

### 10.3 内容检测与帧率override

内容检测(`chooseRefreshRateForContent`)用`mLayerHistory`统计各layer的实际提交帧率,汇总成`contentRequirements`,再喂给selector。而帧率override(某app锁定60fps这类)则走`mFrameRateOverrideMappings`,它按uid记录,最终影响`getVsyncPeriod(uid)`返回的周期和`throttleVsync`的节流判断。这些是刷新率系统的外围,细节在`RefreshRateSelector`/`LayerHistory`各自的文件里,这里点到为止。

---

## 11 VsyncModulator与VsyncConfiguration:相位与预算

### 11.1 VsyncConfiguration:参数表

`VsyncConfiguration`按刷新率存一组"相位预算"参数,`setVsyncConfig`把它们分发到三个消费方:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::setVsyncConfig(const VsyncConfig& config, Period vsyncPeriod) {
    setDuration(Cycle::Render,
                /* workDuration */ config.appWorkDuration,
                /* readyDuration */ config.sfWorkDuration);
    setDuration(Cycle::LastComposite,
                /* workDuration */ vsyncPeriod,
                /* readyDuration */ config.sfWorkDuration);
    setDuration(config.sfWorkDuration);
}
```

| 参数 | 含义 |
|---|---|
| `appWorkDuration` | app从收到vsync到画完一帧的预算 |
| `sfWorkDuration` | SF做commit+composite的预算 |
| `hwcMinWorkDuration` | HWC合成的最小预算(喂给FrameTargeter) |

对应地,三个消费方的duration是:

| 消费方 | workDuration | readyDuration |
|---|---|---|
| `"sf"`(MessageQueue) | `sfWorkDuration` | 0 |
| `"app"`(Cycle::Render) | `appWorkDuration` | `sfWorkDuration` |
| `"appSf"`(Cycle::LastComposite) | `vsyncPeriod` | `sfWorkDuration` |

### 11.2 VsyncModulator:相位平移

`VsyncModulator`在"事务提交、刷新率切换"这类会打乱节奏的事件里,**临时平移vsync相位**,让切换后的第一帧仍能对齐deadline,避免丢帧。它持有当前`VsyncConfig`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
mVsyncModulator = sp<VsyncModulator>::make(mVsyncConfiguration->getCurrentConfigs());
```

两个典型调用点:

1. `updatePhaseConfiguration`:刷新率确定后,`mVsyncModulator->setVsyncConfigSet(...)`算出新配置,再`setVsyncConfig`下发。
2. `modulateVsync`:事务里如果需要在某显示上平移相位,就`mVsyncModulator`算一个新config,再`setVsyncConfig`。

`onFrameSignal`里还用了`mVsyncModulator->getVsyncConfig().sfWorkDuration`作为`BeginFrameArgs`的输入,说明调制后的config直接影响这一帧的目标计算。

---

## 12 FrameTargeter:帧目标计算

`FrameTargeter`是"给这一帧定目标"的类。每个显示一个,在`onFrameSignal`里被`beginFrame`/`endFrame`夹住。

### 12.1 beginFrame

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp (onFrameSignal里)
const FrameTargeter::BeginFrameArgs beginFrameArgs =
        {.frameBeginTime = SchedulerClock::now(),
         .vsyncId = vsyncId,
         .expectedVsyncTime = expectedVsyncTime,
         .sfWorkDuration = mVsyncModulator->getVsyncConfig().sfWorkDuration,
         .hwcMinWorkDuration = mVsyncConfiguration->getCurrentConfigs().hwcMinWorkDuration,
         .debugPresentTimeDelay = debugPresentDelay};

pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);
```

`beginFrame`根据输入算出一个`FrameTarget`,存下这一帧的:期望上屏时间(`expectedPresentTime`)、期望帧时长(`expectedFrameDuration`)、deadline等。`FrameTargeter::target()`返回这个target,`onFrameSignal`把它塞进`FrameTargets`传给`compositor.commit`。

### 12.2 endFrame

合成完(`composite`返回结果)后:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp (onFrameSignal里)
for (const auto& [id, targeter] : targeters) {
    targeter->endFrame(*resultsPerDisplay.get(id));
}
```

`endFrame`把这一帧的**实际**合成结果(实际present时间等)喂回targeter,让它学习"预测和实际差多少",用于下一帧更准的目标。这就是Scheduler的**闭环**:预测→合成→反馈→再预测。

`Scheduler::expectedPresentTimeForPacesetter()`暴露的正是pacesetter的`target().expectedPresentTime()`,供需要知道"下一帧几点上屏"的上层(如FrameTimeline)使用。

---

## 13 硬件vsync的开关

硬件vsync是Scheduler所有预测的原始信号源,但为了省电,它**按需开关**。这一节串起完整链路。

### 13.1 开关入口

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::enableHardwareVsync(PhysicalDisplayId id) {
    getVsyncSchedule(id)->enableHardwareVsync();
}
void Scheduler::disableHardwareVsync(PhysicalDisplayId id, bool disallow) {
    getVsyncSchedule(id)->disableHardwareVsync(disallow);
}
```

`enableHardwareVsync`/`disableHardwareVsync`是`VsyncSchedule`的方法,它们内部最终会调用构造时传入的那个lambda → `Scheduler::onHardwareVsyncRequest`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::onHardwareVsyncRequest(PhysicalDisplayId id, bool enabled) {
    // 排到主线程,串行化对"pending硬件vsync状态"的读写
    schedule([=, this]() {
        if (const auto displayOpt = mDisplays.get(id)) {
            auto& display = displayOpt->get();
            display.schedulePtr->setPendingHardwareVsyncState(enabled);
            if (display.powerMode != hal::PowerMode::OFF) {
                mSchedulerCallback.requestHardwareVsync(id, enabled);
            }
        }
    });
}
```

最后一步`mSchedulerCallback.requestHardwareVsync`由SurfaceFlinger实现,它会调HWC的`setVsyncEnabled`真正开关硬件vsync中断。注意整条链被`schedule()`包起来,**切回主线程执行**,因为硬件vsync开关状态只在主线程串行读写。

### 13.2 resync:重新对齐硬件vsync

`resync`是"让Scheduler重新对齐到硬件vsync"的入口,带节流:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::resync() {
    static constexpr nsecs_t kIgnoreDelay = ms2ns(750);
    const nsecs_t now = systemTime();
    const nsecs_t last = mLastResyncTime.exchange(now);
    if (now - last > kIgnoreDelay) {
        resyncAllToHardwareVsync(false /* allowToEnable */);
    }
}
```

`kIgnoreDelay = 750ms`防止resync风暴:750ms内重复的resync请求被忽略。`resyncAllToHardwareVsync`遍历所有非熄灭的显示,`resyncToHardwareVsyncLocked`里若硬件vsync被允许开启,就`onDisplayModeChanged(force=false)`让tracker用当前模式重新校准。

`IEventThreadCallback::resync`的实现就是它——EventThread在app每次`requestNextVsync`时都会`mCallback.resync()`,保证app要画帧时Scheduler的相位和硬件vsync是准的。

### 13.3 addResyncSample:喂硬件vsync样本

硬件vsync从中断进来,最终落到这里:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
bool Scheduler::addResyncSample(PhysicalDisplayId id, nsecs_t timestamp,
                                std::optional<nsecs_t> hwcVsyncPeriodIn) {
    const auto hwcVsyncPeriod = ftl::Optional(hwcVsyncPeriodIn).transform([](nsecs_t nanos) {
        return Period::fromNs(nanos);
    });
    auto schedule = getVsyncSchedule(id);
    if (!schedule) return false;
    return schedule->addResyncSample(TimePoint::fromNs(timestamp), hwcVsyncPeriod);
}
```

`VsyncSchedule::addResyncSample`把时间戳喂给`VsyncController::addHwVsyncTimestamp`,后者再喂给`VSyncTracker::addVsyncTimestamp`,更新周期/相位模型。它的完整实现揭示了硬件vsync的开关决策:

```cpp
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
    if (needsHwVsync) {
        enableHardwareVsync();                 // 模型还要更多样本
    } else {
        disableHardwareVsync(false /* disallow */);  // 样本够了,关掉省电
    }
    return periodFlushed;                      // 返回"周期切换是否完成"
}
```

注意两点:一是样本只在`mHwVsyncState == Enabled`时才喂给controller(硬件vsync关了就不需要样本了);二是controller返回的`needsHwVsync`直接决定开还是关硬件vsync,而`addResyncSample`对外返回的是`periodFlushed`(VRR屏切周期时表示"周期切换已完成")。这个入口的上游是`SurfaceFlinger::onComposerHalVsync()`(HWC的vsync回调),详见`SurfaceFlinger教程.md`第9节。

### 13.4 硬件vsync的三态:HwVsyncState

`VsyncSchedule`用一个三态枚举跟踪硬件vsync状态,`enableHardwareVsync`/`disableHardwareVsync`/`isHardwareVsyncAllowed`都围着它转:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
void VsyncSchedule::enableHardwareVsyncLocked() {
    if (mHwVsyncState == HwVsyncState::Disabled) {
        getTracker().resetModel();            // 开硬件vsync时重置模型,重新校准
        mRequestHardwareVsync(mId, true);
        mHwVsyncState = HwVsyncState::Enabled;
    }
}

void VsyncSchedule::disableHardwareVsync(bool disallow) {
    switch (mHwVsyncState) {
        case HwVsyncState::Enabled:
            mRequestHardwareVsync(mId, false);
            [[fallthrough]];
        case HwVsyncState::Disabled:
            mHwVsyncState = disallow ? HwVsyncState::Disallowed : HwVsyncState::Disabled;
            break;
        case HwVsyncState::Disallowed:
            break;
    }
}
```

| 状态 | 含义 |
|---|---|
| `Enabled` | 硬件vsync已开启,正在上报样本 |
| `Disabled` | 硬件vsync关闭,但**允许**再次开启 |
| `Disallowed` | 硬件vsync被**禁止**开启(如屏幕熄灭),`isHardwareVsyncAllowed`返回false |

`enableHardwareVsyncLocked`里最关键的一步是`getTracker().resetModel()`:重新开硬件vsync时,旧模型已失准,必须先清空,让`VSyncPredictor`重新收集样本、重新拟合周期相位。

### 13.5 整条硬件vsync链路回顾

| 阶段 | 方法 | 线程 |
|---|---|---|
| HWC产生硬件vsync | `SurfaceFlinger::onComposerHalVsync()` | HWC回调线程 |
| 转给Scheduler | `Scheduler::addResyncSample()` | HWC回调线程 |
| 喂给tracker校准 | `VsyncSchedule::addResyncSample` → `VSyncReactor::addHwVsyncTimestamp` → `VSyncPredictor::addVsyncTimestamp` | HWC回调线程 |
| 决定开/关硬件vsync | controller返回值(`needsHwVsync`)→ `VsyncSchedule::enableHardwareVsync`/`disableHardwareVsync` | HWC回调线程 |
| 真正开关中断 | `Scheduler::onHardwareVsyncRequest`(内部`schedule`切到主线程)→ `ISchedulerCallback::requestHardwareVsync` → HWC `setVsyncEnabled` | 主线程 |

---

## 14 回调接口

Scheduler靠三个接口与外界解耦,它们都由SurfaceFlinger实现:

### 14.1 ICompositor:让Scheduler调SurfaceFlinger干活

Scheduler不知道SurfaceFlinger的合成细节,它只通过`ICompositor`这几个方法发号施令(从调用点归纳,定义在`scheduler/interface/ICompositor.h`):

| 方法 | 作用 |
|---|---|
| `commit(PhysicalDisplayId, FrameTargets)` | 提交这一帧要画的layer与目标,返回是否有东西可合成 |
| `composite(PhysicalDisplayId, FrameTargeters)` | 合成+上屏,返回各显示结果 |
| `configure()` | 请求一次configure(热插拔/模式切换后) |
| `sample()` | 采集这一帧的统计数据 |
| `sendNotifyExpectedPresentHint(PhysicalDisplayId)` | 通知期望上屏时间(VRR相关) |

其中`configure()`通过`MessageQueue::scheduleConfigure`触发(见MessageQueue.cpp里的`ConfigureHandler`),`commit`/`composite`/`sample`都在`onFrameSignal`里调用。

### 14.2 ISchedulerCallback:让Scheduler调SurfaceFlinger做决策

`ISchedulerCallback`(在`services/surfaceflinger/Scheduler/ISchedulerCallback.h`)是反向回调,从Scheduler.cpp的调用点归纳:

| 方法 | 作用 |
|---|---|
| `requestHardwareVsync(PhysicalDisplayId, bool)` | 请求开/关硬件vsync |
| `requestDisplayModes(vector<DisplayModeRequest>)` | 请求切换到选出的显示模式 |
| `onCommitNotComposited()` | commit没有合成任何东西(空帧) |
| `onChoreographerAttached()` | 有Choreographer attach到某layer |
| `kernelTimerChanged(bool expired)` | 内核idle定时器状态变化 |
| `vrrDisplayIdle(bool idle)` | VRR显示idle状态 |
| `onExpectedPresentTimePosted(TimePoint, DisplayModePtr, Fps)` | 期望上屏时间已提交 |

### 14.3 IEventThreadCallback:Scheduler服务EventThread

第1.1节已列出,它让Scheduler作为EventThread的"策略顾问":`throttleVsync`回答"要不要节流这个uid的vsync",`getVsyncPeriod`回答"这个uid的vsync周期",`resync`请求对齐硬件vsync,`onExpectedPresentTimePosted`通知上屏时间。

---

## 15 一帧的完整调度流程

把前面所有类串起来,一次"SF合成一帧 + app收到vsync"的完整时序如下:

1. **硬件vsync中断**:HWC到点产生硬件vsync,回调`SurfaceFlinger::onComposerHalVsync()` → `Scheduler::addResyncSample()` → `VsyncController::addHwVsyncTimestamp()` → `VSyncTracker::addVsyncTimestamp()`,tracker更新周期/相位模型。

2. **tracker预测**:`VSyncDispatchTimerQueue`里的`"sf"`、`"app"`、`"appSf"`三个entry,各自用tracker的`nextAnticipatedVSyncTimeFrom`算出下一帧的`wakeupTime = vsyncTime - workDuration - readyDuration`,取最小的`wakeupTime`去`setTimer`设定`TimeKeeper`。

3. **定时器线程触发**:`TimeKeeper`到点执行`VSyncDispatchTimerQueue::timerCallback`,收集所有该醒的entry,在锁外逐个调用它们的回调。

4. **"sf"回调唤醒主线程**:`MessageQueue::vsyncCallback`(定时器线程上)生成vsyncId,调`Handler::dispatchFrame`,经`sendMessage`把消息投给主线程Looper。主线程从`pollOnce`醒来,执行`Handler::handleMessage` → `Scheduler::onFrameSignal`。

5. **commit+composite**:`onFrameSignal`先`FrameTargeter::beginFrame`算目标,再`ICompositor::commit`提交事务、`composite`合成上屏、`sample`采样、`endFrame`反馈结果。若还有下一帧要画,SurfaceFlinger会再调`MessageQueue::scheduleFrame`让"sf"回调继续排下一次。

6. **"app"回调唤醒EventThread**:同一轮timerCallback里,`EventThread::onVsync`(定时器线程上)把vsync事件塞进`mPendingEvents`并notify,EventThread线程醒来后`threadMain`经`shouldConsumeEvent`(节流判断)→`dispatchEvent`把vsync写进对应连接(`EventThreadConnection`)的BitTube。

7. **app侧Choreographer收到vsync**:app进程的`DisplayEventReceiver`从socket pair读到vsync,Choreographer据此`doFrame`,开始画下一帧;画完提交buffer,再触发SF的下一轮commit/composite,循环往复。

整个过程中,三类线程各司其职:

| 线程 | 干的活 |
|---|---|
| SF主线程 | `run()`→`waitMessage()`循环,被"sf"回调唤醒后做commit/composite |
| 定时器线程(TimeKeeper) | 到点触发`timerCallback`,分发"sf"/"app"/"appSf"三个回调 |
| EventThread线程("app"/"appSf") | 等`onVsync`的通知,把vsync事件分发给app连接 |

这三类线程的协作全景,配合`SurfaceFlinger线程.md`第8节"一帧里各线程如何协作"一起看,能把Scheduler的完整节奏看得更清楚。
