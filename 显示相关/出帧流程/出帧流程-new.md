# 出帧流程

本文按13个步骤展开一帧从App主线程到屏幕上屏的完整路径。类名、方法名、行号取自`android-17.0.0_r1`（<https://cs.android.com/android/platform/superproject/+/android-17.0.0_r1>），抓取时间2026-11，行号会随提交变化，引用时以方法名为准。

## 总览

|步骤|发生位置|核心入口|
|---|---|---|
|1|App主线程|`ViewRootImpl.scheduleTraversals()`|
|2|App主线程|`Choreographer.doFrame()`|
|3|App主线程|`ViewRootImpl.performTraversals()`→`performDraw()`|
|4|App主线程→RenderThread|`HardwareRenderer.syncAndDrawFrame()`→`DrawFrameTask::drawFrame()`|
|5|RenderThread|`DrawFrameTask::run()`→`syncFrameState()`|
|6|RenderThread|`SkiaOpenGLPipeline::getFrame()`→`eglBeginFrame`→`dequeueBuffer`|
|7|RenderThread+GPU|`SkiaOpenGLPipeline::draw()`→`FlushAndSubmit()`|
|8|RenderThread|`eglSwapBuffersWithDamageKHR()`→`queueBuffer()`|
|9|App进程（BLAST）|`BLASTBufferQueue::acquireNextBufferLocked()`→`Transaction::apply()`|
|10|SurfaceFlinger|定时器→`MessageQueue::vsyncCallback()`|
|11|SurfaceFlinger|`Scheduler::onFrameSignal()`→`SurfaceFlinger::commit()`|
|12|SurfaceFlinger+HWC+GPU|`SurfaceFlinger::composite()`→`Output::present()`|
|13|SurfaceFlinger→App|`onCompositionPresented()`→`releaseBufferCallback()`|

时间轴上：App在`vsync-app`之前完成调度，`vsync-app`到来后`doFrame`；SurfaceFlinger在`vsync-app`之后、`vsync-sf`之前拿到App提交的buffer，在`vsync-sf`回调中commit并composite；HWC在下一个刷新周期扫描上屏，present fence此时signal。

## 第1步：窗口调度下一帧

![alt text](image.png)

### 1.1 何时触发调度

进入调度的入口有多个，最终都落到`ViewRootImpl.scheduleTraversals()`：

- `invalidate()`：内容失效。`View.invalidate()`沿父链调用`ViewGroup.invalidateChild()`，到达`ViewRootImpl.invalidateChildInParent()`，最终进入`invalidateRectOnScreen()`。
- `requestLayout()`：测量或位置可能失效。`View.requestLayout()`沿父链传播到`ViewRootImpl.requestLayout()`。
- 输入事件：`scheduleConsumeBatchedInput()`把输入消费挂到`CALLBACK_INPUT`，消费后若状态变化再由View自己调`invalidate()`或`requestLayout()`。
- 动画：属性动画在`CALLBACK_ANIMATION`阶段改属性，改完调用`invalidate()`。
- Insets动画：`CALLBACK_INSETS_ANIMATION`阶段更新Insets，随后触发traversal。
- 窗口变化：`relayoutWindow()`返回新尺寸或Surface，`performTraversals()`内部置位`mFullRedrawNeeded`。

`invalidateChildInParent()`并不遍历View树，它只把脏区域合并进`mDirty`后调`scheduleTraversals()`

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
2. `postTraversalBarrier()`插入同步屏障，只拦同步消息，不拦异步消息。Vsync消息在`onVsync()`里`setAsynchronous(true)`，能越过屏障；屏障保证队列里已有的同步消息不会插到traversal前面。
3. `postVsyncCallback(CALLBACK_TRAVERSAL, mTraversalCallback)`把`mTraversalCallback`对象挂进Choreographer对应的callback队列。
4. `notifyRendererOfFramePending()`通知RenderThread即将出帧；`pokeDrawLockIfNeeded()`在息屏/Doze场景短暂持有绘制锁。

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

## 第2步：Vsync到达与doFrame

![alt text](image-1.png)

### 2.1 从postVsyncCallback到申请Vsync

`postCallbackDelayedInternal()`把回调按`dueTime`插进`mCallbackQueues[callbackType]`；`dueTime <= now`时立刻`scheduleFrameLocked(now)`，否则投一条异步`MSG_DO_SCHEDULE_CALLBACK`。

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

