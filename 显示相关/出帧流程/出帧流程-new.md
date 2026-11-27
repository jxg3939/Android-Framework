# 出帧流程

本文按13个步骤展开一帧从App主线程到屏幕上屏的完整路径。类名、方法名、行号取自`android-17.0.0_r1`（<https://cs.android.com/android/platform/superproject/+/android-17.0.0_r1>），并以本地源码`/home/jxx/android/aosp`（frameworks/base 的 git tag 为`android-17.0.0_r1`）逐节核对；代码块按源码删节，省略处用`...`标出。行号会随提交变化，引用时以方法名为准。

## 总览

|步骤|发生位置|核心入口|
|---|---|---|
|1|App主线程|`ViewRootImpl.scheduleTraversals()`|
|2|App主线程|`Choreographer.doFrame()`|
|3|App主线程|`ViewRootImpl.performTraversals()`→`performDraw()`|
|4|App主线程→RenderThread|`HardwareRenderer.syncAndDrawFrame()`→`DrawFrameTask::drawFrame()`|
|5|RenderThread|`DrawFrameTask::run()`→`syncFrameState()`|
|6|RenderThread|`SkiaOpenGLPipeline::getFrame()`→`eglBeginFrame`→`Surface::dequeueBuffer()`|
|7|RenderThread+GPU|`SkiaOpenGLPipeline::draw()`→`FlushAndSubmit()`|
|8|RenderThread|`eglSwapBuffersWithDamageKHR()`→`queueBuffer()`|
|9|App进程（BLAST）|`BLASTBufferQueue::acquireNextBufferLocked()`→`Transaction::apply()`|
|10|SurfaceFlinger|`scheduleFrame()`预约→`VSyncDispatchTimerQueue`定时器→`vsyncCallback()`|
|11|SurfaceFlinger|`Scheduler::onFrameSignal()`→`SurfaceFlinger::commit()`|
|12|SurfaceFlinger+HWC+GPU|`SurfaceFlinger::composite()`→`Output::present()`|
|13|SurfaceFlinger→App|`onCompositionPresented()`→`releaseBufferCallback()`|

时间轴上：App在`vsync-app`之前完成调度，`vsync-app`到来后`doFrame`；SurfaceFlinger在`vsync-app`之后、`vsync-sf`之前拿到App提交的buffer，在`vsync-sf`回调中commit并composite；HWC在下一个刷新周期扫描上屏，present fence此时signal。更精确地说，App和SF的定时器瞄准的是同一次display vsync，只是唤醒点不同：App提前`appWorkDuration + sfWorkDuration`醒，SF提前`sfWorkDuration`醒，详见10.2。

## 第1步：窗口调度下一帧

![alt text](image.png)

### 1.1 何时触发调度

进入调度的入口有多个，最终都落到`ViewRootImpl.scheduleTraversals()`：

- `invalidate()`：内容失效。硬件加速时`View.invalidate()`经父链调`ViewGroup.invalidateChild()`，后者直接走 HW 快路径`onDescendantInvalidated()`，一路向上到`ViewRootImpl.onDescendantInvalidated()`，再调`invalidate()`把整窗`mDirty`标脏并调度；只有软件渲染才继续沿`invalidateChildInParent()`走到`invalidateRectOnScreen()`合并脏矩形。
- `requestLayout()`：测量或位置可能失效。`View.requestLayout()`沿父链传播到`ViewRootImpl.requestLayout()`。
- 输入事件：`scheduleConsumeBatchedInput()`把输入消费挂到`CALLBACK_INPUT`，消费后若状态变化再由View自己调`invalidate()`或`requestLayout()`。
- 动画：属性动画在`CALLBACK_ANIMATION`阶段改属性，改完调`invalidateViewProperty()`；硬件加速且已有DisplayList时它直接`damageInParent()`→`onDescendantInvalidated()`，不再逐级算脏矩形。
- Insets动画：`CALLBACK_INSETS_ANIMATION`阶段更新Insets，随后触发traversal。
- 窗口变化：`relayoutWindow()`返回新尺寸或Surface，`performTraversals()`内部置位`mFullRedrawNeeded`。

软件渲染路径下，`invalidateChildInParent()`并不遍历View树，它只把脏矩形合并进`mDirty`后调`scheduleTraversals()`；硬件加速路径连矩形合并都不做，`invalidate()`直接把整窗置脏，精确damage留给RenderThread的`DamageAccumulator`去算（第5、7步）。所以“界面只脏了一小块”在Java侧并不会缩小`mDirty`，真正的脏区优化发生在HWUI。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;
    localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
    // 脏区扩张
    final float appScale = mAttachInfo.mApplicationScale;
    // 是否不在窗口边界内
    final boolean intersected = localDirty.intersect(0, 0,
            (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
    if (!intersected) {
        localDirty.setEmpty();
    }
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        scheduleTraversals();
    }
}
```

还有VRI中的全局`invalidate()`

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
```

`requestLayout()`同样最终落到`scheduleTraversals()`，置`mLayoutRequested = true`后调`scheduleTraversals()`。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

### 1.2 scheduleTraversals的合并逻辑

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
void scheduleTraversals() {
    checkThreadCompat();
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        postTraversalBarrier();
        mChoreographer.postVsyncCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalCallback);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

四个动作：

1. `mTraversalScheduled`标记本帧已调度。同一帧内VRI多次`invalidate()`/`requestLayout()`只有第一个去请求Vsync。
2. `postTraversalBarrier()`插入同步屏障，只拦同步消息，不拦异步消息。Vsync消息在`onVsync()`里`setAsynchronous(true)`，能越过屏障；屏障拦的是`scheduleTraversals()`之后post的同步消息，保证它们不会抢在traversal之前执行。`scheduleTraversals()`之前已入队的同步消息不受屏障影响（执行顺序仍由dueTime决定）——ViewRootImpl这段代码上方的注释专门解释了`setText()`后立刻`post()`的Runnable为什么仍能看到新测量结果。
3. `postVsyncCallback(CALLBACK_TRAVERSAL, mTraversalCallback)`把`mTraversalCallback`对象挂进Choreographer对应的callback队列。
4. `notifyRendererOfFramePending()`通知RenderThread即将出帧；`pokeDrawLockIfNeeded()`在Doze场景短暂持有绘制锁。后者条件很窄：仅当`!mDisableDrawWakeLock && mAttachInfo.mDisplayState == STATE_DOZE`，且窗口是有焦点的Activity窗口时才poke；`mDisableDrawWakeLock`由兼容性开关`DISABLE_DRAW_WAKE_LOCK`与`android.view.flags.disable_draw_wake_lock`决定，A17上该flag固定为ENABLED，所以这条路径默认不生效。

### 1.3 doTraversal

`mTraversalCallback`是`TraversalCallback`实例，它收到`FrameData`后只取帧时间：

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
final TraversalCallback mTraversalCallback = new TraversalCallback();

final class TraversalCallback implements Choreographer.VsyncCallback {
    @Override
    public void onVsync(Choreographer.FrameData frameData) {
        doTraversal(frameData.getFrameTimeNanos());
    }
}

// frameworks/base/core/java/android/view/ViewRootImpl.java
void doTraversal(long frameTimeNanos) {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        removeTraversalBarrier();
        performTraversals(frameTimeNanos);
    }
}
```

先清标记、移除屏障，与`scheduleTraversals()`中一一对应，再遍历。清标记和移除同步屏障在`performTraversals()`之前：`performTraversals()`过程中View可能再次`invalidate()`，此时`mTraversalScheduled`已为false，新调度会再插同步屏障并post一次callback，从而申请下一帧。

Choreographer侧还有第二道同样的标记：`doFrame()`在进入各阶段回调之前就把`mFrameScheduled`置为false并更新`mLastFrameTimeNanos`，然后才`doCallbacks()`。所以traversal里再次`scheduleTraversals()`能一路走到`scheduleFrameLocked()`申请下一次`vsync-app`。这两层标记都必须在本帧回调前清掉，缺一层都会让“帧中再次请求下一帧”失效。

## 第2步：Vsync到达与doFrame

![alt text](image-1.png)

### 2.1 从postVsyncCallback到申请Vsync

`postCallbackDelayedInternal()`把回调按`dueTime`插进`mCallbackQueues[callbackType]`；`dueTime <= now`时立刻`scheduleFrameLocked(now)`，否则投一条异步`MSG_DO_SCHEDULE_CALLBACK`

`postVsyncCallback(int callbackType, VsyncCallback callback)`只是把token换成`VSYNC_CALLBACK_TOKEN`再转调：

```java
// frameworks/base/core/java/android/view/Choreographer.java
public void postVsyncCallback(int callbackType, @NonNull VsyncCallback callback) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }
    postCallbackDelayedInternal(callbackType, callback, VSYNC_CALLBACK_TOKEN, 0);
}

private void postCallbackDelayedInternal(int callbackType, Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        // 先把回调加入对应CALLBACK_TRAVERSAL队列，后续VSYNC信号来了之后执行回调
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        // delayMillis=0,本次请求没有延迟，立即执行
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else { // delayMillis>0 发异步消息执行
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

`delayMillis`为0时立刻`scheduleFrameLocked(now)`，这里再合并一次：

```java
// frameworks/base/core/java/android/view/Choreographer.java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            ...
        }
    }
}
```

`mFrameScheduled`保证一次Vsync最多申请一次。`scheduleVsyncLocked()`调用`mDisplayEventReceiver.scheduleVsync()`，native侧落到`NativeDisplayEventReceiver`去`scheduleVsync()`，最终由`DisplayEventReceiver`通过`IDisplayEventConnection`走Binder进入SurfaceFlinger进程的`EventThread::requestNextVsync()`，把该connection标记为需要下一个Vsync。

### 2.2 EventThread投递Vsync

SurfaceFlinger进程在不同版本里EventThread的数量不同。A17的`Scheduler`只创建一条名为`app`的EventThread，服务App的`VSYNC-app`；SF自己的`VSYNC-sf`不走EventThread，而是`MessageQueue`直接注册的定时器回调（第10步）。App侧的Vsync由`VsyncDispatchTimerQueue`按时回调`EventThread::onVsync()`，它把事件塞进`mPendingEvents`并唤醒EventThread线程去`dispatchEvent()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::onVsync(nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime) {
    std::lock_guard<std::mutex> lock(mMutex);
    mLastVsyncCallbackTime = TimePoint::fromNs(vsyncTime);
    LOG_FATAL_IF(!mVSyncState);
    // Trace中观察到VSYNC-app信号翻转
    mVsyncTracer = (mVsyncTracer + 1) % 2;
    mPendingEvents.push_back(makeVSync(mVsyncSchedule->getPhysicalDisplayId(), wakeupTime,
                                       ++mVSyncState->count, vsyncTime, readyTime));
    mCondition.notify_all();
}
```

`mVsyncTracer`就是systrace里`VSYNC-app`的翻转计数线。随后EventThread线程`dispatchEvent()`把Vsync事件写入对应`EventThreadConnection`的`BitTube`，App进程Looper被唤醒。

### 2.3 FrameDisplayEventReceiver把Vsync变成消息

App进程中Looper被唤醒后逐步回调到`FrameDisplayEventReceiver`的`onVsync`

```java
// frameworks/base/core/java/android/view/Choreographer.java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
            VsyncEventData vsyncEventData) {
        long now = System.nanoTime();
        if (timestampNanos > now) {
            timestampNanos = now;
        }
        // 调试保护：正常情况下一次只应有一个在途vsync
        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be one at a time.");
        } else {
            mHavePendingVsync = true;
        }
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        mLastVsyncEventData.copyFrom(vsyncEventData);
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        // 把自己作为异步消息的callback发送，后续执行doFrame()
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }
}
```

关键点是`setAsynchronous(true)`：主线程队列上可能有第1步插入的同步屏障，异步消息可以越过它，同步消息不能。消息时间设为Vsync时刻，队列里更早的同步消息会先处理，Vsync不会饿死整个队列。

### 2.4 doFrame的执行顺序

`doFrame(long frameTimeNanos, int frame, VsyncEventData vsyncEventData)`中回调派发改成了只带callbackType的`doCallbacks(int)`：

```java
// frameworks/base/core/java/android/view/Choreographer.java
mFrameInfo.markInputHandlingStart();
doCallbacks(Choreographer.CALLBACK_INPUT);
mFrameInfo.markAnimationsStart();
doCallbacks(Choreographer.CALLBACK_ANIMATION);
doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION);
mFrameInfo.markPerformTraversalsStart();
doCallbacks(Choreographer.CALLBACK_TRAVERSAL);
doCallbacks(Choreographer.CALLBACK_COMMIT);
```

常量定义为`CALLBACK_INPUT = 0`、`CALLBACK_ANIMATION = 1`、`CALLBACK_INSETS_ANIMATION = 2`、`CALLBACK_TRAVERSAL = 3`、`CALLBACK_COMMIT = 4`。

`doCallbacks(int)`执行时把`mFrameData`传给每条`CallbackRecord.run(FrameData)`：

```java
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

void run(FrameData frameData) {
    frameData.setInCallback(true);
    if (token == VSYNC_CALLBACK_TOKEN) {
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

所以第1步注册的`mTraversalCallback`在`CALLBACK_TRAVERSAL`阶段收到`FrameData`，再调`doTraversal(frameData.getFrameTimeNanos())`。顺序含义：

- `INPUT`最先，保证本帧输入在动画和布局前被消费，后续阶段能看到新状态。
- `ANIMATION`→`INSETS_ANIMATION`：两者都是“改状态”，必须在traversal之前完成。
- `TRAVERSAL`读取前面所有阶段的结果，执行measure、layout、draw。
- `COMMIT`在traversal之后，用于提交本帧事务回调。

`doFrame()`开始回调前还有几件处理，顺序不能颠倒：

1. buffer stuffing恢复判定。`updateBufferStuffingState()`最先执行，返回`DELAY_FRAME`时直接`return`，本帧不发任何回调，只`scheduleVsyncLocked()`等下一次Vsync；返回`OFFSET`时只把`offsetFrameTimeNanos`减一个帧间隔，供FrameTimeline使用。
2. jitter对齐。若`System.nanoTime() - frameTimeNanos`超过一个帧间隔，说明主线程错过Vsync太久，Choreographer把`frameTimeNanos`对齐到当前时间并减去`jitterNanos % frameIntervalNanos`，跳帧过多时打印`Skipped N frames!`。jitter用的是**未**被buffer stuffing调整过的`frameTimeNanos`。
3. 帧时间倒退检查。若`frameTimeNanos < mLastFrameTimeNanos`，`scheduleVsyncLocked()`后返回。
4. FPSDivisor检查。`mFPSDivisor > 1`时若距上一帧不足`frameIntervalNanos * mFPSDivisor`，同样跳过本帧。

`updateBufferStuffingState()`的触发源在别处：第6步`dequeueBuffer()`阻塞在buffer释放上超过半帧时，BBQ会回调`Choreographer.onWaitForBufferRelease()`，把`mBufferStuffingState.isStuffed`置位；下一次`doFrame()`看到这一位就返回`DELAY_FRAME`，从而把App主动推迟一帧、少排一个buffer。`OFFSET`则用于恢复期内把动画时间轴回拨，避免动画在恢复后突然跳进。

## 第3步：performTraversals与DisplayList录制

![alt text](image-2.png)

### 3.1 三个阶段的取舍

`performTraversals()`不是每次都完整执行measure、layout、draw：

- `performMeasure()`：`layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw)`为真时，先走`measureHierarchy()`按期望窗口尺寸测量；窗口尺寸/Insets/配置有变化（`mWidth != host.getMeasuredWidth()`等）时再补一次`performMeasure()`；`mViewMeasureDeferred`为true时用`frame`尺寸补测。根View的MeasureSpec由窗口尺寸和`WindowManager.LayoutParams`算出。
- `performLayout()`：`didLayout = layoutRequested && (!mStopped || mReportNextDraw)`，只有它为真才`performLayout(lp, mWidth, mHeight)`。窗口只是移动时走`maybeHandleWindowMove()`改偏移，不触发layout。
- `performDraw()`：先排除屏幕灭（且不要求报告下一帧）和根View为空两种情况；是否真正绘制由`draw()`里的`!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty`决定；`mFullRedrawNeeded`为true时整窗重绘。硬件加速下进入`draw()`后`dirty`会被`setEmpty()`，矩形脏区到此为止。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
if (layoutRequested) {
    windowSizeMayChange |= measureHierarchy(host, lp, mView.getContext().getResources(),
            desiredWindowWidth, desiredWindowHeight, shouldOptimizeMeasure);
}
...
if (mViewMeasureDeferred) {
    performMeasure(MeasureSpec.makeMeasureSpec(frame.width(), MeasureSpec.EXACTLY),
                   MeasureSpec.makeMeasureSpec(frame.height(), MeasureSpec.EXACTLY));
}
...
final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
if (didLayout) {
    performLayout(lp, mWidth, mHeight);
    ...
}
```

`performMeasure()`调根View的`measure()`，`performLayout()`调根View的`layout()`，二者内部递归整棵树。`performDraw()`则按是否硬件加速走两条完全不同的路径。

### 3.2 硬件加速绘制入口

`performDraw(@Nullable SurfaceSyncGroup)`：

```java
private boolean performDraw(@Nullable SurfaceSyncGroup surfaceSyncGroup) {
    ...
    // 屏幕灭且不是“下一次绘制必须报告”时直接返回
    if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
        mLastPerformDrawSkippedReason = "screen_off";
        return false;
    } else if (mView == null) {
        mLastPerformDrawSkippedReason = "no_root_view";
        return false;
    }

    // 有sync group时强制整窗重绘
    final boolean fullRedrawNeeded = mFullRedrawNeeded || surfaceSyncGroup != null;
    mFullRedrawNeeded = false;
    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, mDrawTrace);
    addFrameCommitCallbackIfNeeded();
    boolean usingAsyncReport;
    try {
        usingAsyncReport = draw(fullRedrawNeeded, surfaceSyncGroup, mSyncBuffer);
        if (mAttachInfo.mThreadedRenderer != null && !usingAsyncReport) {
            mAttachInfo.mThreadedRenderer.setFrameCallback(null);
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```

调用draw():

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
private boolean draw(boolean fullRedrawNeeded, @Nullable SurfaceSyncGroup activeSyncGroup,
            boolean syncBuffer) {
    ...
    if (isHardwareEnabled()) {
        ...
        // Java侧矩形脏区到此为止，HWUI在RenderThread上算damage
        dirty.setEmpty();
        ...
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        ...
    } else {
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                scalingRequired, dirty, surfaceInsets)) {
            return false;
        }
    }
    ...
}
```

硬件加速时并没有直接往Surface上画。`ThreadedRenderer.draw()`做的是“录制+同步”：

```java
// frameworks/base/core/java/android/view/ThreadedRenderer.java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    attachInfo.mViewRootImpl.mViewFrameInfo.markDrawStart();
    updateRootDisplayList(view, callbacks);
    ...
    final FrameInfo frameInfo = attachInfo.mViewRootImpl.getUpdatedFrameInfo();
    int syncResult = syncAndDrawFrame(frameInfo);
    ...
}
```

`updateRootDisplayList()`更新View树的DisplayList，`syncAndDrawFrame()`才把树同步给RenderThread。

### 3.3 每个View的RenderNode与DisplayList

```java
// frameworks/base/core/java/android/view/ThreadedRenderer.java
// 录制根View的DisplayList（节选）
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);
    ...
    // 只有根节点自己需要重录时才开录；否则复用旧的根DisplayList
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onPreDraw(canvas);

            canvas.enableZ();
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            canvas.disableZ();

            callbacks.onPostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
            mRootNode.endRecording();
        }
    }
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}

