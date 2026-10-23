## 1 硬件Vsycn信号发出

SurfaceFlinger在初始化时把自己设为HWC的回调

```cpp
void SurfaceFlinger::init() FTL_FAKE_GUARD(kMainThreadContext){
    ...
    // 建HWComposer(HWC HAL的封装),并把SF自己设为回调
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(mHwcServiceName));
    auto& composer = mCompositionEngine->getHwComposer();
    composer.setCallback(*this);
    ...
}
```

HWC把硬件Vsync给SurfaceFlinger，入口是`SurfaceFlinger::onComposerHalVsync`：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onComposerHalVsync(hal::HWDisplayId hwcDisplayId, int64_t timestamp,
                                        std::optional<hal::VsyncPeriodNanos> vsyncPeriod) {
    // 在Perftto中看到binder调用 onComposerHalVsync 0 833333333ns
    SFTRACE_NAME(vsyncPeriod
                         ? ftl::Concat(__func__, ' ', hwcDisplayId, ' ', *vsyncPeriod, "ns").c_str()
                         : ftl::Concat(__func__, ' ', hwcDisplayId).c_str());

    if (timestamp <= 0) {
        ALOGW("%s: Ignoring invalid timestamp %" PRId64 " from HWC display %" PRIu64, __func__,
              timestamp, hwcDisplayId);
        // 这个在Perfetto中观察到是一个小箭头
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

## 2 Scheduler匹配Display，发送硬件Vsync信号

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
bool Scheduler::addResyncSample(PhysicalDisplayId id, nsecs_t timestamp,
                                std::optional<nsecs_t> hwcVsyncPeriodIn,
                                VSyncTracker::VsyncTimeSource source) {

    // 只接受硬件Vsync来源
    if (source == VSyncTracker::VsyncTimeSource::PresentFence ||
        source == VSyncTracker::VsyncTimeSource::Unknown) {
        LOG_ALWAYS_FATAL("Invalid VsyncTimeSource for addResyncSample: %s",
                         ftl::enum_string(source).c_str());
        return false;
    }
    const auto hwcVsyncPeriod = ftl::Optional(hwcVsyncPeriodIn).transform([](nsecs_t nanos) {
        return Period::fromNs(nanos);
    });
    // 获取Display对应的VsyncSchedule
    auto schedule = getVsyncSchedule(id);
    if (!schedule) {
        ALOGW("%s: Invalid display %s!", __func__, to_string(id).c_str());
        return false;
    }
    return schedule->addResyncSample(TimePoint::fromNs(timestamp), hwcVsyncPeriod, source);
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
bool VsyncSchedule::addResyncSample(TimePoint timestamp, ftl::Optional<Period> hwcVsyncPeriod,
                                    VSyncTracker::VsyncTimeSource source) {
    bool needsHwVsync = false;
    bool periodFlushed = false;
    {
        std::lock_guard<std::mutex> lock(mHwVsyncLock);
        // 只有硬件VSync启用时才喂样本
        if (mHwVsyncState == HwVsyncState::Enabled) {
            needsHwVsync = mController->addHwVsyncTimestamp(timestamp.ns(),
                                                            hwcVsyncPeriod.transform(&Period::ns),
                                                            &periodFlushed, source);
        }
    }
    // 控制器还需要更多样本进行软件模型校准
    if (needsHwVsync) {
        enableHardwareVsync();
    // 关闭硬件Vsync省电
    } else {
        constexpr bool kDisallow = false;
        disableHardwareVsync(kDisallow);
    }
    return periodFlushed;
}
```

## 3 VsyncController中正式喂Vsync样本

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncReactor.cpp
bool VSyncReactor::addHwVsyncTimestamp(nsecs_t timestamp, std::optional<nsecs_t> hwcVsyncPeriod,
                                       bool* periodFlushed, VSyncTracker::VsyncTimeSource source) {
    assert(periodFlushed);
    std::lock_guard lock(mMutex);
    // tracker已经通过足够的硬件Vsync确认了新的Vsync周期
    if (periodConfirmed(timestamp, hwcVsyncPeriod)) {
        SFTRACE_FORMAT("VSR %" PRIu64 ": period confirmed", mId.value);
        // 走了Scheduler::resyncToHardwareVsyncLocked  Scheduler::promotePacesetterDisplay流程
        // 这两个函数都是在SurfaceFlinger主线程中调用，一般resync和display切换时被调
        if (mModePtrTransitioningTo) {
            // 把tracker内部的指针切换到mModePtrTransitioningTo的模式
            mTracker.setDisplayModePtr(ftl::as_non_null(mModePtrTransitioningTo));
            // 标记新周期正式生效
            *periodFlushed = true;
        }
        // 喂mPeriodConfirmationInProgress过程中mLastHwVsync.set(timestamp)的样本
        if (mLastHwVsync.get() && !mLastHwVsync.isFirst()) {
            addVsyncTimestampLocked(*mLastHwVsync.get(), source);
        }
        // 喂一个硬件Vsync样本
        addVsyncTimestampLocked(timestamp, source);
        endPeriodTransition();
        mMoreSamplesNeeded = mTracker.needsMoreSamples();
    // 正在确认新周期
    } else if (mPeriodConfirmationInProgress) {
        SFTRACE_FORMAT("VSR %" PRIu64 ": still confirming period", mId.value);
        mLastHwVsync.set(timestamp);
        // 如果是确认新周期间的第一个样本，同样送入tracker
        if (mLastHwVsync.isFirst()) {
            addVsyncTimestampLocked(*mLastHwVsync.get(), source);
        }
        // 一定需要更多的样本完成周期确认
        mMoreSamplesNeeded = true;
        *periodFlushed = false;
    // 正常喂样本
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


bool VSyncReactor::periodConfirmed(nsecs_t vsync_timestamp, std::optional<nsecs_t> HwcVsyncPeriod) {
    // 走了Scheduler::resyncToHardwareVsyncLocked  Scheduler::promotePacesetterDisplay
    // 在onDisplayModeChanged设置过mPeriodConfirmationInProgress为true
    if (!mPeriodConfirmationInProgress) {
        return false;
    }
    // 特殊低功耗情况 直接放行
    if (mDisplayPowerMode == hal::PowerMode::DOZE ||
        mDisplayPowerMode == hal::PowerMode::DOZE_SUSPEND) {
        return true;
    }
    // 没有上一次的样本，这次硬件Vsync也没有传周期，无法判断
    if (!mLastHwVsync.get() && !HwcVsyncPeriod) {
        return false;
    }
    // 从mModePtrTransitioningTo拿到周期
    const std::optional<Period> newPeriod = mModePtrTransitioningTo
            ? mModePtrTransitioningTo->getVsyncRate().getPeriod()
            : std::optional<Period>{};
    // 确实在切换刷新率
    const bool periodIsChanging = newPeriod && (newPeriod->ns() != mTracker.currentPeriod());
    // TODO KernelIdleTimer机制
    if (mSupportKernelIdleTimer && !periodIsChanging) {
        // Clear out the Composer-provided period and use the allowance logic below
        HwcVsyncPeriod = {};
    }
    // 拿新周期，没有就用tracker中的
    auto const period = newPeriod ? newPeriod->ns() : mTracker.currentPeriod();
    static constexpr int allowancePercent = 10;
    static constexpr std::ratio<allowancePercent, 100> allowancePercentRatio;
    auto const allowance = period * allowancePercentRatio.num / allowancePercentRatio.den;
    // 如果HWC提供了周期
    if (HwcVsyncPeriod) {
        return std::abs(*HwcVsyncPeriod - period) < allowance;
    }
    //HWC没有提供周期 就用本次时间戳与上次样本时间戳的差值来推断
    auto const distance = vsync_timestamp - *mLastHwVsync.get();
    return std::abs(distance - period) < allowance;
}


bool VSyncReactor::addVsyncTimestampLocked(nsecs_t timestamp,
                                           VSyncTracker::VsyncTimeSource source) {
    const auto accuracy = mTracker.getModelAccuracy(timestamp);
    const bool accepted = mTracker.addVsyncTimestamp(timestamp);
    reportModelAccuracyMetric(accuracy, source, accepted);
    return accepted;
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncPredictor.cpp
bool VSyncPredictor::addVsyncTimestamp(nsecs_t timestamp) {
    SFTRACE_CALL();
    std::lock_guard lock(mMutex);
    // 检查出时间戳不合理  但要把mKnownTimestamp更新到最新
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
    // timestamp写入环形缓冲区
    if (mTimestamps.size() != kHistorySize) {
        mTimestamps.push_back(timestamp);
        mLastTimestampIndex = next(mLastTimestampIndex);
    } else {
        mLastTimestampIndex = next(mLastTimestampIndex);
        mTimestamps[mLastTimestampIndex] = timestamp;
    }
    traceInt64If("VSP-ts", timestamp);
    // numSamples参与本次回归的有效样本数  oldestTs最旧的有效时间戳
    const auto [numSamples, oldestTs] = getSampleSizeAndOldestVsync(timestamp);
    traceInt64("VSP-numSamples", static_cast<int64_t>(numSamples));
    mOldestVsync = oldestTs;
    const auto minNumSamples = getMinSamplesRequiredForPrediction();
    if (numSamples < minNumSamples || numSamples == 1) {
        // 单样本模式 不做回归
        mRateMap[idealPeriod()] = {idealPeriod(), 0};
        return true;
    }
    std::vector<nsecs_t> vsyncTS(numSamples);
    std::vector<nsecs_t> ordinals(numSamples);
    // std::unordered_map<nsecs_t, Model> mutable mRateMap GUARDED_BY(mMutex)
    // Model就只包含计算所得周期anticipatedPeriod和相对mOldestVsync偏移intercept
    auto it = mRateMap.find(idealPeriod());
    constexpr int64_t kScalingFactor = 1000;
    nsecs_t meanTS = 0;
    nsecs_t meanOrdinal = 0;
    size_t vsyncIndex = 0;
    for (size_t i = 0; i < mTimestamps.size(); i++) {
        if (!isVsyncWithinThreshold(timestamp, mTimestamps[i])) continue;
        const auto ts = mTimestamps[i] - oldestTs;
        vsyncTS[vsyncIndex] = ts;
        meanTS += ts;
        const auto ordinal = idealPeriod() == 0 ? 0 
             : (vsyncTS[vsyncIndex] + idealPeriod() / 2) / idealPeriod() * kScalingFactor;
        ordinals[vsyncIndex] = ordinal;
        meanOrdinal += ordinal;
        ++vsyncIndex;
    }
    LOG_FATAL_IF(vsyncIndex != numSamples,
                 "samples size does not match with the number of vsyncs in window for prediction");
    meanTS /= numSamples;
    meanOrdinal /= numSamples;
    for (size_t i = 0; i < numSamples; i++) {
        vsyncTS[i] -= meanTS;
        ordinals[i] -= meanOrdinal;
    }
    nsecs_t top = 0;
    nsecs_t bottom = 0;
    for (size_t i = 0; i < numSamples; i++) {
        top += vsyncTS[i] * ordinals[i];
        bottom += ordinals[i] * ordinals[i];
    }
    // 斜率分母为0  所有Vsync的ordinary相同  退化到理想周期模型
    if (CC_UNLIKELY(bottom == 0)) {
        it->second = {idealPeriod(), 0};
        clearTimestamps(/* clearTimelines */ true);
        return false;
    }
    nsecs_t const anticipatedPeriod = top * kScalingFactor / bottom;
    nsecs_t const intercept = meanTS - (anticipatedPeriod * meanOrdinal / kScalingFactor);
    // 周期偏离理想值太多 退化到理想周期模型
    auto const percent = std::abs(anticipatedPeriod - idealPeriod()) * kMaxPercent / idealPeriod();
    if (percent >= kOutlierTolerancePercent) {
        it->second = {idealPeriod(), 0};
        clearTimestamps(/* clearTimelines */ true);
        return false;
    }
    // 计算所得的周期和相对mOldestVsync偏移
    traceInt64("VSP-period", anticipatedPeriod);
    traceInt64("VSP-intercept", intercept);
    it->second = {anticipatedPeriod, intercept};
    ALOGV("model update ts %" PRIu64 ": %" PRId64 " slope: %" PRId64 " intercept: %" PRId64,
          mId.value, timestamp, anticipatedPeriod, intercept);
    return true;
}