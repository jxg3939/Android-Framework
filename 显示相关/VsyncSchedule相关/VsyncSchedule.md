## 1 来历

在Scheduler初始化时

```cpp
mScheduler->registerDisplay(displayId, connectionType, display->holdRefreshRateSelector());
```

初始化一个VsyncSchedule，分配给一个display

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::registerDisplay(PhysicalDisplayId displayId,
                                ui::DisplayConnectionType connectionType,
                                RefreshRateSelectorPtr selectorPtr) {
    // 创建VsyncSchedule
    auto schedulePtr = std::make_shared<VsyncSchedule>(selectorPtr->getActiveMode().modePtr, mFeatures,
                                            [this](PhysicalDisplayId id, bool enable) {
                                                onHardwareVsyncRequest(id, enable);
                                            });
    // 注册Display
    registerDisplayInternal(displayId, connectionType, std::move(selectorPtr),
                            std::move(schedulePtr));
}
```

## 2 初始化

<a id="VsyncSchedule创建"></a>

```cpp
VsyncSchedule::VsyncSchedule(ftl::NonNull<DisplayModePtr> modePtr, FeatureFlags features,
                             RequestHardwareVsync requestHardwareVsync)
      : mId(modePtr->getPhysicalDisplayId()),
        // 开启硬件Vsync的函数
        mRequestHardwareVsync(std::move(requestHardwareVsync)),
        // VsyncPredictor  负责预测
        mTracker(createTracker(modePtr, features)),
        // VSyncDispatchTimerQueue  负责分发
        mDispatch(createDispatch(mTracker)),
        // VsyncReator  负责控制
        mController(createController(modePtr->getPhysicalDisplayId(), *mTracker, features)),
        mTracer(features.test(Feature::kTracePredictedVsync)
                        ? std::make_unique<PredictedVsyncTracer>(mDispatch)
                        : nullptr) {}