SurfaceFlinger进程根据不同Android版本有不同的EventThread，最新的A17中，只保留了服务App的`VSYNC-app`。Vsync由`VsyncDispatchTimerQueue`按时回调`EventThread`的`onVsync`，EventThread的`onVsync()`把Vsync事件塞进`mPendingEvents`并唤醒自己去`dispatchEvent()`：

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

`doFrame()`开始回调前还有几件处理：

1. jitter对齐。若`System.nanoTime() - frameTimeNanos`超过一个帧间隔，说明主线程错过Vsync太久，Choreographer把`frameTimeNanos`对齐到当前时间并减去`jitterNanos % frameIntervalNanos`，跳帧过多时打印`Skipped N frames!`。
2. 帧时间倒退检查。若`frameTimeNanos < mLastFrameTimeNanos`，`scheduleVsyncLocked()`后返回。
3. buffer stuffing恢复。`updateBufferStuffingState()`可能返回`DELAY_FRAME`，主动推迟一帧以降低排队buffer数；返回`OFFSET`时会把动画时间轴整体减一个帧间隔。

## 第3步：performTraversals与DisplayList录制

![alt text](image-2.png)

### 3.1 三个阶段的取舍

`performTraversals()`不是每次都完整执行measure、layout、draw：

- `performMeasure()`：窗口尺寸、配置变化、`mLayoutRequested`、Insets变化时执行。根View的MeasureSpec由窗口尺寸和`WindowManager.LayoutParams`算出。
- `performLayout()`：测量完成、`layoutRequested`为true、窗口位置变化时执行。
- `performDraw()`：窗口可见、Surface有效、确实有脏区域时执行；`mFullRedrawNeeded`为true时整窗重绘。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
if (mViewMeasureDeferred) {
    performMeasure(MeasureSpec.makeMeasureSpec(frame.width(), MeasureSpec.EXACTLY),
                   MeasureSpec.makeMeasureSpec(frame.height(), MeasureSpec.EXACTLY));
}
// frameworks/base/core/java/android/view/ViewRootImpl.java
performLayout(lp, mWidth, mHeight);
// frameworks/base/core/java/android/view/ViewRootImpl.java
if (!performDraw(mActiveSurfaceSyncGroup)) { ... }
```

`performMeasure()`调根View的`measure()`，`performLayout()`调根View的`layout()`，二者内部递归整棵树。`performDraw()`则按是否硬件加速走两条完全不同的路径。

### 3.2 硬件加速绘制入口

`performDraw(@Nullable SurfaceSyncGroup)`：

```java
private boolean performDraw(@Nullable SurfaceSyncGroup surfaceSyncGroup) {
    ...
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
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        ...
    } else {
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                scalingRequired, dirty)) {
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
// 录制根View的DisplayList
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);
    ...
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