private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```

根RenderNode的DisplayList里核心只有一条`drawRenderNode(view的RenderNode)`（`onPreDraw()`/`onPostDraw()`还可能前后加少量指令，例如窗口背景、模糊等），这就是“整棵树靠引用串起来”的直接证据。

`View.updateDisplayListIfDirty()`（节选）：

```java
// frameworks/base/core/java/android/view/View.java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if (!canHaveDisplayList()) {
        return renderNode;
    }
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.hasDisplayList()
            || (mRecreateDisplayList)) {
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            // 这里递归遍历子view
            dispatchGetDisplayList();
            return renderNode; // no work needed
        }
        mRecreateDisplayList = true;
        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();
        renderNode.clearStretch();
        final RecordingCanvas canvas = renderNode.beginRecording(width, height);
        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
                ...
            } else {
                computeScroll();
                canvas.translate(-mScrollX, -mScrollY);
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                mPrivateFlags4 |= PFLAG4_HAS_DRAWN;
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    // 这里递归绘制子View
                    dispatchDraw(canvas);
                    ...
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
    } else {
        ...
    }
    return renderNode;
}
```

- 每个View持有一个`RenderNode`（`mRenderNode`）。
- `beginRecording()`返回`RecordingCanvas`，`View.draw()`、`onDraw()`、`dispatchDraw()`里的`drawRect()`、`drawPath()`、`drawText()`都变成DisplayList指令。
- `endRecording()`把DisplayList封进RenderNode。
- 父子View之间不是子指令并进父的DisplayList的关系，而是引用：子View在自己的`draw(Canvas, ViewGroup, long)`末尾，对父View传进来的`RecordingCanvas`调`drawRenderNode(自己的mRenderNode)`，使得父View的DisplayList里多一条指向子RenderNode的引用。

```java
// frameworks/base/core/java/android/view/View.java
boolean draw(@NonNull Canvas canvas, ViewGroup parent, long drawingTime) {
    ...
    if (drawingWithRenderNode) {
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        ((RecordingCanvas) canvas).drawRenderNode(renderNode);
    }
    ...
}
```

RenderNode树引用的实际收益是“只重录脏子树”：硬件加速下后代`invalidate()`沿`onDescendantInvalidated()`上行，只清祖先的`PFLAG_DRAWING_CACHE_VALID`、不置`PFLAG_INVALIDATED`，根节点`updateDisplayListIfDirty()`只走`dispatchGetDisplayList()`往下刷脏节点，父的DisplayList不重录；因为每个View的RenderNode身份不变，旧引用继续指向该子节点最新的DisplayList。所以根RenderNode的DisplayList里是子RenderNode的引用，整棵树没有被压成扁平指令流。

例外是软件层：`onDescendantInvalidated()`遇到`mLayerType == LAYER_TYPE_SOFTWARE`的父容器时，会把该父容器置上`PFLAG_INVALIDATED | PFLAG_DIRTY`并把target升级为它自己，这时它必须重录整层（软件层是一张Bitmap，没有子DisplayList可复用）。

### 3.4 本步结束时的状态

- 所有脏View的DisplayList已更新。
- 根RenderNode已挂好整棵RenderNode树。
- UI线程还没把树交给RenderThread，GPU未执行任何指令。

真正的跨线程同步在下一步。

## 第4步：把RenderNode树交给RenderThread

![alt text](image-3.png)

### 4.1 Java到JNI

```java
// frameworks/base/graphics/java/android/graphics/HardwareRenderer.java
@SyncAndDrawResult
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
    return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}
```

`mNativeProxy`是`private final long`，在构造函数里由`nCreateProxy(!mOpaque, mRootNode.mNativeRenderNode, useIpcCanvas)`赋值；第三参`useIpcCanvas`决定这条渲染通道是否走IPC Canvas（`SkiaIpcPipeline`）。JNI侧`android_view_ThreadedRenderer_createProxy()`执行`new RenderProxy(...)`后把指针按`jlong`返回。

native方法在Java侧声明（`frameworks/base/graphics/java/android/graphics/HardwareRenderer.java`）：

```java
private static native long nCreateProxy(boolean translucent, long rootRenderNode,
        boolean useIpcCanvas);
private static native int nSyncAndDrawFrame(long nativeProxy, long[] frameInfo, int size);
```

注册项在JNI文件（`frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp`）：

```cpp
{"nCreateProxy", "(ZJZ)J", (void*)android_view_ThreadedRenderer_createProxy},
{"nSyncAndDrawFrame", "(J[JI)I", (void*)android_view_ThreadedRenderer_syncAndDrawFrame},
```

JNI函数体三步：校验长度、把Java数组拷进task的`mFrameInfo`、调进`RenderProxy`：

```cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    LOG_ALWAYS_FATAL_IF(frameInfoSize != UI_THREAD_FRAME_INFO_SIZE,
            "Mismatched size expectations, given %d expected %zu", frameInfoSize,
            UI_THREAD_FRAME_INFO_SIZE);
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}
```

### 4.2 RenderProxy与DrawFrameTask

```cpp
// frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
int DrawFrameTask::drawFrame() {
    LOG_ALWAYS_FATAL_IF(!mContext, "Cannot drawFrame with no CanvasContext!");
    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(SYSTEM_TIME_MONOTONIC);
    postAndWait();
    return mSyncResult;
}

