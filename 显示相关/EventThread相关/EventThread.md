## 1 来历

在Scheduler初始化的过程(initScheduler)中会创建EventThread(android-17.0.0_r1中只有app一个，android-16.0.0_r1中还有appSf，这个信号可在Perftto中观测到)

```cpp

// android-17.0.0_r1
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
mScheduler->createEventThread(mFrameTimeline->getTokenManager(),
                                /* workDuration */ configs.late.appWorkDuration,
                                /* readyDuration */ configs.late.sfWorkDuration);

// android-16.0.0_r1
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
mScheduler->createEventThread(scheduler::Cycle::Render, mFrameTimeline->getTokenManager(),
                                /* workDuration */ configs.late.appWorkDuration,
                                /* readyDuration */ configs.late.sfWorkDuration);
mScheduler->createEventThread(scheduler::Cycle::LastComposite,
                                mFrameTimeline->getTokenManager(),
                                /* workDuration */ activeRefreshRate.getPeriod(),
                                /* readyDuration */ configs.late.sfWorkDuration);
```

实际EventThread的创建情况以项目为准，联想A17的项目中还保留A14返回ConnectionHandle的形式，后面内容以android-17.0.0_r1为参考

```cpp
// 注意的是这里的getVsyncSchedule()不带displayid返回的是PacesetterDisplay的VsyncSchedule
// android-17.0.0_r1中就只有app一个了
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
<a id="createEventThread"></a>
void Scheduler::createEventThread(TokenManager* tokenManager, std::chrono::nanoseconds workDuration,
                                  std::chrono::nanoseconds readyDuration) {
    mEventThread =
            std::make_unique<android::impl::EventThread>("app", getVsyncSchedule(), tokenManager,
                                                         *this, workDuration, readyDuration);
}

// android-16.0.0_r1中还会根据cycle类型创建两个
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::createEventThread(Cycle cycle, frametimeline::TokenManager* tokenManager,
                                  std::chrono::nanoseconds workDuration,
                                  std::chrono::nanoseconds readyDuration) {
    auto eventThread =
            std::make_unique<android::impl::EventThread>(cycle == Cycle::Render ? "app" : "appSf",
                                                         getVsyncSchedule(), tokenManager, *this,
                                                         workDuration, readyDuration);
    if (cycle == Cycle::Render) {
        mRenderEventThread = std::move(eventThread);
    } else {
        mLastCompositeEventThread = std::move(eventThread);
    }
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
EventThread::EventThread(const char* name, std::shared_ptr<scheduler::VsyncSchedule> vsyncSchedule,
                         android::scheduler::TokenManager* tokenManager,
                         IEventThreadCallback& callback, std::chrono::nanoseconds workDuration,
                         std::chrono::nanoseconds readyDuration)
      : mThreadName(name),
        mEventThreadStateName(base::StringPrintf("EventThreadState-%s", name)),
        // 这里是Perftto中的观测到的VSYNC-app信号
        mVsyncTracer(base::StringPrintf("VSYNC-%s", name), 0),
        mWorkDuration(base::StringPrintf("VsyncWorkDuration-%s", name), workDuration),
        mReadyDuration(readyDuration),
        mVsyncSchedule(std::move(vsyncSchedule)),
        // 这里createDispatchCallback()创建的callback最终是落到VSyncDispatchTimerQueueEntry的callback
        mVsyncRegistration(mVsyncSchedule->getDispatch(), createDispatchCallback(), name),
        mTokenManager(tokenManager),
        mCallback(callback) {
    // 启动线程 EventThread主要运行线程的事件循环中
    mThread = std::thread([this]() NO_THREAD_SAFETY_ANALYSIS {
        std::unique_lock<std::mutex> lock(mMutex);
        threadMain(lock);
    });
    pthread_setname_np(mThread.native_handle(), mThreadName);
    pid_t tid = pthread_gettid_np(mThread.native_handle());
    // Use SCHED_FIFO to minimize jitter
    constexpr int EVENT_THREAD_PRIORITY = 2;
    struct sched_param param = {0};
    param.sched_priority = EVENT_THREAD_PRIORITY;
    if (pthread_setschedparam(mThread.native_handle(), SCHED_FIFO, &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO for EventThread");
    }
    set_sched_policy(tid, SP_FOREGROUND);
}
```

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
    // Perftto中Vsync-app信号翻转
    mVsyncTracer = (mVsyncTracer + 1) % 2;
    // mPendingEvents中push了Vsync事件
    mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), wakeupTime,
                                       ++mVSyncState->count, vsyncTime, readyTime));
    mCondition.notify_all();
}

