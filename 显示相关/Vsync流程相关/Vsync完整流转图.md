# 一个Vsync信号的完整流转图

本文是[Vsync信号完整流转.md](Vsync信号完整流转.md)第7节"全链路分步清单"的配图,以`android-17.0.0_r1`为准。图中节点编号与清单序号一一对应,节点内第一行是函数名,第二行是关键动作;实线表示按时间先后的调用,虚线表示闭环回边。

## 1 全链路总图

```mermaid
flowchart TB

    subgraph HWC["HWC回调线程"]
        direction TB
        n1["1 SurfaceFlinger::onComposerHalVsync<br/>校验时间戳,把HWC displayId转成PhysicalDisplayId"]
        n2["2 Scheduler::addResyncSample<br/>校验来源,按displayId找VsyncSchedule"]
        n3["3 VsyncSchedule::addResyncSample<br/>持mHwVsyncLock,仅HwVsyncState::Enabled时继续"]
        n4["4 VSyncReactor::addHwVsyncTimestamp<br/>确认周期,或把样本加入确认队列"]
        n5["5 VSyncPredictor::addVsyncTimestamp<br/>校验样本,写入环形缓冲区,必要时做线性回归"]
        n6["6 reportModelAccuracyMetric<br/>把模型误差和来源打到Perfetto"]
    end

    subgraph SF["SF主线程"]
        direction TB
        n7["7 Scheduler::onHardwareVsyncRequest<br/>需要更多样本时打开硬件Vsync,够了就关闭"]
        n8["8 SurfaceFlinger::requestHardwareVsync<br/>setVsyncEnabled(ENABLE/DISABLE),打开时查询getDisplayKnownVsyncSample"]
        n23["23 Scheduler::onFrameSignal<br/>FrameTargeter::beginFrame()→commit()→composite()"]
        n32["32 SurfaceFlinger::addPresentFence<br/>PresentFence交给Scheduler::addPresentFence"]
        n33["33 Scheduler::addPresentFence<br/>转给VsyncController::addPresentFence"]
        n34["34 VSyncReactor::addPresentFence<br/>signal的直接喂Tracker,pending的入mUnfiredFences"]
        n35["35 VSyncPredictor::addVsyncTimestamp<br/>用PresentFence时间校准模型"]
    end

    subgraph APP["应用主线程"]
        direction TB
        n9["9 Choreographer.scheduleVsyncLocked<br/>调用DisplayEventReceiver.scheduleVsync()"]
        n10["10 DisplayEventDispatcher::scheduleVsync<br/>mWaitingForVsync去重,requestNextVsync()发Binder"]
        n28["28 DisplayEventDispatcher::handleEvent<br/>读空BitTube,只保留最后一个Vsync"]
        n29["29 NativeDisplayEventReceiver::dispatchVsync<br/>JNI回调DisplayEventReceiver.dispatchVsync"]
        n30["30 Choreographer.FrameDisplayEventReceiver.onVsync<br/>按wakeupTime投递异步Message"]
        n31["31 Choreographer.doFrame<br/>输入/动画/遍历/绘制,末尾再次scheduleVsyncLocked()"]
    end

    subgraph BINDER["SF Binder线程"]
        n11["11 EventThread::requestNextVsync<br/>vsyncRequest=Single,mCondition.notify_all()"]
    end

    subgraph ET["SF EventThread线程"]
        direction TB
        n12["12 EventThread::threadMain<br/>发现vsyncRequested,状态切到State::VSync"]
        n13["13 VSyncCallbackRegistration::schedule<br/>把workDuration/readyDuration/lastVsync/committedVsync传给Dispatch"]
        n14["14 VSyncDispatchTimerQueue::scheduleLocked<br/>找到Entry;rearmImminent时先存mWorkloadUpdateInfo"]
        n15["15 VSyncDispatchTimerQueueEntry::schedule<br/>预测nextVsyncTime,算出nextReadyTime和nextWakeupTime"]
        n16["16 VSyncPredictor::nextAnticipatedVSyncTimeFrom<br/>按模型算出下一个Vsync"]
        n17["17 rearmTimerSkippingUpdateFor<br/>取所有Entry里最小的wakeupTime"]
        n18["18 Timer::alarmAt<br/>timerfd_settime(TFD_TIMER_ABSTIME)"]
        n25["25 EventThread::threadMain<br/>取mPendingEvents,shouldConsumeEvent()筛选连接"]
        n26["26 EventThread::dispatchEvent<br/>按Uid取frameInterval,generateFrameTimeline()"]
        n27["27 EventThreadConnection::postEvent<br/>写入BitTube"]
    end

    subgraph TIMER["Timer线程"]
        direction TB
        n19["19 Timer::dispatch<br/>epoll_wait返回,读出mCallback并执行"]
        n20["20 VSyncDispatchTimerQueue::timerCallback<br/>挑出到期Entry,executing()后重新arm剩下的Entry"]
        n21["21 VSyncDispatchTimerQueueEntry::callback<br/>设置mRunning=true,调用mCallback"]
        n22["22 MessageQueue::vsyncCallback<br/>翻转VSYNC-sf,生成vsyncId,投递到SF主线程"]
        n24["24 EventThread::onVsync<br/>翻转VSYNC-app,makeVSync()入队,唤醒EventThread线程"]
    end

    n1 --> n2
    n2 --> n3
    n3 --> n4
    n4 --> n5
    n5 --> n6
    n6 -.->|"样本处理完,按needsHwVsync切换开关"| n7
    n7 --> n8
    n8 -.->|"开启后,下一次硬件Vsync回到HWC"| n1

    n9 --> n10
    n10 -->|"Binder:requestNextVsync()"| n11
    n11 -->|"notify_all"| n12
    n12 --> n13
    n13 --> n14
    n14 --> n15
    n15 --> n16
    n16 --> n17
    n17 --> n18
    n18 -->|"timerfd到期"| n19
    n19 --> n20
    n20 --> n21

    n21 -->|"sf"| n22
    n22 -->|"投递到SF主线程"| n23

    n21 -->|"app"| n24
    n24 -->|"唤醒EventThread线程"| n25
    n25 --> n26
    n26 --> n27
    n27 -->|"BitTube"| n28
    n28 --> n29
    n29 --> n30
    n30 --> n31
    n31 -.->|"请求下一帧"| n9

    n23 -.->|"一帧上屏后"| n32
    n32 --> n33
    n33 --> n34
    n34 --> n35
    n35 -.->|"校准模型,闭环回到第5步"| n5

    classDef hw fill:#E3F2FD,stroke:#1565C0,color:#0D47A1
    classDef req fill:#FFF3E0,stroke:#E65100,color:#BF360C
    classDef sf fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    classDef app fill:#F3E5F5,stroke:#6A1B9A,color:#4A148C
    classDef fence fill:#ECEFF1,stroke:#455A64,color:#263238

    class n1,n2,n3,n4,n5,n6,n7,n8 hw
    class n9,n10,n11,n12,n13,n14,n15,n16,n17,n18,n19,n20,n21 req
    class n22,n23 sf
    class n24,n25,n26,n27,n28,n29,n30,n31 app
    class n32,n33,n34,n35 fence
```