VsyncSchedule::TrackerPtr VsyncSchedule::createTracker(ftl::NonNull<DisplayModePtr> modePtr,
                                                       FeatureFlags features) {
    size_t historySize = 20; // 保留多少个历史样本
    size_t minSamples = 6; // 至少多少个样本才能预测
    const bool isVrr = modePtr->getVrrConfig().has_value();
    const bool hasPresentFences = features.test(Feature::kPresentFences);
    // 开启单样本预测
    if (FlagManager::getInstance().use_last_vsync_predict() && isVrr && hasPresentFences) {
        ALOGI("%s: Using one sample prediction mode for display %s", __func__,
              to_string(modePtr->getPhysicalDisplayId()).c_str());
        historySize = 1;
        minSamples = 1;
    } else {
        ALOGI("%s: Using default prediction mode for display %s", __func__,
              to_string(modePtr->getPhysicalDisplayId()).c_str());
    }
    constexpr uint32_t kDiscardOutlierPercent = 20; // 丢弃多少比例的离群样本
    return std::make_unique<VSyncPredictor>(std::make_unique<SystemClock>(), modePtr, historySize,
                                            minSamples, kDiscardOutlierPercent);
}
VsyncSchedule::DispatchPtr VsyncSchedule::createDispatch(TrackerPtr tracker) {
    using namespace std::chrono_literals;
    constexpr std::chrono::nanoseconds kGroupDispatchWithin = 500us;  // 将触发时间接近的回调合并调度
    constexpr std::chrono::nanoseconds kSnapToSameVsyncWithin = 3ms;  // minVsyncDistance 将符合条件的回调时间对齐到同一Vsync
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
    // 如果没有启用PresentFences，就让reactor忽略PresentFence信息
    reactor->setIgnorePresentFences(!features.test(Feature::kPresentFences));
    return reactor;
}
```

## 3 三个主要成员

| 组件 | 接口文件 | 实现类 | 职责 |
|---|---|---|---|
| `VSyncTracker` | `VSyncTracker.h` | `VSyncPredictor` | 据历史 VSync 时间预测下一次 VSync 时间。预测结果会传给 VSyncDispatchTimerQueue |
| `VsyncController` | `VsyncController.h` | `VSyncReactor` | 负责接收硬件VSync事件和present fence反馈，并把它们喂给Tracker，让Vsync预测更准 |
| `VSyncDispatch` | `VSyncDispatch.h` | `VSyncDispatchTimerQueue` | 负责按预测的VSync时间设置定时器，到点触发回调 |


`VsyncSchedule`中，`VSyncPredictor`、`VSyncDispatchTimerQueue`和`VSyncReactor`分别负责VSYNC时间预测、回调调度和时序跟踪校正。`VSyncPredictor`根据历史VSYNC样本预测未来的VSYNC时间；`VSyncDispatchTimerQueue`使用`Tracker`提供的时间预测安排定时器，并管理回调的触发时机，包括合并时间接近的回调；`VSyncReactor`跟踪实际VSYNC时序，结合硬件事件及Present Fence等信息校正VSYNC模型，使后续预测更贴近实际显示节奏

### 3.1 VSyncTracker

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncTracker.h
class VSyncTracker {
public:
    // 喂一个vsync时间戳(硬件vsync或present fence)来校准软件Vysnc模型，返回此次时间戳是否被接受
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

- `addVsyncTimestamp`喂进的样本,既可以是硬件vsync时间戳(通过`VsyncController::addHwVsyncTimestamp`间接调它),也可以是present fence的签名时间(通过`VsyncController::addPresentFence`间接调它)。tracker对两种来源一视同仁,统一用来拟合软件模型
- `nextAnticipatedVSyncTimeFrom`是核心方法:获取下一次预期Vsync时间，dispatch每排一个回调都要用它算唤醒时间。它保证返回的时间`>= timePoint`
- `lastVsyncOpt`参数是给VRR用的:客户端告诉tracker上次用的Vsync是哪个,tracker据此避免跨过VRR的最小帧周期
- `setRenderRate`处理屏是120Hz但内容只跑60Hz的情况:tracker继续按120Hz追踪硬件vsync时间线,但`nextAnticipatedVSyncTimeFrom`按60Hz的节奏返回

```cpp
nsecs_t VSyncPredictor::nextAnticipatedVSyncTimeFrom(nsecs_t timePoint,
                                                     std::optional<nsecs_t> lastVsyncOpt) {
    SFTRACE_CALL();
    std::lock_guard lock(mMutex);
    const auto now = TimePoint::fromNs(mClock->now());
    // 
    purgeTimelines(now);
    // 保证预测起点不早于已发生的VSync
    if (lastVsyncOpt && *lastVsyncOpt > timePoint) {
        timePoint = *lastVsyncOpt;
    }
    const auto model = getVSyncPredictionModelLocked();
    const auto threshold = model.slope / 2;
    std::optional<Period> minFramePeriodOpt;
    // 一帧需要多个周期
    if (mNumVsyncsForFrame > 1) {
        minFramePeriodOpt = minFramePeriodLocked();
    }
    std::optional<TimePoint> vsyncOpt;
    // 遍历mTimelines 找第一个可用预测
    for (auto& timeline : mTimelines) {
        vsyncOpt = timeline.nextAnticipatedVSyncTimeFrom(model, minFramePeriodOpt,
                    snapToVsync(timePoint), mMissedVsync, 
                    lastVsyncOpt ? snapToVsync(*lastVsyncOpt - threshold): lastVsyncOpt);
        if (vsyncOpt) {
            break;
        }
    }
    LOG_ALWAYS_FATAL_IF(!vsyncOpt);
    // 如果本次预测的Vsync时间大于目前已经承诺过的最大Vsync时间，则更新
    if (*vsyncOpt > mLastCommittedVsync) {
        mLastCommittedVsync = *vsyncOpt;
        SFTRACE_FORMAT_INSTANT("mLastCommittedVsync in %.2fms",
                               float(mLastCommittedVsync.ns() - mClock->now()) / 1e6f);
    }
    return vsyncOpt->ns();
}
```

```cpp
std::optional<TimePoint> VSyncPredictor::VsyncTimeline::nextAnticipatedVSyncTimeFrom(
        Model model, std::optional<Period> minFramePeriodOpt, nsecs_t vsync,
        MissedVsync missedVsync, std::optional<nsecs_t> lastVsyncOpt) {
    if (CC_UNLIKELY(SFTRACE_ENABLED())) {
        SFTRACE_FORMAT("renderRate %s", mRenderRateOpt ? to_string(*mRenderRateOpt).c_str() : "NA");
    }
    nsecs_t vsyncTime = snapToVsyncAlignedWithRenderRate(model, vsync);
    const auto threshold = model.slope / 2;
    const auto lastFrameMissed =
            lastVsyncOpt && std::abs(*lastVsyncOpt - missedVsync.vsync.ns()) < threshold;
    const auto mightBackpressure = minFramePeriodOpt && mRenderRateOpt &&
            mRenderRateOpt->getPeriod() < 2 * (*minFramePeriodOpt);
    if (lastFrameMissed) {
        // If the last frame missed is the last vsync, we already shifted the timeline. Depends
        // on whether we skipped the frame (onFrameMissed) or not (onFrameBegin) we apply a
        // different fixup if we are violating the minFramePeriod.
        // There is no need to shift the vsync timeline again.
        if (vsyncTime - missedVsync.vsync.ns() < minFramePeriodOpt->ns()) {
            vsyncTime += missedVsync.fixup.ns();
            SFTRACE_FORMAT_INSTANT("lastFrameMissed");
        }
    } else if (mightBackpressure && lastVsyncOpt) {
        const auto vsyncDiff = vsyncTime - *lastVsyncOpt;
        if (vsyncDiff <= minFramePeriodOpt->ns() - threshold) {
            // avoid a duplicate vsync
            SFTRACE_FORMAT_INSTANT("skipping a vsync to avoid duplicate frame. next in %.2f "
                                   "which "
                                   "is %.2f "
                                   "from "
                                   "prev. "
                                   "adjust by %.2f",
                                   static_cast<float>(vsyncTime - TimePoint::now().ns()) / 1e6f,
                                   static_cast<float>(vsyncDiff) / 1e6f,
                                   static_cast<float>(mRenderRateOpt->getPeriodNsecs()) / 1e6f);
            vsyncTime += mRenderRateOpt->getPeriodNsecs();
        }
    }

    SFTRACE_FORMAT_INSTANT("vsync in %.2fms", float(vsyncTime - TimePoint::now().ns()) / 1e6f);
    if (isWithin(TimePoint::fromNs(vsyncTime)) == VsyncOnTimeline::Outside) {
        SFTRACE_FORMAT_INSTANT("no longer valid for vsync in %.2f",
                               static_cast<float>(vsyncTime - TimePoint::now().ns()) / 1e6f);
        return std::nullopt;
    }

    return TimePoint::fromNs(vsyncTime);
}
```

### 3.2 VsyncController

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

两条路径校准Controller

喂PresentFence

```cpp
bool VSyncReactor::addPresentFence(std::shared_ptr<FenceTime> fence) {
    // 获取Fence呈现时间
    nsecs_t const signalTime = fence->getCachedSignalTime();
    ...
    // 忽略无效Fence或被禁止使用的Fence
    if (signalTime == Fence::SIGNAL_TIME_INVALID) return true;
    if (mExternalIgnoreFences || mInternalIgnoreFences) return true;
    // 处理已触发的Fence，更新Tracker
    bool timestampAccepted = updateTrackerWithSignaledFences();
    if (signalTime == Fence::SIGNAL_TIME_PENDING) {
        if (mPendingLimit == mUnfiredFences.size()) {
            mUnfiredFences.erase(mUnfiredFences.begin());
        }
        mUnfiredFences.push_back(std::move(fence));
    } else {
        timestampAccepted &= addVsyncTimestampLocked(signalTime, VSyncTracker::VsyncTimeSource::PresentFence);
    }
    // 事件戳未被接受
    if (!timestampAccepted) {
        mMoreSamplesNeeded = true;
        setIgnorePresentFencesInternal(true);
        mPeriodConfirmationInProgress = true;
    }
    return mMoreSamplesNeeded;
}
```

喂HwVsync信号

```cpp
bool VSyncReactor::addHwVsyncTimestamp(nsecs_t timestamp, std::optional<nsecs_t> hwcVsyncPeriod,
                                       bool* periodFlushed, VSyncTracker::VsyncTimeSource source) {
    ...
    if (periodConfirmed(timestamp, hwcVsyncPeriod)) {
        // 刷新周期已确认
        ...
        addVsyncTimestampLocked(timestamp, source);
        endPeriodTransition();
        mMoreSamplesNeeded = mTracker.needsMoreSamples();

    } else if (mPeriodConfirmationInProgress) {
        // 正在确认刷新周期
        mLastHwVsync.set(timestamp);
        if (mLastHwVsync.isFirst()) {
            addVsyncTimestampLocked(*mLastHwVsync.get(), source);
        }
        mMoreSamplesNeeded = true;
        *periodFlushed = false;
    } else {
        // 正常添加采样
        addVsyncTimestampLocked(timestamp, source);
        mMoreSamplesNeeded = mTracker.needsMoreSamples();
    }
    // 只要外部要求忽略Present Fence，就继续保持采样
    if (mExternalIgnoreFences) {
      // keep HWVSync on as long as we ignore present fences.
      mMoreSamplesNeeded = true;
    }
    // 不再需要更多采样时，解除内部忽略Present Fence的状态
    if (!mMoreSamplesNeeded) {
        setIgnorePresentFencesInternal(false);
    }
    return mMoreSamplesNeeded;
}
```

无论走PresentFence还是HwVsync最终都是通过Tracker校准模型

```cpp
bool VSyncReactor::addVsyncTimestampLocked(nsecs_t timestamp,
                                           VSyncTracker::VsyncTimeSource source) {
    const auto accuracy = mTracker.getModelAccuracy(timestamp);
    const bool accepted = mTracker.addVsyncTimestamp(timestamp);
    reportModelAccuracyMetric(accuracy, source, accepted);
    return accepted;
}
```

### 3.3 VsyncDispatch

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatch.h
class VSyncDispatch {
public:
    using Callback = std::function<void(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime)>;
    // 注册客户端回调，返回唯一的CallbackToken
    virtual CallbackToken registerCallback(Callback, std::string callbackName) = 0;
    // 注销回调注册
    virtual void unregisterCallback(CallbackToken token) = 0;
    // 为指定客户端安排一次Vsync回调
    virtual std::optional<ScheduleResult> schedule(CallbackToken, ScheduleTiming) = 0;
    // 更新已经安排的Vsync回调
    virtual std::optional<ScheduleResult> update(CallbackToken, ScheduleTiming) = 0;
    // 取消已安排的Vsync回调
    virtual CancelResult cancel(CallbackToken) = 0;
    ...
};
```