DisplayEventReceiver::Event makeVSync(PhysicalDisplayId displayId, nsecs_t timestamp,
                                      uint32_t count, nsecs_t expectedPresentationTime,
                                      nsecs_t deadlineTimestamp) {
    DisplayEventReceiver::Event event;
    // 事件类型DISPLAY_EVENT_VSYNC
    event.header = {DisplayEventType::DISPLAY_EVENT_VSYNC, displayId, timestamp};
    // Vsync序号
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

## 2 工作流程

EvenThread的工作方式仍然是事件循环

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;
    // mState 是状态机，可能取值：Idle、VSync、SyntheticVSync、Quit
    while (mState != State::Quit) {
        // 存储要处理的事件
        std::optional<DisplayEventReceiver::Event> event;
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
            // 处理热插拔事件（屏幕连接/断开）
            if (event->header.type == DisplayEventType::DISPLAY_EVENT_HOTPLUG) {
                if (event->hotplug.connectionError == 0) {
                    if (event->hotplug.connected && !mVSyncState) {
                        // 按默认值创建一个新的 count=0 synthetic=false omitted=false
                        mVSyncState.emplace();
                    } else if (!event->hotplug.connected &&
                               mVsyncSchedule->getPhysicalDisplayId() == event->header.displayId) {
                        // 显示断开时置空
                        mVSyncState.reset();
                    }
                } else {
                    // Ignore vsync stuff on an error.
                }
            }
        }
        bool vsyncRequested = false;
        // 找出该消费此事件的连接
        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) {
                if (event && shouldConsumeEvent(*event, connection)) {
                    consumers.push_back(connection);
                }
                // 这里的逻辑是所有的DisplayEventConnection有没有一个真的请求Vsync
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;
                ++it;
            } else {
                // 删除无效的连接
                it = mDisplayEventConnections.erase(it);
            }
        }
        // 分发事件给所有shouldConsumeEvent的DisplayEventConnection
        if (!consumers.empty()) {
            dispatchEvent(*event, consumers);
            consumers.clear();
        }
        // Vsync可用，且有连接请求Vsync
        if (mVSyncState && vsyncRequested) {
            const bool vsyncOmitted = FlagManager::getInstance().no_vsyncs_on_screen_off() && mVSyncState->omitted;
            // Vsync信号该被省略
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
        // 根据当前状态，要么向调度器注册下一次VSync回调，要么取消注册
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
        // 如果在上面处理过程中mPendingEvents又有了新事件 直接去处理
        if (!mPendingEvents.empty()) {
            continue;
        }
        // 没有连接请求，等待事件注册或请求，或者前面判断Vsync被省略（比如息屏时省略）
        if (mState == State::Idle) {
            mCondition.wait(lock);
        } else {
            // 能走到这代表还是要等待Vsync事件(mState == State::VSync)，或者自己生成fake Vsync事件(mState == State::SyntheticVSync)
            // 处于VSync状态，等待最多1000ms内发送Vsync信号，超时后生成fake Vsync兜底
            // 处于SyntheticVSync状态，等待最多16ms，超时后生成fake Vsync
            const std::chrono::nanoseconds timeout = mState == State::SyntheticVSync ? 16ms : 1000ms;
            if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
                 // 正常Vsync模式下长时间没有收到事件，可能是驱动停滞
                if (mState == State::VSync) {
                    ALOGW("Faking VSYNC due to driver stall for thread %s", mThreadName);
                }
                // 构造并加入一个 fake VSYNC 事件，供后续主循环处理。
                LOG_FATAL_IF(!mVSyncState);
                const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
                const auto deadlineTimestamp = now + timeout.count();
                const auto expectedVSyncTime = deadlineTimestamp + timeout.count();
                mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), now,
                                                   ++mVSyncState->count, expectedVSyncTime,
                                                   deadlineTimestamp));
            }
        }
    }
    // 退出之前取消所有Vsync事件
    mVsyncRegistration.cancel();
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        // 每个消费者的VSync数据比如frame timeline可能不同，不修改原始数据
        DisplayEventReceiver::Event copy = event;
        // Vsync事件
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
    // Vsync事件记录预期呈现事件
    if (event.header.type == DisplayEventType::DISPLAY_EVENT_VSYNC) {
        mLastCommittedVsyncTime = TimePoint::fromNs(event.vsync.vsyncData.preferredExpectedPresentationTime());
        mCallback.onExpectedPresentTimePosted(mLastCommittedVsyncTime);
    }
}
```

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
status_t EventThreadConnection::postEvent(const DisplayEventReceiver::Event& event) {
    constexpr auto toStatus = [](ssize_t size) {
        return size < 0 ? status_t(size) : status_t(NO_ERROR);
    };
    // 帧率重写和帧率重写刷新事件重新放回mPendingEvents中下一次处理
    // 注意这里的mPendingEvents是EventThreadConnection的 不是EventThread的
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

然后这里就进入通过BitTUbe发消息的流程

```cpp
// frameworks/native/libs/gui/DisplayEventReceiver.cpp
ssize_t DisplayEventReceiver::sendEvents(Event const* events, size_t count) {
    return DisplayEventReceiver::sendEvents(mDataChannel.get(), events, count);
}

ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

## 3 APP端的请求Vsync事件
mChoreographer内的scheduleFrameLocked请求的Vsync最终落到
DisplayEventReceiver(CPP侧)的requestNextVsync()

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
// EventThreadConnection继承自BnDisplayEventConnection作为DisplayEventConnection的服务端
binder::Status EventThreadConnection::requestNextVsync() {
    SFTRACE_CALL();
    mEventThread->requestNextVsync(sp<EventThreadConnection>::fromExisting(this));
    return binder::Status::ok();
}

// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::requestNextVsync(const sp<EventThreadConnection>& connection) {
    mCallback.resync(IEventThreadCallback::ResyncCaller::RequestNextVsync);
    std::lock_guard<std::mutex> lock(mMutex);
    // 当前connection没有请求Vsync，后续EventThread发Vsync回调
    if (connection->vsyncRequest == VSyncRequest::None) {
        // 标记单次请求
        connection->vsyncRequest = VSyncRequest::Single;
        // 唤醒mCondition.wait()的EventThread线程
        mCondition.notify_all();
    // connection->vsyncRequest为VSyncRequest::Single时发出了一次Vsync后变为VSyncRequest::SingleSuppressCallback
    // VSyncRequest::SingleSuppressCallback表示Vsync回调已投递但未被确认
    // 如果这时又有一次请求，和前面遇到VSyncRequest::None一样标记单次请求，但EventThread不需要唤醒
    // 这里只尚未确定应用消费Vsync信号，此时抑制再发    
    } else if (connection->vsyncRequest == VSyncRequest::SingleSuppressCallback) {
        connection->vsyncRequest = VSyncRequest::Single;
    }
}
```