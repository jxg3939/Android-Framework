# SurfaceFlinger的commit与composite

SurfaceFlinger主线程每被唤醒一次,就依次做两件事:先`commit`,再`composite`。`commit`负责把应用进程通过binder提交上来的事务合并进前端状态、把新buffer从各Layer的drawing state搬到可合成的buffer信息里;`composite`负责把这些状态交给CompositionEngine,由CompositionEngine决定哪些层走HWC(硬件合成)、哪些层走GPU(即RenderEngine的client合成),最后把结果提交给HWC上屏。两者之后还有一整套收尾工作:present fence的收集、release fence的回传、FrameTimeline的记账、VsyncController的校准、app回调的分发。

本文基于`android-17.0.0_r1`的源码,逐段拆开这条链路。涉及的目录主要为:

| 目录 | 内容 |
|---|---|
| `frameworks/native/services/surfaceflinger/` | `SurfaceFlinger.cpp`、`Layer.cpp`、`LayerFE.cpp`、`TransactionCallbackInvoker.cpp`等 |
| `frameworks/native/services/surfaceflinger/Scheduler/` | `Scheduler.cpp`、`MessageQueue.cpp`、`FrameTimeline.cpp`、`VSyncDispatchTimerQueue.cpp`、`VSyncReactor.cpp`、`VsyncSchedule.cpp`、`VsyncController.h` |
| `frameworks/native/services/surfaceflinger/Scheduler/src/` | `FrameTargeter.cpp`、`PresentLatencyTracker.cpp`、`Timer.cpp` |
| `frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/` | `FrameTargeter.h`、`TransactionSchedule.h`、`VsyncId.h`,以及`interface/`下的`ICompositor.h`、`CompositeResult.h`、`CompositionCoverage.h` |
| `frameworks/native/services/surfaceflinger/CompositionEngine/` | `CompositionEngine.cpp`、`Output.cpp`、`OutputLayer.cpp`、`Display.cpp`、`RenderSurface.cpp` |
| `frameworks/native/services/surfaceflinger/DisplayHardware/` | `HWComposer.cpp`、`HWC2.cpp`、`FramebufferSurface.cpp` |

本文与`SurfaceFlinger的Scheduler.md`、`Vsync信号完整流转.md`、`SurfaceFlinger教程.md`是互补关系:Scheduler篇讲"节奏是怎么算出来的",Vsync篇讲"vsync信号怎么从硬件走到SF",教程篇给出整条图形栈的概览。本文只回答三个问题:怎么走到commit、commit怎么进入composite、composite之后做了什么。

---

## 1 两个阶段在主循环里的位置

### 1.1 Scheduler::run就是SurfaceFlinger主线程

SurfaceFlinger的`main`最终调用`SurfaceFlinger::run`,它只做一件事:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::run() {
    mScheduler->run();
}
```

`Scheduler::run`是一个永不退出的循环:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::run() {
    while (true) {
        waitMessage();
    }
}
```

`waitMessage`来自`Scheduler`的另一个基类`android::impl::MessageQueue`:

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
            case Looper::POLL_ERROR:
                ALOGE("Looper::POLL_ERROR");
                continue;
            case Looper::POLL_TIMEOUT:
                // timeout (should not happen)
                continue;
            default:
                // should not happen
                ALOGE("Looper::pollOnce() returned unknown status %d", ret);
                continue;
        }
    } while (true);
}
```

主线程绝大部分时间阻塞在`mLooper->pollOnce(-1)`上。把它唤醒的方式有两类:一类是事务到达后主动`sendMessage`(见2.1),另一类是vsync定时器线程回调`vsyncCallback`后再`sendMessage`(见2.2、2.3)。两条路最终都汇入同一个消息处理函数,也就是`MessageQueue::Handler::handleMessage`。

### 1.2 commit与composite由ICompositor接口定义

Scheduler并不直接认识SurfaceFlinger。它只认识一个接口`ICompositor`,SurfaceFlinger私有继承并实现了它:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/interface/ICompositor.h
struct ICompositor {
    // Configures physical displays, processing hotplug and/or mode setting via the Composer HAL.
    virtual void configure() = 0;
    // Commits transactions for layers and displays. Returns whether any state has been invalidated,
    // i.e. whether a frame should be composited for each display.
    virtual bool commit(PhysicalDisplayId pacesetterId, const scheduler::FrameTargets&) = 0;
    // Composites a frame for each display. CompositionEngine performs GPU and/or HAL composition
    // via RenderEngine and the Composer HAL, respectively.
    virtual CompositeResultsPerDisplay composite(PhysicalDisplayId pacesetterId,
                                                 const scheduler::FrameTargeters&) = 0;
    // Sends a hint about the expected present time
    virtual void sendNotifyExpectedPresentHint(PhysicalDisplayId) = 0;
    // Samples the composited frame via RegionSamplingThread.
    virtual void sample() = 0;

protected:
    ~ICompositor() = default;
};
```

这里的注释已经把两个阶段的职责写清楚了:`commit`提交事务,返回值表示"状态是否失效、是否还需要合成";`composite`真正驱动GPU/HAL合成。

`FrameTargets`和`FrameTargeters`是两个不同的类型:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/interface/ICompositor.h
using FrameTargets = ui::PhysicalDisplayMap<PhysicalDisplayId, const scheduler::FrameTarget*>;
using FrameTargeters = ui::PhysicalDisplayMap<PhysicalDisplayId, scheduler::FrameTargeter*>;
```

`FrameTarget`是"某一帧的目标"这一份只读数据(期望上屏时间、vsync id、是否丢帧等),`FrameTargeter`是产生并持有`FrameTarget`的对象,还额外记录present fence历史和上一帧的合成覆盖情况。`commit`接收`FrameTargets`,因为commit只需要读目标;`composite`接收`FrameTargeters`,因为合成结束后Scheduler要调用`endFrame`把本帧覆盖情况写回targeter。

### 1.3 返回值约定:commit返回false就没有composite

`Scheduler::onFrameSignal`里的控制流是硬性的:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
if (!compositor.commit(pacesetterPtr->displayId, targets)) {
    compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
    mSchedulerCallback.onCommitNotComposited();
    return;
}
```

只要`commit`返回`false`,`onFrameSignal`直接返回:不会调用`composite`,不会调用`sample`,也不会对任何targeter调用`endFrame`。所以理解"哪些情况commit会返回false"是理解整条链路的前提,3.1节会逐条列出。

### 1.4 两个阶段的产物

| 阶段 | 主要入口 | 关键产物 |
|---|---|---|
| commit | `SurfaceFlinger::commit` | 最新的前台/后端Layer状态、已latch的`mBufferInfo`、`mustComposite`布尔值、FrameTimeline的SfWakeUp记录 |
| composite | `SurfaceFlinger::composite` → `CompositionEngine::present` | 每个Output的合成策略、GPU合成结果、HWC present、每层的release fence、每个显示器的`CompositeResult` |

---

## 2 如何走到commit

走到commit有两条路:应用提交事务触发的"主动调度",以及vsync定时器到点触发的"定时唤醒"。两条路最终都会让主线程的Looper收到一条消息,消息处理函数调用`onFrameSignal`,再由`onFrameSignal`调用`ICompositor::commit`。

### 2.1 主动路径:事务到达触发scheduleFrame

应用进程通过`SurfaceComposerClient`提交事务,最终进入SurfaceFlinger的binder方法`setTransactionState`。这个函数在完成权限检查、状态清洗、构建`QueuedTransactionState`之后,把事务放进无锁队列:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    const auto schedule = [](uint32_t flags) {
        if (flags & eEarlyWakeupEnd) return TransactionSchedule::EarlyEnd;
        if (flags & eEarlyWakeupStart) return TransactionSchedule::EarlyStart;
        return TransactionSchedule::Late;
    }(state.flags);

    const auto frameHint = state.isFrameActive() ? FrameHint::kActive : FrameHint::kNone;
    // Copy fields of |state| needed after it is moved into queueTransaction
    {
        // Transactions are added via a lockless queue and does not need to be added from the main
        // thread.
        ftl::FakeGuard guard(kMainThreadContext);
        mTransactionHandler.queueTransaction(std::move(state));
    }
```

注意`queueTransaction`可以在任意binder线程执行,因为它用的是无锁队列。真正把事务"排进SF这一帧"的动作在下一行:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    setTransactionFlags(eTransactionFlushNeeded, schedule, frameHint,
                        std::move(transactionState.mEarlyWakeupInfos));
```

`setTransactionFlags`是主动路径的核心:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::setTransactionFlags(uint32_t mask, TransactionSchedule schedule,
                                         FrameHint frameHint,
                                         std::vector<gui::EarlyWakeupInfo> earlyWakeupInfos) {
    mScheduler->modulateVsync({}, &VsyncModulator::setTransactionSchedule, schedule,
                              std::move(earlyWakeupInfos));
    uint32_t transactionFlags = mTransactionFlags.fetch_or(mask);
    SFTRACE_INT("mTransactionFlags", transactionFlags);

    if (const bool scheduled = transactionFlags & mask; !scheduled) {
        mScheduler->resync(IEventThreadCallback::ResyncCaller::Transaction);
        scheduleCommit(frameHint);
    } else if (frameHint == FrameHint::kActive) {
        // Even if the next frame is already scheduled, we should reset the idle timer
        // as a new activity just happened.
        mScheduler->resetAllIdleTimers();
    }
}
```

逐点说明:

1. `modulateVsync`根据事务的`TransactionSchedule`(EarlyStart/EarlyEnd/Late)调整SF这一帧的work duration,也就是SF要多早被vsync唤醒。
2. `mTransactionFlags.fetch_or(mask)`返回的是"置位之前"的旧值。只有旧值里没有对应位(`!scheduled`),才说明这一位是本次新置上的,需要真正去调度一帧。如果已经调度过,就不重复调度。
3. `mScheduler->resync(...)`让Scheduler在必要时重新对齐硬件vsync。
4. `scheduleCommit(frameHint)`最终落到`mScheduler->scheduleFrame`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::scheduleCommit(FrameHint hint, Duration workDurationSlack) {
    if (hint == FrameHint::kActive) {
        mScheduler->resetAllIdleTimers();
    }
    mPowerAdvisor->notifyDisplayUpdateImminentAndCpuReset();
    mScheduler->scheduleFrame(workDurationSlack);
}
```

`Scheduler`没有自己重写`scheduleFrame`,而是通过`using Impl::scheduleFrame;`直接复用`MessageQueue::scheduleFrame`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h
    using Impl::scheduleFrame;
```

除了事务,其他主动调度入口还有:

| 入口 | 行为 |
|---|---|
| `SurfaceFlinger::scheduleComposite(FrameHint)` | 先置`mMustComposite = true`,再`scheduleCommit`;用于强制下一帧走合成 |
| `SurfaceFlinger::scheduleRepaint()` | 置`mGeometryDirty = true`,再`scheduleComposite(FrameHint::kActive)` |
| `SurfaceFlinger::composite`末尾的`mCompositionEngine->needsAnotherUpdate()` | 若为真,调用`scheduleCommit(FrameHint::kNone)`再补一帧 |
| `SurfaceFlinger::composite`中的`mScheduler->onCompositionPresented(presentTime)` | 若返回true(刷新率切换还差一帧),调用`scheduleComposite(FrameHint::kNone)` |
| `commit`里的`mScheduler->scheduleFrame()` | 显示器增删、模式切换、commit被推迟时,重新排一帧 |

`scheduleComposite`的代码如下:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::scheduleComposite(FrameHint hint) {
    mMustComposite = true;
    scheduleCommit(hint);
}
```

`mMustComposite`是`std::atomic_bool`,因此binder线程里的`scheduleComposite`和主线程`commit`里的`exchange(false)`不冲突。

### 2.2 从scheduleFrame到vsync定时器

`MessageQueue::scheduleFrame`只是把"SF希望在哪一个vsync被唤醒、并留出多少工作时间"告诉VSyncDispatch:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::scheduleFrame(Duration workDurationSlack) {
    SFTRACE_CALL();

    std::lock_guard lock(mVsync.mutex);
    const auto workDuration = Duration(mVsync.workDuration.get() - workDurationSlack);
    mVsync.scheduledFrameTimeOpt =
            mVsync.registration->schedule({.workDuration = workDuration.ns(),
                                           .readyDuration = 0,
                                           .lastVsync = mVsync.lastCallbackTime.ns()});
}
```

`mVsync.registration`是在Scheduler初始化vsync时注册的`VSyncCallbackRegistration`,名字固定为`"sf"`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    ...
    mVsync.registration = std::make_unique<
            scheduler::VSyncCallbackRegistration>(std::move(dispatch),
                                                  std::bind(&MessageQueue::vsyncCallback, this,
                                                            std::placeholders::_1,
                                                            std::placeholders::_2,
                                                            std::placeholders::_3),
                                                  "sf");
```

`VSyncCallbackRegistration::schedule`最终进入`VSyncDispatchTimerQueueEntry::schedule`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
ScheduleResult VSyncDispatchTimerQueueEntry::schedule(VSyncDispatch::ScheduleTiming timing,
                                                      VSyncTracker& tracker, nsecs_t now) {
    SFTRACE_NAME("VSyncDispatchTimerQueueEntry::schedule");
    auto nextVsyncTime =
            tracker.nextAnticipatedVSyncTimeFrom(std::max(timing.lastVsync,
                                                          now + timing.workDuration +
                                                                  timing.readyDuration),
                                                 timing.committedVsyncOpt.value_or(
                                                         timing.lastVsync));
    auto nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
    ...
    nextWakeupTime = std::max(now, nextVsyncTime - timing.workDuration - timing.readyDuration);

    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mScheduleTiming = timing;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return ScheduleResult{TimePoint::fromNs(nextWakeupTime), TimePoint::fromNs(nextVsyncTime)};
}
```

上面省略掉的那段是"防止跳过vsync"的逻辑。完整代码在两次计算之间还有:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
    bool const wouldSkipAVsyncTarget =
            mArmedInfo && (nextVsyncTime > (mArmedInfo->mActualVsyncTime + mMinVsyncDistance));
    bool const wouldSkipAWakeup =
            mArmedInfo && ((nextWakeupTime > (mArmedInfo->mActualWakeupTime + mMinVsyncDistance)));
    SFTRACE_FORMAT_INSTANT("%s: wouldSkipAVsyncTarget=%d wouldSkipAWakeup=%d", mName.c_str(),
                           wouldSkipAVsyncTarget, wouldSkipAWakeup);
    if (wouldSkipAVsyncTarget || wouldSkipAWakeup) {
        nextVsyncTime = mArmedInfo->mActualVsyncTime;
    } else {
        nextVsyncTime = adjustVsyncIfNeeded(tracker, nextVsyncTime);
    }
    nextWakeupTime = std::max(now, nextVsyncTime - timing.workDuration - timing.readyDuration);
```

`mArmedInfo`保存的是上一次`schedule`算出来的三个时间。如果这次算出的`nextVsyncTime`比上一次的目标晚超过`mMinVsyncDistance`,或者`nextWakeupTime`比上一次的唤醒时间晚超过`mMinVsyncDistance`,说明新目标已经跨过了上一次的vsync,继续用新目标会让上一次的目标被"跳过"。这时直接沿用上一次的`mActualVsyncTime`,避免因为一次迟到的`schedule`把某一帧整个吞掉。

`adjustVsyncIfNeeded`处理两种情况:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
nsecs_t VSyncDispatchTimerQueueEntry::adjustVsyncIfNeeded(VSyncTracker& tracker,
                                                          nsecs_t nextVsyncTime) const {
    bool const alreadyDispatchedForVsync = mLastDispatchTime &&
            ((*mLastDispatchTime + mMinVsyncDistance) >= nextVsyncTime &&
             (*mLastDispatchTime - mMinVsyncDistance) <= nextVsyncTime);
    const nsecs_t currentPeriod = tracker.currentPeriod();
    bool const nextVsyncTooClose = mLastDispatchTime &&
            (nextVsyncTime - *mLastDispatchTime + mMinVsyncDistance) <= currentPeriod;
    if (alreadyDispatchedForVsync) {
        SFTRACE_FORMAT_INSTANT("alreadyDispatchedForVsync");
        return tracker.nextAnticipatedVSyncTimeFrom(*mLastDispatchTime + mMinVsyncDistance,
                                                    *mLastDispatchTime);
    }

    if (nextVsyncTooClose) {
        SFTRACE_FORMAT_INSTANT("nextVsyncTooClose");
        return tracker.nextAnticipatedVSyncTimeFrom(*mLastDispatchTime + currentPeriod,
                                                    *mLastDispatchTime + currentPeriod);
    }

    return nextVsyncTime;
}
```

- `alreadyDispatchedForVsync`:新的`nextVsyncTime`落在上一次派发时刻`mLastDispatchTime`的`mMinVsyncDistance`范围内,说明这个vsync已经派发过了,直接把目标推到`mLastDispatchTime + mMinVsyncDistance`之后的下一个vsync。
- `nextVsyncTooClose`:新目标距离上一次派发不足一个`currentPeriod`(把`mMinVsyncDistance`算进去之后),说明两个vsync挨得太近,强行推到`mLastDispatchTime + currentPeriod`。

这两步之后才重新计算`nextWakeupTime`,并`std::max(now, ...)`保证唤醒时间不会落在过去。

三个时间点的含义:

| 变量 | 含义 |
|---|---|
| `nextVsyncTime` | 预测的下一个vsync时间戳,也是这一帧的目标上屏时刻 |
| `nextWakeupTime` | 等于`nextVsyncTime - workDuration - readyDuration`,即定时器应该唤醒回调的时刻 |
| `nextReadyTime` | 等于`nextVsyncTime - readyDuration`,即回调必须完成工作、结果就绪的时刻 |

`MessageQueue`注册时`readyDuration`传0,所以SF这一路实际上只有`workDuration`。`workDuration`由`VsyncModulator`根据刷新率和事务调度策略算出。

定时器线程到点后进入`VSyncDispatchTimerQueue::timerCallback`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
void VSyncDispatchTimerQueue::timerCallback() {
    SFTRACE_CALL();
    struct Invocation {
        std::shared_ptr<VSyncDispatchTimerQueueEntry> callback;
        nsecs_t vsyncTimestamp;
        nsecs_t wakeupTimestamp;
        nsecs_t deadlineTimestamp;
    };
    std::vector<Invocation> invocations;
    {
        std::lock_guard lock(mMutex);
        ...
        auto const now = mTimeKeeper->now();
        mLastTimerCallback = now;
        for (auto it = mCallbacks.begin(); it != mCallbacks.end(); it++) {
            auto& callback = it->second;
            auto const wakeupTime = callback->wakeupTime();
            if (!wakeupTime) {
                continue;
            }
            ...
            auto const readyTime = callback->readyTime();
            auto const lagAllowance = std::max(now - mIntendedWakeupTime, static_cast<nsecs_t>(0));
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
        ...
        invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                                      invocation.deadlineTimestamp);
    }
}
```

回调是在锁外执行的,这样`vsyncCallback`里再去拿`mVsync.mutex`不会死锁。回调的三个参数依次是:预测的vsync时间、唤醒时间、就绪截止时间。

### 2.3 vsyncCallback把帧信号投递成Looper消息

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    SFTRACE_CALL();
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

这里做了四件事:

1. `mVsync.value`在0/1之间翻转。它的类型是`TracedOrdinal<int>`,声明为:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.h
        TracedOrdinal<int> value = {"VSYNC-sf", 0};
```

也就是说,它在atrace里表现为一个名为`VSYNC-sf`的计数器,每收到一次vsync回调就翻转一次,在trace上形成0/1方波,用于直观看到SF这一路的vsync节拍。
2. 记录`lastCallbackTime`,并清掉`scheduledFrameTimeOpt`——因为这次调度已经被消费。
3. 通过`TokenManager::generateTokenForPredictions`生成一个`VsyncId`,把`targetWakeupTime`、`readyTime`、`vsyncTime`三元组注册进去。这个id会一路带到FrameTimeline,用于把"应用侧的vsync"和"SF侧的vsync"对上。
4. `dispatchFrame`把消息投给Looper:

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

`mFramePending.exchange(true)`是一个去重门闩:如果已经有一帧在排队等主线程处理,就不再重复投递消息,但会覆盖`mVsyncId`和`mExpectedVsyncTime`为最新的值。这保证了主线程不会被同一时刻的多个vsync回调淹没。

Looper消息的处理函数:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

先把门闩打开,再调用`onFrameSignal`。注意`mCompositor`是构造`MessageQueue`时传进来的`ICompositor&`,对Scheduler来说就是SurfaceFlinger自己。

### 2.4 Scheduler::onFrameSignal

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
    const auto debugPresentDelay = mDebugPresentDelay.load();
    mDebugPresentDelay.store(std::nullopt);

    const FrameTargeter::BeginFrameArgs beginFrameArgs =
            {.frameBeginTime = mClock->now(),
             .vsyncId = vsyncId,
             .expectedVsyncTime = expectedVsyncTime,
             .sfWorkDuration = mVsyncModulator->getVsyncConfig().sfWorkDuration,
             .hwcMinWorkDuration = getCurrentVsyncConfigs().hwcMinWorkDuration,
             .debugPresentTimeDelay = debugPresentDelay};

    ftl::NonNull<const Display*> pacesetterPtr = pacesetterPtrLocked();
    pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);

    ui::PhysicalDisplayVector<PhysicalDisplayId> presentableDisplays = {pacesetterPtr->displayId};
    {
        FrameTargets targets;
        targets.try_emplace(pacesetterPtr->displayId, &pacesetterPtr->targeterPtr->target());

        const bool followerDisplayBackpressure =
                FlagManager::getInstance().follower_display_backpressure_combined();
        if (!followerDisplayBackpressure) {
            expectedVsyncTime = pacesetterPtr->targeterPtr->target().expectedPresentTime();
        }

        for (const auto& [id, display] : mDisplays) {
            if (id == pacesetterPtr->displayId) continue;

            auto followerBeginFrameArgs = beginFrameArgs;
            const TimePoint nextFollowerVsync =
                    display.schedulePtr->vsyncDeadlineAfter(expectedVsyncTime);
            followerBeginFrameArgs.expectedVsyncTime = nextFollowerVsync;

            FrameTargeter& targeter = *display.targeterPtr;
            if (followerDisplayBackpressure && !isLockstepFollowerLocked(id)) {
                const size_t pendingFenceCount =
                        targeter.countPresentFencesPendingAt(beginFrameArgs.frameBeginTime);
                const TimePoint nextFollowerVsync =
                        display.schedulePtr->vsyncDeadlineAfter(beginFrameArgs.frameBeginTime);
                const bool isNextFollowerVsyncBeforeNextPacesetterVsync =
                        nextFollowerVsync < expectedVsyncTime;
                if (pendingFenceCount > 0 &&
                    // Tolerate the temporary lockup caused by backpressure if there's one pending
                    // present, and the vsync is anticipated to happen before the next pacesetter
                    // vsync, as it will resolve itself before the next pacesetter present.
                    !(pendingFenceCount == 1 && isNextFollowerVsyncBeforeNextPacesetterVsync)) {
                    SFTRACE_FORMAT_INSTANT("Avoiding backpressure for follower display %lu",
                                           id.value);
                    continue;
                }
            }

            targeter.beginFrame(followerBeginFrameArgs, *display.schedulePtr);
            targets.try_emplace(id, &targeter.target());
            presentableDisplays.push_back(id);
        }

        if (!compositor.commit(pacesetterPtr->displayId, targets)) {
            compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
            mSchedulerCallback.onCommitNotComposited();
            return;
        }
    }
```

这里有三个关键设计:

1. **pacesetter(领跑显示器)**:`pacesetterPtrLocked()`选出当前的主显示器。它的vsync决定整个SF的节奏,所以`beginFrame`的参数直接来自这次定时器回调。
2. **follower(跟随显示器)**:其他物理显示器各有自己的`VsyncSchedule`,它们的期望上屏时间由`display.schedulePtr->vsyncDeadlineAfter(expectedVsyncTime)`换算。当`follower_display_backpressure_combined`开启且不是lockstep follower时,如果该显示器还有pending的present fence,本帧就直接跳过它,不加入`presentableDisplays`。这样避免某个慢显示器拖住整个主线程。
3. **presentableDisplays**:只有真正参与本帧的显示器才会出现在后面`composite`的`FrameTargeters`里。

`beginFrame`之后,`FrameTarget`里的`expectedPresentTime()`等字段就固定了。`SurfaceFlinger::commit`读取的是`pacesetterFrameTargetPtr->...`,不是自己再算一遍。

### 2.5 beginFrame:把"现在"换算成"目标上屏时刻"

`FrameTargeter::beginFrame`是理解backpressure和丢帧统计的入口:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
void FrameTargeter::beginFrame(const BeginFrameArgs& args, const IVsyncSource& vsyncSource,
                               IsFencePendingFuncPtr isFencePendingFuncPtr) {
    mVsyncId = args.vsyncId;
    mFrameBeginTime = args.frameBeginTime;
    mDebugPresentTimeDelay = args.debugPresentTimeDelay;

    // The `expectedVsyncTime`, which was predicted when this frame was scheduled, is normally in
    // the future relative to `frameBeginTime`, but may not be for delayed frames. Adjust
    // `mExpectedPresentTime` accordingly, but not `mScheduledPresentTime`.
    const TimePoint lastScheduledPresentTime = mScheduledPresentTime;
    mScheduledPresentTime = args.expectedVsyncTime;

    const Period vsyncPeriod = vsyncSource.period();
    const Period minFramePeriod = vsyncSource.minFramePeriod();

    // Calculate the expected present time once and use the cached value throughout this frame to
    // make sure all layers are seeing this same value.
    if (args.expectedVsyncTime >= args.frameBeginTime) {
        mExpectedPresentTime = args.expectedVsyncTime;
    } else {
        mExpectedPresentTime = vsyncSource.vsyncDeadlineAfter(args.frameBeginTime);
        if (args.sfWorkDuration > vsyncPeriod) {
            // Inflate the expected present time if we're targeting the next VSYNC.
            mExpectedPresentTime += vsyncPeriod;
        }
    }

    if (!mSupportsExpectedPresentTime) {
        mEarliestPresentTime =
                computeEarliestPresentTime(vsyncPeriod, minFramePeriod, args.hwcMinWorkDuration);
    }
    ...
}
```

逐点说明:

- `mScheduledPresentTime`原样保留调度时的预测值,不做修正。
- `mExpectedPresentTime`则要做修正:如果定时器回调延迟了,`expectedVsyncTime`可能已经过去,此时用`vsyncSource.vsyncDeadlineAfter(frameBeginTime)`重新取下一个vsync deadline。如果`sfWorkDuration`大于一个vsync周期,说明这一帧本来就打算跨过一个vsync,再`+= vsyncPeriod`。
- `mEarliestPresentTime`只在composer不支持"expected present time"参数时才计算,用于HWC侧防止过早present。

同一函数后半段计算backpressure和丢帧:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
    const auto [wouldBackpressure, fence] =
            expectedSignaledPresentFence(vsyncPeriod, minFramePeriod);
    ...
    mFramePending = fence.fenceTime != FenceTime::NO_FENCE &&
            isFencePending(fence.fenceTime, graceTimeForPresentFenceMs);
    ...
    mFrameMissed = mFramePending || [&] {
        const nsecs_t pastPresentTime = fence.fenceTime->getSignalTime();
        if (pastPresentTime < 0) return false;
        mLastSignaledFrameTime = {.signalTime = TimePoint::fromNs(pastPresentTime),
                                  .expectedPresentTime = fence.expectedPresentTime};
        ...
        const nsecs_t frameMissedSlop = vsyncPeriod.ns() / 2;
        return lastScheduledPresentTime.ns() < pastPresentTime - frameMissedSlop;
    }();

    mHwcFrameMissed = mFrameMissed && mCompositionCoverage.test(CompositionCoverage::Hwc);
    mGpuFrameMissed = mFrameMissed && mCompositionCoverage.test(CompositionCoverage::Gpu);
    ...
    mWouldBackpressureHwc = mFramePending && wouldBackpressure;
```

