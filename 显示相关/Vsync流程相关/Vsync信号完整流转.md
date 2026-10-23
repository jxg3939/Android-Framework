# 一个Vsync信号的完整流转流程

本文以`android-17.0.0_r1`为准，把[EventThread.md](EventThread.md)、[Scheduler.md](Scheduler.md)、[VsyncSchedule.md](VsyncSchedule.md)三份笔记串成一条完整链路：硬件Vsync如何进入SurfaceFlinger、如何被`VSyncPredictor`拟合成软件模型、客户端请求后`VSyncDispatchTimerQueue`如何算出唤醒时间并布防`timerfd`、定时器到点后如何分别唤醒`sf`和`app`两个回调、Vsync事件如何通过`BitTube`送到应用进程，以及PresentFence如何反过来校准模型。

## 0 全景图

一条Vsync信号实际横跨两条闭环：

- 预测/校准闭环：硬件Vsync和PresentFence不断喂给`VSyncPredictor`，让它能算出下一次Vsync时间。
- 请求/分发闭环：客户端请求Vsync，`VSyncDispatchTimerQueue`用预测值设置定时器，到点后回调`sf`和`app`。

```text
硬件Vsync(HWC)
   │
   ▼
SurfaceFlinger::onComposerHalVsync
   │
   ▼
Scheduler::addResyncSample
   │
   ▼
VsyncSchedule::addResyncSample          (只有硬件Vsync处于Enabled态才继续)
   │
   ▼
VSyncReactor::addHwVsyncTimestamp
   │
   ▼
VSyncPredictor::addVsyncTimestamp        (滑动窗口+线性回归,更新mRateMap)
   │
   │ 模型
   ▼
VSyncPredictor::nextAnticipatedVSyncTimeFrom   (被schedule查询)
   ▲
   │ 查询预测
VSyncDispatchTimerQueueEntry::schedule
   │
   ▼
VSyncDispatchTimerQueue::rearmTimerSkippingUpdateFor
   │
   ▼
Timer::alarmAt → timerfd_settime(TFD_TIMER_ABSTIME)
   │
   ▼ 到点(Timer线程)
VSyncDispatchTimerQueue::timerCallback
   │
   ├── Entry "sf"  → MessageQueue::vsyncCallback
   │                    → Scheduler::onFrameSignal
   │                    → FrameTargeter::beginFrame
   │                    → SurfaceFlinger::commit/composite
   │
   └── Entry "app" → EventThread::onVsync
                        → mPendingEvents
                        → EventThread::threadMain
                        → dispatchEvent
                        → EventThreadConnection::postEvent
                        → BitTube(AF_UNIX SOCK_SEQPACKET)
                        → App进程DisplayEventDispatcher
                        → JNI dispatchVsync
                        → Choreographer.onVsync
                        → doFrame
                        → scheduleVsyncLocked(请求下一次)

反馈闭环:
PresentFence → SurfaceFlinger::addPresentFence → Scheduler::addPresentFence
             → VSyncReactor::addPresentFence → VSyncPredictor::addVsyncTimestamp
```

两个回调共用同一个`Timer`和同一个`timerfd`，因此在Perfetto里能看到Timer线程一次唤醒后连续回调`sf`和`app`。

## 1 阶段一：硬件Vsync进入SurfaceFlinger

### 1.1 HWC回调

HWC把硬件Vsync上报给SurfaceFlinger，入口是`SurfaceFlinger::onComposerHalVsync`：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onComposerHalVsync(hal::HWDisplayId hwcDisplayId, int64_t timestamp,
                                        std::optional<hal::VsyncPeriodNanos> vsyncPeriod) {
    SFTRACE_NAME(vsyncPeriod
                         ? ftl::Concat(__func__, ' ', hwcDisplayId, ' ', *vsyncPeriod, "ns").c_str()
                         : ftl::Concat(__func__, ' ', hwcDisplayId).c_str());

    if (timestamp <= 0) {
        ALOGW("%s: Ignoring invalid timestamp %" PRId64 " from HWC display %" PRIu64, __func__,
              timestamp, hwcDisplayId);
        SFTRACE_FORMAT_INSTANT("%s: Ignoring invalid timestamp %" PRId64, __func__, timestamp);
        return;
    }

    Mutex::Autolock lock(mStateLock);
    if (const auto displayIdOpt = getHwComposer().onVsync(hwcDisplayId, timestamp)) {
        REQUIRE_SCHEDULER;

        if (mScheduler->addResyncSample(*displayIdOpt, timestamp, vsyncPeriod,
                                        VSyncTracker::VsyncTimeSource::HwVsyncCallback)) {
            // period flushed
            mScheduler->modulateVsync(displayIdOpt, &VsyncModulator::onRefreshRateChangeCompleted);
        }
    }
}
```

这里有两个要点：

- `getHwComposer().onVsync()`把HWC的`hwcDisplayId`转换成SurfaceFlinger内部的`PhysicalDisplayId`，同时记录这一帧的时间戳。
- `addResyncSample()`返回`true`表示`VSyncReactor`确认刷新周期发生了变化，也就是注释里的`period flushed`，此时要通知`VsyncModulator`刷新率切换完成。

`VsyncTimeSource::HwVsyncCallback`表示这条样本来自HWC主动上报的Vsync回调。

### 1.2 Scheduler转发样本

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
bool Scheduler::addResyncSample(PhysicalDisplayId id, nsecs_t timestamp,
                                std::optional<nsecs_t> hwcVsyncPeriodIn,
                                VSyncTracker::VsyncTimeSource source) {
    if (source == VSyncTracker::VsyncTimeSource::PresentFence ||
        source == VSyncTracker::VsyncTimeSource::Unknown) {
        LOG_ALWAYS_FATAL("Invalid VsyncTimeSource for addResyncSample: %s",
                         ftl::enum_string(source).c_str());
        return false;
    }

    const auto hwcVsyncPeriod = ftl::Optional(hwcVsyncPeriodIn).transform([](nsecs_t nanos) {
        return Period::fromNs(nanos);
    });
    auto schedule = getVsyncSchedule(id);
    if (!schedule) {
        ALOGW("%s: Invalid display %s!", __func__, to_string(id).c_str());
        return false;
    }
    return schedule->addResyncSample(TimePoint::fromNs(timestamp), hwcVsyncPeriod, source);
}
```

这里入口先做一次来源校验：`addResyncSample`只接受硬件Vsync来源，PresentFence走另一条`addPresentFence`路径。然后按`PhysicalDisplayId`找到该显示器对应的`VsyncSchedule`。

### 1.3 VsyncSchedule的门控

样本不是无条件进入`VSyncReactor`，`VsyncSchedule`会先看硬件Vsync当前是否被允许打开：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
bool VsyncSchedule::addResyncSample(TimePoint timestamp, ftl::Optional<Period> hwcVsyncPeriod,
                                    VSyncTracker::VsyncTimeSource source) {
    bool needsHwVsync = false;
    bool periodFlushed = false;
    {
        std::lock_guard<std::mutex> lock(mHwVsyncLock);
        if (mHwVsyncState == HwVsyncState::Enabled) {
            needsHwVsync = mController->addHwVsyncTimestamp(timestamp.ns(),
                                                            hwcVsyncPeriod.transform(&Period::ns),
                                                            &periodFlushed, source);
        }
    }
    if (needsHwVsync) {
        enableHardwareVsync();
    } else {
        constexpr bool kDisallow = false;
        disableHardwareVsync(kDisallow);
    }
    return periodFlushed;
}

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

`mHwVsyncState`有三个取值：`Enabled`、`Disabled`、`Disallowed`。它的实际含义是：

- `Enabled`：HWC正在持续上报硬件Vsync，`VSyncReactor`需要这些样本校准模型。
- `Disabled`：暂时不需要硬件Vsync，HWC的Vsync已经关掉，模型靠PresentFence继续校准。
- `Disallowed`：上层明确禁止打开硬件Vsync，`enableHardwareVsync()`此时无效，只有`isHardwareVsyncAllowed(true)`能解除。

注意`enableHardwareVsyncLocked()`会先调用`mController->resetModel()`，也就是重新开硬件Vsync时把预测模型清空，从头学习新周期。