## 2 怎么读这张图

### 2.1 颜色对应五个阶段

| 颜色 | 节点 | 阶段 |
|---|---|---|
| 蓝色 | 1~8 | 硬件Vsync进入SurfaceFlinger,并决定硬件Vsync开还是关 |
| 橙色 | 9~21 | 应用请求Vsync,EventThread算出时间点并布防Timer |
| 绿色 | 22~23 | `"sf"`路径,Timer回调让SurfaceFlinger开始一帧 |
| 紫色 | 24~31 | `"app"`路径,Vsync经BitTube送回应用并触发`doFrame` |
| 灰色 | 32~35 | PresentFence反馈,反过来校准预测模型 |

### 2.2 图里有三个闭环

- 硬件Vsync开关闭环:1→2→3→4→5→6→7→8,第8步打开硬件Vsync后,下一次HWC回调又回到第1步。样本攒够后第7步会关闭硬件Vsync,这个环就停转,只剩PresentFence继续喂模型。
- 请求/分发闭环:9→10→11→12→13→14→15→16→17→18→19→20→21,然后分叉到`"sf"`(22→23)和`"app"`(24→31),应用在`doFrame`末尾再次请求,虚线回到第9步。
- 校准闭环:第23步合成完成、帧上屏后产生PresentFence,32→33→34→35,虚线回到第5步,和第1步的硬件样本走同一个`VSyncPredictor::addVsyncTimestamp`入口。

### 2.3 四个容易看错的细节

- 第7、8步画在SF主线程的泳道里,调用起点却在HWC回调线程:第4步`VSyncReactor::addHwVsyncTimestamp`把`needsHwVsync`返回给第3步`VsyncSchedule::addResyncSample`,后者在锁外调用`enableHardwareVsync()`或`disableHardwareVsync()`,再经过注册进`VsyncSchedule`的`mRequestHardwareVsync`回调进入第7步`Scheduler::onHardwareVsyncRequest`。该函数用`schedule()`把真正的处理投到主线程,而且只有`HwVsyncState`真正翻转时才会走到第8步,所以图中第6步到第7步画成虚线,它不是一次普通函数返回。
- 第8步打开硬件Vsync时,只有`get_display_known_vsync_sample_enabled()`这个flag打开,才会用`getDisplayKnownVsyncSample()`拿到的样本再调一次`mScheduler->addResyncSample(..., VSyncTimeSource::HwVsyncQuery)`,相当于从第8步直接绕回第2步,不用等HWC下一次回调。
- 第17步不是每次`scheduleLocked`都执行:`VSyncDispatchTimerQueue::scheduleLocked`只在`callback->wakeupTime() < mIntendedWakeupTime - mTimerSlack`时才调用`rearmTimerSkippingUpdateFor(now, it)`,并且会跳过刚刚`schedule()`过的那个Entry,给其余Entry做`update()`后取最小的`wakeupTime`。如果新的唤醒时间不比已布防的时间早,定时器保持原样,不会重新arm。
- 第21步之后的两条路径由同一条Timer线程串行执行:先收集到期的Entry,再在锁外依次调用`MessageQueue::vsyncCallback`和`EventThread::onVsync`。两者的`wakeupTime`落在`mTimerSlack = 500us`以内时,Perfetto里看到的是同一次`timerCallback`唤醒。

### 2.4 时间点标注

图中第15步算出的三个时间点是整条链路的骨架,和[Vsync信号完整流转.md](Vsync信号完整流转.md)第8节一致:

```text
nextWakeupTime          nextReadyTime              nextVsyncTime
     │                       │                          │
     │  workDuration         │  readyDuration           │
     ├──────────────────────►├─────────────────────────►│
   Timer在此时唤醒         应用必须把buffer交给SF    这一帧上屏
   (nextWakeupTime)        (nextReadyTime)           (nextVsyncTime)
```

三个时间点都在第15步`VSyncDispatchTimerQueueEntry::schedule`里算好,第18步布防的是`nextWakeupTime`;`VSyncDispatchTimerQueueEntry::callback(nsecs_t vsyncTimestamp, nsecs_t wakeupTimestamp, nsecs_t deadlineTimestamp)`再把这三个值原样交给`MessageQueue::vsyncCallback`和`EventThread::onVsync`。