void DrawFrameTask::postAndWait() {
    ATRACE_CALL();
    AutoMutex _lock(mLock);
    mRenderThread->queue().post([this]() { run(); });
    mSignal.wait(mLock);
}
```

`postAndWait()`把`DrawFrameTask::run`投递到RenderThread队列，然后在条件变量上等待。这个等待不是等整帧画完：`run()`在`syncFrameState()`返回后调用`unblockUiThread()`提前放行UI线程。是否提前放行只看`syncFrameState()`的返回值`info.prepareTextures`（纹理缓存是否备齐），与`canDrawThisFrame`无关——即使本帧注定被丢，只要纹理备齐也照样先放行。

`drawFrame()`返回的`mSyncResult`是位掩码，含`FrameDropped`、`UIRedrawRequired`、`LostSurfaceRewardIfFound`、`ContextIsStopped`等标志，`ThreadedRenderer.draw()`据此决定是否`invalidate()`或强制relayout。

### 4.3 同步边界的准确含义

- 边界前：DisplayList写入、RenderNode属性修改都在UI线程完成。
- 边界处：UI线程`postAndWait()`，RenderThread开始`run()`。
- 同步点：`syncFrameState()`返回，RenderThread已接走树状态快照和DisplayList引用，`unblockUiThread()`唤醒UI线程；纹理未备齐时这次同步会推迟到`draw()`之后。
- 同步点后：UI线程可以修改View树，RenderThread继续`context->draw()`。

所以UI线程与RenderThread的同步边界是等到RenderNode树状态已被RenderThread接走。

## 第5步：RenderThread同步树状态

![alt text](image-4.png)

UI线程`postAndWait()`之后，RenderThread的`threadLoop()`本来在`waitForWork()`（`mLooper->pollOnce(timeout)`）中阻塞，如今被唤醒，开始进入`processQueue()`（`WorkQueue::process()`在锁外逐个执行`item.work()`,`item.work()`调用最终落到`DrawFrameTask::run()`）

### 5.1 DrawFrameTask::run()

```cpp
// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
void DrawFrameTask::run() {
    mContext->setSyncDelayDuration(systemTime(SYSTEM_TIME_MONOTONIC) - mSyncQueued);
    mContext->setTargetSdrHdrRatio(mRenderSdrHdrRatio);
    ...
    bool canUnblockUiThread;
    bool canDrawThisFrame;
    bool solelyTextureViewUpdates;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        info.forceDrawFrame = mForceDrawFrame;
        mForceDrawFrame = false;
        // 帧状态同步，重点在RenderNode树的同步
        canUnblockUiThread = syncFrameState(info);
        // 决定了本帧能不能绘制
        canDrawThisFrame = !info.out.skippedFrameReason.has_value();
        solelyTextureViewUpdates = info.out.solelyTextureViewUpdates;
    }
    // UI线程postAndWait结束 可以去准备下一帧的绘制
    if (canUnblockUiThread) {
        unblockUiThread();
    }
    ...
    // 省略：把frameCallback/frameCompleteCallback搬到局部变量、enqueueFrameWork()
    if (CC_LIKELY(canDrawThisFrame)) {
        // 走到第6步的Draw()
        context->draw(solelyTextureViewUpdates);
    } else {
        if (GrDirectContext* grContext = mRenderThread->getGrContext()) {
            grContext->flushAndSubmit();
        }
        context->waitOnFences();
    }
    ...
    // 省略：frameCompleteCallback、hardware buffer的pipeline->flush()
    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    int64_t vsync = mFrameInfo[static_cast<int>(FrameInfoIndex::Vsync)];
    ...
    mRenderThread->timeLord().vsyncReceived(vsync, intendedVsync, vsyncId, frameDeadline,
            frameInterval);
    // 当前渲染线程即将使用这个CanvasContext绑定的图形上下文
    bool canDraw = mContext->makeCurrent();
    // 释放之前为了复用而缓存的图像资源
    mContext->unpinImages();
    for (size_t i = 0; i < mLayers.size(); i++) {
        if (mLayers[i]) {
            mLayers[i]->apply();
        }
    }
    mLayers.clear();
    mContext->setContentDrawBounds(mContentDrawBounds);
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);
    bool hasTarget = mContext->hasOutputTarget();
    if (CC_UNLIKELY(!hasTarget || !canDraw)) {
        if (!hasTarget) {
            mSyncResult |= SyncResult::LostSurfaceRewardIfFound;
            info.out.skippedFrameReason = SkippedFrameReason::NoOutputTarget;
        } else {
            mSyncResult |= SyncResult::ContextIsStopped;
            info.out.skippedFrameReason = SkippedFrameReason::ContextIsStopped;
        }
    }
    if (info.out.hasAnimations) {
        if (info.out.requiresUiRedraw) {
            mSyncResult |= SyncResult::UIRedrawRequired;
        }
    }
    if (info.out.skippedFrameReason) {
        mSyncResult |= SyncResult::FrameDropped;
    }
    return info.prepareTextures;
}
```

`TimeLord::vsyncReceived()`记录本帧Vsync、deadline、frameInterval；`makeCurrent()`确保EGL context可用；`unpinImages()`放开UI线程上一帧pin住的图片（pin是为了保证RenderThread上传期间Bitmap不被回收）；`apply()`处理DeferredLayerUpdater（TextureView、SurfaceView等外部纹理层）的更新；`prepareTree()`遍历RenderNode树。

注意`hasOutputTarget()`的检查放在`prepareTree()`之后，prepareTree可能flush掉pending操作（树状态、预取layer等），先查会读到旧状态。失败原因写进`info.out.skippedFrameReason`，同时置`mSyncResult`位，`run()`里的`canDrawThisFrame`和UI线程读到的返回值都从这里来。返回值`info.prepareTextures`为false表示纹理缓存已满、没备齐，所以`run()`不敢提前放行UI线程。

### 5.2 prepareTree做了什么

```cpp
// frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued,
                                RenderNode* target) {
    ...
    mCurrentFrameInfo->importUiThreadInfo(uiFrameInfo);
    mCurrentFrameInfo->markSyncStart();
    info.damageAccumulator = &mDamageAccumulator;
    info.layerUpdateQueue = &mLayerUpdateQueue;
    info.damageGenerationId = mDamageId++;
    info.out.skippedFrameReason = std::nullopt;
    mAnimationContext->startFrame(info.mode);
    if (target) {
        determineColors(target);
    }
    for (const sp<RenderNode>& node : mRenderNodes) {
        info.mode = (node.get() == target ? TreeInfo::MODE_FULL : TreeInfo::MODE_RT_ONLY);
        node->prepareTree(info);
    }
    ...
}
// frameworks/base/libs/hwui/RenderNode.cpp
void RenderNode::prepareTree(TreeInfo& info) {
    ATRACE_CALL();
    LOG_ALWAYS_FATAL_IF(!info.damageAccumulator, "DamageAccumulator missing");
    MarkAndSweepRemoved observer(&info);
    const int before = info.disableForceDark;
    // 递归处理整颗树
    prepareTreeImpl(observer, info, false);
    LOG_ALWAYS_FATAL_IF(before != info.disableForceDark, "Mis-matched force dark");
}
```

`prepareTree()`逐个`node->prepareTree(info)`，把DisplayList属性同步到RenderThread可见状态：变换矩阵、裁剪、alpha、DisplayList版本。每个节点的动画在`RenderNode::prepareTreeImpl()`里由`mAnimatorManager.animate(info)`推进（`info.runAnimations`为真时）；遍历结束后`mAnimationContext->runRemainingAnimations(info)`只处理“遍历时没轮到”的剩余动画节点，随后`freePrefetchedLayers()`、`mIsDirty = true`。

`MODE_FULL`用于窗口主RenderNode，完整同步；`MODE_RT_ONLY`用于其他node，只同步RenderThread需要的最小状态。

`prepareTree()`末尾还有几道决定“本帧画不画”的门：`hasOutputTarget()`为false（NoOutputTarget）、同一Vsync已经画过（AlreadyDrawn）、`mRenderNodes[1]`不可绘制（NothingToDraw）、`reserveNext()`失败（NoBuffer）时，都会写`skippedFrameReason`并提前返回，第6步的`getFrame()`就不会被调用。

### 5.3 syncFrameState()完成

`syncFrameState()`返回后，`run()`按`canDrawThisFrame`决定：true则`CanvasContext::draw()`进入第6步；false则不画，直接`flushAndSubmit()`并`waitOnFences()`，这里的`waitOnFences()`是回收本帧已提交的内部异步工作，与后续Buffer的fence无关

同步点之后UI线程已可继续处理消息（`prepareTextures`为false时例外，它会等到`draw()`结束）。RenderThread拿到的是属性快照+DisplayList引用。这个快照机制依赖于RenderNode内的`mStagingDisplayList`和`mDisplayList`双缓存，`mStagingDisplayList`用于UI线程录制，`mDisplayList`用于RenderThread绘制。

## 第6步：从BLAST BufferQueue申请Buffer

![alt text](image-5.png)

### 6.1 从draw()到dequeue

第5步结束时，`CanvasContext::draw()`已经确定这一帧要画。但在下发第一条绘制命令之前，它必须先拿到一块可写的 buffer：算完damage、标记`IssueDrawCommandsStart`，然后调`getFrame()`。

```cpp
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    ...
    // 获取本帧脏区域
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);
    ...
    // 初始化帧信息，获取当前 Frame
    mCurrentFrameInfo->set(FrameInfoIndex::FrameInterval) =
            mRenderThread.timeLord().frameIntervalNanos();
    mCurrentFrameInfo->markIssueDrawCommandsStart();
    Frame frame = getFrame();
    ...
}
```

`getFrame()`有`mHardwareBuffer`时直接返回逻辑尺寸，否则转给`mRenderPipeline->getFrame()`。以`SkiaOpenGLPipeline`为例，它只是把`mEglSurface`交给`EglManager::beginFrame()`；`beginFrame()`做的是`makeCurrent()`、查询surface尺寸和bufferAge，为后续绘制准备EGL环境：

```cpp
// frameworks/base/libs/hwui/renderthread/EglManager.cpp
Frame EglManager::beginFrame(EGLSurface surface) {
    LOG_ALWAYS_FATAL_IF(surface == EGL_NO_SURFACE, "Tried to beginFrame on EGL_NO_SURFACE!");
    makeCurrent(surface);
    Frame frame;
    frame.mSurface = surface;
    eglQuerySurface(mEglDisplay, surface, EGL_WIDTH, &frame.mWidth);
    eglQuerySurface(mEglDisplay, surface, EGL_HEIGHT, &frame.mHeight);
    frame.mBufferAge = queryBufferAge(surface);
    eglBeginFrame(mEglDisplay, surface);
    return frame;
}
```

注意`beginFrame()`这一步并没有dequeue。`eglBeginFrame()`在AOSP的EGL loader里只做转发：`eglApi.cpp`调`cnx->platform.eglBeginFrame()`，平台默认实现（`egl_platform_entries.cpp:903`）只校验display/surface。真正发起dequeue的是EGL驱动——它在需要back buffer时通过`ANativeWindow`的函数表调`dequeueBuffer`：

```cpp
// frameworks/native/libs/nativewindow/ANativeWindow.cpp
int ANativeWindow_dequeueBuffer(ANativeWindow* window, ANativeWindowBuffer** buffer, int* fenceFd) {
    return window->dequeueBuffer(window, buffer, fenceFd);
}
```

从这里到拿到buffer是一条固定调用链：

```text
EGL驱动 -> ANativeWindow_dequeueBuffer(window, &buf, &fence)
       -> Surface::hook_dequeueBuffer
          -> ReliableSurface::hook_dequeueBuffer（HWUI注册的拦截器）
             -> Surface::dequeueBufferInternal
                -> Surface::dequeueBuffer
                   -> BBQBufferQueueProducer::dequeueBuffer  未实现，调用BufferQueueProducer::dequeueBuffer
                      -> BufferQueueProducer::dequeueBuffer
```

`ANativeWindow_dequeueBuffer()`只是查函数表。链上的每一层各有分工：Surface负责拦截器和参数转换，HWUI的`ReliableSurface`负责兜底，`BufferQueueProducer`负责真正找slot或等待。dequeue的具体时机（beginFrame、首次绘制或swap）由驱动决定，AOSP不保证，但无论何时都会先经过HWUI的拦截器。下面按这条链逐层看。

### 6.2 dequeueBuffer的调用链

链上的第一层是`Surface::hook_dequeueBuffer()`：`ANativeWindow`的函数指针指向它，它的职责是分发——看有没有注册拦截器；有就带上正常路径的函数指针`Surface::dequeueBufferInternal`一起转给拦截器，没有就直接走`Surface::dequeueBuffer()`。

```cpp
// frameworks/native/libs/gui/Surface.cpp
int Surface::hook_dequeueBuffer(ANativeWindow* window,
        ANativeWindowBuffer** buffer, int* fenceFd) {
    Surface* c = getSelf(window);
    // 拦截器路径
    {
        std::shared_lock<std::shared_mutex> lock(c->mInterceptorMutex);
        if (c->mDequeueInterceptor != nullptr) {
            auto interceptor = c->mDequeueInterceptor;
            auto data = c->mDequeueInterceptorData;
            return interceptor(window, Surface::dequeueBufferInternal, data, buffer, fenceFd);
        }
    }
    sp<GraphicBuffer> graphicBuffer;
    status_t result = c->dequeueBuffer(&graphicBuffer, fenceFd);
    *buffer = graphicBuffer.get();
    return result;
}
```

HWUI注册的拦截器是`ReliableSurface`：`CanvasContext::setSurface()`用`std::make_unique<ReliableSurface>(window)`把Surface包一层，`init()`里通过`ANativeWindow_setDequeueBufferInterceptor()`把`ReliableSurface::hook_dequeueBuffer`注册上去。它干两件事：预取命中时直接交出预留的buffer；真实dequeue失败时返回一块1x1的scratch buffer兜底，让本帧“画个假的”，queue时再丢弃，避免整个surface被判定为损坏。A17里buffer预取被`constexpr bool DISABLE_BUFFER_PREFETCH = true`关掉，所以预取分支基本不生效，正常情况下直接走真实dequeue，失败时才用这块兜底buffer。

```cpp
// frameworks/base/libs/hwui/renderthread/ReliableSurface.cpp（节选）
int ReliableSurface::hook_dequeueBuffer(ANativeWindow* window,
                                        ANativeWindow_dequeueBufferFn dequeueBuffer, void* data,
                                        ANativeWindowBuffer** buffer, int* fenceFd) {
    ReliableSurface* rs = reinterpret_cast<ReliableSurface*>(data);
    // 预取命中：直接把预留的buffer和fence交出去
    if (rs->mReservedBuffer) {
        *buffer = rs->mReservedBuffer;
        *fenceFd = rs->mReservedFenceFd.release();
        rs->mReservedBuffer = nullptr;
        return OK;
    }
    int result = dequeueBuffer(window, buffer, fenceFd);
    if (result != OK) {
        *buffer = rs->acquireFallbackBuffer(result);   // 1x1兜底buffer
        *fenceFd = -1;
        return *buffer ? OK : INVALID_OPERATION;
    }
    rs->mHasDequeuedBuffer = true;
    return OK;
}
```

拦截器内部调用的就是`Surface::dequeueBufferInternal()`：它调`Surface::dequeueBuffer()`拿到`sp<GraphicBuffer>`，再以`ANativeWindowBuffer*`的形式返回给拦截器。

```cpp
int Surface::dequeueBufferInternal(ANativeWindow* window, ANativeWindowBuffer** buffer,
                                   int* fenceFd) {
    Surface* c = getSelf(window);

    sp<GraphicBuffer> graphicBuffer;
    status_t result = c->dequeueBuffer(&graphicBuffer, fenceFd);
    *buffer = graphicBuffer.get();
    return result;
}
```

`Surface::dequeueBuffer()`把请求转给`mGraphicBufferProducer`：

```cpp
// frameworks/native/libs/gui/Surface.cpp
int Surface::dequeueBuffer(sp<GraphicBuffer>* buffer, int* fenceFd) {
    {
        std::scoped_lock _dl(mDebugMutex);
        ATRACE_FORMAT("dequeueBuffer - %s", mDebugName.c_str());
    }
    SURF_LOGV("Surface::dequeueBuffer");
    ...
    // BLAST下Surface实例通常是BBQSurface，它的mGraphicBufferProducer来源于BLASTBufferQueue::createBufferQueue，该方法内部会创建BBQBufferQueueProducer作为生产者实体，这个指针实际指向的是本进程内的BBQBufferQueueProducer对象
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &mBufferAge,
                                                            dqInput.getTimestamps ?
                                                                    &frameTimestamps : nullptr);
    ...
}
```

`mGraphicBufferProducer`在BLAST下指向本进程的`BBQBufferQueueProducer`。这个类只重写了`connect()`、`waitForBufferRelease()`等少数方法，没有重写`dequeueBuffer()`，所以请求直接进`BufferQueueProducer::dequeueBuffer()`：

```cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    ATRACE_CALL();
    ...
    int found = BufferItem::INVALID_BUFFER_SLOT;
    while (found == BufferItem::INVALID_BUFFER_SLOT) {
        status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &found);
        if (status != NO_ERROR) {
            return status;
        }
        ...
    }
    ...
    *outSlot = found;
    ...
    mSlots[found].mBufferState.dequeue();
    ...
    *outFence = mSlots[found].mFence;
    ...
    addAndGetFrameTimestamps(nullptr, outTimestamps);
    return returnFlags;   // 可能是 OK、WOULD_BLOCK 或 TIMED_OUT
}
```

`BufferQueueProducer::dequeueBuffer()`先做参数检查，然后在这个 while 里反复调`waitForFreeSlotThenRelock()`找slot；找到后用`*outSlot`把slot号交出去。注意`mSlots[found].mBufferState.dequeue()`并不是“拿到buffer”，它只是状态记账：把这块slot从FREE迁到DEQUEUED（A17里是`mDequeueCount++`），表示slot归生产者所有，消费者不能再动它。真正的buffer本体是`mSlots[found].mGraphicBuffer`，可能为空；为空或属性不匹配时`returnFlags`会带`BUFFER_NEEDS_REALLOCATION`，由调用方`Surface::dequeueBuffer()`再`requestBuffer()`取回。最后返回给上层的还有`*outFence`（这块buffer上一轮使用者的fence）和`returnFlags`。

找不到空闲slot时，它会停在`waitForFreeSlotThenRelock()`里等待，具体见6.3。

### 6.3 什么时候会在这里等待

等待的调用关系是`dequeueBuffer()` → `waitForFreeSlotThenRelock()` → `waitForBufferRelease()`，等待发生在`waitForFreeSlotThenRelock()`的循环尾部：找不到空闲slot（或队列里buffer太多）时，它先判断能不能不阻塞，不能就调`waitForBufferRelease()`：

```cpp
// frameworks/native/libs/gui/BufferQueueProducer.cpp（节选：waitForFreeSlotThenRelock() 的循环尾部）
...
tryAgain = (*found == BufferQueueCore::INVALID_BUFFER_SLOT) ||
           tooManyBuffers;