`mRequestHardwareVsync`是构造`VsyncSchedule`时传进来的回调，最终落到`Scheduler::onHardwareVsyncRequest`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::onHardwareVsyncRequest(PhysicalDisplayId id, bool enabled) {
    static const auto& whence = __func__;
    SFTRACE_NAME(ftl::Concat(whence, ' ', id.value, ' ', enabled).c_str());

    // On main thread to serialize reads/writes of pending hardware VSYNC state.
    static_cast<void>(
            schedule([=, this]() FTL_FAKE_GUARD(mDisplayLock) FTL_FAKE_GUARD(kMainThreadContext) {
                SFTRACE_NAME(ftl::Concat(whence, ' ', id.value, ' ', enabled).c_str());

                if (const auto displayOpt = mDisplays.get(id)) {
                    auto& display = displayOpt->get();
                    display.schedulePtr->setPendingHardwareVsyncState(enabled);

                    if (display.powerMode != hal::PowerMode::OFF) {
                        mSchedulerCallback.requestHardwareVsync(id, enabled);
                    }
                }
            }));
}
```

`Scheduler::schedule()`是`Scheduler.h`里的模板，把任务封装成`Task`后`postMessage()`到SF主线程。真正的开关动作在`SurfaceFlinger::requestHardwareVsync`：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::requestHardwareVsync(PhysicalDisplayId displayId, bool enable) {
    getHwComposer().setVsyncEnabled(displayId, enable ? hal::Vsync::ENABLE : hal::Vsync::DISABLE);

    // Query HWC for the actual Vsync time and provide it to the scheduler when enabled.
    if (enable && FlagManager::getInstance().get_display_known_vsync_sample_enabled()) {
        if (auto sample = getHwComposer().getDisplayKnownVsyncSample(displayId)) {
            if (sample->timestampNs > 0) {
                mScheduler->addResyncSample(displayId, sample->timestampNs, sample->vsyncPeriodNs,
                                            VSyncTracker::VsyncTimeSource::HwVsyncQuery);
            } else {
                ALOGW("%s: Ignoring invalid known Vsync sample timestamp %" PRId64
                      " for display %s",
                      __func__, sample->timestampNs, to_string(displayId).c_str());
                SFTRACE_FORMAT_INSTANT("%s: Ignoring invalid timestamp %" PRId64, __func__,
                                       sample->timestampNs);
            }
        }
    }
}
```

打开硬件Vsync时还会顺手向HWC查一次`getDisplayKnownVsyncSample`，把已知的Vsync时间戳作为第一颗样本直接喂进`Scheduler::addResyncSample`，来源标记为`HwVsyncQuery`。这样模型不用等下一个硬件脉冲就能开始工作。

### 1.4 VSyncReactor接收硬件样本

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::addHwVsyncTimestamp(nsecs_t timestamp, std::optional<nsecs_t> hwcVsyncPeriod,
                                       bool* periodFlushed, VSyncTracker::VsyncTimeSource source) {
    assert(periodFlushed);

    std::lock_guard lock(mMutex);
    if (periodConfirmed(timestamp, hwcVsyncPeriod)) {
        SFTRACE_FORMAT("VSR %" PRIu64 ": period confirmed", mId.value);
        if (mModePtrTransitioningTo) {
            mTracker.setDisplayModePtr(ftl::as_non_null(mModePtrTransitioningTo));
            *periodFlushed = true;
        }

        if (mLastHwVsync.get() && !mLastHwVsync.isFirst()) {
            addVsyncTimestampLocked(*mLastHwVsync.get(), source);
        }
        addVsyncTimestampLocked(timestamp, source);

        endPeriodTransition();
        mMoreSamplesNeeded = mTracker.needsMoreSamples();
    } else if (mPeriodConfirmationInProgress) {
        SFTRACE_FORMAT("VSR %" PRIu64 ": still confirming period", mId.value);
        mLastHwVsync.set(timestamp);
        // Add the first vsync callback to the tracker to be based on a fresh vsync
        if (mLastHwVsync.isFirst()) {
            addVsyncTimestampLocked(*mLastHwVsync.get(), source);
        }
        mMoreSamplesNeeded = true;
        *periodFlushed = false;
    } else {
        SFTRACE_FORMAT("VSR %" PRIu64 ": adding sample", mId.value);
        *periodFlushed = false;
        addVsyncTimestampLocked(timestamp, source);
        mMoreSamplesNeeded = mTracker.needsMoreSamples();
    }

    if (mExternalIgnoreFences) {
      // keep HWVSync on as long as we ignore present fences.
      mMoreSamplesNeeded = true;
    }

    if (!mMoreSamplesNeeded) {
        setIgnorePresentFencesInternal(false);
    }
    return mMoreSamplesNeeded;
}

bool VSyncReactor::addVsyncTimestampLocked(nsecs_t timestamp,
                                           VSyncTracker::VsyncTimeSource source) {
    const auto accuracy = mTracker.getModelAccuracy(timestamp);
    const bool accepted = mTracker.addVsyncTimestamp(timestamp);
    reportModelAccuracyMetric(accuracy, source, accepted);
    return accepted;
}
```

三条分支的语义：

- `periodConfirmed()`返回`true`：当前刷新周期已经确认，`mLastHwVsync`和当前`timestamp`都可以作为有效样本喂给Tracker，然后结束周期切换状态。
- `mPeriodConfirmationInProgress`为`true`但周期还没确认：只记录`mLastHwVsync`，第一次进入时先把这颗样本喂给Tracker做锚点，继续等下一颗样本验证周期。
- 其余情况就是正常运行时的逐颗样本添加。

`addVsyncTimestampLocked()`在喂样本前后各做一件事：喂之前用`getModelAccuracy(timestamp)`记录当前模型对这颗粒子的误差，喂之后用返回值判断Tracker是否接受。误差会通过`reportModelAccuracyMetric()`打到Perfetto的`VSYNC_PREDICTION_ERROR_REPORT`。

`addHwVsyncTimestamp()`的返回值`mMoreSamplesNeeded`决定硬件Vsync是否继续保持打开：还需要样本就继续开着，不需要了就由`VsyncSchedule::addResyncSample`调用`disableHardwareVsync()`关掉。

### 1.5 VSyncPredictor拟合模型

Tracker的具体实现是`VSyncPredictor`。它用固定长度的环形缓冲区保存最近若干个Vsync时间戳，再用线性回归求出周期（斜率）和截距：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncPredictor.cpp
bool VSyncPredictor::addVsyncTimestamp(nsecs_t timestamp) {
    SFTRACE_CALL();

    std::lock_guard lock(mMutex);

    if (!validate(timestamp)) {
        // VSR could elect to ignore the incongruent timestamp or resetModel(). If ts is ignored,
        // don't insert this ts into mTimestamps ringbuffer. If we are still
        // in the learning phase we should just clear all timestamps and start
        // over.
        if (mTimestamps.size() < kMinimumSamplesForPrediction) {
            // Add the timestamp to mTimestamps before clearing it so we could
            // update mKnownTimestamp based on the new timestamp.
            mTimestamps.push_back(timestamp);

            // Do not clear timelines as we don't want to break the phase while
            // we are still learning.
            clearTimestamps(/* clearTimelines */ false);
        } else if (!mTimestamps.empty()) {
            mKnownTimestamp =
                    std::max(timestamp, *std::max_element(mTimestamps.begin(), mTimestamps.end()));
        } else {
            mKnownTimestamp = timestamp;
        }
        SFTRACE_FORMAT_INSTANT("timestamp rejected. mKnownTimestamp was %.2fms ago",
                               (mClock->now() - *mKnownTimestamp) / 1e6f);
        return false;
    }

    if (mTimestamps.size() != kHistorySize) {
        mTimestamps.push_back(timestamp);
        mLastTimestampIndex = next(mLastTimestampIndex);
    } else {
        mLastTimestampIndex = next(mLastTimestampIndex);
        mTimestamps[mLastTimestampIndex] = timestamp;
    }

    traceInt64If("VSP-ts", timestamp);

    const auto [numSamples, oldestTs] = getSampleSizeAndOldestVsync(timestamp);
    traceInt64("VSP-numSamples", static_cast<int64_t>(numSamples));
    mOldestVsync = oldestTs;
    const auto minNumSamples = getMinSamplesRequiredForPrediction();
    if (numSamples < minNumSamples || numSamples == 1) {
        // In one sample prediction mode (minNumSamples == 1), we can't run regression with
        // only one sample. Instead, we anchor the model to the latest pulse (mOldestVsync)
        // and trust that the hardware is running at the ideal period. Setting intercept to 0
        // makes the next prediction exactly: last_pulse + ideal_period.
        mRateMap[idealPeriod()] = {idealPeriod(), 0};
        return true;
    }

    // This is a 'simple linear regression' calculation of Y over X, with Y being the
    // vsync timestamps, and X being the ordinal of vsync count.
    // The calculated slope is the vsync period.
    // ...
    std::vector<nsecs_t> vsyncTS(numSamples);
    std::vector<nsecs_t> ordinals(numSamples);
    // 后面用最小二乘法算出slope和intercept，存进mRateMap
    ...
}
```

