## 定位
Scheduler是SurfaceFlinger中负责VSync信号注册与分发的核心组件
VsyncSchedule是Scheduler下一个针对单个物理显示器的、完整的VSync子系统

## 1 Scheduler初始化

<a id="scheduler初始化"></a>

根据当前物理显示屏的刷新能力和系统配置，创建并初始化Scheduler，注册显示屏、建立VSync事件分发机制，最后启动定时器

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::initScheduler(const sp<const DisplayDevice>& display) {
    using namespace scheduler;
    LOG_ALWAYS_FATAL_IF(mScheduler);
    // 当前激活的显示模式，刷新率等
    const auto activeMode = display->refreshRateSelector().getActiveMode();
    const Fps activeRefreshRate = activeMode.fps;
    // 收集Scheduler的要启动的功能
    FeatureFlags features;
    const auto defaultContentDetectionValue = sysprop::enable_frame_rate_override(true);
    if (sysprop::use_content_detection_for_refresh_rate(defaultContentDetectionValue)) {
        features |= Feature::kContentDetection;
        features |= Feature::kSmallDirtyContentDetection;
    }
    if (base::GetBoolProperty("debug.sf.show_predicted_vsync"s, false)) {
        features |= Feature::kTracePredictedVsync;
    }
    if (!base::GetBoolProperty("debug.sf.vsync_reactor_ignore_present_fences"s, false) &&
        mHasReliablePresentFences) {
        features |= Feature::kPresentFences;
    }
    if (display->refreshRateSelector().kernelIdleTimerController()) {
        features |= Feature::kKernelIdleTimer;
    }
    if (mBackpressureGpuComposition) {
        features |= Feature::kBackpressureGpuComposition;
    }
    if (getHwComposer().getComposer()->isSupported(
                Hwc2::Composer::OptionalFeature::ExpectedPresentTime)) {
        features |= Feature::kExpectedPresentTime;
    }
    // 创建Scheduler ICompositor就是surfaceflinger，ISchedulerCallback用于回调surfaceflinger
    {
        std::lock_guard lock(mSchedulerLock);
        mScheduler = std::make_unique<Scheduler>(static_cast<ICompositor&>(*this),
                                                 static_cast<ISchedulerCallback&>(*this), features,
                                                 getFactory(), activeRefreshRate, *mTimeStats);
    }
    // The pacesetter must be registered before EventThread creation below.
    // 在initScheduler中创建的是pacesetter
    const auto displayId = display->getPhysicalId();
    const auto connectionType = mPhysicalDisplays.get(displayId)
                                        .transform(&PhysicalDisplay::snapshotRef)
                                        .transform(&display::DisplaySnapshot::connectionType)
                                        .value_or(ui::DisplayConnectionType::External);
    // 注册显示屏并设置初始渲染帧率
    mScheduler->registerDisplay(displayId, connectionType, display->holdRefreshRateSelector());
    // 设置显示模式
    if (FlagManager::getInstance().modeset_state_machine()) {
        applyActiveMode({.mode = activeMode});
    } else {
        mScheduler->setRenderRate(displayId, activeMode.fps,
                                  /*applyImmediately*/ true);
    }
    const auto configs = mScheduler->getCurrentVsyncConfigs();
    // 这里依据的是android-17.0.0_r1的代码，只创建一个app的EventThread
    mScheduler->createEventThread(mFrameTimeline->getTokenManager(),
                                  /* workDuration */ configs.late.appWorkDuration,
                                  /* readyDuration */ configs.late.sfWorkDuration);

    // 必须先有EventThread，才能确保显示屏连接事件可以通过已经建立的事件分发机制传递给相应客户端
    mScheduler->dispatchHotplug(displayId, scheduler::Scheduler::Hotplug::Connected);
    // 初始化Scheduler的VSync相关机制，并关联FrameTimeline的TokenManager
    mScheduler->initVsync(*mFrameTimeline->getTokenManager(), configs.late.sfWorkDuration);
    mRegionSamplingThread = sp<RegionSamplingThread>::make(*this,
                                           RegionSamplingThread::EnvironmentTimingTunables());
    mFpsReporter = sp<FpsReporter>::make(*mFrameTimeline);
    // Timer callbacks may fire, so do this last.
    mScheduler->startTimers();
}
```
[mScheduler->registerDisplay](#12-注册单个显示器Display)
[mScheduler->createEventThread](../EventThread相关/EventThread.md#createEventThread)
[mScheduler->initVsync](#initVsync)

在以上Scheduler的初始化过程中，有几个重要的调用

### 1.1 Scheduler构造

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
Scheduler::Scheduler(ICompositor& compositor, ISchedulerCallback& callback, FeatureFlags features,
                     surfaceflinger::Factory& factory, Fps activeRefreshRate, TimeStats& timeStats)
        // Scheduler继承了android::impl::MessageQueue，这里先构造基类,让消息队列和surfaceflinger关联
      : android::impl::MessageQueue(compositor),
        mFeatures(features),
        mVsyncConfiguration(factory.createVsyncConfiguration(activeRefreshRate)),
        mVsyncModulator(sp<VsyncModulator>::make(mVsyncConfiguration->getCurrentConfigs())),
        mRefreshRateStats(std::make_unique<RefreshRateStats>(timeStats, activeRefreshRate)),
        // Scheduler后续回调SurfaceFlinger
        mSchedulerCallback(callback) {}
```