### 3.3.1 三个重要的结构

1. ScheduleTiming

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatch.h
struct ScheduleTiming {
    nsecs_t workDuration = 0;                  // 客户端工作需要的时长
    nsecs_t readyDuration = 0;                 // 客户端要就绪多久（给后续预留多长时间）
    nsecs_t lastVsync = 0;                     // 目标显示时间,会同步到它之后最近的vsync
    std::optional<nsecs_t> committedVsyncOpt;  // 已承诺给客户端的vsync时间
    ...
};
```

回调会在`workDuration + readyDuration`个ns之前于某个vsync被触发

- **内部**消费方(如`"sf"`):`readyDuration = 0`,因为不需要给SF自己额外预留buffer处理时间。
- **外部**消费方(如`"app"`):`readyDuration = sfWorkDuration`,因为app的buffer要经过SF处理,得把SF的处理时间也算进端到端预算,才能给app一个合理的deadline。

2. VSyncCallbackRegistration
本质是VsyncDispatch的代理类
内部持有
    `std::shared_ptr<VSyncDispatch> mDispatch`和
    `std::optional<VSyncDispatch::CallbackToken> mToken`

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
VSyncCallbackRegistration::VSyncCallbackRegistration(std::shared_ptr<VSyncDispatch> dispatch,
                                                     VSyncDispatch::Callback callback,
                                                     std::string callbackName)
      : mDispatch(std::move(dispatch)),
        mToken(mDispatch->registerCallback(std::move(callback), std::move(callbackName))) {}

VSyncCallbackRegistration::VSyncCallbackRegistration(VSyncCallbackRegistration&& other)
      : mDispatch(std::move(other.mDispatch)), mToken(std::exchange(other.mToken, std::nullopt)) {}


// 内部的三个方法都是用自己绑定的mToken（由一个VsyncDispatchTimerQueueEntry生成）传递给VsyncDispatch
std::optional<ScheduleResult> VSyncCallbackRegistration::schedule(
        VSyncDispatch::ScheduleTiming scheduleTiming) {
    if (!mToken) {
        return std::nullopt;
    }
    return mDispatch->schedule(*mToken, scheduleTiming);
}

std::optional<ScheduleResult> VSyncCallbackRegistration::update(
        VSyncDispatch::ScheduleTiming scheduleTiming) {
    if (!mToken) {
        return std::nullopt;
    }
    return mDispatch->update(*mToken, scheduleTiming);
}

CancelResult VSyncCallbackRegistration::cancel() {
    if (!mToken) {
        return CancelResult::Error;
    }
    return mDispatch->cancel(*mToken);
}
```