关键常量来自`VsyncSchedule::createTracker()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
VsyncSchedule::TrackerPtr VsyncSchedule::createTracker(ftl::NonNull<DisplayModePtr> modePtr,
                                                       FeatureFlags features) {
    size_t historySize = 20;
    size_t minSamples = 6;
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

    constexpr uint32_t kDiscardOutlierPercent = 20;

    return std::make_unique<VSyncPredictor>(std::make_unique<SystemClock>(), modePtr, historySize,
                                            minSamples, kDiscardOutlierPercent);
}
```

正常模式下攒够6颗样本才开始回归，最多保留20颗；`outlierTolerancePercent = 20`表示`validate()`允许样本偏离预期Vsync最多20%，超出这个范围的时间戳会被判为离群并拒绝。VRR且开启PresentFence时退化成单样本预测，直接把模型锚在最后一颗脉冲上，认为下一个Vsync就是`last_pulse + ideal_period`。

预测接口是`nextAnticipatedVSyncTimeFrom()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncPredictor.cpp
nsecs_t VSyncPredictor::nextAnticipatedVSyncTimeFrom(nsecs_t timePoint,
                                                     std::optional<nsecs_t> lastVsyncOpt) {
    SFTRACE_CALL();
    std::lock_guard lock(mMutex);

    const auto now = TimePoint::fromNs(mClock->now());
    purgeTimelines(now);

    if (lastVsyncOpt && *lastVsyncOpt > timePoint) {
        timePoint = *lastVsyncOpt;
    }
    const auto model = getVSyncPredictionModelLocked();
    const auto threshold = model.slope / 2;
    std::optional<Period> minFramePeriodOpt;
    if (mNumVsyncsForFrame > 1) {
        minFramePeriodOpt = minFramePeriodLocked();
    }

    std::optional<TimePoint> vsyncOpt;
    for (auto& timeline : mTimelines) {
        vsyncOpt = timeline.nextAnticipatedVSyncTimeFrom(model, minFramePeriodOpt,
                                                         snapToVsync(timePoint), mMissedVsync,
                                                         lastVsyncOpt ? snapToVsync(*lastVsyncOpt -
                                                                                    threshold)
                                                                      : lastVsyncOpt);
        if (vsyncOpt) {
            break;
        }
    }
    LOG_ALWAYS_FATAL_IF(!vsyncOpt);

    if (*vsyncOpt > mLastCommittedVsync) {
        mLastCommittedVsync = *vsyncOpt;
        SFTRACE_FORMAT_INSTANT("mLastCommittedVsync in %.2fms",
                               float(mLastCommittedVsync.ns() - mClock->now()) / 1e6f);
    }

    return vsyncOpt->ns();
}
```

这段是整条链路的“心脏”：

- 保证返回值不早于`timePoint`。
- `lastVsyncOpt`给了VRR场景一个下限，避免预测跨过客户端已经用过的Vsync。
- 模型会把`timePoint`先`snapToVsync()`对齐到Vsync网格，再按`mTimelines`里记录的相位算下一个点。
- `mNumVsyncsForFrame > 1`对应120Hz屏幕按60Hz渲染的场景，此时预测按`minFramePeriod`的倍数跳。

## 2 阶段二：客户端请求Vsync

以应用进程的`Choreographer`为例。`Choreographer`在`doFrame()`末尾或者`postCallback`时调用`scheduleVsyncLocked()`：

```java
// frameworks/base/core/java/android/view/Choreographer.java
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
private void scheduleVsyncLocked() {
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#scheduleVsyncLocked");
        mDisplayEventReceiver.scheduleVsync();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

`FrameDisplayEventReceiver`继承自`DisplayEventReceiver`，所以`scheduleVsync()`是`DisplayEventReceiver.java`的方法：

```java
// frameworks/base/core/java/android/view/DisplayEventReceiver.java
/**
 * Schedules a single vertical sync pulse to be delivered when the next
 * display frame begins.
 */
@UnsupportedAppUsage
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

JNI层把调用转给`NativeDisplayEventReceiver`：

```cpp
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    status_t status = receiver->scheduleVsync();
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.c_str());
    }
}
```

`NativeDisplayEventReceiver`继承自`DisplayEventDispatcher`，真正发出Binder请求的是父类：

```cpp
// frameworks/native/libs/gui/DisplayEventDispatcher.cpp
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);

        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        PhysicalDisplayId vsyncDisplayId;
        uint32_t vsyncCount;
        VsyncEventData vsyncEventData;
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "", this,
                  ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }

        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
        mLastScheduleVsyncTime = systemTime(SYSTEM_TIME_MONOTONIC);
    }
    return OK;
}
```

`mWaitingForVsync`是一个去重门闩：已经请求过但还没收到Vsync时，重复调用`scheduleVsync()`直接返回，不会向SurfaceFlinger重复发Binder请求。

`mReceiver.requestNextVsync()`是`DisplayEventReceiver`的C++方法，Binder调用到SurfaceFlinger侧的`EventThreadConnection::requestNextVsync()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
binder::Status EventThreadConnection::requestNextVsync() {
    SFTRACE_CALL();
    mEventThread->requestNextVsync(sp<EventThreadConnection>::fromExisting(this));
    return binder::Status::ok();
}

void EventThread::requestNextVsync(const sp<EventThreadConnection>& connection) {
    mCallback.resync(IEventThreadCallback::ResyncCaller::RequestNextVsync);

    std::lock_guard<std::mutex> lock(mMutex);

    if (connection->vsyncRequest == VSyncRequest::None) {
        connection->vsyncRequest = VSyncRequest::Single;
        mCondition.notify_all();
    } else if (connection->vsyncRequest == VSyncRequest::SingleSuppressCallback) {
        connection->vsyncRequest = VSyncRequest::Single;
    }
}
```

这里有两个细节：

- `mCallback.resync()`要求Scheduler检查模型是否已经偏离硬件太久，必要时重新同步硬件Vsync。
- `VSyncRequest::Single`和`SingleSuppressCallback`的区别：`Single`表示请求一次Vsync，EventThread会唤醒等待中的线程；`SingleSuppressCallback`表示上一次Vsync已经投递但还没确认，此时再请求只改状态不唤醒，避免应用还没消费完就被下一次信号打断。

`resync()`的实现如下：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::resync(ResyncCaller caller) {
    static constexpr int64_t kDefaultResyncOnChoreographerTimeout = ms2ns(750);
    static const int64_t resyncOnTxTimeout =
            sysprop::resync_on_tx_timeout(VSyncTracker::kPredictorThreshold.ns());
    static const int64_t resyncOnChoreographerTimeout =
            sysprop::resync_on_choreographer_timeout(kDefaultResyncOnChoreographerTimeout);

    const nsecs_t now = systemTime();
    const nsecs_t last = (caller == ResyncCaller::Transaction) ? mLastResyncTimeOnTx.exchange(now)
                                                    : mLastResyncTime.exchange(now);
    const nsecs_t ignoreDelay = (caller == ResyncCaller::Transaction) ? resyncOnTxTimeout
                                                        : resyncOnChoreographerTimeout;

    if (ignoreDelay != 0 && now - last > ignoreDelay) {
        resyncAllToHardwareVsync(false /* allowToEnable */);
    }
}
```

`ResyncCaller::RequestNextVsync`对应的阈值默认是750ms。超过750ms没有resync时，`resyncAllToHardwareVsync()`会对所有非OFF的显示器调用`resyncToHardwareVsyncLocked()`，进而走到`VsyncSchedule::onDisplayModeChanged()`：它先调用`mController->onDisplayModeChanged()`，`VSyncReactor`按需进入`startPeriodTransitionInternal()`把`mPeriodConfirmationInProgress`置位；随后`VsyncSchedule::onDisplayModeChanged()`再调用`enableHardwareVsyncLocked()`打开硬件Vsync。这样即使模型长时间没有校准，也会被一次应用请求拉回硬件时间线。

`EventThreadConnection`是Binder服务端，它的`mChannel`是一个`gui::BitTube`，之后Vsync事件就从这里写回应用进程。

## 3 阶段三：计算目标Vsync并布防Timer

### 3.1 EventThread决定调度

`EventThread`自己的线程运行`threadMain()`，每次循环都会检查是否有连接请求Vsync：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
bool vsyncRequested = false;

// Find connections that should consume this event.
auto it = mDisplayEventConnections.begin();
while (it != mDisplayEventConnections.end()) {
    if (const auto connection = it->promote()) {
        if (event && shouldConsumeEvent(*event, connection)) {
            consumers.push_back(connection);
        }

        vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;

        ++it;
    } else {
        mDisplayEventConnections.erase(it);
    }
}

if (!consumers.empty()) {
    dispatchEvent(*event, consumers);
    consumers.clear();
}

if (mVSyncState && vsyncRequested) {
    const bool vsyncOmitted =
            FlagManager::getInstance().no_vsyncs_on_screen_off() && mVSyncState->omitted;
    if (vsyncOmitted) {
        updateState(State::Idle);
        SFTRACE_INT("VsyncPendingScreenOn", 1);
    } else {
        updateState(mVSyncState->synthetic ? State::SyntheticVSync : State::VSync);
        if (FlagManager::getInstance().no_vsyncs_on_screen_off()) {
            SFTRACE_INT("VsyncPendingScreenOn", 0);
        }
    }
} else {
    ALOGW_IF(!mVSyncState, "Ignoring VSYNC request while display is disconnected");
    updateState(State::Idle);
}

if (mState == State::VSync) {
    const auto scheduleResult = mVsyncRegistration.schedule(
            {.workDuration = mWorkDuration.get().count(),
             .readyDuration = mReadyDuration.count(),
             .lastVsync = mLastVsyncCallbackTime.ns(),
             .committedVsyncOpt = mLastCommittedVsyncTime.ns()});
    LOG_ALWAYS_FATAL_IF(!scheduleResult, "Error scheduling callback");
} else {
    mVsyncRegistration.cancel();
}
```