`View.updateDisplayListIfDirty()`：

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
        renderNode.clearStretch();
        final RecordingCanvas canvas = renderNode.beginRecording(width, height);
        try {
            if (getLayerType() == LAYER_TYPE_SOFTWARE) {
                ...
            } else {
                computeScroll();
                canvas.translate(-mScrollX, -mScrollY);
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    // 这里递归绘制子View
                    dispatchDraw(canvas);
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
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

RenderNode树引用的实际收益是“只重录脏子树”：后代`invalidate()`时只清祖先的`PFLAG_DRAWING_CACHE_VALID`、不置`PFLAG_INVALIDATED`，根节点`updateDisplayListIfDirty()`只走`dispatchGetDisplayList()`往下刷脏节点，父的DisplayList不重录；因为每个View的RenderNode身份不变，旧引用继续指向该子节点最新的DisplayList。所以根RenderNode的DisplayList里是子RenderNode的引用，整棵树没有被压成扁平指令流。

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

`mNativeProxy`是`private final long`，在构造函数里由`nCreateProxy(!mOpaque, mRootNode.mNativeRenderNode)`赋值；JNI侧`android_view_ThreadedRenderer_createProxy()`执行`new RenderProxy(...)`后把指针按`jlong`返回

native方法声明与注册项（`android_graphics_HardwareRenderer.cpp`）：

```java
private static native int nSyncAndDrawFrame(long nativeProxy, long[] frameInfo, int size);
```

```cpp
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

`postAndWait()`把`DrawFrameTask::run`投递到RenderThread队列，然后在条件变量上等待。这个等待不是等整帧画完：`run()`在`syncFrameState()`完成并确认可绘制后调用`unblockUiThread()`，提前放行UI线程；能否提前放行取决于`syncFrameState()`返回的`info.prepareTextures`，纹理都准备好才允许。

`drawFrame()`返回的`mSyncResult`是位掩码，含`FrameDropped`、`UIRedrawRequired`、`LostSurfaceRewardIfFound`、`ContextIsStopped`等标志，`ThreadedRenderer.draw()`据此决定是否`invalidate()`或强制relayout。

### 4.3 同步边界的准确含义

- 边界前：DisplayList写入、RenderNode属性修改都在UI线程完成。
- 边界处：UI线程`postAndWait()`，RenderThread开始`run()`。
- 同步点：`syncFrameState()`完成，RenderThread确认树可绘制，`unblockUiThread()`唤醒UI线程。
- 同步点后：UI线程可以修改View树，RenderThread继续`context->draw()`。

所以UI线程与RenderThread的同步边界是等到树状态已被RenderThread接走。

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
    if (CC_LIKELY(canDrawThisFrame)) {
        // 走到第6步的Draw()
        context->draw(solelyTextureViewUpdates);
    } else {
        if (GrDirectContext* grContext = mRenderThread->getGrContext()) {
            grContext->flushAndSubmit();
        }
        context->waitOnFences();
    }
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

`prepareTree()`逐个`node->prepareTree(info)`，把DisplayList属性同步到RenderThread可见状态：变换矩阵、裁剪、alpha、动画属性、DisplayList版本。动画属性在`mAnimationContext->runRemainingAnimations()`里推进。

`MODE_FULL`用于窗口主RenderNode，完整同步；`MODE_RT_ONLY`用于其他node，只同步RenderThread需要的最小状态

### 5.3 syncFrameState()完成

`syncFrameState()`返回后，`run()`按`canDrawThisFrame`决定：true则`CanvasContext::draw()`进入第6步；false则不画，直接`flushAndSubmit()`并`waitOnFences()`，这里的`waitOnFences()`是回收本帧已提交的内部异步工作，与后续Buffer的fence无关

同步点之后UI线程已可继续处理消息。RenderThread拿到的是属性快照+DisplayList引用。这个快照机制依赖于RenderNode内的`mStagingDisplayList`和`mDisplayList`双缓存，`mStagingDisplayList`用于UI线程录制，`mDisplayList`用于RenderThread绘制

## 第6步：从BLAST BufferQueue申请Buffer

![alt text](image-5.png)

### 6.1 draw()到getFrame()

```cpp
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    ...
    // 获取本帧脏区域
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);
    ...
    // 判断是否跳过本帧绘制
    const auto skippedFrameReason = [&]() -> std::optional<SkippedFrameReason> {
        if (!Properties::isDrawingEnabled() || !mDrawingEnabled) {
            return SkippedFrameReason::DrawingOff;
        }
        if (dirty.isEmpty() && Properties::skipEmptyFrames && !surfaceRequiresRedraw()) {
            return SkippedFrameReason::NothingToDraw;
        }
        return std::nullopt;
    }();
    if (skippedFrameReason) {
        // 跳过绘制，通知回调本帧没有产生 Buffer
        ...
        waitOnFences();
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, false);
        }
        mFrameCommitCallbacks.clear();
        return;
    }
    ScopedActiveContext activeContext(this);
    // 初始化帧信息，获取当前 Frame
    mCurrentFrameInfo->set(FrameInfoIndex::FrameInterval) =
            mRenderThread.timeLord().frameIntervalNanos();
    mCurrentFrameInfo->markIssueDrawCommandsStart();
    Frame frame = getFrame();
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
    uint64_t frameCompleteNr = getFrameNumber();
    waitOnFences();
    ...
    // 执行Buffer Swap，将绘制结果提交到Surface
    bool requireSwap = false;
    bool didSwap = mRenderPipeline->swapBuffers(
            frame, drawResult, windowDirty, mCurrentFrameInfo, &requireSwap);
    ...
    // 处理Swap结果
    if (requireSwap) {
        ...
        if (error == TIMED_OUT) {
            mRenderThread.postFrameCallback(this);
        } else if (error != OK || !didSwap) {
            setSurface(nullptr);
        }
        ..
    }
    mCurrentFrameInfo->markSwapBuffersCompleted();
    // 通知帧提交回调
    if (didSwap) {
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, true);
        }
        mFrameCommitCallbacks.clear();
    }
    ...
}
```

`CanvasContext::draw()`在算完脏区域后调用`getFrame()`；`getFrame()`若有`mHardwareBuffer`就直接返回逻辑尺寸，否则转给`mRenderPipeline->getFrame()`。参考SKia渲染管线`SkiaOpenGLPipeline`，它的`getFrame()`只把`mEglSurface`交给`mEglManager.beginFrame()`。

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

`eglBeginFrame()`由EGL实现落到`ANativeWindow_dequeueBuffer()`

```cpp
// frameworks/native/libs/nativewindow/ANativeWindow.cpp
int ANativeWindow_dequeueBuffer(ANativeWindow* window, ANativeWindowBuffer** buffer, int* fenceFd) {
    return window->dequeueBuffer(window, buffer, fenceFd);
}
```

```text
window->dequeueBuffer(window, &buf, &fence)   // ANativeWindow_dequeueBuffer
        ->Surface::dequeueBuffer(sp<GraphicBuffer>*, int*)
         -> IGraphicBufferProducer::dequeueBuffer  这里Binder调用还在本进程内
           ->BBQBufferQueueProducer::dequeueBuffer  未实现，调用BufferQueueProducer的dequeueBuffer
            ->BufferQueueProducer::dequeueBuffer