`mCompositionCoverage`是**上一帧**由`endFrame`写进来的值,所以`mHwcFrameMissed`/`mGpuFrameMissed`的含义是"上一帧是HWC合成还是GPU合成,以及这一帧是否因为上一帧的present fence还没signal而被迫等待"。这条反馈链在6.3节展开。

`isFencePending`的实现是用带grace时间的`FenceTime::wait`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
bool FrameTargeter::isFencePending(const FenceTimePtr& fence, int graceTimeMs) {
    SFTRACE_CALL();
    const status_t status = fence->wait(graceTimeMs);

    // This is the same as Fence::Status::Unsignaled, but it saves a call to getStatus,
    // which calls wait(0) again internally.
    return status == -ETIME;
}
```

如果present fence在grace time内没有signal,就认为上一帧还没真正上屏,本帧"pending"。

---

## 3 commit阶段

### 3.1 commit的提前返回

`SurfaceFlinger::commit`的签名是:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
bool SurfaceFlinger::commit(PhysicalDisplayId pacesetterId,
                            const scheduler::FrameTargets& frameTargets) EXCLUDES(mStateLock) {
    const scheduler::FrameTarget* pacesetterFrameTargetPtr = frameTargets.get(pacesetterId)->get();
    const VsyncId vsyncId = pacesetterFrameTargetPtr->vsyncId();
```

它先记一次丢帧统计,然后进入一连串"检查不通过就返回false"的分支。

**检查一:显示器是否还在**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    {
        for (const auto [displayId, _] : frameTargets) {
            const bool hasDisplay =
                    FTL_FAKE_GUARD(mStateLock, mPhysicalDisplays.contains(displayId));
            if (!hasDisplay) {
                MODE_TRANSITION_LOCK();
                processDisplayChangesLocked();
                mScheduler->scheduleFrame();
                return false;
            }
        }
    }
```

注释说明了原因:`processHotplugDisconnect()`不会立刻把显示器移除同步给Scheduler和CompositionEngine,需要下一次commit调用`processDisplayChangesLocked()`把剩余步骤做完。所以这一帧放弃,重新排一帧。

**检查二:模式切换尚未完成,且上一帧还pending**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (std::any_of(frameTargets.begin(), frameTargets.end(),
                    [this](const auto& pair) FTL_FAKE_GUARD(kMainThreadContext) {
                        const auto [displayId, target] = pair;
                        return target->isFramePending() &&
                                mDisplayModeController.isModeSetPending(displayId);
                    })) {
        mScheduler->scheduleFrame();
        return false;
    }
```

**检查三:模式切换完成失败**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    {
        MODE_TRANSITION_LOCK();

        for (const auto [displayId, _] : frameTargets) {
            if (mDisplayModeController.isModeSetPending(displayId)) {
                if (!finalizeDisplayModeChange(displayId)) {
                    mScheduler->scheduleFrame();
                    return false;
                }
            }
        }
    }
```

**检查四:HWC backpressure**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (pacesetterFrameTargetPtr->wouldBackpressureHwc()) {
        if (mBackpressureGpuComposition || pacesetterFrameTargetPtr->didMissHwcFrame()) {
            mScheduler->getVsyncSchedule()->getTracker().onFrameMissed(
                    pacesetterFrameTargetPtr->expectedPresentTime());
            const Duration slack = TimePoint::now() - pacesetterFrameTargetPtr->frameBeginTime();
            scheduleCommit(FrameHint::kNone, slack);
            return false;
        }
    }
```

`wouldBackpressureHwc()`表示"上一帧还没上屏,而本帧的上屏时间又和它撞上了"。此时只要满足下面两个条件之一,就告诉VsyncTracker这一帧miss了,并带着已经消耗掉的slack重新调度:

- `mBackpressureGpuComposition`为true;
- `didMissHwcFrame()`为true。

`mBackpressureGpuComposition`不是"上一帧用了GPU"的运行时状态,而是构造函数里从系统属性读出来的开关:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mBackpressureGpuComposition = base::GetBoolProperty("debug.sf.enable_gl_backpressure"s, true);
    ALOGI_IF(mBackpressureGpuComposition, "Enabling backpressure for GPU composition");
```

默认值是true,所以默认情况下只要`wouldBackpressureHwc()`成立就会放弃本帧。这个开关同时决定`FrameTargeter`是否给"即将signal的present fence"留1ms宽限(`Scheduler/src/FrameTargeter.cpp`里`beginFrame`的`considerBackpressure`):

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
    const int graceTimeForPresentFenceMs = [&] {
        const bool considerBackpressure =
                mBackpressureGpuComposition || !mCompositionCoverage.test(CompositionCoverage::Gpu);

        if (!wouldBackpressure || !considerBackpressure) {
            return 0;
        }

        return static_cast<int>((std::abs(fence.expectedPresentTime.ns() - mFrameBeginTime.ns()) <=
                                 Duration(1ms).ns()));
    }();
```

当backpressure开关关闭、且上一帧的`mCompositionCoverage`里没有`Gpu`时,不授予宽限。

注意这里用`workDurationSlack`把`workDuration`减掉已经浪费的时间,避免下一帧又算错唤醒时刻。

如果`mBackpressureGpuComposition`关闭且`didMissHwcFrame()`也为false,则继续往下走,不提前返回。

**提前返回汇总**

| 条件 | 返回前的动作 | 含义 |
|---|---|---|
| `frameTargets`里有显示器已不在`mPhysicalDisplays` | `processDisplayChangesLocked()`、`mScheduler->scheduleFrame()` | 热插拔需要下一帧重新commit |
| 有显示器模式切换pending且`isFramePending()` | `mScheduler->scheduleFrame()` | 等模式切换fence |
| `finalizeDisplayModeChange`失败 | `mScheduler->scheduleFrame()` | 模式切换失败重试 |
| HWC backpressure且(`mBackpressureGpuComposition`打开或HWC丢帧) | `VsyncTracker::onFrameMissed`、`scheduleCommit(kNone, slack)` | 主动放弃本帧,避免堆积 |

### 3.2 commit开始前的准备

检查通过后,commit先处理电源提示和叠加层动画:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    bool shouldEnablePowerHintSession = true;
    if (!FlagManager::getInstance().align_adpf_with_sf_opt_policy()) {
        shouldEnablePowerHintSession =
                FTL_FAKE_GUARD(mStateLock, hasDisplay([](const DisplayDevice& display) {
                                   return display.getPowerMode() == hal::PowerMode::ON;
                               }));
    }

    // Save this once per commit + composite to ensure consistency.
    mPowerHintSessionEnabled = mPowerAdvisor->usePowerHintSession() && shouldEnablePowerHintSession;
    if (mPowerHintSessionEnabled) {
        mPowerAdvisor->setCommitStart(pacesetterFrameTargetPtr->frameBeginTime());
        mPowerAdvisor->setExpectedPresentTime(pacesetterFrameTargetPtr->expectedPresentTime());

        // Frame delay is how long we should have minus how long we actually have.
        const Duration idealSfWorkDuration =
                mScheduler->vsyncModulator().getVsyncConfig().sfWorkDuration;
        const Duration frameDelay =
                idealSfWorkDuration - pacesetterFrameTargetPtr->expectedFrameDuration();

        mPowerAdvisor->setFrameDelay(frameDelay);
        mPowerAdvisor->setTotalFrameTargetWorkDuration(idealSfWorkDuration);

        const Period idealVsyncPeriod =
                mDisplayModeController.getActiveMode(pacesetterId).fps.getPeriod();
        mPowerAdvisor->updateTargetWorkDuration(idealVsyncPeriod);
    }
```

`mPowerHintSessionEnabled`在这里算一次,贯穿commit和composite,保证两阶段看到的是同一个值。`frameDelay = idealSfWorkDuration - expectedFrameDuration()`:如果预期可用的工作时间比理想值短,frameDelay就是负的,表示已经迟了。

然后是HDR/SDR比率叠加层与刷新率叠加层的动画:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (mRefreshRateOverlaySpinner) {
        Mutex::Autolock lock(mStateLock);
        for (const auto& [_, display] : mDisplays) {
            display->animateRefreshRateOverlay();
        }
    }
    if (mHdrSdrRatioOverlay) {
        Mutex::Autolock lock(mStateLock);
        for (const auto& [_, display] : mDisplays) {
            display->animateHdrSdrRatioOverlay();
        }
    }
```

接下来处理渲染资源缓存。注释说明了顺序要求:必须在snapshot尝试解析render resource token之前完成。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // render resource tokens.
    mIpcCache->processPendingOperations();
```

### 3.3 FrameTimeline记录的wakeup

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // Composite if transactions were committed, or if requested by HWC.
    bool mustComposite = mMustComposite.exchange(false);
    {
        const bool powerModeChangeInProgress = mPowerModeInProgressCount > 0 ||
                (mustComposite ? mPowerModeChangeInProgress.exchange(false)
                               : mPowerModeChangeInProgress.load());
        scheduler::FrameTimelineDisplayState displayState = {
                .poweredOn = FTL_FAKE_GUARD(mStateLock, getDisplayDeviceLocked(pacesetterId))
                                     ->getPowerMode() == hal::PowerMode::ON,
                .modeChangeInProgress = mScheduler->getVsyncSchedule()->isModeChangeInProgress(),
                .powerModeChangeInProgress = powerModeChangeInProgress,
        };
        mFrameTimeline->setSfWakeUp(ftl::to_underlying(vsyncId),
                                    pacesetterFrameTargetPtr->frameBeginTime().ns(),
                                    Fps::fromPeriodNsecs(vsyncPeriod.ns()),
                                    mScheduler->getPacesetterRefreshRate(), displayState);
```

`mMustComposite.exchange(false)`一次性读走并清零"强制合成"标志。也就是说,只要这一帧被`scheduleComposite`标记过,哪怕没有任何新事务,也必须合成。

`setSfWakeUp`把SF侧的vsync信息写进FrameTimeline。参数依次是:vsync token、SF开始工作的时刻、刷新率、pacesetter渲染率、显示状态。FrameTimeline据此判断这一帧是否jank,以及jank的原因是应用没按时交buffer还是SF自己太慢。`displayState`里的三个布尔值用于标记"显示器没开""正在切模式""正在切电源状态",这些情况下FrameTimeline不应把延迟算成jank。

### 3.4 updateLayerSnapshots

`updateLayerSnapshots`是commit的主体,可以分成五段。

**第一段:收集并应用事务**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
bool SurfaceFlinger::updateLayerSnapshots(VsyncId vsyncId, nsecs_t frameTimeNs,
                                          nsecs_t expectedPresentTimeNs, bool flushTransactions,
                                          bool& outTransactionsAreEmpty) EXCLUDES(mStateLock) {
    using Changes = frontend::RequestedLayerState::Changes;
    SFTRACE_CALL();
    SFTRACE_NAME_FOR_TRACK(WorkloadTracer::TRACK_NAME, "Transaction Handling");
    frontend::Update update;
    if (flushTransactions) {
        SFTRACE_NAME("TransactionHandler:flushTransactions");
        // Locking:
        // 1. to prevent onHandleDestroyed from being called while the state lock is held,
        // we must keep a copy of the transactions (specifically the composer
        // states) around outside the scope of the lock.
        // 2. Transactions and created layers do not share a lock. To prevent applying
        // transactions with layers still in the createdLayer queue, collect the transactions
        // before committing the created layers.
        // 3. Transactions can only be flushed after adding layers, since the layer can be a newly
        // created one
        mTransactionHandler.collectTransactions();
        {
            // TODO(b/238781169) lockless queue this and keep order.
            std::scoped_lock<std::mutex> lock(mCreatedLayersLock);
            update.legacyLayers = std::move(mCreatedLayers);
            mCreatedLayers.clear();
            update.newLayers = std::move(mNewLayers);
            mNewLayers.clear();
            update.layerCreationArgs = std::move(mNewLayerArgs);
            mNewLayerArgs.clear();
            update.destroyedHandles = std::move(mDestroyedHandles);
            mDestroyedHandles.clear();
        }

        size_t addedLayers = update.newLayers.size();
        mLayerLifecycleManager.addLayers(std::move(update.newLayers));
        mLayerLifecycleManager.updateDisplayMirrors(mFrontEndDisplayInfos,
                                                    mFrontEndDisplayInfosChanged);
        update.transactions = mTransactionHandler.flushTransactions();
        ftl::Flags<adpf::Workload> committedWorkload;
        for (auto& transaction : update.transactions) {
            committedWorkload |= transaction.workloadHint;
        }
        ...
        mPowerAdvisor->setCommittedWorkload(committedWorkload);
        if (mTransactionTracing) {
            mTransactionTracing->addCommittedTransactions(ftl::to_underlying(vsyncId), frameTimeNs,
                                                          update, mFrontEndDisplayInfos,
                                                          mFrontEndDisplayInfosChanged);
        }
        mLayerLifecycleManager.applyTransactions(update.transactions);
        mLayerLifecycleManager.onHandlesDestroyed(update.destroyedHandles);
        for (auto& legacyLayer : update.legacyLayers) {
            mLegacyLayers[legacyLayer->sequence] = legacyLayer;
        }
        mLayerHierarchyBuilder.update(mLayerLifecycleManager);
    }
```

顺序是刻意安排的,注释里写了三条:

1. `collectTransactions()`先把"已经可以应用"的事务挑出来,这样后面拿着`mStateLock`时不会再触发`onHandleDestroyed`之类的回调。
2. 从`mCreatedLayersLock`下取走新建Layer、新建Layer参数、销毁句柄,防止"事务引用了还在创建队列里的Layer"。
3. 必须先把新Layer加进`LayerLifecycleManager`,再`flushTransactions`,因为事务可能引用刚创建的Layer。

`flushTransactions`之后,事务依次经过`LayerLifecycleManager::applyTransactions`,然后由`LayerHierarchyBuilder::update`重建Layer树。`LayerHierarchyBuilder`负责把扁平的Layer列表按父子关系组织成树,后续的snapshot和可见性计算都基于这棵树。

**第二段:应用显示器事务并生成snapshot**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // Keep a copy of the drawing state (that is going to be overwritten
    // by commitTransactionsLocked) outside of mStateLock so that the side
    // effects of the State assignment don't happen with mStateLock held,
    // which can cause deadlocks.
    State drawingState(mDrawingState);
    Mutex::Autolock lock(mStateLock);
    bool mustComposite = false;
    mustComposite |= applyAndCommitDisplayTransactionStatesLocked(update.transactions);

    frontend::LayerSnapshotBuilder::Args
            args{.root = mLayerHierarchyBuilder.getHierarchy(),
                 .layerLifecycleManager = mLayerLifecycleManager,
                 .forceUpdate = FlagManager::getInstance().frontend_caching_v0()
                         ? frontend::LayerSnapshotBuilder::ForceUpdateFlags::ALL
                         : frontend::LayerSnapshotBuilder::ForceUpdateFlags::NONE,
                 .includeMetadata = mCompositionEngine->getFeatureFlags().test(
                         compositionengine::Feature::kSnapshotLayerMetadata),
                 .displays = mFrontEndDisplayInfos,
                 .displayChanges = mFrontEndDisplayInfosChanged,
                 .globalShadowSettings = mDrawingState.globalShadowSettings,
                 .supportsBlur = mSupportsBlur,
                 .forceFullDamage = mForceFullDamage,
                 .supportedLayerGenericMetadata =
                         getHwComposer().getSupportedLayerGenericMetadata(),
                 .genericLayerMetadataKeyMap = getGenericLayerMetadataKeyMap(),
                 .skipRoundCornersWhenProtected = !getRenderEngine().supportsProtectedContent(),
                 .mergeableHierarchyManager = FlagManager::getInstance().frontend_caching_v0()
                         ? &mMergeableHierarchyManager
                         : nullptr,
                 .renderResourceCache = mIpcCache.get(),
                 .shaderRegistry = mShaderRegistry.get()};
```

这里先复制一份`mDrawingState`到局部变量`drawingState`。注释解释了原因:`State`的赋值有副作用,在`mStateLock`下做可能死锁,所以放到锁外。

`LayerSnapshotBuilder::Args`是一只"大参数包",把这一帧snapshot需要知道的所有上下文都传进去:根节点、生命周期管理器、是否强制全量更新、显示器信息、全局阴影参数、是否支持blur、是否强制全量damage、HWC支持的通用metadata、资源缓存、shader注册表等。

当`frontend_caching_v0`开启时,还会走`MergeableHierarchyManager`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (FlagManager::getInstance().frontend_caching_v0()) {
        {
            SFTRACE_NAME("MergeableHierarchyManager::update");
            mMergeableHierarchyManager.update(mLayerHierarchyBuilder.getHierarchy());
        }
        {
            SFTRACE_NAME("MergeableHierarchyManager::constructSnapshots");
            std::unique_ptr<compositionengine::CompositionEngine> compositionEngine =
                    mFactory.createCompositionEngine();
            compositionEngine->setRenderEngine(mRenderEngine.get());
            compositionEngine->setHwComposer(mHWComposer.get());

            mMergeableHierarchyManager.constructSnapshots(mLayerSnapshotBuilder, args,
                                                          *compositionEngine, mLegacyLayers);
        }
    }
    {
        SFTRACE_NAME("LayerSnapshotBuilder:update");
        mLayerSnapshotBuilder.update(args);
    }
```

`mLayerSnapshotBuilder.update(args)`完成后,每个前端Layer都有一份`frontend::LayerSnapshot`,里面是这一帧的合成参数(位置、裁剪、变换、dataspace、buffer、透明度、圆角、阴影等)。`LayerSnapshot`继承`LayerFECompositionState`,所以CompositionEngine可以直接把它当作`LayerFE`的合成状态来读。

**第三段:根据全局变化打脏标记**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::Geometry | Changes::Input |
                                                      Changes::Hierarchy | Changes::Visibility)) {
        mUpdateInputInfo = true;
    }
    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::VisibleRegion | Changes::Hierarchy |
                                                      Changes::Visibility | Changes::Geometry) ||
        FlagManager::getInstance().frontend_caching_v0()) {
        mVisibleRegionsDirty = true;
    }
    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::Hierarchy | Changes::FrameRate)) {
        // The frame rate of attached choreographers can only change as a result of a
        // FrameRate change (including when Hierarchy changes).
        mUpdateAttachedChoreographer = true;
    }
    outTransactionsAreEmpty = mLayerLifecycleManager.getGlobalChanges().get() == 0;
    mustComposite |= mLayerLifecycleManager.getGlobalChanges().any(
            frontend::RequestedLayerState::kMustComposite);
```

| 变化类型 | 后果 |
|---|---|
| `Geometry`/`Input`/`Hierarchy`/`Visibility` | `mUpdateInputInfo = true`,下一帧要重新计算input窗口 |
| `VisibleRegion`/`Hierarchy`/`Visibility`/`Geometry` | `mVisibleRegionsDirty = true`,CompositionEngine需要重建layer stack和可见区域 |
| `Hierarchy`/`FrameRate` | `mUpdateAttachedChoreographer = true`,要通知挂接的Choreographer |
| 任意全局变化 | `outTransactionsAreEmpty = false`,commit回调不能提前发 |
| `kMustComposite`集合 | `mustComposite = true` |

`outTransactionsAreEmpty`会决定commit回调的发送时机(见3.6)。

**第四段:应用事务并latch buffer**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    bool newDataLatched = false;
    SFTRACE_NAME("DisplayCallbackAndStatsUpdates");
    mustComposite |= applyTransactionsLocked(update.transactions);
    traverseLegacyLayers([&](Layer* layer) { layer->commitTransaction(); });
    const nsecs_t latchTime = systemTime();
    bool unused = false;

    for (auto& layer : mLayerLifecycleManager.getLayers()) {
        if (layer->changes.test(frontend::RequestedLayerState::Changes::Created) &&
            layer->bgColorLayer) {
            sp<Layer> bgColorLayer = getFactory().createLayer(
                    LayerCreationArgs(this, nullptr, layer->name,
                                      ISurfaceComposerClient::eFXSurfaceEffect, LayerMetadata(),
                                      std::make_optional(layer->id), true));
            mLegacyLayers[bgColorLayer->sequence] = bgColorLayer;
        }
        const bool willReleaseBufferOnLatch = layer->willReleaseBufferOnLatch();

        auto it = mLegacyLayers.find(layer->id);
        if (it == mLegacyLayers.end() &&
            layer->changes.test(frontend::RequestedLayerState::Changes::Destroyed)) {
            // Layer handle was created and immediately destroyed. It was destroyed before it
            // was added to the map.
            continue;
        }

        LLOG_ALWAYS_FATAL_WITH_TRACE_IF(it == mLegacyLayers.end(),
                                        "Couldnt find layer object for %s",
                                        layer->getDebugString().c_str());
        if (!layer->hasReadyFrame() && !willReleaseBufferOnLatch) {
            if (!it->second->hasBuffer()) {
                // The last latch time is used to classify a missed frame as buffer stuffing
                // instead of a missed frame. This is used to identify scenarios where we
                // could not latch a buffer or apply a transaction due to backpressure.
                // We only update the latch time for bufferless layers here, the latch time
                // is updated for buffer layers when the buffer is latched.
                it->second->updateFrameTimelinePastTimestamps({
                        .latchTime = latchTime,
                        .expectedPresentTime = expectedPresentTimeNs,
                });
            }
            continue;
        }

        const bool bgColorOnly =
                !layer->externalTexture && (layer->bgColorLayerId != UNASSIGNED_LAYER_ID);
        if (willReleaseBufferOnLatch) {
            mLayersWithBuffersRemoved.emplace(it->second);
        }
        it->second->latchBufferImpl(unused, latchTime, expectedPresentTimeNs, bgColorOnly);
        newDataLatched = true;

        frontend::LayerSnapshot* snapshot = mLayerSnapshotBuilder.getSnapshot(it->second->sequence);
        gui::GameMode gameMode = (snapshot) ? snapshot->gameMode : gui::GameMode::Unsupported;
        mLayersWithQueuedFrames.emplace(it->second, gameMode);
    }
```

逐点说明:

- `applyTransactionsLocked(update.transactions)`把剩余需要加锁的部分应用掉;如果这一帧有几何、可见性之类的变化,返回true。
- `traverseLegacyLayers(... layer->commitTransaction())`把每个`Layer`的pending state推进到drawing state。这一步之后,`mDrawingState.buffer`才是本帧要用的buffer。
- 遍历`LayerLifecycleManager`里的每个前端Layer,找到对应的legacy `Layer`对象。
- 如果前端Layer是背景色Layer且刚创建,额外创建一个`eFXSurfaceEffect`类型的`Layer`来承载背景色。
- `hasReadyFrame()`为false且不需要释放buffer,说明这一帧没有新内容,跳过latch。但如果这个Layer还没有buffer,仍然要调用`updateFrameTimelinePastTimestamps`更新latch时间,避免FrameTimeline把"没有buffer可latch"误判成丢帧。
- `bgColorOnly`表示只合成背景色,不需要真实buffer。
- `willReleaseBufferOnLatch()`为true时,把这个Layer加入`mLayersWithBuffersRemoved`,后续在`onCompositionPresented`里专门处理它的release回调。
- `latchBufferImpl`真正把buffer推进到`mBufferInfo`。
- latch成功后,`newDataLatched = true`,并把Layer加入`mLayersWithQueuedFrames`,后续composite会为它准备release fence future。

**第五段:更新历史、脏区域、提交前端变化**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    updateLayerHistory(latchTime);
    mLayerSnapshotBuilder.forEachSnapshot([&](const frontend::LayerSnapshot& snapshot) {
        // update output's dirty region if a snapshot is visible and its
        // content is dirty or if a snapshot recently became invisible
        if ((snapshot.isVisible && snapshot.contentDirty) ||
            (!snapshot.isVisible && snapshot.changes.test(Changes::Visibility))) {
            Region visibleReg;
            visibleReg.set(snapshot.transformedBoundsWithoutTransparentRegion);
            invalidateLayerStack(snapshot.outputFilter, visibleReg);
        }
    });

    for (auto& destroyedLayer : mLayerLifecycleManager.getDestroyedLayers()) {
        mLegacyLayers.erase(destroyedLayer->id);
    }

    {
        SFTRACE_NAME("LayerLifecycleManager:commitChanges");
        mLayerLifecycleManager.commitChanges();
    }

    // enter boot animation on first buffer latch
    if (CC_UNLIKELY(mBootStage == BootStage::BOOTLOADER && newDataLatched)) {
        ALOGI("Enter boot animation");
        mBootStage = BootStage::BOOTANIMATION;
    }

    mustComposite |= (getTransactionFlags() & ~eTransactionFlushNeeded) || newDataLatched;
    if (mustComposite) {
        commitTransactions();
    }

    return mustComposite;
}
```

- `updateLayerHistory(latchTime)`把每个Layer的帧历史更新给Scheduler的`LayerHistory`,后续`chooseRefreshRateForContent`据此挑选刷新率。
- `forEachSnapshot`按snapshot的可见性和内容脏标记去`invalidateLayerStack`,把需要重绘的区域记到对应LayerStack的dirty region上。CompositionEngine的`Output::beginFrame`后面就是读这个dirty region决定是否重算。
- `commitChanges()`清掉`LayerLifecycleManager`里的change标记。
- 第一次latch到buffer时,从`BOOTLOADER`阶段进入`BOOTANIMATION`阶段。
- `mustComposite`再并上两个来源:除`eTransactionFlushNeeded`之外的任何事务标志,以及`newDataLatched`。前者对应"这一帧还有别的脏标记要处理",后者对应"这一帧真的latch到了新buffer"。
- 只要有`mustComposite`,`commitTransactions()`会再跑一次`commitTransactionsLocked`,把所有挂起的`eTransactionMask`事务彻底提交。

### 3.5 latchBufferImpl:buffer进入mBufferInfo

`Layer::latchBufferImpl`是commit阶段唯一把应用buffer"接进来"的地方:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
bool Layer::latchBufferImpl(bool& recomputeVisibleRegions, nsecs_t latchTime,
                            nsecs_t expectedPresentTime, bool bgColorOnly)
        REQUIRES(mFlinger->mStateLock) {
    SFTRACE_FORMAT_INSTANT("latchBuffer %s - %" PRIu64, getDebugName(),
                           getDrawingState().frameNumber);

    bool refreshRequired = latchSidebandStream(recomputeVisibleRegions);

    if (refreshRequired) {
        return refreshRequired;
    }

    // If the head buffer's acquire fence hasn't signaled yet, return and
    // try again later
    if (mDrawingState.buffer && !fenceHasSignaled()) {
        SFTRACE_NAME("!fenceHasSignaled()");
        mFlinger->onLayerUpdate();
        return false;
    }
    latchBufferStatsAndHandles(latchTime, expectedPresentTime, bgColorOnly);

    // Capture the old state of the layer for comparisons later
    BufferInfo oldBufferInfo = mBufferInfo;
    mPreviousFrameNumber = mCurrentFrameNumber;
    mCurrentFrameNumber = mDrawingState.frameNumber;
    gatherBufferInfo();

    if (mBufferInfo.mBuffer) {
        // We latched a buffer that will be presented soon. Clear the previously presented layer
        // stack list.
        mPreviouslyPresentedLayerStacks.clear();
    }

    if (mDrawingState.buffer == nullptr) {
        const bool bufferReleased = oldBufferInfo.mBuffer != nullptr;
        recomputeVisibleRegions = bufferReleased;
        return bufferReleased;
    }

    if (oldBufferInfo.mBuffer == nullptr) {
        // the first time we receive a buffer, we need to trigger a
        // geometry invalidation.
        recomputeVisibleRegions = true;
    }

    if ((mBufferInfo.mCrop != oldBufferInfo.mCrop) ||
        (mBufferInfo.mTransform != oldBufferInfo.mTransform) ||
        (mBufferInfo.mTransformToDisplayInverse != oldBufferInfo.mTransformToDisplayInverse)) {
        recomputeVisibleRegions = true;
    }

    if (oldBufferInfo.mBuffer != nullptr) {
        uint32_t bufWidth = mBufferInfo.mBuffer->getWidth();
        uint32_t bufHeight = mBufferInfo.mBuffer->getHeight();
        if (bufWidth != oldBufferInfo.mBuffer->getWidth() ||
            bufHeight != oldBufferInfo.mBuffer->getHeight()) {
            recomputeVisibleRegions = true;
        }
    }
    return true;
}
```

逐个环节:

1. **sideband stream**:如果Layer用的是sideband stream(例如视频层直接走硬件通路),`latchSidebandStream`会更新`mSidebandStream`,并返回true表示"这一帧只需处理sideband"。
2. **acquire fence检查**:如果drawing state里有buffer,但acquire fence还没signal,就调用`mFlinger->onLayerUpdate()`安排下一次尝试,并返回false。这就是"buffer还没绘制完成,不能拿来做合成"。
3. **`latchBufferStatsAndHandles`**:记录TimeStats和FrameTracer,并把`OnCommit`回调从drawing state的`callbackHandles`里挑出来提前发。
4. **`gatherBufferInfo`**:把`mDrawingState.buffer`提升为`mBufferInfo`。
5. **返回值的含义**:`recomputeVisibleRegions`告诉调用方是否需要重算可见区域。首次拿到buffer、crop/transform变化、buffer尺寸变化都会置true。

`fenceHasSignaled`的实现值得注意,因为它受一个全局配置控制:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
bool Layer::fenceHasSignaled() const {
    if (SurfaceFlinger::enableLatchUnsignaledConfig != LatchUnsignaledConfig::Disabled) {
        return true;
    }

    const bool fenceSignaled =
            getDrawingState().acquireFence->getStatus() == Fence::Status::Signaled;
    if (!fenceSignaled) {
        mFlinger->mTimeStats->incrementLatchSkipped(getSequence(),
                                                    TimeStats::LatchSkipReason::LateAcquire);
    }

    return fenceSignaled;
}
```

当`enableLatchUnsignaledConfig`不是`Disabled`时,即使acquire fence没signal也允许latch。这就是`latch-unsignaled`优化:允许SF在GPU还没画完时先把buffer接进来,提前做合成准备,最后的HWC present fence会自然地把两者同步起来。

`latchBufferStatsAndHandles`里最关键的是这两段:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
    const auto acquireFence = std::make_shared<FenceTime>(mDrawingState.acquireFence);
    // TODO(b/485971052):
    // For OOPR the fence tracing is going to produce SIGNAL_TIME_INVALID. This may
    // cause issues
    mFlinger->mTimeStats->setAcquireFence(layerId, frameNumber, acquireFence);
    mFlinger->mTimeStats->setLatchTime(layerId, frameNumber, latchTime);

    mFlinger->mFrameTracer->traceFence(layerId, bufferId, frameNumber, acquireFence,
                                       FrameTracer::FrameEvent::ACQUIRE_FENCE);
    mFlinger->mFrameTracer->traceTimestamp(layerId, bufferId, frameNumber, latchTime,
                                           FrameTracer::FrameEvent::LATCH);
```

以及:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
    mFlinger->getTransactionCallbackInvoker().addOnCommitCallbackHandles(
            mDrawingState.callbackHandles);
```

`addOnCommitCallbackHandles`把只关心`onCommit`的回调提前拿走,剩下的(需要release fence的)留在`callbackHandles`里,等present之后再处理。这样应用可以尽早收到"事务已提交"的通知,而不必等上屏。

`gatherBufferInfo`的核心:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::gatherBufferInfo() {
    mPreviousReleaseCallbackId = {getLatchedBufferId(), mBufferInfo.mFrameNumber};
    mPreviousReleaseBufferEndpoint = mBufferInfo.mReleaseBufferEndpoint;
    if (!mDrawingState.buffer) {
        if (mDrawingState.hasRenderCommandBufferFrameId) {
            gatherBufferInfoRenderCommandBuffer();
            return;
        }
        mBufferInfo = {};
        return;
    }

    if ((!mBufferInfo.mBuffer || !mDrawingState.buffer->hasSameBuffer(*mBufferInfo.mBuffer))) {
        decrementPendingBufferCount();
    }

    mBufferInfo.mBuffer = mDrawingState.buffer;
    mBufferInfo.mReleaseBufferEndpoint = mDrawingState.releaseBufferEndpoint;
    mBufferInfo.mFence = mDrawingState.acquireFence;
    mBufferInfo.mFrameNumber = mDrawingState.frameNumber;
    mBufferInfo.mPixelFormat =
            !mBufferInfo.mBuffer ? PIXEL_FORMAT_NONE : mBufferInfo.mBuffer->getPixelFormat();
    ...
}
```

- `mPreviousReleaseCallbackId`保存上一帧的buffer id和帧号,这一对就是应用侧`releaseBufferCallback`的key。
- `decrementPendingBufferCount()`在换新buffer时把`mPendingBuffers`减一,并调用`tracePendingBufferCount`写一条trace:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::decrementPendingBufferCount() {
    int32_t pendingBuffers = --mPendingBuffers;
    tracePendingBufferCount(pendingBuffers);
}

void Layer::tracePendingBufferCount(int32_t pendingBuffers) {
    SFTRACE_INT(mBlastTransactionName.c_str(), pendingBuffers);
}
```

`mPendingBuffers`是SF侧统计的"已经latch、还没把release回调送回去"的buffer数量。这个计数器在创建Layer时通过`getPendingBufferCounter()`交给`SurfaceFlinger::mBufferCountTracker`,并作为`args.pendingBuffers`参与Layer创建,用于atrace和调试,不是BufferQueue的`maxDequeuedBufferCount`配额。

- `mBufferInfo.mBuffer`指向`mDrawingState.buffer`,注意这里没有复制buffer,只是把`sp<GraphicBuffer>`的引用提升上来。
- `mBufferInfo.mFence`保存acquire fence。
- `mBufferInfo.mFrameNumber`保存帧号,后续FrameTimeline和TimeStats都用它。
- 之后还会设置dataspace、crop、transform、HDR标志等,这些字段最终会被`LayerSnapshot`读取,转成`LayerFECompositionState`。

到这一步为止,commit的职责就完成了:**没有发生任何GPU或HWC工作**。commit产出的是"一份可以拿去合成的状态"。

### 3.6 commit的后半段

`updateLayerSnapshots`返回后,commit还有几件事要做。

**通知VsyncTracker**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
        // A display transaction for hotplug reconnect may have just destroyed the original.
        pacesetterFrameTargetPtr = mScheduler->pacesetterFrameTarget();

        // Tell VsyncTracker that we are going to present this frame before scheduling
        // setTransactionFlags which will schedule another SF frame. This was if the tracker
        // needs to adjust the vsync timeline, it will be done before the next frame.
        if (mustComposite) {
            mScheduler->getVsyncSchedule()
                    ->getTracker()
                    .onFrameBegin(pacesetterFrameTargetPtr->expectedPresentTime(),
                                  pacesetterFrameTargetPtr->lastSignaledFrameTime());
        }
        if (transactionFlushNeeded()) {
            setTransactionFlags(eTransactionFlushNeeded);
        }
```

`pacesetterFrameTargetPtr`要重新取一次,因为热插拔可能在commit过程中换掉pacesetter。注释明确说了顺序:必须在`setTransactionFlags`再次调度下一帧之前通知`VsyncTracker`,这样tracker对vsync时间线的修正才能作用于下一帧。

**发送commit回调**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
        // This has to be called after latchBuffers because we want to include the layers that have
        // been latched in the commit callback
        if (transactionsAreEmpty) {
            // Invoke empty transaction callbacks early.
            mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
        } else {
            // Invoke OnCommit callbacks.
            mTransactionCallbackInvoker.sendCallbacks(true /* onCommitOnly */);
        }
```

`transactionsAreEmpty`来自`mLayerLifecycleManager.getGlobalChanges().get() == 0`。如果这一帧没有任何状态变化,就把所有还没发的回调一次性发掉;否则只发`onCommit`类型的回调,需要release fence的回调留到present之后。

**选择刷新率**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // Layers need to get updated (in the previous line) before we can use them for
    // choosing the refresh rate.
    // Hold mStateLock as chooseRefreshRateForContent promotes wp<Layer> to sp<Layer>
    // and may eventually call to ~Layer() if it holds the last reference
    {
        SFTRACE_NAME_FOR_TRACK(WorkloadTracer::TRACK_NAME, "Refresh Rate Selection");
        bool updateAttachedChoreographer = mUpdateAttachedChoreographer;
        mUpdateAttachedChoreographer = false;

        MODE_TRANSITION_LOCK();
        mScheduler->chooseRefreshRateForContent(&mLayerHierarchyBuilder.getHierarchy(),
                                                updateAttachedChoreographer);

        initiateDisplayModeChanges();

        // A resolution change without refresh required may have just destroyed the original.
        if (!shouldSyncResolutionSwitch()) {
            pacesetterFrameTargetPtr = mScheduler->pacesetterFrameTarget();
        }
    }
```

`chooseRefreshRateForContent`必须在Layer状态更新之后调用,因为它要看本帧的Layer内容来决定刷新率。`initiateDisplayModeChanges`可能会触发模式切换,模式切换会改变pacesetter,所以之后还要再取一次。

**收尾**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    updateCursorAsync();
    if (!mustComposite) {
        updateInputFlinger(vsyncId, pacesetterFrameTargetPtr->frameBeginTime());
    }
    doActiveLayersTracingIfNeeded(false, mVisibleRegionsDirty,
                                  pacesetterFrameTargetPtr->frameBeginTime(), vsyncId);

    mLastCommittedVsyncId = vsyncId;

    persistDisplayBrightness(mustComposite);

    return mustComposite && CC_LIKELY(mBootStage != BootStage::BOOTLOADER);
```

- `updateCursorAsync`把硬件光标的位置异步写给HWC。
- 如果这一帧不合成,就在这里顺手更新input flinger;如果要合成,input更新留到composite之后,因为composite会改变哪些Layer可见。
- `mLastCommittedVsyncId`记录最后一次commit的vsync id,供后续查询。
- `persistDisplayBrightness(mustComposite)`在合成前把亮度写下去。
- 最终返回值是`mustComposite && mBootStage != BOOTLOADER`。在bootloader阶段即使有事务也不合成,避免和开机动画的初始化抢资源。

**mustComposite的完整来源**

| 来源 | 位置 |
|---|---|
| `mMustComposite.exchange(false)`,由`scheduleComposite`设置 | commit开头 |
| `applyAndCommitDisplayTransactionStatesLocked` | updateLayerSnapshots |
| `LayerLifecycleManager`全局变化里的`kMustComposite`集合 | updateLayerSnapshots |
| `applyTransactionsLocked` | updateLayerSnapshots |
| `getTransactionFlags() & ~eTransactionFlushNeeded` | updateLayerSnapshots |
| `newDataLatched`(这一帧latch到了新buffer) | updateLayerSnapshots |

只要其中任何一项为真,commit就返回true,进入composite。

---

## 4 从commit进入composite

commit返回true后,`Scheduler::onFrameSignal`继续往下:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
    // The pacesetter may have changed or been registered anew during commit.
    pacesetterPtr = pacesetterPtrLocked();

    FrameTargeters targeters;
    targeters.try_emplace(pacesetterPtr->displayId, pacesetterPtr->targeterPtr.get());

    for (auto& [id, display] : mDisplays) {
        if (id == pacesetterPtr->displayId) continue;
        if (!ftl::contains(presentableDisplays, id)) continue;

        FrameTargeter& targeter = *display.targeterPtr;
        targeters.try_emplace(id, &targeter);
    }

    if (CC_UNLIKELY(mPacesetterFrameDurationFractionToSkip > 0.f)) {
        const auto period = pacesetterPtr->targeterPtr->target().expectedFrameDuration();
        const auto skipDuration = Duration::fromNs(
                static_cast<nsecs_t>(period.ns() * mPacesetterFrameDurationFractionToSkip));
        SFTRACE_FORMAT("Injecting jank for %f%% of the frame (%" PRId64 " ns)",
                       mPacesetterFrameDurationFractionToSkip * 100, skipDuration.ns());
        std::this_thread::sleep_for(skipDuration);
        mPacesetterFrameDurationFractionToSkip = 0.f;
    }

    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
    compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
    compositor.sample();

    for (const auto& [id, targeter] : targeters) {
        const auto resultOpt = resultsPerDisplay.get(id);
        LOG_ALWAYS_FATAL_IF(!resultOpt);
        targeter->endFrame(*resultOpt);
    }
}
```

关键点:

1. **重新取pacesetter**:commit期间可能发生热插拔或模式切换,pacesetter可能已经变了。注释明确写了"may have changed or been registered anew during commit"。
2. **只把presentableDisplays放进targeters**:2.4节因为follower backpressure被跳过的显示器不会参与composite,也就不会出现在`resultsPerDisplay`里。
3. **调试用的jank注入**:`mPacesetterFrameDurationFractionToSkip`是调试开关,会让主线程主动睡一段时间,用来复现丢帧。生产环境为0。
4. **composite的返回值是每个显示器的`CompositeResultsPerDisplay`**:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/interface/CompositeResult.h
struct CompositeResult {
    CompositionCoverageFlags compositionCoverage;
};

using CompositeResultsPerDisplay = ui::PhysicalDisplayMap<PhysicalDisplayId, CompositeResult>;
```