3. VSyncDispatchTimerQueueEntry

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
// 负责管理单个Vsync回调的任务对象。它负责计算回调应该在哪个Vsync执行、何时唤醒线程，以及管理任务的调度、执行和取消状态
class VSyncDispatchTimerQueueEntry {
    // 三种状态: 
    //          armed: 已调度，保存了计算出的唤醒时间和目标Vsync
    //          running: 回调正在执行。
    //          disarmed：未调度，没有有效的待执行时间。
    // 三种valid transition：
    //          disarmed -> armed(被schedule时)
    //          armed -> running -> disarmed(定时器触发时)
    //          armed -> disarmed(被cancel时)
    ...
    VSyncDispatchTimerQueueEntry::VSyncDispatchTimerQueueEntry(std::string name,
                                                           VSyncDispatch::Callback callback,
                                                           nsecs_t minVsyncDistance)
      : mName(std::move(name)),
        mCallback(std::move(callback)),
        mMinVsyncDistance(minVsyncDistance) {}
    // 根据调度参数和VSyncTracker计算时间，进入armed状态
    ScheduleResult schedule(VSyncDispatch::ScheduleTiming, VSyncTracker&, nsecs_t now);
    // 根据最新Vsync预测更新已调度任务的时间信息
    void update(VSyncTracker&, nsecs_t now);
    // 获取预计唤醒时间
    std::optional<nsecs_t> wakeupTime() const;
    // 获取任务需要准备就绪的时间
    std::optional<nsecs_t> readyTime() const;
    // 获取目标Vsync时间
    std::optional<nsecs_t> targetVsync() const;
    // 取消已调度任务
    void disarm();
    // 定时器触发时记录本次目标时间，进入running
    nsecs_t executing();
    // 执行回调，并完成状态清理
    void callback(nsecs_t vsyncTimestamp, nsecs_t wakeupTimestamp, nsecs_t deadlineTimestamp);
    void ensureNotRunning();
private:
    struct ArmingInfo {
        // 希望回调对应的目标Vsync时间
        nsecs_t mActualWakeupTime;
        // 线程应该被唤醒的时间
        nsecs_t mActualVsyncTime;
        // 工作需要准备就绪的时间点
        nsecs_t mActualReadyTime;
    };
    nsecs_t adjustVsyncIfNeeded(VSyncTracker& tracker, nsecs_t nextVsyncTime) const;
    ArmingInfo getArmedInfo(VSyncTracker&, nsecs_t now, VSyncDispatch::ScheduleTiming,
                            std::optional<ArmingInfo>) const;
    const std::string mName;
    const VSyncDispatch::Callback mCallback; // 实际执行的回调函数
    VSyncDispatch::ScheduleTiming mScheduleTiming; // 当前任务的调度参数
    const nsecs_t mMinVsyncDistance; // 相邻调度目标之间的最小Vsync间隔约束
    std::optional<ArmingInfo> mArmedInfo; // 当前任务的唤醒、就绪和目标时间
    std::optional<nsecs_t> mLastDispatchTime; // 上次分发的时间记录
    std::optional<VSyncDispatch::ScheduleTiming> mWorkloadUpdateInfo;
    mutable std::mutex mRunningMutex;
    std::condition_variable mCv;
    bool mRunning GUARDED_BY(mRunningMutex) = false;
};
```

### 3.3.2 VSyncDispatchTimerQueue

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
class VSyncDispatchTimerQueue : public VSyncDispatch {
public:
    // Constructs a VSyncDispatchTimerQueue.
    // \param[in] tk                    A timekeeper.
    // \param[in] tracker               A tracker.
    // \param[in] timerSlack            The threshold at which different similarly timed callbacks
    //                                  should be grouped into one wakeup.
    // \param[in] minVsyncDistance      The minimum distance between two vsync estimates before the
    //                                  vsyncs are considered the same vsync event.
    VSyncDispatchTimerQueue::VSyncDispatchTimerQueue(std::unique_ptr<TimeKeeper> tk,
                                                    VsyncSchedule::TrackerPtr tracker,
                                                    nsecs_t timerSlack, nsecs_t minVsyncDistance)
        : mTimeKeeper(std::move(tk)),
            mTracker(std::move(tracker)),
            mTimerSlack(timerSlack),
            mMinVsyncDistance(minVsyncDistance) {}

    // 这部分函数继承自VsyncDispatch
    CallbackToken registerCallback(Callback, std::string callbackName) final;
    void unregisterCallback(CallbackToken) final;
    std::optional<ScheduleResult> schedule(CallbackToken, ScheduleTiming) final;
    std::optional<ScheduleResult> update(CallbackToken, ScheduleTiming) final;
    CancelResult cancel(CallbackToken) final;
    void dump(std::string&) const final;

private:
    VSyncDispatchTimerQueue(const VSyncDispatchTimerQueue&) = delete;
    VSyncDispatchTimerQueue& operator=(const VSyncDispatchTimerQueue&) = delete;
    // The static capacity was chosen to exceed the expected number of callbacks.
    // 每个callBack都是VSyncDispatchTimerQueueEntry
    using CallbackMap = ftl::SmallMap<CallbackToken, std::shared_ptr<VSyncDispatchTimerQueueEntry>, 5>;
    void timerCallback();
    void setTimer(nsecs_t, nsecs_t) REQUIRES(mMutex);
    void rearmTimer(nsecs_t now) REQUIRES(mMutex);
    void rearmTimerSkippingUpdateFor(nsecs_t now, CallbackMap::const_iterator skipUpdate) REQUIRES(mMutex);
    void cancelTimer() REQUIRES(mMutex);
    std::optional<ScheduleResult> scheduleLocked(CallbackToken, ScheduleTiming) REQUIRES(mMutex);
    ...
    // 定时器及时间管理抽象
    std::unique_ptr<TimeKeeper> const mTimeKeeper;
    VsyncSchedule::TrackerPtr mTracker;
    // 合并相近回调唤醒时间的容差
    nsecs_t const mTimerSlack;
    // 判断两个预测Vsync是否属于同一Vsync事件的最小时间距离
    nsecs_t const mMinVsyncDistance;
    CallbackToken mCallbackToken GUARDED_BY(mMutex);
    // 保存所有已注册的回调及其调度状态
    CallbackMap mCallbacks GUARDED_BY(mMutex);
    ...
};
```