```

### 6.2 Surface::dequeueBuffer

```cpp
// frameworks/native/libs/gui/Surface.cpp
status_t Surface::dequeueBuffer(sp<GraphicBuffer>* buffer, sp<Fence>* outFence) {
    if (buffer == nullptr || outFence == nullptr) {
        return BAD_VALUE;
    }
    sp<GraphicBuffer> tmpBuffer;
    int fd = -1;
    status_t res = dequeueBuffer(&tmpBuffer, &fd);
    if (res != NO_ERROR) {
        SURF_LOGV("dequeueBuffer() returned %d", res);
        return res;
    }
    *buffer = tmpBuffer;
    *outFence = sp<Fence>::make(fd);
    return res;
}

内部转发

int Surface::dequeueBuffer(sp<GraphicBuffer>* buffer, int* fenceFd) {
    {
        std::scoped_lock _dl(mDebugMutex);
        ATRACE_FORMAT("dequeueBuffer - %s", mDebugName.c_str());
    }
    ...
    int buf = -1;
    sp<Fence> fence;
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &mBufferAge,
                                                            dqInput.getTimestamps ?
                                                                    &frameTimestamps : nullptr);
    ...
}
```


```cpp
virtual status_t dequeueBuffer(int* buf, sp<Fence>* fence, uint32_t width, uint32_t height,
                                PixelFormat format, uint64_t usage, uint64_t* outBufferAge,
                                FrameEventHistoryDelta* outTimestamps) {
    ...
    status_t result = remote()->transact(DEQUEUE_BUFFER, data, &reply);
    if (result != NO_ERROR) {
        return result;
    }
    *buf = reply.readInt32();
    *fence = sp<Fence>::make();
    result = reply.read(**fence);
    return result;
}
```

`mGraphicBufferProducer`指向`BLASTBufferQueue`创建的`BufferQueue`的producer。它用的是自定义的`BBQBufferQueueProducer`，`dequeueBuffer`在`BLASTBufferQueue`未实现，调用BufferQueueProducer的dequeueBuffer


```cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    ATRACE_CALL();
    ...
    mSlots[found].mBufferState.dequeue();
    ...
    addAndGetFrameTimestamps(nullptr, outTimestamps);
}    
```

### 6.3 什么时候会在这里等待

普通路径在拿不到空闲slot时调用`waitForBufferRelease()`，`BufferQueueProducer`的默认实现阻塞在`mDequeueCondition`上：

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

`mBufferReleaseReader`读`BufferReleaseChannel`，SurfaceFlinger释放buffer时往channel写release信息（第13步）。所以“释放后才有空闲slot”在这里是显式的：dequeue阻塞在`readBlocking()`，channel有数据后`releaseBufferCallback()`把slot还给队列，`BufferQueueProducer`重新检查空闲slot，dequeue才返回。`BLASTBufferQueue::initialize()`把dequeue timeout设为最大值，默认无限等待。

### 6.4 本步拿到的两样东西

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
    canvas->androidFramework_setDeviceClipRestriction(clipRestriction);
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

命令提交后，准备swapBuffer，据绘制结果和EGL状态决定是否把本帧缓冲区提交给系统

```cpp
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    ...
    bool requireSwap = false;
    bool didSwap = mRenderPipeline->swapBuffers(
            frame, drawResult, windowDirty, mCurrentFrameInfo, &requireSwap);
    ...
    // 处理Swap结果
    if (requireSwap) {
        ...
        if (error == TIMED_OUT) {
            mRenderThread.postFrameCallback(this);
        } else if (error != OK || !didSwap) {
            setSurface(nullptr);
        }
        ..
    }
    mCurrentFrameInfo->markSwapBuffersCompleted();
    // 通知帧提交回调
    if (didSwap) {
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, true);
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