`mVsyncRegistration`是`VSyncCallbackRegistration`，它在`EventThread`构造函数里创建：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
EventThread::EventThread(const char* name, std::shared_ptr<scheduler::VsyncSchedule> vsyncSchedule,
                         android::scheduler::TokenManager* tokenManager,
                         IEventThreadCallback& callback, std::chrono::nanoseconds workDuration,
                         std::chrono::nanoseconds readyDuration)
      : mThreadName(name),
        mEventThreadStateName(base::StringPrintf("EventThreadState-%s", name)),
        mVsyncTracer(base::StringPrintf("VSYNC-%s", name), 0),
        mWorkDuration(base::StringPrintf("VsyncWorkDuration-%s", name), workDuration),
        mReadyDuration(readyDuration),
        mVsyncSchedule(std::move(vsyncSchedule)),
        mVsyncRegistration(mVsyncSchedule->getDispatch(), createDispatchCallback(), name),
        mTokenManager(tokenManager),
        mCallback(callback) {
    ...
}
```

`name`是`"app"`，所以这个回调在`VSyncDispatchTimerQueue`里注册的名字就是`"app"`。SurfaceFlinger自己的回调在`MessageQueue::onNewVsyncScheduleLocked()`里注册，名字是`"sf"`。

三个时间参数的含义：

- `workDuration`：客户端从被唤醒到完成工作需要的时长。应用侧来自`configs.late.appWorkDuration`。
- `readyDuration`：回调时间点距离目标Vsync还要预留的时长。应用侧来自`configs.late.sfWorkDuration`，因为应用画完后还要经过SurfaceFlinger合成。
- `lastVsync`：上一次回调对应的目标Vsync时间。
- `committedVsyncOpt`：上一次已经承诺给客户端的Vsync时间，用于VRR下避免重复承诺同一个Vsync。

### 3.2 VSyncCallbackRegistration到VSyncDispatch

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
std::optional<ScheduleResult> VSyncCallbackRegistration::schedule(
        VSyncDispatch::ScheduleTiming scheduleTiming) {
    if (!mToken) {
        return std::nullopt;
    }
    return mDispatch->schedule(*mToken, scheduleTiming);
}

std::optional<ScheduleResult> VSyncDispatchTimerQueue::schedule(CallbackToken token,
                                                                ScheduleTiming scheduleTiming) {
    std::lock_guard lock(mMutex);
    return scheduleLocked(token, scheduleTiming);
}

std::optional<ScheduleResult> VSyncDispatchTimerQueue::scheduleLocked(
        CallbackToken token, ScheduleTiming scheduleTiming) {
    auto it = mCallbacks.find(token);
    if (it == mCallbacks.end()) {
        return {};
    }
    auto& callback = it->second;
    auto const now = mTimeKeeper->now();

    /* If the timer thread will run soon, we'll apply this work update via the callback
     * timer recalculation to avoid cancelling a callback that is about to fire. */
    auto const rearmImminent = now > mIntendedWakeupTime;
    if (CC_UNLIKELY(rearmImminent)) {
        return callback->addPendingWorkloadUpdate(*mTracker, now, scheduleTiming);
    }

    const auto result = callback->schedule(scheduleTiming, *mTracker, now);

    if (callback->wakeupTime() < mIntendedWakeupTime - mTimerSlack) {
        rearmTimerSkippingUpdateFor(now, it);
    }

    return result;
}
```

`mCallbacks`的键是注册时生成的`CallbackToken`，值是一个`VSyncDispatchTimerQueueEntry`。`scheduleLocked()`分两种情况：

- `now > mIntendedWakeupTime`：定时器线程马上要跑或者已经该跑但还没处理，此时不能直接改这个Entry，否则可能取消一个即将触发的回调。把这次`ScheduleTiming`存进`mWorkloadUpdateInfo`，等`timerCallback()`重新计算时再合并。
- 正常情况：直接调用`VSyncDispatchTimerQueueEntry::schedule()`算新的时间，如果算出的唤醒时间比当前定时器计划早超过`mTimerSlack`，立刻重排定时器。

### 3.3 Entry计算三个时间点

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

    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mScheduleTiming = timing;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return ScheduleResult{TimePoint::fromNs(nextWakeupTime), TimePoint::fromNs(nextVsyncTime)};
}
```

时间计算可以拆成四步：

1. `earliestVsync = max(lastVsync, now + workDuration + readyDuration)`：目标Vsync不能早于上次Vsync，也不能早到客户端来不及准备。
2. `nextVsyncTime = tracker.nextAnticipatedVSyncTimeFrom(earliestVsync, committedVsync)`：问Tracker要“不早于`earliestVsync`的下一个Vsync”。
3. `nextReadyTime = nextVsyncTime - readyDuration`：客户端必须在这个时间点前把buffer交给SurfaceFlinger。
4. `nextWakeupTime = nextReadyTime - workDuration`：再往前推`workDuration`，就是客户端线程该被唤醒的时间。

`wouldSkipAVsyncTarget`和`wouldSkipAWakeup`是防跳帧保护：如果新算出来的目标Vsync比上一次已经armed的目标晚了超过`mMinVsyncDistance`（默认3ms），说明预测可能跳过了本该命中 的那个Vsync，这时不再采用新值，直接沿用`mArmedInfo->mActualVsyncTime`。

`adjustVsyncIfNeeded()`处理另外两个边界：

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

- `alreadyDispatchedForVsync`：新目标Vsync和上次已经分发过的Vsync在3ms以内，说明是同一个Vsync，必须往后推一个，不能对同一个Vsync回调两次。
- `nextVsyncTooClose`：新目标Vsync离上次分发时间太近，不足一个Vsync周期，同样往后推一个周期。

### 3.4 重排Timer

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
void VSyncDispatchTimerQueue::rearmTimerSkippingUpdateFor(
        nsecs_t now, CallbackMap::const_iterator skipUpdateIt) {
    SFTRACE_CALL();
    std::optional<nsecs_t> min;
    std::optional<nsecs_t> targetVsync;
    std::optional<std::string_view> nextWakeupName;
    for (auto it = mCallbacks.cbegin(); it != mCallbacks.cend(); ++it) {
        auto& callback = it->second;
        if (!callback->wakeupTime() && !callback->hasPendingWorkloadUpdate()) {
            continue;
        }

        if (it != skipUpdateIt) {
            callback->update(*mTracker, now);
        }

        traceEntry(*callback, now);

        const auto wakeupTime = *callback->wakeupTime();
        if (!min || *min > wakeupTime) {
            nextWakeupName = callback->name();
            min = wakeupTime;
            targetVsync = callback->targetVsync();
        }
    }

    if (min && min < mIntendedWakeupTime) {
        setTimer(*min, now);
    } else {
        SFTRACE_NAME("cancel timer");
        cancelTimer();
    }
}

void VSyncDispatchTimerQueue::setTimer(nsecs_t targetTime, nsecs_t /*now*/) {
    mIntendedWakeupTime = targetTime;
    mTimeKeeper->alarmAt(std::bind(&VSyncDispatchTimerQueue::timerCallback, this),
                         mIntendedWakeupTime);
    mLastTimerSchedule = mTimeKeeper->now();
}

void VSyncDispatchTimerQueue::cancelTimer() {
    mIntendedWakeupTime = kInvalidTime;
    mTimeKeeper->alarmCancel();
}
```

