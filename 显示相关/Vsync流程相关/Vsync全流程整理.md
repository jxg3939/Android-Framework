# Android 17 一帧从应用绘制到屏幕显示的完整流程

可以把整个过程整理为 帧调度 → UI 线程绘制 → RenderThread 渲染 → BufferQueue 交付 → SurfaceFlinger 合成 → HWC 提交显示 六个阶段。

## 一、整体流程

1\. 帧调度

ViewRootImpl.scheduleTraversals() → Choreographer#doFrame()

2\. UI 线程执行 Traversal

measure / layout / draw → ThreadedRenderer.draw()

3\. RenderThread 渲染并提交 Buffer

syncFrameState() → Skia / GPU → queueBuffer()

4\. BLASTBufferQueue 提交图形事务

acquire BufferItem → Transaction::setBuffer() → apply()

5\. SurfaceFlinger 更新 Layer 并执行合成决策

FrontEnd 更新 → latch buffer → HWC validate / present-or-validate

6\. 完成合成提交

必要时 RenderEngine 合成 Client Target → HWC present → 返回 fences

## 二、分阶段梳理

### 阶段 1：帧调度

1. 输入事件、动画、`invalidate()`、`requestLayout()`、Insets 或窗口状态变化等原因，使应用需要更新界面。
2. [`ViewRootImpl.scheduleTraversals()`](../Choreographer相关/Choreographer如何与SurfaceFlinger建立联系.md#scheduleTraversals()) 注册 traversal callback，通常通过 `Choreographer.postCallback()` 调度。
3. `Choreographer` 通过 `mFrameScheduled` 合并重复的帧调度请求，避免同一待处理帧被重复安排。
4. 收到 `vsync-app` 后，主线程进入 `Choreographer#doFrame()`，依次处理：

`INPUT → ANIMATION → INSETS_ANIMATION → TRAVERSAL → COMMIT`

### 阶段 2：UI 线程执行绘制

1. `Traversal` 根据 View 树状态决定是否需要执行 measure、layout 和 draw。
2. 硬件加速路径进入 `performDraw()`，再调用 `ThreadedRenderer.draw()`。
3. UI 线程更新 RenderNode / DisplayList 中的绘制状态。
4. 调用 `HardwareRenderer.syncAndDrawFrame()`，经 JNI 进入 native `RenderProxy::syncAndDrawFrame()`，将 `DrawFrameTask` 提交给 RenderThread。

这里的关键边界是：UI 线程负责更新绘制状态，RenderThread 负责同步这些状态并执行后续渲染工作。

### 阶段 3：RenderThread 渲染并生产 Buffer

1. RenderThread 执行 `syncFrameState()`，同步 UI 线程提交的 RenderNode 状态及相关帧数据。
2. 渲染管线准备 Skia / GPU 工作。
3. 通过应用窗口对应的 `Surface` 获取可用 buffer。底层连接到 BLASTBufferQueue 的生产者侧；当没有可用 slot，或者必须等待的 release fence 尚未满足时，获取 buffer 可能受阻。
4. Skia / GPU 生成并提交绘制命令。
5. 渲染完成状态通过 producer completion fence 表达。该 fence 随 buffer 提交后，在 Consumer 侧作为 acquire fence 使用，约束 Consumer 何时能够安全读取 buffer。
6. RenderThread 调用 `queueBuffer()`，把 buffer、相关 fence 和帧信息提交给 BLASTBufferQueue。

注意：`queueBuffer()` 返回并不意味着 GPU 已完成全部渲染，GPU 工作可能仍在进行。

### 阶段 4：BLASTBufferQueue 提交 Transaction

1. 应用进程内的 BLAST Consumer 获取已提交的 `BufferItem`。
2. BLASTBufferQueue 将 buffer 与对应的 SurfaceControl 图层事务关联，通过 `Transaction::setBuffer()` 设置 buffer。
3. 调用 `apply()`，将事务提交给 SurfaceFlinger。
4. SurfaceFlinger 接收到尚未完成处理的 buffer transaction 后，会更新相应的待处理事务状态；你提到的 `BufferTX - <layerName>` counter 用于观测这一类待处理 buffer transaction。

这一阶段完成的是 buffer 向 SurfaceFlinger 的交付，不代表 buffer 已经被合成或显示。

### 阶段 5：SurfaceFlinger 更新 Layer 并决定合成方式

1. SurfaceFlinger 在 `vsync-sf` 驱动的合成节奏下处理事务。
2. Android 17 FrontEnd 将事务更新合入 `RequestedLayerState`，再构建或更新 `LayerSnapshot`。
3. 对目标 App Window layer 执行 buffer latch。事务 readiness、acquire fence 和 latch 策略共同决定 buffer 何时能够进入后续合成流程；在满足相应条件时，也允许 latch acquire fence 尚未 signal 的 buffer。
4. SurfaceFlinger 将本轮可见 Layer 提交给 HWC，协商 composition strategy。

HWC 路径分为两类：

- 可以尝试跳过单独的 validate： 先调用 `presentOrValidateDisplay()`。
  - `PresentSucceeded`：HAL 调用走了 present 分支，返回相应的 present / release fence；这不代表面板已经完成显示。
  - `Validated`：本次调用完成了 validate，SurfaceFlinger 继续获取 composition type 变更和 HWC requests，必要时调用 `acceptChanges()`。
- 不能跳过 validate： 先调用 `validateDisplay()`，再处理 composition type 变更、requests 和 `acceptChanges()`，之后继续提交显示。

这里的 `PresentSucceeded` 与 `Validated` 是 `presentOrValidateDisplay()` 的结果状态，不应简单理解成“屏幕已显示”和“屏幕未显示”。

### 阶段 6：Client Target、HWC Present 与 Fence 回传

1. 如果存在 `CLIENT` composition layer，SurfaceFlinger 使用 RenderEngine 消费相关输入 buffer，并生成供 HWC 使用的 client target。
2. 调用 `setClientTarget()`，传入 client target 及其 output acquire fence。
3. 通过 `presentAndGetReleaseFences()` 完成相应收尾；如果前面的 `presentOrValidateDisplay()` 尚未执行 present，则最终需要调用 `presentDisplay()`。
4. HWC 返回 per-display present fence 和 per-layer release fence。
5. SurfaceFlinger 根据 client/device composition 等情况处理、合并或替换 release 信息，并通过相应的 release callback / channel 将释放信息回传给 Producer。

需要区分两种 fence：

| Fence         | 含义                                     |
| ------------- | -------------------------------------- |
| Acquire fence | Consumer 读取对应 buffer 前需要遵守的同步条件        |
| Release fence | Producer 可以安全复用或覆盖对应 buffer 前需要遵守的同步条件 |
| Present fence | 显示栈用来标记本次提交的呈现时序，是判断显示进度的重要时间锚点        |

Present fence 不等于面板完成光学响应的时刻。

## 三、理解整个流程时最重要的三个边界

1. UI 绘制完成 ≠ GPU 渲染完成。 UI 线程提交的是绘制状态和渲染任务，GPU 工作由后续渲染管线执行。
2. Buffer 提交 ≠ SurfaceFlinger latch ≠ HWC present。 `queueBuffer()`、buffer latch 和显示提交是三个不同阶段，分别对应生产者交付、SurfaceFlinger 采用 buffer、显示硬件提交。
3. HWC present ≠ 面板光学显示完成。 Present fence 和 release fence 描述不同的同步关系，不能直接把它们等同于用户实际看到像素变化的时间。

以上流程描述的是 Android 17 硬件加速窗口的典型路径；具体实现会因是否存在 Client composition、是否能跳过 validate，以及 buffer 同步状态而有所不同。