if (tryAgain) {
    // 非阻塞模式直接返回，不在这里等
    if ((mCore->mDequeueBufferCannotBlock || mCore->mAsyncMode) &&
            (acquiredCount <= mCore->mMaxAcquiredBufferCount)) {
        return WOULD_BLOCK;
    }
    if (status_t status = waitForBufferRelease(lock, mDequeueTimeout);
        status == TIMED_OUT) {
        return TIMED_OUT;
    }
}
```

上面这段就是等待的入口，但真正会不会阻塞还取决于配置：普通BufferQueue里若producer与consumer都由App控制、又没有显式设dequeue timeout，`mDequeueBufferCannotBlock`为true，`dequeueBuffer()`直接返回`WOULD_BLOCK`；BLAST在`initialize()`里把timeout设成`int64_t`最大值，所以一定走阻塞等待。

确定要等之后，具体怎么等由`waitForBufferRelease()`的两个实现决定。普通BufferQueue等`mDequeueCondition`：

```cpp
// frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::waitForBufferRelease(std::unique_lock<std::mutex>& lock,
                                                   nsecs_t timeout) const {
    if (mDequeueTimeout >= 0) {
        std::cv_status result =
                mCore->mDequeueCondition.wait_for(lock, std::chrono::nanoseconds(timeout));
        if (result == std::cv_status::timeout) {
            return TIMED_OUT;
        }
    } else {
        mCore->mDequeueCondition.wait(lock);
    }
    return OK;
}
```

BLAST路径在`BBQBufferQueueProducer`里重写这个方法，改为等SurfaceFlinger的release通知：

```cpp
// frameworks/native/libs/gui/BLASTBufferQueue.cpp
status_t waitForBufferRelease(std::unique_lock<std::mutex>& bufferQueueLock,
                              nsecs_t timeout) const override {
    mBufferReleaseReader->clearInterrupts();
    UnlockGuard unlockGuard{bufferQueueLock};
    ...
    status_t status =
            mBufferReleaseReader->readBlocking(id, fence, maxAcquiredBufferCount, timeout);
    ...
    bbq->releaseBufferCallback(id, fence, maxAcquiredBufferCount);
    return OK;
}
```

`mBufferReleaseReader`读`BufferReleaseChannel`（epoll监听一个eventfd），SurfaceFlinger释放buffer时往channel写release信息（第13步）。所以“释放后才有空闲slot”在这里是显式的：dequeue阻塞在`readBlocking()`的`epoll_wait()`上，channel有数据后eventfd被写、epoll返回，`releaseBufferCallback()`把slot还给队列，`BufferQueueProducer`重新检查空闲slot，dequeue才返回。

`waitForBufferRelease()`在返回前还会调`getWaitForBufferReleaseCallback()`，把“这次阻塞了多久”报给Choreographer：超过半帧就置位buffer stuffing，下一次`doFrame()`主动`DELAY_FRAME`一帧（见2.4）。所以第6步的阻塞和App侧的丢帧策略是连动的。

### 6.4 本步拿到的两样关键东西

等slot找到、`Surface::dequeueBuffer()`把buffer取回，本步就拿到两样东西：

- 一个空闲slot对应的`GraphicBuffer`。
- 该buffer从消费者侧继承的release fence（`fenceFd`）。EGL开始往这块buffer画之前必须等它，保证上一轮使用者已读完。

此时还没有任何绘制指令被提交。

## 第7步：Skia提交GPU命令

![alt text](image-6.png)

### 7.1 建立SkSurface

```cpp
// frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    ...
    SkRect windowDirty = computeDirtyRect(frame, &dirty);
    // 执行实际绘制，生成绘制结果
    ATRACE_FORMAT("Drawing " RECT_STRING, SK_RECT_ARGS(dirty));
    IRenderPipeline::DrawResult drawResult;
    {
    drawResult = mRenderPipeline->draw(
            frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue,
            mContentDrawBounds, mOpaque, mLightInfo, mRenderNodes,
            &(profiler()), mBufferParams, profilerLock());
    }
    waitOnFences();
    ...
}
```

注意`waitOnFences()`等的不是GPU fence，也不是BufferQueue的acquire/release fence，而是`mFrameFences`里的一批`std::future<void>`：`DrawFrameTask::run()`会把UI线程注册的`FrameDrawingCallback`通过`enqueueFrameWork()`投到`CommonPool`上执行，future存进`mFrameFences`；`waitOnFences()`在这里逐个`get()`，确保这些回调（比如按frameNumber合并事务、注册`FrameCommitCallback`）在`swapBuffers()`把buffer交给BufferQueue之前完成。所以它等的是CPU侧的异步回调，不是等GPU画完。

此处`mRenderPipeline`参考`SkiaOpenGLPipeline`

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp
IRenderPipeline::DrawResult SkiaOpenGLPipeline::draw(
        const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
        const LightGeometry& lightGeometry, LayerUpdateQueue* layerUpdateQueue,
        const Rect& contentDrawBounds, bool opaque, const LightInfo& lightInfo,
        const std::vector<sp<RenderNode>>& renderNodes, FrameInfoVisualizer* profiler,
        const HardwareBufferRenderParams& bufferParams, std::mutex& profilerLock) {
    ...
    renderFrame(*layerUpdateQueue, dirty, renderNodes, opaque, contentDrawBounds, surface,
                preTransform);
    ...
}
```

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaPipeline.cpp
void SkiaPipeline::renderFrame(const LayerUpdateQueue& layers, const SkRect& clip,
                               const std::vector<sp<RenderNode>>& nodes, bool opaque,
                               const Rect& contentDrawBounds, sk_sp<SkSurface> surface,
                               const SkMatrix& preTransform) {
    ...
    // draw all layers up front
    renderLayersImpl(layers, opaque);
    renderFrameImpl(clip, nodes, opaque, contentDrawBounds, canvas, preTransform);
    ...
}
```

### 7.2 遍历RenderNode树

`SkiaPipeline::renderFrame()`取canvas、画layer队列后调`renderFrameImpl()`,把RenderNode树画到SkCanvas上

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaPipeline.cpp
void SkiaPipeline::renderFrameImpl(const SkRect& clip,
                                   const std::vector<sp<RenderNode>>& nodes, bool opaque,
                                   const Rect& contentDrawBounds, SkCanvas* canvas,
                                   const SkMatrix& preTransform) {
    SkAutoCanvasRestore saver(canvas, true);
    auto clipRestriction = preTransform.mapRect(clip).roundOut();
    if (CC_UNLIKELY(isCapturingSkp())) {
        // 录SKP时不能把clip限制写进画布，否则回放会丢内容
        canvas->drawAnnotation(SkRect::Make(clipRestriction), "AndroidDeviceClipRestriction",
            nullptr);
    } else {
        canvas->androidFramework_setDeviceClipRestriction(clipRestriction);
    }
    canvas->concat(preTransform);
    if (!opaque) {
        canvas->clear(SK_ColorTRANSPARENT);
    }
    if (1 == nodes.size()) {
        if (!nodes[0]->nothingToDraw()) {
            RenderNodeDrawable root(nodes[0].get(), canvas);
            root.draw(canvas);
        }
    } else if (0 == nodes.size()) {
        // nothing to draw
    } else {
        // 多node：backdrop / content / overlay
        ...
    }
}
```

DisplayList里的每条指令由`SkiaDisplayList`中的drawable重放。UI线程录制的`drawRect`、`drawPath`、`drawRenderNode`在RenderThread上翻译成`SkCanvas`调用；`drawRenderNode`对应的drawable递归进入子RenderNode，直到叶子节点的绘制指令

### 7.3 提交

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp
IRenderPipeline::DrawResult SkiaOpenGLPipeline::draw(
        const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
        const LightGeometry& lightGeometry, LayerUpdateQueue* layerUpdateQueue,
        const Rect& contentDrawBounds, bool opaque, const LightInfo& lightInfo,
        const std::vector<sp<RenderNode>>& renderNodes, FrameInfoVisualizer* profiler,
        const HardwareBufferRenderParams& bufferParams, std::mutex& profilerLock) {
    ...
    {
        ATRACE_NAME("flush commands");
        skgpu::ganesh::FlushAndSubmit(surface);
    }
    layerUpdateQueue->clear();
}
```

`FlushAndSubmit()`把Skia累积的绘制命令flush成GPU命令并提交给驱动。提交是异步的：不等待GPU执行。`CanvasContext::draw()`调用`mRenderPipeline->draw(...)`，然后`waitOnFences()`

第7步不负责把buffer交还BufferQueue，buffer仍被生产者持有；要等`swapBuffers()`调`eglSwapBuffersWithDamageKHR()`，EGL才把buffer连同GPU完成信号交给`Surface::queueBuffer()`。

## 第8步：queueBuffer与QueuedBuffer计数

![alt text](image-7.png)

### 8.1 eglSwapBuffers到Surface::queueBuffer

命令提交后准备swapBuffer，由绘制结果和EGL状态决定是否把本帧缓冲区提交给系统：

```cpp
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    ...
    bool requireSwap = false;
    bool didSwap = mRenderPipeline->swapBuffers(
            frame, drawResult, windowDirty, mCurrentFrameInfo, &requireSwap);
    ...
    mCurrentFrameInfo->markSwapBuffersCompleted();
    ...
    // 通知帧提交回调：didSwap才是真的把buffer交给了BufferQueue
    if (didSwap) {
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, true /* didProduceBuffer */);
        }
        mFrameCommitCallbacks.clear();
    }
    ...
}
```

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp
bool SkiaOpenGLPipeline::swapBuffers(const Frame& frame, IRenderPipeline::DrawResult& drawResult,
                                     const SkRect& screenDirty, FrameInfo* currentFrameInfo,
                                     bool* requireSwap) {
    GL_CHECKPOINT(LOW);
    currentFrameInfo->markSwapBuffers();
    if (mHardwareBuffer) {
        return false;
    }
    *requireSwap = drawResult.success || mEglManager.damageRequiresSwap();
    if (*requireSwap && (CC_UNLIKELY(!mEglManager.swapBuffers(frame, screenDirty)))) {
        return false;
    }
    return *requireSwap;
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/EglManager.cpp
bool EglManager::swapBuffers(const Frame& frame, const SkRect& screenDirty) {
    if (CC_UNLIKELY(Properties::waitForGpuCompletion)) {
        ATRACE_NAME("Finishing GPU work");
        fence();
    }
    EGLint rects[4];
    frame.map(screenDirty, rects);
    eglSwapBuffersWithDamageKHR(mEglDisplay, frame.mSurface, rects, screenDirty.isEmpty() ? 0 : 1);
    ...
}
```

`eglSwapBuffersWithDamageKHR()`由EGL驱动实现，最终调`ANativeWindow_queueBuffer()`，进入`Surface::hook_queueBuffer()`，再转成内部类型：

```text
EGL驱动 -> ANativeWindow_queueBuffer(window, buffer, fenceFd)
  ->Surface::hook_queueBuffer
     ->ReliableSurface::hook_queueBuffer（HWUI拦截器：假buffer直接丢弃）
        ->Surface::queueBufferInternal
           ->Surface::queueBuffer(const sp<GraphicBuffer>&, const sp<Fence>&, SurfaceQueueBufferOutput*)
              ->Surface::queueBufferImpl()
                 ->BufferQueueProducer::queueBuffer()   // BBQ 未重写
```


```cpp
// frameworks/native/libs/gui/Surface.cpp
int Surface::hook_queueBuffer(ANativeWindow* window,
        ANativeWindowBuffer* buffer, int fenceFd) {
    Surface* c = getSelf(window);
    // 拦截器路径
    {
        std::shared_lock<std::shared_mutex> lock(c->mInterceptorMutex);
        if (c->mQueueInterceptor != nullptr) {
            auto interceptor = c->mQueueInterceptor;
            auto data = c->mQueueInterceptorData;
            return interceptor(window, Surface::queueBufferInternal, data, buffer, fenceFd);
        }
    }
    return c->queueBuffer(GraphicBuffer::from(buffer), sp<Fence>::make(fenceFd));
}
```

拦截器正常情况下落到`queueBufferInternal`

```cpp
int Surface::queueBufferInternal(ANativeWindow* window, ANativeWindowBuffer* buffer, int fenceFd) {
    Surface* c = getSelf(window);
    return c->queueBuffer(GraphicBuffer::from(buffer), sp<Fence>::make(fenceFd));
}
```

正常路径下走拦截器也就是多一步调用，最终还是落到`Surface::queueBuffer`

```cpp
// frameworks/native/libs/gui/Surface.cpp
status_t Surface::queueBuffer(const sp<GraphicBuffer>& buffer, const sp<Fence>& fence,
                              SurfaceQueueBufferOutput* surfaceOutput) {
    ATRACE_CALL();
    SURF_LOGV("Surface::queueBuffer");
    return queueBufferImpl(buffer, fence, nullptr, surfaceOutput);
}
```

```cpp
// frameworks/native/libs/gui/Surface.cpp
status_t Surface::queueBufferImpl(const sp<GraphicBuffer>& buffer, const sp<Fence>& fence,
                                  const SurfaceQueueBufferInput* maybeSurfaceInput,
                                  SurfaceQueueBufferOutput* surfaceOutput) {

    ...
    slot = getSlotFromBufferLocked(buffer);
    if (slot < 0) {
        return slot;
    }
    ...
    //  BLAST下Surface实例通常是BBQSurface，它的mGraphicBufferProducer来源于BLASTBufferQueue::createBufferQueue，该方法内部会创建BBQBufferQueueProducer作为生产者实体，这个指针实际指向的是本进程内的BBQBufferQueueProducer对象
    status_t err = mGraphicBufferProducer->queueBuffer(slot, igbpInput, &igbpOutput);
    ...
}
```

`queueBufferImpl()`先按buffer找出slot，再调用`mGraphicBufferProducer->queueBuffer(slot, igbpInput, &igbpOutput)`。`igbpInput`里的fence就是“GPU何时写完这块buffer”的signal fence，随buffer入队，App侧不会等它。但`BufferQueueProducer::queueBuffer()`末尾有一道EGL限流：`connectedApi == NATIVE_WINDOW_API_EGL && enableEglCpuThrottling`时，它会`lastQueuedFence->waitForever("Throttling EGL Production")`，等的是**上一帧**的GPU完成fence——最多允许两个buffer在途，不允许第三个。所以`queueBuffer()`本身可能在这里阻塞，但等的不是本帧的fence。`Properties::waitForGpuCompletion`是另一回事，它在`EglManager::swapBuffers()`里、swap之前同步等GPU，属于调试开关。

### 8.2 queueBuffer与onFrameAvailable

`mGraphicBufferProducer`指向`BLASTBufferQueue`创建的`BufferQueue`的producer。它用的是自定义的`BBQBufferQueueProducer`

