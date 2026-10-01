## 1 涉及的类和接口概览

- Choreographer(Java) 
   持有FrameDisplayEventReceiver，通过它请求Vsync并在回调中驱动doFrame

- FrameDisplayEventReceiver(Java) 
   DisplayEventReceiver的子类,专门用于请求下一帧,收到Vsync后，把doFrame作为一个消息post到消息队列

- DisplayEventReceiver(Java) 
  Java层对native层DisplayEventReceiver的封装，内部持有native对象的指针（mReceiverPtr）
  
- NativeDisplayEventReceiver(CPP)  
  Java层DisplayEventReceiver在native侧的对应实现
  
- DisplayEventReceiver(CPP) 
   APP端与SurfaceFlinger中EventThread通信的封装，内部持有sp<IDisplayEventConnection>，通过它向SurfaceFlinger请求Vsync并接收事件，管理一个用于接收事件的BitTube(socket pair),SurfaceFlinger端把Vsync数据写进来，APP端通过它读出来
  
- IDisplayEventConnection(CPP)
   Binder接口，是App进程与SurfaceFlinger进程之间Vsync的通信接口
  
- EventThreadConnection(CPP)  
  IDisplayEventConnection的服务端实现(BnDisplayEventConnection子类)
  
- EventThread(CPP)  
  SurfaceFlinger中真正管理Vsync的对象，维护所有EventThreadConnection，决定Vsync何时发生,分发给哪些连接
  
- SurfaceFlinger(CPP)  
  进程的核心,暴露ISurfaceComposer的Binder接口

## 2 与SurfaceFlinger通信的核心成员：mDisplayEventReceiver

### 2.1 在Choreographer初始化时对成员对象mDisplayEventReceiver初始化创建
  
```java
mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper, layerHandle) : null;
```

**最终是由父类`super(looper, /* eventRegistration */ 0, layerHandle)`进行初始化**

```java
// frameworks/base/core/java/android/view/DisplayEventReceiver.java
public DisplayEventReceiver(Looper looper, int eventRegistration,
    long layerHandle) {
    ...
    // 持有DisplayEventReceiver在CPP的指针
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this),
            new WeakReference<VsyncEventData>(mVsyncEventData),
            mMessageQueue,
            eventRegistration, layerHandle);
    ...
}
```

### 2.2 由nativeInit落到JNI层进行初始化