```cpp
// frameworks/native/libs/gui/Surface.cpp
int Surface::hook_queueBuffer(ANativeWindow* window,
        ANativeWindowBuffer* buffer, int fenceFd) {
    Surface* c = getSelf(window);
    ...
    return c->queueBuffer(GraphicBuffer::from(buffer), sp<Fence>::make(fenceFd));
}
```

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
    status_t err = mGraphicBufferProducer->queueBuffer(slot, igbpInput, &igbpOutput);
    ...
}
```

```cpp
// frameworks/native/libs/gui/IGraphicBufferProducer.cpp
virtual status_t queueBuffer(int buf,
        const QueueBufferInput& input, QueueBufferOutput* output) {
    ...
    status_t result = remote()->transact(QUEUE_BUFFER, data, &reply);
    ...
}
```

`queueBufferImpl()`先按buffer找出slot，再调用`mGraphicBufferProducer->queueBuffer(slot, igbpInput, &igbpOutput)`。`igbpInput`里的fence就是“GPU何时写完这块buffer”的signal fence，随buffer入队。`queueBuffer()`返回时并不等它：除非`Properties::waitForGpuCompletion`为true。消费者拿到的acquire fence会替它等。

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
    frameAvailableListener->onFrameAvailable(item);
}
```
`BufferQueueProducer::queueBuffer`中完成了对`BufferItem`的包装，`BufferItem`包含了缓冲区的句柄以及所有相关的元数据，如mGraphicBuffer, mFence, mSlot, mFrameNumber

`queueBuffer`落到`BLASTBufferQueue::onFrameAvailable()`。

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

`mNumFrameAvailable`在queue时+1，在`acquireNextBufferLocked`中-1；`mNumAcquired`在`acquireNextBufferLocked`中+1，在`releaseBuffer()`里-1

`mNumAcquired`不能超过`mMaxAcquiredBuffers + 2`。`acquireNextBufferLocked()`开头就做这两项检查，超过时返回`NO_BUFFER_AVAILABLE`，`onFrameAvailable()`在条件变量上等待

## 第9步：BLAST Consumer把buffer提交给SurfaceFlinger

### 9.1 acquire并setBuffer

```cpp
// frameworks/native/libs/gui/BLASTBufferQueue.cpp:567
status_t BLASTBufferQueue::acquireNextBufferLocked(
        const std::optional<SurfaceComposerClient::Transaction*> transaction) {
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
    status_t status =
            mBufferItemConsumer->acquireBuffer(&bufferItem, 0 /* expectedPresent */, false);
    ...
    auto buffer = bufferItem.mGraphicBuffer;
    mNumFrameAvailable--;
    mNumAcquired++;
    mLastAcquiredFrameNumber = bufferItem.mFrameNumber;
    ReleaseCallbackId releaseCallbackId(buffer->getId(), mLastAcquiredFrameNumber);
    mSubmitted.emplace_or_replace(releaseCallbackId, bufferItem);
    ...
    auto releaseBufferCallback = makeReleaseBufferCallbackThunk();
    sp<Fence> fence =
            bufferItem.mFence ? sp<Fence>::make(bufferItem.mFence->dup()) : Fence::NO_FENCE;
    t->setBuffer(mSurfaceControl, buffer, fence, bufferItem.mFrameNumber, mProducerId,
                 releaseBufferCallback, dequeueTime);
    t->setDataspace(...);
    t->setHdrMetadata(...);
    t->setSurfaceDamageRegion(...);
    t->addTransactionCompletedCallback(makeTransactionCallbackThunk(), nullptr);
    ...
    if (applyTransaction) {
        t->setApplyToken(mApplyToken).apply(false, true);
        mAppliedLastTransaction = true;
        mLastAppliedFrameNumber = bufferItem.mFrameNumber;
    } else {
        t->setBufferHasBarrier(mSurfaceControl, mLastAppliedFrameNumber);
        mAppliedLastTransaction = false;
    }
    ...
}
```

逐点：

