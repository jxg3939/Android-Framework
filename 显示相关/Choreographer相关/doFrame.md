
## 1.Vsync事件来到APP

```cpp
承接EventThread发送Vsync事件
// frameworks/native/libs/gui/DisplayEventReceiver.cpp
ssize_t DisplayEventReceiver::sendEvents(Event const* events, size_t count) {
    return DisplayEventReceiver::sendEvents(mDataChannel.get(), events, count);
}

ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}

// frameworks/native/libs/gui/BitTube.cpp
ssize_t BitTube::sendObjects(BitTube* tube, void const* events, size_t count, size_t objSize) {
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count * objSize);

    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
                        "BitTube::sendObjects(count=%zu, size=%zu), res=%zd (partial events were "
                        "sent!)",
                        count, objSize, size);

    // ALOGE_IF(size<0, "error %d sending %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

Looper监听fd
经过`Looper::pollOnce`后`Looper::pollInner`中走到`response.request.callback->handleEvent(fd, events, data)`

## 2. DisplayEventDispatcher一路调回到Choreographer

```cpp
// frameworks/native/libs/gui/DisplayEventDispatcher.cpp
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    ...
    // Drain all pending events, keep the last vsync.
    nsecs_t vsyncTimestamp;
    PhysicalDisplayId vsyncDisplayId;
    uint32_t vsyncCount;
    VsyncEventData vsyncEventData;
    // 拿到有效Vsync事件
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64
              ", displayId=%s, count=%d, vsyncId=%" PRId64,
              this, ns2ms(vsyncTimestamp), to_string(vsyncDisplayId).c_str(), vsyncCount,
              vsyncEventData.preferredVsyncId());
        mWaitingForVsync = false;
        mLastVsyncCount = vsyncCount;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount, vsyncEventData);
    }

    // 超时保护
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

`DisplayEventDispatcher`在APP里的实现是`NativeDisplayEventReceiver`

```cpp
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, PhysicalDisplayId displayId,
                                               uint32_t count, VsyncEventData vsyncEventData) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    ...
    ScopedLocalRef<jobject> receiverObj(env, GetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        ...
        // 回调回java
        env->CallVoidMethod(receiverObj.get(), gDisplayEventClassInfo.dispatchVsync,
                            timestamp, displayId.value, count);
        ALOGV("receiver %p ~ Returned from vsync handler.", this);
    }

    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```

```cpp
// frameworks/base/core/java/android/view/DisplayEventReceiver.java
// Called from native code.
@SuppressWarnings("unused")
private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame) {
    onVsync(timestampNanos, physicalDisplayId, frame, mVsyncEventData);
}

public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
        VsyncEventData vsyncEventData) {
}
```

`DisplayEventReceiver`在APP里的实现是`FrameDisplayEventReceiver`