`VSyncDispatchTimerQueue`里所有Entry共用同一个`Timer`，所以每次调度都要遍历`mCallbacks`，取最小的`wakeupTime`作为下一次唤醒时间。只有新的最小唤醒时间比`mIntendedWakeupTime`更早时才调用`setTimer()`重排，否则保持原有定时器；如果所有的Entry都不再armed，就`cancelTimer()`。

`VSyncDispatchTimerQueue`构造时的两个常量决定了合并策略：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
constexpr std::chrono::nanoseconds kGroupDispatchWithin = 500us;
constexpr std::chrono::nanoseconds kSnapToSameVsyncWithin = 3ms;

return std::make_unique<VSyncDispatchTimerQueue>(std::make_unique<Timer>(), std::move(tracker),
                                                 kGroupDispatchWithin.count(),
                                                 kSnapToSameVsyncWithin.count());
```

- `mTimerSlack = 500us`：唤醒时间相差500us以内的回调合并成一次定时器唤醒。`timerCallback()`判断条件里的`mTimerSlack`就来自这里。
- `mMinVsyncDistance = 3ms`：两个Vsync时间差小于3ms时视为同一个Vsync，用于`adjustVsyncIfNeeded()`和防跳帧判断。

### 3.5 Timer设置timerfd

`Timer`自己维护一个线程、一个`timerfd`和一个`epoll`实例：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/Timer.cpp
void Timer::alarmAt(std::function<void()> callback, nsecs_t time) {
    std::lock_guard lock(mMutex);
    using namespace std::literals;
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
```

`time`是`CLOCK_MONOTONIC`的绝对纳秒时间，`TFD_TIMER_ABSTIME`表示按绝对时间触发，不是相对当前时间。这样即使`alarmAt()`调用有延迟，也不会逐次累积误差。

## 4 阶段四：Timer线程分发回调

### 4.1 Timer线程醒来

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/src/Timer.cpp
bool Timer::dispatch() {
    setDebugState(DebugState::Running);
    struct sched_param param = {0};
    param.sched_priority = 2;
    ...
    enum DispatchType : uint32_t { TIMER, TERMINATE, MAX_DISPATCH_TYPE };
    ...
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
            ...
        }
    }
}
```

Timer线程名是`TimerDispatch`，默认设置`SCHED_FIFO`优先级2（`disable_sched_fifo_sf_sched()`为true时跳过），用`epoll_wait()`阻塞等待`timerfd`可读。到点后读取`timerfd`清掉计数，取`mCallback`并在锁外执行。这个`mCallback`就是`VSyncDispatchTimerQueue::setTimer()`里`std::bind`出来的`timerCallback`。

### 4.2 VSyncDispatchTimerQueue::timerCallback

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
        if (!mRunning) {
            ALOGD("TimerQueue is not running. Skipping callback.");
            return;
        }
        auto const now = mTimeKeeper->now();
        mLastTimerCallback = now;
        for (auto it = mCallbacks.begin(); it != mCallbacks.end(); it++) {
            auto& callback = it->second;
            auto const wakeupTime = callback->wakeupTime();
            if (!wakeupTime) {
                continue;
            }

            traceEntry(*callback, now);

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
        ftl::Concat trace(ftl::truncated<5>(invocation.callback->name()));
        SFTRACE_FORMAT("%s: %s", __func__, trace.c_str());
        invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                                      invocation.deadlineTimestamp);
    }
}
```

这里的处理顺序很重要：

1. 持锁遍历所有Entry，只挑出`wakeupTime < mIntendedWakeupTime + mTimerSlack + lagAllowance`的Entry。
2. 对选中的Entry调用`executing()`，把`mLastDispatchTime`更新为目标Vsync并`disarm()`，防止同一个Vsync被回调两次。
3. `mIntendedWakeupTime`清空，然后`rearmTimer()`为剩下的Entry重新算定时器。
4. 释放锁后再逐个执行`Entry::callback()`，避免客户端回调里反向调用`VSyncDispatch`造成死锁。

`executing()`的实现：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
nsecs_t VSyncDispatchTimerQueueEntry::executing() {
    mLastDispatchTime = mArmedInfo->mActualVsyncTime;
    disarm();
    return *mLastDispatchTime;
}

void VSyncDispatchTimerQueueEntry::callback(nsecs_t vsyncTimestamp, nsecs_t wakeupTimestamp,
                                            nsecs_t deadlineTimestamp) {
    {
        std::lock_guard<std::mutex> lk(mRunningMutex);
        mRunning = true;
    }

    mCallback(vsyncTimestamp, wakeupTimestamp, deadlineTimestamp);

    std::lock_guard<std::mutex> lk(mRunningMutex);
    mRunning = false;
    mCv.notify_all();
}
```

`Entry::callback()`的第三个参数在`Invocation`里叫`deadlineTimestamp`，它来自`readyTime()`，也就是`nextVsyncTime - readyDuration`。`mRunning`用于`unregisterCallback()`时等待回调真正退出。

## 5 阶段五：两条消费路径

`Timer`里注册了两个回调：`"sf"`和`"app"`。它们共享同一次预测和同一个定时器，但消费者完全不同。

### 5.1 "sf"路径：SurfaceFlinger开始一帧

`MessageQueue`在`initVsyncInternal()`时把`vsyncCallback`注册给`VSyncDispatch`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    const bool reschedule = mVsync.registration &&
            mVsync.registration->cancel() == scheduler::CancelResult::Cancelled;
    auto oldRegistration = std::move(mVsync.registration);
    mVsync.registration = std::make_unique<
            scheduler::VSyncCallbackRegistration>(std::move(dispatch),
                                                  std::bind(&MessageQueue::vsyncCallback, this,
                                                            std::placeholders::_1,
                                                            std::placeholders::_2,
                                                            std::placeholders::_3),
                                                  "sf");
    if (reschedule) {
        mVsync.scheduledFrameTimeOpt =
                mVsync.registration->schedule({.workDuration = mVsync.workDuration.get().count(),
                                               .readyDuration = 0,
                                               .lastVsync = mVsync.lastCallbackTime.ns()});
    }
    return oldRegistration;
}
```

`"sf"`的`readyDuration = 0`，因为SurfaceFlinger自己不需要再给别人预留合成时间。被唤醒后：

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

void MessageQueue::Handler::dispatchFrame(VsyncId vsyncId, TimePoint expectedVsyncTime) {
    if (!mFramePending.exchange(true)) {
        mVsyncId = vsyncId;
        mExpectedVsyncTime = expectedVsyncTime;
        mQueue.mLooper->sendMessage(sp<MessageHandler>::fromExisting(this), Message());
    }
}

void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

`mVsync.value`在0和1之间翻转，这就是Perfetto里`VSYNC-sf`计数器的来源。它本身是一个`TracedOrdinal`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.h
struct Vsync {
    scheduler::TokenManager* tokenManager = nullptr;

    mutable std::mutex mutex;
    std::unique_ptr<scheduler::VSyncCallbackRegistration> registration GUARDED_BY(mutex);
    TracedOrdinal<std::chrono::nanoseconds> workDuration
            GUARDED_BY(mutex) = {"VsyncWorkDuration-sf", std::chrono::nanoseconds(0)};
    TimePoint lastCallbackTime GUARDED_BY(mutex);
    std::optional<scheduler::ScheduleResult> scheduledFrameTimeOpt GUARDED_BY(mutex);
    TracedOrdinal<int> value = {"VSYNC-sf", 0};
};

Vsync mVsync;
```

`generateTokenForPredictions({targetWakeupTime, readyTime, vsyncTime})`把三个时间点登记到FrameTimeline，返回的`vsyncId`会随这一帧一路带到应用的FrameTimeline上报里。

`onFrameSignal()`是`Scheduler`的方法：

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

    // ...（后续遍历follower display并调用compositor.commit()，见Scheduler.cpp）
}
```

`beginFrame()`根据`expectedVsyncTime`给这一帧定出`expectedPresentTime`，`commit()`走后面的`SurfaceFlinger::commit()`和`composite()`。合成完成后，PresentFence回到`SurfaceFlinger::addPresentFence()`，进入阶段六的反馈闭环。

### 5.2 "app"路径：EventThread回调