```cpp
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak, jobject vsyncEventDataWeak, jobject messageQueueObj, jint eventRegistration, jlong layerHandle) {
        sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
        ...
        // 在这里面真正初始化了DisplayEventReceiver(CPP)
        sp<NativeDisplayEventReceiver> receiver =
                new NativeDisplayEventReceiver(env, receiverWeak, vsyncEventDataWeak, messageQueue,
                                            eventRegistration, layerHandle);
        status_t status = receiver->initialize();
        if (status) {
            String8 message;
            message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
            jniThrowRuntimeException(env, message.c_str());
            return 0;
        }
        ...
        receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
        return reinterpret_cast<jlong>(receiver.get());
}
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env, jobject receiverWeak,
                                                    jobject vsyncEventDataWeak,
                                                    const sp<MessageQueue>& messageQueue,
                                                    jint eventRegistration, jlong layerHandle):
        // 这里是因为NativeDisplayEventReceiver继承自DisplayEventDispatcher
        // 先执行父类初始化父类构造函数初始化列表
        DisplayEventDispatcher(messageQueue->getLooper(),
        static_cast<gui::ISurfaceComposer::EventRegistration>(eventRegistration),
        layerHandle != 0 ? sp<IBinder>::fromExisting(reinterpret_cast<IBinder*>(layerHandle)) :nullptr),
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mVsyncEventDataWeakGlobal(env->NewGlobalRef(vsyncEventDataWeak)),
        mMessageQueue(messageQueue) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}

// frameworks/native/libs/gui/DisplayEventDispatcher.cpp
// 父类构造函数中关注两个成员：   
// sp<Looper> mLooper; 
// DisplayEventReceiver mReceiver;
DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper,
                                            EventRegistrationFlags eventRegistration,
                                            const sp<IBinder>& layerHandle):
        mLooper(looper),
        // 这里真正构造了DisplayEventReceiver(CPP)
        mReceiver(eventRegistration, layerHandle),
        mWaitingForVsync(false),
        mLastVsyncCount(0),
        mLastScheduleVsyncTime(0) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}


// frameworks/native/libs/gui/DisplayEventReceiver.cpp
DisplayEventReceiver::DisplayEventReceiver(EventRegistrationFlags eventRegistration, const sp<IBinder>& layerHandle) {
    // 获取SurfaceFlinger的Binder代理
    sp<gui::ISurfaceComposer> sf(ComposerServiceAIDL::getComposerService());
    if (sf != nullptr) {
        mEventConnection = nullptr;
        mSurfaceflingerAlive = IInterface::asBinder(sf)->isBinderAlive();
        // 头文件中声明了 sp<IDisplayEventConnection> mEventConnection
        // 重点在这里 通过Binder传入mEventConnection向SurfaceFlinger请求获取IDisplayEventConnection，最终得到EventThreadConnection的一个Proxy
        binder::Status status = sf->createDisplayEventConnection(
            static_cast<gui::ISurfaceComposer::EventRegistration>(eventRegistration.get()), 
            layerHandle, &mEventConnection);
        // mEventConnection已经可用了
        if (status.isOk() && mEventConnection != nullptr) {
            mDataChannel = std::make_unique<gui::BitTube>();
            // 这里通过Binder调用传入mDataChannel，从SurfaceFlinger获取了BitTube接收端
            // 之后就通过BitTube发送VSYNC请求，不用走Binder
            status = mEventConnection->stealReceiveChannel(mDataChannel.get());
            if (!status.isOk()) {
                ALOGE("stealReceiveChannel failed: %s", status.toString8().c_str());
                mInitError = status.transactionError();
                mDataChannel.reset();
                mEventConnection.clear();
            }
        } else {
            ALOGE("DisplayEventConnection creation failed: status=%s", status.toString8().c_str());
            mInitError = status.transactionError();
        }
    }
}
```

### 2.3 获取DisplayEventConnection

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
binder::Status SurfaceComposerAIDL::createDisplayEventConnection(
        EventRegistration eventRegistration, const sp<IBinder>& layerHandle,
        sp<IDisplayEventConnection>* outConnection) {
    // 真正本地获取DisplayEventConnection
    sp<IDisplayEventConnection> conn = mFlinger->createDisplayEventConnection(eventRegistration, layerHandle);
    if (conn == nullptr) {
        *outConnection = nullptr;
        return binderStatusFromStatusT(BAD_VALUE);
    } else {
        *outConnection = conn;
        return binder::Status::ok();
    }
}

// 返回Binder接口IDisplayEventConnection（跨进程给APP用）
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        EventRegistrationFlags eventRegistration, const sp<IBinder>& layerHandle) {
    return mScheduler->createDisplayEventConnection(eventRegistration, layerHandle);
}

// frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
// 返回Binder接口IDisplayEventConnection（跨进程给APP用）
sp<IDisplayEventConnection> Scheduler::createDisplayEventConnection(
        EventRegistrationFlags eventRegistration, const sp<IBinder>& layerHandle) {
    // 交给EventThread
    const auto connection = mEventThread->createEventConnection(eventRegistration);
    // SF内部用于跨线程/进程安全引用layer
    const auto layerId = static_cast<int32_t>(LayerHandle::getLayerId(layerHandle));
    if (layerId != static_cast<int32_t>(UNASSIGNED_LAYER_ID)) {
        mSchedulerCallback.onChoreographerAttached();
        std::scoped_lock lock(mChoreographerLock);
        const auto [iter, emplaced] = mAttachedChoreographers.emplace(layerId, AttachedChoreographers{Fps(), {connection}});
        if (!emplaced) {
            iter->second.connections.emplace(connection);
            connection->frameRate = iter->second.frameRate;
        }
    }
    return connection;
}

// frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
sp<EventThreadConnection> EventThread::createEventConnection(EventRegistrationFlags eventRegistration) const {
    const auto& ipc = IPCThreadState::self();
    // 这里正式创建EventConnection
    // eventRegistration=0 建立一个用于按需请求VSync的通道，不订阅任何被动推送的显示事件
    auto connection = sp<EventThreadConnection>::make(const_cast<EventThread*>(this), ipc->getCallingUid(), ipc->getCallingPid(), eventRegistration);
    if (!FlagManager::getInstance().disable_sched_fifo_sf_sched()) {
        const int policy = SCHED_FIFO;
        connection->setMinSchedulerPolicy(policy, sched_get_priority_min(policy));
    }
    return connection;
}
```

至此`Choreographer`的成员`mDisplayEventReceiver`(一个继承自DisplayEventReceiver的FrameDisplayEventReceiver对象)初始化完毕，其内部的`NativeDisplayEventReceiver`(Java端持有的CPP指针)对象也初始化完毕，`NativeDisplayEventReceiver`的成员`mReceiver`(CPP端的DisplayEventReceiver)也初始化完毕

## 3 一次Vsync请求

### 3.1 APP invalidate()或requestLayout()后 VRI中开始请求Vsync
```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
void scheduleTraversals() {
    checkThreadCompat();
    // 同帧只调用一次 防止其它view调用
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 添加同步屏障，调度Traversal之后post的任何同步消息都在Traversal执行之后运行
        postTraversalBarrier();
        // 通过Choreographer注册VSync回调
        mChoreographer.postVsyncCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalCallback);
        // 通知渲染器
        notifyRendererOfFramePending();
        // 保持屏幕常亮
        pokeDrawLockIfNeeded();
    }
}
```

### 3.2 交给mChoreographer进行VSYNC请求
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
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        ...
        if (isRunningOnLooperThreadLocked()) {
            scheduleVsyncLocked(); 
            ...
        }
        ...
    }           
}
private void scheduleVsyncLocked() {
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#scheduleVsyncLocked");
        // 从这里开始就交给mDisplayEventReceiver真正的去请求VSYNC信号了
        mDisplayEventReceiver.scheduleVsync();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

### 3.3 交给mDisplayEventReceiver开始请求Vsync

```java
// frameworks/base/core/java/android/view/DisplayEventReceiver.java
// 这是实际执行的是父类实现的方法，FrameDisplayEventReceiver只实现了onVsync()和run()
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        // 进入JNI调用，转发到NativeDisplayEventReceiver
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

### 3.4 交给native的DisplayEventReceiver去处理

从这里由DisplayEventReceiver(Java)转到NativeDisplayEventReceiver(Cpp)

```cpp
// frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    //  这个方法子类NativeDisplayEventReceiver没有重写
    //  在的父类DisplayEventDispatcher中有实现
    status_t status = receiver->scheduleVsync();
    ...
}
// frameworks/native/libs/gui/DisplayEventDispatcher.cpp
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ...
        // mReceiver是自己的成员变量，一个DisplayEventReceiver(CPP)的实例
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
// frameworks/native/libs/gui/DisplayEventReceiver.cpp
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != nullptr) {
        // 在头文件DisplayEventReceiver.h中有声明sp<IDisplayEventConnection> mEventConnection:
        // 实际是一个BpDisplayEventConnection,在构造时通过binder调用请求了SurfaceFlinger返回了EventThreadConnection的代理对象
        // 这里是Cpp层面的binder调用 落到BnDisplayEventConnection::onTransact()
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return mInitError;
}
```

### 3.4 SurfaceFlinger下EventThreadConnection标记请求Vsync

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
    // TODO 这里面的具体请求机制待补充
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