- `acquireBuffer()`拿到`BufferItem`，其`mFence`就是生产者queue时的GPU完成fence，在SurfaceFlinger侧作为acquire fence。
- `mNumFrameAvailable--`、`mNumAcquired++`：从“可用”变成“已acquire”。
- `mSubmitted[ReleaseCallbackId(buffer->getId(), frameNumber)]`记录buffer id和frame number，SF释放时用这个映射找回`BufferItem`。
- `setBuffer()`写入buffer和acquire fence，并带上第13步要用的`releaseBufferCallback`。
- `setDataspace()`、`setHdrMetadata()`、`setSurfaceDamageRegion()`在同一个事务里提交数据空间、HDR元数据、脏区域。
- `addTransactionCompletedCallback()`注册事务完成回调，供`TransactionCompletedListener`使用。
- `setApplyToken(mApplyToken).apply(false, true)`：第二个参数`synchronous=false`、第三个参数`oneWay=true`，单向Binder调用，不等SurfaceFlinger返回。

### 9.2 apply之后：BufferTX计数

事务经`ISurfaceComposerClient`送到SurfaceFlinger，Binder线程进入`setTransactionState()`。带buffer的transaction在这里给`BufferTX`计数+1：

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:5649
if (resolvedState.state.hasBufferChanges() && resolvedState.state.hasValidBuffer() &&
    resolvedState.state.surface) {
    ...
    mBufferCountTracker.increment(resolvedState.layerId);
}
```

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h:425
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

名字在`Layer.h:556`拼出：`const std::string mBlastTransactionName{"BufferTX - " + mName};`。`BufferTX - <layerName>`在Binder线程收到带buffer的transaction时+1，在`Layer::gatherBufferInfo()`latch新buffer时-1，或在`Layer::releasePreviousBuffer()`丢弃旧buffer时-1，反映“已到达SF但未latch的buffer数”。

### 9.3 这一步结束时

- buffer所有权已通过`Transaction`交给SurfaceFlinger的待处理事务队列。
- SF还没应用事务，`Layer::mDrawingState.buffer`未更新。
- `BufferTX`已+1，`QueuedBuffer`保持“App侧未释放”的值。
- App的`mSubmitted`保留`BufferItem`，等待第13步release回调。

## 第10步：vsync-sf触发SurfaceFlinger出帧

### 10.1 定时器

SurfaceFlinger的Vsync由`VSyncDispatchTimerQueue`按预测时间提前`workDuration + readyDuration`安排定时器：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp:90
ScheduleResult VSyncDispatchTimerQueueEntry::schedule(VSyncDispatch::ScheduleTiming timing,
                                                      VSyncTracker& tracker, nsecs_t now) {
    auto nextVsyncTime =
            tracker.nextAnticipatedVSyncTimeFrom(std::max(timing.lastVsync,
                                                          now + timing.workDuration +
                                                                  timing.readyDuration),
                                                 timing.committedVsyncOpt.value_or(
                                                         timing.lastVsync));
    auto nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
}
```

到点后`timerCallback()`（334）收集到期条目，在锁外逐个调用（376）：

```cpp
// VSyncDispatchTimerQueue.cpp:376
invocation.callback->callback(invocation.vsyncTimestamp, invocation.wakeupTimestamp,
                              invocation.deadlineTimestamp);
```

`MessageQueue`在`onNewVsyncScheduleLocked()`里注册`vsyncCallback`（`MessageQueue.cpp:111`）：

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:117
mVsync.registration = std::make_unique<
        scheduler::VSyncCallbackRegistration>(std::move(dispatch),
                                              std::bind(&MessageQueue::vsyncCallback, this,
                                                        std::placeholders::_1,
                                                        std::placeholders::_2,
                                                        std::placeholders::_3),
                                              "sf");