### 1.2 注册单个显示器Display

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
// 该函数的主要职责就是创建一个VsyncSchedule，然后注册一个display与之关联绑定
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
[`VsyncSchedule`创建](../VsyncSchedule相关/VsyncSchedule.md#VsyncSchedule创建)
[`registerDisplayInternal`](#registerDisplayInternal)

#### 1.2.1 VsyncSchedule内的三个重要成员

| 组件 | 接口文件 | 实现类 | 职责 |
|---|---|---|---|
| `VSyncTracker` | `VSyncTracker.h` | `VSyncPredictor` | 据历史 VSync 时间预测下一次 VSync 时间。预测结果会传给 VSyncDispatchTimerQueue |
| `VsyncController` | `VsyncController.h` | `VSyncReactor` | 负责接收硬件VSync事件和present fence反馈，并把它们喂给Tracker，让Vsync预测更准 |
| `VSyncDispatch` | `VSyncDispatch.h` | `VSyncDispatchTimerQueue` | 负责按预测的VSync时间设置定时器，到点触发回调 |

```cpp
VsyncSchedule::VsyncSchedule(ftl::NonNull<DisplayModePtr> modePtr, FeatureFlags features,
                             RequestHardwareVsync requestHardwareVsync)
      : mId(modePtr->getPhysicalDisplayId()),
        mRequestHardwareVsync(std::move(requestHardwareVsync)),
        // VsyncPredictor
        mTracker(createTracker(modePtr, features)),
        // VSyncDispatchTimerQueue
        mDispatch(createDispatch(mTracker)),
        // VsyncReator
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
    constexpr std::chrono::nanoseconds kGroupDispatchWithin = 500us;  // 两个回调唤醒时间差在这个范围内就合并成一次唤醒
    constexpr std::chrono::nanoseconds kSnapToSameVsyncWithin = 3ms;  // minVsyncDistance
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

#### 1.2.2 VsyncSchedule初始化完成后注册Display

<a id="registerDisplayInternal"></a>

```cpp
void Scheduler::registerDisplayInternal(PhysicalDisplayId displayId,
                                        ui::DisplayConnectionType connectionType,
                                        RefreshRateSelectorPtr selectorPtr,
                                        VsyncSchedulePtr schedulePtr) {

    // 如果当前没有设置 pacesetter，那么这个显示设备即将成为主显示
    const bool isPrimary = (ftl::FakeGuard(mDisplayLock), !mPacesetterDisplayId);
    // Start the idle timer for the first registered (i.e. primary) display.
    const PromotionParams promotionParams = {
            .toggleIdleTimer = isPrimary &&
                    !FlagManager::getInstance()
                             .follower_arbitrary_refresh_rate_selection_combined()};
    // 停止旧pacesetter的Idle Timer，清理依赖旧pacesetter的状态
    demotePacesetterDisplay(promotionParams);
    RefreshRateSelector& selector = *selectorPtr;
    // 将显示设备插入或替换到mDisplays，并判断该设备是否为首次注册
    const bool isNew = [&]() REQUIRES(kMainThreadContext) {
        std::scoped_lock lock(mDisplayLock);
        return mDisplays
                .emplace_or_replace(displayId, displayId, connectionType, std::move(selectorPtr),
                                    std::move(schedulePtr), mFeatures).second;
    }();
    // 确定并配置新的pacesetter
    promotePacesetterDisplay(promotionParams);
    if (isNew && FlagManager::getInstance().follower_arbitrary_refresh_rate_selection_combined()) {
        initializeIdleTimer(displayId);
        selector.startIdleTimer();
        startPowerTimer(displayId);
    }
    // Disable hardware VSYNC if the registration is new, as opposed to a renewal.
    if (isNew) {
        onHardwareVsyncRequest(displayId, false);
    }
    // 通知当前设备处于连接状态
    dispatchHotplug(displayId, Hotplug::Connected);
}
```

### 1.3 EventThread创建

EventThread主要负责向注册的客户端分发VSync事件，并根据客户端需求管理事件的发送

```cpp
void Scheduler::createEventThread(TokenManager* tokenManager, std::chrono::nanoseconds workDuration,
                                  std::chrono::nanoseconds readyDuration) {
    mEventThread =
            std::make_unique<android::impl::EventThread>("app", getVsyncSchedule(), tokenManager,
                                                         *this, workDuration, readyDuration);
}
```

### 1.4 初始化Scheduler的VSync相关机制

这里面最重要的就是创建mVsync.registration
它是VSyncDispatch和调用方之间的一个回调句柄，负责注册回调、调度回调、取消回调

<a id="initVsync"></a>

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::initVsync(TokenManager& tokenManager, std::chrono::nanoseconds workDuration) {
    Impl::initVsyncInternal(getVsyncSchedule()->getDispatch(), tokenManager, workDuration);
}

// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
// 这里只涉及到了MessageQueue的VSyncCallbackRegistration
void MessageQueue::initVsyncInternal(std::shared_ptr<scheduler::VSyncDispatch> dispatch,
                                     scheduler::TokenManager& tokenManager,
                                     std::chrono::nanoseconds workDuration) {
    std::unique_ptr<scheduler::VSyncCallbackRegistration> oldRegistration;
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.workDuration = workDuration;
        mVsync.tokenManager = &tokenManager;
        // 第一次执行时注册，后续执行都是注册新的 删除旧的
        oldRegistration = onNewVsyncScheduleLocked(std::move(dispatch));
    }
    // 锁外销毁
    oldRegistration.reset();
}

std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    // 检查并尝试取消旧回调
    const bool reschedule = mVsync.registration &&
            mVsync.registration->cancel() == scheduler::CancelResult::Cancelled;
    // move走旧的registration
    auto oldRegistration = std::move(mVsync.registration);
    // 创建新的registration
    mVsync.registration = std::make_unique<
            scheduler::VSyncCallbackRegistration>(std::move(dispatch),
            // 把回调对象绑定为vsyncCallback，等价于
            // [this](nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
            //     vsyncCallback(vsyncTime, targetWakeupTime, readyTime);
            // }
                                                  std::bind(&MessageQueue::vsyncCallback, this,
                                                            std::placeholders::_1,
                                                            std::placeholders::_2,
                                                            std::placeholders::_3),
                                                  "sf");
    // 切换到新的registration后，可能丢失原本待执行的VSync回调，待执行任务迁移到新的registration
    if (reschedule) {
        mVsync.scheduledFrameTimeOpt =
                mVsync.registration->schedule({.workDuration = mVsync.workDuration.get().count(),
                                               .readyDuration = 0,
                                               .lastVsync = mVsync.lastCallbackTime.ns()});
    }
    return oldRegistration;
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
// 用的shared_ptr，VSyncCallbackRegistration活着，VSyncDispatch 就不会被销毁
VSyncCallbackRegistration::VSyncCallbackRegistration(std::shared_ptr<VSyncDispatch> dispatch,
                                                     VSyncDispatch::Callback callback,
                                                     std::string callbackName)
      : mDispatch(std::move(dispatch)),
        mToken(mDispatch->registerCallback(std::move(callback), std::move(callbackName))) {}


// 把传入的MessageQueue::vsyncCallback()函数对象保存到mCallbacks，并返回唯一的CallbackToken，供后续schedule()和 cancel()使用
VSyncDispatchTimerQueue::CallbackToken VSyncDispatchTimerQueue::registerCallback(
        Callback callback, std::string callbackName) {
    std::lock_guard lock(mMutex);
    return mCallbacks.try_emplace(++mCallbackToken,
                         std::make_shared<VSyncDispatchTimerQueueEntry>(std::move(callbackName),
                                                                        std::move(callback),
                                                                        mMinVsyncDistance))
            .first->first;
}
```

### 1.5 startTimers

启动Scheduler中与触摸事件、显示屏空闲及电源管理相关的定时器，相关定时策略开始执行

```cpp
void Scheduler::startTimers() {
    using namespace sysprop;
    using namespace std::string_literals;
    const int32_t defaultTouchTimerValue = sysprop::enable_frame_rate_override(true) ? 200 : 0;
    if (const int32_t millis = set_touch_timer_ms(defaultTouchTimerValue); millis > 0) {
        // Touch events are coming to SF every 100ms, so the timer needs to be higher than that
        // 触摸定时器，在无触摸时逐步降低刷新率
        mTouchTimer.emplace(
                "TouchTimer", std::chrono::milliseconds(millis),
                [this] { touchTimerCallback(TimerState::Reset); },
                [this] { touchTimerCallback(TimerState::Expired); });
        mTouchTimer->start();
    }
    {
        // 驱动对应显示的动态刷新率策略
        ftl::FakeGuard guard(mDisplayLock);
        mShouldStartPowerTimers = true;
        for (const auto& [id, _] : mDisplays) {
            if (FlagManager::getInstance().follower_arbitrary_refresh_rate_selection_combined() ||
                mPacesetterDisplayId == id) {
                startPowerTimer(id);
            }
        }
    }
}
```