5. **sample**:`SurfaceFlinger::sample()`把合成完成的时机告诉`RegionSamplingThread`,用于亮度采样(环境光自适应)。它必须在composite之后调用,因为采样的是刚合成出来的画面。
6. **endFrame**:每个targeter记录本帧的`compositionCoverage`,供下一帧`beginFrame`计算`mHwcFrameMissed`/`mGpuFrameMissed`。注意`endFrame`发生在`composite`之后,所以它写的是"刚刚完成的这一帧"的覆盖情况。

`CompositeResult`里的`compositionCoverage`来自`CompositionCoverage`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/include/scheduler/interface/CompositionCoverage.h
// Whether composition was covered by HWC and/or GPU.
enum class CompositionCoverage : std::uint8_t {
    Hwc = 1 << 0,

    // Mutually exclusive: The composition either used the GPU, or reused a buffer that had been
    // composited on the GPU.
    Gpu = 1 << 1,
    GpuReuse = 1 << 2,
};
```

`Hwc`、`Gpu`、`GpuReuse`三个标志合起来描述"这一帧的像素是谁画出来的",6.1节会讲它怎么被填上。

---

## 5 composite阶段

### 5.1 组装CompositionRefreshArgs

`SurfaceFlinger::composite`的第一大段是把所有合成所需信息塞进`compositionengine::CompositionRefreshArgs`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
CompositeResultsPerDisplay SurfaceFlinger::composite(
        PhysicalDisplayId pacesetterId, const scheduler::FrameTargeters& frameTargeters) {
    ...
    compositionengine::CompositionRefreshArgs refreshArgs;
    // adpf load up hint
    refreshArgs.powerCallback = this;
    refreshArgs.outputs.reserve(displays.size());

    std::vector<DisplayId> displayIds;
    for (const auto& [_, display] : displays) {
        displayIds.push_back(display->getId());
        display->tracePowerMode();
    }
    mPowerAdvisor->setDisplays(displayIds);

    refreshArgs.bufferIdsToUncache = std::move(mBufferIdsToUncache);
    refreshArgs.outputColorSetting = mDisplayColorSetting;
    refreshArgs.forceOutputColorMode = mForceColorMode;

    refreshArgs.updatingOutputGeometryThisFrame = mVisibleRegionsDirty;
    refreshArgs.updatingGeometryThisFrame = mGeometryDirty.exchange(false) ||
            mVisibleRegionsDirty || mDrawingState.colorMatrixChanged;
    refreshArgs.internalDisplayRotationFlags = getFrontInternalDisplayRotationFlags();

    if (CC_UNLIKELY(mDrawingState.colorMatrixChanged)) {
        refreshArgs.colorTransformMatrix = mDrawingState.colorMatrix;
        mDrawingState.colorMatrixChanged = false;
    }

    setForcedClientCompositionLayerStacks(refreshArgs);

    // TODO(b/255601557) Update frameInterval per display
    refreshArgs.frameInterval =
            mScheduler->getNextFrameInterval(pacesetterId, pacesetterTarget.expectedPresentTime());
    const auto scheduledFrameResultOpt = mScheduler->getScheduledFrameResult();
    const auto scheduledFrameTimeOpt = scheduledFrameResultOpt
            ? std::optional{scheduledFrameResultOpt->callbackTime}
            : std::nullopt;
    refreshArgs.scheduledFrameTime = scheduledFrameTimeOpt;
    refreshArgs.hasTrustedPresentationListener = mNumTrustedPresentationListeners > 0;
    // Store the present time just before calling to the composition engine so we could notify
    // the scheduler.
    const auto presentTime = systemTime();
    refreshArgs.refreshStartTime = presentTime;
```

逐字段说明:

| 字段 | 来源 | 作用 |
|---|---|---|
| `powerCallback` | `this` | CompositionEngine回调SurfaceFlinger做电源相关操作 |
| `bufferIdsToUncache` | 上一帧累积的`mBufferIdsToUncache` | 让CompositionEngine清掉不再使用的buffer缓存 |
| `outputColorSetting` | `mDisplayColorSetting` | 输出色彩模式 |
| `forceOutputColorMode` | `mForceColorMode` | 强制输出色彩模式(调试用) |
| `updatingOutputGeometryThisFrame` | `mVisibleRegionsDirty` | 本帧是否需要重建layer stack |
| `updatingGeometryThisFrame` | `mGeometryDirty.exchange(false) \|\| mVisibleRegionsDirty \|\| colorMatrixChanged` | 本帧是否需要更新每层的几何 |
| `internalDisplayRotationFlags` | `getFrontInternalDisplayRotationFlags()` | 内屏旋转 |
| `colorTransformMatrix` | `mDrawingState.colorMatrix` | 全局颜色变换矩阵 |
| `forcedClientCompositionLayerStacks` | `setForcedClientCompositionLayerStacks` | 强制走GPU合成的LayerStack集合 |
| `frameInterval` | `mScheduler->getNextFrameInterval(...)` | 本帧预期的vsync间隔,传给HWC |
| `scheduledFrameTime` | `mScheduler->getScheduledFrameResult()`的`callbackTime` | 本帧计划被唤醒的时刻 |
| `hasTrustedPresentationListener` | `mNumTrustedPresentationListeners > 0` | 是否需要计算可信呈现状态 |
| `refreshStartTime` | `systemTime()` | 合成开始时刻,用于FrameTimeline和CompositorTiming |

`setForcedClientCompositionLayerStacks`在以下情况把显示器的LayerStack塞进强制client合成集合:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::setForcedClientCompositionLayerStacks(
        compositionengine::CompositionRefreshArgs& refreshArgs) {
    bool forceAllDisplaysToClientComposition = false;
    if (mDebugDisableHWC) {
        forceAllDisplaysToClientComposition = true;
    }

    if (mDebugFlashDelay != 0) {
        forceAllDisplaysToClientComposition = true;
        refreshArgs.devOptFlashDirtyRegionsDelay = std::chrono::milliseconds(mDebugFlashDelay);
    }

    const auto& displays = FTL_FAKE_GUARD(mStateLock, mDisplays);
    if (forceAllDisplaysToClientComposition) {
        for (const auto& [_, display] : displays) {
            refreshArgs.forcedClientCompositionLayerStacks.insert(display->getLayerStack());
        }
        return;
    }

    if (!FlagManager::getInstance().force_slower_follower_gpu_composition_combined()) {
        return;
    }

    for (const auto& [_, display] : displays) {
        if (display->isVirtual()) {
            // Assume that virtual displays composite at the same rate as the pacesetter.
            continue;
        }

        if (mScheduler->isLockstepFollower(display->getPhysicalId())) {
            // Follower displays in lockstep with pacesetter are allowed to composite on DPU.
            continue;
        }

        refreshArgs.forcedClientCompositionLayerStacks.insert(display->getLayerStack());
    }
}
```

- `mDebugDisableHWC`为真时,所有显示器强制GPU合成。这是`adb shell service call SurfaceFlinger ...`或调试属性常用的手段。
- `mDebugFlashDelay`不为0时,除了强制GPU合成,还设置`devOptFlashDirtyRegionsDelay`,让`Output::devOptRepaintFlash`把脏区域闪烁显示一段时间。
- `force_slower_follower_gpu_composition_combined`开启时,非lockstep的物理follower显示器强制走GPU合成,避免慢显示器上的DPU合成导致pace setter等待。

这个集合最终在`Output::updateCompositionState`里被消费:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
    for (auto* layer : getOutputLayersOrderedByZ()) {
        const ui::LayerStack outputLayerStack =
                layer->getOutput().getState().layerFilter.layerStack;
        const bool layerForceClientComposition =
                refreshArgs.forcedClientCompositionLayerStacks.contains(outputLayerStack);

        layer->updateCompositionState(refreshArgs.updatingGeometryThisFrame,
                                      layerForceClientComposition || forceClientComposition,
                                      refreshArgs.internalDisplayRotationFlags,
                                      properties ? properties->lutProperties : std::nullopt);
        ...
    }
```

### 5.2 addOutputsToRefreshArgs与多显示器划分

