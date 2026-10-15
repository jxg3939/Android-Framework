## Choreographer的构成

Choreographer是主线程上的帧调度器,线程内单例(ThreadLocal持有,`Choreographer.getInstance()`返回当前线程实例)。核心成员:

| 重要成员| 作用|
| ---------- | -------- |
| FrameHandler | 主线程Handler,把VSYNC回调与调度动作投递回主线程执行,保证回调都在主线程运行 |
| FrameDisplayEventReceiver | DisplayEventReceiver子类,与SurfaceFlinger之间接收VSYNC的通道 |
| CallbackQueue数组 | 按回调类型分队列,五类回调各自排队,同类型按时间、跨类型按固定顺序执行  |
| mFrameScheduled | 布尔标记, 避免重复登记VSYNC,同一帧只登记一次 |

```java
// frameworks/base/core/java/android/view/Choreographer.java
// 这里有两个实例：sThreadInstance 是普通 APP 用的，sSfThreadInstance 是 SF 相关进程用的

// Thread local storage for the choreographer.
private static final ThreadLocal<Choreographer> sThreadInstance = new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        // Choreographer创建时就绑定looper，主要是handler要绑定looper
        Choreographer choreographer = new Choreographer(looper);
        // 绑定的主线程的looper，代表自己时主线程的Choreographer
        if (looper == Looper.getMainLooper()) {
            sMainInstance = choreographer;
        }
        return choreographer;
    }
};

private static final ThreadLocal<Choreographer> sSfThreadInstance = new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        return new Choreographer(looper);
    }
};

private Choreographer(Looper looper) {
    this(looper, /* layerHandle */ 0L);
}

// layerHandle 默认为 0，接收标准的 vsync 信号
private Choreographer(Looper looper, long layerHandle) {
    mLooper = looper;
    mHandler = new FrameHandler(looper);
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper, layerHandle) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;
    mFrameIntervalNanos = (long) (1000000000 / getRefreshRate());
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
    // b/68769804: For low FPS experiments.
    setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

> `FrameHandler` 和 `FrameDisplayEventReceiver` 都是内部类

```java
private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }
    // FrameHandler提供handleMessage兜底实现
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME: // 开始下一帧绘制
                doFrame(System.nanoTime(), 0, new DisplayEventReceiver.VsyncEventData());
                break;
            case MSG_DO_SCHEDULE_VSYNC: // 请求Vsync 最终还是mDisplayEventReceiver.scheduleVsync()
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK: // 处理callback
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}

private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;
    private final VsyncEventData mLastVsyncEventData = new VsyncEventData();
    // 这个FrameDisplayEventReceiver后面很重要，是Choreographer与SurfaceFlinger通信的关键
    FrameDisplayEventReceiver(Looper looper, long layerHandle) {
        super(looper, /* eventRegistration */ 0, layerHandle);
    }
    // 这个就是Choreographer在收到VSYNC-app信号后的回调
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame, VsyncEventData vsyncEventData) {
        ...
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        mLastVsyncEventData.copyFrom(vsyncEventData);
        // 把自己作为Runable给发送了
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        ...
    }
    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }
    // 父类DisplayEventReceiver中的默认实现
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
}
```