```

### 10.2 vsyncCallback

```cpp
// MessageQueue.cpp:59
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    SFTRACE_CALL();
    mVsync.value = (mVsync.value + 1) % 2;   // VSYNC-sf翻转
    const auto expectedVsyncTime = TimePoint::fromNs(vsyncTime);
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.lastCallbackTime = expectedVsyncTime;
    }
    const auto vsyncId = VsyncId{mVsync.tokenManager->generateTokenForPredictions(
            {targetWakeupTime, readyTime, vsyncTime})};
    mHandler->dispatchFrame(vsyncId, expectedVsyncTime);
}
```

三件事：`mVsync.value`取反（systrace里`VSYNC-sf`翻转）；生成`VsyncId`（`TokenManager`把`targetWakeupTime`、`readyTime`、`vsyncTime`编码成token，沿FrameTimeline传给App，用于把App一帧、SF一帧、HWC一次present对应起来）；`dispatchFrame()`把消息投给SF主线程。

`vsyncCallback`运行在VsyncDispatchTimerQueue的定时器线程，不是SF主线程；commit/composite在主线程执行。

### 10.3 dispatchFrame到onFrameSignal

```cpp
// MessageQueue.cpp:32
void MessageQueue::Handler::dispatchFrame(VsyncId vsyncId, TimePoint expectedVsyncTime) {
    if (!mFramePending.exchange(true)) {
        mVsyncId = vsyncId;
        mExpectedVsyncTime = expectedVsyncTime;
        mQueue.mLooper->sendMessage(sp<MessageHandler>::fromExisting(this), Message());
    }
}

// MessageQueue.cpp:44
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

`mFramePending`保证帧消息不堆叠。主线程从`waitMessage()`返回后进入`onFrameSignal()`。这一步与App侧第2步对称：App侧是`VSYNC-app`写异步消息，SF侧是`VSYNC-sf`写帧消息；两者时钟源相同，但`workDuration`不同，App覆盖主线程`doFrame`，SF覆盖commit+composite。

## 第11步：commit与latch

### 11.1 Scheduler::onFrameSignal

```cpp
// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp:411
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
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
            mSchedulerCallback.onCommitNotComposited();
            return;
        }
    }
    ...
    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
    ...
}
```

`beginFrame()`算出`expectedPresentTime`、`frameBeginTime`、`vsyncId`。`pacesetter`是主显示器，其他显示器的Vsync以它为基准；17上还有`follower_display_backpressure_combined`逻辑，follower有未完成的present fence时可以跳过本帧。`commit()`返回false时不进入composite。

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
3. `updateLayerSnapshots()`（3138）应用事务并latch buffer。`applyTransactionState()`把待处理事务合并进layer状态，`latchBufferImpl()`把`mDrawingState.buffer`换成新buffer，更新`mBufferInfo`。
4. `chooseRefreshRateForContent()`（3179）按可见layer的frame rate投票、touch boost、游戏模式等选刷新率。必须在latch之后，因为要用刚latch进来的状态。

`mustComposite`为false时commit返回false，SF不进入composite，屏幕内容没有变化。

### 11.3 latchBufferImpl

`latchBufferImpl()`（1506）先检查sideband stream，再检查新buffer的acquire fence是否signal，然后调用`gatherBufferInfo()`：

```cpp
// Layer.cpp:1506
bool Layer::latchBufferImpl(bool& recomputeVisibleRegions, nsecs_t latchTime,
                            nsecs_t expectedPresentTime, bool bgColorOnly) {
    ...
    if (mDrawingState.buffer && !fenceHasSignaled()) {
        mFlinger->onLayerUpdate();
        return false;
    }
    ...
    gatherBufferInfo();
    ...
}
```

真正更新计数和上一帧回调id的是`gatherBufferInfo()`（1299）：