`BBQBufferQueueProducer`没有重写`queueBuffer()`，走`BufferQueueProducer::queueBuffer()`：标记`QUEUED`、记录frame number、更新`FrameEventHistory`，然后回调consumer listener。`BLASTBufferQueue::initialize()`把`mBufferItemConsumer`的listener设为BBQ自己：

```cpp
// frameworks/native/libs/gui/BLASTBufferQueue.cpp
mBufferItemConsumer->setFrameAvailableListener(wp<BLASTBufferQueue>::fromExisting(this));
```

```cpp
// frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ATRACE_CALL();
    ATRACE_BUFFER_INDEX(slot);
    sp<Fence> acquireFence;
    ...
    mSlots[slot].mFence = acquireFence; 
    mSlots[slot].mBufferState.queue();
    ...
    addAndGetFrameTimestamps(&newFrameEventsEntry,
            getFrameTimestamps ? &output->frameTimestamps : nullptr);
    ...
    frameAvailableListener->onFrameAvailable(item);
}
```

`BufferQueueProducer::queueBuffer`中完成了对`BufferItem`的包装，`BufferItem`包含了缓冲区的句柄以及所有相关的元数据，如mGraphicBuffer, mFence, mSlot, mFrameNumber；listener是`BLASTBufferQueue`自己（它实现了`ConsumerBase::FrameAvailableListener`，不是另一个consumer对象）。

`queueBuffer`落到`BLASTBufferQueue::onFrameAvailable()`。注意这里没有Binder跨越：BLASTBufferQueue和BufferQueue都在App进程，listener是在调用`queueBuffer()`的RenderThread上同步回调的，所以第8步和第9步的acquire发生在同一个线程、同一段调用栈里。

### 8.3 onFrameAvailable里的计数

```cpp
// frameworks/native/libs/gui/BLASTBufferQueue.cpp
void BLASTBufferQueue::onFrameAvailable(const BufferItem& item) {
    ...
    {
        UNIQUE_LOCK_WITH_ASSERTION(mMutex);
        BBQ_TRACE();
        bool waitForTransactionCallback = !mSyncedFrameNumbers.empty();
        ...
        mNumFrameAvailable++;
        if (waitForTransactionCallback && mNumFrameAvailable >= 2) {
            acquireAndReleaseBuffer();
        }
        // mQueuedBufferTrace = "QueuedBuffer - " + mName + "BLAST#" + std::to_string(mProducerId);
        ATRACE_INT(mQueuedBufferTrace.c_str(),
                   mNumFrameAvailable + mNumAcquired - mPendingRelease.size());
        ...
        if (syncTransactionSet) {
            ...
        } else if (!waitForTransactionCallback) {
            acquireNextBufferLocked(std::nullopt);
        }
    }
}
```

Perfetto里看到的是`QueuedBuffer - VRIBLAST#N`。值是`mNumFrameAvailable + mNumAcquired - mPendingRelease.size()`，含义是App已queue、尚未被SurfaceFlinger释放回去的buffer总数，不是BufferQueue内部队列长度。

`mNumFrameAvailable`在queue时+1；-1有两处：`acquireNextBufferLocked()`成功acquire时，以及`acquireAndReleaseBuffer()`（sync事务丢弃/排空影子队列）时。`mNumAcquired`在`acquireNextBufferLocked()`中+1，在`releaseBuffer()`里-1。被`rejectBuffer()`拒绝的buffer只走`mNumFrameAvailable--`，不会+`mNumAcquired`。

`mNumAcquired`不能超过`mMaxAcquiredBuffers + 2`。`acquireNextBufferLocked()`开头就做这两项检查，超过时返回`NO_BUFFER_AVAILABLE`；但只有在sync事务路径（`mTransactionReadyCallback != nullptr`）里`onFrameAvailable()`才会在`mCallbackCV`上等，普通路径直接忽略返回值——这一帧继续留在影子队列里，等`releaseBufferCallbackLocked()`或事务回调把buffer还回来后再acquire。

## 第9步：BLAST Consumer把buffer提交给SurfaceFlinger

![alt text](image-8.png)

### 9.1 acquire并setBuffer

```cpp
// frameworks/native/libs/gui/BLASTBufferQueue.cpp
status_t BLASTBufferQueue::acquireNextBufferLocked(const std::optional<SurfaceComposerClient::Transaction*> transaction) {
    if (mNumFrameAvailable == 0) {
        return BufferQueue::NO_BUFFER_AVAILABLE;
    }
    if (mNumAcquired >= (mMaxAcquiredBuffers + 2)) {
        return BufferQueue::NO_BUFFER_AVAILABLE;
    }
    ...
    SurfaceComposerClient::Transaction localTransaction;
    bool applyTransaction = true;
    SurfaceComposerClient::Transaction* t = &localTransaction;
    ...
    BufferItem bufferItem;
    status_t status = mBufferItemConsumer->acquireBuffer(&bufferItem, 0 /* expectedPresent */, false);
    ...
    auto buffer = bufferItem.mGraphicBuffer;
    mNumFrameAvailable--;
    ...
    // buffer为空、或尺寸不符时releaseBuffer后递归重试，这条路径不会再++mNumAcquired
    mNumAcquired++;
    mLastAcquiredFrameNumber = bufferItem.mFrameNumber;
    ReleaseCallbackId releaseCallbackId(buffer->getId(), mLastAcquiredFrameNumber);
    mSubmitted.emplace_or_replace(releaseCallbackId, bufferItem);
    ...
    auto releaseBufferCallback = makeReleaseBufferCallbackThunk();
    sp<Fence> fence = bufferItem.mFence ? sp<Fence>::make(bufferItem.mFence->dup()) : Fence::NO_FENCE;
    t->setBuffer(mSurfaceControl, buffer, fence, bufferItem.mFrameNumber, mProducerId,
                 releaseBufferCallback, dequeueTime);
    t->setDataspace(...);
    t->setHdrMetadata(...);
    t->setSurfaceDamageRegion(...);
    t->addTransactionCompletedCallback(makeTransactionCallbackThunk(), nullptr);
    ...
    mergePendingTransactions(t, bufferItem.mFrameNumber);
    if (applyTransaction) {
        t->setApplyToken(mApplyToken).apply(false, true);
        mAppliedLastTransaction = true;
        mLastAppliedFrameNumber = bufferItem.mFrameNumber;
        mSetBufferBarrier = true;
    } else {
        // 等同一Surface上帧号为mLastAppliedFrameNumber的buffer
        if (mSetBufferBarrier) {
            t->setBufferHasBarrier(mSurfaceControl, mLastAppliedFrameNumber);
        }
        mAppliedLastTransaction = false;
    }
    ...
}
```

- `acquireBuffer()`拿到`BufferItem`，其`mFence`就是生产者queue时的GPU完成fence，在SurfaceFlinger侧作为acquire fence。第三个参数`false`是`waitForFence`，表示消费端**不**在这里等fence，而是把它随`setBuffer()`交给SF，由SF latch时判断（`latchBufferImpl()`里的`fenceHasSignaled()`）。
- `mNumFrameAvailable--`、`mNumAcquired++`：从“可用”变成“已acquire”。
- `mSubmitted[ReleaseCallbackId(buffer->getId(), frameNumber)]`记录buffer id和frame number，SF释放时用这个映射找回`BufferItem`。
- `setBuffer()`写入buffer和acquire fence，并带上第13步要用的`releaseBufferCallback`。
- `setDataspace()`、`setHdrMetadata()`、`setSurfaceDamageRegion()`在同一个事务里提交数据空间、HDR元数据、脏区域。
- `addTransactionCompletedCallback()`注册事务完成回调，供`TransactionCompletedListener`使用。
- `setApplyToken(mApplyToken).apply(false, true)`：token让SF按队列排序，`synchronous=false`不等提交完成，`oneWay=true`走单向Binder，详见9.2。
- `t->setBufferHasBarrier(mSurfaceControl, mLastAppliedFrameNumber)`：只有在`mSetBufferBarrier`为true（之前apply过带buffer的事务）时才加，让本帧事务等同一Surface上帧号为`mLastAppliedFrameNumber`的buffer先完成。

### 9.2 apply：从客户端到SF的事务队列

`setApplyToken(mApplyToken)`只是把token存进`Transaction`，真正的提交在`apply(false, true)`

```cpp
// frameworks/native/libs/gui/SurfaceComposerClient.cpp
status_t SurfaceComposerClient::Transaction::apply(bool synchronous, bool oneWay) {
    if (mStatus != NO_ERROR) {
        return mStatus;
    }
    // SyncCallback恒创建，但只有synchronous才init并注册
    std::shared_ptr<SyncCallback> syncCallback = std::make_shared<SyncCallback>();
    if (synchronous) {
        syncCallback->init();
        addTransactionCommittedCallback(SyncCallback::getCallback(syncCallback),
                                        /*callbackContext=*/nullptr);
    }
    ...
    cacheBuffers();                       // BufferCache 命中就把 buffer 换成 cacheId
    if (oneWay) {
        if (synchronous) {
            ALOGE("Transaction attempted to set synchronous and one way at the same time"
                  " this is an invalid request. Synchronous will win for safety");
        } else {
            mState.mFlags |= ISurfaceComposer::eOneWay;
        }
    }
    ...
    sp<IBinder> applyToken = mApplyToken ? mApplyToken : getDefaultApplyToken();
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    TransactionState state = std::move(mState);
    status_t binderStatus = sf->setTransactionState(std::move(state), applyToken);
    clear();
    if (synchronous && binderStatus == OK) {
        syncCallback->wait();
    }
    mStatus = NO_ERROR;
    return binderStatus;
}
```

两个参数`synchronous=false`：`SyncCallback`虽然会创建，但不init、不注册、也不`wait()`；`oneWay=true`：给`mState.mFlags`置`ISurfaceComposer::eOneWay`，Binder proxy据此用`FLAG_ONEWAY`发起transact；然后`sf->setTransactionState(std::move(state), applyToken)`，`clear()`后返回。注意`Transaction::apply()`在`SurfaceComposerClient.cpp`里，不在`Transaction.cpp`。

```cpp
// frameworks/native/libs/gui/ISurfaceComposer.cpp
status_t BpSurfaceComposer::setTransactionState(TransactionState&& state, const sp<IBinder>& applyToken) {
    ...
    if (state.mFlags & ISurfaceComposer::eOneWay) {
        return remote()->transact(BnSurfaceComposer::SET_TRANSACTION_STATE, data, &reply,
                                  IBinder::FLAG_ONEWAY);
    } else {
        return remote()->transact(BnSurfaceComposer::SET_TRANSACTION_STATE, data, &reply);
    }
}
```

接口是`ISurfaceComposer`，到了SF，Binder线程进入`SurfaceFlinger::setTransactionState()`，

### 9.3 SurfaceFlinger::setTransactionState()

`setTransactionState()`这个操作并不在SurfaceFlinger主线程上，而是在一个Binder线程上

主要做以下几件事：

- 识别调用者并裁剪权限：读取getCallingPid/Uid，对每个`layer_state_t`、`display_state`执行`sanitize(permissions)`，把无权改的字段静默抹掉
- 处理buffer缓存：按`uncacheBuffers`调用`ClientCache::erase()`，释放客户端不再需要的cache条目
- `transactionState.mComposerStates`逐条解析成`ResolvedComposerState`；其中`hasBufferChanges() && hasValidBuffer() && surface`的才执行`mBufferCountTracker.increment()`，`BufferTX -`计数+1
- 封装`QueuedTransactionState`，`frameTimelineInfo`、`resolvedStates`、`flags`、`applyToken`、barriers、early wakeup info等全部封装进去
- `QueuedTransactionState`进入无锁队列：mTransactionHandler.queueTransaction()只做push和`PendingTransactionCount`计数，不会被主线程commit阻塞Binder线程；oneway下客户端连这一步的返回都不等
- `setTransactionFlags()`

### 9.4 setTransactionFlags()

```cpp
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
        // Even if the next frame is already scheduled, we should reset the idle timer as a new activity just happened.
        mScheduler->resetAllIdleTimers();
    }
}
```

1. `modulateVsync()`用于调整VSync调度策略，决定SF提前/延后多久醒
2. Binder线程`fetch_or(mask)`置位，SF主线程`commit()`里 `clearTransactionFlags(mask)` 取走，并据此决定`flushTransactions`。`fetch_or(mask)`的返回值是旧值，`transactionFlags & mask` 为 0 说明之前没排过，才需要真正叫醒 SF；否则说明已经有在途调度，不重复排，新事务会被已经排好的那一帧一起flush掉，同一帧内不管来多少笔事务（多个 App、同一 App多次apply），只有第一笔会触发`resync` + `scheduleCommit`，后面的只置位
3. 如果没有被排过走`resync() + scheduleCommit()`:`resync()`内部还有节流——同一调用方在`VSyncTracker::kPredictorThreshold`（默认200ms）内只真正对齐一次，避免每笔事务都去摸硬件Vsync；`scheduleCommit()`最终落到`mScheduler->scheduleFrame()`，在SF自己的`MessageQueue`上arm一次VSync预约。“提前多久、几点醒”是第10步的主题，这里只记住一句：**事务到达SF时，唤醒SF的那个定时器已经被排好了**
4. 如果被排过了，且`frameHint == FrameHint::kActive`，还是要走`mScheduler->resetAllIdleTimers()`，保证这帧不会被Display显示器的idle状态拖住

### 9.5 BufferTX计数

带buffer的transaction在`SurfaceFlinger::setTransactionState()`（Binder线程）里给`BufferTX`计数+1：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
if (resolvedState.state.hasBufferChanges() && resolvedState.state.hasValidBuffer() &&
    resolvedState.state.surface) {
    ...
    mBufferCountTracker.increment(resolvedState.layerId);
}
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h
void increment(uint32_t layerId) {
    std::lock_guard<std::mutex> lock(mLock);
    auto it = mCounterByLayerId.find(layerId);
    if (it != mCounterByLayerId.end()) {
        auto [name, pendingBuffers] = it->second;
        int32_t count = ++(*pendingBuffers);
        SFTRACE_INT(name.c_str(), count);
    } else {
        ALOGW("Layer ID not found! %d", layerId);
    }
}
```

name为`const std::string mBlastTransactionName{"BufferTX - " + mName};`，`BufferTX - <layerName>`在Binder线程收到带buffer的transaction时+1，在`Layer::gatherBufferInfo()`latch新buffer时-1，或在`Layer::releasePreviousBuffer()`丢弃旧buffer时-1，反映“已到达SF但未latch的buffer数”。

### 9.6 这一步结束时

- buffer所有权已通过`Transaction`交给SurfaceFlinger的待处理事务队列。
- SF还没应用事务，`Layer::mDrawingState.buffer`未更新。
- `BufferTX`已+1。
- 这次事务已经在`setTransactionFlags()`里置位`mTransactionFlags`的`eTransactionFlushNeeded`位（没有单独的`mTransactionFlushNeeded`成员），并在需要时`scheduleCommit()`排好了一次`Vsync-sf`唤醒

到第9步结束，App这一侧的活已经干完

## 第10步：Vsync-sf定时器唤醒SurfaceFlinger

第9步停在这样一个状态：`apply()`已在App进程返回，带buffer的`Transaction`进了SF的待处理事务队列，`BufferTX - <layerName>`+1，而`Layer::mDrawingState.buffer`还是上一帧的buffer。

### 10.1 排唤醒：从scheduleCommit到scheduleFrame

SF侧没有单独的调度器线程：`class Scheduler : public IEventThreadCallback, android::impl::MessageQueue`,`Scheduler`本身就是“消息队列 + VSync预约器”

`setTransactionState()`末尾的`setTransactionFlags()`如果发现这一帧还没排过调度，于是`resync()`（受200ms节流）把预测器对齐到硬件vsync，再由`scheduleCommit()`落到`mScheduler->scheduleFrame()`。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2508
void SurfaceFlinger::scheduleCommit(FrameHint hint, Duration workDurationSlack) {
    if (hint == FrameHint::kActive) {
        mScheduler->resetAllIdleTimers();
    }
    mPowerAdvisor->notifyDisplayUpdateImminentAndCpuReset();
    mScheduler->scheduleFrame(workDurationSlack);
}
```