### 3.3.3 callback注册

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h
VSyncDispatchTimerQueue::CallbackToken VSyncDispatchTimerQueue::registerCallback(
        Callback callback, std::string callbackName) {
    std::lock_guard lock(mMutex);
    // mCallbacks本身就是一个CallbackToken到VSyncDispatchTimerQueueEntry的映射
    return mCallbacks
            .try_emplace(++mCallbackToken,
                         std::make_shared<VSyncDispatchTimerQueueEntry>(std::move(callbackName),
                                                                        std::move(callback),
                                                                        mMinVsyncDistance))
            .first->first;
}

void VSyncDispatchTimerQueue::unregisterCallback(CallbackToken token) {
    std::shared_ptr<VSyncDispatchTimerQueueEntry> entry = nullptr;
    {
        std::lock_guard lock(mMutex);
        auto it = mCallbacks.find(token);
        if (it != mCallbacks.end()) {
            entry = it->second;
            mCallbacks.erase(it->first);
        }
    }
    // 锁外销毁VSyncDispatchTimerQueueEntry
    if (entry) {
        entry->ensureNotRunning();
    }
}
```

### 3.3.4 callback注册

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
std::optional<ScheduleResult> VSyncDispatchTimerQueue::schedule(CallbackToken token,
                                                                ScheduleTiming scheduleTiming) {
    std::lock_guard lock(mMutex);
    return scheduleLocked(token, scheduleTiming);
}
std::optional<ScheduleResult> VSyncDispatchTimerQueue::scheduleLocked(
        CallbackToken token, ScheduleTiming scheduleTiming) {
    // 根据CallbackToken找到VSyncDispatchTimerQueueEntry
    auto it = mCallbacks.find(token);
    if (it == mCallbacks.end()) {
        return {};
    }
    auto& callback = it->second;
    auto const now = mTimeKeeper->now();
    /* If the timer thread will run soon, we'll apply this work update via the callback
     * timer recalculation to avoid cancelling a callback that is about to fire. */
    // 定时器线程计划要醒来的时间点已经过去
    auto const rearmImminent = now > mIntendedWakeupTime;
    // 将callback暂存起来，留待后续Timer回调重新计算调度
    if (CC_UNLIKELY(rearmImminent)) {
        return callback->addPendingWorkloadUpdate(*mTracker, now, scheduleTiming);
    }
    // 正常情况下直接走VSyncDispatchTimerQueueEntry调度
    const auto result = callback->schedule(scheduleTiming, *mTracker, now);
    // callback的wakeupTime比当前定时器计划唤醒时间早，设置一个更早的唤醒，避免callback迟到
    if (callback->wakeupTime() < mIntendedWakeupTime - mTimerSlack) {
        rearmTimerSkippingUpdateFor(now, it);
    }
    return result;
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
// 暂存callback
ScheduleResult VSyncDispatchTimerQueueEntry::addPendingWorkloadUpdate(
        VSyncTracker& tracker, nsecs_t now, VSyncDispatch::ScheduleTiming timing) {
    mWorkloadUpdateInfo = timing;
    const auto armedInfo = getArmedInfo(tracker, now, timing, mArmedInfo);
    return {TimePoint::fromNs(armedInfo.mActualWakeupTime),
            TimePoint::fromNs(armedInfo.mActualVsyncTime)};
}

auto VSyncDispatchTimerQueueEntry::getArmedInfo(VSyncTracker& tracker, nsecs_t now,
                                                VSyncDispatch::ScheduleTiming timing,
                                                std::optional<ArmingInfo> armedInfo) const
        -> ArmingInfo {
    SFTRACE_NAME("VSyncDispatchTimerQueueEntry::getArmedInfo");
    // 从now开始work，最早也要到earliestReadyBy才能ready
    const auto earliestReadyBy = now + timing.workDuration + timing.readyDuration;
    // 不能早于earliestReadyBy，否则无法ready，不能早于lastVsync，否则会选到lastVsync
    const auto earliestVsync = std::max(earliestReadyBy, timing.lastVsync);
    const auto nextVsyncTime =
            adjustVsyncIfNeeded(tracker, /*nextVsyncTime*/
                                tracker.nextAnticipatedVSyncTimeFrom(earliestVsync,
                                                                     timing.lastVsync));
    // 要在VSync前readyDuration完成绘制
    const auto nextReadyTime = nextVsyncTime - timing.readyDuration;
    // 要在ready前workDuration唤醒app
    const auto nextWakeupTime = nextReadyTime - timing.workDuration;
    // tracker算出的nextVsyncTime太晚了，可能会跳过下一个实际的Vsync
    bool const wouldSkipAVsyncTarget =
            armedInfo && (nextVsyncTime > (armedInfo->mActualVsyncTime + mMinVsyncDistance));
    // 计划唤醒app的时间太晚了，可能会跳过下一个应该的唤醒点
    bool const wouldSkipAWakeup =
            armedInfo && (nextWakeupTime > (armedInfo->mActualWakeupTime + mMinVsyncDistance));
    SFTRACE_FORMAT_INSTANT("%s: wouldSkipAVsyncTarget=%d wouldSkipAWakeup=%d", mName.c_str(),
                            wouldSkipAVsyncTarget, wouldSkipAWakeup);
    // 如果重新arm会导致跳过Vsync和唤醒点
    if (wouldSkipAVsyncTarget || wouldSkipAWakeup) {
        return *armedInfo;
    }
    // 
    return ArmingInfo{nextWakeupTime, nextVsyncTime, nextReadyTime};
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
ScheduleResult VSyncDispatchTimerQueueEntry::schedule(VSyncDispatch::ScheduleTiming timing,
                                                      VSyncTracker& tracker, nsecs_t now) {
    SFTRACE_NAME("VSyncDispatchTimerQueueEntry::schedule");
    // 预测下一个Vsync  不能早于上次VSync 也不早于workDuration和readyDuration后
    auto nextVsyncTime =
            tracker.nextAnticipatedVSyncTimeFrom(std::max(timing.lastVsync,
                                                          now + timing.workDuration +
                                                                  timing.readyDuration),
                                                 timing.committedVsyncOpt.value_or(
                                                         timing.lastVsync));
    // 定时器必须在VSync之前workDuration + readyDuration就醒来，留足准备时间
    auto nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
    // 新算出的VSync比上次的目标VSync晚了超过一个周期，代表新Vsync超过太多，可能跳过实际需要的Vsync
    bool const wouldSkipAVsyncTarget =
            mArmedInfo && (nextVsyncTime > (mArmedInfo->mActualVsyncTime + mMinVsyncDistance));
    // 新的唤醒时间比上次的唤醒时间晚了超过一个周期，将会跳过需要的唤醒
    bool const wouldSkipAWakeup =
            mArmedInfo && ((nextWakeupTime > (mArmedInfo->mActualWakeupTime + mMinVsyncDistance)));
    SFTRACE_FORMAT_INSTANT("%s: wouldSkipAVsyncTarget=%d wouldSkipAWakeup=%d", mName.c_str(),
                           wouldSkipAVsyncTarget, wouldSkipAWakeup);
    // 如果跳过tracker给出的预测Vsync和WakeupTime，对齐到mArmedInfo->mActualVsyncTime
    if (wouldSkipAVsyncTarget || wouldSkipAWakeup) {
        nextVsyncTime = mArmedInfo->mActualVsyncTime;
    // 不需要跳过就微调
    } else {
        nextVsyncTime = adjustVsyncIfNeeded(tracker, nextVsyncTime);
    }
    // 最终唤醒时间不能早于now
    nextWakeupTime = std::max(now, nextVsyncTime - timing.workDuration - timing.readyDuration);
    // 更新本VSyncDispatchTimerQueueEntry的ScheduleTiming和ArmedInfo信息
    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mScheduleTiming = timing;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return ScheduleResult{TimePoint::fromNs(nextWakeupTime), TimePoint::fromNs(nextVsyncTime)};
}
```