```cpp
// Layer.cpp:1299
void Layer::gatherBufferInfo() {
    mPreviousReleaseCallbackId = {getLatchedBufferId(), mBufferInfo.mFrameNumber};
    mPreviousReleaseBufferEndpoint = mBufferInfo.mReleaseBufferEndpoint;
    ...
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

`decrementPendingBufferCount()`（1410）使`BufferTX`-1。latch之后buffer从“已到达未使用”变成“即将被合成”，`mPreviousReleaseCallbackId`记下上一帧buffer，供第13步释放。

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

HWC的`validate()`返回后，SF从HWC读三类信息：

- `getChangedCompositionTypes()`：哪些layer要改合成方式。最关键的是把某个layer从`DEVICE`改成`CLIENT`，即“HWC合成不了，请用GPU画到client target”。
- `getRequests()`：显示级请求（如`FLIP_CLIENT_TARGET`）和layer级请求（清除、亮度、色彩变换）。
- `getClientTargetProperty()`：client target buffer的dataspace和像素格式。

`applyCompositionStrategy()`（295）把`changedTypes`、`displayRequests`、`layerRequests`、`clientTargetProperty`、`layerLuts`逐项落到layer/output状态，最后重算`usesClientComposition`与`usesDeviceComposition`。

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

`HWComposer::presentAndGetReleaseFences()`（658）里，如果上一轮validate被跳过，它会显式`executeCommands()`并直接复用已经拿到的present fence；否则先`hwcDisplay->present(&displayData.lastPresentFence)`，再`getReleaseFences()`取回每个HWC layer的release fence。提交结束，但像素还没稳定，present fence继续跟踪。

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

`addPresentFence()`把fence挂到事务完成回调上，App的`TransactionCompletedListener`收到的`presentFence`就是它；`targeter->setPresentFence()`把fence交给Scheduler的FrameTarget，用于Vsync预测；`mScheduler->addPresentFence()`（3992）把fence喂给VsyncTracker。`mFrameTimeline->setSfPresent()`（3822）同时上报FrameTimeline。

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

注意：HWC的release fence描述“上一帧的buffer何时可复用”。HWC在present这一帧时才给出上一帧buffer的release fence。所以`presentFrameAndReleaseLayers()`拿到的是上一次present的layer release fence，用它释放上一帧的buffer。

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

`BufferReleaseChannel`是App与SF之间的fd通道。`BLASTBufferQueue::createBufferQueue()`在1358行`open()`它并创建`BufferReleaseReader`（代码见6.2）。

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

`mNumAcquired--`加`releaseBuffer()`把slot交还BufferQueue，`mDequeueCondition`被signal，第6步可能正在等待的`dequeueBuffer()`因此继续。`QueuedBuffer`计数在`ATRACE_INT`处下降，systrace上表现为向下的台阶。

### 13.5 present fence与release fence的区别

|对比项|present fence|release fence|
|---|---|---|
|谁产生|HWC在present时产生|HWC为每个layer产生，或GPU合成完成fence，或present fence兜底|
|回答什么|这一帧何时真正显示在屏幕上|这块buffer何时可被生产者重新写入|
|谁等待|App的FrameCommitCallback、Scheduler的VsyncTracker、FrameTimeline|App的`dequeueBuffer()`、BufferQueueProducer|
|signal条件|显示控制器完成这一帧的扫描输出|该layer的buffer不再被显示管线或GPU读取|
|代码位置|`HWComposer::getPresentFence()`、`FrameTimeline::setSfPresent()`|`HWComposer::getLayerReleaseFence()`、`LayerFE::setReleaseFence()`、`releaseBufferCallback()`|

present fence signal了不代表buffer立刻可复用：如果HWC还在读它（例如双缓冲下的当前front buffer），release fence会晚于present fence。反过来，release fence signal也不代表这一帧已上屏，只说明buffer可被覆盖。BLAST的`QueuedBuffer`计数跟随release，不跟随present。

## 收束：一帧的完整时序

一帧可以按四个阶段看：

- App侧（步骤1-5）：`scheduleTraversals()`合并脏区域并通过`postVsyncCallback(CALLBACK_TRAVERSAL, mTraversalCallback)`申请`VSYNC-app`；`doFrame()`依次跑INPUT、ANIMATION、INSETS_ANIMATION、TRAVERSAL、COMMIT；TRAVERSAL里完成measure/layout并把DisplayList录进RenderNode，`syncAndDrawFrame()`把树交给RenderThread，`prepareTree()`同步属性后`unblockUiThread()`放行UI线程。
- GPU侧（步骤6-8）：`eglBeginFrame`从BLAST BufferQueue dequeue（可能阻塞在BufferReleaseChannel），Skia把RenderNode树翻译成GPU命令并`FlushAndSubmit()`，`eglSwapBuffersWithDamageKHR()`把buffer连同GPU完成fence交给`Surface::queueBuffer()`，`QueuedBuffer`+1。
- 提交侧（步骤9-11）：`onFrameAvailable()`→`acquireNextBufferLocked()`→`Transaction::setBuffer().apply()`，事务到达SF，`BufferTX`+1；`VSYNC-sf`回调触发`Scheduler::onFrameSignal()`，`commit()`latch buffer并选刷新率，`gatherBufferInfo()`使`BufferTX`-1。
- 合成与回收（步骤12-13）：`composite()`→`Output::present()`→HWC`validate()`决定device/client合成→`presentDisplay()`；`onCompositionPresented()`把present fence喂给FrameTimeline和Scheduler，`releasePendingBuffer()`把上一帧release fence写入`BufferReleaseChannel`，App端`releaseBufferCallback()`收buffer，`QueuedBuffer`-1。

present fence决定“这一帧何时算显示完成”，release fence决定“这块buffer何时能再次被App使用”，两者共同构成buffer从dequeue到release的完整生命周期。