`mScheduler->scheduleFrame()`就是`MessageQueue::scheduleFrame()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:191
void MessageQueue::scheduleFrame(Duration workDurationSlack) {
    SFTRACE_CALL();
    std::lock_guard lock(mVsync.mutex);
    const auto workDuration = Duration(mVsync.workDuration.get() - workDurationSlack);
    mVsync.scheduledFrameTimeOpt = mVsync.registration->schedule({.workDuration = workDuration.ns(),
                                           .readyDuration = 0,
                                           .lastVsync = mVsync.lastCallbackTime.ns()});
}
```

`mVsync.registration`是`VSyncCallbackRegistration`，在`onNewVsyncScheduleLocked()`里创建，绑定的回调是`vsyncCallback`，名字叫`"sf"`。
SF中`VSyncCallbackRegistration`的来历参考[mScheduler->initVsync](../Scheduler相关/Scheduler.md#initVsync)；

### 10.2 workDuration、readyDuration与VsyncModulator

App和SF的唤醒点都由同一个`VsyncConfig`派生，只是取了不同字段。`SurfaceFlinger::initScheduler()`里：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:5216
const auto configs = mScheduler->getCurrentVsyncConfigs();
mScheduler->createEventThread(mFrameTimeline->getTokenManager(),
                              /* workDuration */ configs.late.appWorkDuration,
                              /* readyDuration */ configs.late.sfWorkDuration);
mScheduler->initVsync(*mFrameTimeline->getTokenManager(), configs.late.sfWorkDuration);
```

配置变化时`Scheduler::setVsyncConfig()`把同一组值分发给两边：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp:802
void Scheduler::setVsyncConfig(const VsyncConfig& config) {
    setDuration(/* workDuration */ config.appWorkDuration,
                /* readyDuration */ config.sfWorkDuration);
    setDuration(config.sfWorkDuration);
}
```

带两个参数的`setDuration()`给App的`EventThread`，带一个参数的给SF自己的`MessageQueue`。于是同一条Vsync时间轴被切成两段：

```text
        App 唤醒                                App deadline / SF 唤醒              vsync
  ----------|------------------------------------------|----------------------------|---->
   vsync-(appWorkDuration+sfWorkDuration)         vsync - sfWorkDuration           vsync
```

- App：`workDuration = appWorkDuration`、`readyDuration = sfWorkDuration`，即“提前`appWorkDuration + sfWorkDuration`醒，可用工作窗口是`appWorkDuration`，最晚在`vsync - sfWorkDuration`前把buffer交出去”。
- SF：`workDuration = sfWorkDuration`、`readyDuration = 0`，即“提前`sfWorkDuration`醒，deadline就是vsync本身”。

这解释了第6步的dequeue为什么不能拖：App错过自己的readyTime，SF这一帧就来不及latch新buffer，只能拿旧buffer或跳过合成。

`VsyncConfig`有三套配置（`early`/`earlyGpu`/`late`），由`VsyncModulator`选：

- `early`：有`eEarlyWakeupStart`请求、正在切刷新率、或刚提交过事务后的一小段窗口；
- `earlyGpu`：最近几帧用过GPU合成（`onDisplayRefresh(usedGpuComposition)`）；
- `late`：默认。

`VsyncModulator::updateVsyncConfigLocked()`切换时会写`SFTRACE_INT("Vsync-Early")`、`"Vsync-EarlyGpu"`、`"Vsync-Late"`，Perfetto里能直接看到当前用哪套。`VsyncConfiguration`把`sfOffset`/`appOffset`换算成`sfWorkDuration`/`appWorkDuration`，所以底层配置其实还是“相位偏移”，只是调度器用“工作窗口”表达。

### 10.3 选哪一次vsync、几点醒：VSyncDispatchTimerQueue

`schedule()`最终落到`VSyncDispatchTimerQueueEntry::schedule()`

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp:90
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
    // 新算出的VSync比已armed的VSync晚超过mMinVsyncDistance（3ms），说明会跳过原本要响的那次
    bool const wouldSkipAVsyncTarget =
            mArmedInfo && (nextVsyncTime > (mArmedInfo->mActualVsyncTime + mMinVsyncDistance));
    // 新的唤醒时间比已armed的唤醒时间晚超过mMinVsyncDistance，唤醒也会被跳过
    bool const wouldSkipAWakeup =
            mArmedInfo && ((nextWakeupTime > (mArmedInfo->mActualWakeupTime + mMinVsyncDistance)));
    SFTRACE_FORMAT_INSTANT("%s: wouldSkipAVsyncTarget=%d wouldSkipAWakeup=%d", mName.c_str(),
                           wouldSkipAVsyncTarget, wouldSkipAWakeup);
    // 会跳过就退回已armed的那次vsync，宁可早响也不漏响
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

- `nextAnticipatedVSyncTimeFrom(max(lastVsync, now + workDuration + readyDuration))`取的是从现在起还留得出工作时间的最早那次vsync
- `nextWakeupTime = nextVsyncTime - workDuration - readyDuration`。SF的`readyDuration`是0，所以SF在目标vsync前`workDuration`醒来；对于APP来说，`workDuration`是自己的工作时间，`readyDuration`通常是SF的`workDuration`，这个结论在`Scheduler`中创建APP的`EventThread`的时候可以得出（见10.2）
- `wouldSkipAVsyncTarget`/`wouldSkipAWakeup`比较的是`mMinVsyncDistance`，不是“一个周期”。这个值由`VsyncSchedule`传入`kSnapToSameVsyncWithin = 3ms`：新目标比已armed的vsync/wakeup晚3ms以上就沿用已armed的那次，避免把定时器一路推到下一个vsync。
- `adjustVsyncIfNeeded()`防止同一帧被响应两次：如果这次vsync已经派发过，或者新目标离上次派发不足`mMinVsyncDistance`，就顺延到下一次。

vsync时间本身来自`VsyncTracker`的预测：它吃硬件vsync采样`addResyncSample()`来校准，也吃第13步的present fence的`Scheduler::addPresentFence()`。预测准不准，取决于上一帧真实上屏的时刻。

到点后`timerCallback()`在锁内把到期entry收进`invocations`，再在锁外逐个回调：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
for (auto const& invocation : invocations) {
    ftl::Concat trace(ftl::truncated<5>(invocation.callback->name()));
    SFTRACE_FORMAT("%s: %s", __func__, trace.c_str());
    invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                                  invocation.deadlineTimestamp);
}
```

### 10.4 到点：vsyncCallback

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:59
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    SFTRACE_CALL();
    mVsync.value = (mVsync.value + 1) % 2;   // VSYNC-sf翻转
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

`vsyncCallback`跑在`VSyncDispatchTimerQueue`的定时器线程上，不是SF主线程；它只做四件事：

1. `mVsync.value`取反，Perfetto里`VSYNC-sf`翻转（`TracedOrdinal<int> value = {"VSYNC-sf", 0}`）。
2. 记下`lastCallbackTime`，并`mVsync.scheduledFrameTimeOpt.reset()`——这次(wakeup, vsync)预约已兑现。注意`getScheduledFrameResult()`不会因此返回空：它先看`mHandler->isFramePending()`，`dispatchFrame()`刚把pending置true，所以在此之前仍能拿到`expectedVsyncTime`；两者都没有才返回`nullopt`。
3. 生成`VsyncId`：`TokenManager::generateTokenForPredictions()`取一个自增token号（`mCurrentToken++`），把`targetWakeupTime`、`readyTime`、`vsyncTime`按token号存进`mPredictions`表。同一帧在App侧和SF侧各有一个VsyncId，都由这个`TokenManager`生成、只是取各自的预测，FrameTimeline用它们把surface frame和display frame对上。
4. `dispatchFrame()`把帧消息投给SF主线程。commit和composite都在主线程做，定时器线程到这里就交棒了。

### 10.5 主线程醒来：dispatchFrame到onFrameSignal

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:32
void MessageQueue::Handler::dispatchFrame(VsyncId vsyncId, TimePoint expectedVsyncTime) {
    if (!mFramePending.exchange(true)) {
        mVsyncId = vsyncId;
        mExpectedVsyncTime = expectedVsyncTime;
        mQueue.mLooper->sendMessage(sp<MessageHandler>::fromExisting(this), Message());
    }
}

// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:44
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

`mFramePending`保证帧消息不堆叠，但它的窗口比“主线程还没处理完上一帧”更窄：`dispatchFrame()`在消息入队前`exchange(true)`，`handleMessage()`一进来就`store(false)`，所以真正的丢弃只发生在“上一条帧消息已经`sendMessage()`、还没被Looper取出”的这段时间。主线程正在`onFrameSignal()`里commit/composite时新到的vsync会照常再入一条消息排队，不会覆盖`mVsyncId`。代价是极端情况下SF会连做两帧，好处是不会为错过的vsync补跑一串。

`handleMessage()`由`mLooper`在SF主线程上调用，就是`Scheduler::run()`里`waitMessage()`的`pollOnce(-1)`返回处。`onFrameSignal()`在`MessageQueue`里是纯虚函数（`MessageQueue.h:115`），由`Scheduler`实现（`Scheduler.h:396`）。

到这里，SF主线程手上只有三样东西：`vsyncId`、`expectedVsyncTime`，以及“这一帧要干活”的隐含信号。它进入`Scheduler::onFrameSignal()`（`Scheduler.cpp:411`）——第11步正是从那里开始：`beginFrame()`接收这三个值，由`FrameTargeter`算出本帧的`expectedPresentTime`，然后`commit()`去latch第9步送来的那块buffer。

和App侧第2步对照：那边是`VSYNC-app`翻转后进`Choreographer.doFrame()`，这边是`VSYNC-sf`翻转后进`onFrameSignal()`；两边共享同一个`VsyncTracker`，唤醒点分别由`appWorkDuration`和`sfWorkDuration`决定，一个覆盖UI线程的录制，一个覆盖commit+composite。

## 第11步：commit与latch

### 11.1 Scheduler::onFrameSignal

第10步最后一行的`onFrameSignal()`就是这里。它先把这个`vsync-sf`翻译成本帧的时间参数，再调用`commit()`：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp:411
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
    ...
    pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);
    {
        FrameTargets targets;
        targets.try_emplace(pacesetterPtr->displayId, &pacesetterPtr->targeterPtr->target());
        ...
        if (!compositor.commit(pacesetterPtr->displayId, targets)) {
            compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
            mSchedulerCallback.onCommitNotComposited();
            return;
        }
    }
    ...
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

`beginFrame()`的入参是`frameBeginTime`、`vsyncId`、`expectedVsyncTime`，`FrameTargeter::beginFrame()`再据此算出`expectedPresentTime`：目标vsync不早于`frameBeginTime`时直接用它，否则取`frameBeginTime`之后的下一个vsync deadline（若`sfWorkDuration`超过一个vsync周期，再顺延一个周期）。`pacesetter`是主显示器，其他显示器的Vsync以它为基准；17上还有`follower_display_backpressure_combined`逻辑，非lockstep的follower有未完成的present fence时，这一帧只把该follower从targets里排除（`continue`），pacesetter照常commit/composite，而不是整帧跳过。`commit()`返回false时直接返回，不进入composite。

### 11.2 SurfaceFlinger::commit

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2996
bool SurfaceFlinger::commit(PhysicalDisplayId pacesetterId,
                            const scheduler::FrameTargets& frameTargets) EXCLUDES(mStateLock) {
    const scheduler::FrameTarget* pacesetterFrameTargetPtr = frameTargets.get(pacesetterId)->get();
    const VsyncId vsyncId = pacesetterFrameTargetPtr->vsyncId();
    ...
    if (pacesetterFrameTargetPtr->wouldBackpressureHwc()) {
        if (mBackpressureGpuComposition || pacesetterFrameTargetPtr->didMissHwcFrame()) {
            mScheduler->getVsyncSchedule()->getTracker().onFrameMissed(
                    pacesetterFrameTargetPtr->expectedPresentTime());
            const Duration slack = TimePoint::now() - pacesetterFrameTargetPtr->frameBeginTime();
            scheduleCommit(FrameHint::kNone, slack);
            return false;
        }
    }

    bool mustComposite = mMustComposite.exchange(false);
    {
        const bool flushTransactions = clearTransactionFlags(eTransactionFlushNeeded);
        bool transactionsAreEmpty = false;
        mustComposite |=
                updateLayerSnapshots(vsyncId, pacesetterFrameTargetPtr->frameBeginTime().ns(),
                                     pacesetterFrameTargetPtr->expectedPresentTime().ns(),
                                     flushTransactions, transactionsAreEmpty);
        ...
    }
    {
        MODE_TRANSITION_LOCK();
        mScheduler->chooseRefreshRateForContent(&mLayerHierarchyBuilder.getHierarchy(),
                                                updateAttachedChoreographer);
        initiateDisplayModeChanges();
    }
    return mustComposite && CC_LIKELY(mBootStage != BootStage::BOOTLOADER);
}
```

commit做四件事：

1. 检查display集合与mode set。display被移除时先`processDisplayChangesLocked()`再重试；mode set未完成则`scheduleFrame()`等下一次。
2. 检查HWC背压。上一帧HWC未present完且当前需要GPU合成时，按slack重新调度commit。
3. `updateLayerSnapshots()`（3138）应用事务并latch buffer。`applyTransactionState()`在应用事务时就把`mDrawingState.previousBuffer`/`mDrawingState.buffer`换成了新buffer；`latchBufferImpl()`只把`mDrawingState`里的buffer、fence、frameNumber提交到`mBufferInfo`。
4. `chooseRefreshRateForContent()`（3179）按可见layer的frame rate投票、touch boost、游戏模式等选刷新率。必须在latch之后，因为要用刚latch进来的状态。