### 3.3.5 Timer

Timer在VSyncDispatchTimerQueue构造时完成初始化

```cpp
VSyncDispatchTimerQueue::VSyncDispatchTimerQueue(std::unique_ptr<TimeKeeper> tk,
                                                VsyncSchedule::TrackerPtr tracker,
                                                nsecs_t timerSlack, nsecs_t minVsyncDistance)
```

Timer事件驱动机制：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/Timer.cpp
Timer::Timer() {
    reset();
    mDispatchThread = std::thread([this]() { threadMain(); });
}

void Timer::threadMain() {
    while (dispatch()) {
        reset();
    }
}

bool Timer::dispatch() {
    setDebugState(DebugState::Running);
    struct sched_param param = {0};
    param.sched_priority = 2;
    ...
    enum DispatchType : uint32_t { TIMER, TERMINATE, MAX_DISPATCH_TYPE };
    ...
    uint64_t iteration = 0;
    while (true) {
        setDebugState(DebugState::Waiting);
        epoll_event events[DispatchType::MAX_DISPATCH_TYPE];
        // 直到监听的文件描述符出现事件或发生错误
        int nfds = epoll_wait(mEpollFd, events, DispatchType::MAX_DISPATCH_TYPE, -1);
        ...
        for (auto i = 0; i < nfds; i++) {
            if (events[i].data.u32 == DispatchType::TIMER) {
                static uint64_t mIgnored = 0;
                setDebugState(DebugState::Reading);
                read(mTimerFd, &mIgnored, sizeof(mIgnored));
                setDebugState(DebugState::Running);
                std::function<void()> cb;
                {
                    std::lock_guard lock(mMutex);
                    cb = mCallback;
                    mExpectingCallback = false;
                }
                if (cb) {
                    // 执行VSyncDispatchTimerQueue::timerCallback回调
                    setDebugState(DebugState::InCallback);
                    cb();
                    setDebugState(DebugState::Running);
                }
            }
            if (events[i].data.u32 == DispatchType::TERMINATE) {
                ALOGE("Terminated");
                setDebugState(DebugState::Running);
                return false;
            }
        }
    }
}