`addOutputsToRefreshArgs`负责把物理显示器和虚拟显示器分开:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
std::optional<compositionengine::CompositionRefreshArgs> SurfaceFlinger::addOutputsToRefreshArgs(
        PhysicalDisplayId pacesetterId,
        compositionengine::CompositionRefreshArgs& mainThreadRefreshArgs,
        const scheduler::FrameTargeters& frameTargeters) {
    const auto& pacesetterTarget = frameTargeters.get(pacesetterId)->get()->target();
    const auto& displays = FTL_FAKE_GUARD(mStateLock, mDisplays);
    ...
    bool anyMainThreadClientComposition = false;
    for (const auto& [id, targeter] : frameTargeters) {
        ftl::FakeGuard guard(mStateLock);

        if (const auto display = getCompositionDisplayLocked(id)) {
            const auto layerStack = physicalDisplayLayerStacks.get(id)->get();
            if (isUniqueOutputLayerStack(display->getId(), layerStack)) {
                mainThreadRefreshArgs.outputs.push_back(display);
                if (display->getState().usesClientComposition) {
                    anyMainThreadClientComposition = true;
                }
            }
        }

        mainThreadRefreshArgs.frameTargets.try_emplace(id, &targeter->target());
    }
```

要点:

- 只有出现在`frameTargeters`里的物理显示器才会加入`outputs`。
- `isUniqueOutputLayerStack`去重:如果两个显示器共享同一个LayerStack,只保留第一个,避免同一组Layer被合成两次。被丢弃的显示器会打一条`ALOGD`日志。
- 每个显示器的`FrameTarget`都会放进`refreshArgs.frameTargets`,后续`Output::writeCompositionState`从这里读`earliestPresentTime`和`expectedPresentTime`。

虚拟显示器在同一个函数的后半段处理,而且不是无条件加入`outputs`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // Lazily initialize offloadedRefreshArgs which is a less-commonly used mode.
    std::optional<compositionengine::CompositionRefreshArgs> offloadedRefreshArgs;

    const bool canOffloadGpuComposition =
            FlagManager::getInstance().offload_gpu_composition() && mRenderEngine->isThreaded();
    for (const auto& [_, display] : displays) {
        if (!display->isVirtual()) {
            continue;
        }

        const Fps refreshRate = display->getAdjustedRefreshRate();
        const auto vsyncTime = FlagManager::getInstance().bugfix_virtual_display_refresh_rate()
                ? pacesetterTarget.expectedPresentTime()
                : pacesetterTarget.frameBeginTime();
        if (refreshRate.isValid() && !mScheduler->isVsyncInPhase(vsyncTime, refreshRate)) {
            if (FlagManager::getInstance().bugfix_virtual_display_refresh_rate()) {
                mScheduler->scheduleFrame();
            }
            continue;
        }

        if (!isUniqueOutputLayerStack(display->getId(), display->getLayerStack())) {
            continue;
        }

        // Offload GPU backed display to background thread if none of physical displays use
        // client composition to avoid performance issue.
        if (canOffloadGpuComposition && !anyMainThreadClientComposition &&
            display->isGpuVirtualDisplay()) {
            if (!offloadedRefreshArgs) {
                // Initialize the offloadedRefreshArgs now that it is known to be needed.
                offloadedRefreshArgs = mainThreadRefreshArgs;
                offloadedRefreshArgs->outputs.clear();
                offloadedRefreshArgs->frameTargets.clear();

                // Populate properties for offload thread
                offloadedRefreshArgs->hasTrustedPresentationListener = false;
                offloadedRefreshArgs->bufferIdsToUncache = {};
            }
            offloadedRefreshArgs->outputs.push_back(display->getCompositionDisplay());

        } else {
            mainThreadRefreshArgs.outputs.push_back(display->getCompositionDisplay());
        }
    }

    return offloadedRefreshArgs;
}
```

要真的offload到后台线程,必须同时满足三个条件:

- `canOffloadGpuComposition`:flag`offload_gpu_composition`打开,并且`mRenderEngine->isThreaded()`为真。
- `!anyMainThreadClientComposition`:所有主线程物理显示器都没有走client合成。只要有一块物理屏在用GPU合成,虚拟屏就不再offload,避免和它抢GPU。
- `display->isGpuVirtualDisplay()`:只有GPU合成的虚拟屏才值得offload。

虚拟屏在进入上面这三个判断之前还有两道过滤:

- 刷新率检查:`refreshRate.isValid() && !mScheduler->isVsyncInPhase(vsyncTime, refreshRate)`时直接`continue`。`vsyncTime`取`pacesetterTarget.expectedPresentTime()`还是`frameBeginTime()`取决于`bugfix_virtual_display_refresh_rate`这个flag;走这条跳过路径时,如果flag打开还会调用`mScheduler->scheduleFrame()`再等下一帧。
- `isUniqueOutputLayerStack`:LayerStack已经给别的显示器用过就跳过。

`offloadedRefreshArgs`是`mainThreadRefreshArgs`的拷贝,然后清空`outputs`和`frameTargets`,再把`hasTrustedPresentationListener`改成false、`bufferIdsToUncache`清空。之后虚拟屏的Layer列表也是单独填的,`composite`里会对它再调一次`addLayerSnapshotsToCompositionArgs`和`prepareLayersForComposition`。

回到`composite`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    std::optional<std::future<void>> offloadedCompositionFuture;
    if (optionalOffloadedRefreshArgs) {
        offloadedCompositionFuture =
                offloadGpuCompositedDisplays(std::move(*optionalOffloadedRefreshArgs));
    }

    mCompositionEngine->present(refreshArgs);
```

`offloadGpuCompositedDisplays`把整个`CompositionRefreshArgs`move进lambda,交给`BackgroundExecutor::getInstance().sendCallbacks`,在后台执行`mOffloadedCompositionEngine->present(offloadedRefreshArgs)`;完成时通过`std::promise`兑现future。主线程不等它,先跑自己的`mCompositionEngine->present(refreshArgs)`。等到`composite`末尾才在这里汇合:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (optionalOffloadedRefreshArgs) {
        offloadedCompositionFuture->wait();
        moveSnapshotsFromCompositionArgs(offloadedLayers);
    }
```

`moveSnapshotsFromCompositionArgs`把后台线程用过的snapshot搬回`mLayerSnapshotBuilder`,调用点就排在`offloadedCompositionFuture->wait()`之后,即要等后台的`present`返回才做搬运。

### 5.3 准备Layer列表和release fence future

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    constexpr bool kCursorOnly = false;
    const auto layers = mLayerSnapshotBuilder.hasMergedSnapshots()
            ? copyMergedSnapshots(refreshArgs)
            : addLayerSnapshotsToCompositionArgs(refreshArgs, kCursorOnly);
    prepareLayersForComposition(refreshArgs, kCursorOnly, layers);

    for (auto& [layer, layerFE] : layers) {
        validateForReadback(layerFE);
    }
    setupOutputsForReadback(refreshArgs.outputs);
```

`addLayerSnapshotsToCompositionArgs`把每个`LayerSnapshot`对应的`LayerFE`放进`refreshArgs.layers`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mLayerSnapshotBuilder.forEachNonNullSnapshot(
            [&](std::unique_ptr<frontend::LayerSnapshot>& snapshot) FTL_FAKE_GUARD(
                    kMainThreadContext) {
                if (cursorOnly &&
                    snapshot->compositionType !=
                            aidl::android::hardware::graphics::composer3::Composition::CURSOR) {
                    return;
                }

                if (!snapshot->hasSomethingToDraw()) {
                    return;
                }

                if (std::find(expectedLayerStacks.begin(), expectedLayerStacks.end(),
                              snapshot->outputFilter.layerStack) == expectedLayerStacks.end()) {
                    return;
                }

                auto it = mLegacyLayers.find(snapshot->sequence);
                LLOG_ALWAYS_FATAL_WITH_TRACE_IF(it == mLegacyLayers.end(),
                                                "Couldnt find layer object for %s",
                                                snapshot->getDebugString().c_str());
                auto& legacyLayer = it->second;
                sp<LayerFE> layerFE = legacyLayer->getCompositionEngineLayerFE(snapshot->path);
                snapshot->fps = getLayerFramerate(currentTime, snapshot->sequence);
                layerFE->mSnapshot = std::move(snapshot);
                refreshArgs.layers.push_back(layerFE);
                layers.emplace_back(legacyLayer.get(), layerFE.get());
            },
            [needsMetadata](const frontend::LayerSnapshot& snapshot) {
                return snapshot.isVisible ||
                        (needsMetadata &&
                         snapshot.changes.test(frontend::RequestedLayerState::Changes::Metadata));
            });
```

过滤条件有三条:

1. `kCursorOnly`为true时只要CURSOR类型的层。这是`updateCursorAsync`单独更新光标时用的路径。
2. `hasSomethingToDraw()`为false的层跳过。没有buffer、没有背景色、没有阴影之类的层不需要画。
3. 层的`outputFilter.layerStack`不在本次`refreshArgs.outputs`的LayerStack集合里时跳过。这保证每个Layer只会被送到包含它的显示器。

通过过滤的层会:

- 设置`snapshot->fps`,用于后续帧率决策。
- 把snapshot移动到`layerFE->mSnapshot`。这一步之后,`LayerFE`就可以通过`getCompositionState()`读到这层这一帧的全部合成参数。
- 把`LayerFE`推进`refreshArgs.layers`,把`(Layer*, LayerFE*)`对留在返回的`layers`里。

如果开了merged snapshot模式,则用`copyMergedSnapshots`。

`prepareLayersForComposition`做两件事:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::prepareLayersForComposition(
        compositionengine::CompositionRefreshArgs& refreshArgs, bool kCursorOnly,
        const std::vector<std::pair<Layer*, LayerFE*>>& layers) {
    setVisibleRegionDirtyIfNeeded(refreshArgs);

    for (auto& [layer, layerFE] : layers) {
        layer->onPreComposition(refreshArgs.refreshStartTime);
        attachReleaseFenceFutureToLayer(layer, layerFE,
                                        layerFE->mSnapshot->outputFilter.layerStack);
    }
}
```

- `setVisibleRegionDirtyIfNeeded`:如果`mVisibleRegionsDirty`为真,就把`refreshArgs.updatingOutputGeometryThisFrame`和`refreshArgs.updatingGeometryThisFrame`都置为真并返回;否则遍历所有已启用Output的Z序层,一旦发现某层的`getCompositionState()`是nullptr,就打一条`ALOGE`、调用`TransactionTraceWriter::getInstance().invoke(__func__, /* overwrite= */ false)`抓trace、把`mVisibleRegionsDirty`置true并同步写进这两个字段,然后返回。
- `layer->onPreComposition(refreshStartTime)`把刷新开始时间写进每个`CallbackHandle`,供应用侧CompositorTiming使用。
- `attachReleaseFenceFutureToLayer`为每一层创建一个release fence future。

`attachReleaseFenceFutureToLayer`的实现:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::attachReleaseFenceFutureToLayer(Layer* layer, LayerFE* layerFE,
                                                     ui::LayerStack layerStack) {
    ftl::Future<FenceResult> futureFence = layerFE->createReleaseFenceFuture();
    layer->prepareReleaseCallbacks(std::move(futureFence), layerStack);
}
```

这个future在合成完成后由`LayerFE::setReleaseFence`填充,最终通过`TransactionCallbackInvoker`送回应用。它是整条"buffer什么时候可以复用"链路的起点。

对`mLayersWithQueuedFrames`里的Layer还要额外处理:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    for (auto& [layer, _] : mLayersWithQueuedFrames) {
        if (const auto& layerFE =
                    layer->getCompositionEngineLayerFE({static_cast<uint32_t>(layer->sequence)})) {
            if (!optionalOffloadedRefreshArgs ||
                (layerFE->mSnapshot != nullptr &&
                 layerFE->mSnapshot->outputFilter.layerStack != ui::UNASSIGNED_LAYER_STACK &&
                 mainThreadLayerStacks.count(layerFE->mSnapshot->outputFilter.layerStack.id))) {
                refreshArgs.layersWithQueuedFrames.push_back(layerFE);
            } else {
                optionalOffloadedRefreshArgs->layersWithQueuedFrames.push_back(layerFE);
            }
            // Some layers are not displayed and do not yet have a future release fence
            if (layerFE->getReleaseFencePromiseStatus() ==
                        LayerFE::ReleaseFencePromiseStatus::UNINITIALIZED ||
                layerFE->getReleaseFencePromiseStatus() ==
                        LayerFE::ReleaseFencePromiseStatus::FULFILLED) {
                // layerStack is unassigned because layer is not on a display
                attachReleaseFenceFutureToLayer(layer.get(), layerFE.get(),
                                                ui::UNASSIGNED_LAYER_STACK);
            }
        }
    }
```

这段代码处理一个边界情况:**有buffer但没显示的Layer**。例如一个Layer被完全遮挡了,或者它根本不在任何显示器的LayerStack上。这种Layer不会出现在`refreshArgs.layers`里,但它的buffer仍然必须被释放,否则应用会一直等buffer。所以这里给它挂一个`UNASSIGNED_LAYER_STACK`的future,后面`CompositionEngine::postComposition`会用一个`NO_FENCE`把它兑掉。

### 5.4 CompositionEngine::present的五步

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
void CompositionEngine::present(CompositionRefreshArgs& args) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    preComposition(args);

    {
        // latchedLayers is used to track the set of front-end layer state that
        // has been latched across all outputs for the prepare step, and is not
        // needed for anything else.
        LayerFESet latchedLayers;

        for (const auto& output : args.outputs) {
            output->prepare(args, latchedLayers);
        }
    }

    // Offloading the HWC call for `present` allows us to simultaneously call it
    // on multiple displays. This is desirable because these calls block and can
    // be slow.
    offloadOutputs(args.outputs);

    ui::DisplayVector<ftl::Future<std::monostate>> presentFutures;
    for (const auto& output : args.outputs) {
        presentFutures.push_back(output->present(args));
        if (auto displayId = output->getDisplayId(); displayId) {
            panopticon::terminate(std::to_string(displayId->value));
        }
    }

    {
        SFTRACE_NAME("Waiting on HWC");
        for (auto& future : presentFutures) {
            // TODO(b/185536303): Call ftl::Future::wait() once it exists, since
            // we do not need the return value of get().
            future.get();
        }
    }
    postComposition(args);
}
```

五个步骤:

1. **`preComposition`**:通知每层"合成要开始了",并收集`needsAnotherUpdate`。
2. **`output->prepare`**:重建每个Output的layer stack,清掉不再需要的buffer缓存。
3. **`offloadOutputs`**:决定哪些Output的present调用可以卸载到后台线程。
4. **`output->present`**:对每个Output执行完整的合成流程,返回一个future。
5. **等待所有future,然后`postComposition`**:确保所有release fence promise都被兑现。

`preComposition`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
void CompositionEngine::preComposition(CompositionRefreshArgs& args) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    bool needsAnotherUpdate = false;

    mRefreshStartTime = args.refreshStartTime;

    for (auto& layer : args.layers) {
        if (layer->onPreComposition(args.updatingOutputGeometryThisFrame)) {
            needsAnotherUpdate = true;
        }
    }

    mNeedsAnotherUpdate = needsAnotherUpdate;
}
```

`LayerFE::onPreComposition`如果返回true,说明这一层有需要下一帧继续处理的状态(例如动画还没结束),`mNeedsAnotherUpdate`会被置true,最终导致`SurfaceFlinger::composite`末尾调用`scheduleCommit`。

`Output::prepare`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::prepare(const compositionengine::CompositionRefreshArgs& refreshArgs,
                     LayerFESet& geomSnapshots) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    rebuildLayerStacks(refreshArgs, geomSnapshots);
    uncacheBuffers(refreshArgs.bufferIdsToUncache);
}
```

`rebuildLayerStacks`只在`updatingOutputGeometryThisFrame`为true时才真正重建,否则直接返回:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
    if (!outputState.isEnabled || CC_LIKELY(!refreshArgs.updatingOutputGeometryThisFrame)) {
        return;
    }
    ...
    compositionengine::Output::CoverageState coverage{layerFESet};
    coverage.aboveCoveredLayersExcludingOverlays = refreshArgs.hasTrustedPresentationListener
            ? std::make_optional<Region>()
            : std::nullopt;
    collectVisibleLayers(refreshArgs, coverage);

    // Compute the resulting coverage for this output, and store it for later
    const ui::Transform& tr = outputState.transform;
    Region undefinedRegion{outputState.displaySpace.getBoundsAsRect()};
    undefinedRegion.subtractSelf(tr.transform(coverage.aboveOpaqueLayers));

    outputState.undefinedRegion = undefinedRegion;
    outputState.dirtyRegion.orSelf(coverage.dirtyRegion);
```

`collectVisibleLayers`从**前到后**遍历Layer:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::collectVisibleLayers(const compositionengine::CompositionRefreshArgs& refreshArgs,
                                  compositionengine::Output::CoverageState& coverage) {
    // Evaluate the layers from front to back to determine what is visible. This
    // also incrementally calculates the coverage information for each layer as
    // well as the entire output.
    for (auto layer : reversed(refreshArgs.layers)) {
        // Incrementally process the coverage for each layer
        ensureOutputLayerIfVisible(layer, coverage);

        // TODO(b/121291683): Stop early if the output is completely covered and
        // no more layers could even be visible underneath the ones on top.
    }

    setReleasedLayers(refreshArgs);

    finalizePendingOutputLayers();
}
```

`reversed`是因为`refreshArgs.layers`按Z轴从底到顶排列,而可见性判断需要从顶到底累积"已经被上面遮住的区域"。

`ensureOutputLayerIfVisible`会跳过不满足条件的层:

- `includesLayer(layerFE)`为false(不属于这个Output的LayerStack)。
- `isVisible`为false。
- 可见区域为空。

通过检查的层会创建或更新`OutputLayer`,并计算opaque/visible/covered/transparent/shadow区域,累积到`coverage`里。

### 5.5 Output::present的十一个步骤

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
ftl::Future<std::monostate> Output::present(
        const compositionengine::CompositionRefreshArgs& refreshArgs) {
    ...
    updateColorProfile(refreshArgs);
    updateCompositionState(refreshArgs);
    planComposition();
    writeCompositionState(refreshArgs);
    setColorTransform(refreshArgs);
    beginFrame();

    if (isPowerHintSessionEnabled()) {
        // always reset the flag before the composition prediction
        setHintSessionRequiresRenderEngine(false);
    }
    GpuCompositionResult result;
    const bool predictCompositionStrategy = canPredictCompositionStrategy(refreshArgs);
    if (predictCompositionStrategy) {
        result = prepareFrameAsync();
    } else {
        prepareFrame();
    }

    devOptRepaintFlash(refreshArgs);
    finishFrame(std::move(result));
    ftl::Future<std::monostate> future;
    const bool flushEvenWhenDisabled = !refreshArgs.bufferIdsToUncache.empty();
    if (mOffloadPresent) {
        future = presentFrameAndReleaseLayersAsync(flushEvenWhenDisabled);

        // Only offload for this frame. The next frame will determine whether it
        // needs to be offloaded. Leave the HwcAsyncWorker in place. For one thing,
        // it is currently presenting. Further, it may be needed next frame, and
        // we don't want to churn.
        mOffloadPresent = false;
    } else {
        presentFrameAndReleaseLayers(flushEvenWhenDisabled);
        future = ftl::yield<std::monostate>({});
    }
    renderCachedSets(refreshArgs);
    return future;
}
```

按顺序拆:

**第一步:updateColorProfile**

根据`refreshArgs.outputColorSetting`和`forceOutputColorMode`设置`DisplayColorProfile`。如果色彩模式有变化,还会把整个Output标脏。

**第二步:updateCompositionState**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::updateCompositionState(const compositionengine::CompositionRefreshArgs& refreshArgs) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        return;
    }

    mLayerRequestingBackgroundBlur = findLayerRequestingBackgroundComposition();
    bool forceClientComposition = mLayerRequestingBackgroundBlur != nullptr;

    auto* properties = getOverlaySupport();

    for (auto* layer : getOutputLayersOrderedByZ()) {
        const ui::LayerStack outputLayerStack =
                layer->getOutput().getState().layerFilter.layerStack;
        const bool layerForceClientComposition =
                refreshArgs.forcedClientCompositionLayerStacks.contains(outputLayerStack);

        layer->updateCompositionState(refreshArgs.updatingGeometryThisFrame,
                                      layerForceClientComposition || forceClientComposition,
                                      refreshArgs.internalDisplayRotationFlags,
                                      properties ? properties->lutProperties : std::nullopt);

        if (mLayerRequestingBackgroundBlur == layer) {
            forceClientComposition = false;
        }
    }
    commitPictureProfilesToCompositionState();
}
```

这里有个方向问题必须先钉死:`getOutputLayersOrderedByZ()`到底是从上往下还是从下往上?不先确定这一点,上面`forceClientComposition`的开关逻辑就无从解释。

`Output`是抽象接口,真正的枚举落在`impl::Output`里:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/Output.h
        OutputLayer* getOutputLayerOrderedByZByIndex(size_t index) const override {
            if (index >= mCurrentOutputLayersOrderedByZ.size()) {
                return nullptr;
            }
            return mCurrentOutputLayersOrderedByZ[index].get();
        }
```

`mCurrentOutputLayersOrderedByZ`的填法:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/Output.h
        void finalizePendingOutputLayers() override {
            // The pending layers are added in reverse order. Reverse them to
            // get the back-to-front ordered list of layers.
            std::reverse(mPendingOutputLayersOrderedByZ.begin(),
                         mPendingOutputLayersOrderedByZ.end());

            mCurrentOutputLayersOrderedByZ = std::move(mPendingOutputLayersOrderedByZ);
        }
```

pending列表是在`collectVisibleLayers`里一层一层`emplace_back`进去的,而这个函数是从前往后遍历的:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::collectVisibleLayers(const compositionengine::CompositionRefreshArgs& refreshArgs,
                                  compositionengine::Output::CoverageState& coverage) {
    // Evaluate the layers from front to back to determine what is visible. This
    // also incrementally calculates the coverage information for each layer as
    // well as the entire output.
    for (auto layer : reversed(refreshArgs.layers)) {
        // Incrementally process the coverage for each layer
        ensureOutputLayerIfVisible(layer, coverage);

        // TODO(b/121291683): Stop early if the output is completely covered and
        // no more layers could even be visible underneath the ones on top.
    }

    setReleasedLayers(refreshArgs);

    finalizePendingOutputLayers();
}
```

`reversed`是`Output.cpp`里本地定义的小helper,把容器从后往前枚举。`refreshArgs.layers`本身是按`globalZ`从小到大排的,也就是最底层在前。这个顺序来自`SurfaceFlinger::addLayerSnapshotsToCompositionArgs`里的遍历:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mLayerSnapshotBuilder.forEachNonNullSnapshot(
            [&](std::unique_ptr<frontend::LayerSnapshot>& snapshot) FTL_FAKE_GUARD(
                    kMainThreadContext) {
                ...
                refreshArgs.layers.push_back(layerFE);
```

而`forEachNonNullSnapshot`就是按下标从小到大走:

```cpp
// frameworks/native/services/surfaceflinger/FrontEnd/LayerSnapshotBuilder.cpp
void LayerSnapshotBuilder::forEachNonNullSnapshot(const Visitor& visitor,
                                                  const ConstPredicate& predicate) {
    for (int i = 0; i < mNumInterestingSnapshots; i++) {
        std::unique_ptr<frontend::LayerSnapshot>& snapshot = mSnapshots.at((size_t)i);
        if (!snapshot || !predicate(*snapshot)) continue;
        visitor(snapshot);
    }
}
```

下标就是`globalZ`,而`globalZ`由`LayerHierarchy::traverseInZOrder`按Z值从小到大分配,`layerZCompare`也是`lhsLayer->z < rhsLayer->z`,所以`refreshArgs.layers`是自下而上。走merged snapshot那条路时,`copyMergedSnapshots`用`forEachMergedSnapshot`按下标遍历`mMergedSnapshots`,而这个数组同样是在`traverseInZOrder`里按Z序`emplace_back`出来的,顺序一致。

`collectVisibleLayers`用`reversed`把它翻转成自上而下,于是pending列表里先塞进去的是最上面的层;`finalizePendingOutputLayers`再整体反转一次,最终`mCurrentOutputLayersOrderedByZ[0]`是最底层,索引越大越靠上。注释里"back-to-front ordered list"说的就是这个。

所以`for (auto* layer : getOutputLayersOrderedByZ())`是自下而上走。`forceClientComposition`初值为"存在请求模糊的层",每处理完一层就检查是不是模糊层,到了就把开关复位成false。于是最底层到模糊层(含模糊层本身)都拿到true,模糊层之上拿到false。

那个"请求模糊的层"是`findLayerRequestingBackgroundComposition`选出来的:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
compositionengine::OutputLayer* Output::findLayerRequestingBackgroundComposition() const {
    compositionengine::OutputLayer* layerRequestingBgComposition = nullptr;
    for (auto* layer : getOutputLayersOrderedByZ()) {
        const auto* compState = layer->getLayerFE().getCompositionState();

        // If any layer has a sideband stream, we will disable blurs. In that case, we don't
        // want to force client composition because of the blur.
        if (compState->sidebandStream != nullptr) {
            return nullptr;
        }

        // If RenderEngine cannot render protected content, we cannot blur.
        if (compState->hasProtectedContent &&
            !getCompositionEngine().getRenderEngine().supportsProtectedContent()) {
            return nullptr;
        }
        if (compState->isOpaque) {
            continue;
        }
        if (compState->backgroundBlurRadius > 0 || compState->blurRegions.size() > 0 ||
            compState->isTextureSamplingBehind) {
            layerRequestingBgComposition = layer;
        }
    }
    return layerRequestingBgComposition;
}
```

它也是自下而上扫:遇到`compState->isOpaque`为true的层直接`continue`,不参与选择;遇到带sideband stream的层,或者带受保护内容而RenderEngine不支持受保护内容的层,直接返回`nullptr`,整屏都不因为模糊而强制client合成;其余层里,凡是`backgroundBlurRadius > 0`、`blurRegions`非空或`isTextureSamplingBehind`为true的,就记到`layerRequestingBgComposition`里继续往上走,循环结束时返回的是最后记下的那一层,也就是最上面那个请求背景模糊的层。

- 如果某一层请求背景模糊,那么从最底层开始一直到这一层(含这一层本身)都必须走GPU合成,因为模糊需要RenderEngine。对应到代码就是:循环开始前`forceClientComposition`已经置为true,自下而上每处理完一层就检查一次,走到模糊层那一次之后才复位成false,所以模糊层之上的层不再受这个开关影响。
- 每层调用`OutputLayer::updateCompositionState`,计算displayFrame、sourceCrop、bufferTransform,判断是否需要强制client合成(不安全内容、旋转、圆角、背景模糊等),设置dataspace、HDR、dimming。
- `commitPictureProfilesToCompositionState`把picture profile(LUT)写进合成状态。

`OutputLayer::updateCompositionState`中决定强制client合成的关键代码:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
    if (layerFEState->forceClientComposition || !profile.isDataspaceSupported(state.dataspace) ||
        forceClientComposition) {
        state.forceClientComposition = true;
    }
```

**第三步:planComposition**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::planComposition() {
    if (!mPlanner || !getState().isEnabled) {
        return;
    }

    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    mPlanner->plan(getOutputLayersOrderedByZ());
}
```

`Planner`负责:
- 记录上一帧的Layer集合和几何状态。
- 计算`NonBufferHash`,判断哪些Layer的内容没变。
- 通过`Flattener`把内容不变的Layer对合并成cached set,减少HWC层的数量。
- 运行`Predictor`预测合成策略。

`Planner::plan`的结果会影响后面`writeCompositionState`里哪些层被跳过、哪些层使用override buffer。

**第四步:writeCompositionState**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::writeCompositionState(const compositionengine::CompositionRefreshArgs& refreshArgs) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        return;
    }

    if (auto frameTargetPtrOpt = getDisplayIdVariant()
                                         .and_then(asPhysicalDisplayId)
                                         .and_then([&refreshArgs](PhysicalDisplayId id) {
                                             return refreshArgs.frameTargets.get(id);
                                         })) {
        editState().earliestPresentTime = frameTargetPtrOpt->get()->earliestPresentTime();
        editState().expectedPresentTime = frameTargetPtrOpt->get()->expectedPresentTime().ns();
        const auto debugPresentDelay = frameTargetPtrOpt->get()->debugPresentDelay();
        if (debugPresentDelay) {
            SFTRACE_FORMAT_INSTANT("DEBUG delaying presentation by %.2fms",
                                   debugPresentDelay->ns() / 1e6f);
            editState().expectedPresentTime += debugPresentDelay->ns();
        }
    }
    editState().frameInterval = refreshArgs.frameInterval;
    editState().powerCallback = refreshArgs.powerCallback;

    applyPictureProfile();
    ...
}
```

这里把`FrameTarget`里的`earliestPresentTime`和`expectedPresentTime`复制到Output状态,后续`Display::presentFrame`会把它传给HWC。`debugPresentDelay`是调试用的present延迟开关。

随后遍历所有OutputLayer,调用`writeStateToHWC`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
        layer->writeStateToHWC(includeGeometry, skipLayer, z++, overrideZ, isPeekingThrough,
                               hasLutsProperties);
```

`OutputLayer::writeStateToHWC`的关键逻辑:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
    auto requestedCompositionType = outputIndependentState->compositionType;
    if (requestedCompositionType == Composition::SOLID_COLOR && state.overrideInfo.buffer) {
        requestedCompositionType = Composition::DEVICE;
    }

    const bool isOverridden =
            state.overrideInfo.buffer != nullptr || isPeekingThrough || zIsOverridden;
    ...
    writeCompositionTypeToHWC(hwcLayer.get(), requestedCompositionType, isPeekingThrough, skipLayer);
```

`writeCompositionTypeToHWC`里还要再判断一次是否强制client合成:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
    if (isClientCompositionForced(isPeekingThrough)) {
        requestedCompositionType = Composition::CLIENT;
    }
    ...
```

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
bool OutputLayer::isClientCompositionForced(bool isPeekingThrough) const {
    return getState().forceClientComposition ||
            (!isPeekingThrough && getLayerFE().hasRoundedCorners());
}
```

也就是说,强制client合成只有两个条件:

| 条件 | 原因 |
|---|---|
| `forceClientComposition` | 此前各步算出来的:不安全内容、dataspace不支持、旋转、背景模糊、调试开关等 |
| `hasRoundedCorners()`且不是peeking-through | 圆角半径需要SF自己画 |

`hasRoundedCorners()`的定义:

```cpp
// frameworks/native/services/surfaceflinger/LayerFE.cpp
bool LayerFE::hasRoundedCorners() const {
    return mSnapshot->roundedCorner.hasSfDrawnRadius();
}
```

注意是`hasSfDrawnRadius()`,即"由SF绘制圆角半径"这种圆角。如果圆角由HWC绘制(`hasClientDrawnRadius()`那一路),不会走到这个分支。`isPeekingThrough`时圆角可以不画,所以不强制client。

顺带澄清一个容易混进来的概念:`Composition::SOLID_COLOR`**不是**强制client合成的条件。它来自`RequestedLayerState::getCompositionType()`:

```cpp
// frameworks/native/services/surfaceflinger/FrontEnd/RequestedLayerState.cpp
aidl::android::hardware::graphics::composer3::Composition RequestedLayerState::getCompositionType()
        const {
    using aidl::android::hardware::graphics::composer3::Composition;
    // TODO(b/238781169) check about sidestream ready flag
    if (sidebandStream.get()) {
        return Composition::SIDEBAND;
    }
    if (renderCommandBuffer) {
        return Composition::CLIENT;
    }
    if (!externalTexture) {
        return Composition::SOLID_COLOR;
    }
    if (flags & layer_state_t::eLayerIsDisplayDecoration) {
        return Composition::DISPLAY_DECORATION;
    }
    if (flags & layer_state_t::eLayerIsRefreshRateIndicator) {
        return Composition::REFRESH_RATE_INDICATOR;
    }
    if (potentialCursor) {
        return Composition::CURSOR;
    }
    return Composition::DEVICE;
}
```

`SOLID_COLOR`只是"这一层没有外部纹理"的标记,默认由HWC以`Composition::SOLID_COLOR`类型直接刷纯色,并不经过GPU。只有发生override(例如屏幕截图场景给某一层塞了替代buffer)时,`writeStateToHWC`才把它改成`DEVICE`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
    auto requestedCompositionType = outputIndependentState->compositionType;
    if (requestedCompositionType == Composition::SOLID_COLOR && state.overrideInfo.buffer) {
        requestedCompositionType = Composition::DEVICE;
    }
```

真正的client合成要求来自`forceClientComposition`和`hasRoundedCorners()`两个布尔值,而不是`SOLID_COLOR`。

**第五步:setColorTransform**

如果全局颜色矩阵有变化,把矩阵写进Output状态并把整个Output标脏。没有变化则直接返回,不产生额外开销。

**第六步:beginFrame**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::beginFrame() {
    auto& outputState = editState();
    const bool dirty = !getDirtyRegion().isEmpty();
    const bool empty = getOutputLayerCount() == 0;
    const bool wasEmpty = !outputState.lastCompositionHadVisibleLayers;

    // If nothing has changed (!dirty), don't recompose.
    // If something changed, but we don't currently have any visible layers,
    //   and didn't when we last did a composition, then skip it this time.
    // The second rule does two things:
    // - When all layers are removed from a display, we'll emit one black
    //   frame, then nothing more until we get new layers.
    // - When a display is created with a private layer stack, we won't
    //   emit any black frames until a layer is added to the layer stack.
    mMustRecompose = dirty && !(empty && wasEmpty);

    const char flagPrefix[] = {'-', '+'};
    static_cast<void>(flagPrefix);
    ALOGV("%s: %s composition for %s (%cdirty %cempty %cwasEmpty)", __func__,
          mMustRecompose ? "doing" : "skipping", getName().c_str(), flagPrefix[dirty],
          flagPrefix[empty], flagPrefix[wasEmpty]);

    mRenderSurface->beginFrame(mMustRecompose);

    if (mMustRecompose) {
        outputState.lastCompositionHadVisibleLayers = !empty;
    }
}
```

`mMustRecompose = dirty && !(empty && wasEmpty)`这条规则很关键:
- 没有脏区域,不重算。
- 有脏区域,但当前没有可见层,上一帧也没有可见层,不重算。这样避免显示器空着的时候反复刷黑帧。
- 有脏区域,且当前有层(或者上一帧有层但这一帧刚被清空),重算。清空的那一帧会输出一帧黑,之后停止。