`EventThread`注册的回调是`createDispatchCallback()`返回的lambda：

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

    LOG_FATAL_IF(!mVSyncState);
    mVsyncTracer = (mVsyncTracer + 1) % 2;
    mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), wakeupTime,
                                       ++mVSyncState->count, vsyncTime, readyTime));
    mCondition.notify_all();
}
```

`mVsyncTracer`的翻转对应Perfetto里的`VSYNC-app`。注意`makeVSync()`的参数顺序：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
DisplayEventReceiver::Event makeVSync(PhysicalDisplayId displayId, nsecs_t timestamp,
                                      uint32_t count, nsecs_t expectedPresentationTime,
                                      nsecs_t deadlineTimestamp) {
    DisplayEventReceiver::Event event;
    event.header = {DisplayEventType::DISPLAY_EVENT_VSYNC, displayId, timestamp};
    event.vsync.count = count;
    event.vsync.vsyncData.preferredFrameTimelineIndex = 0;
    // Temporarily store the current vsync information in frameTimelines[0], marked as
    // platform-preferred. When the event is dispatched later, the frame interval at that time is
    // used with this information to generate multiple frame timeline choices.
    event.vsync.vsyncData.frameTimelines[0] = {.vsyncId = FrameTimelineInfo::INVALID_VSYNC_ID,
                                               .deadlineTimestamp = deadlineTimestamp,
                                               .expectedPresentationTime =
                                                       expectedPresentationTime};
    return event;
}
```

对应关系是：

- `event.header.timestamp = wakeupTime`：应用被唤醒的时间，Choreographer收到的`timestampNanos`就是这个值。
- `expectedPresentationTime = vsyncTime`：这一帧真正应该上屏的时间。
- `deadlineTimestamp = readyTime`：应用把buffer交给SurfaceFlinger的截止时间。

`onVsync()`只是把事件放进`mPendingEvents`并唤醒`EventThread`线程，不做分发。真正的分发在`threadMain()`里：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        DisplayEventReceiver::Event copy = event;
        if (event.header.type == DisplayEventType::DISPLAY_EVENT_VSYNC) {
            const Period frameInterval = mCallback.getVsyncPeriod(consumer->mOwnerUid);
            copy.vsync.vsyncData.frameInterval = frameInterval.ns();
            generateFrameTimeline(copy.vsync.vsyncData, frameInterval.ns(), copy.header.timestamp,
                                  event.vsync.vsyncData.preferredExpectedPresentationTime(),
                                  event.vsync.vsyncData.preferredDeadlineTimestamp());
        }
        switch (consumer->postEvent(copy)) {
            case NO_ERROR:
                break;

            case -EAGAIN:
                // TODO: Try again if pipe is full.
                ALOGW("Failed dispatching %s for %s", toString(event).c_str(),
                      toString(*consumer).c_str());
                break;

            default:
                // Treat EPIPE and other errors as fatal.
                removeDisplayEventConnectionLocked(consumer);
        }
    }
    if (event.header.type == DisplayEventType::DISPLAY_EVENT_VSYNC) {
        mLastCommittedVsyncTime =
                TimePoint::fromNs(event.vsync.vsyncData.preferredExpectedPresentationTime());
        mCallback.onExpectedPresentTimePosted(mLastCommittedVsyncTime);
    }
}
```

`generateFrameTimeline()`会以当前`frameInterval`为步长，往前后各生成多个候选FrameTimeline，客户端可以从中选择提前一帧或多个Vsync上屏的方案：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::generateFrameTimeline(VsyncEventData& outVsyncEventData, nsecs_t frameInterval,
                                        nsecs_t timestamp,
                                        nsecs_t preferredExpectedPresentationTime,
                                        nsecs_t preferredDeadlineTimestamp) const {
    uint32_t currentIndex = 0;
    // Add 1 to ensure the preferredFrameTimelineIndex entry (when multiplier == 0) is included.
    for (int64_t multiplier = -VsyncEventData::kFrameTimelinesCapacity + 1;
         currentIndex < VsyncEventData::kFrameTimelinesCapacity; multiplier++) {
        nsecs_t deadlineTimestamp = preferredDeadlineTimestamp + multiplier * frameInterval;
        // Valid possible frame timelines must have future values, so find a later frame timeline.
        if (deadlineTimestamp <= timestamp) {
            continue;
        }

        nsecs_t expectedPresentationTime =
                preferredExpectedPresentationTime + multiplier * frameInterval;
        ...
        if (multiplier == 0) {
            outVsyncEventData.preferredFrameTimelineIndex = currentIndex;
        }

        outVsyncEventData.frameTimelines[currentIndex] =
                {.vsyncId = generateToken(timestamp, deadlineTimestamp, expectedPresentationTime),
                 .deadlineTimestamp = deadlineTimestamp,
                 .expectedPresentationTime = expectedPresentationTime};
        currentIndex++;
    }
    ...
    outVsyncEventData.frameTimelinesLength = currentIndex;
}
```

`dispatchEvent()`最后更新`mLastCommittedVsyncTime`，下一次`EventThread::threadMain()`调度时它作为`committedVsyncOpt`传回`VSyncDispatchTimerQueueEntry::schedule()`，形成Vsync承诺的闭环。

### 5.3 BitTube写回应用进程

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
status_t EventThreadConnection::postEvent(const DisplayEventReceiver::Event& event) {
    constexpr auto toStatus = [](ssize_t size) {
        return size < 0 ? status_t(size) : status_t(NO_ERROR);
    };

    switch (event.header.type) {
        case DisplayEventType::DISPLAY_EVENT_FRAME_RATE_OVERRIDE:
            [[fallthrough]];
        case DisplayEventType::DISPLAY_EVENT_SUPPORTED_REFRESH_RATE:
            mPendingEvents.emplace_back(event);
            return NO_ERROR;
        default:
            break;
    }

    mPendingEvents.emplace_back(event);
    const auto size = DisplayEventReceiver::sendEvents(&mChannel, mPendingEvents.data(),
                                                       mPendingEvents.size());
    mPendingEvents.clear();
    return toStatus(size);
}
```

`EventThreadConnection::mChannel`是一个`gui::BitTube`，`sendEvents()`最终调用`gui::BitTube::sendObjects()`，把`DisplayEventReceiver::Event`结构体写入socketpair。`BitTube::init()`用的是`socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)`，默认收发缓冲区各4KB，两端都设为非阻塞：

```cpp
// frameworks/native/libs/gui/BitTube.cpp
static const size_t DEFAULT_SOCKET_BUFFER_SIZE = 4 * 1024;

void BitTube::init(size_t rcvbuf, size_t sndbuf) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets) == 0) {
        size_t size = DEFAULT_SOCKET_BUFFER_SIZE;
        setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
        setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
        // since we don't use the "return channel", we keep it small...
        setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        fcntl(sockets[0], F_SETFL, O_NONBLOCK);
        fcntl(sockets[1], F_SETFL, O_NONBLOCK);
        mReceiveFd.reset(sockets[0]);
        mSendFd.reset(sockets[1]);
    } else {
        mReceiveFd.reset();
        ...
    }
}
```

应用进程的`DisplayEventDispatcher`在`Looper`上监听`mReceiveFd`。`EventThreadConnection::stealReceiveChannel()`在创建连接时把接收端fd通过Binder传给应用进程，`mChannel`自己保留发送端fd。

### 5.4 应用进程收到Vsync

```cpp
// frameworks/native/libs/gui/DisplayEventDispatcher.cpp
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    ...
    // Drain all pending events, keep the last vsync.
    nsecs_t vsyncTimestamp;
    PhysicalDisplayId vsyncDisplayId;
    uint32_t vsyncCount;
    VsyncEventData vsyncEventData;
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64
              ", displayId=%s, count=%d, vsyncId=%" PRId64,
              this, ns2ms(vsyncTimestamp), to_string(vsyncDisplayId).c_str(), vsyncCount,
              vsyncEventData.preferredVsyncId());
        mWaitingForVsync = false;
        mLastVsyncCount = vsyncCount;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount, vsyncEventData);
    }

    if (mWaitingForVsync) {
        const nsecs_t currentTime = systemTime(SYSTEM_TIME_MONOTONIC);
        const nsecs_t vsyncScheduleDelay = currentTime - mLastScheduleVsyncTime;
        if (vsyncScheduleDelay > WAITING_FOR_VSYNC_TIMEOUT) {
            ALOGW("Vsync time out! vsyncScheduleDelay=%" PRId64 "ms", ns2ms(vsyncScheduleDelay));
            mWaitingForVsync = false;
            dispatchVsync(currentTime, vsyncDisplayId /* displayId is not used */,
                          ++mLastVsyncCount, vsyncEventData /* empty data */);
        }
    }

    return 1; // keep the callback
}
```

`processPendingEvents()`会一次读空`BitTube`里的所有事件，但只保留最后一个Vsync的时间戳和`VsyncEventData`。这对应“应用积压了多帧时只处理最新一帧”的策略。

`dispatchVsync()`是虚函数，`NativeDisplayEventReceiver`的实现通过JNI回到Java：

```cpp
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, PhysicalDisplayId displayId,
                                               uint32_t count, VsyncEventData vsyncEventData) {
    ...
        env->CallVoidMethod(receiverObj.get(), gDisplayEventReceiverClassInfo.dispatchVsync,
                            timestamp, displayId.value, count);
    ...
}