void Timer::reset() {
    std::function<void()> cb;
    {
        std::lock_guard lock(mMutex);
        // reset时不丢掉一个待执行的VSyncDispatchTimerQueue::timerCallback
        if (mExpectingCallback && mCallback) {
            cb = mCallback;
        }
        cleanup();
        mTimerFd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC | TFD_NONBLOCK);
        mEpollFd = epoll_create1(EPOLL_CLOEXEC);
        if (pipe2(mPipes.data(), O_CLOEXEC | O_NONBLOCK)) {
            ALOGE("could not create TimerDispatch mPipes");
        }
    }
    if (cb) {
        setDebugState(DebugState::InCallback);
        cb();
        setDebugState(DebugState::Running);
    }
    setDebugState(DebugState::Reset);
}
```

与VsyncDispatch交互核心：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/Timer.cpp
void Timer::alarmAt(std::function<void()> callback, nsecs_t time) {
    std::lock_guard lock(mMutex);
    using namespace std::literals;C
    static constexpr int ns_per_s =
            std::chrono::duration_cast<std::chrono::nanoseconds>(1s).count();
    // 设置mCallback  供dispatch中驱动执行
    setCallback(std::move(callback));
    struct itimerspec old_timer;
    // 计算触发时间
    struct itimerspec new_timer {
        .it_interval = {.tv_sec = 0, .tv_nsec = 0},
        .it_value = {.tv_sec = static_cast<long>(time / ns_per_s),
                     .tv_nsec = static_cast<long>(time % ns_per_s)},
    };
    // 设置到点触发
    if (timerfd_settime(mTimerFd, TFD_TIMER_ABSTIME, &new_timer, &old_timer)) {
        ALOGW("Failed to set timerfd %s (%i)", strerror(errno), errno);
    }
}

void Timer::alarmCancel() {
    std::lock_guard lock(mMutex);
    struct itimerspec old_timer;
    // 立即触发
    struct itimerspec new_timer {
        .it_interval = {.tv_sec = 0, .tv_nsec = 0},
        .it_value = {
                .tv_sec = 0,
                .tv_nsec = 0,
        },
    };
    if (timerfd_settime(mTimerFd, 0, &new_timer, &old_timer)) {
        ALOGW("Failed to disarm timerfd");
    }
    // 这里把mCallback置空时持有锁，dispatch中在获取mCallback以及alarmAt中设置Callback时也是要锁的，不会冲突
    setCallback({});
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
void VSyncDispatchTimerQueue::cancelTimer() {
    mIntendedWakeupTime = kInvalidTime;
    mTimeKeeper->alarmCancel();
}

// 设置一个定时醒来的任务  这个函数都是在rearmTimerSkippingUpdateFor中调用
void VSyncDispatchTimerQueue::setTimer(nsecs_t targetTime, nsecs_t /*now*/) {
    mIntendedWakeupTime = targetTime;
    mTimeKeeper->alarmAt(std::bind(&VSyncDispatchTimerQueue::timerCallback, this), mIntendedWakeupTime);
    mLastTimerSchedule = mTimeKeeper->now();
}

void VSyncDispatchTimerQueue::rearmTimer(nsecs_t now) {
    rearmTimerSkippingUpdateFor(now, mCallbacks.cend());
}

void VSyncDispatchTimerQueue::rearmTimerSkippingUpdateFor(nsecs_t now, CallbackMap::const_iterator skipUpdateIt) {
    SFTRACE_CALL();
    std::optional<nsecs_t> min;
    std::optional<nsecs_t> targetVsync;
    std::optional<std::string_view> nextWakeupName;
    for (auto it = mCallbacks.cbegin(); it != mCallbacks.cend(); ++it) {
        auto& callback = it->second;
        // 过滤不需要处理的Entry
        // 1.wakeupTime()没有值，即Entry没有计划的唤醒时间；2.没有被addPendingWorkload
        if (!callback->wakeupTime() && !callback->hasPendingWorkloadUpdate()) {
            continue;
        }
        // 这个callback是要rearm的那个，就不update
        if (it != skipUpdateIt) {
            // 更新这个callback的arminfo
            callback->update(*mTracker, now);
        }
        traceEntry(*callback, now);
        const auto wakeupTime = *callback->wakeupTime();
        // 取最小的wakeuptime
        if (!min || *min > wakeupTime) {
            nextWakeupName = callback->name();
            min = wakeupTime;
            targetVsync = callback->targetVsync();
        }
    }
    if (min && min < mIntendedWakeupTime) {
        setTimer(*min, now);
    } else { // 有可能其他所有callback的wakeuptime都没有mIntendedWakeupTime早 取消掉这次Timer的wakeup
        SFTRACE_NAME("cancel timer");
        cancelTimer();
    }
}

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
        if (!mRunning) {
            ALOGD("TimerQueue is not running. Skipping callback.");
            return;
        }
        auto const now = mTimeKeeper->now();
        mLastTimerCallback = now;
        for (auto it = mCallbacks.begin(); it != mCallbacks.end(); it++) {
            auto& callback = it->second;
            // mArmedInfo->mActualWakeupTime
            auto const wakeupTime = callback->wakeupTime();
            if (!wakeupTime) {
                continue;
            }
            traceEntry(*callback, now);
            // mArmedInfo->mActualReadyTime
            auto const readyTime = callback->readyTime();
            // 相对计划晚了多少
            auto const lagAllowance = std::max(now - mIntendedWakeupTime, static_cast<nsecs_t>(0));
            if (*wakeupTime < mIntendedWakeupTime + mTimerSlack + lagAllowance) {
                // 锁内标记执行
                // nsecs_t VSyncDispatchTimerQueueEntry::executing() {
                    // mLastDispatchTime = mArmedInfo->mActualVsyncTime;
                    // disarm();
                    // return *mLastDispatchTime;
                    // }
                callback->executing();
                // callback->lastExecutedVsyncTarget()返回mLastDispatchTime
                invocations.emplace_back(Invocation{callback, *callback->lastExecutedVsyncTarget(),
                                                    *wakeupTime, *readyTime});
            }
        }
        mIntendedWakeupTime = kInvalidTime;
        rearmTimer(mTimeKeeper->now());
    }
    // 锁外实际执行所有entry内部的callback
    for (auto const& invocation : invocations) {
        ftl::Concat trace(ftl::truncated<5>(invocation.callback->name()));
        SFTRACE_FORMAT("%s: %s", __func__, trace.c_str());
        invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                                      invocation.deadlineTimestamp);
    }
}
```