`mRenderSurface->beginFrame(mMustRecompose)`调到`DisplaySurface::beginFrame`。各实现的动作并不一样:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::beginFrame(bool /*mustRecompose*/) {
    return NO_ERROR;
}
```

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
status_t VirtualDisplaySurface::beginFrame(bool mustRecompose) {
    ATRACE_CALL();

    std::scoped_lock _l(mMutex);

    ALOGE_IF(mCurrentFrame != std::nullopt,
             "%s: called while another frame is being processed. Overwriting.", __func__);
    mCurrentFrame.reset();

    if (!mIsReady) {
        ALOGI("%s: attempting to connect to still-unconnected sink surface.", __func__);

        if (mSinkSurfaceDataFuture.valid()) {
            prepareSurfacesLocked();
        } else {
            ALOGW("%s: unable to begin frame because the underlying surface is not connected.",
                  __func__);
            return NO_INIT;
        }
    }

    if (mSinkHelper->isFrozen()) {
        ALOGW("%s: mWorkerThread is frozen! Skipping frame.", __func__);
        return WOULD_BLOCK;
    }

    if (mPendingResize) {
        applyResizeLocked(*mPendingResize);
        mPendingResize.reset();
    }

    mCurrentFrame.emplace(FrameInfo());
    mCurrentFrame->mustRecompose = mustRecompose;

    return OK;
}
```

物理屏走的是`FramebufferSurface`或`LegacyFramebufferSurface`(取决于`wb_framebuffersurface2`这个flag),两者的`beginFrame`都是空实现,`mustRecompose`参数连名字都没有,直接返回`NO_ERROR`:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/LegacyFramebufferSurface.cpp
status_t LegacyFramebufferSurface::beginFrame(bool /*mustRecompose*/) {
    return NO_ERROR;
}
```

虚拟屏的`VirtualDisplaySurface`则把它存进`mCurrentFrame->mustRecompose`。`mMustRecompose`真正被读的地方之一是`Display::finishFrame`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Display.cpp
void Display::finishFrame(GpuCompositionResult&& result) {
    // We only need to actually compose the display if:
    // 1) It is being handled by hardware composer, which may need this to
    //    keep its virtual display state machine in sync, or
    // 2) There is work to be done (the dirty region isn't empty)
    if (isGpuVirtualDisplay() && !mustRecompose()) {
        ALOGV("Skipping display composition");
        return;
    }

    impl::Output::finishFrame(std::move(result));
}
```

也就是说,GPU虚拟屏在没有脏区域时直接跳过`impl::Output::finishFrame`,连dequeue、compose、queueBuffer都不做。

**第七步:prepareFrame或prepareFrameAsync**

见5.6和5.10。

**第八步:devOptRepaintFlash**

只在`devOptFlashDirtyRegionsDelay`有值(调试属性)时执行:先把脏区域合成一帧、present、睡一段时间,再重新prepareFrame。这是"脏区域闪烁"调试功能。

**第九步:finishFrame**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::finishFrame(GpuCompositionResult&& result) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);
    const auto& outputState = getState();
    if (!outputState.isEnabled) {
        return;
    }

    std::optional<base::unique_fd> optReadyFence;
    std::shared_ptr<renderengine::ExternalTexture> buffer;
    base::unique_fd bufferFence;
    if (outputState.strategyPrediction == CompositionStrategyPredictionState::SUCCESS) {
        optReadyFence = std::move(result.fence);
    } else {
        if (result.bufferAvailable()) {
            buffer = std::move(result.buffer);
            bufferFence = std::move(result.fence);
        } else {
            updateProtectedContentState();
            if (!dequeueRenderBuffer(&bufferFence, &buffer)) {
                return;
            }
        }
        // Repaint the framebuffer (if needed), getting the optional fence for when the
        // composition completes.
        optReadyFence = composeSurfaces(Region::INVALID_REGION, buffer, bufferFence);
    }
    if (!optReadyFence) {
        return;
    }
    ...
    // swap buffers (presentation)
    mRenderSurface->queueBuffer(std::move(*optReadyFence), getHdrSdrRatio(buffer));
}
```

- 如果策略预测成功,`prepareFrameAsync`已经在后台线程完成了dequeue和GPU合成,这里直接复用那个fence。
- 否则,如果`prepareFrameAsync`已经dequeue了buffer但因预测失败没合成,复用那个buffer;否则重新dequeue。
- 调用`composeSurfaces`做GPU合成。
- `mRenderSurface->queueBuffer(readyFence, hdrSdrRatio)`把渲染结果还给BufferQueue,并触发`FramebufferSurface::advanceFrame`把buffer交给HWC作为client target。

**第十步:presentFrameAndReleaseLayers(Async)**

见5.8、5.9。

**第十一步:renderCachedSets**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::renderCachedSets(const CompositionRefreshArgs& refreshArgs) {
    const auto& outputState = getState();
    if (mPlanner && outputState.isEnabled) {
        mPlanner->renderCachedSets(outputState, refreshArgs.scheduledFrameTime,
                                   outputState.usesDeviceComposition || getSkipColorTransform());
    }
}
```

`Planner`的cached set功能会把"内容不变的一组Layer"预先渲染成一个buffer,后续帧直接让HWC用这个buffer,从而省掉GPU合成。`renderCachedSets`负责在合适的时机把这些cached set渲染出来。

### 5.6 合成策略:validate与applyCompositionStrategy

**prepareFrame(同步路径)**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::prepareFrame() {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    auto& outputState = editState();
    if (!outputState.isEnabled) {
        return;
    }

    std::optional<android::HWComposer::DeviceRequestedChanges> changes;
    bool success = chooseCompositionStrategy(&changes);

    if (success && changes.has_value()) {
        for (const compositionengine::OutputLayer* layer : getOutputLayersOrderedByZ()) {
            HWC2::Layer* hwcLayer = layer->getHwcLayer();
            if (!hwcLayer || changes->changedTypes.contains(hwcLayer)) continue;
            changes->changedTypes[hwcLayer] = layer->getState().hwc->hwcCompositionType;
        }
    }

    resetCompositionStrategy();
    outputState.strategyPrediction = CompositionStrategyPredictionState::DISABLED;
    outputState.previousDeviceRequestedChanges = changes;
    outputState.previousDeviceRequestedSuccess = success;
    if (success) {
        applyCompositionStrategy(changes);
    }
    finishPrepareFrame();
}
```

流程:

1. `chooseCompositionStrategy(&changes)`问HWC:"这一帧你打算怎么合成?"
2. HWC只返回**发生变化的层**。对于没变化的层,把当前`hwcCompositionType`补进`changedTypes`,这样`changes`就是一份完整的合成类型快照。这份快照会被`prepareFrameAsync`用来做预测比对。
3. `resetCompositionStrategy()`先清空上一帧的策略。
4. 保存`previousDeviceRequestedChanges`和`previousDeviceRequestedSuccess`,供下一帧预测。
5. `applyCompositionStrategy(changes)`应用HWC的要求。
6. `finishPrepareFrame()`把`usesClientComposition/usesDeviceComposition`同步给RenderSurface。

**Display::chooseCompositionStrategy**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Display.cpp
bool Display::chooseCompositionStrategy(
        std::optional<android::HWComposer::DeviceRequestedChanges>* outChanges) {
    SFTRACE_FORMAT("%s for %s", __func__, getNamePlusId().c_str());
    ALOGV(__FUNCTION__);

    if (mIsDisconnected) {
        return false;
    }

    // If we don't have a HWC display, then we are done.
    const auto halDisplayId = getDisplayIdVariant().and_then(asHalDisplayId<DisplayIdVariant>);
    if (!halDisplayId) {
        return false;
    }

    // Get any composition changes requested by the HWC device, and apply them.
    auto& hwc = getCompositionEngine().getHwComposer();
    const bool requiresClientComposition = anyLayersRequireClientComposition();

    const TimePoint hwcValidateStartTime = TimePoint::now();

    const auto physicalDisplayId = getDisplayIdVariant().and_then(asPhysicalDisplayId);

    if (physicalDisplayId && getState().readbackBuffer) {
        hwc.setReadbackBuffer(*physicalDisplayId, getState().readbackBuffer, Fence::NO_FENCE);
    }

    if (status_t result = hwc.getDeviceCompositionChanges(*halDisplayId, requiresClientComposition,
                                                          getState().earliestPresentTime,
                                                          getState().expectedPresentTime,
                                                          getState().frameInterval, outChanges);
        result != NO_ERROR) {
        ALOGE("chooseCompositionStrategy failed for %s: %d (%s)", getName().c_str(), result,
              strerror(-result));
        return false;
    }
    ...
    return true;
}
```

`anyLayersRequireClientComposition()`判断是否有任何层被强制要求client合成,判断依据是`OutputLayer::requiresClientComposition`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
bool OutputLayer::requiresClientComposition() const {
    const auto& state = getState();
    return !state.hwc || state.hwc->hwcCompositionType == Composition::CLIENT;
}
```

**HWComposer::getDeviceCompositionChanges**

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
status_t HWComposer::getDeviceCompositionChanges(
        HalDisplayId displayId, bool frameUsesClientComposition,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime,
        nsecs_t expectedPresentTime, Fps frameInterval,
        std::optional<android::HWComposer::DeviceRequestedChanges>* outChanges) {
    SFTRACE_CALL();

    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;
    if (!hwcDisplay->isConnected()) {
        return NO_ERROR;
    }

    uint32_t numTypes = 0;
    uint32_t numRequests = 0;

    hal::Error error = hal::Error::NONE;

    // First try to skip validate altogether. We can do that when
    // 1. The previous frame has not been presented yet or already passed the
    // earliest time to present. Otherwise, we may present a frame too early.
    // 2. There is no client composition. Otherwise, we first need to render the
    // client target buffer.
    const bool canSkipValidate = [&] {
        // We must call validate if we have client composition
        if (frameUsesClientComposition) {
            return false;
        }

        // If composer supports getting the expected present time, we can skip
        // as composer will make sure to prevent early presentation
        if (!earliestPresentTime) {
            return true;
        }

        // composer doesn't support getting the expected present time. We can only
        // skip validate if we know that we are not going to present early.
        return std::chrono::steady_clock::now() >= *earliestPresentTime;
    }();

    displayData.validateWasSkipped = false;
    SFTRACE_FORMAT("NextFrameInterval %d_Hz", frameInterval.getIntValue());
    if (canSkipValidate) {
        sp<Fence> outPresentFence = Fence::NO_FENCE;
        uint32_t state = UINT32_MAX;
        error = hwcDisplay->presentOrValidate(expectedPresentTime, frameInterval.getPeriodNsecs(),
                                              &numTypes, &numRequests, &outPresentFence, &state);
        if (!hasChangesError(error)) {
            RETURN_IF_HWC_ERROR_FOR("presentOrValidate", error, displayId, UNKNOWN_ERROR);
        }
        if (state == 1) { // Present Succeeded.
            std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
            error = hwcDisplay->getReleaseFences(&releaseFences);
            displayData.releaseFences = std::move(releaseFences);
            displayData.lastPresentFence = outPresentFence;
            displayData.validateWasSkipped = true;
            displayData.presentError = error;
            return NO_ERROR;
        }
        // Present failed but Validate ran.
    } else {
        error = hwcDisplay->validate(expectedPresentTime, frameInterval.getPeriodNsecs(), &numTypes,
                                     &numRequests);
    }
    ALOGV("SkipValidate failed, Falling back to SLOW validate/present");
    if (!hasChangesError(error)) {
        RETURN_IF_HWC_ERROR_FOR("validate", error, displayId, BAD_INDEX);
    }

    android::HWComposer::DeviceRequestedChanges::ChangedTypes changedTypes;
    changedTypes.reserve(numTypes);
    error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
    RETURN_IF_HWC_ERROR_FOR("getChangedCompositionTypes", error, displayId, BAD_INDEX);

    auto displayRequests = static_cast<hal::DisplayRequest>(0);
    android::HWComposer::DeviceRequestedChanges::LayerRequests layerRequests;
    layerRequests.reserve(numRequests);
    error = hwcDisplay->getRequests(&displayRequests, &layerRequests);
    RETURN_IF_HWC_ERROR_FOR("getRequests", error, displayId, BAD_INDEX);

    DeviceRequestedChanges::ClientTargetProperty clientTargetProperty;
    error = hwcDisplay->getClientTargetProperty(&clientTargetProperty);
    RETURN_IF_HWC_ERROR_FOR("getClientTargetProperty", error, displayId, BAD_INDEX);

    DeviceRequestedChanges::LayerLuts layerLuts;
    error = hwcDisplay->getRequestedLuts(&layerLuts, mLutFileDescriptorMapper);
    RETURN_IF_HWC_ERROR_FOR("getRequestedLuts", error, displayId, BAD_INDEX);

    outChanges->emplace(DeviceRequestedChanges{std::move(changedTypes), std::move(displayRequests),
                                               std::move(layerRequests),
                                               std::move(clientTargetProperty),
                                               std::move(layerLuts)});
    error = hwcDisplay->acceptChanges();
    RETURN_IF_HWC_ERROR_FOR("acceptChanges", error, displayId, BAD_INDEX);

    return NO_ERROR;
}
```

这段代码有两个分支,非常关键:

**分支A:skip validate,`presentOrValidate`**

当同时满足以下条件时可以跳过validate:

1. 这一帧没有client合成。如果有client合成,必须先让GPU把client target渲染出来,HWC才能validate。
2. composer支持接收`expectedPresentTime`,或者当前时间已经超过`earliestPresentTime`。

`presentOrValidate`是一个合并调用:如果HWC确认可以present,直接present并返回`state == 1`;否则它内部会跑一次validate并返回`state != 1`,同时把changed types和requests的数量带回来。返回1时:

- `getReleaseFences()`拿到每层的release fence。
- `lastPresentFence`保存present fence。
- `validateWasSkipped = true`,后续`presentAndGetReleaseFences`只需`executeCommands`,不再present。

**分支B:普通validate**

先`validateDisplay`,然后依次取:

| 调用 | 内容 |
|---|---|
| `getChangedCompositionTypes` | HWC要求改变合成类型的层(例如把DEVICE改成CLIENT) |
| `getRequests` | display requests和layer requests(例如清client target、读back buffer) |
| `getClientTargetProperty` | client target的dataspace、亮度、dimming |
| `getRequestedLuts` | 请求的LUT |
| `acceptChanges` | 告诉HWC"这些变更我接受了" |

`acceptChanges`必须在present之前调用,否则HWC可能拒绝后续操作。

**applyCompositionStrategy**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Display.cpp
void Display::applyCompositionStrategy(const std::optional<DeviceRequestedChanges>& changes) {
    if (changes) {
        applyChangedTypesToLayers(changes->changedTypes);
        applyDisplayRequests(changes->displayRequests);
        applyLayerRequestsToLayers(changes->layerRequests);
        applyClientTargetRequests(changes->clientTargetProperty);
        applyLayerLutsToLayers(changes->layerLuts);
    }

    // Determine what type of composition we are doing from the final state
    auto& state = editState();
    state.usesClientComposition = anyLayersRequireClientComposition();
    state.usesDeviceComposition = !allLayersRequireClientComposition();
}
```

五个apply分别把HWC的要求落到`OutputLayer`和Output状态上:

- `applyChangedTypesToLayers`:把HWC指定的合成类型写到`hwcState.hwcCompositionType`。
- `applyDisplayRequests`:处理display级别的request。
- `applyLayerRequestsToLayers`:处理layer级别的request(例如clear client target)。
- `applyClientTargetRequests`:设置client target的dataspace、亮度、dimming,并同步给`RenderSurface`。
- `applyLayerLutsToLayers`:应用LUT。

最后两个布尔值是整条合成链路的分叉点:

| 字段 | 计算 | 含义 |
|---|---|---|
| `usesClientComposition` | `anyLayersRequireClientComposition()` | 至少有一层需要GPU合成 |
| `usesDeviceComposition` | `!allLayersRequireClientComposition()` | 不是所有层都需要GPU合成,即至少一层由HWC合成 |

注意这两个值可以同时为true,这就是Mixed合成:部分层GPU画、部分层DPU画。

`finishPrepareFrame`会把这个结果同步给RenderSurface:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/RenderSurface.cpp
void RenderSurface::prepareFrame(bool usesClientComposition, bool usesDeviceComposition) {
    const auto compositionType = [=] {
        using CompositionType = DisplaySurface::CompositionType;

        if (usesClientComposition && usesDeviceComposition) return CompositionType::Mixed;
        if (usesClientComposition) return CompositionType::Gpu;
        if (usesDeviceComposition) return CompositionType::Hwc;

        // Nothing to do -- when turning the screen off we get a frame like
        // this. Call it a HWC frame since we won't be doing any GPU work but
        // will do a prepare/set cycle.
        return CompositionType::Hwc;
    }();

    if (status_t result = mDisplaySurface->prepareFrame(compositionType); result != NO_ERROR) {
        ALOGE("updateCompositionType failed for %s: %d (%s)", mDisplay.getName().c_str(), result,
              strerror(-result));
    }
}
```

### 5.7 client合成

只有当`usesClientComposition`或`flipClientTarget`为true时才需要dequeue buffer:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
bool Output::dequeueRenderBuffer(base::unique_fd* bufferFence,
                                 std::shared_ptr<renderengine::ExternalTexture>* tex) {
    const auto& outputState = getState();

    // If we aren't doing client composition on this output, but do have a
    // flipClientTarget request for this frame on this output, we still need to
    // dequeue a buffer.
    if (outputState.usesClientComposition || outputState.flipClientTarget) {
        *tex = mRenderSurface->dequeueBuffer(bufferFence);
        if (*tex == nullptr) {
            ALOGW("Dequeuing buffer for display [%s] failed, bailing out of "
                  "client composition for this frame",
                  mName.c_str());
            return false;
        }
    }
    return true;
}
```

`RenderSurface::dequeueBuffer`从BufferQueue拿一块buffer,并把它包装成`ExternalTexture`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/RenderSurface.cpp
std::shared_ptr<renderengine::ExternalTexture> RenderSurface::dequeueBuffer(
        base::unique_fd* bufferFence) {
    SFTRACE_CALL();
    int fd = -1;
    ANativeWindowBuffer* buffer = nullptr;

    status_t result = mNativeWindow->dequeueBuffer(mNativeWindow.get(), &buffer, &fd);

    if (result != NO_ERROR) {
        ALOGE("ANativeWindow::dequeueBuffer failed for display [%s] with error: %d",
              mDisplay.getName().c_str(), result);
        // Return fast here as we can't do much more - any rendering we do
        // now will just be wrong.
        return mTexture;
    }

    ALOGW_IF(mTexture != nullptr, "Clobbering a non-null pointer to a buffer [%p].",
             mTexture->getBuffer()->getNativeBuffer()->handle);

    sp<GraphicBuffer> newBuffer = GraphicBuffer::from(buffer);

    std::shared_ptr<renderengine::ExternalTexture> texture;

    for (auto it = mTextureCache.begin(); it != mTextureCache.end(); it++) {
        const auto& cachedTexture = *it;
        if (cachedTexture->getBuffer()->getId() == newBuffer->getId()) {
            texture = cachedTexture;
            mTextureCache.erase(it);
            break;
        }
    }

    if (texture) {
        mTexture = texture;
    } else {
        mTexture = std::make_shared<
                renderengine::impl::ExternalTexture>(GraphicBuffer::from(buffer),
                                                     mCompositionEngine.getRenderEngine(),
                                                     renderengine::impl::ExternalTexture::Usage::
                                                             WRITEABLE);
    }
    mTextureCache.push_back(mTexture);
    if (mTextureCache.size() > mMaxTextureCacheSize) {
        mTextureCache.erase(mTextureCache.begin());
    }

    *bufferFence = base::unique_fd(fd);

    return mTexture;
}
```

- `mTextureCache`按buffer id缓存`ExternalTexture`,避免每帧重新创建EGLImage。缓存超过`mMaxTextureCacheSize`时淘汰最旧的一项。
- `bufferFence`是dequeue自带的fence,表示这块buffer什么时候可以被写入。它会被一路传给`RenderEngine::drawLayers`,由RenderEngine在GPU侧等待。

**composeSurfaces**

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
std::optional<base::unique_fd> Output::composeSurfaces(
        const Region& debugRegion, std::shared_ptr<renderengine::ExternalTexture> tex,
        base::unique_fd& fd) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    const auto& outputState = getState();
    const TracedOrdinal<bool> hasClientComposition =
            {CC_UNLIKELY(SFTRACE_ENABLED())
                     ? base::StringPrintf("hasClientComposition %s", mNamePlusId.c_str())
                     : "",
             outputState.usesClientComposition};
    if (!hasClientComposition) {
        setExpensiveRenderingExpected(false);
        return base::unique_fd();
    }

    if (tex == nullptr) {
        ALOGW("Buffer not valid for display [%s], bailing out of "
              "client composition for this frame",
              mName.c_str());
        return {};
    }

    ALOGV("hasClientComposition");

    renderengine::DisplaySettings clientCompositionDisplay =
            generateClientCompositionDisplaySettings(tex);

    // Generate the client composition requests for the layers on this output.
    auto& renderEngine = getCompositionEngine().getRenderEngine();
    const bool supportsProtectedContent = renderEngine.supportsProtectedContent();
    std::vector<LayerFE*> clientCompositionLayersFE;
    std::vector<LayerFE::LayerSettings> clientCompositionLayers =
            generateClientCompositionRequests(supportsProtectedContent,
                                              clientCompositionDisplay.outputDataspace,
                                              clientCompositionLayersFE);
    appendRegionFlashRequests(debugRegion, clientCompositionLayers);

    OutputCompositionState& outputCompositionState = editState();
    // Check if the client composition requests were rendered into the provided graphic buffer. If
    // so, we can reuse the buffer and avoid client composition.
    if (mClientCompositionRequestCache) {
        if (mClientCompositionRequestCache->exists(tex->getBuffer()->getId(),
                                                   clientCompositionDisplay,
                                                   clientCompositionLayers)) {
            SFTRACE_NAME("ClientCompositionCacheHit");
            outputCompositionState.reusedClientComposition = true;
            setExpensiveRenderingExpected(false);
            // b/239944175 pass the fence associated with the buffer.
            return base::unique_fd(std::move(fd));
        }
        SFTRACE_NAME("ClientCompositionCacheMiss");
        mClientCompositionRequestCache->add(tex->getBuffer()->getId(), clientCompositionDisplay,
                                            clientCompositionLayers);
    }
    ...
    const nsecs_t renderEngineStart = systemTime();
    auto fenceResult = renderEngine
                               .drawLayers(clientCompositionDisplay, clientRenderEngineLayers, tex,
                                           std::move(fd))
                               .get();
    ...
    for (auto* clientComposedLayer : clientCompositionLayersFE) {
        clientComposedLayer->setWasClientComposed(fence);
    }

    return base::unique_fd(fence->dup());
}
```

逐段:

1. **没有client合成直接返回**:返回空的`unique_fd`,表示没有GPU工作。
2. **生成DisplaySettings**:`generateClientCompositionDisplaySettings`组装物理显示区域、裁剪区域、方向、输出dataspace、亮度、颜色变换矩阵等。
3. **生成LayerSettings**:`generateClientCompositionRequests`遍历Z序Layer,跳过空clip,对每个`requiresClientComposition()`的层生成一份`LayerFE::LayerSettings`。
4. **client合成缓存**:如果相同的buffer id、相同的DisplaySettings、相同的LayerSettings之前已经渲染过,直接复用,置`reusedClientComposition = true`,返回dequeue时的fence。这样避免重复GPU工作。
5. **昂贵渲染检测**:如果有背景模糊,或者某层的sourceDataspace和输出dataspace不一致(需要颜色空间转换),标记为expensive rendering,让PowerAdvisor提升GPU频率。
6. **调用RenderEngine**:`renderEngine.drawLayers(...)`返回一个future,fence就是GPU渲染完成的fence。`.get()`会阻塞主线程直到RenderEngine把命令提交并返回fence(注意RenderEngine的draw是异步的,fence本身在GPU执行完才signal)。
7. **标记client合成层**:对每个参与client合成的LayerFE调用`setWasClientComposed(fence)`,后续`Layer::onCompositionPresented`会读这个信息并设置`FALLBACK_COMPOSITION`。
8. **返回fence**:`fence->dup()`复制一份fd返回给调用方,原fence留给后续使用。

`generateClientCompositionRequests`里的关键判断:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
    for (auto* layer : getOutputLayersOrderedByZ()) {
        const auto& layerState = layer->getState();
        const auto* layerFEState = layer->getLayerFE().getCompositionState();
        auto& layerFE = layer->getLayerFE();
        layerFE.setWasClientComposed(nullptr);

        const Region clip(viewportRegion.intersect(layerState.visibleRegion));
        ALOGV("Layer: %s", layerFE.getDebugName());
        if (clip.isEmpty()) {
            ALOGV("  Skipping for empty clip");
            firstLayer = false;
            continue;
        }

        disableBlursWholesale |= layerFEState->sidebandStream != nullptr;
        bool disableBlurForLayer = layer->getState().ignoreBlur || disableBlursWholesale;
        ...
```

- 每层先清空`wasClientComposed`。
- 可见区域与Viewport求交,空则跳过。
- 有sideband stream的层会全局禁用模糊。

真正判断是否参与client合并在后面:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
        if (!layer->requiresClientComposition()) {
            continue;
        }
```

只有`requiresClientComposition()`返回true的层才会被加进`clientCompositionLayers`。

**queueBuffer与advanceFrame**

`finishFrame`里:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
    mRenderSurface->queueBuffer(std::move(*optReadyFence), getHdrSdrRatio(buffer));
```

`RenderSurface::queueBuffer`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/RenderSurface.cpp
void RenderSurface::queueBuffer(base::unique_fd readyFence, float hdrSdrRatio) {
    auto& state = mDisplay.getState();

    if (state.usesClientComposition || state.flipClientTarget) {
        // hasFlipClientTargetRequest could return true even if we haven't
        // dequeued a buffer before. Try dequeueing one if we don't have a
        // buffer ready.
        if (mTexture == nullptr) {
            ALOGI("Attempting to queue a client composited buffer without one "
                  "previously dequeued for display [%s]. Attempting to dequeue a "
                  "scratch buffer now",
                  mDisplay.getName().c_str());
            // We shouldn't deadlock here, since mTexture == nullptr only
            // after a successful call to queueBuffer, or if dequeueBuffer has
            // never been called.
            base::unique_fd unused;
            dequeueBuffer(&unused);
        }

        if (mTexture == nullptr) {
            ALOGE("No buffer is ready for display [%s]", mDisplay.getName().c_str());
        } else {
            status_t result = mNativeWindow->queueBuffer(mNativeWindow.get(),
                                                         mTexture->getBuffer()->getNativeBuffer(),
                                                         dup(readyFence));
            if (result != NO_ERROR) {
                ALOGE("Error when queueing buffer for display [%s]: %d", mDisplay.getName().c_str(),
                      result);
                // We risk blocking on dequeueBuffer if the primary display failed
                // to queue up its buffer, so crash here.
                if (!mDisplay.isVirtual()) {
                    LOG_ALWAYS_FATAL("ANativeWindow::queueBuffer failed with error: %d", result);
                } else {
                    mNativeWindow->cancelBuffer(mNativeWindow.get(),
                                                mTexture->getBuffer()->getNativeBuffer(),
                                                dup(readyFence));
                }
            }

            mTexture = nullptr;
        }
    }

    status_t result = mDisplaySurface->advanceFrame(hdrSdrRatio);
    if (result != NO_ERROR) {
        ALOGE("[%s] failed pushing new frame to HWC: %d", mDisplay.getName().c_str(), result);
    }
}
```

- 只有`usesClientComposition`或`flipClientTarget`才需要queueBuffer。纯HWC合成时,SF没有渲染任何东西,BufferQueue里也没有buffer要还。
- `mNativeWindow->queueBuffer`把buffer还给BufferQueue,并带上GPU完成的readyFence。
- 最后无论是否做了client合成,都要调用`mDisplaySurface->advanceFrame(hdrSdrRatio)`。

`FramebufferSurface::advanceFrame`:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::advanceFrame(float hdrSdrRatio) {
    Mutex::Autolock lock(mMutex);
    BufferItem item;
    status_t err = mRendererConsumer->acquireBuffer(&item, 0, /*waitForFence*/ false,
                                                    [&](const sp<GraphicBuffer>& buffer) {
                                                        ASSERT_MUTEX(mMutex);
                                                        onBufferFreedLocked(buffer);
                                                    });
    if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
        mDataspace = Dataspace::UNKNOWN;
        return NO_ERROR;
    } else if (err != NO_ERROR) {
        ALOGE("error acquiring buffer: %s (%d)", strerror(-err), err);
        mDataspace = Dataspace::UNKNOWN;
        return err;
    }

    if (mFrameData && mFrameData->mBuffer != item.mGraphicBuffer) {
        mPreviousFrameData.swap(mFrameData);
    }
    mFrameData = {item.mGraphicBuffer, item.mFence};
    mDataspace = static_cast<Dataspace>(item.mDataSpace);

    // If HWC hasn't seen the buffer before, or the buffer has changed, send it to HWC. Otherwise,
    // all we need is the slot itself.
    auto [requiresRefresh, hwcSlot] = mHwcSlotTracker.getSlot(mFrameData->mBuffer);
    sp<GraphicBuffer> hwcBuffer = requiresRefresh ? mFrameData->mBuffer : nullptr;

    status_t result = mHwc.setClientTarget(mDisplayId, hwcSlot, mFrameData->mFence, hwcBuffer,
                                           mDataspace, hdrSdrRatio);
    if (result != NO_ERROR) {
        ALOGE("error posting framebuffer: %s (%d)", strerror(-result), result);
        return result;
    }

    return NO_ERROR;
}
```