// frameworks/base/core/java/android/view/DisplayEventReceiver.java
private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame) {
    onVsync(timestampNanos, physicalDisplayId, frame, mVsyncEventData);
}
```

`Choreographer`内部的`FrameDisplayEventReceiver`重写了`onVsync()`：

```java
// frameworks/base/core/java/android/view/Choreographer.java
public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
        VsyncEventData vsyncEventData) {
    try {
        ...
        // Post the vsync event to the Handler.
        // The idea is to prevent incoming vsync events from completely starving
        // the message queue.  If there are no messages in the queue with timestamps
        // earlier than the frame time, then the vsync event will be processed immediately.
        // Otherwise, messages that predate the vsync event will be handled first.
        long now = System.nanoTime();
        if (timestampNanos > now) {
            ...
            timestampNanos = now;
        }

        if (mHavePendingVsync) {
            ...
        } else {
            mHavePendingVsync = true;
        }

        mTimestampNanos = timestampNanos;
        mFrame = frame;
        mLastVsyncEventData.copyFrom(vsyncEventData);
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
}
```

`Choreographer`把Vsync事件封装成一条异步Message，按`timestampNanos`（也就是`wakeupTime`）投递到主线程消息队列。`doFrame()`执行完输入、动画、测量布局、绘制之后，会再次调用`scheduleVsyncLocked()`请求下一次Vsync，回到阶段二。

### 5.5 应用侧的兜底超时

`DisplayEventDispatcher::handleEvent()`里还有一个兜底：发出请求后如果超过`WAITING_FOR_VSYNC_TIMEOUT`还没收到Vsync，就自己构造一个假Vsync时间戳（当前时间）调用`dispatchVsync()`，让`Choreographer`继续走一帧。这防止SurfaceFlinger异常时应用完全卡死。

对应地，SurfaceFlinger侧`EventThread::threadMain()`也有兜底：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
// Generate a fake VSYNC after a long timeout in case the driver stalls. When the
// display is off, keep feeding clients at 60 Hz.
const std::chrono::nanoseconds timeout =
        mState == State::SyntheticVSync ? 16ms : 1000ms;
if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
    if (mState == State::VSync) {
        ALOGW("Faking VSYNC due to driver stall for thread %s", mThreadName);
    }

    LOG_FATAL_IF(!mVSyncState);
    const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
    const auto deadlineTimestamp = now + timeout.count();
    const auto expectedVSyncTime = deadlineTimestamp + timeout.count();
    mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), now,
                                       ++mVSyncState->count, expectedVSyncTime,
                                       deadlineTimestamp));
}
```

- `State::SyntheticVSync`：息屏等场景，每16ms造一个Vsync，让应用继续按60Hz节奏跑。
- `State::VSync`：等真实Vsync超过1000ms，说明驱动可能停滞，打一条`Faking VSYNC due to driver stall`日志并造一个假Vsync兜底。

## 6 阶段六：PresentFence反馈校准

合成上屏后，SurfaceFlinger拿到每个显示器的PresentFence，并交给Scheduler：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
for (auto&& [id, presentFence] : presentFences) {
    mScheduler->addPresentFence(id, std::move(presentFence));
}
```

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

PresentFence进入`VSyncReactor`：

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

PresentFence有两个状态：

- `Fence::SIGNAL_TIME_PENDING`：还没signal，先挂进`mUnfiredFences`，下次`addPresentFence()`时再调`updateTrackerWithSignaledFences()`检查。
- 已经signal：直接把`signalTime`当Vsync时间戳喂给Tracker。

`mUnfiredFences`有上限`kMaxPendingFences = 20`，超了就丢掉最老的一个。

关键点：PresentFence的时间和硬件Vsync的时间在Tracker眼里是同一类东西。`VSyncReactor::addVsyncTimestampLocked()`对两者一视同仁：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::addVsyncTimestampLocked(nsecs_t timestamp,
                                           VSyncTracker::VsyncTimeSource source) {
    const auto accuracy = mTracker.getModelAccuracy(timestamp);
    const bool accepted = mTracker.addVsyncTimestamp(timestamp);
    reportModelAccuracyMetric(accuracy, source, accepted);
    return accepted;
}
```

差别只在`VsyncTimeSource`，Perfetto里能区分`HwVsyncCallback`、`HwVsyncQuery`和`PresentFence`三种来源。

`addPresentFence()`返回的`mMoreSamplesNeeded`和`addHwVsyncTimestamp()`一样决定是否打开硬件Vsync：如果当前模型还需要更多样本，或者PresentFence与模型不一致导致`timestampAccepted == false`，就调用`enableHardwareVsync()`重新打开硬件Vsync；否则继续关掉，只靠PresentFence维持模型。

### 6.1 刷新率切换时的周期确认

刷新率变化时，`VSyncReactor`会进入`mPeriodConfirmationInProgress`状态，并强制打开硬件Vsync来确认新周期：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
void VSyncReactor::startPeriodTransitionInternal(ftl::NonNull<DisplayModePtr> modePtr) {
    SFTRACE_FORMAT("%s %" PRIu64, __func__, mId.value);
    mPeriodConfirmationInProgress = true;
    mModePtrTransitioningTo = modePtr.get();
    mMoreSamplesNeeded = true;
    setIgnorePresentFencesInternal(true);
    ...
}
```

`periodConfirmed()`用`HwcVsyncPeriod`或相邻两个硬件Vsync的间隔，与目标周期比较，误差在10%以内就认为新周期确认：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::periodConfirmed(nsecs_t vsync_timestamp, std::optional<nsecs_t> HwcVsyncPeriod) {
    if (!mPeriodConfirmationInProgress) {
        return false;
    }
    ...
    static constexpr int allowancePercent = 10;
    static constexpr std::ratio<allowancePercent, 100> allowancePercentRatio;
    auto const allowance = period * allowancePercentRatio.num / allowancePercentRatio.den;
    if (HwcVsyncPeriod) {
        return std::abs(*HwcVsyncPeriod - period) < allowance;
    }

    auto const distance = vsync_timestamp - *mLastHwVsync.get();
    return std::abs(distance - period) < allowance;
}
```

确认之后`endPeriodTransition()`清掉状态，Tracker的预测模型切到新周期，硬件Vsync再根据`needsMoreSamples()`决定是否关闭。

## 7 全链路分步清单

把上面的内容按时间顺序压成一份清单，便于对照Perfetto和日志：

| 序号 | 执行线程 | 调用 | 关键动作 |
|---|---|---|---|
| 1 | HWC回调线程 | `SurfaceFlinger::onComposerHalVsync` | 校验时间戳，`getHwComposer().onVsync()`把HWC displayId转成`PhysicalDisplayId` |
| 2 | HWC回调线程 | `Scheduler::addResyncSample` | 校验来源，按displayId找`VsyncSchedule` |
| 3 | HWC回调线程 | `VsyncSchedule::addResyncSample` | 持`mHwVsyncLock`，仅在`HwVsyncState::Enabled`时继续 |
| 4 | HWC回调线程 | `VSyncReactor::addHwVsyncTimestamp` | 确认周期，或把样本加入确认队列 |
| 5 | HWC回调线程 | `VSyncPredictor::addVsyncTimestamp` | 校验样本，写入环形缓冲区，必要时做线性回归 |
| 6 | HWC回调线程 | `reportModelAccuracyMetric` | 把模型误差和来源打到Perfetto |
| 7 | SF主线程 | `Scheduler::onHardwareVsyncRequest` | 需要更多样本时打开硬件Vsync；样本够了就关闭 |
| 8 | SF主线程 | `SurfaceFlinger::requestHardwareVsync` | `setVsyncEnabled(ENABLE/DISABLE)`，打开时顺带查询`getDisplayKnownVsyncSample` |
| 9 | 应用主线程 | `Choreographer.scheduleVsyncLocked` | 调用`DisplayEventReceiver.scheduleVsync()` |
| 10 | 应用主线程 | `DisplayEventDispatcher::scheduleVsync` | `mWaitingForVsync`去重，`requestNextVsync()`发Binder |
| 11 | SF Binder线程 | `EventThread::requestNextVsync` | `vsyncRequest = Single`，`mCondition.notify_all()` |
| 12 | SF EventThread线程 | `EventThread::threadMain` | 发现`vsyncRequested`，状态切到`State::VSync` |
| 13 | SF EventThread线程 | `VSyncCallbackRegistration::schedule` | 把`workDuration/readyDuration/lastVsync/committedVsync`传给Dispatch |
| 14 | SF EventThread线程 | `VSyncDispatchTimerQueue::scheduleLocked` | 找到Entry；`rearmImminent`时先存`mWorkloadUpdateInfo` |
| 15 | SF EventThread线程 | `VSyncDispatchTimerQueueEntry::schedule` | 用Tracker预测`nextVsyncTime`，算出`nextReadyTime`和`nextWakeupTime` |
| 16 | SF EventThread线程 | `VSyncPredictor::nextAnticipatedVSyncTimeFrom` | 按模型算出下一个Vsync |
| 17 | SF EventThread线程 | `rearmTimerSkippingUpdateFor` | 取所有Entry里最小的`wakeupTime` |
| 18 | SF EventThread线程 | `Timer::alarmAt` | `timerfd_settime(TFD_TIMER_ABSTIME)` |
| 19 | Timer线程 | `Timer::dispatch` | `epoll_wait`返回，读出`mCallback`并执行 |
| 20 | Timer线程 | `VSyncDispatchTimerQueue::timerCallback` | 挑出到期的Entry，`executing()`后重新arm剩下的Entry |
| 21 | Timer线程 | `VSyncDispatchTimerQueueEntry::callback` | 设置`mRunning = true`，调用`mCallback` |
| 22 | Timer线程 | `MessageQueue::vsyncCallback` | 翻转`VSYNC-sf`，生成`vsyncId`，投递到SF主线程 |
| 23 | SF主线程 | `Scheduler::onFrameSignal` | `FrameTargeter::beginFrame()`，`commit()`，`composite()` |
| 24 | Timer线程 | `EventThread::onVsync` | 翻转`VSYNC-app`，`makeVSync()`入队，唤醒EventThread线程 |
| 25 | SF EventThread线程 | `EventThread::threadMain` | 取`mPendingEvents`，`shouldConsumeEvent()`筛选连接 |
| 26 | SF EventThread线程 | `EventThread::dispatchEvent` | 按Uid取`frameInterval`，`generateFrameTimeline()` |
| 27 | SF EventThread线程 | `EventThreadConnection::postEvent` | 写入`BitTube` |
| 28 | 应用主线程 | `DisplayEventDispatcher::handleEvent` | 读空BitTube，只保留最后一个Vsync |
| 29 | 应用主线程 | `NativeDisplayEventReceiver::dispatchVsync` | JNI回调`DisplayEventReceiver.dispatchVsync` |
| 30 | 应用主线程 | `Choreographer.FrameDisplayEventReceiver.onVsync` | 按`wakeupTime`投递异步Message |
| 31 | 应用主线程 | `Choreographer.doFrame` | 执行输入、动画、遍历、绘制，末尾再次`scheduleVsyncLocked()` |
| 32 | SF主线程 | `SurfaceFlinger::addPresentFence` | PresentFence交给`Scheduler::addPresentFence` |
| 33 | SF主线程 | `Scheduler::addPresentFence` | 转给`VsyncController::addPresentFence` |
| 34 | SF主线程 | `VSyncReactor::addPresentFence` | signal的直接喂Tracker，pending的入`mUnfiredFences` |
| 35 | SF主线程 | `VSyncPredictor::addVsyncTimestamp` | 用PresentFence时间校准模型，闭环回到第5步 |

## 8 关键时间点关系

一次调度里出现的三个时间点到上屏的关系：

```text
nextWakeupTime          nextReadyTime              nextVsyncTime
     │                       │                          │
     │  workDuration         │  readyDuration           │
     ├──────────────────────►├─────────────────────────►│
     │                       │                          │
   客户端被唤醒            buffer交给SF              这一帧上屏
```

- `nextVsyncTime`：Tracker预测的下一次硬件Vsync，也是`DisplayEventReceiver.Event::vsyncData.expectedPresentationTime`。
- `nextReadyTime = nextVsyncTime - readyDuration`：应用必须把buffer交给SurfaceFlinger的截止时间，对应`Event::vsyncData.frameTimelines[].deadlineTimestamp`。
- `nextWakeupTime = nextReadyTime - workDuration`：应用线程被唤醒的时间，对应`Event::header.timestamp`，也是`Choreographer.onVsync()`收到的`timestampNanos`。
- 对`"sf"`回调，`readyDuration = 0`，所以`nextWakeupTime = nextVsyncTime - workDuration`，`nextReadyTime = nextVsyncTime`。
- 对`"app"`回调，`readyDuration = configs.late.sfWorkDuration`，把SurfaceFlinger合成需要的时间也算进应用的工作预算里。

`VSyncDispatchTimerQueue`一次唤醒可能同时触发`sf`和`app`两个Entry，因为两者的`wakeupTime`可能落在`mTimerSlack = 500us`以内。`timerCallback()`在锁内只收集需要执行的Entry，锁外再串行执行它们的回调，所以Perfetto里看到的是同一条Timer线程依次进入`VSyncDispatchTimerQueue::timerCallback`、`MessageQueue::vsyncCallback`、`EventThread::onVsync`。

## 9 三个容易混淆的点

### 9.1 硬件Vsync不是一直开着的

硬件Vsync只在`VSyncTracker`需要样本时打开：模型还没攒够样本、刷新率切换后确认新周期、PresentFence与模型不一致需要重新学习。模型稳定后硬件Vsync关闭，Tracker靠PresentFence继续校准。因此Perfetto里`VSYNC-app`/`VSYNC-sf`是软件定时器产生的，和HWC的硬件Vsync回调不是同一条trace。

### 9.2 `EventThread::onVsync`里的三个时间参数

```cpp
void EventThread::onVsync(nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime)
```

- `vsyncTime`来自`VSyncDispatchTimerQueueEntry::executing()`里的`mLastDispatchTime`，是目标Vsync。
- `wakeupTime`来自Entry的`mActualWakeupTime`，是定时器实际唤醒时间。
- `readyTime`来自Entry的`mActualReadyTime`，是应用交付buffer的截止时间。

`makeVSync()`把`wakeupTime`放进`event.header.timestamp`，把`vsyncTime`放进`expectedPresentationTime`，把`readyTime`放进`deadlineTimestamp`。这三者不要弄反。

### 9.3 `VSyncRequest`的状态转换

```cpp
enum class VSyncRequest {
    None = -2,
    // Single wakes up for the next two frames to avoid scheduler overhead
    Single = -1,
    // SingleSuppressCallback only wakes up for the next frame
    SingleSuppressCallback = 0,
    Periodic = 1,
    ...
};
```

状态转换实际发生在`shouldConsumeEvent()`里：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
case DisplayEventType::DISPLAY_EVENT_VSYNC:
    switch (connection->vsyncRequest) {
        case VSyncRequest::None:
            return false;
        case VSyncRequest::SingleSuppressCallback:
            connection->vsyncRequest = VSyncRequest::None;
            return false;
        case VSyncRequest::Single: {
            if (throttleVsync()) {
                return false;
            }
            connection->vsyncRequest = VSyncRequest::SingleSuppressCallback;
            return true;
        }
        case VSyncRequest::Periodic:
            if (throttleVsync()) {
                return false;
            }
            return true;
        default:
            ...
    }
```

- `None` → `Single`：第一次请求，`notify_all()`唤醒EventThread线程。
- `Single` → `SingleSuppressCallback`：`shouldConsumeEvent()`确认该连接要消费这次Vsync后，把请求降级为“已投递未确认”。
- `SingleSuppressCallback` → `None`：如果应用在下一次Vsync到来前没有再请求，就回到`None`，这次回调被抑制。
- `SingleSuppressCallback` → `Single`：应用在确认前又请求一次，EventThread只改状态，不再唤醒线程。
- `Periodic{N}`：通过`setVsyncRate(N)`设置，每N个Vsync回调一次。

`throttleVsync()`会按连接的`frameRate`或`mCallback.throttleVsync()`做帧率节流，所以一个`"app"`回调不一定分发给所有连接。