`mustComposite`为false时commit返回false，SF不进入composite，屏幕内容没有变化；另外`mBootStage == BootStage::BOOTLOADER`阶段也返回false，此时SF还没有正式接管屏幕。

### 11.3 latchBufferImpl

`latchBufferImpl()`（1506）先处理sideband stream，再检查新buffer的acquire fence是否signal，未signal就`onLayerUpdate()`后返回false（本帧这个layer不参与合成），signal了才`latchBufferStatsAndHandles()`、`gatherBufferInfo()`：

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp:1506
bool Layer::latchBufferImpl(bool& recomputeVisibleRegions, nsecs_t latchTime,
                            nsecs_t expectedPresentTime, bool bgColorOnly) {
    bool refreshRequired = latchSidebandStream(recomputeVisibleRegions);
    if (refreshRequired) {
        return refreshRequired;
    }
    // 新buffer的acquire fence还没signal，等下一次
    if (mDrawingState.buffer && !fenceHasSignaled()) {
        SFTRACE_NAME("!fenceHasSignaled()");
        mFlinger->onLayerUpdate();
        return false;
    }
    latchBufferStatsAndHandles(latchTime, expectedPresentTime, bgColorOnly);
    ...
    gatherBufferInfo();
    ...
}
```

真正更新计数和上一帧回调id的是`gatherBufferInfo()`（1299）：

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp:1299
void Layer::gatherBufferInfo() {
    mPreviousReleaseCallbackId = {getLatchedBufferId(), mBufferInfo.mFrameNumber};
    mPreviousReleaseBufferEndpoint = mBufferInfo.mReleaseBufferEndpoint;
    ...
    // 只有确实换了buffer才减BufferTX
    if ((!mBufferInfo.mBuffer || !mDrawingState.buffer->hasSameBuffer(*mBufferInfo.mBuffer))) {
        decrementPendingBufferCount();
    }
    mBufferInfo.mBuffer = mDrawingState.buffer;
    mBufferInfo.mFence = mDrawingState.acquireFence;
    mBufferInfo.mFrameNumber = mDrawingState.frameNumber;
    ...
    mBufferInfo.mFrameLatencyNeeded = true;
    mBufferInfo.mFenceTime = std::make_shared<FenceTime>(mDrawingState.acquireFence);
    ...
}
```

`decrementPendingBufferCount()`（1410）使`BufferTX`-1。latch之后buffer从“已到达未使用”变成“即将被合成”，`mPreviousReleaseCallbackId`记下上一帧buffer，供第13步释放。注意`mDrawingState.buffer`不是在这里换的：`Layer::applyTransactionState()`应用事务时就已经`mDrawingState.previousBuffer = std::move(mDrawingState.buffer); mDrawingState.buffer = std::move(buffer);`，latch只是把它提交到`mBufferInfo`。

## 第12步：composite与HWC协商

### 12.1 composite的入口

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:3330
CompositeResultsPerDisplay SurfaceFlinger::composite(
        PhysicalDisplayId pacesetterId, const scheduler::FrameTargeters& frameTargeters) {
    const scheduler::FrameTarget& pacesetterTarget =
            frameTargeters.get(pacesetterId)->get()->target();
    const VsyncId vsyncId = pacesetterTarget.vsyncId();
    ...
    compositionengine::CompositionRefreshArgs refreshArgs;
    ...
    addOutputsToRefreshArgs(pacesetterId, refreshArgs, frameTargeters);
    const auto layers = mLayerSnapshotBuilder.hasMergedSnapshots()
            ? copyMergedSnapshots(refreshArgs)
            : addLayerSnapshotsToCompositionArgs(refreshArgs, kCursorOnly);
    prepareLayersForComposition(refreshArgs, kCursorOnly, layers);
    ...
    mCompositionEngine->present(refreshArgs);
    ...
    onCompositionPresented(pacesetterId, frameTargeters, presentTime);
}
```

`prepareLayersForComposition()`（3661）对每个layer调用`onPreComposition()`并`attachReleaseFenceFutureToLayer()`，把release fence的future提前挂到layer上。`mCompositionEngine->present()`（3448）遍历每个output调用`Output::present()`。

### 12.2 Output::present

```cpp
// frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp:450
ftl::Future<std::monostate> Output::present(
        const compositionengine::CompositionRefreshArgs& refreshArgs) {
    ...
    updateColorProfile(refreshArgs);
    updateCompositionState(refreshArgs);
    planComposition();
    writeCompositionState(refreshArgs);
    setColorTransform(refreshArgs);
    beginFrame();
    if (canPredictCompositionStrategy(refreshArgs)) {
        result = prepareFrameAsync();
    } else {
        prepareFrame();
    }
    finishFrame(std::move(result));
    ...
    if (mOffloadPresent) {
        future = presentFrameAndReleaseLayersAsync(flushEvenWhenDisabled);
    } else {
        presentFrameAndReleaseLayers(flushEvenWhenDisabled);
    }
}
```

`planComposition()`把可见layer组织成composition plan；`prepareFrame()`/`prepareFrameAsync()`调用`chooseCompositionStrategy()`做HWC协商；`finishFrame()`在需要client composition时用RenderEngine把layer画到client target；`presentFrameAndReleaseLayers()`调HWC present并收集fence。17上还有`mOffloadPresent`：整帧present可以投给`HwcAsyncWorker`异步执行。

### 12.3 HWC决定谁走硬件合成

```cpp
// Output.cpp:1177
void Output::prepareFrame() {
    auto& outputState = editState();
    if (!outputState.isEnabled) {
        return;
    }
    std::optional<android::HWComposer::DeviceRequestedChanges> changes;
    bool success = chooseCompositionStrategy(&changes);
    if (success && changes.has_value()) {
        ...
    }
    resetCompositionStrategy();
    outputState.previousDeviceRequestedChanges = changes;
    outputState.previousDeviceRequestedSuccess = success;
    if (success) {
        applyCompositionStrategy(changes);
    }
    finishPrepareFrame();
}
```

`chooseCompositionStrategy()`（`Display.cpp:248`）先取`halDisplayId`，算出`anyLayersRequireClientComposition()`，再把`earliestPresentTime`、`expectedPresentTime`、`frameInterval`交给`HWComposer::getDeviceCompositionChanges()`；HAL出错时返回false。

`HWComposer::getDeviceCompositionChanges()`（537）调用HWC HAL的`validate()`或`presentOrValidate()`：

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:537
status_t HWComposer::getDeviceCompositionChanges(
        HalDisplayId displayId, bool frameUsesClientComposition,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime,
        nsecs_t expectedPresentTime, Fps frameInterval,
        std::optional<android::HWComposer::DeviceRequestedChanges>* outChanges) {
    ...
    ...
    if (canSkipValidate) {
        error = hwcDisplay->presentOrValidate(expectedPresentTime, frameInterval.getPeriodNsecs(),
                                              &numTypes, &numRequests, &outPresentFence, &state);
        if (state == 1) { // Present Succeeded.
            return NO_ERROR;
        }
        // Present失败但Validate已经跑过
    } else {
        error = hwcDisplay->validate(expectedPresentTime, frameInterval.getPeriodNsecs(), &numTypes,
                                     &numRequests);
    }
    error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
    error = hwcDisplay->getRequests(&displayRequests, &layerRequests);
    ...
    return NO_ERROR;
}
```

HWC的`validate()`返回后，SF从HWC读四类信息：

- `getChangedCompositionTypes()`：哪些layer要改合成方式。最关键的是把某个layer从`DEVICE`改成`CLIENT`，即“HWC合成不了，请用GPU画到client target”。
- `getRequests()`：显示级请求（如`FLIP_CLIENT_TARGET`）和layer级请求（清除、亮度、色彩变换）。
- `getClientTargetProperty()`：client target buffer的dataspace和像素格式。
- `getRequestedLuts()`：layer级的LUT（色彩查找表）请求。

读完之后还要`acceptChanges()`，HWC才会接受这组合成决策并允许后续present；漏掉这一步，`present()`会直接失败。

`applyCompositionStrategy()`（295）把`changedTypes`、`displayRequests`、`layerRequests`、`clientTargetProperty`、`layerLuts`逐项落到layer/output状态，最后重算`usesClientComposition`与`usesDeviceComposition`。

快速路径要注意：`canSkipValidate`为true且`presentOrValidate()`返回`state == 1`（本帧已经present成功）时，`getDeviceCompositionChanges()`在`getReleaseFences()`、存好`lastPresentFence`和`validateWasSkipped = true`后就返回，上面这四类读取和`acceptChanges()`全都不执行。

`usesClientComposition`为true时，`finishFrame()`会dequeue一块client target buffer，用RenderEngine（GPU）把需要client合成的layer画进去，再把client target作为一个layer交给HWC。`usesDeviceComposition`为true时，至少部分layer由HWC直接扫描。layer能否走HWC合成涉及格式、缩放、色彩空间、叠加限制、保护内容等条件，最终由vendor的HWC HAL决定。

### 12.4 present

`presentFrameAndReleaseLayers()`（1726）：

```cpp
// Output.cpp:1726（节选）
void Output::presentFrameAndReleaseLayers(bool flushEvenWhenDisabled) {
    ...
    auto frame = presentFrame();
    for (auto* layer : getOutputLayersOrderedByZ()) {
        sp<Fence> releaseFence = Fence::NO_FENCE;
        if (auto hwcLayer = layer->getHwcLayer()) {
            if (auto f = frame.layerFences.find(hwcLayer); f != frame.layerFences.end()) {
                releaseFence = f->second;
            }
        }
        if (outputState.usesClientComposition) {
            releaseFence =
                    Fence::merge("LayerRelease", releaseFence, frame.clientTargetAcquireFence);
        }
        layer->getLayerFE().setReleaseFence(releaseFence);
    }
}
```

17上`Output::presentFrame()`（1828）只负责在`usesClientComposition`时取`mRenderSurface->getClientTargetAcquireFence()`。真正的HWC提交在`Display::presentFrame()`（416）：

```cpp
// Display.cpp:416（节选）
compositionengine::Output::FrameFences Display::presentFrame() {
    auto fences = impl::Output::presentFrame();
    hwc.presentAndGetReleaseFences(*halDisplayIdOpt, getState().earliestPresentTime);
    fences.presentFence = hwc.getPresentFence(*halDisplayIdOpt);
    for (const auto* layer : getOutputLayersOrderedByZ()) {
        auto hwcLayer = layer->getHwcLayer();
        fences.layerFences.emplace(hwcLayer, hwc.getLayerReleaseFence(*halDisplayIdOpt, hwcLayer));
    }
    hwc.clearReleaseFences(*halDisplayIdOpt);
    return fences;
}
```

`HWComposer::presentAndGetReleaseFences()`（658）里，如果**本帧**的validate被跳过（即`presentOrValidate()`在validate阶段就已经present成功），它会显式`executeCommands()`并直接复用已经存好的present fence；否则先`hwcDisplay->present(&displayData.lastPresentFence)`，再`getReleaseFences()`取回每个HWC layer的release fence。提交结束，但像素还没稳定，present fence继续跟踪。

## 第13步：present fence、release fence与buffer回收

### 13.1 present fence的归属

```cpp
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:637
sp<Fence> HWComposer::getPresentFence(HalDisplayId displayId) const {
    RETURN_IF_INVALID_DISPLAY(displayId, Fence::NO_FENCE);
    return mDisplayData.at(displayId).lastPresentFence;
}
```

present fence在HWC完成本次扫描输出后signal，回答“这一帧何时真正呈现在屏幕上”。SF在`onCompositionPresented()`（3775）处理它：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:3775（节选）
void SurfaceFlinger::onCompositionPresented(PhysicalDisplayId pacesetterId,
                                            const scheduler::FrameTargeters& frameTargeters,
                                            nsecs_t presentStartTime) {
    for (const auto& [id, targeter] : frameTargeters) {
        auto presentFence = getHwComposer().getPresentFence(id);
        if (id == pacesetterId) {
            mTransactionCallbackInvoker.addPresentFence(presentFence);
        }
        if (auto fenceTime = targeter->setPresentFence(std::move(presentFence));
            fenceTime->isValid()) {
            presentFences.try_emplace(id, std::move(fenceTime));
        }
        ...
    }
    ...
    mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
    ...
    for (auto&& [id, presentFence] : presentFences) {
        mScheduler->addPresentFence(id, std::move(presentFence));
    }
}
```

`addPresentFence()`把fence挂到事务完成回调上，App的`TransactionCompletedListener`收到的`presentFence`就是它；`targeter->setPresentFence()`把fence交给Scheduler的FrameTarget，用于Vsync预测；`mScheduler->addPresentFence()`（3992）把fence喂给VsyncTracker。`mFrameTimeline->setSfPresent()`（3822）同时上报FrameTimeline；`usesClientComposition`的display还会额外把`clientTargetAcquireFence`作为gpuCompositionDoneFence一起上报，用来区分“GPU画完”和“真正上屏”。

### 13.2 release fence的来源

release fence回答“这块buffer何时可被生产者安全重写”，来源有三种：

- HWC为每个layer返回的release fence：

```cpp
// HWComposer.cpp:647
sp<Fence> HWComposer::getLayerReleaseFence(HalDisplayId displayId, HWC2::Layer* layer) const {
    RETURN_IF_INVALID_DISPLAY(displayId, Fence::NO_FENCE);
    const auto& displayFences = mDisplayData.at(displayId).releaseFences;
    auto fence = displayFences.find(layer);
    if (fence == displayFences.end()) {
        return Fence::NO_FENCE;
    }
    return fence->second;
}
```

注意：HWC的release fence描述“上一帧的buffer何时可复用”，但它是在**本帧**present时才返回的。`presentFrameAndReleaseLayers()`拿到的是本次present的layer release fence，作用对象是上一帧的buffer——`Output.cpp`那里的注释写得很直白：“The layer buffer from the previous frame ... is released by HWC only when the release fence from this frame ... is signaled”。

- client composition的acquire fence。本帧用了GPU合成时，client target buffer的acquire fence要并进layer的release fence（`Fence::merge("LayerRelease", releaseFence, frame.clientTargetAcquireFence)`）。因为layer的buffer被读进了client target，必须等client target的GPU工作完成。

- present fence兜底。不在`getOutputLayersOrderedByZ()`里的已释放layer只能用present fence。

`setReleaseFence()`（`LayerFE.cpp:438`）兑现promise：

```cpp
// LayerFE.cpp:438（节选）
if (mReleaseFencePromiseStatus == ReleaseFencePromiseStatus::FULFILLED) {
    return;
}
...
mReleaseFence.set_value(releaseFence);
mReleaseFencePromiseStatus = ReleaseFencePromiseStatus::FULFILLED;
```

这个future在composite之前就通过`attachReleaseFenceFutureToLayer()`（`SurfaceFlinger.cpp:8341`）→`Layer::prepareReleaseCallbacks()`（787）挂到`callbackHandle.fenceMerger`上。present时`setReleaseFence()`兑现promise，后面的`fenceMerger.waitAndGetFence()`才能取出release fence。

### 13.3 从SurfaceFlinger回到App

`onCompositionPresented()`对每个有queued frame的layer调用`releasePendingBuffer()`（`SurfaceFlinger.cpp:3864`与3872）：

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp:814
void Layer::releasePendingBuffer(nsecs_t dequeueReadyTime) {
    for (auto& handle : mDrawingState.callbackHandles) {
        handle.bufferReleaseChannel = mBufferReleaseChannel;
        handle.dequeueReadyTime = dequeueReadyTime;
        handle.currentMaxAcquiredBufferCount =
                mFlinger->getMaxAcquiredBufferCountForCurrentRefreshRate(mOwnerUid);
    }
    ...
    mFlinger->getTransactionCallbackInvoker().addCallbackHandles(
            std::move(mDrawingState.callbackHandles));
}
```