- `acquireBuffer`从BufferQueue拿到刚才queue进去的buffer,`waitForFence=false`,不阻塞。
- `mPreviousFrameData.swap(mFrameData)`:只有当buffer对象换了才把旧帧记为previous。注意这里用`swap`而不是赋值,所以`mPreviousFrameData`保留的是更早一帧的buffer。
- `mHwcSlotTracker.getSlot`判断这块buffer之前是否已经作为client target slot注册过。没注册过才需要把完整buffer传给`setClientTarget`,否则只传slot号。
- `setClientTarget`把buffer、acquire fence、dataspace、HDR/SDR比率提交给HWC。

到这一步,GPU合成的结果已经进入HWC的client target slot,接下来HWC可以开始合成。

### 5.8 device合成与HWC present

`Output::presentFrameAndReleaseLayers`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
void Output::presentFrameAndReleaseLayers(bool flushEvenWhenDisabled) {
    SFTRACE_FORMAT("%s for %s", __func__, mNamePlusId.c_str());
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        if (flushEvenWhenDisabled) {
            // Some commands, like clearing buffer slots, should still be executed
            // even if the display is not enabled.
            executeCommands();
        }
        return;
    }

    auto& outputState = editState();
    outputState.dirtyRegion.clear();

    auto frame = presentFrame();

    mRenderSurface->onPresentDisplayCompleted();
    ...
```

- 显示器未启用时,只有`flushEvenWhenDisabled`为true才执行`executeCommands`。这个标志来自`refreshArgs.bufferIdsToUncache`非空,即"还有命令必须刷给HWC"。
- `outputState.dirtyRegion.clear()`:present之后脏区域清空,下一帧重新累积。
- `presentFrame()`是虚函数,`Display`重写了它。
- `mRenderSurface->onPresentDisplayCompleted()`最终调到`FramebufferSurface::onFrameCommitted()`。

`Display::presentFrame`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Display.cpp
compositionengine::Output::FrameFences Display::presentFrame() {
    auto fences = impl::Output::presentFrame();

    const auto halDisplayIdOpt = getDisplayIdVariant().and_then(asHalDisplayId<DisplayIdVariant>);
    if (mIsDisconnected || !halDisplayIdOpt) {
        return fences;
    }

    auto& hwc = getCompositionEngine().getHwComposer();

    const TimePoint startTime = TimePoint::now();

    if (isPowerHintSessionEnabled() && getState().earliestPresentTime) {
        mPowerAdvisor->setHwcPresentDelayedTime(*halDisplayIdOpt, *getState().earliestPresentTime);
    }

    hwc.presentAndGetReleaseFences(*halDisplayIdOpt, getState().earliestPresentTime);

    if (isPowerHintSessionEnabled()) {
        mPowerAdvisor->setHwcPresentTiming(*halDisplayIdOpt, startTime, TimePoint::now());
    }

    fences.presentFence = hwc.getPresentFence(*halDisplayIdOpt);

    // TODO(b/121291683): Change HWComposer call to return entire map
    for (const auto* layer : getOutputLayersOrderedByZ()) {
        auto hwcLayer = layer->getHwcLayer();
        if (!hwcLayer) {
            continue;
        }

        fences.layerFences.emplace(hwcLayer, hwc.getLayerReleaseFence(*halDisplayIdOpt, hwcLayer));
    }

    hwc.clearReleaseFences(*halDisplayIdOpt);

    return fences;
}
```

- 基类`impl::Output::presentFrame`先处理client composition的client target acquire fence:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
compositionengine::Output::FrameFences Output::presentFrame() {
    compositionengine::Output::FrameFences result;
    if (getState().usesClientComposition) {
        result.clientTargetAcquireFence = mRenderSurface->getClientTargetAcquireFence();
    }
    return result;
}
```

`getClientTargetAcquireFence`返回的是`FramebufferSurface::mFrameData->mFence`,也就是GPU渲染完成、HWC可以读的fence。

- `hwc.presentAndGetReleaseFences(...)`:执行present。
- `getPresentFence(...)`:取本次present的fence。
- 遍历所有OutputLayer,取每层的release fence。
- `clearReleaseFences(...)`:清空HWC里缓存的release fence,避免下一帧误用。

`HWComposer::presentAndGetReleaseFences`:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
status_t HWComposer::presentAndGetReleaseFences(
        HalDisplayId displayId,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime) {
    SFTRACE_CALL();

    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;

    if (displayData.validateWasSkipped) {
        // explicitly flush all pending commands
        auto error = static_cast<hal::Error>(mComposer->executeCommands(hwcDisplay->getId()));
        RETURN_IF_HWC_ERROR_FOR("executeCommands", error, displayId, UNKNOWN_ERROR);
        RETURN_IF_HWC_ERROR_FOR("present", displayData.presentError, displayId, UNKNOWN_ERROR);
        return NO_ERROR;
    }

    if (earliestPresentTime) {
        SFTRACE_NAME("wait for earliest present time");
        std::this_thread::sleep_until(*earliestPresentTime);
    }

    auto error = hwcDisplay->present(&displayData.lastPresentFence);
    RETURN_IF_HWC_ERROR_FOR("present", error, displayId, UNKNOWN_ERROR);

    std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
    error = hwcDisplay->getReleaseFences(&releaseFences);
    RETURN_IF_HWC_ERROR_FOR("getReleaseFences", error, displayId, UNKNOWN_ERROR);

    displayData.releaseFences = std::move(releaseFences);

    return NO_ERROR;
}
```

两个分支:

**validateWasSkipped为true**:present已经在`presentOrValidate`里做过了,这里只需要`executeCommands`把命令真正刷下去,并检查当时记录的`presentError`。`lastPresentFence`和`releaseFences`已经在validate阶段存好了。

**validateWasSkipped为false**:如果`earliestPresentTime`有值,先`sleep_until`,防止present得太早导致画面撕裂或HWC内部排队。然后`hwcDisplay->present(&lastPresentFence)`,再`getReleaseFences`。

`HWC2::Display::presentOrValidate`:

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Error Display::presentOrValidate(nsecs_t expectedPresentTime, int32_t frameIntervalNs,
                                 uint32_t* outNumTypes, uint32_t* outNumRequests,
                                 sp<android::Fence>* outPresentFence, uint32_t* state) {
    uint32_t numTypes = 0;
    uint32_t numRequests = 0;
    int32_t presentFenceFd = -1;
    auto intError =
            mComposer.presentOrValidateDisplay(mId, expectedPresentTime, frameIntervalNs, &numTypes,
                                               &numRequests, &presentFenceFd, state);
    auto error = static_cast<Error>(intError);
    if (error != Error::NONE && !hasChangesError(error)) {
        return error;
    }

    if (*state == 1) {
        *outPresentFence = sp<Fence>::make(presentFenceFd);
    }

    if (*state == 0) {
        *outNumTypes = numTypes;
        *outNumRequests = numRequests;
    }
    return error;
}
```

这里是`HWC2`命名空间下的`Error`,不是HAL的`hal::Error`,返回时`static_cast<Error>(intError)`。

`state`的取值决定了这次调用的结果:

| state | 含义 | 输出 |
|---|---|---|
| 1 | present成功 | `outPresentFence`填入fence;`outNumTypes`/`outNumRequests`不填 |
| 0 | 只完成了validate | `outNumTypes`/`outNumRequests`填入变更数量;没有present fence |

注意`presentOrValidateDisplay`只返回changed types的**数量**,不返回具体内容。所以validate分支下`HWComposer::getDeviceCompositionChanges`还要再调一次`getChangedCompositionTypes`才能拿到每层的具体类型。

### 5.9 release fence的生成

`presentFrameAndReleaseLayers`的后半段是整条链路里最容易被忽略、但最影响应用帧率的部分:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
    const bool force_slower_follower_gpu_composition =
            FlagManager::getInstance().force_slower_follower_gpu_composition_combined();
    for (auto* layer : getOutputLayersOrderedByZ()) {
        // The layer buffer from the previous frame (if any) is released
        // by HWC only when the release fence from this frame (if any) is
        // signaled.  Always get the release fence from HWC first.
        sp<Fence> releaseFence = Fence::NO_FENCE;

        if (auto hwcLayer = layer->getHwcLayer()) {
            if (auto f = frame.layerFences.find(hwcLayer); f != frame.layerFences.end()) {
                releaseFence = f->second;
            }
        }

        // If the layer was client composited in the previous frame, we
        // need to merge with the previous client target acquire fence.
        if (force_slower_follower_gpu_composition) {
            releaseFence =
                    Fence::merge("LayerRelease", releaseFence,
                                 layer->getLayerFE().getAndClearLastClientTargetAcquireFence());

            // If there's no present fence nor last composition acquire fence, then return the
            // current acquire fence rather than a NO_FENCE which would release immediately.
            if (releaseFence == Fence::NO_FENCE) {
                releaseFence = frame.clientTargetAcquireFence;
            }

            layer->getLayerFE().setLastClientTargetAcquireFence(frame.clientTargetAcquireFence);
        } else {
            // Since we do not track that, always merge with the current
            // client target acquire fence when it is available, even though
            // this is suboptimal.
            if (outputState.usesClientComposition) {
                releaseFence =
                        Fence::merge("LayerRelease", releaseFence, frame.clientTargetAcquireFence);
            }
        }
        layer->getLayerFE().setReleaseFence(releaseFence);
        layer->getLayerFE().setReleasedBuffer(layer->getLayerFE().getCompositionState()->buffer);
    }

    // We've got a list of layers needing fences, that are disjoint with
    // OutputLayersOrderedByZ.  The best we can do is to
    // supply them with the present fence.
    for (auto& weakLayer : mReleasedLayers) {
        if (const auto layer = weakLayer.promote()) {
            layer->setReleaseFence(frame.presentFence);
        }
    }

    // Clear out the released layers now that we're done with them.
    mReleasedLayers.clear();
```

逐层说明:

1. **HWC release fence**:HWC为每个HWC层返回一个release fence,表示"这一层上一帧用的buffer什么时候可以被应用重新写入"。没有HWC层(纯GPU合成)时是`NO_FENCE`。
2. **合并client target acquire fence**:
   - 默认路径:如果这个Output这一帧做了client合成,说明上一帧的client target buffer可能还被HWC读着,所以把`frame.clientTargetAcquireFence`并进release fence。这是保守做法,因为SF没有精确追踪"上一层到底是哪一帧被GPU画进client target的"。
   - `force_slower_follower_gpu_composition`路径:精确追踪每一层上一帧的client target acquire fence(`getAndClearLastClientTargetAcquireFence`),避免把所有层的release都拖到本帧GPU渲染完成。这是对慢follower显示器的优化。
3. **setReleaseFence**:把合并后的fence交给`LayerFE`。`LayerFE`内部有一个`std::promise<FenceResult>`,这个调用就是`set_value`:

```cpp
// frameworks/native/services/surfaceflinger/LayerFE.cpp
void LayerFE::setReleaseFence(const FenceResult& releaseFence) {
    // Promises should not be fulfilled more than once. This case can occur if virtual
    // displays with the same layerstack ID are being created and destroyed in quick
    // succession, such as in tests. This would result in a race condition in which
    // multiple displays have the same layerstack ID within the same vsync interval.
    if (mReleaseFencePromiseStatus == ReleaseFencePromiseStatus::FULFILLED) {
        return;
    }

    if (releaseFence.has_value()) {
        if (FlagManager::getInstance().monitor_buffer_fences()) {
            if (auto strongBuffer = mReleasedBuffer.promote()) {
                strongBuffer->getDependencyMonitor()
                        .addAccessCompletion(FenceTime::makeValid(releaseFence.value()), "HWC");
            }
        }
    }
    mReleaseFence.set_value(releaseFence);
    mReleaseFencePromiseStatus = ReleaseFencePromiseStatus::FULFILLED;
}
```

注释指出了一个真实存在的场景:两个虚拟显示器用同一个layerstack ID,在同一个vsync周期内快速创建销毁,可能导致同一个promise被兑现两次,所以第二次直接返回。

`monitor_buffer_fences`开启时,还会把release fence登记到buffer的dependency monitor上(`addAccessCompletion(..., "HWC")`),用于分析fence依赖链路。

另外要保证promise不会被丢弃:`LayerFE`的析构函数里有兜底:

```cpp
// frameworks/native/services/surfaceflinger/LayerFE.cpp
LayerFE::~LayerFE() {
    // Ensures that no promise is left unfulfilled before the LayerFE is destroyed.
    // An unfulfilled promise could occur when a screenshot is attempted, but the
    // render area is invalid and there is no memory for the capture result.
    if (mReleaseFencePromiseStatus == ReleaseFencePromiseStatus::INITIALIZED) {
        setReleaseFence(Fence::NO_FENCE);
    }
}
```

4. **setReleasedBuffer**:把本帧使用的buffer记下来,供后续统计和buffer生命周期追踪。
5. **mReleasedLayers**:这些是已经不在Z序列表里、但还需要release fence的层。它们没有对应的HWC层,只能用present fence兜底。

### 5.10 策略预测与present卸载

`Output::present`里有两个异步优化,都基于`HwcAsyncWorker`。

**策略预测**

`canPredictCompositionStrategy`判断是否可以做预测:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
bool Output::canPredictCompositionStrategy(const CompositionRefreshArgs& refreshArgs) {
    uint64_t lastOutputLayerHash = getState().lastOutputLayerHash;
    uint64_t outputLayerHash = getState().outputLayerHash;
    editState().lastOutputLayerHash = outputLayerHash;

    if (!getState().isEnabled || !mPredictCompositionStrategy) {
        ALOGV("canPredictCompositionStrategy disabled");
        return false;
    }

    if (!getState().previousDeviceRequestedChanges) {
        ALOGV("canPredictCompositionStrategy previous changes not available");
        return false;
    }

    if (!mRenderSurface->supportsCompositionStrategyPrediction()) {
        ALOGV("canPredictCompositionStrategy surface does not support");
        return false;
    }

    if (refreshArgs.devOptFlashDirtyRegionsDelay) {
        ALOGV("canPredictCompositionStrategy devOptFlashDirtyRegionsDelay");
        return false;
    }

    if (lastOutputLayerHash != outputLayerHash) {
        ALOGV("canPredictCompositionStrategy output layers changed");
        return false;
    }

    // If no layer uses clientComposition, then don't predict composition strategy
    // because we have less work to do in parallel.
    if (!anyLayersRequireClientComposition()) {
        ALOGV("canPredictCompositionStrategy no layer uses clientComposition");
        return false;
    }

    return true;
}
```

预测的前提是:上一帧有完整的`previousDeviceRequestedChanges`、RenderSurface支持预测、没有调试闪烁、这一帧的Layer集合hash和上一帧相同,并且这一帧至少有一层需要client合成。最后一条注释说得很直接:"如果没有层需要client合成,就不要预测,因为并行能省下的工作本来就少"。也就是说,预测的价值在于把HWC validate和GPU合成重叠起来;纯HWC合成时没有GPU工作可以重叠,预测反而变成纯开销。

这里`outputLayerHash`在函数开头就被写进`lastOutputLayerHash`,所以即使本次返回false,下一次比较用的也是本帧的hash。

`prepareFrameAsync`的核心:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
GpuCompositionResult Output::prepareFrameAsync() {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);
    auto& state = editState();
    const auto& previousChanges = state.previousDeviceRequestedChanges;
    std::optional<android::HWComposer::DeviceRequestedChanges> changes;
    resetCompositionStrategy();
    // Store all layer composition types before appplying composition strategy
    android::HWComposer::DeviceRequestedChanges::ChangedTypes backups;
    for (const compositionengine::OutputLayer* layer : getOutputLayersOrderedByZ()) {
        HWC2::Layer* hwcLayer = layer->getHwcLayer();
        if (!hwcLayer) continue;
        backups[hwcLayer] = layer->getState().hwc->hwcCompositionType;
    }

    auto hwcResult = chooseCompositionStrategyAsync(&changes);
    if (state.previousDeviceRequestedSuccess) {
        applyCompositionStrategy(previousChanges);
    }
    finishPrepareFrame();

    base::unique_fd bufferFence;
    std::shared_ptr<renderengine::ExternalTexture> buffer;
    updateProtectedContentState();
    const bool dequeueSucceeded = dequeueRenderBuffer(&bufferFence, &buffer);
    GpuCompositionResult compositionResult;
    if (dequeueSucceeded) {
        std::optional<base::unique_fd> optFd =
                composeSurfaces(Region::INVALID_REGION, buffer, bufferFence);
        if (optFd) {
            compositionResult.fence = std::move(*optFd);
        }
    }

    auto chooseCompositionSuccess = hwcResult.get();
    if (chooseCompositionSuccess && changes.has_value()) {
        // Keep track of all layer composition types, not just changes
        for (const compositionengine::OutputLayer* layer : getOutputLayersOrderedByZ()) {
            HWC2::Layer* hwcLayer = layer->getHwcLayer();
            if (!hwcLayer || changes->changedTypes.contains(hwcLayer)) continue;
            changes->changedTypes[hwcLayer] = backups[hwcLayer];
        }
    }
    const bool predictionSucceeded = dequeueSucceeded && changes == previousChanges;
    state.strategyPrediction = predictionSucceeded ? CompositionStrategyPredictionState::SUCCESS
                                                   : CompositionStrategyPredictionState::FAIL;
    if (!predictionSucceeded) {
        SFTRACE_NAME("CompositionStrategyPredictionMiss");
        resetCompositionStrategy();
        if (chooseCompositionSuccess) {
            applyCompositionStrategy(changes);
        }
        finishPrepareFrame();
        // Track the dequeued buffer to reuse so we don't need to dequeue another one.
        compositionResult.buffer = buffer;
    } else {
        SFTRACE_NAME("CompositionStrategyPredictionHit");
    }
    state.previousDeviceRequestedChanges = std::move(changes);
    state.previousDeviceRequestedSuccess = chooseCompositionSuccess;
    return compositionResult;
}
```

时间线:

1. 备份当前所有层的合成类型。
2. `chooseCompositionStrategyAsync`把validate丢到后台worker执行,立即返回一个future。
3. 主线程不等结果,先用**上一帧的**`previousChanges`应用策略,`finishPrepareFrame`。
4. 主线程接着dequeue buffer、跑GPU合成。这两步与HWC validate并行。
5. GPU合成完,等`hwcResult.get()`拿到真实的changes。
6. 比较真实changes和上一帧的previousChanges,同时还要看dequeue是否成功:`predictionSucceeded = dequeueSucceeded && changes == previousChanges`。两者都成立才算预测成功,GPU合成的结果可以直接用。
7. 如果不等,预测失败:重置策略、应用真实策略、重新`finishPrepareFrame`。已经dequeue的buffer不浪费,放进`compositionResult.buffer`给`finishFrame`复用。

预测命中时,GPU合成本来就是按上一帧策略做的,而策略没变,所以结果有效。预测失败时,GPU可能按错误策略画了一些层,但由于`finishFrame`会复用buffer并重新`composeSurfaces`,最终结果仍然正确,只是多花了一次GPU。

**present卸载**

如果`mOffloadPresent`为true,`presentFrameAndReleaseLayers`会被丢到`HwcAsyncWorker`:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
ftl::Future<std::monostate> Output::presentFrameAndReleaseLayersAsync(bool flushEvenWhenDisabled) {
    return ftl::Future<bool>(mHwComposerAsyncWorker->send([this, flushEvenWhenDisabled,
                                                           registration = panopticon::share()]() {
               registration->start();
               presentFrameAndReleaseLayers(flushEvenWhenDisabled);
               return true;
           }))
            .then([](bool) { return std::monostate{}; });
}
```

`CompositionEngine::offloadOutputs`决定哪些Output可以卸载。注释说得很清楚:"Offloading the HWC call for `present` allows us to simultaneously call it on multiple displays. This is desirable because these calls block and can be slow."

所有Output的present future会在`present()`最后统一`get()`,所以卸载只是让多个显示器的HWC调用并行,不改变语义。

### 5.11 CompositionEngine::postComposition

`postComposition`是"兜底"环节,保证所有release fence promise都有结果:

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
// If a buffer is latched but the layer is not presented, such as when
// obscured by another layer, the previous buffer needs to be released. We find
// these buffers and fire a NO_FENCE, or the last client composition acquire fence to release it.
// This ensures that all promises for buffer releases are fulfilled at the end of composition.
void CompositionEngine::postComposition(CompositionRefreshArgs& args) {
    SFTRACE_CALL();
    ALOGV(__FUNCTION__);

    const bool force_slower_follower_gpu_composition =
            FlagManager::getInstance().force_slower_follower_gpu_composition_combined();
    for (auto& layerFE : args.layers) {
        if (layerFE->getReleaseFencePromiseStatus() ==
            LayerFE::ReleaseFencePromiseStatus::INITIALIZED) {
            if (force_slower_follower_gpu_composition) {
                layerFE->setReleaseFence(layerFE->getAndClearLastClientTargetAcquireFence());
            } else {
                layerFE->setReleaseFence(Fence::NO_FENCE);
            }
        }
    }

    // List of layersWithQueuedFrames does not necessarily overlap with
    // list of layers, so those layersWithQueuedFrames also need any
    // unfulfilled promises to be resolved for completeness.
    for (auto& layerFE : args.layersWithQueuedFrames) {
        if (layerFE->getReleaseFencePromiseStatus() ==
            LayerFE::ReleaseFencePromiseStatus::INITIALIZED) {
            if (force_slower_follower_gpu_composition) {
                layerFE->setReleaseFence(layerFE->getAndClearLastClientTargetAcquireFence());
            } else {
                layerFE->setReleaseFence(Fence::NO_FENCE);
            }
        }
    }
}
```

为什么会有未兑现的promise?因为`output->presentFrameAndReleaseLayers`只对`getOutputLayersOrderedByZ()`里的层设置release fence。如果一个Layer:

- 因为被完全遮挡而没有进入任何Output的Z序列表,但它在`refreshArgs.layers`里;
- 或者它在`layersWithQueuedFrames`里但不在任何Output的可合成列表里;

它的promise就没人兑现。如果不处理,应用会永远等不到`onTransactionCompleted`,buffer池就会耗尽。`postComposition`用一个`NO_FENCE`立即兑现,告诉应用"这个buffer现在就可以复用"。

### 5.12 composite的返回值

`SurfaceFlinger::composite`最后统计每个显示器的合成覆盖情况:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    const bool hadGpuComposited =
            multiDisplayUnion(mCompositionCoverage).test(CompositionCoverage::Gpu);
    mCompositionCoverage.clear();

    TimeStats::ClientCompositionRecord clientCompositionRecord;

    for (const auto& [_, display] : displays) {
        const auto& state = display->getCompositionDisplay()->getState();
        CompositionCoverageFlags& flags =
                mCompositionCoverage.try_emplace(display->getDisplayIdVariant()).first->second;

        if (state.usesDeviceComposition) {
            flags |= CompositionCoverage::Hwc;
        }

        if (state.reusedClientComposition) {
            flags |= CompositionCoverage::GpuReuse;
        } else if (state.usesClientComposition) {
            flags |= CompositionCoverage::Gpu;
        }

        clientCompositionRecord.predicted |=
                (state.strategyPrediction != CompositionStrategyPredictionState::DISABLED);
        clientCompositionRecord.predictionSucceeded |=
                (state.strategyPrediction == CompositionStrategyPredictionState::SUCCESS);
    }

    const auto coverage = multiDisplayUnion(mCompositionCoverage);
    const bool hasGpuComposited = coverage.test(CompositionCoverage::Gpu);

    clientCompositionRecord.hadClientComposition = hasGpuComposited;
    clientCompositionRecord.reused = coverage.test(CompositionCoverage::GpuReuse);
    clientCompositionRecord.changed = hadGpuComposited != hasGpuComposited;

    mTimeStats->pushCompositionStrategyState(clientCompositionRecord);
```

映射关系:

| Output状态 | CompositionCoverage |
|---|---|
| `usesDeviceComposition` | `Hwc` |
| `reusedClientComposition` | `GpuReuse` |
| `usesClientComposition`且没复用 | `Gpu` |

注意`usesDeviceComposition`和`usesClientComposition`可以同时为true(Mixed合成),所以flags可以是`Hwc | Gpu`。

`hadGpuComposited`是上一帧的GPU合成状态(来自`mCompositionCoverage`,在清空之前读),`hasGpuComposited`是本帧的,两者不等就说明合成策略发生了变化,记录到TimeStats。

然后:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    using namespace ftl::flag_operators;

    // TODO(b/160583065): Enable skip validation when SF caches all client composition layers.
    const bool hasGpuUseOrReuse =
            coverage.any(CompositionCoverage::Gpu | CompositionCoverage::GpuReuse);
    mScheduler->modulateVsync({}, &VsyncModulator::onDisplayRefresh, hasGpuUseOrReuse);

    mLayersWithQueuedFrames.clear();
    doActiveLayersTracingIfNeeded(true, mVisibleRegionsDirty, pacesetterTarget.frameBeginTime(),
                                  vsyncId);

    updateInputFlinger(vsyncId, pacesetterTarget.frameBeginTime());

    if (mVisibleRegionsDirty) mHdrLayerInfoChanged = true;
    mVisibleRegionsDirty = false;

    if (mCompositionEngine->needsAnotherUpdate()) {
        scheduleCommit(FrameHint::kNone);
    }

    if (mPowerHintSessionEnabled) {
        mPowerAdvisor->setCompositeEnd(TimePoint::now());
    }

    CompositeResultsPerDisplay resultsPerDisplay;

    // Filter out virtual displays.
    for (const auto& [idVar, coverage] : mCompositionCoverage) {
        if (const auto idOpt = asPhysicalDisplayId(idVar)) {
            resultsPerDisplay.try_emplace(*idOpt, CompositeResult{coverage});
        }
    }

    return resultsPerDisplay;
}
```

- `hasGpuUseOrReuse`告诉`VsyncModulator`:这一帧用了GPU。GPU合成比纯HWC慢,所以接下来两帧改用early offsets,让SF提前醒来,给合成留出更多时间。