```cpp
// frameworks/base/core/java/android/view/Choreographer.java
public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
        VsyncEventData vsyncEventData) {
    try {
        if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW,
                    "Choreographer#onVsync "
                            + vsyncEventData.preferredFrameTimeline().vsyncId);
        }
        long now = System.nanoTime();
        if (timestampNanos > now) {
            if (DEBUG_JANK) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
            }
            timestampNanos = now;
        }
        if (mHavePendingVsync) {
            if (DEBUG_JANK) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            }
        } else {
            mHavePendingVsync = true;
        }
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        mLastVsyncEventData.copyFrom(vsyncEventData);
        // mDisplayEventReceiver自己作为消息回调
        Message msg = Message.obtain(mHandler, this);
        // 异步消息突破postTraversalBarrier()
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

## 3.进入doFrame()

`mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS)`之后
由Looper和Handler去调度
最终走到`message.callback.run();`

```cpp
// frameworks/base/core/java/android/view/Choreographer.java
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
}
```

```cpp
// frameworks/base/core/java/android/view/Choreographer.java
void doFrame(long frameTimeNanos, int frame, DisplayEventReceiver.VsyncEventData vsyncEventData) {
    final long startNanos;
    // 帧间隔直接取自VSYNC事件本身(支持动态刷新率),不用构造时的mFrameIntervalNanos
    final long frameIntervalNanos = vsyncEventData.frameInterval;
    // 未经jitter与buffer stuffing调整的原始意图帧时间,用于jank跟踪
    final long intendedFrameTimeNanos = frameTimeNanos;
    long offsetFrameTimeNanos = frameTimeNanos;
    boolean resynced = false;
    // 阶段一 buffer stuffing恢复检查
    switch (updateBufferStuffingState(frameTimeNanos, vsyncEventData)) {
        case NONE:
            break;
        case OFFSET:  // 给动画时间轴加负偏移(减一帧间隔)
            offsetFrameTimeNanos = frameTimeNanos - frameIntervalNanos;
            break;
        case DELAY_FRAME:  // 主动延迟一帧,等下一个VSYNC 给SurfaceFlinger留出时间来消费掉队列中积压的缓冲
            mBufferStuffingState.numberWaitsForNextVsync++;
            mBufferStuffingState.accumulatedDelayNanos += frameIntervalNanos;
            scheduleVsyncLocked();
            return;
        default:
            break;
    }

    try {
        // 阶段二 把本帧时间与一组FrameTimeline写入mFrameData
        FrameTimeline timeline = mFrameData.update(offsetFrameTimeNanos, vsyncEventData);
        ...
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return;  // 没登记过本帧
            }
            mLastNoOffsetFrameTimeNanos = frameTimeNanos;
            // 阶段三 掉帧判定与重同步
            startNanos = System.nanoTime();
            // 当前实际处理时间和这一帧被VSYNC安排的时间 的差值
            final long jitterNanos = startNanos - frameTimeNanos;
            // 主线程处理的时间过久，导致错过了这一帧本该被处理的时间
            if (jitterNanos >= frameIntervalNanos) {
                frameTimeNanos = startNanos;
                if (frameIntervalNanos == 0) {
                    Log.i(TAG, "Vsync data empty due to timeout");
                } else {
                    long lastFrameOffset = jitterNanos % frameIntervalNanos;
                    // 往前对齐到上一次VSYNC信号该被处理的时间，就是对齐到往前的最近要绘制的动画状态
                    frameTimeNanos -= lastFrameOffset;
                    // 判断主线程少处理了多少次VSYNC信号
                    final long skippedFrames = jitterNanos / frameIntervalNanos;
                    if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                        // 掉帧时 经常看到的Skipped XXX frames!
                        Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                                + "The application may be doing too much work on its main "
                                + "thread.");
                    }
                    ...
                }
                // 如果处于恢复状态，代表这次提供的frameTimeNanos
                if (mBufferStuffingState.isRecovering) {
                    frameTimeNanos -= frameIntervalNanos;
                }
                timeline = mFrameData.update(frameTimeNanos, mDisplayEventReceiver, jitterNanos);
                resynced = true;
            }
            if (frameTimeNanos < mLastFrameTimeNanos) {
                scheduleVsyncLocked();  // 时间倒流,等下一帧重新对齐
                return;
            }
            if (mFPSDivisor > 1) {
                long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
                if (timeSinceVsync < (frameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                    scheduleVsyncLocked();  // 限帧率,跳过本帧
                    return;
                }
            }
            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos,
                    vsyncEventData.preferredFrameTimeline().vsyncId,
                    vsyncEventData.preferredFrameTimeline().deadline, startNanos,
                    vsyncEventData.frameInterval, frameTimeNanos);
            mFrameScheduled = false;  // 本帧已消费,允许下一帧重新登记
            mLastFrameTimeNanos = frameTimeNanos;
            mLastFrameIntervalNanos = frameIntervalNanos;
            mLastVsyncEventData.copyFrom(vsyncEventData);
        }
        // 阶段四 锁动画时钟,按序执行五类回调
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT);
        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION);
        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL);
        doCallbacks(Choreographer.CALLBACK_COMMIT);
    } finally {
        AnimationUtils.unlockAnimationClock();
        ...
    }
}
```

## 4. 执行回调

```cpp
// frameworks/base/core/java/android/view/Choreographer.java
void doCallbacks(int callbackType) {
    CallbackRecord callbacks;
    long frameTimeNanos = mFrameData.mFrameTimeNanos;
    synchronized (mLock) {
        // 用now而非frameTime判定到期，一帧内先执行的阶段(input)可能post了后阶段(animation)要跑的回调,这些回调dueTime介于frameTime与now之间,用now才能保证本帧就执行
        final long now = System.nanoTime();
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;
    }
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            ...
            // callback执行
            c.run(mFrameData);
        }
    } finally {
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

```cpp
// frameworks/base/core/java/android/view/Choreographer.java
void run(FrameData frameData) {
    frameData.setInCallback(true);
    if (token == VSYNC_CALLBACK_TOKEN) {
        // Vsync的回调走这
        ((VsyncCallback) action).onVsync(frameData);
    } else {
        run(frameData.getFrameTimeNanos());
    }
    frameData.setInCallback(false);
}

public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
        ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
        ((Runnable)action).run();
    }
}
```

```cpp
// frameworks/base/core/java/android/view/ViewRootImpl.java
public void onVsync(Choreographer.FrameData frameData) {
    // 遍历整个View树，处理所有需要更新的View
    doTraversal(frameData.getFrameTimeNanos());
}

void doTraversal(long frameTimeNanos) {
    if (mTraversalScheduled) {
        // 这里在performTraversals之前是因为如果在performTraversals时有requestLayout或invalidate来的话可以去预约下一次，而不是被丢掉
        mTraversalScheduled = false;
        removeTraversalBarrier(); // 移除同步屏障 performTraversals()可能会发送同步消息，防止死锁
        performTraversals(frameTimeNanos);
    }
}
```

`performTraversals`中根据当前状态决定是否执行`performLayout()`,`performDraw()`,`performDraw()`

`performDraw()`中`mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this)`

// frameworks/base/core/java/android/view/ThreadedRenderer.java
`ThreadedRenderer`中走`syncAndDrawFrame(frameInfo)`