`callbackHandles`里的`previousReleaseCallbackId`指向上一帧buffer，release fence由`prepareReleaseCallbacks()`（787）通过`ch->fenceMerger.addFuture(futureFenceResult)`挂进来。`TransactionCallbackInvoker::addCallbackHandle()`（91）用`handle.fenceMerger.waitAndGetFence()`取出fence，收进`mBufferReleases`（emplace在145行附近）；`sendCallbacks()`（158）对每一项调用`channel->writeReleaseFence(callbackId, fence, currentMaxAcquiredBufferCount)`，失败时打印`writeReleaseFence failed`，最后清空队列。

### 13.4 App端releaseBufferCallback与计数-1

`BufferReleaseChannel`是App与SF之间的fd通道。`BLASTBufferQueue::createBufferQueue()`在1358行`open()`它并创建`BufferReleaseReader`。

producer端通过单独的`updateBufferReleaseProducer()`（1453）注册到`SurfaceControl`：

```cpp
// BLASTBufferQueue.cpp:1453
SurfaceComposerClient::Transaction()
        .setApplyToken(mApplyToken)
        .setBufferReleaseChannel(mSurfaceControl, mBufferReleaseProducer)
        .apply(false /* synchronous */, true /* oneWay */);
```

注释说明这是故意拆成单独事务的：SELinux可能阻止本进程把channel fd发给SurfaceFlinger，单独发送可以避免整个buffer事务被丢弃。

`BLASTBufferQueue`用`BufferReleaseReader`通过epoll监听channel，`makeReleaseBufferCallbackThunk()`（467）返回的lambda在收到回调后先处理一条，再把channel里积压的全部排空：

```cpp
// BLASTBufferQueue.cpp:467（节选）
bbq->releaseBufferCallback(id, releaseFence, currentMaxAcquiredBufferCount);
bbq->drainBufferReleaseConsumer();
```

`drainBufferReleaseConsumer()`（1463）循环调用`mBufferReleaseReader->readNonBlocking()`，每读到一项就`releaseBufferCallback()`，读到非OK为止。

```cpp
// BLASTBufferQueue.cpp:490（节选）
const auto it = mSubmitted.find(id);
const bool isEGL = it != mSubmitted.end() && it->second.mApi == NATIVE_WINDOW_API_EGL;
if (currentMaxAcquiredBufferCount) {
    mCurrentMaxAcquiredBufferCount = *currentMaxAcquiredBufferCount;
}
const uint32_t numPendingBuffersToHold =
        isEGL ? std::max(0, mMaxAcquiredBuffers - (int32_t)mCurrentMaxAcquiredBufferCount) : 0;
mPendingRelease.emplace_back(ReleasedBuffer{id, releaseFence});
...
while (mPendingRelease.size() > numPendingBuffersToHold) {
    const auto releasedBuffer = mPendingRelease.front();
    mPendingRelease.pop_front();
    releaseBuffer(releasedBuffer.callbackId, releasedBuffer.releaseFence);
    if (mSyncedFrameNumbers.empty()) {
        acquireNextBufferLocked(std::nullopt);
    }
}
ATRACE_INT(mQueuedBufferTrace.c_str(), mNumFrameAvailable + mNumAcquired - mPendingRelease.size());
```

`mSubmitted.find(id)`找回当初acquire的`BufferItem`。`numPendingBuffersToHold`在低刷新率下故意多留一块buffer，避免下一帧dequeue时无buffer可用反而增加延迟；只有EGL客户端有暂留逻辑。

`releaseBuffer()`（540）先`mSubmitted.find(callbackId)`，找不到就直接返回；找到则`mNumAcquired--`、`mBufferItemConsumer->releaseBuffer(it->second, releaseFence)`、从`mSubmitted`和`mSyncedFrameNumbers`里删掉对应项。

`mNumAcquired--`加`releaseBuffer()`把slot交还BufferQueue。对BLAST来说，`BBQBufferQueueCore::notifyBufferReleased()`被重写为`mBufferReleaseReader->interruptBlockingRead()`：写eventfd打断第6步正在`readBlocking()`里`epoll_wait()`的dequeue，而不是去notify `mDequeueCondition`。dequeue被唤醒后重新检查空闲slot，没有才继续等。`QueuedBuffer`计数在`ATRACE_INT`处下降，systrace上表现为向下的台阶。

### 13.5 present fence与release fence的区别

|对比项|present fence|release fence|
|---|---|---|
|谁产生|HWC在present时产生|HWC为每个layer产生，或GPU合成完成fence，或present fence兜底|
|回答什么|这一帧何时真正显示在屏幕上|这块buffer何时可被生产者重新写入|
|谁等待|App的`TransactionCompletedListener`（`TransactionStats.presentFence`）、Scheduler的VsyncTracker、FrameTimeline|App的`dequeueBuffer()`（BLAST下阻塞在BufferReleaseChannel）、BufferQueueProducer|
|signal条件|显示控制器完成这一帧的扫描输出|该layer的buffer不再被显示管线或GPU读取|
|代码位置|`HWComposer::getPresentFence()`、`FrameTimeline::setSfPresent()`|`HWComposer::getLayerReleaseFence()`、`LayerFE::setReleaseFence()`、`releaseBufferCallback()`|

present fence signal了不代表buffer立刻可复用：如果HWC还在读它（例如双缓冲下的当前front buffer），release fence会晚于present fence。反过来，release fence signal也不代表这一帧已上屏，只说明buffer可被覆盖。BLAST的`QueuedBuffer`计数跟随release，不跟随present。

## 收束：一帧的完整时序

一帧可以按四个阶段看：

- App侧（步骤1-5）：`scheduleTraversals()`申请`VSYNC-app`（硬件加速下`invalidate()`只整窗标脏，不合并矩形），`doFrame()`依次跑INPUT、ANIMATION、INSETS_ANIMATION、TRAVERSAL、COMMIT；TRAVERSAL里完成measure/layout并把DisplayList录进RenderNode，`syncAndDrawFrame()`把树交给RenderThread，`prepareTree()`同步属性后`unblockUiThread()`放行UI线程（只看`prepareTextures`，不看本帧是否会被丢）。
- GPU侧（步骤6-8）：EGL驱动在`eglBeginFrame`阶段发起dequeue，经`ReliableSurface`拦截后进`Surface::dequeueBuffer`（可能阻塞在BufferReleaseChannel）；Skia把RenderNode树翻译成GPU命令并`FlushAndSubmit()`，`eglSwapBuffersWithDamageKHR()`把buffer连同GPU完成fence交给`Surface::queueBuffer()`，`QueuedBuffer`+1。`queueBuffer()`可能等上一帧的GPU fence（EGL限流，最多两个buffer在途）。
- 提交侧（步骤9-11）：`onFrameAvailable()`→`acquireNextBufferLocked()`→`Transaction::setBuffer().apply()`，事务到达SF，`BufferTX`+1；`VSYNC-sf`回调触发`Scheduler::onFrameSignal()`，`commit()`把事务应用进`mDrawingState`并由`latchBufferImpl()`提交到`mBufferInfo`、选刷新率，`gatherBufferInfo()`使`BufferTX`-1。
- 合成与回收（步骤12-13）：`composite()`→`Output::present()`→HWC`validate()`决定device/client合成→`presentFrame()`；`onCompositionPresented()`把present fence喂给FrameTimeline和Scheduler，`releasePendingBuffer()`把上一帧release fence写入`BufferReleaseChannel`，App端`releaseBufferCallback()`收buffer，`QueuedBuffer`-1。

present fence决定“这一帧何时算显示完成”，release fence决定“这块buffer何时能再次被App使用”，两者共同构成buffer从dequeue到release的完整生命周期。

## 附录A：执行体速查

|执行体|所在进程|这一帧里干什么|
|---|---|---|
|App主线程|App|`doFrame()`各阶段、measure/layout、把DisplayList录进RenderNode、调`syncAndDrawFrame()`后阻塞在`postAndWait()`|
|RenderThread|App|`prepareTree()`同步树状态、dequeue、Skia回放DisplayList、`FlushAndSubmit()`、`swapBuffers()`/`queueBuffer()`；BLAST的`onFrameAvailable()`/`acquireNextBufferLocked()`也在这条线程上同步执行|
|App Binder线程|App|`TransactionCompletedListener`收SF的事务完成回调、present fence|
|SF Binder线程|surfaceflinger|`setTransactionState()`：sanitize、解析事务、`BufferTX`+1、入无锁队列、`setTransactionFlags()`|
|VSyncDispatch定时器线程|surfaceflinger|`EventThread::onVsync()`（app）、`MessageQueue::vsyncCallback()`（sf）；`timerCallback()`在锁外回调entry|
|EventThread线程|surfaceflinger|`dispatchEvent()`把Vsync事件写进各连接的`BitTube`|
|SF主线程|surfaceflinger|`onFrameSignal()`→`commit()`→`composite()`→`present()`→`onCompositionPresented()`，以及`releasePendingBuffer()`|
|HWC/composer|vendor HAL|`validate()`/`presentOrValidate()`/`present()`，产出present fence与每层release fence；`mOffloadPresent`时present可投给`HwcAsyncWorker`|
|显示控制器|硬件|按vsync扫描输出，present fence在此时signal|

## 附录B：丢帧与背压速查

App侧：

- `syncFrameState()`把丢帧原因写进`SkippedFrameReason`：`NoOutputTarget`、`ContextIsStopped`、`AlreadyDrawn`（同一vsync画过）、`NothingToDraw`（backdrop不可绘制）、`NoBuffer`（`reserveNext()`失败）。`run()`据`skippedFrameReason`决定跳过`draw()`，只`flushAndSubmit()`+`waitOnFences()`；同时置`SyncResult::FrameDropped`回传给UI线程。注意A17里`DISABLE_BUFFER_PREFETCH = true`，`reserveNext()`直接返回OK，`NoBuffer`实际不可达。
- jitter超过一个帧间隔：对齐帧时间并打印`Skipped N frames!`；帧时间倒退：`scheduleVsyncLocked()`等下一帧；`mFPSDivisor > 1`：按分频跳过。
- buffer stuffing：dequeue阻塞超过半帧→`Choreographer.onWaitForBufferRelease()`置位→下一次`doFrame()`返回`DELAY_FRAME`主动推迟一帧；恢复期用`OFFSET`回拨动画时间轴。

系统侧：

- SF的HWC背压：`FrameTarget::wouldBackpressureHwc()`为真且需要GPU合成或错过HWC帧时，`VsyncTracker::onFrameMissed()`后带slack重新`scheduleCommit()`，本帧`commit()`返回false。
- follower display背压：`follower_display_backpressure_combined`开启时，非lockstep的follower有未完成present fence就把该follower从本帧targets排除，pacesetter照常合成。
- `commit()`返回false（`mustComposite == false`或`BOOTLOADER`阶段）：不进入composite，屏幕内容不变。
- BLAST容量：`mNumAcquired >= mMaxAcquiredBuffers + 2`时`acquireNextBufferLocked()`返回`NO_BUFFER_AVAILABLE`；普通路径不等待，帧留在影子队列等release回调再acquire；sync事务路径才在`mCallbackCV`上等。
- EGL限流：`BufferQueueProducer::queueBuffer()`默认等上一帧GPU fence，最多两个buffer在途，第三个要等。
- `ReliableSurface`兜底：真实dequeue失败时返回1x1 scratch buffer，本帧画进假buffer并在queue时丢弃，`getAndClearError()`可读到错误码。

## 附录C：Perfetto/systrace观察点

|计数/切片|产生位置|含义|
|---|---|---|
|`VSYNC-app`|`EventThread.cpp`的`mVsyncTracer`|App侧vsync翻转，每次`EventThread::onVsync()`一次|
|`VSYNC-sf`|`MessageQueue.h`的`TracedOrdinal<int> value`|SF侧vsync翻转，`MessageQueue::vsyncCallback()`里取反|
|`Choreographer#doFrame <vsyncId>`|`Choreographer.doFrame()`|一帧App回调的开始；resynced时会额外带`resynced to ...`|
|`input`/`animation`/`insets_animation`/`traversal`/`commit`|`Choreographer.doCallbacks()`|各阶段回调耗时|
|`DrawFrames <vsyncId>`、`syncFrameState`、`prepareTree`|HWUI的ATRACE|UI→RT同步与树状态准备|
|`dequeueBuffer - <surface>`、`queueBuffer`|`Surface.cpp`|buffer进出生产者|
|`QueuedBuffer - VRIBLAST#N`|`BLASTBufferQueue`的`mQueuedBufferTrace`|App已queue、尚未被SF释放的buffer数（`mNumFrameAvailable + mNumAcquired - mPendingRelease.size()`）|
|`BufferTX - <layer>`|`BufferCountTracker`|已到达SF但未latch的buffer数|
|`PendingRelease`|`BLASTBufferQueue::releaseBufferCallbackLocked()`|被暂留、还没真正release的buffer数|
|`Vsync-Early`/`Vsync-EarlyGpu`/`Vsync-Late`|`VsyncModulator::updateVsyncConfigLocked()`|当前生效的Vsync配置类型|
|`EarlyWakeupRequests`|`VsyncModulator`|在途的early wakeup请求数|

观察顺序建议：先看`VSYNC-app`→`Choreographer#doFrame`是否对齐，再看`DrawFrame`/`dequeueBuffer`有没有长阻塞，然后看`QueueBuffer`与`BufferTX`是否在`VSYNC-sf`前到位，最后用present fence确认上屏时刻。