`modulateVsync`的第三个布尔参数就是`hasGpuUseOrReuse`,最终落到`VsyncModulator::onDisplayRefresh(bool usedGpuComposition)`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.cpp
VsyncModulator::VsyncConfigOpt VsyncModulator::onDisplayRefresh(bool usedGpuComposition) {
    bool updateOffsetsNeeded = false;

    if (mEarlyTransactionStartTime.load() + MIN_EARLY_TRANSACTION_TIME <=
        mLastTransactionCommitTime.load()) {
        if (mEarlyTransactionFrames > 0) {
            mEarlyTransactionFrames--;
            updateOffsetsNeeded = true;
        }
    }
    if (usedGpuComposition) {
        mEarlyGpuFrames = MIN_EARLY_GPU_FRAMES;
        updateOffsetsNeeded = true;
    } else if (mEarlyGpuFrames > 0) {
        mEarlyGpuFrames--;
        updateOffsetsNeeded = true;
    }

    if (!updateOffsetsNeeded) return std::nullopt;
    return updateVsyncConfig();
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.h
    // Number of frames to keep early offsets after an early transaction or GPU composition.
    // This acts as a low-pass filter in case subsequent transactions are delayed, or if the
    // composition strategy alternates on subsequent frames.
    static constexpr int MIN_EARLY_TRANSACTION_FRAMES = 2;
    static constexpr int MIN_EARLY_GPU_FRAMES = 2;
```

注意这是"低通滤波":一次GPU合成会把`mEarlyGpuFrames`重置为2,之后没有GPU合成的帧只是把这个计数减1,直到减到0才回到`Late`配置:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.cpp
auto VsyncModulator::getNextVsyncConfigType() const -> VsyncConfigType {
    // Early offsets are used if we're in the middle of a refresh rate
    // change, or if we recently begin a transaction.
    if (!mEarlyWakeupRequests.empty() || mTransactionSchedule == Schedule::EarlyEnd ||
        mEarlyTransactionFrames > 0 || mRefreshRateChangePending) {
        return VsyncConfigType::Early;
    } else if (mEarlyGpuFrames > 0) {
        return VsyncConfigType::EarlyGpu;
    } else {
        return VsyncConfigType::Late;
    }
}
```

三档配置`early`、`earlyGpu`、`late`各含`sfOffset`、`appOffset`、`sfWorkDuration`、`appWorkDuration`四个字段,由`PhaseOffsets::getDefaultOffsets`按`vsyncDuration`算出,`sfWorkDuration = vsyncDuration - sfOffset`。SF的`sfOffset`越小,越早醒来,`sfWorkDuration`就越长。

回到`SurfaceFlinger::composite`尾部剩下的几行:

- `mLayersWithQueuedFrames.clear()`:这批Layer的release fence处理已经在`onCompositionPresented`里完成了。
- `updateInputFlinger`:更新input窗口信息,必须在合成之后,因为合成决定了哪些Layer最终可见。
- `mCompositionEngine->needsAnotherUpdate()`:如果`preComposition`里有Layer要求下一帧继续更新,就再排一帧。
- 返回的`resultsPerDisplay`只包含物理显示器,虚拟显示器的覆盖情况不反馈给Scheduler。

---

## 6 composite之后的后处理

### 6.1 SurfaceFlinger::composite里的Post Composition块

`mCompositionEngine->present(refreshArgs)`返回后,`SurfaceFlinger::composite`进入Post Composition阶段:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mCompositionEngine->present(refreshArgs);

    finalizeReadback(refreshArgs.outputs);
```

`finalizeReadback`处理屏幕捕获:如果某个Output配了readback buffer,把`ScreenCaptureResults`填好;否则调用错误回调。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    ftl::Flags<adpf::Workload> compositedWorkload;
    if (refreshArgs.updatingGeometryThisFrame || refreshArgs.updatingOutputGeometryThisFrame) {
        compositedWorkload |= adpf::Workload::VISIBLE_REGION;
    }
    if (mFrontEndDisplayInfosChanged) {
        compositedWorkload |= adpf::Workload::DISPLAY_CHANGES;
        SFTRACE_INSTANT_FOR_TRACK(WorkloadTracer::TRACK_NAME, "Display Changes");
    }

    traceCompositionSummary(layers);
    for (auto& [layer, layerFE] : layers) {
        CompositionResult compositionResult{layerFE->stealCompositionResult()};
        if (layerFE->mSnapshot->hasEffect()) {
            compositedWorkload |= adpf::Workload::EFFECTS;
        }

        if (compositionResult.lastClientCompositionFence) {
            layer->setWasClientComposed(compositionResult.lastClientCompositionFence);
        }
        if (com_android_graphics_libgui_flags_apply_picture_profiles()) {
            mActivePictureTracker.onLayerComposed(*layer, *layerFE, compositionResult);
        }
    }
```

- `traceCompositionSummary`:只在atrace开启时执行,把每层的合成方式压成简短的字符序列(`b`/`rrc`等),方便看trace。
- `LayerFE::stealCompositionResult()`把每层的`CompositionResult`取走。`lastClientCompositionFence`是这一层最后一次被GPU合成时的fence。
- `layer->setWasClientComposed(fence)`把这个信息记到Layer上,供`Layer::onCompositionPresented`判断是否需要打`FALLBACK_COMPOSITION` trace。
- picture profile的处理在flag开启时执行。

然后:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mPowerAdvisor->setCompositedWorkload(compositedWorkload);
    ...
    SFTRACE_NAME_FOR_TRACK(WorkloadTracer::TRACK_NAME, "Post Composition");
    SFTRACE_NAME("postComposition");

    for (const auto& [id, _] : frameTargeters) {
        ftl::FakeGuard guard(mStateLock);
        if (const auto display = getCompositionDisplayLocked(id)) {
            if (!display->isSecure() && display->hasSecureLayers()) {
                mDisplayModeController.startHdcpNegotiation(id);
            }
        }
    }
```

HDCP协商:如果显示器不是secure的,但上面有secure layer,就启动HDCP协商。这是一个异步过程,不会阻塞当前帧。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (!mLayerSnapshotBuilder.hasMergedSnapshots()) {
        moveSnapshotsFromCompositionArgs(layers);
    }
    if (optionalOffloadedRefreshArgs) {
        offloadedCompositionFuture->wait();
        moveSnapshotsFromCompositionArgs(offloadedLayers);
    }
    mTimeStats->recordFrameDuration(pacesetterTarget.frameBeginTime().ns(), systemTime());

    // Send a power hint after presentation is finished.
    if (mPowerHintSessionEnabled) {
        // Now that the current frame has been presented above, PowerAdvisor needs the present time
        // of the previous frame (whose fence is signaled by now) to determine how long the HWC had
        // waited on that fence to retire before presenting.
        // TODO(b/355238809) `presentFenceForPreviousFrame` might not always be signaled (e.g. on
        // devices
        //  where HWC does not block on the previous present fence). Revise this assumtion.
        const auto& previousPresentFence = pacesetterTarget.presentFenceForPreviousFrame();

        mPowerAdvisor->setSfPresentTiming(TimePoint::fromNs(previousPresentFence->getSignalTime()),
                                          TimePoint::now());
        mPowerAdvisor->reportActualWorkDuration();
    }
```

- `moveSnapshotsFromCompositionArgs`:把`refreshArgs`里用完的snapshot收回`LayerSnapshotBuilder`,供下一帧复用。
- 虚拟显示器的offload合成必须在这里`wait()`,确保它的release fence promise也兑现了。
- `recordFrameDuration`:记录SF从`frameBeginTime`到现在的总耗时。
- PowerAdvisor用**上一帧**的present fence signal时间计算HWC等待时间,因为上一帧的fence到现在一定已经signal了。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (mScheduler->onCompositionPresented(presentTime)) {
        scheduleComposite(FrameHint::kNone);
    }
    mNotifyExpectedPresentMap[pacesetterId].hintStatus = NotifyExpectedPresentHintStatus::Start;
    onCompositionPresented(pacesetterId, frameTargeters, presentTime);
```

`Scheduler::onCompositionPresented(presentTime)`处理刷新率切换的时间线:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
bool Scheduler::onCompositionPresented(nsecs_t presentTime) {
    std::lock_guard<std::mutex> lock(mVsyncTimelineLock);
    if (mLastVsyncPeriodChangeTimeline && mLastVsyncPeriodChangeTimeline->refreshRequired) {
        if (presentTime < mLastVsyncPeriodChangeTimeline->refreshTimeNanos) {
            // We need to composite again as refreshTimeNanos is still in the future.
            return true;
        }

        mLastVsyncPeriodChangeTimeline->refreshRequired = false;
    }
    return false;
}
```

切换刷新率时,`mLastVsyncPeriodChangeTimeline->refreshTimeNanos`是"新vsync周期开始生效"的时刻。如果当前present还没到这个时刻,说明还需要再合成一帧,返回true,`SurfaceFlinger`就会调用`scheduleComposite(FrameHint::kNone)`再排一帧。

`mNotifyExpectedPresentMap[pacesetterId].hintStatus`从`ScheduleOnPresent`切到`Start`。`sendNotifyExpectedPresentHint`在composite前后都会调用,用来通知HWC"SF预期的上屏时间",这是`NotifyExpectedPresent`优化的一部分。

### 6.2 onCompositionPresented:present fence与layer回调

`SurfaceFlinger::onCompositionPresented`是后处理的核心。它先收集present fence:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onCompositionPresented(PhysicalDisplayId pacesetterId,
                                            const scheduler::FrameTargeters& frameTargeters,
                                            nsecs_t presentStartTime) {
    SFTRACE_CALL();

    ui::PhysicalDisplayMap<PhysicalDisplayId, std::shared_ptr<FenceTime>> presentFences;
    ui::PhysicalDisplayMap<PhysicalDisplayId, const sp<Fence>> gpuCompositionDoneFences;

    for (const auto& [id, targeter] : frameTargeters) {
        auto presentFence = getHwComposer().getPresentFence(id);

        if (id == pacesetterId) {
            mTransactionCallbackInvoker.addPresentFence(presentFence);
        }

        if (auto fenceTime = targeter->setPresentFence(std::move(presentFence));
            fenceTime->isValid()) {
            presentFences.try_emplace(id, std::move(fenceTime));
        }

        ftl::FakeGuard guard(mStateLock);
        if (const auto display = getCompositionDisplayLocked(id);
            display && display->getState().usesClientComposition) {
            gpuCompositionDoneFences
                    .try_emplace(id, display->getRenderSurface()->getClientTargetAcquireFence());
        }
    }
```

对每个参与本帧的显示器:

1. `getHwComposer().getPresentFence(id)`取present fence。
2. 如果是pacesetter,把present fence交给`TransactionCallbackInvoker`。应用侧的`onTransactionCompleted`会带上这个present fence。
3. `targeter->setPresentFence(std::move(presentFence))`把fence存进`FrameTargeter`的present fence历史。`setPresentFence`内部会调用`addFence`,记录fence和它的`expectedPresentTime`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
FenceTimePtr FrameTargeter::setPresentFence(sp<Fence> presentFence) {
    auto presentFenceTime = std::make_shared<FenceTime>(presentFence);
    return setPresentFence(std::move(presentFence), std::move(presentFenceTime));
}

FenceTimePtr FrameTargeter::setPresentFence(sp<Fence> presentFence, FenceTimePtr presentFenceTime) {
    addFence(std::move(presentFence), presentFenceTime, mExpectedPresentTime);
    return presentFenceTime;
}
```

这里记录的是"这块fence原本期望在哪个时刻signal"。下一帧`beginFrame`里的`expectedSignaledPresentFence`就是拿这些历史记录来判断上一帧是否已经上屏、是否迟到。

4. 如果这个Output做了client合成,取`getClientTargetAcquireFence()`作为GPU合成完成fence。这个fence已经是`FramebufferSurface`里保存的client target fence。

然后:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    const auto pacesetterDisplay = FTL_FAKE_GUARD(mStateLock, getDisplayDeviceLocked(pacesetterId));

    std::shared_ptr<FenceTime> pacesetterPresentFenceTime =
            presentFences.get(pacesetterId)
                    .transform([](const FenceTimePtr& ptr) { return ptr; })
                    .value_or(FenceTime::NO_FENCE);

    std::shared_ptr<FenceTime> pacesetterGpuCompositionDoneFenceTime =
            gpuCompositionDoneFences.get(pacesetterId)
                    .transform([](sp<Fence> fence) {
                        return std::make_shared<FenceTime>(std::move(fence));
                    })
                    .value_or(FenceTime::NO_FENCE);

    const TimePoint presentTime = TimePoint::now();

    // Set presentation information before calling Layer::releasePendingBuffer, such that jank
    // information from previous' frame classification is already available when sending jank info
    // to clients, so they get jank classification as early as possible.
    mFrameTimeline->setSfPresent(presentTime.ns(), pacesetterPresentFenceTime,
                                 pacesetterGpuCompositionDoneFenceTime);
```

注释点出了一个顺序约束:`setSfPresent`必须在`releasePendingBuffer`之前调用。这样应用收到`onTransactionCompleted`时,FrameTimeline已经能给这一帧附上jank分类。

接着构造`CompositorTiming`:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    // We use the CompositionEngine::getLastFrameRefreshTimestamp() which might
    // be sampled a little later than when we started doing work for this frame,
    // but that should be okay since CompositorTiming has snapping logic.
    const TimePoint compositeTime =
            TimePoint::fromNs(mCompositionEngine->getLastFrameRefreshTimestamp());
    const Duration presentLatency = mHasReliablePresentFences
            ? mPresentLatencyTracker.trackPendingFrame(compositeTime, pacesetterPresentFenceTime)
            : Duration::zero();

    const auto schedule = mScheduler->getVsyncSchedule();
    const TimePoint vsyncDeadline = schedule->vsyncDeadlineAfter(presentTime);
    const Fps renderRate = pacesetterDisplay->refreshRateSelector().getActiveMode().fps;
    const nsecs_t vsyncPhase = mScheduler->getCurrentVsyncConfigs().late.sfOffset;

    const CompositorTiming compositorTiming(vsyncDeadline.ns(), renderRate.getPeriodNsecs(),
                                            vsyncPhase, presentLatency.ns());
```

| 字段 | 来源 | 含义 |
|---|---|---|
| `vsyncDeadline` | `schedule->vsyncDeadlineAfter(presentTime)` | 下一个vsync deadline |
| `renderRate` | pacesetter当前刷新率 | 渲染帧率 |
| `vsyncPhase` | `getCurrentVsyncConfigs().late.sfOffset` | SF在vsync周期中的相位偏移 |
| `presentLatency` | `mPresentLatencyTracker.trackPendingFrame(...)` | 从composite到上屏的延迟估计 |

`CompositorTiming`会通过`Layer::onCompositionPresented`传给应用的`CallbackHandle`,应用可以用它来预测下一帧的时序。

然后处理两类Layer:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    ui::DisplayMap<ui::LayerStack, const DisplayDevice*> layerStackToDisplay;
    {
        if (!mLayersWithBuffersRemoved.empty() || mNumTrustedPresentationListeners > 0) {
            Mutex::Autolock lock(mStateLock);
            for (const auto& [token, display] : mDisplays) {
                layerStackToDisplay.emplace_or_replace(display->getLayerStack(), display.get());
            }
        }
    }

    for (auto layer : mLayersWithBuffersRemoved) {
        std::vector<ui::LayerStack> previouslyPresentedLayerStacks =
                std::move(layer->mPreviouslyPresentedLayerStacks);
        layer->mPreviouslyPresentedLayerStacks.clear();
        for (auto layerStack : previouslyPresentedLayerStacks) {
            auto optDisplay = layerStackToDisplay.get(layerStack);
            if (optDisplay && !optDisplay->get()->isVirtual()) {
                auto fence = getHwComposer().getPresentFence(optDisplay->get()->getPhysicalId());
                layer->prepareReleaseCallbacks(ftl::yield<FenceResult>(fence),
                                               ui::UNASSIGNED_LAYER_STACK);
            }
        }
        layer->releasePendingBuffer(presentTime.ns());
    }
    mLayersWithBuffersRemoved.clear();

    for (const auto& [layer, gameMode] : mLayersWithQueuedFrames) {
        layer->onCompositionPresented(pacesetterDisplay.get(),
                                      pacesetterGpuCompositionDoneFenceTime,
                                      pacesetterPresentFenceTime, compositorTiming, gameMode);
        layer->releasePendingBuffer(presentTime.ns());
    }
```

**第一类:`mLayersWithBuffersRemoved`**

这些是`willReleaseBufferOnLatch()`为true的Layer,也就是在latch时就把buffer移除的Layer(例如buffer被显式丢弃)。它们不会走正常的present路径,所以需要特殊处理:

- 遍历`mPreviouslyPresentedLayerStacks`,对每个之前出现过的LayerStack,取对应物理显示器的present fence。
- `prepareReleaseCallbacks(fence, UNASSIGNED_LAYER_STACK)`用present fence兑现release promise。
- `releasePendingBuffer(presentTime)`把回调句柄交给`TransactionCallbackInvoker`。

**第二类:`mLayersWithQueuedFrames`**

这些是本帧latch了新buffer的Layer:

- `layer->onCompositionPresented(...)`记录GPU合成完成fence、present fence、CompositorTiming、game mode,并设置TimeStats和FrameTracer。
- `layer->releasePendingBuffer(presentTime.ns())`处理回调。

`Layer::onCompositionPresented`:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::onCompositionPresented(const DisplayDevice* display,
                                   const std::shared_ptr<FenceTime>& glDoneFence,
                                   const std::shared_ptr<FenceTime>& presentFence,
                                   const CompositorTiming& compositorTiming,
                                   gui::GameMode gameMode) {
    // mFrameLatencyNeeded is true when a new frame was latched for the
    // composition.
    if (!mBufferInfo.mFrameLatencyNeeded) return;

    for (auto& handle : mDrawingState.callbackHandles) {
        handle.gpuCompositionDoneFence = glDoneFence;
        handle.compositorTiming = compositorTiming;
    }

    nsecs_t desiredPresentTime = mBufferInfo.mDesiredPresentTime;

    const int32_t layerId = getSequence();
    mFlinger->mTimeStats->setDesiredTime(layerId, mCurrentFrameNumber, desiredPresentTime);

    const auto outputLayer = findOutputLayerForDisplay(display);
    if (outputLayer && outputLayer->requiresClientComposition()) {
        nsecs_t clientCompositionTimestamp = outputLayer->getState().clientCompositionTimestamp;
        mFlinger->mFrameTracer->traceTimestamp(layerId, getLatchedBufferId(), mCurrentFrameNumber,
                                               clientCompositionTimestamp,
                                               FrameTracer::FrameEvent::FALLBACK_COMPOSITION);
        // Update the SurfaceFrames in the drawing state
        if (mDrawingState.bufferSurfaceFrameTX) {
            mDrawingState.bufferSurfaceFrameTX->setGpuComposition();
        }
        for (auto& [token, surfaceFrame] : mDrawingState.bufferlessSurfaceFramesTX) {
            surfaceFrame->setGpuComposition();
        }
    }

    if (display) {
        const auto activeMode = display->refreshRateSelector().getActiveMode();
        const Fps refreshRate = activeMode.fps;
        const std::optional<Fps> renderRate =
                mFlinger->mScheduler->getFrameRateOverride(getOwnerUid());

        const auto vote = frameRateToSetFrameRateVotePayload(getFrameRateForLayerTree());

        if (presentFence->isValid()) {
            mFlinger->mTimeStats->setPresentFence(layerId, mCurrentFrameNumber, presentFence,
                                                  refreshRate, renderRate, vote, gameMode);
            mFlinger->mFrameTracer->traceFence(layerId, getLatchedBufferId(), mCurrentFrameNumber,
                                               presentFence,
                                               FrameTracer::FrameEvent::PRESENT_FENCE);
        } else if (...) {
            ...
        }
    }

    mFrameStatsHistorySize++;
    mBufferInfo.mFrameLatencyNeeded = false;
}
```

要点:

- `mFrameLatencyNeeded`为false直接返回,说明这一帧没有新latch的buffer,不需要记录延迟。
- 把`glDoneFence`和`compositorTiming`写进所有`CallbackHandle`。
- 如果这一层被GPU合成,打一条`FALLBACK_COMPOSITION` trace,并把SurfaceFrame标记为GPU合成。`FALLBACK_COMPOSITION`在Perfetto里就是"fallback"的意思:本该HWC合成的层掉到了GPU。
- 设置`PRESENT_FENCE` trace和TimeStats。
- 最后清掉`mFrameLatencyNeeded`。

`Layer::releasePendingBuffer`:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::releasePendingBuffer(nsecs_t dequeueReadyTime) {
    for (auto& handle : mDrawingState.callbackHandles) {
        handle.bufferReleaseChannel = mBufferReleaseChannel;
        handle.transformHint = mTransformHint;
        handle.cornerRadii = mCornerRadii;
        handle.dequeueReadyTime = dequeueReadyTime;
        handle.currentMaxAcquiredBufferCount =
                mFlinger->getMaxAcquiredBufferCountForCurrentRefreshRate(mOwnerUid);
        SFTRACE_FORMAT_INSTANT("releasePendingBuffer %s - %" PRIu64, getDebugName(),
                               handle.previousReleaseCallbackId.framenumber);
    }

    for (auto& handle : mDrawingState.callbackHandles) {
        if (handle.releasePreviousBuffer && mPreviousReleaseBufferEndpoint == handle.listener) {
            handle.previousReleaseCallbackId = mPreviousReleaseCallbackId;
            break;
        }
    }

    mFlinger->getTransactionCallbackInvoker().addCallbackHandles(
            std::move(mDrawingState.callbackHandles));
    mDrawingState.callbackHandles.clear();
}
```

- 给每个`CallbackHandle`填上buffer release channel、transform hint、圆角半径、dequeue ready time、当前最大acquired buffer数。
- 对于`releasePreviousBuffer`类型的handle,把`previousReleaseCallbackId`设为`mPreviousReleaseCallbackId`,这是应用侧`releaseBufferCallback`需要的key。
- 把所有handle交给`TransactionCallbackInvoker`,然后清空。

接着发送剩余回调:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
    mTransactionCallbackInvoker.clearCompletedTransactions();

    mTimeStats->incrementTotalFrames();
    mTimeStats->setPresentFenceGlobal(pacesetterPresentFenceTime);

    for (auto&& [id, presentFence] : presentFences) {
        mScheduler->addPresentFence(id, std::move(presentFence));
    }
```

这里`sendCallbacks(false)`的意思是"不限于onCommit,所有回调都发"。此时所有release fence promise都已经兑现,可以安全地把release fence发给应用。

`for (auto&& [id, presentFence] : presentFences) mScheduler->addPresentFence(...)`把每个显示器的present fence交给VsyncController,用于校准vsync模型。

最后:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    const bool hasPacesetterDisplay =
            pacesetterDisplay && getHwComposer().isConnected(pacesetterId);

    if (!hasSyncFramework) {
        if (hasPacesetterDisplay && pacesetterDisplay->isPoweredOn()) {
            mScheduler->enableHardwareVsync(pacesetterId);
        }
    }

    if (hasPacesetterDisplay && !pacesetterDisplay->isPoweredOn()) {
        getRenderEngine().cleanupPostRender();
        return;
    }

    // Cleanup any outstanding resources due to rendering a prior frame.
    getRenderEngine().cleanupPostRender();

    if (mNumTrustedPresentationListeners > 0) {
        // We avoid any reverse traversal upwards so this shouldn't be too expensive
        traverseLegacyLayers([&](Layer* layer) FTL_FAKE_GUARD(kMainThreadContext) {
            if (!layer->hasTrustedPresentationListener()) {
                return;
            }
            const frontend::LayerSnapshot* snapshot =
                    mLayerSnapshotBuilder.getSnapshot(layer->sequence);
            std::optional<const DisplayDevice*> displayOpt = std::nullopt;
            if (snapshot) {
                displayOpt = layerStackToDisplay.get(snapshot->outputFilter.layerStack);
            }
            const DisplayDevice* display = displayOpt.value_or(nullptr);
            layer->updateTrustedPresentationState(display, snapshot,
                                                  nanoseconds_to_milliseconds(presentStartTime),
                                                  false);
        });
    }
```

- `!hasSyncFramework`时,SF自己控制硬件vsync的开关,每帧present后显式`enableHardwareVsync`。
- 显示器已经power off时,只做`cleanupPostRender`然后返回,不再做可信呈现状态更新。
- `getRenderEngine().cleanupPostRender()`清理上一帧RenderEngine的临时资源。注释说"Cleanup any outstanding resources due to rendering a prior frame",也就是说清理的是上一帧的,不是这一帧刚提交的——因为这一帧的GPU工作可能还在执行。
- 可信呈现状态(TrustedPresentation)在present之后更新,因为它需要知道这一帧是否真的按时上屏。

### 6.3 Scheduler的反馈

**addPresentFence**

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::addPresentFence(PhysicalDisplayId id, std::shared_ptr<FenceTime> fence) {
    SFTRACE_NAME(ftl::Concat(__func__, ' ', id.value).c_str());
    const auto scheduleOpt =
            (ftl::FakeGuard(mDisplayLock), mDisplays.get(id)).and_then([](const Display& display) {
                return display.powerMode == hal::PowerMode::OFF
                        ? std::nullopt
                        : std::make_optional(display.schedulePtr);
            });

    if (!scheduleOpt) return;
    const auto& schedule = scheduleOpt->get();

    const bool needMoreSignals = schedule->getController().addPresentFence(std::move(fence));
    if (needMoreSignals) {
        schedule->enableHardwareVsync();
    } else {
        constexpr bool kDisallow = false;
        schedule->disableHardwareVsync(kDisallow);
    }
}
```

- 显示器power off时直接返回,不做校准。
- `getController().addPresentFence(...)`把present fence喂给`VsyncController`。`VsyncController`用present fence的signal时间反推真实vsync相位。
- 返回`needMoreSignals`为true时,说明控制器还需要更多样本,打开硬件vsync;否则关闭硬件vsync,进入软件预测模式。这是Android省电策略的一部分:模型稳定后就不需要每帧都开硬件vsync。

`needMoreSignals`的确切含义写在`VsyncController`接口的注释里。注意android-17.0.0_r1里**没有**`VsyncController.cpp`,`VsyncController`只是纯虚接口,实现在`VSyncReactor`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncController.h
    /*
     * Adds a present fence to the model. The controller will use the fence time as
     * a vsync signal.
     *
     * \param [in] fence    The present fence given from the display
     * \return              True if the model needs more vsync signals to make
     *                      an accurate prediction,
     *                      False otherwise
     */
    virtual bool addPresentFence(std::shared_ptr<FenceTime>) = 0;
```

`VSyncReactor`的实现把`mMoreSamplesNeeded`原样返回,并且在时间戳不被接受时重新置位:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::addPresentFence(std::shared_ptr<FenceTime> fence) {
    SFTRACE_CALL();

    if (!fence) {
        return false;
    }

    nsecs_t const signalTime = fence->getCachedSignalTime();
    if (signalTime == Fence::SIGNAL_TIME_INVALID) {
        return true;
    }

    std::lock_guard lock(mMutex);
    if (mExternalIgnoreFences || mInternalIgnoreFences) {
        SFTRACE_FORMAT_INSTANT("mExternalIgnoreFences=%d mInternalIgnoreFences=%d",
                               mExternalIgnoreFences, mInternalIgnoreFences);
        return true;
    }

    bool timestampAccepted = updateTrackerWithSignaledFences();
    if (signalTime == Fence::SIGNAL_TIME_PENDING) {
        if (mPendingLimit == mUnfiredFences.size()) {
            mUnfiredFences.erase(mUnfiredFences.begin());
        }
        mUnfiredFences.push_back(std::move(fence));
    } else {
        timestampAccepted &=
                addVsyncTimestampLocked(signalTime, VSyncTracker::VsyncTimeSource::PresentFence);
    }

    if (!timestampAccepted) {
        mMoreSamplesNeeded = true;
        setIgnorePresentFencesInternal(true);
        mPeriodConfirmationInProgress = true;
    }

    return mMoreSamplesNeeded;
}
```

从这个实现能读出四个分支:

| 条件 | 返回值 | 含义 |
| --- | --- | --- |
| `fence`为空 | `false` | 没有fence可用,直接告诉调用方不需要硬件vsync |
| `getCachedSignalTime()`为`Fence::SIGNAL_TIME_INVALID` | `true` | fence时间无效,模型拿不到时间戳,仍然需要硬件vsync继续喂 |
| 外部或内部处于忽略present fence状态 | `true` | 当前正在忽略present fence,模型未更新,继续要硬件vsync |
| 时间戳被tracker接受 | `mMoreSamplesNeeded`的原值 | `addPresentFence`自己不重算这个值,原样返回上一次硬件vsync路径(`addHwVsyncTimestamp`)算出来的结果 |
| 时间戳不被接受 | 置`mMoreSamplesNeeded = true`后返回`true` | 相位或周期对不上,需要重新收集样本,同时`setIgnorePresentFencesInternal(true)`把present fence暂时屏蔽,`mPeriodConfirmationInProgress = true`表示正在进行周期确认 |

`SIGNAL_TIME_PENDING`的fence不会立刻喂给tracker,而是压进`mUnfiredFences`队列,等下一次`addPresentFence`开头调用的`updateTrackerWithSignaledFences()`把已经signal的fence取出来补喂。队列长度上限是构造时传入的`kMaxPendingFences`,值为20,满了就丢掉最旧的一个:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
VsyncSchedule::ControllerPtr VsyncSchedule::createController(PhysicalDisplayId id,
                                                             VsyncTracker& tracker,
                                                             FeatureFlags features) {
    // TODO(b/144707443): Tune constants.
    constexpr size_t kMaxPendingFences = 20;
    const bool hasKernelIdleTimer = features.test(Feature::kKernelIdleTimer);

    auto reactor = std::make_unique<VSyncReactor>(id, std::make_unique<SystemClock>(), tracker,
                                                  kMaxPendingFences, hasKernelIdleTimer);

    reactor->setIgnorePresentFences(!features.test(Feature::kPresentFences));
    return reactor;
}
```

`createController`最后一行还揭示了一个前提:`Feature::kPresentFences`关闭时,`setIgnorePresentFences(true)`,此后所有present fence都会被忽略,`addPresentFence`永远走"外部或内部处于忽略present fence状态"那一行,返回`true`,硬件vsync一直开着。

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::updateTrackerWithSignaledFences() {
    bool timestampAccepted = true;
    for (auto it = mUnfiredFences.begin(); it != mUnfiredFences.end();) {
        auto const time = (*it)->getSignalTime();
        if (time == Fence::SIGNAL_TIME_PENDING) {
            it++;
        } else if (time == Fence::SIGNAL_TIME_INVALID) {
            it = mUnfiredFences.erase(it);
        } else {
            timestampAccepted &=
                    addVsyncTimestampLocked(time, VSyncTracker::VsyncTimeSource::PresentFence);

            it = mUnfiredFences.erase(it);
        }
    }
    return timestampAccepted;
}
```

注意这个函数用`&=`累积:只要队列里有一个fence的时间戳没被tracker接受,整个函数就返回`false`。还没signal的fence保留在队列里等下次;`SIGNAL_TIME_INVALID`的直接丢弃。所以`addPresentFence`里的`timestampAccepted`实际上同时表示"历史积压的fence"和"当前这个fence"两条路径的接受情况。

`enableHardwareVsync`这一侧也不是简单的开关。`VsyncSchedule`里维护三态`HwVsyncState`:`Enabled`、`Disabled`、`Disallowed`,初值是`Disallowed`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.h
    enum class HwVsyncState {
        // Hardware VSYNCs are currently enabled.
        Enabled,

        // Hardware VSYNCs are currently disabled. They can be enabled by a call
        // to `enableHardwareVsync`.
        Disabled,

        // Hardware VSYNCs are not currently allowed (e.g. because the display
        // is off).
        Disallowed,

        ftl_last = Disallowed,
    };
    HwVsyncState mHwVsyncState GUARDED_BY(mHwVsyncLock) = HwVsyncState::Disallowed;
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
void VsyncSchedule::enableHardwareVsyncLocked() {
    SFTRACE_CALL();
    if (mHwVsyncState == HwVsyncState::Disabled) {
        mController->resetModel();
        mRequestHardwareVsync(mId, true);
        mHwVsyncState = HwVsyncState::Enabled;
    }
}

void VsyncSchedule::disableHardwareVsync(bool disallow) {
    SFTRACE_CALL();
    std::lock_guard<std::mutex> lock(mHwVsyncLock);
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

- `enableHardwareVsyncLocked`只在状态是`Disabled`时才真正动作,并且**先`mController->resetModel()`再打开硬件vsync**。也就是说从软件预测切回硬件vsync时,模型会清空重新学。
- `disableHardwareVsync(false)`在`Enabled`时先`mRequestHardwareVsync(mId, false)`关闭,再落到`Disabled`;在`Disallowed`时什么都不做。这里的`kDisallow = false`意味着这次关闭**不允许**阻止后续的`enableHardwareVsync`,所以下一帧只要`needMoreSignals`为true就能立刻重新打开。
- 顺带一提,`VsyncSchedule::addResyncSample`用的是同一套逻辑:硬件vsync时间戳喂给`mController->addHwVsyncTimestamp(...)`,`needsHwVsync`决定开还是关。present fence这条线路和硬件vsync时间戳这条线路最终都作用在同一个`HwVsyncState`上。

**endFrame与下一帧的beginFrame**

`endFrame`把本帧的`compositionCoverage`存进`FrameTargeter`:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
void FrameTargeter::endFrame(const CompositeResult& result) {
    mCompositionCoverage = result.compositionCoverage;
}
```

下一帧`beginFrame`里:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/FrameTargeter.cpp
    mHwcFrameMissed = mFrameMissed && mCompositionCoverage.test(CompositionCoverage::Hwc);
    mGpuFrameMissed = mFrameMissed && mCompositionCoverage.test(CompositionCoverage::Gpu);
```

这就是反馈闭环:上一帧是谁合成的,决定了这一帧的丢帧该归咎于HWC还是GPU。`SurfaceFlinger::commit`里的`mBackpressureGpuComposition`也是类似思路:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    if (pacesetterFrameTargetPtr->wouldBackpressureHwc()) {
        if (mBackpressureGpuComposition || pacesetterFrameTargetPtr->didMissHwcFrame()) {
            ...
        }
    }
```

`mBackpressureGpuComposition`在3.1节已经展开:它是`debug.sf.enable_gl_backpressure`属性(默认true)的读值,控制"上一帧的present fence还没signal时,本帧是否直接放弃"。它和`mGpuFrameMissed`是两个不同层次的东西:`mGpuFrameMissed`是根据上一帧的`mCompositionCoverage`归因出来的统计结果,`mBackpressureGpuComposition`是决定要不要施加backpressure的开关。`FrameTargeter::beginFrame`里两者通过`considerBackpressure`发生交集,只有"开关关闭且上一帧没有GPU合成"时才不给宽限。

**sample**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::sample() {
    if (!mLumaSampling || !mRegionSamplingThread) {
        return;
    }

    const auto scheduledFrameResultOpt = mScheduler->getScheduledFrameResult();
    const auto scheduleFrameTimeOpt = scheduledFrameResultOpt
            ? std::optional{scheduledFrameResultOpt->callbackTime}
            : std::nullopt;
    mRegionSamplingThread->onCompositionComplete(scheduleFrameTimeOpt);
}
```

`sample`只在亮度采样开启时工作。它通知`RegionSamplingThread`"这一帧合成完成了",并把本帧计划的回调时间传过去,采样线程据此决定什么时候去读framebuffer。

### 6.4 回调链:release fence如何回到app

这一节把从composite后半段到应用侧`onTransactionCompleted`的完整链路串起来。

**第一步:创建promise**

在`SurfaceFlinger::composite`准备`refreshArgs`时,`prepareLayersForComposition`对每个Layer调用`attachReleaseFenceFutureToLayer`,进而调用`LayerFE::createReleaseFenceFuture`:

```cpp
// frameworks/native/services/surfaceflinger/LayerFE.cpp
ftl::Future<FenceResult> LayerFE::createReleaseFenceFuture() {
    if (mReleaseFencePromiseStatus == ReleaseFencePromiseStatus::INITIALIZED) {
        LOG_ALWAYS_FATAL_IF(!FlagManager::getInstance().frontend_caching_v0(),
                            "Attempting to create a new promise while one is still unfulfilled.");
        ALOGW("Creating a new promise while one is still unfulfilled. Fulfilling with NO_FENCE.");
        mReleaseFence.set_value(Fence::NO_FENCE);
    }
    mReleaseFence = std::promise<FenceResult>();
    mReleaseFencePromiseStatus = ReleaseFencePromiseStatus::INITIALIZED;
    return mReleaseFence.get_future();
}
```

如果上一个promise还没兑现就又要创建新的,在非`frontend_caching_v0`模式下直接fatal。这保证了每个Layer同一时刻最多只有一个未兑现的release promise。

`Layer::prepareReleaseCallbacks`拿到future:

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::prepareReleaseCallbacks(ftl::Future<FenceResult> futureFenceResult,
                                    ui::LayerStack layerStack) {
    CallbackHandle* ch = findCallbackHandle();

    if (ch != nullptr) {
        ch->previousReleaseCallbackId = mPreviousReleaseCallbackId;
        ch->fenceMerger.addFuture(std::move(futureFenceResult));
        ch->name = mName;
    } else {
        // If we didn't get a release callback yet (e.g. some scenarios when capturing
        // screenshots asynchronously) then make sure we don't drop the fence.
        // Older fences for the same layer stack can be dropped when a new fence arrives.
        // An assumption here is that RenderEngine performs work sequentially, so an
        // incoming fence will not fire before an existing fence.
        mAdditionalPreviousReleaseFences.emplace_or_replace(layerStack,
                                                            std::move(futureFenceResult));
    }

    if (mBufferInfo.mBuffer) {
        mPreviouslyPresentedLayerStacks.push_back(layerStack);
    }

    if (mDrawingState.frameNumber > 0) {
        mDrawingState.previousFrameNumber = mDrawingState.frameNumber;
    }
}
```

- 如果能找到`CallbackHandle`,把future加进它的`fenceMerger`,并记录`previousReleaseCallbackId`。
- 找不到handle时(例如异步截图场景),future存进`mAdditionalPreviousReleaseFences`,不能丢。
- 如果本帧确实有buffer,记录本帧出现在哪个LayerStack,供`mLayersWithBuffersRemoved`路径使用。
- 更新`previousFrameNumber`。

**第二步:兑现promise**

在`Output::presentFrameAndReleaseLayers`里,每层调用`LayerFE::setReleaseFence`,即`std::promise::set_value`(见5.9)。`CompositionEngine::postComposition`负责兑底(见5.11)。

**第三步:把fence交给TransactionCallbackInvoker**

`Layer::releasePendingBuffer`把`callbackHandles`交给`TransactionCallbackInvoker::addCallbackHandles`:

```cpp
// frameworks/native/services/surfaceflinger/TransactionCallbackInvoker.cpp
void TransactionCallbackInvoker::addCallbackHandles(std::vector<CallbackHandle>&& handles) {
    for (auto& handle : handles) {
        addCallbackHandle(std::move(handle));
    }
}
```

`addCallbackHandle`:

```cpp
// frameworks/native/services/surfaceflinger/TransactionCallbackInvoker.cpp
void TransactionCallbackInvoker::addCallbackHandle(CallbackHandle&& handle) {
    TransactionStats& transactionStats =
            findOrCreateTransactionStats(handle.listener, handle.callbackIds);

    transactionStats.latchTime = handle.latchTime;
    // If the layer has already been destroyed, don't add the SurfaceControl to the callback.
    // The client side keeps a sp<> to the SurfaceControl so if the SurfaceControl has been
    // destroyed the client side is dead and there won't be anyone to send the callback to.
    sp<IBinder> surfaceControl = handle.surfaceControl.promote();
    if (!surfaceControl) {
        return;
    }

    FrameEventHistoryStats eventStats(handle.frameNumber, handle.previousFrameNumber,
                                      handle.gpuCompositionDoneFence->getSnapshot().fence,
                                      handle.compositorTiming, handle.refreshStartTime,
                                      handle.dequeueReadyTime);
    if (FlagManager::getInstance().fence_handling()) {
        if (handle.previousReleaseCallbackId != ReleaseCallbackId::INVALID_ID) {
            transactionStats.surfaceStats.emplace_back(surfaceControl, handle.acquireTimeOrFence,
                                                       Fence::NO_FENCE, handle.transformHint,
                                                       handle.currentMaxAcquiredBufferCount,
                                                       handle.cornerRadii, eventStats,
                                                       handle.previousReleaseCallbackId);
            mReleaseIdToFenceMerger.try_emplace(handle.previousReleaseCallbackId,
                                                 std::move(handle.fenceMerger));
            if (handle.bufferReleaseChannel) {
                mBufferReleases.emplace_back(std::move(handle.name),
                                             std::move(handle.bufferReleaseChannel),
                                             handle.previousReleaseCallbackId,
                                             sp<Fence>(),
                                             handle.currentMaxAcquiredBufferCount,
                                             FlagManager::getInstance().monitor_buffer_fences()
                                                     ? handle.previousBuffer
                                                     : std::weak_ptr<renderengine::ExternalTexture>());
            }
        }
    } else {
        sp<Fence> previousReleaseFence =
                handle.fenceMerger.waitAndGetFence(handle.name.c_str());
        transactionStats.surfaceStats.emplace_back(surfaceControl, handle.acquireTimeOrFence,
                                                   previousReleaseFence, handle.transformHint,
                                                   handle.currentMaxAcquiredBufferCount,
                                                   handle.cornerRadii, eventStats,
                                                   handle.previousReleaseCallbackId);
        if (handle.bufferReleaseChannel &&
            handle.previousReleaseCallbackId != ReleaseCallbackId::INVALID_ID) {
            ...
            mBufferReleases.emplace_back(std::move(handle.name),
                                         std::move(handle.bufferReleaseChannel),
                                         handle.previousReleaseCallbackId,
                                         std::move(previousReleaseFence),
                                         handle.currentMaxAcquiredBufferCount);
        }
    }
}
```

- `findOrCreateTransactionStats`按listener和callbackIds把同一个事务的多个Layer回调聚合在一起。
- `FrameEventHistoryStats`打包这一帧的帧号、GPU完成fence、CompositorTiming等,用于应用侧计算帧延迟。
- 开启`fence_handling`时,不在这里阻塞等fence,而是把`fenceMerger`存进`mReleaseIdToFenceMerger`,后续由background线程合并。
- 未开启时,`fenceMerger.waitAndGetFence(...)`会阻塞等待future兑现,拿到release fence。
- `mBufferReleases`收集buffer释放通道,用于`Surface::BufferReleaseChannel`,让应用不等binder回调就能知道buffer可以复用。

**第四步:sendCallbacks**

`SurfaceFlinger::onCompositionPresented`调用:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
    mTransactionCallbackInvoker.clearCompletedTransactions();
```

`sendCallbacks`负责把`mCompletedTransactions`和`mBufferReleases`打包,最终通过`ITransactionCompletedListener::onTransactionCompleted`发给应用。在`fence_handling`开启时,它会用`FenceMerger`把同一事务里所有层的fence合并成一个,再通知listener。

**完整时间线**

| 时刻 | 动作 |
|---|---|
| t0 | 应用`queueBuffer`,提交事务 |
| t1 | `setTransactionState`入队,`setTransactionFlags`触发`scheduleFrame` |
| t2 | vsync定时器唤醒,`onFrameSignal` |
| t3 | `commit`→`latchBufferImpl`,acquire fence可能还没signal,`latch-unsignaled`允许先latch |
| t4 | `composite`→`prepareLayersForComposition`,创建release fence promise |
| t5 | GPU合成/RenderEngine,或纯HWC合成 |
| t6 | HWC present,拿到present fence和每层release fence |
| t7 | `setReleaseFence`兑现promise |
| t8 | `FrameTimeline::setSfPresent`,更新jank信息 |
| t9 | `releasePendingBuffer`,把回调交给`TransactionCallbackInvoker` |
| t10 | `sendCallbacks`,应用收到`onTransactionCompleted`,带上release fence |
| t11 | 应用侧`releaseBufferCallback`触发,buffer回到BufferQueue |

注意t7到t10之间,SF只是把fence"记录"下来,并没有等fence signal。应用收到的是fence对象,不是同步等待。真正等fence是应用侧GPU或BufferQueue的事。

### 6.5 FrameTimeline的收尾

FrameTimeline在commit和composite两侧各有一次写入。

**commit侧**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
        mFrameTimeline->setSfWakeUp(ftl::to_underlying(vsyncId),
                                    pacesetterFrameTargetPtr->frameBeginTime().ns(),
                                    Fps::fromPeriodNsecs(vsyncPeriod.ns()),
                                    mScheduler->getPacesetterRefreshRate(), displayState);
```

`setSfWakeUp`告诉FrameTimeline:SF在这个vsync被唤醒,开始处理这一帧。这一步建立了`VsyncId`和`DisplayFrame`的对应关系。

**composite侧**

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
    mFrameTimeline->setSfPresent(presentTime.ns(), pacesetterPresentFenceTime,
                                 pacesetterGpuCompositionDoneFenceTime);
```

`setSfPresent`设置:
- SF认为present完成的时刻(`presentTime`)。
- present fence。
- GPU合成完成fence。

FrameTimeline会在present fence signal后把这一帧标记为完成,并做jank分类。分类依据包括:
- 应用是否按时提交buffer(`SurfaceFrame`的`queueBuffer`时间)。
- SF是否按时latch。
- SF是否按时present。
- 是否走了GPU合成(`FALLBACK_COMPOSITION`)。

**commit没合成时**

如果`commit`返回false,`Scheduler::onFrameSignal`里调用:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
            mSchedulerCallback.onCommitNotComposited();
```

SurfaceFlinger的实现:

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onCommitNotComposited() {
    mFrameTimeline->onCommitNotComposited();
}
```

`FrameTimeline::onCommitNotComposited`会遍历当前DisplayFrame的SurfaceFrame:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/FrameTimeline.cpp
void FrameTimeline::onCommitNotComposited() {
    SFTRACE_CALL();
    std::scoped_lock lock(mMutex);
    mCurrentDisplayFrame->onCommitNotComposited();
    mCurrentDisplayFrame.reset();
    mCurrentDisplayFrame = std::make_shared<DisplayFrame>(mTimeStats, mJankClassificationThresholds,
                                                          &mTraceCookieCounter);
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/FrameTimeline.cpp
void FrameTimeline::DisplayFrame::onCommitNotComposited() {
    for (auto& surfaceFrame : mSurfaceFrames) {
        surfaceFrame->onCommitNotComposited(mRefreshRate, mRenderRate);
    }
}
```

`SurfaceFrame::onCommitNotComposited`的实现说明了这样做的意图:

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/FrameTimeline.cpp
void SurfaceFrame::onCommitNotComposited(Fps refreshRate, Fps displayFrameRenderRate) {
    std::scoped_lock lock(mMutex);

    mDisplayFrameRenderRate = displayFrameRenderRate;
    mActuals.presentTime = mPredictions.presentTime;
    classifyJankLocked(JankType::None, JankType::None, refreshRate, displayFrameRenderRate, nullptr,
                       nullptr);
}
```

它把`mActuals.presentTime`直接设成预测的present时间,并用`JankType::None`、`JankType::None`做分类。也就是说,SF自己放弃的这一帧被当作"按预测时间正常present"记账,不会因为SF没合成而把jank归到应用头上。

`FrameTimeline::onCommitNotComposited`随后执行`mCurrentDisplayFrame.reset()`并新建一个`DisplayFrame`,因为这一帧的`DisplayFrame`已经结账,下一帧必须用新的token对象。

### 6.6 其他后处理

| 动作 | 位置 | 作用 |
|---|---|---|
| `mTimeStats->recordFrameDuration(frameBeginTime, systemTime())` | composite Post Composition | 记录SF整帧耗时 |
| `mPowerAdvisor->setSfPresentTiming(...)`/`reportActualWorkDuration()` | composite Post Composition | ADPF电源提示 |
| `mTimeStats->incrementTotalFrames()` | onCompositionPresented | 帧计数 |
| `mTimeStats->setPresentFenceGlobal(pacesetterPresentFenceTime)` | onCompositionPresented | 全局present fence |
| `mFrameTracer->traceFence(...)` | Layer::onCompositionPresented | Perfetto帧跟踪 |
| `mFpsReporter->dispatchLayerFps(...)` | onCompositionPresented | 帧率上报 |
| `mTunnelModeEnabledReporter->updateTunnelModeStatus()` | onCompositionPresented | 隧道模式状态 |
| HDR layer info listener | onCompositionPresented | HDR信息回调 |
| `getRenderEngine().cleanupPostRender()` | onCompositionPresented | RenderEngine资源清理 |
| `updateTrustedPresentationState(...)` | onCompositionPresented | 可信呈现状态 |
| `mCompositionEngine->needsAnotherUpdate()`→`scheduleCommit` | composite末尾 | 补帧 |
| `scheduleComposite(FrameHint::kNone)` | 刷新率切换未完成 | 补合成帧 |

---

## 7 一帧完整时序表

下表把一次典型的"应用提交buffer→上屏→回调"按调用顺序列出。假设单显示器、无backpressure、走client+HWC混合合成。

| 序号 | 线程 | 调用 | 关键动作 |
|---|---|---|---|
| 1 | 应用binder线程 | `SurfaceFlinger::setTransactionState` | 事务入队 |
| 2 | 应用binder线程 | `setTransactionFlags(eTransactionFlushNeeded, ...)` | 首次调度→`scheduleCommit` |
| 3 | 应用binder线程 | `Scheduler::scheduleFrame`→`MessageQueue::scheduleFrame` | 向VSyncDispatch注册`workDuration` |
| 4 | 定时器线程 | `VSyncDispatchTimerQueue::timerCallback` | 到点,调用`"sf"`回调 |
| 5 | 定时器线程 | `MessageQueue::vsyncCallback` | 生成`VsyncId`,调用`dispatchFrame` |
| 6 | 定时器线程 | `Handler::dispatchFrame` | `sendMessage`到主线程Looper |
| 7 | SF主线程 | `Handler::handleMessage` | 调用`onFrameSignal` |
| 8 | SF主线程 | `Scheduler::onFrameSignal` | pacesetter/follower`beginFrame` |
| 9 | SF主线程 | `SurfaceFlinger::commit` | 显示器/模式/backpressure检查 |
| 10 | SF主线程 | `FrameTimeline::setSfWakeUp` | 记录SF唤醒 |
| 11 | SF主线程 | `updateLayerSnapshots` | 应用事务、生成snapshot、latch buffer |
| 12 | SF主线程 | `Layer::latchBufferImpl`→`gatherBufferInfo` | buffer进入`mBufferInfo` |
| 13 | SF主线程 | `VsyncTracker::onFrameBegin` | 告知tracker本帧即将present |
| 14 | SF主线程 | `TransactionCallbackInvoker::sendCallbacks(true)` | 发onCommit回调 |
| 15 | SF主线程 | `Scheduler::chooseRefreshRateForContent` | 选刷新率 |
| 16 | SF主线程 | `Scheduler::composite`→`SurfaceFlinger::composite` | 组装`CompositionRefreshArgs` |
| 17 | SF主线程 | `prepareLayersForComposition` | 创建release fence promise |
| 18 | SF主线程 | `CompositionEngine::present` | 进入合成 |
| 19 | SF主线程 | `Output::prepare` | 重建layer stack |
| 20 | SF主线程 | `Output::present` | 十一步合成 |
| 21 | SF主线程 | `Display::chooseCompositionStrategy` | HWC validate/presentOrValidate |
| 22 | SF主线程 | `Display::applyCompositionStrategy` | 应用HWC要求,定client/device |
| 23 | SF主线程 | `Output::finishFrame`(预测路径则提前到`prepareFrameAsync`)→`composeSurfaces` | GPU合成 |
| 24 | SF主线程 | `RenderSurface::queueBuffer`→`FramebufferSurface::advanceFrame` | client target交给HWC |
| 25 | SF主线程 | `Display::presentFrame`→`HWComposer::presentAndGetReleaseFences` | HWC present |
| 26 | SF主线程 | `Output::presentFrameAndReleaseLayers` | 生成每层release fence |
| 27 | SF主线程 | `LayerFE::setReleaseFence` | 兑现promise |
| 28 | SF主线程 | `CompositionEngine::postComposition` | 兜底未兑现promise |
| 29 | SF主线程 | `finalizeReadback` | 屏幕捕获收尾 |
| 30 | SF主线程 | `Scheduler::onCompositionPresented` | 刷新率切换检查 |
| 31 | SF主线程 | `SurfaceFlinger::onCompositionPresented` | 收present fence、FrameTimeline、回调 |
| 32 | SF主线程 | `FrameTimeline::setSfPresent` | 记录present |
| 33 | SF主线程 | `Layer::onCompositionPresented` | 记录TimeStats/FrameTracer |
| 34 | SF主线程 | `Layer::releasePendingBuffer` | 回调交给`TransactionCallbackInvoker` |
| 35 | SF主线程 | `TransactionCallbackInvoker::sendCallbacks(false)` | 发release回调 |
| 36 | SF主线程 | `Scheduler::addPresentFence` | present fence喂给VsyncController |
| 37 | SF主线程 | `Scheduler::onFrameSignal`→`compositor.sample` | 亮度采样 |
| 38 | SF主线程 | `FrameTargeter::endFrame` | 记录本帧合成覆盖 |
| 39 | 后台线程 | `TransactionCallbackInvoker` | 合并fence、通知`ITransactionCompletedListener` |
| 40 | 应用 | `onTransactionCompleted` | 拿到release fence |

---

## 8 关键结论

1. **commit和composite是同一个主线程调用栈上的两个函数**,由`Scheduler::onFrameSignal`串起来。没有独立的合成线程(offload只是把present调用扔给worker,主线程仍然等future)。
2. **commit不产生像素**。它应用事务、生成snapshot、latch buffer,产出的是"可合成的状态"和`mustComposite`布尔值。
3. **commit返回false是正常的控制流**,不是错误。热插拔、模式切换、backpressure都会让commit主动放弃本帧。
4. **composite的第一个分叉点是`applyCompositionStrategy`**:`usesClientComposition`和`usesDeviceComposition`两个布尔值决定这一帧是纯GPU、纯HWC还是Mixed。
5. **`presentOrValidate`把validate和present合并成一次HAL调用**:没有client合成且时间合适时,一次调用要么直接present成功(`state == 1`),要么完成validate并带回变更数量到`state == 0`的分支,省掉一次往返。
6. **release fence是一条promise链**:`LayerFE::createReleaseFenceFuture`创建,`Output::presentFrameAndReleaseLayers`兑现,`CompositionEngine::postComposition`兜底,`Layer::releasePendingBuffer`转交,`TransactionCallbackInvoker`合并后发给应用。
7. **composite之后不是结束,而是反馈的开始**:present fence喂给`VsyncController`校准vsync模型,`CompositeResult`告诉下一帧的`FrameTargeter`上一帧是谁合成的,`FrameTimeline`据此做jank分类。
8. **未兑现的release promise必须兜底**,否则应用会耗尽buffer。这就是`CompositionEngine::postComposition`存在的根本原因。

如果把整条链路压缩成一句话:**应用提交事务唤醒SF,SF在vsync时刻先commit把状态准备好,再composite让HWC/GPU把状态变成像素,最后用present fence和release fence把"什么时候上屏、buffer什么时候能复用"告诉每一个等待者。**
