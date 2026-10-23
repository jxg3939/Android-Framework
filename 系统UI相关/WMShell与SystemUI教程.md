# WMShell与SystemUI教程

本文讲清楚两件事:WMShell(WindowManager Shell)是什么、在SystemUI进程里怎么被拉起来;SystemUI又是什么、它的架构如何组织。最后讲两者之间的协作面,以及怎么调试。

源码基准是最新的AOSP:`frameworks/base`的`refs/heads/main`(即android-latest-release),全部内容都以实际源码为准。

## 1 为什么要拆成WMCore与WMShell

Android的窗口管理分成两层:

- **WMCore**,即`WindowManagerService`及其核心框架,跑在`system_server`进程。它负责窗口的**真相**:窗口树结构、层级与z序、可见性、焦点、insets、task与activity容器的组织。任何窗口状态的最终裁决都在这里。
- **WMShell**(WindowManager Shell),代码在`frameworks/base/libs/WindowManager/Shell`,跑在**SystemUI进程**。它负责窗口的**表现**:过渡动画、分屏、画中画、气泡、桌面模式的窗口装饰、启动闪屏等。

这个划分在AOSP的设计文档里写得很直白(`libs/WindowManager/Shell/src/com/android/wm/shell/docs/overview.md`):

> The primary motivation for the WindowManager Shell (WMShell) library is to effectively scale WindowManager by making it easy and safe to create windowing features to fit the needs of various Android products and form factors.
>
> To achieve this, WindowManager separates the policy of managing windows (WMCore) from the presentation of surfaces (WMShell) and provides a minimal interface boundary for the two to communicate.

一句话:**策略归WMCore,呈现归WMShell**,两者之间只留一条尽量窄的接口边界。这样做的收益是,OEM和不同形态设备(手机、平板、TV、Auto、Wear)可以只改WMShell这一层就定制出不同的多窗口体验,而不用改WMCore。

三者的定位:

| | WMCore | WMShell | SystemUI |
|---|---|---|---|
| 进程 | `system_server` | `com.android.systemui` | `com.android.systemui` |
| 代码位置 | `services/core/java/com/android/server/wm` | `libs/WindowManager/Shell` | `packages/SystemUI` |
| 编译产物 | 编进`services.jar` | `android_library`名字叫`WindowManager-Shell` | `SystemUI.apk` |
| 职责 | 窗口策略与状态真相 | 窗口表现与多窗口功能 | 状态栏、通知面板、导航栏等系统界面 |
| 生命周期 | 系统启动即存在 | 随SystemUI进程 | 随SystemUI进程 |

关键结论:**WMShell不是一个独立进程,也不是一个独立的apk或apex**。它是一个`android_library`静态库,被编进SystemUI的apk,和SystemUI跑在同一个进程`com.android.systemui`里。这一点决定了后面所有的线程、依赖注入和进程边界讨论。

`libs/WindowManager/Shell/Android.bp`里:

```python
android_library {
    name: "WindowManager-Shell",
    srcs: [
        "src/com/android/wm/shell/EventLogTags.logtags",
        ":wm_shell_protolog_src",
        ":wm_shell-sources-kt",
        ":wm_shell-aidl/s",
        ":wm_shell-shared-aidls",
    ],
    ...
    static_libs: [
        ...
        "WindowManager-Shell-shared",
        "dagger2",
        "jsr330",
    ],
    plugins: ["dagger2-compiler"],
}
```

也可以单独编:`m WindowManager-Shell`,但这只用于验证库本身能否编过或看产物内容,实际使用时它是跟着`m SystemUI`一起编的。

## 2 代码结构与编译产物

### 2.1 三个代码仓库位置

| 路径 | 内容 |
|---|---|
| `libs/WindowManager/Shell` | WMShell主体,`WindowManager-Shell`库 |
| `libs/WindowManager/Shell/shared` | Shell对外共享的接口与常量,`WindowManager-Shell-shared`库 |
| `libs/WindowManager/Jetpack` | Jetpack WindowManager库(第三方依赖,与Shell无关) |
| `packages/SystemUI` | SystemUI主体,`SystemUI.apk` |
| `packages/SystemUI/shared` | SystemUI对Launcher共享的库,`SystemUI-shared` |
| `packages/SystemUI/plugin` | 插件接口库,`SystemUI-plugin` |
| `packages/SystemUI/plugin_core` | 插件框架核心实现 |

`Shell/shared`存在的意义:AOSP的Launcher(`packages/apps/Launcher3`)需要用到一部分Shell的AIDL和常量,但Launcher**不**编译依赖`WindowManager-Shell`(那会把整个Shell拖进去)。所以约定:所有AIDL文件、以及`com.android.wm.shell.shared`包下的类,都自动编进`SystemUISharedLib`(即`SystemUI-shared`),供Launcher使用。

Shell侧目录(`libs/WindowManager/Shell/src/com/android/wm/shell/`):

```
EventLogTags.logtags        ProtoLogController.java     RootDisplayAreaOrganizer.java
RootTaskDisplayAreaOrganizer.java   ShellTaskOrganizer.java   WindowManagerShellWrapper.java
activityembedding/  animation/  apptoweb/  automotive/  back/  bubbles/  common/
compatui/  dagger/  desktopmode/  displayareahelper/  docs/  draganddrop/  freeform/
fullscreen/  hidedisplaycutout/  keyguard/  onehanded/  performance/  pip/  pip2/
protolog/  recents/  splitscreen/  startingsurface/  sysui/  taskview/  transition/
unfold/  windowdecor/
```

其中两个目录需要特别记住:

- `sysui/`:Shell与SystemUI对接的所有类都在这里,只有9个文件。
- `dagger/`:Shell的所有依赖注入代码都在这里,AOSP刻意把所有"魔法"集中在一个目录。

### 2.2 Shell/shared里的共享接口

```
CounterRotator.java     FocusTransitionListener.java   GroupedTaskInfo.aidl/java
IFocusTransitionListener.aidl   IHomeTransitionListener.aidl   IShellTransitions.aidl
ShellSharedConstants.java   ShellTransitions.java   TransactionPool.java
TransitionUtil.java   TriangleShape.java
animation/  annotations/  bubbles/  desktopmode/  draganddrop/  handles/
magnetictarget/  multiinstance/  pip/  split/  startingsurface/
```

`ShellTransitions`(`libs/WindowManager/Shell/shared/src/com/android/wm/shell/shared/ShellTransitions.java`)是给SystemUI用的接口:

```java
/**
 * Interface to manage remote transitions.
 */
@ExternalThread
public interface ShellTransitions {
    default void registerRemote(@NonNull TransitionFilter filter,
            @NonNull RemoteTransition remoteTransition) {}
    default void registerRemoteForTakeover(@NonNull TransitionFilter filter,
            @NonNull RemoteTransition remoteTransition) {}
    default void unregisterRemote(@NonNull RemoteTransition remoteTransition) {}
    default void setFocusTransitionListener(@NonNull FocusTransitionListener listener,
            Executor executor) {}
    default void unsetFocusTransitionListener(@NonNull FocusTransitionListener listener) {}
}
```

注意`@ExternalThread`这个注解。Shell里所有暴露给外部(指SystemUI主线程或其它进程)的接口都标它,含义是:接口的实现自己负责把调用post到正确的线程,调用方不需要关心线程问题。这个约定后面第5节还会展开。

`IShellTransitions.aidl`是给**其它进程**(主要是Launcher)用的binder版本:

```java
interface IShellTransitions {
    oneway void registerRemote(in TransitionFilter filter,
            in RemoteTransition remoteTransition) = 1;
    oneway void unregisterRemote(in RemoteTransition remoteTransition) = 2;
    /** Retrieves the apply-token used by transactions in Shell */
    IBinder getShellApplyToken() = 3;
    oneway void setHomeTransitionListener(in IHomeTransitionListener listener) = 4;
    /** Returns a container surface for the home root task. */
    SurfaceControl getHomeTaskOverlayContainer() = 5;
    ...
}
```

一个功能要同时给三种调用方用,就会有三种形态的接口:进程内SystemUI用`ShellTransitions`(Java接口),跨进程用`IShellTransitions`(AIDL),Shell内部用`Transitions`的`TransitionHandler`。这是Shell里反复出现的三段式。

### 2.3 feature flag与trace proto

Shell用aconfig管理特性开关:`libs/WindowManager/Shell/aconfig/multitasking.aconfig`,编译出`com_android_wm_shell_flags_lib`。代码里通过`com.android.wm.shell.Flags`访问,例如`StageCoordinator`里的`Flags.enableFlexibleSplit`。

Shell还带两个proto用于Winscope trace:

```
libs/WindowManager/Shell/proto/wm_shell_trace.proto
libs/WindowManager/Shell/proto/wm_shell_transition_trace.proto
```

编译出`WindowManager-Shell-proto`(nano)和`WindowManager-Shell-lite-proto`。过渡动画的Perfetto/Winscope追踪就靠它们。

## 3 启动流程:从system_server到Shell初始化完成

这一节把整条链走一遍。它是理解WMShell与SystemUI关系的地基。

### 3.1 system_server拉起SystemUI

`services/java/com/android/server/SystemServer.java`:

```java
// startOtherServices()里
t.traceBegin("StartSystemUI");
try {
    startSystemUi(context, windowManagerF);
} catch (Throwable e) {
    reportWtf("starting System UI", e);
}
t.traceEnd();
```

```java
private static void startSystemUi(Context context, WindowManagerService windowManager) {
    PackageManagerInternal pm = LocalServices.getService(PackageManagerInternal.class);
    Intent intent = new Intent();
    intent.setComponent(pm.getSystemUiServiceComponent());
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //Slog.d(TAG, "Starting service: " + intent);
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
    windowManager.onSystemUiStarted();
}
```

三个要点:

1. 时机是在`startOtherServices()`的后段,即系统服务基本起完之后。
2. Component名不是写死的,而是`pm.getSystemUiServiceComponent()`,由`PackageManagerInternal`从配置里解析。OEM可以换掉SystemUI的实现类。
3. `windowManager.onSystemUiStarted()`通知WMS:SystemUI已经来了。WMS后续关于系统栏、insets的很多交互都以SystemUI存在为前提。

### 3.2 AppComponentFactory:比Application构造更早的注入点

SystemUI的启动不是普通的`Application.onCreate`路径,而是靠`AndroidManifest.xml`里声明的`android.app.AppComponentFactory`。

`SystemUIAppComponentFactoryBase`(`packages/SystemUI/src/com/android/systemui/SystemUIAppComponentFactoryBase.kt`)的关键部分:

```kotlin
abstract class SystemUIAppComponentFactoryBase : AppComponentFactory() {
    companion object {
        private const val TAG = "AppComponentFactory"
        // Must be static due to http://b/141008541.
        var systemUIInitializer: SystemUIInitializer? = null
    }

    @set:Inject
    lateinit var componentHelper: ContextComponentHelper

    /** Returns a new [SystemUIInitializer]. The returned implementation should be specific to your build. */
    protected abstract fun createSystemUIInitializer(context: Context): SystemUIInitializer

    private fun createSystemUIInitializerInternal(context: Context): SystemUIInitializer {
        return systemUIInitializer ?: run {
            val initializer = createSystemUIInitializer(context.applicationContext)
            try {
                initializer.init(false)
            } catch (exception: ExecutionException) {
                throw RuntimeException("Failed to initialize SysUI", exception)
            } catch (exception: InterruptedException) {
                throw RuntimeException("Failed to initialize SysUI", exception)
            }
            initializer.sysUIComponent.inject(this@SystemUIAppComponentFactoryBase)
            systemUIInitializer = initializer
            return initializer
        }
    }

    override fun instantiateApplicationCompat(cl: ClassLoader, className: String): Application {
        val app = super.instantiateApplicationCompat(cl, className)
        if (app !is ContextInitializer) {
            throw RuntimeException("App must implement ContextInitializer")
        } else {
            app.setContextAvailableCallback { context ->
                createSystemUIInitializerInternal(context)
            }
        }
        return app
    }
    // ...
}
```

注意`run`包住的这段:`SystemUIInitializer`是单例,只建一次;建的时候顺便把**整个Dagger图**立起来。所以"依赖注入图构建完成"这件事发生在`Application`对象被创建的过程中,而不是之后。

手机形态的实现只有一个方法:

```kotlin
// packages/SystemUI/src/com/android/systemui/PhoneSystemUIAppComponentFactory.kt
class PhoneSystemUIAppComponentFactory : SystemUIAppComponentFactoryBase() {
    override fun createSystemUIInitializer(context: Context) = SystemUIInitializerImpl(context)
}
```

```kotlin
// packages/SystemUI/src/com/android/systemui/SystemUIInitializerImpl.kt
class SystemUIInitializerImpl(context: Context) : SystemUIInitializer(context) {
    override fun getGlobalRootComponentBuilder(): GlobalRootComponent.Builder {
        return DaggerReferenceGlobalRootComponent.builder()
    }
}
```

`SystemUIApplication.onCreate`随后才跑:

```java
// packages/SystemUI/src/com/android/systemui/SystemUIApplication.java
@Override
public void onCreate() {
    super.onCreate();
    Log.v(TAG, "SystemUIApplication created.");
    // This line is used to setup Dagger's dependency injection and should be kept at the
    // top of this method.
    TimingsTraceLog log = new TimingsTraceLog("SystemUIBootTiming",
            Trace.TRACE_TAG_APP);
    log.traceBegin("DependencyInjection");
    mInitializer = mContextAvailableCallback.onContextAvailable(this);
    mSysUIComponent = mInitializer.getSysUIComponent();
    mBootCompleteCache = mSysUIComponent.provideBootCacheImpl();
    log.traceEnd();
    // ...
}
```

`onContextAvailable`就是刚才`setContextAvailableCallback`塞进去的那个回调,所以这行等价于"如果还没建过,就把整张Dagger图建起来"。

### 3.3 SystemUIInitializer.init():三棵依赖树的交接

这是整个启动流程里最核心的一个方法,`packages/SystemUI/src/com/android/systemui/SystemUIInitializer.java`:

```java
public void init(boolean fromTest) throws ExecutionException, InterruptedException {
    mRootComponent = getGlobalRootComponentBuilder()
            .context(mContext)
            .instrumentationTest(fromTest)
            .build();

    mInitializationChecker = mRootComponent.getInitializationChecker();
    boolean initializeComponents = mInitializationChecker.initializeComponents();

    // Stand up WMComponent
    setupWmComponent(mContext);

    // And finally, retrieve whatever SysUI needs from WMShell and build SysUI.
    SysUIComponent.Builder builder = mRootComponent.getSysUIComponent();
    if (initializeComponents) {
        builder = prepareSysUIComponentBuilder(builder, mWMComponent)
                .setShell(mWMComponent.getShell())
                .setPip(mWMComponent.getPip())
                .setSplitScreen(mWMComponent.getSplitScreen())
                .setOneHanded(mWMComponent.getOneHanded())
                .setBubbles(mWMComponent.getBubbles())
                .setTaskViewFactory(mWMComponent.getTaskViewFactory())
                .setShellTransitions(mWMComponent.getShellTransitions())
                .setKeyguardTransitions(mWMComponent.getKeyguardTransitions())
                .setStartingSurface(mWMComponent.getStartingSurface())
                .setDisplayAreaHelper(mWMComponent.getDisplayAreaHelper())
                .setRecentTasks(mWMComponent.getRecentTasks())
                .setBackAnimation(mWMComponent.getBackAnimation())
                .setDesktopMode(mWMComponent.getDesktopMode());

        // Only initialize when not starting from tests since this currently initializes some
        // components that shouldn't be run in the test environment
        mWMComponent.init();
    } else {
        // TODO: Call on prepareSysUIComponentBuilder but not with real components. Other option
        // is separating this logic into newly creating SystemUITestsFactory.
        builder = prepareSysUIComponentBuilder(builder, mWMComponent)
                .setShell(new ShellInterface() {})
                .setPip(Optional.ofNullable(null))
                // ... 其余接口全部给null
    }
    mSysUIComponent = builder.build();

    // Every other part of our codebase currently relies on Dependency, so we
    // really need to ensure the Dependency gets initialized early on.
    Dependency dependency = mSysUIComponent.createDependency();
    dependency.start();
}
```

顺序极其重要,拆开看:

1. **先建`GlobalRootComponent`**。它是`@Singleton`作用域,提供`Context`、主`Looper`、`SystemPropertiesHelper`这些最基础的东西。
2. **再建`WMComponent`**(`setupWmComponent`)。这是Shell的`@WMSingleton`作用域。
3. **把WMComponent导出的13个接口喂给`SysUIComponent.Builder`**。这是Shell与SystemUI之间**唯一**的注入通道,第11节会逐个列。
4. **调`mWMComponent.init()`**,触发Shell自己的初始化(见3.4)。
5. **`builder.build()`建出`SysUIComponent`**(`@SysUISingleton`作用域),再`createDependency().start()`。

`initializeComponents`为false时(比如测试环境),所有Shell接口都被替换成空实现或null。这就是为什么SysUI里注入Shell接口时类型常常写成`Optional<Pip>`——测试环境里真的可能是空的。

`setupWmComponent`本身也有讲究:

```java
private void setupWmComponent(Context context) {
    WMComponent.Builder wmBuilder = mRootComponent.getWMComponentBuilder();
    if (!mInitializationChecker.initializeComponents()
            || !WMShellConcurrencyModule.enableShellMainThread(context)) {
        // If running under tests or shell thread is not enabled, we don't need anything special
        mWMComponent = wmBuilder.build();
        return;
    }

    // If the shell main thread is enabled, initialize the component on that thread
    HandlerThread shellThread = WMShellConcurrencyModule.createShellMainThread();
    shellThread.start();

    // Use an async handler since we don't care about synchronization
    Handler shellHandler = Handler.createAsync(shellThread.getLooper());
    boolean built = shellHandler.runWithScissors(() -> {
        wmBuilder.setShellMainThread(shellThread);
        mWMComponent = wmBuilder.build();
    }, 5000);
    if (!built) {
        Log.w(TAG, "Failed to initialize WMComponent");
        throw new RuntimeException();
    }
}
```

方法上的注释把原因说清楚了:

> On devices where the Shell runs on its own main thread, this will pre-create the thread to ensure that the components are constructed on the same thread, to reduce the likelihood of side effects from running the constructors on a different thread than the rest of the class logic.

也就是说,如果设备开了独立Shell主线程(`config_enableShellMainThread`),那么**所有Shell组件的构造函数都在那条线程上跑**,包括对象图的构建本身。这是为了避免"对象在SystemUI主线程构造、逻辑却在Shell线程上跑"导致的各种隐式状态问题。超时5000ms,超了直接抛异常——Shell图建不起来,SystemUI宁可不启动。

### 3.4 WMComponent.init()与ShellInit:有序的初始化回调

`WMComponent`自己带一个`init()`:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/dagger/WMComponent.java
/**
 * Initializes all the WMShell components before starting any of the SystemUI components.
 */
default void init() {
    getShell().onInit();
}
```

`getShell()`返回`ShellInterface`,真实实现是`ShellController`内部的`ShellInterfaceImpl`:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/sysui/ShellController.java
@ExternalThread
private class ShellInterfaceImpl implements ShellInterface {
    @Override
    public void onInit() {
        mMainExecutor.execute(ShellController.this::handleInit);
    }
    // ...
}

private void handleInit() {
    SurfaceControlRegistry.createProcessInstance(mContext);
    mShellInit.init();
}
```

`SurfaceControlRegistry.createProcessInstance`这一行很关键:它给SystemUI进程建了一个`SurfaceControl`注册表。之后Shell和SystemUI创建的所有`SurfaceControl`,在debug版里都会被登记,用于排查"surface泄漏"和"释放后使用"。

于是到了`ShellInit`。它是Shell"初始化回调"的编排器(`libs/WindowManager/Shell/src/com/android/wm/shell/sysui/ShellInit.java`):

```java
public class ShellInit {
    private final ShellExecutor mMainExecutor;
    // An ordered list of init callbacks to be made once shell is first started
    private final ArrayList<Pair<String, Runnable>> mInitCallbacks = new ArrayList<>();
    private boolean mHasInitialized;

    public ShellInit(ShellExecutor mainExecutor) {
        mMainExecutor = mainExecutor;
        ProtoLog.init(ShellProtoLogGroup.values());
    }

    public <T extends Object> void addInitCallback(Runnable r, T instance) {
        if (mHasInitialized) {
            if (Build.isDebuggable()) {
                // All callbacks must be added prior to the Shell being initialized
                throw new IllegalArgumentException("Can not add callback after init");
            }
            return;
        }
        final String className = instance.getClass().getSimpleName();
        mInitCallbacks.add(new Pair<>(className, r));
        ProtoLog.v(WM_SHELL_INIT, "Adding init callback for %s", className);
    }

    @VisibleForTesting
    public void init() {
        ProtoLog.v(WM_SHELL_INIT, "Initializing Shell Components: %d", mInitCallbacks.size());
        SurfaceControl.setDebugUsageAfterRelease(true);
        // Init in order of registration
        for (int i = 0; i < mInitCallbacks.size(); i++) {
            final Pair<String, Runnable> info = mInitCallbacks.get(i);
            final long t1 = SystemClock.uptimeMillis();
            info.second.run();
            final long t2 = SystemClock.uptimeMillis();
            ProtoLog.v(WM_SHELL_INIT, "\t%s init took %dms", info.first, (t2 - t1));
        }
        mInitCallbacks.clear();
        mHasInitialized = true;
    }
}
```

Shell初始化模型可以总结成三条规则:

- **每个Shell组件在自己的构造函数里调`shellInit.addInitCallback(this::onInit, this)`**,不在构造函数里做真正的初始化。
- **回调按注册顺序执行**。因为注册发生在构造函数里,而Dagger保证依赖先构造,所以"依赖注入的顺序"就是"初始化顺序"。`onInit`里的日志还会打印每个组件花了多少毫秒,方便看启动耗时。
- **初始化之后不能再注册**。debug版会直接抛`IllegalArgumentException("Can not add callback after init")`。

典型用法:

```java
// ShellController.java
public ShellController(Context context, ShellInit shellInit, ShellCommandHandler shellCommandHandler,
        DisplayInsetsController displayInsetsController, ShellExecutor mainExecutor) {
    // ...
    shellInit.addInitCallback(this::onInit, this);
}

private void onInit() {
    mShellCommandHandler.addDumpCallback(this::dump, this);
    mDisplayInsetsController.addInsetsChangedListener(
            mContext.getDisplayId(), mInsetsChangeListener);
}
```

验证组件有没有被初始化,可以打开`WM_SHELL_INIT`这个ProtoLog组,重启SystemUI:

```shell
adb shell wm logging enable-text WM_SHELL_INIT
adb shell kill $(pidof com.android.systemui)
adb logcat *:S WindowManagerShell
```

### 3.5 CoreStartable与WMShell.start()

Shell的图建完之后,SystemUI才开始启动自己的"服务"。这里的`WMShell`(`packages/SystemUI/src/com/android/systemui/wmshell/WMShell.java`)名字容易和Shell库混淆,它的类注释写得很清楚:

> A SystemUI service that starts with the SystemUI application and sets up any bindings between Shell and SysUI components. This service starts happens after the {@link WMComponent} has already been initialized and may only reference Shell components that are explicitly exported to SystemUI.

注释下面还画了一遍完整链路:

```
 * eg. SysUI application starts
 *     -> SystemUIFactory is initialized
 *       -> WMComponent is created
 *         -> WMShellBaseModule dependencies are injected
 *         -> WMShellModule (form-factory specific) dependencies are injected
 *       -> SysUIComponent is created
 *         -> WMComponents are explicitly provided to SysUIComponent for injection into SysUI code
 *     -> SysUI services are started
 *       -> WMShell starts and binds SysUI with Shell components via exported Shell interfaces
```

它是`@SysUISingleton`的`CoreStartable`:

```java
@SysUISingleton
public final class WMShell implements CoreStartable, CommandQueue.Callbacks {
    // ...
    @Override
    public void start() {
        // Notify with the initial configuration and subscribe for new config changes
        mShell.onConfigurationChanged(mContext.getResources().getConfiguration());
        mConfigurationController.addCallback(mConfigurationListener);

        // Subscribe to keyguard changes
        mKeyguardStateController.addCallback(mKeyguardStateCallback);
        mKeyguardUpdateMonitor.registerCallback(mKeyguardUpdateMonitorCallback);

        // Subscribe to user changes
        mUserTracker.addCallback(mUserChangedCallback, mContext.getMainExecutor());

        mCommandQueue.addCallback(this);
        mPipOptional.ifPresent(this::initPip);
        mSplitScreenOptional.ifPresent(this::initSplitScreen);
        mOneHandedOptional.ifPresent(this::initOneHanded);
        mDesktopModeOptional.ifPresent(this::initDesktopMode);
        mRecentTasksOptional.ifPresent(this::initRecentTasks);

        mNoteTaskInitializer.initialize();
    }
```

注意方向:**SystemUI把自己的事件推给Shell**。配置变化、锁屏状态、用户切换,都由SystemUI侧监听,然后调`ShellInterface`对应方法通知Shell。Shell不主动去SystemUI里注册监听器。

`start()`是什么时候被调的?`SystemUIService.onCreate`:

```java
// packages/SystemUI/src/com/android/systemui/SystemUIService.java
@Override
public void onCreate() {
    super.onCreate();
    // Start all of SystemUI
    ((SystemUIApplication) getApplication()).startSystemUserServicesIfNeeded();
    // ...
}
```

再到`SystemUIApplication.startServicesIfNeeded`:

```java
private void startServicesIfNeeded(
        Map<Class<?>, Provider<CoreStartable>> startables,
        String metricsPrefix,
        String vendorComponent) {
    if (mServicesStarted) {
        return;
    }
    mServices = new CoreStartable[startables.size() + (vendorComponent == null ? 0 : 1)];
    // ...
    HashSet<Class<?>> startedStartables = new HashSet<>();

    // Perform a form of topological sort:
    // 1) Iterate through a queue of all non-started startables
    //   If the startable has all of its dependencies met
    //     - start it
    //   Else
    //     - enqueue it for the next iteration
    // 2) If anything was started and the "next" queue is not empty, loop back to 1
    // 3) If we're done looping and there are any non-started startables left, throw an error.
    // ...
}
```

这是一次**拓扑排序**:如果某个`CoreStartable`声明了依赖其它`CoreStartable`,就等依赖先启动。全都启动不了就抛`RuntimeException("Failed to start all CoreStartables. Check logcat!")`。

### 3.6 完整时序

| # | 位置 | 动作 |
|---|---|---|
| 1 | `SystemServer.startOtherServices()` | `startSystemUi(context, windowManagerF)` |
| 2 | `SystemServer.startSystemUi` | `startServiceAsUser(SystemUI的Service组件, SYSTEM)` + `wms.onSystemUiStarted()` |
| 3 | `SystemUIAppComponentFactoryBase.instantiateApplicationCompat` | 给Application装`ContextAvailableCallback` |
| 4 | `SystemUIApplication.onCreate` | 调`onContextAvailable` → 触发`createSystemUIInitializerInternal` |
| 5 | `SystemUIInitializer.init` | `getGlobalRootComponentBuilder().build()` 建出`GlobalRootComponent` |
| 6 | 同上 | `setupWmComponent`:必要时先在`wmshell.main`线程上建`WMComponent` |
| 7 | 同上 | 把13个Shell接口`setXxx`进`SysUIComponent.Builder` |
| 8 | 同上 | `mWMComponent.init()` → `getShell().onInit()` → `ShellController.handleInit()` |
| 9 | `ShellController.handleInit` | `SurfaceControlRegistry.createProcessInstance(context)`;`ShellInit.init()` |
| 10 | `ShellInit.init` | 按注册顺序跑所有`onInit`回调(各控制器开始注册TaskOrganizer、DisplayAreaOrganizer、TransitionHandler等) |
| 11 | `SystemUIInitializer.init` | `mSysUIComponent = builder.build()`;`createDependency().start()` |
| 12 | `SystemUIService.onCreate` | `startSystemUserServicesIfNeeded()` |
| 13 | `SystemUIApplication.startServicesIfNeeded` | 拓扑排序启动所有`CoreStartable` |
| 14 | `WMShell.start` | 把SystemUI的配置/锁屏/用户/CommandQueue事件桥接到`ShellInterface` |

一句话概括:**Shell先于SystemUI启动**。Shell把图建好、把`onInit`回调都跑完,此时它已经能接收`TaskOrganizer`的task回调了;SystemUI才开始启动自己的服务,并通过`WMShell`这个`CoreStartable`把两边接起来。

## 4 Dagger依赖注入体系

### 4.1 三个作用域

SystemUI与WMShell共用一套Dagger图,但分出三个作用域:

| 作用域 | 注解 | 组件 | 说明 |
|---|---|---|---|
| 全局单例 | `@Singleton` | `GlobalRootComponent` | `Context`、主`Looper`、`SystemPropertiesHelper`等最基础的对象 |
| Shell | `@WMSingleton` | `WMComponent` | 所有WindowManager相关代码。**刻意不让它依赖SystemUI的其余部分**,也不让SystemUI窥探它的内部 |
| SystemUI | `@SysUISingleton` | `SysUIComponent` | 绝大部分SystemUI代码。Shell导出的接口在这里可见 |

AOSP的SystemUI文档(`packages/SystemUI/docs/dagger.md`)对`@WMShell`这个作用域的原话:

> `@WMShell` - WindowManager related code in the SystemUI process. We don't want this code relying on the rest of SystemUI, and we don't want the rest of SystemUI peeking into its internals, so it runs in its own Subcomponent.

这个注解在代码里的实际名字是`@WMSingleton`,在Shell库里定义(`libs/WindowManager/Shell/src/com/android/wm/shell/dagger/WMSingleton.java`):

```java
/**
 * Scope annotation for singleton items within the WMComponent.
 */
@Documented
@Retention(RUNTIME)
@Scope
public @interface WMSingleton {
}
```

`WMComponent`是`GlobalRootComponent`的子组件:

```java
// packages/SystemUI/src/com/android/systemui/dagger/GlobalRootComponent.java
/**
 * Base root component for Dagger injection.
 *
 * This class is not actually annotated as a Dagger component, since it is not used directly as one.
 * Doing so generates unnecessary code bloat.
 *
 * See {@link ReferenceGlobalRootComponent} for the one actually used by AOSP.
 */
public interface GlobalRootComponent {

    interface Builder {
        @BindsInstance Builder context(Context context);
        @BindsInstance Builder instrumentationTest(@InstrumentationTest boolean test);
        GlobalRootComponent build();
    }

    /** Builder for a {@link WMComponent}, which makes it a subcomponent of this class. */
    WMComponent.Builder getWMComponentBuilder();

    /** Builder for a {@link ReferenceSysUIComponent}, which makes it a subcomponent of this class. */
    SysUIComponent.Builder getSysUIComponent();
    // ...
}
```

注意`GlobalRootComponent`**故意不加`@Component`注解**,也不叫`DaggerGlobalRootComponent`,真正的实现叫`ReferenceGlobalRootComponent`。这是OEM替换入口:OEM写自己的`GlobalRootComponent`子接口和`SystemUIInitializer`,就能换掉整张图。

### 4.2 Shell的模块树

Shell所有Dagger代码都在`com.android.wm.shell.dagger`包里。AOSP刻意这么做,把"魔法"集中在一处(`docs/dagger.md`)。

模块的包含关系:

```
WMShellConcurrencyModule (线程相关)
  └── WMShellBaseModule (所有产品通用的组件,如DisplayController、Transitions)
        ├── WMShellModule (手机/平板专有)
        └── TvPipModule (TV的PiP)
              └── TvWMShellModule (TV专有)
```

对应的`@Module(includes=...)`:

```java
// dagger/WMShellModule.java
@Module(includes = {WMShellBaseModule.class, PipModule.class, ShellBackAnimationModule.class,
        LetterboxModule.class})
public abstract class WMShellModule { }
```

```java
// dagger/WMShellBaseModule.java
@Module(
        includes = {
                WMShellConcurrencyModule.class,
                WMShellCoroutinesModule.class
        })
public abstract class WMShellBaseModule { }
```

`WMShellBaseModule`的类注释说明了它对外暴露的方式:

> dependencies … only accessible from components within the WM subcomponent (can be explicitly exposed to the SysUIComponent, see `com.android.systemui.dagger.WMComponent`).

几个关键Provider:

```java
@WMSingleton
@Provides
static ShellInit provideShellInit(@ShellMainThread ShellExecutor mainExecutor) {
    return new ShellInit(mainExecutor);
}

@WMSingleton
@Provides
static ShellCommandHandler provideShellCommandHandler() {
    return new ShellCommandHandler();
}

@WMSingleton
@Provides
static ProtoLogController provideProtoLogController(ShellInit shellInit,
        ShellCommandHandler shellCommandHandler) {
    return new ProtoLogController(shellInit, shellCommandHandler);
}
```

`ShellTaskOrganizer`的Provider里藏了一个开关:

```java
@WMSingleton
@Provides
static ShellTaskOrganizer provideShellTaskOrganizer(
        Context context, ShellInit shellInit, ShellCommandHandler shellCommandHandler,
        Optional<CompatUIHandler> compatUI, Optional<UnfoldAnimationController> unfoldAnimationController,
        Optional<RecentTasksController> recentTasksOptional,
        @ShellMainThread ShellExecutor mainExecutor) {
    if (!context.getResources().getBoolean(R.bool.config_registerShellTaskOrganizerOnInit)) {
        // TODO(b/238217847): Force override shell init if registration is disabled
        shellInit = new ShellInit(mainExecutor);
    }
    return new ShellTaskOrganizer(shellInit, shellCommandHandler, compatUI.orElse(null),
            unfoldAnimationController, recentTasksOptional, mainExecutor);
}
```

如果资源`config_registerShellTaskOrganizerOnInit`为false,就换一个全新的`ShellInit`,让`ShellTaskOrganizer`脱离主初始化流程——这是给那些不想让Shell接管task组织的产品留的口子。

### 4.3 ShellCreateTrigger:让"没人依赖的组件"也能被建出来

Dagger只会构建"被别人需要"的对象。但Shell里很多组件是独立工作的,没有任何导出接口依赖它们,不显式触发就永远不会被创建。

`@ShellCreateTrigger`(`dagger/ShellCreateTrigger.java`)就是干这个的:

```java
/**
 * An annotation to specifically mark the provider that is triggering the creation of independent
 * shell components that are not created as a part of the dependencies for interfaces passed to
 * SysUI.
 * TODO: This will be removed once we have a more explicit method for specifying components to start
 *       with SysUI
 */
@Documented @Inherited @Qualifier @Retention(RetentionPolicy.RUNTIME)
public @interface ShellCreateTrigger {}
```

用法是在`WMShellBaseModule`里写一组"什么也不干、只把组件拉起来"的Provider,标上`@ShellCreateTrigger`,再让它们被某个必然被构建的对象依赖到。

### 4.4 DynamicOverride:产品覆盖基类实现

有些基类组件在不同产品上行为不同。`docs/dagger.md`给的建议是优先在组件里加hook;实在不行才用`@DynamicOverride`:

> The alternative is to use the `@DynamicOverride` annotation to allow the product module to provide an implementation that the base module can reference. This is most useful if the existence of the entire component is controlled by the product and the override implementation is optional (there is a default implementation).

产品模块(如OEM的SystemUI变体)提供一个实现并标`@DynamicOverride`,基类模块的Provider就会拿到这个覆盖版本。

`docs/extending.md`里还有一条硬性禁忌:

> **Don't** override classes provided by WMShellBaseModule, it makes it difficult to make simple changes to the Shell library base modules which are shared by all products.
> - If possible add mechanisms to modify the base class behavior

### 4.5 新增一个Shell组件的完整步骤

这部分直接来自官方文档`docs/changes.md`,是最实用的"照做就行"清单。

**只给Shell内部用的工具类**:直接加,放`com.android.wm.shell.common`之类的包里,补单元测试。

**给Shell内部其它组件用的组件**:

1. 建一个自己的包,按约定写一个`XxxController`负责初始化。
2. 决定放哪一层:`WMShellBaseModule`(基类和产品组件都会依赖)还是`WMShellModule`/`TvWMShellModule`(产品专有,基类组件不依赖它)。

**要给SystemUI用的组件**(需要两边互通):

1. 定义一个平行的`Optional<Xxx>`接口,由`XxxController`实现,并在Shell主线程上处理(见第5节)。
2. 因为暴露给SystemUI注入的组件是显式列出来的,所以要在`WMComponent`里加一个`getXxx()`,并更新`SysUIComponent`的`Builder`,让它可以被注入。
3. 两者的绑定在`SystemUIFactory#init()`里完成。

**要给外部进程(如Launcher)用的组件**:

1. 在controller里写一个实现该接口binder的`Stub`类,继承`ExternalInterfaceBinder`,实现`invalidate()`以保证它不会长期持有外层controller的引用。
2. 让controller实现`RemoteCallable<T>`,所有进来的调用走`executeRemoteCallWithTaskPermission()`,以校验调用方身份并保证在Shell主线程上执行。
3. 注入`ShellController`,用`addExternalInterface()`注册。
4. 在Launcher侧更新`TouchInteractionService`,把接口交给它的代理对象。

**初始化组件**:

- Shell侧:注入`ShellInit`并加init回调;确保组件在Dagger图里。方法有两个——让它成为某个已暴露给SystemUI的组件的依赖,或者显式加到`WMShellBaseModule`的`@ShellCreateTrigger`Provider里。
- SystemUI侧:更新`WMShell`,把依赖SystemUI代码的绑定接上。

官方还给了两条"不要":

> - **Don't** do initialization in the constructor, only do initialization in the init callbacks. Otherwise it complicates the building of the dependency graph.
> - **Don't** create dependencies from base-module components on specific features (the base module is intended for use with all products). Try adding a mechanism to register and listen for changes from the base module component instead.
> - **Don't** add blocking synchronous calls in the SysUI interface between Shell & SysUI. Try adding a push-mechanism to share data, or an async callback to request data.

### 4.6 SystemUI侧对CoreStartable的约定

同样的一套规范在SystemUI侧由`packages/SystemUI/docs/corestartable.md`给出。`CoreStartable`的定义(`packages/SystemUI/src/com/android/systemui/CoreStartable.java`):

```java
/**
 * Code that needs to be run when SystemUI is started.
 *
 * Which CoreStartable modules are loaded is controlled via the dagger graph. Bind them into the
 * CoreStartable map with code such as:
 *
 *  @Binds
 *  @IntoMap
 *  @ClassKey(FoobarStartable::class)
 *  abstract fun bind(impl: FoobarStartable): CoreStartable
 *
 * If your CoreStartable depends on different CoreStartables starting before it, you can specify
 * another map binding listing out its dependencies:
 *
 *  @Provides
 *  @IntoMap
 *  @Dependencies  // Important! com.android.systemui.startable.Dependencies.
 *  @ClassKey(FoobarStartable::class)
 *  fun providesDeps(): Set<Class<out CoreStartable>> {
 *      return setOf(OtherStartable::class.java)
 *  }
 */
public interface CoreStartable extends Dumpable {
    String STARTABLE_DEPENDENCIES = "startable_dependencies";

    /** Main entry point for implementations. Called shortly after SysUI startup. */
    void start();

    @Override
    default void dump(@NonNull PrintWriter pw, @NonNull String[] args) {
    }

    /** Called to determine if the dumpable should be registered as critical or normal priority */
    default boolean isDumpCritical() {
        return true;
    }

    /** Called immediately after the system broadcasts ACTION_LOCKED_BOOT_COMPLETED or during
     * SysUI startup if the property {@code sys.boot_completed} is already set to 1. ... */
    default void onBootCompleted() {
    }

    /** No op implementation that can be used when feature flagging on the Dagger Module level. */
    CoreStartable NOP = new Nop();

    class Nop implements CoreStartable {
        @Override
        public void start() {}
    }
}
```

定义一个新`CoreStartable`的步骤(官方文档):

1. 继承`CoreStartable`,把初始化逻辑放进`start()`。最好放到自己的包和目录里。
2. 类标`@SysUISingleton`,构造函数标`@Inject`。
3. 在同一个包里写一个Dagger模块,命名遵循`Start<Feature>Module`。
4. 模块里写:

```java
@Binds
@IntoMap
@ClassKey(Feature.class)
abstract CoreStartable bindFeature(Feature impl);
```

5. 把模块include进需要它的地方。对AOSP来说就是`SystemUICoreStartableModule`。

两条经验:

- **一个`CoreStartable`只做一件事**。如果需要一段启动逻辑找不到合适的归属,就新建一个,而不是塞进已有的某个类。
- **`CoreStartable`之间应当无序依赖**。默认的顺序是固定但无保证的,AOSP不承诺未来不改。如果确实需要别人先起来,就给自己加监听器,而不是依赖启动顺序。

`Dependencies.kt`(`packages/SystemUI/src/com/android/systemui/startable/Dependencies.kt`)就是上面`@Dependencies`注解的实现,它让`SystemUIApplication`能做那次拓扑排序。

注意`CoreStartable extends Dumpable`,`Dumpable`(`packages/SystemUI/src/com/android/systemui/Dumpable.java`)是`dumpsys`的注册单元,`isDumpCritical()`决定它在bugreport里是critical还是普通优先级。

## 5 线程模型

### 5.1 五条线程

WMShell和SystemUI在同一个进程里,但它们不共用一条主线程。`docs/threading.md`给出了一张边界图:

```text
                                    Thread boundary
                                          |
                                WM Shell  | SystemUI
                                          |
                                          |
FeatureController <-> FeatureInterface <--|--> WMShell <-> SysUI
       |          (^post to shell thread) |            (^post to main thread)
      ...                                 |
       |                                  |
 OtherControllers                         |
```

Shell里实际存在的线程:

| 线程名 | 优先级 | 何时存在 | 用途 |
|---|---|---|---|
| SystemUI主线程 | 标准main | 总是 | 状态栏、通知面板等SystemUI的UI |
| `wmshell.main` | `THREAD_PRIORITY_DISPLAY` | 仅当`config_enableShellMainThread`为true | Shell的"主线程",几乎所有Shell逻辑和Shell自己创建的UI都在这 |
| `wmshell.anim` | `THREAD_PRIORITY_DISPLAY` | 总是 | 过渡动画、分屏动画 |
| `wmshell.splashscreen` | `THREAD_PRIORITY_TOP_APP_BOOST` | 总是 | 启动闪屏(starting window) |
| `wmshell.background` | `THREAD_PRIORITY_BACKGROUND`,可boost到`THREAD_PRIORITY_FOREGROUND` | 总是 | 耗时较长、不应阻塞Shell主线程的任务 |

对应的构造代码(`libs/WindowManager/Shell/src/com/android/wm/shell/dagger/WMShellConcurrencyModule.java`):

```java
public static boolean enableShellMainThread(Context context) {
    return context.getResources().getBoolean(R.bool.config_enableShellMainThread);
}

public static HandlerThread createShellMainThread() {
    HandlerThread mainThread = new HandlerThread("wmshell.main", THREAD_PRIORITY_DISPLAY);
    ...
}

HandlerThread shellAnimationThread = new HandlerThread("wmshell.anim",
        THREAD_PRIORITY_DISPLAY);
HandlerThread shellSplashscreenThread = new HandlerThread("wmshell.splashscreen",
        THREAD_PRIORITY_TOP_APP_BOOST);
HandlerThread shellBackgroundThread = new HandlerThread("wmshell.background",
        THREAD_PRIORITY_BACKGROUND);
```

`wmshell.main`优先级的注释值得抄一遍:

> **Note**: This thread runs with `THREAD_PRIORITY_DISPLAY` priority since so many windowing-critical components depend on it.

`wmshell.main`和`wmshell.anim`都用`THREAD_PRIORITY_DISPLAY`,因为它们直接决定窗口反应的快慢。

还要注意,这些Looper都被设了慢消息阈值:

```java
private static final int MSGQ_SLOW_DELIVERY_THRESHOLD_MS = 30;
private static final int MSGQ_SLOW_DISPATCH_THRESHOLD_MS = 30;

mainThread.getLooper().setSlowLogThresholdMs(MSGQ_SLOW_DISPATCH_THRESHOLD_MS,
        MSGQ_SLOW_DELIVERY_THRESHOLD_MS);
```

也就是说,Shell主线程上单条消息处理超过30ms就会在logcat里打出`Slow dispatch`/`Slow delivery`警告。调Shell卡顿问题时,这是第一个该看的信号。

**如果产品关闭了`config_enableShellMainThread`**,`wmshell.main`就不存在,Shell回退到SystemUI主线程。此时Shell和SystemUI共用一条线程,`ShellExecutor`和`@Main Executor`指向同一个Looper。

### 5.2 双向调用的post规则

Shell与SystemUI互相调用时,约定是**接口的提供方负责post到自己的线程**,调用方只管调。这个约定由`@ExternalThread`注解标记,例如:

```java
// libs/WindowManager/Shell/shared/src/com/android/wm/shell/shared/ShellTransitions.java
@ExternalThread
public interface ShellTransitions {
    // ...
}
```

`ShellController`的实现是这个约定的样板:

```java
@ExternalThread
private class ShellInterfaceImpl implements ShellInterface {
    @Override
    public void onInit() {
        mMainExecutor.execute(ShellController.this::handleInit);
    }

    @Override
    public void onConfigurationChanged(Configuration newConfiguration) {
        mMainExecutor.execute(() ->
                ShellController.this.onConfigurationChanged(newConfiguration));
    }
    // ...
}
```

SystemUI主线程调`onConfigurationChanged`,Shell在自己的主线程上处理。

反方向,Shell里需要回调SystemUI时,**回调必须带一个`Executor`参数**,由SystemUI决定在哪个线程上跑。例子:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/sysui/ShellInterface.java
default void addDisplayImeChangeListener(DisplayImeChangeListener listener,
        Executor executor) {}

default void setFocusTransitionListener(@NonNull FocusTransitionListener listener,
        Executor executor) {}
```

### 5.3 几条硬性禁忌

`docs/threading.md`的"Best practices"里列了几条,都是踩过坑总结的:

- **构造函数里不做初始化**。因为SystemUI可能在某些线程上构造Shell组件,构造期做的事很可能跑错线程。初始化统一放`ShellInit`回调。
- **谨慎使用`CountDownLatch`等阻塞同步**。如果Shell主线程其实就是SystemUI主线程,阻塞等待Shell主线程就会死锁。
- **任何非Shell组件的回调都要post到Shell主线程**。包括Binder调用、SystemUI调用、广播接收器。
- **不需要显式`synchronize`**。因为Shell里所有东西都跑在Shell主线程上,除非要防重入。
- **一般不要自己new Handler**,注入`@ShellMainThread ShellExecutor`。因为`new Handler()`绑定的是当前Looper,构造时机不对就会绑错。
- **绝对不要用`Looper.getMainLooper()`**。它在Shell里几乎总是错的那个,要拿SystemUI主线程就注入`@Main ShellExecutor`。

有一处例外很难避免:`handleCommand`这种需要返回值的命令处理,只能用`executeBlocking`:

```java
// ShellController.java
@Override
public boolean handleCommand(String[] args, PrintWriter pw) {
    try {
        boolean[] result = new boolean[1];
        mMainExecutor.executeBlocking(() -> {
            result[0] = mShellCommandHandler.handleCommand(args, pw);
        });
        return result[0];
    } catch (InterruptedException e) {
        throw new RuntimeException("Failed to handle Shell command in 2s", e);
    }
}
```

这就是为什么Shell的命令行处理超时时间是2秒。

## 6 Shell与WMCore的边界

WMShell跑在SystemUI进程,WMCore跑在system_server。两边靠Binder通信,主要有四类通道。

### 6.1 通道一:TaskOrganizer——把task的"表象"交给Shell

`TaskOrganizer`(`core/java/android/window/TaskOrganizer.java`)是framework提供的抽象类,一个进程注册成organizer之后,可以接管task的surface组织。

Shell侧的统一实现是`ShellTaskOrganizer`(`libs/WindowManager/Shell/src/com/android/wm/shell/ShellTaskOrganizer.java`):

```java
/**
 * Unified task organizer for all components in the shell.
 * TODO(b/167582004): may consider consolidating this class and TaskOrganizer
 */
public class ShellTaskOrganizer extends TaskOrganizer {
    private static final String TAG = "ShellTaskOrganizer";

    // Intentionally using negative numbers here so the positive numbers can be used
    // for task id specific listeners that will be added later.
    public static final int TASK_LISTENER_TYPE_UNDEFINED = -1;
    public static final int TASK_LISTENER_TYPE_FULLSCREEN = -2;
    public static final int TASK_LISTENER_TYPE_MULTI_WINDOW = -3;
    public static final int TASK_LISTENER_TYPE_PIP = -4;
    public static final int TASK_LISTENER_TYPE_FREEFORM = -5;
```

用负数做key的原因注释写得明白:正数留给"按taskId注册的监听器"。所以`mTaskListeners`(一个`SparseArray<TaskListener>`)的key既可能是taskId(正数),也可能是listener type(负数)。

`TaskListener`定义:

```java
public interface TaskListener {
    default void onTaskAppeared(RunningTaskInfo taskInfo, SurfaceControl leash) {}
    default void onTaskInfoChanged(RunningTaskInfo taskInfo) {}
    default void onTaskVanished(RunningTaskInfo taskInfo) {}
    default void onBackPressedOnTaskRoot(RunningTaskInfo taskInfo) {}
    /** Whether this task listener supports compat UI. */
    default boolean supportCompatUI() {
        // All TaskListeners should support compat UI except PIP and StageCoordinator.
        return true;
    }
    /** Attaches a child window surface to the task surface. */
    default void attachChildSurfaceToTask(int taskId, SurfaceControl.Builder b) {
        throw new IllegalStateException(
                "This task listener doesn't support child surface attachment.");
    }
    /** Reparents a child window surface to the task surface. */
    default void reparentChildSurfaceToTask(int taskId, SurfaceControl sc,
            SurfaceControl.Transaction t) {
        throw new IllegalStateException(
                "This task listener doesn't support child surface reparent.");
    }
    default void dump(@NonNull PrintWriter pw, String prefix) {};
}
```

**分发优先级**在`getTaskListener`里:

```java
private TaskListener getTaskListener(RunningTaskInfo runningTaskInfo,
        boolean removeLaunchCookieIfNeeded) {
    final int taskId = runningTaskInfo.taskId;
    TaskListener listener;
    // First priority goes to listener that might be pending for this task.
    final ArrayList<IBinder> launchCookies = runningTaskInfo.launchCookies;
    for (int i = launchCookies.size() - 1; i >= 0; --i) {
        final IBinder cookie = launchCookies.get(i);
        listener = mLaunchCookieToListener.get(cookie);
        if (listener == null) continue;
        if (removeLaunchCookieIfNeeded) {
            mLaunchCookieToListener.remove(cookie);
            mTaskListeners.put(taskId, listener);
        }
        return listener;
    }
    // Next priority goes to taskId specific listeners.
    listener = mTaskListeners.get(taskId);
    if (listener != null) return listener;
    // Next priority goes to the listener listening to its parent.
    if (runningTaskInfo.hasParentTask()) {
        listener = mTaskListeners.get(runningTaskInfo.parentTaskId);
        if (listener != null) return listener;
    }
    // Next we try type specific listeners.
    final int taskListenerType = taskInfoToTaskListenerType(runningTaskInfo);
    return mTaskListeners.get(taskListenerType);
}
```

四级优先级:**launch cookie → taskId → parentTaskId → windowing mode对应的类型**。

`launch cookie`是第一优先级,这是"先占坑"机制:模块要启动一个新task时,先申请一个cookie注册好listener,等这个task真的出现,就能保证由自己接管,而不是被类型监听器抢走。

windowing mode到listener类型:

```java
static @TaskListenerType int taskInfoToTaskListenerType(RunningTaskInfo runningTaskInfo) {
    switch (runningTaskInfo.getWindowingMode()) {
        case WINDOWING_MODE_FULLSCREEN:   return TASK_LISTENER_TYPE_FULLSCREEN;
        case WINDOWING_MODE_MULTI_WINDOW: return TASK_LISTENER_TYPE_MULTI_WINDOW;
        case WINDOWING_MODE_PINNED:       return TASK_LISTENER_TYPE_PIP;
        case WINDOWING_MODE_FREEFORM:     return TASK_LISTENER_TYPE_FREEFORM;
        case WINDOWING_MODE_UNDEFINED:
        default:                          return TASK_LISTENER_TYPE_UNDEFINED;
    }
}
```

某个模块要接管一类task,就把自己注册进去:

```java
public void addListenerForType(TaskListener listener, @TaskListenerType int... listenerTypes) {
    synchronized (mLock) {
        for (int listenerType : listenerTypes) {
            if (mTaskListeners.get(listenerType) != null) {
                throw new IllegalArgumentException("Listener for listenerType=" + listenerType
                        + " already exists");
            }
            mTaskListeners.put(listenerType, listener);
        }

        // Notify the listener of all existing tasks with the given type.
        for (int i = mTasks.size() - 1; i >= 0; --i) {
            final TaskAppearedInfo data = mTasks.valueAt(i);
            final TaskListener taskListener = getTaskListener(data.getTaskInfo());
            if (taskListener != listener) continue;
            listener.onTaskAppeared(data.getTaskInfo(), data.getLeash());
        }
    }
}
```

注意最后那段:**注册时会立刻把已存在的同类task补报一遍**。所以模块的注册时机不影响它能否看到已有task。但这只在"每个类型只能有一个监听器"的前提下成立,重复注册同一个类型会直接抛`IllegalArgumentException`。

`FullscreenTaskListener`是注册样板(`libs/WindowManager/Shell/src/com/android/wm/shell/fullscreen/FullscreenTaskListener.java`):

```java
public class FullscreenTaskListener implements ShellTaskOrganizer.TaskListener {
    private final ShellTaskOrganizer mShellTaskOrganizer;

    private final SparseArray<State> mTasks = new SparseArray<>();

    private static class State {
        RunningTaskInfo mTaskInfo;
        SurfaceControl mLeash;
    }
    // ...

    public FullscreenTaskListener(ShellInit shellInit,
            ShellTaskOrganizer shellTaskOrganizer,
            SyncTransactionQueue syncQueue,
            Optional<RecentTasksController> recentTasksOptional,
            Optional<WindowDecorViewModel> windowDecorViewModelOptional) {
        mShellTaskOrganizer = shellTaskOrganizer;
        mSyncQueue = syncQueue;
        mRecentTasksOptional = recentTasksOptional;
        mWindowDecorViewModelOptional = windowDecorViewModelOptional;
        // Note: Some derivative FullscreenTaskListener implementations do not use ShellInit
        if (shellInit != null) {
            shellInit.addInitCallback(this::onInit, this);
        }
    }

    private void onInit() {
        mShellTaskOrganizer.addListenerForType(this, TASK_LISTENER_TYPE_FULLSCREEN);
    }
    // ...
}
```

它内部用`SparseArray<State>`按taskId记住每个task的`RunningTaskInfo`和`leash`,因为后续的`onTaskInfoChanged`/`onTaskVanished`只给taskInfo,需要自己维护映射。

**注册到system_server的过程**在`ShellTaskOrganizer.onInit`:

```java
private void onInit() {
    mShellCommandHandler.addDumpCallback(this::dump, this);
    // ...
    registerOrganizer();
}

@Override
public List<TaskAppearedInfo> registerOrganizer() {
    synchronized (mLock) {
        ProtoLog.v(WM_SHELL_TASK_ORG, "Registering organizer");
        final List<TaskAppearedInfo> taskInfos = super.registerOrganizer();
        for (int i = 0; i < taskInfos.size(); i++) {
            final TaskAppearedInfo info = taskInfos.get(i);
            // ...
            onTaskAppeared(info);
        }
        return taskInfos;
    }
}
```

`super.registerOrganizer()`走的是`WindowOrganizer`的binder调用,向system_server的`WindowOrganizerController`注册`ITaskOrganizer`,并把**当前所有已存在task**的`TaskAppearedInfo`列表同步返回。

### 6.2 通道二:DisplayAreaOrganizer——接管一块显示区域

`DisplayAreaOrganizer`(`core/java/android/window/DisplayAreaOrganizer.java`)让Shell侧拥有自己的`DisplayArea`(显示区域容器),可以在里面放窗口surface。

Shell里有三个organizer:

| 类 | 路径 | 作用 |
|---|---|---|
| `RootTaskDisplayAreaOrganizer` | `libs/WindowManager/Shell/src/com/android/wm/shell/RootTaskDisplayAreaOrganizer.java` | 管理root/default的TaskDisplayArea(如默认显示区、分屏的两个stage其实挂在它下面) |
| `RootDisplayAreaOrganizer` | `libs/WindowManager/Shell/src/com/android/wm/shell/RootDisplayAreaOrganizer.java` | 管理root display area(每个物理display一个) |
| `OneHandedDisplayAreaOrganizer` | `libs/WindowManager/Shell/src/com/android/wm/shell/onehanded/OneHandedDisplayAreaOrganizer.java` | 单手模式的整体位移 |

两者类注释很短:

```java
/** Display area organizer for the root/default TaskDisplayAreas */
public class RootTaskDisplayAreaOrganizer extends DisplayAreaOrganizer { }
```

```java
/** Display area organizer for the root display areas */
public class RootDisplayAreaOrganizer extends DisplayAreaOrganizer { }
```

`OneHandedDisplayAreaOrganizer`的类注释信息量大一些,说明了"动画用Transaction、收尾用WCT"这个通用套路:

> This class listens on `DisplayAreaOrganizer` callbacks for windowing mode change both to and from OneHanded and issues corresponding animation if applicable.
> Normally, we apply series of `SurfaceControl.Transaction` when the animator is running and files a final `WindowContainerTransaction` at the end of the transition.

**这是Shell里做动画的标准模式**:动画过程中用轻量的`SurfaceControl.Transaction`直接改surface的位置/透明度;动画结束时提交一个`WindowContainerTransaction`(WCT),让WMCore把窗口树状态真正改到最终形态。

需要用`DisplayArea`的模块通过`DisplayAreaHelper`拿到入口:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/displayareahelper/DisplayAreaHelperController.java
public class DisplayAreaHelperController implements DisplayAreaHelper {

    private final Executor mExecutor;
    private final RootDisplayAreaOrganizer mRootDisplayAreaOrganizer;

    public DisplayAreaHelperController(Executor executor,
            RootDisplayAreaOrganizer rootDisplayAreaOrganizer) {
        mExecutor = executor;
        mRootDisplayAreaOrganizer = rootDisplayAreaOrganizer;
    }

    @Override
    public void attachToRootDisplayArea(int displayId, SurfaceControl.Builder builder,
            Consumer<SurfaceControl.Builder> onUpdated) {
        mExecutor.execute(() -> {
            mRootDisplayAreaOrganizer.attachToDisplayArea(displayId, builder);
            onUpdated.accept(builder);
        });
    }
}
```

`DisplayAreaHelper`是`WMComponent.getDisplayAreaHelper()`导出的13个接口之一,所以SystemUI侧也能用。

### 6.3 通道三:PinnedStackListenerForwarder——残留的旧接口

`WindowManagerShellWrapper`是Shell和WMS之间的一个"旧桥"。它的类注释:

```java
/**
 * The singleton wrapper to communicate between WindowManagerService and WMShell features
 * (e.g: PIP, SplitScreen, Bubble, OneHandedMode...etc)
 * TODO: Remove once PinnedStackListenerForwarder can be removed
 */
public class WindowManagerShellWrapper {
    private final PinnedStackListenerForwarder mPinnedStackListenerForwarder;

    public WindowManagerShellWrapper(ShellExecutor mainExecutor) {
        mPinnedStackListenerForwarder = new PinnedStackListenerForwarder(mainExecutor);
    }

    public void addPinnedStackListener(PinnedTaskListener listener) throws RemoteException {
        mPinnedStackListenerForwarder.addListener(listener);
        mPinnedStackListenerForwarder.register(DEFAULT_DISPLAY);
    }

    public void removePinnedStackListener(PinnedTaskListener listener) {
        mPinnedStackListenerForwarder.removeListener(listener);
    }
}
```

它本身只是个转发器,存在的理由是WMS的旧接口只允许注册一个监听器:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/pip/PinnedStackListenerForwarder.java
/**
 * PinnedStackListener that simply forwards all calls to each listener added via
 * {@link #addListener}. This is necessary since calling
 * {@link com.android.server.wm.WindowManagerService#registerPinnedTaskListener} replaces any
 * previously set listener.
 */
```

真正注册到WMS:

```java
public void register(int displayId) throws RemoteException {
    WindowManagerGlobal.getWindowManagerService().registerPinnedTaskListener(
            displayId, mListenerImpl);
}
```

目前的主要使用者是气泡(`BubbleController`),它靠这个接口拿到IME高度变化:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/bubbles/BubbleController.java
try {
    mWindowManagerShellWrapper.addPinnedStackListener(new BubblesImeListener());
} catch (RemoteException e) {
    e.printStackTrace();
}

/** PinnedStackListener that dispatches IME visibility updates to the stack. */
private class BubblesImeListener extends PinnedStackListenerForwarder.PinnedTaskListener {
    @Override
    public void onImeVisibilityChanged(boolean imeVisible, int imeHeight) {
        mBubblePositioner.setImeVisible(imeVisible, imeHeight);
        // ...
    }
}
```

注意类名里带`PinnedStack`,这是画中画早期叫法的遗留。`onImeVisibilityChanged`挂在这个接口上属于历史包袱,类注释里的TODO就是打算清掉它。

### 6.4 通道四:直接注入IWindowManager

除上述三类,Shell组件也可以直接注入`IWindowManager`,用于查询显示信息、注册显示变化监听等。例如`DisplayController`:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/common/DisplayController.java
/**
 * This module deals with display rotations coming from WM. When WM starts a rotation: after it has
 * frozen the screen, it will call into this class. This will then call all registered local
 * controllers and give them a chance to queue up task changes to be applied synchronously with that
 * rotation.
 */
public class DisplayController {
    private final ShellExecutor mMainExecutor;
    private final Context mContext;
    private final IWindowManager mWmService;
    private final DisplayChangeController mChangeController;
    // ...
}
```

### 6.5 SyncTransactionQueue:同步事务的串行化

上面提到"动画用`SurfaceControl.Transaction`,收尾用WCT"。但有些场景必须让surface的变化和窗口树的变化**同步生效**,否则会闪。这时用`SyncTransactionQueue`。

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/common/SyncTransactionQueue.java
/**
 * Helper for serializing sync-transactions and corresponding callbacks.
 */
public final class SyncTransactionQueue {
    private static final boolean DEBUG = false;
    private static final String TAG = "SyncTransactionQueue";

    // Just a little longer than the sync-engine timeout of 5s
    private static final int REPLY_TIMEOUT = 5300;

    private final TransactionPool mTransactionPool;
    private final ShellExecutor mMainExecutor;

    // Sync Transactions currently don't support nesting or interleaving properly, so
    // queue up transactions to run them serially.
    private final ArrayList<SyncCallback> mQueue = new ArrayList<>();

    private SyncCallback mInFlight = null;
    private final ArrayList<TransactionRunnable> mRunnables = new ArrayList<>();

    private final Runnable mOnReplyTimeout = () -> {
        synchronized (mQueue) {
            if (mInFlight != null && mQueue.contains(mInFlight)) {
                Slog.w(TAG, "Sync Transaction timed-out: " + mInFlight.mWCT);
                mInFlight.onTransactionReady(mInFlight.mId, new SurfaceControl.Transaction());
            }
        }
    };

    public SyncTransactionQueue(TransactionPool pool, ShellExecutor mainExecutor) {
        mTransactionPool = pool;
        mMainExecutor = mainExecutor;
    }

    /**
     * Queues a sync transaction to be sent serially to WM.
     */
    public void queue(WindowContainerTransaction wct) {
        // ...
    }
}
```

几个设计点:

- 同步事务**不支持嵌套和交错**,所以用`mQueue`串行化,`mInFlight`记录当前在途的那一个。
- 超时时间是5300ms,比WM同步引擎本身的5秒超时"稍长一点"。超时后不再等WM回调,而是直接用一个空的Transaction把回调走完,避免Shell卡死。
- `TransactionPool`用来复用`SurfaceControl.Transaction`对象,减少GC压力。它定义在`Shell/shared`里(`TransactionPool.java`),因为SystemUI也要用。

### 6.6 进程边界总结

| 组件 | 所在进程 | 角色 |
|---|---|---|
| `WindowManagerService` | `system_server` | 窗口策略与状态真相 |
| `WindowOrganizerController` | `system_server` | task/displayArea注册与事务入口 |
| `TransitionController` | `system_server` | 发起、收集、结束transition |
| `DisplayPolicy` | `system_server` | 决定系统栏窗的可见性与insets |
| `ShellTaskOrganizer` | `com.android.systemui` | 唯一的TaskOrganizer,分发到各模块 |
| `RootTaskDisplayAreaOrganizer`等 | `com.android.systemui` | DisplayArea的组织 |
| `Transitions` | `com.android.systemui` | 过渡动画的状态机 |
| `WindowManagerShellWrapper` | `com.android.systemui` | 转发旧的PinnedStack监听 |
| `WMShell`(SystemUI侧) | `com.android.systemui` | Shell与SystemUI的事件桥 |

一个容易搞混的点:**`WindowManagerShellWrapper`虽然名字里有WindowManager,它在Shell库里、跑在SystemUI进程**;而`com.android.server.wm`包里那些才是system_server侧。

## 7 Shell Transition

Shell Transition是WMShell最核心的一块。它把原来散落在WMCore里的窗口动画逻辑,整体搬到了SystemUI进程。

### 7.1 为什么要有Shell Transition

在Shell Transition之前(也就是Legacy模式),一次Activity跳转的动画由WMCore和App共同完成:WMCore把窗口布局算好,App侧自己撸`Animation`,`AppTransitionController`再撮合。问题是:

- 动画代码分散在WMCore、App、Launcher三处,时序难以统一。
- Launcher要给跳转做转场动画,只能靠`RemoteAnimationAdapter`这种一次性接口,拿到一堆surface自己摆,拿不到窗口树的结构信息。
- 一次跳转涉及多个窗口(启动窗口、壁纸、IME、状态栏)时,没人负责统一编排。

Shell Transition的思路是:**由WMCore把"这次变化是什么"整理成一个数据结构,交给Shell,Shell来演**。

- WMCore负责**算**:哪些容器动了、动到哪、谁覆盖谁。算完打包成`TransitionInfo`。
- Shell负责**演**:把`TransitionInfo`交给一堆`TransitionHandler`,谁认得这个场景谁就来演,演完回调通知WMCore收尾。

`ShellTransitions.java`(在`Shell/shared`里)是Shell对外(主要是SystemUI和Launcher)暴露的接口:

```java
@ExternalThread
public interface ShellTransitions {
    /** Registers a remote transition handler for all operations excluding takeovers. */
    void registerRemote(@NonNull TransitionFilter filter,
            @NonNull RemoteTransition remoteTransition);

    void registerRemoteForTakeover(@NonNull TransitionFilter filter,
            @NonNull RemoteTransition remoteTransition);

    void unregisterRemote(@NonNull RemoteTransition remoteTransition);

    /** Returns the apply-token used by transactions in Shell. */
    IBinder getShellApplyToken();

    /** Set listener that will receive callbacks about transitions involving home activity. */
    void setHomeTransitionListener(@NonNull HomeTransitionListener listener);

    /** Returns a container surface for the home root task. */
    SurfaceControl getHomeTaskOverlayContainer();

    /** Set listener that will receive callbacks about transitions involving focus switch. */
    void setFocusTransitionListener(@NonNull FocusTransitionListener listener);

    void unsetFocusTransitionListener(@NonNull FocusTransitionListener listener);
}
```

跨进程版本是`IShellTransitions.aidl`(`Shell/shared/src/com/android/wm/shell/shared/IShellTransitions.aidl`):

```aidl
interface IShellTransitions {
    oneway void registerRemote(in TransitionFilter filter, in RemoteTransition remoteTransition) = 1;
    oneway void unregisterRemote(in RemoteTransition remoteTransition) = 2;
    IBinder getShellApplyToken() = 3;
    oneway void setHomeTransitionListener(in IHomeTransitionListener listener) = 4;
    SurfaceControl getHomeTaskOverlayContainer() = 5;
    oneway void registerRemoteForTakeover(in TransitionFilter filter, in RemoteTransition remoteTransition) = 6;
    oneway void setFocusTransitionListener(in IFocusTransitionListener listener) = 7;
}
```

Launcher就是通过这个AIDL把Quickstep的转场动画注册进来的。

### 7.2 生命周期:PENDING → READY → ACTIVE

`Transitions.java`的类注释把状态机写得很清楚:

```text
Plays transition animations. Within this player, each transition has a lifecycle.
1. When a transition is directly started or requested, it is added to "pending" state.
2. Once WMCore applies the transition and notifies, the transition moves to "ready" state.
3. When a transition starts animating, it is moved to the "active" state.

Basically: --start--> PENDING --onTransitionReady--> READY --play--> ACTIVE --finish--> |
                                                           --merge--> MERGED --^
```

三个状态的对应字段:

```java
/** All transitions that we have created, but not yet finished. */
private final ArrayMap<IBinder, ActiveTransition> mKnownTransitions = new ArrayMap<>();

/** Keeps track of transitions which have been started, but aren't ready yet. */
private final ArrayList<ActiveTransition> mPendingTransitions = new ArrayList<>();

/**
 * Transitions which are ready to play, but haven't been sent to a track yet because a sync
 * is ongoing.
 */
private final ArrayList<ActiveTransition> mReadyDuringSync = new ArrayList<>();

private final ArrayList<Track> mTracks = new ArrayList<>();
```

`ActiveTransition`承载一次过渡的全部运行时信息:

```java
private static final class ActiveTransition {
    final IBinder mToken;

    TransitionHandler mHandler;
    boolean mAborted;
    TransitionInfo mInfo;
    SurfaceControl.Transaction mStartT;
    SurfaceControl.Transaction mFinishT;

    /** Ordered list of transitions which have been merged into this one. */
    private ArrayList<ActiveTransition> mMerged;

    ActiveTransition(IBinder token) {
        mToken = token;
    }

    boolean isSync() {
        return (mInfo.getFlags() & TransitionInfo.FLAG_SYNC) != 0;
    }

    int getTrack() {
        return mInfo != null ? mInfo.getTrack() : -1;
    }

    @Override
    public String toString() {
        if (mInfo != null && mInfo.getDebugId() >= 0) {
            return "(#" + mInfo.getDebugId() + ") " + mToken + "@" + getTrack();
        }
        return mToken.toString() + "@" + getTrack();
    }
}
```

`toString()`的输出格式`(#debugId) token@trackId`是看ProtoLog和trace时最常打交道的字符串。

### 7.3 Track:并行播放的串行轨道

**一次过渡在一条Track里是串行的,但不同Track可以并行。** 这是理解Shell Transition的关键。

```java
private static class Track {
    /** Keeps track of transitions which are ready to play but still waiting for their turn. */
    final ArrayList<ActiveTransition> mReadyTransitions = new ArrayList<>();

    /** The currently playing transition in this track. */
    ActiveTransition mActiveTransition = null;

    boolean isIdle() {
        return mActiveTransition == null && mReadyTransitions.isEmpty();
    }
}
```

一条Track同一时刻只能有一个`mActiveTransition`。后到的过渡先进入`mReadyTransitions`排队,等当前这个演完才能上。

**Track是谁分配的?** 类注释:

> Track assignments are expected to be provided by WMCore and this generally tries to maintain the same assignments. If, however, WMCore decides that a transition conflicts with >1 active track, it will be marked as SYNC. This means that all currently active tracks must be flushed before the SYNC transition can play.

TrackId来自`TransitionInfo.getTrack()`,由WMCore给定。典型情况下两个display各一条Track,互不干扰。

当WMCore发现一次过渡同时影响了多条正在播放的Track时,会给它打上`FLAG_SYNC`。SYNC过渡必须等所有Track都排空才能播。`dispatchReady`里处理这件事:

```java
boolean dispatchReady(ActiveTransition active) {
    final TransitionInfo info = active.mInfo;

    if (info.getType() == TRANSIT_SLEEP || active.isSync()) {
        // Adding to *front*! If we are here, it means that it was pulled off the front
        // so we are just putting it back; or, it is the first one so it doesn't matter.
        mReadyDuringSync.add(0, active);
        boolean hadPreceding = false;
        // Now flush all the tracks.
        for (int i = 0; i < mTracks.size(); ++i) {
            final Track tr = mTracks.get(i);
            if (tr.isIdle()) continue;
            hadPreceding = true;
            // Sleep starts a process of forcing all prior transitions to finish immediately
            ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS,
                    "Start finish-for-sync track %d", i);
            finishForSync(active.mToken, i, null /* forceFinish */);
        }
        if (hadPreceding) {
            return false;
        }
        // Actually able to process the sleep now, so re-remove it from the queue and continue
        ...
```

`TRANSIT_SLEEP`(熄屏)也走同一条路:先把所有在播的过渡**强制立即结束**,再播息屏动画。

### 7.4 Merge:把新过渡并进正在播的那个

上一节说Track内是串行的,但有一个例外:**新的过渡可以合并进正在播的那个,而不是排队等**。这是"动画不被打断"的关键。

类注释:

> While a transition is animating in a track, transitions dispatched to the track will be queued in the "ready" state for their turn. At the same time, whenever a transition makes it to the head of the "ready" queue, it will attempt to merge to with the "active" transition. If the merge succeeds, it will be moved to the "active" transition's "merged" list and then the next "ready" transition can attempt to merge. Once the "active" transition animation is finished, the next "ready" transition can play.

合并不成再排队,合并成功就直接并进去。`ActiveTransition.mMerged`就是这个"已合并列表"。

Handler通过`mergeAnimation`决定是否接受合并:

```java
/**
 * Attempts to merge a different transition's animation into an animation that this handler
 * is currently playing. If a merge is not possible/supported, this should be a no-op.
 *
 * This gets called if another transition becomes ready while this handler is still playing
 * an animation. This is called regardless of whether this handler claims to support that
 * particular transition or not.
 *
 * When this happens, there are 2 options:
 *  1. Do nothing. This effectively rejects the merge request. This is the "safest" option.
 *  2. Merge the incoming transition into this one. The implementation is up to this
 *     handler. To indicate that this handler has "consumed" the merge transition, it
 *     must call the finishCallback immediately, or at-least before the original
 *     transition's finishCallback is called.
 */
default void mergeAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
        @NonNull SurfaceControl.Transaction t, @NonNull IBinder mergeTarget,
        @NonNull TransitionFinishCallback finishCallback) { }
```

用户快速连点App图标时,"打开A"和"打开B"两次过渡就是这样处理的:第二次直接并进第一次的动画里。

### 7.5 Handler:谁能演谁上

Handler列表按"特异性"排序,**从后往前扫描**,谁先返回true谁就演:

```java
/** List of possible handlers. Ordered by specificity (eg. tapped back to front). */
private final ArrayList<TransitionHandler> mHandlers = new ArrayList<>();
```

构造时先放两个:

```java
// The very last handler (0 in the list) should be the default one.
mHandlers.add(mDefaultTransitionHandler);
ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, "addHandler: Default");
// Next lowest priority is remote transitions.
mHandlers.add(mRemoteTransitionHandler);
ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, "addHandler: Remote");
```

即:

| 索引 | Handler | 优先级 |
|---|---|---|
| 0 | `DefaultTransitionHandler` | 兜底,一定能演 |
| 1 | `RemoteTransitionHandler` | 次低,处理Launcher等远端注册的过渡 |
| ≥2 | 各功能模块注册的,如`KeyguardTransitionHandler`、分屏的`SplitScreenTransitions` | 越后加优先级越高 |

新Handler从尾部追加:

```java
/**
 * Adds a handler candidate.
 * @see TransitionHandler
 */
public void addHandler(@NonNull TransitionHandler handler) {
    if (mHandlers.isEmpty()) {
        throw new RuntimeException("Unexpected handler added prior to initialization, please "
                + "use ShellInit callbacks to ensure proper ordering");
    }
    mHandlers.add(handler);
    // Set initial scale settings.
    handler.setAnimScaleSetting(mTransitionAnimationScaleSetting);
    ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, "addHandler: %s",
            handler.getClass().getSimpleName());
}
```

异常信息里的"please use ShellInit callbacks to ensure proper ordering"对应第3章讲的初始化顺序问题——`Transitions`自己的构造必须在`ShellInit`回调里跑,否则`mHandlers`还是空的。

`DefaultTransitionHandler`那一行注释**值得单独记住**:

> The very last handler (0 in the list) should be the default one.

因为有它兜底,`dispatchTransition`里的异常才"不应该发生":

```java
for (int i = mHandlers.size() - 1; i >= 0; --i) {
    if (mHandlers.get(i) == skip) {
        ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, " skip handler %s",
                mHandlers.get(i));
        continue;
    }
    boolean consumed = mHandlers.get(i).startAnimation(transition, info, startT, finishT,
            finishCB);
    if (consumed) {
        ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, " animated by %s",
                mHandlers.get(i));
        mTransitionTracer.logDispatched(info.getDebugId(), mHandlers.get(i));
        if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
            Trace.instant(TRACE_TAG_WINDOW_MANAGER,
                    mHandlers.get(i).getClass().getSimpleName()
                            + "#startAnimation animated "
                            + transitTypeToString(info.getType()));
        }
        return mHandlers.get(i);
    }
}
throw new IllegalStateException(
        "This shouldn't happen, maybe the default handler is broken.");
```

`skip`参数用于"重试"场景:某个Handler演到一半放弃,再扫一遍时跳过它,避免又选回同一个。

**请求阶段**(在过渡真正开始之前)也走同样的从后往前扫描,但调的是`handleRequest`,返回`WindowContainerTransaction`表示"我要参与,这些变化我先声明":

```java
/**
 * Gives every handler (in order) a chance to handle request until one consumes the transition.
 * @return the WindowContainerTransaction given by the handler which consumed the transition.
 */
public Pair<TransitionHandler, WindowContainerTransaction> dispatchRequest(
        @NonNull IBinder transition, @NonNull TransitionRequestInfo request,
        @Nullable TransitionHandler skip) {
    for (int i = mHandlers.size() - 1; i >= 0; --i) {
        if (mHandlers.get(i) == skip) continue;
        WindowContainerTransaction wct = mHandlers.get(i).handleRequest(transition, request);
        if (wct != null) {
            return new Pair<>(mHandlers.get(i), wct);
        }
    }
    return null;
}
```

`Transitions.java`里对`TransitionHandler`接口的注释,把`startAnimation`的两个Transaction语义讲得最细,这是写自定义Handler时最容易搞错的地方:

```java
/**
 * Starts a transition animation. This is always called if handleRequest returned non-null
 * for a particular transition. Otherwise, it is only called if no other handler before
 * it handled the transition.
 * @param startTransaction the transaction given to the handler to be applied before the
 *                         transition animation. Note the handler is expected to call on
 *                         {@link SurfaceControl.Transaction#apply()} for startTransaction.
 * @param finishTransaction the transaction given to the handler to be applied after the
 *                       transition animation. Unlike startTransaction, the handler is NOT
 *                       expected to apply this transaction. The Transition system will
 *                       apply it when finishCallback is called.
 * @param finishCallback Call this when finished. This MUST be called on main thread.
 * @return true if transition was handled, false if not (falls-back to default).
 */
boolean startAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
        @NonNull SurfaceControl.Transaction startTransaction,
        @NonNull SurfaceControl.Transaction finishTransaction,
        @NonNull TransitionFinishCallback finishCallback);
```

一句话总结:**`startTransaction`你自己apply,`finishTransaction`你别apply**。后者由框架在`finishCallback`被调用时统一apply。

`takeOverAnimation`是"接管正在播的动画"的接口,用于回退手势打断转场这类场景:

```java
/**
 * Attempt to take over a running transition. This must succeed if this handler was returned
 * by {@link TransitionHandler#getHandlerForTakeover(IBinder, TransitionInfo)}.
 *
 * @param finishCallback Call this at the end of the animation, if the take-over succeeds.
 *                       Note that this will be called instead of the callback originally
 *                       passed to startAnimation(), so the caller should make sure all
 *                       necessary cleanup happens here. This MUST be called on main thread.
 */
default boolean takeOverAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
        @NonNull SurfaceControl.Transaction transaction,
        @NonNull TransitionFinishCallback finishCallback,
        @NonNull WindowAnimationState[] states) {
    return false;
}
```

注意"原本传给`startAnimation`的callback不会再生效"——接管方要自己负责收尾。

### 7.6 TransitionObserver:不演,只看

有些组件需要知道过渡的进度但自己不演戏,比如SystemUI要跟着过渡做音量面板的淡出。这类用`TransitionObserver`:

```java
/**
 * Interface for something that needs to know the lifecycle of some transitions, but never
 * handles any transition by itself.
 */
public interface TransitionObserver {
    /**
     * Called when the transition is ready to play. It may later be merged into other
     * transitions. Note this doesn't mean this transition will be played anytime soon.
     */
    default void onTransitionReady(@NonNull IBinder transition, @NonNull TransitionInfo info,
            @NonNull SurfaceControl.Transaction startTransaction,
            @NonNull SurfaceControl.Transaction finishTransaction) {}

    /**
     * Called when the transition is starting to play. It isn't called for merged transitions.
     */
    default void onTransitionStarting(@NonNull IBinder transition) {}

    /**
     * Called when a transition is merged into another transition. There won't be any following
     * lifecycle calls for the merged transition.
     */
    default void onTransitionMerged(@NonNull IBinder merged, @NonNull IBinder playing) {}

    /**
     * Called when the transition is finished. This isn't called for merged transitions.
     */
    default void onTransitionFinished(@NonNull IBinder transition, boolean aborted) {}
}
```

注册方式:

```java
mObservers.add(observer);
```

分发(在`onTransitionReady`里):

```java
for (int i = 0; i < mObservers.size(); i++) {
    if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER,
                mObservers.get(i).getClass().getSimpleName() + "#onTransitionReady: "
                        + transitTypeToString(info.getType()));
    }
    mObservers.get(i).onTransitionReady(
            transitionToken, info, startTransaction, finishTransaction);
    Trace.endSection();
}
```

`HomeTransitionObserver`和`FocusTransitionObserver`是Shell自带的两个Observer,分别监听"涉及Launcher home的过渡"和"焦点切换",把回调转发给Launcher。

### 7.7 TransitionInfo:WMCore给Shell的"剧本"

`TransitionInfo`(`core/java/android/window/TransitionInfo.java`)是一次过渡的完整描述。

顶层字段:

```java
public static final int FLAG_NONE = 0;
public static final int FLAG_SHOW_WALLPAPER = 1;
public static final int FLAG_IS_WALLPAPER = 1 << 1;
public static final int FLAG_TRANSLUCENT = 1 << 2;
public static final int FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT = 1 << 3;
public static final int FLAG_IS_VOICE_INTERACTION = 1 << 4;
public static final int FLAG_IS_DISPLAY = 1 << 5;
public static final int FLAG_DISPLAY_HAS_ALERT_WINDOWS = 1 << 7;
public static final int FLAG_IS_INPUT_METHOD = 1 << 8;
public static final int FLAG_IN_TASK_WITH_EMBEDDED_ACTIVITY = 1 << 9;
public static final int FLAG_FILLS_TASK = 1 << 10;
public static final int FLAG_WILL_IME_SHOWN = 1 << 11;
public static final int FLAG_CROSS_PROFILE_OWNER_THUMBNAIL = 1 << 12;
public static final int FLAG_CROSS_PROFILE_WORK_THUMBNAIL = 1 << 13;
public static final int FLAG_IS_BEHIND_STARTING_WINDOW = 1 << 14;
public static final int FLAG_IS_OCCLUDED = 1 << 15;
public static final int FLAG_IS_SYSTEM_WINDOW = 1 << 16;
public static final int FLAG_BACK_GESTURE_ANIMATED = 1 << 17;
public static final int FLAG_NO_ANIMATION = 1 << 18;
public static final int FLAG_TASK_LAUNCHING_BEHIND = 1 << 19;
public static final int FLAG_MOVED_TO_TOP = 1 << 20;
public static final int FLAG_SYNC = 1 << 21;
public static final int FLAG_CONFIG_AT_END = 1 << 22;
public static final int FLAG_IS_TASK_DISPLAY_AREA = 1 << 23;
public static final int FLAG_FIRST_CUSTOM = 1 << 24;
```

注意第6位(`1 << 6`)在列表里是缺的——历史上曾有`FLAG_IS_BEHIND_STARTING_WINDOW`以外的用法被移除,保留下来的位号不再连续。

一次过渡包含若干`Change`,每个`Change`描述一个窗口容器(Activity、Task、DisplayArea、壁纸等)的变化:

```java
public static final class Change implements Parcelable {
    private final WindowContainerToken mContainer;
    private WindowContainerToken mParent;
    private WindowContainerToken mLastParent;
    private SurfaceControl mLeash;
    private @TransitionMode int mMode = TRANSIT_NONE;
    private @ChangeFlags int mFlags = FLAG_NONE;
    private final Rect mStartAbsBounds = new Rect();
    private final Rect mEndAbsBounds = new Rect();
    private final Point mEndRelOffset = new Point();
    private final Point mEndParentSize = new Point();
    private ActivityManager.RunningTaskInfo mTaskInfo = null;
    private boolean mAllowEnterPip;
    private int mStartDisplayId = INVALID_DISPLAY;
    private int mEndDisplayId = INVALID_DISPLAY;
    private @Surface.Rotation int mStartRotation = ROTATION_UNDEFINED;
    private @Surface.Rotation int mEndRotation = ROTATION_UNDEFINED;
    private @Surface.Rotation int mEndFixedRotation = ROTATION_UNDEFINED;
    private int mRotationAnimation = ROTATION_ANIMATION_UNSPECIFIED;
    private @ColorInt int mBackgroundColor;
    private SurfaceControl mSnapshot = null;
    private float mSnapshotLuma;
    private ComponentName mActivityComponent = null;
    private AnimationOptions mAnimationOptions = null;
    private IBinder mTaskFragmentToken = null;
```

几个关键字段:

- `mLeash`:这个容器的surface句柄。Handler动画就是操作它——平移、缩放、透明度。
- `mMode`:动作类型,**取值就是`TRANSIT_*`**:

```java
/** @return which action this change represents. */
public @TransitionMode int getMode() {
    return mMode;
}
```

- `mStartAbsBounds`/`mEndAbsBounds`:变化前后的绝对边界。
- `mSnapshot`/`mSnapshotLuma`:WMCore生成的截图及其亮度,用于做模糊背景。
- `mBackgroundColor`:Activity声明的背景色,用于动画期间的填充色。

**注意一个常见误解**:老资料里常提`CHANGE_*`系列常量(如`CHANGE_APPEARED`、`CHANGE_DISAPPEARED`)。在当前源码里这些常量**已经不存在**,`Change.getMode()`返回的直接是`TRANSIT_OPEN`、`TRANSIT_CLOSE`、`TRANSIT_TO_FRONT`、`TRANSIT_TO_BACK`、`TRANSIT_CHANGE`这些值。判断"这个容器是出现还是消失"现在用`TransitionUtil`:

```java
import static com.android.wm.shell.shared.TransitionUtil.isClosingType;
import static com.android.wm.shell.shared.TransitionUtil.isOpeningType;
```

### 7.8 TRANSIT类型

框架内置的过渡类型定义在`core/java/android/view/WindowManager.java`:

```java
/** @hide */
int TRANSIT_NONE = 0;
/**
 * A window that didn't exist before has been created and made visible.
 * @hide
 */
int TRANSIT_OPEN = 1;
/**
 * A window that was visible no-longer exists (was finished or destroyed).
 * @hide
 */
int TRANSIT_CLOSE = 2;
/**
 * A window that already existed but was not visible is made visible.
 * @hide
 */
int TRANSIT_TO_FRONT = 3;
/**
 * A window that was visible is made invisible but still exists.
 * @hide
 */
int TRANSIT_TO_BACK = 4;
/** @hide */
int TRANSIT_RELAUNCH = 5;
/**
 * A window is visible before and after but changes in some way (eg. it resizes or changes
 * windowing-mode).
 * @hide
 */
int TRANSIT_CHANGE = 6;
```

另外两个旧的keyguard类型已标`@Deprecated`:

```java
/**
 * The keyguard was visible and has been dismissed.
 * @deprecated use {@link #TRANSIT_TO_BACK} + {@link #TRANSIT_FLAG_KEYGUARD_GOING_AWAY} for
 *             keyguard going away with Shell transition.
 * @hide
 */
@Deprecated
int TRANSIT_KEYGUARD_GOING_AWAY = 7;
```

1000以上留给Shell自定义:

```java
/**
 * The first slot for custom transition types. Callers (like Shell) can make use of custom
 * transition types for dealing with special cases. These types are effectively ignored by
 * Core and will just be passed along as part of TransitionInfo objects. An example is
 * split-screen using a custom type for it's snap-to-dismiss action. By using a custom type,
 * Shell can properly dispatch the results of that transition to the split-screen
 * implementation.
 * @hide
 */
int TRANSIT_FIRST_CUSTOM = 1000;
```

Shell自定义的类型在`Transitions.java`里,数量不少:

```java
/** Transition type for exiting PIP via the Shell, via pressing the expand button. */
public static final int TRANSIT_EXIT_PIP = TRANSIT_FIRST_CUSTOM + 1;

public static final int TRANSIT_EXIT_PIP_TO_SPLIT =  TRANSIT_FIRST_CUSTOM + 2;

/** Transition type for removing PIP via the Shell, either via Dismiss bubble or Close. */
public static final int TRANSIT_REMOVE_PIP = TRANSIT_FIRST_CUSTOM + 3;

/** Transition type for launching 2 tasks simultaneously. */
public static final int TRANSIT_SPLIT_SCREEN_PAIR_OPEN = TRANSIT_FIRST_CUSTOM + 4;

/** Transition type for entering split by opening an app into side-stage. */
public static final int TRANSIT_SPLIT_SCREEN_OPEN_TO_SIDE = TRANSIT_FIRST_CUSTOM + 5;

/** Transition type for dismissing split-screen via dragging the divider off the screen. */
public static final int TRANSIT_SPLIT_DISMISS_SNAP = TRANSIT_FIRST_CUSTOM + 6;

/** Transition type for dismissing split-screen. */
public static final int TRANSIT_SPLIT_DISMISS = TRANSIT_FIRST_CUSTOM + 7;

/** Transition type for freeform to maximize transition. */
public static final int TRANSIT_MAXIMIZE = WindowManager.TRANSIT_FIRST_CUSTOM + 8;

/** Transition type for maximize to freeform transition. */
public static final int TRANSIT_RESTORE_FROM_MAXIMIZE = WindowManager.TRANSIT_FIRST_CUSTOM + 9;

/** Transition type for starting the drag to desktop mode. */
public static final int TRANSIT_DESKTOP_MODE_START_DRAG_TO_DESKTOP =
        WindowManager.TRANSIT_FIRST_CUSTOM + 10;

/** Transition type for finalizing the drag to desktop mode. */
public static final int TRANSIT_DESKTOP_MODE_END_DRAG_TO_DESKTOP =
        WindowManager.TRANSIT_FIRST_CUSTOM + 11;

/** Transition type to cancel the drag to desktop mode. */
public static final int TRANSIT_DESKTOP_MODE_CANCEL_DRAG_TO_DESKTOP =
        WindowManager.TRANSIT_FIRST_CUSTOM + 13;

/** Transition type to animate the toggle resize between the max and default desktop sizes. */
public static final int TRANSIT_DESKTOP_MODE_TOGGLE_RESIZE =
        WindowManager.TRANSIT_FIRST_CUSTOM + 14;

/** Transition to resize PiP task. */
public static final int TRANSIT_RESIZE_PIP = TRANSIT_FIRST_CUSTOM + 16;
```

看这个列表就能反推Shell都接管了哪些转场:PiP进出、分屏各种操作、桌面模式(desktop mode)的拖拽。

### 7.9 开关与配置

```java
/** Set to {@code true} to enable shell transitions. */
public static final boolean ENABLE_SHELL_TRANSITIONS = getShellTransitEnabled();
public static final boolean SHELL_TRANSITIONS_ROTATION = ENABLE_SHELL_TRANSITIONS
        && SystemProperties.getBoolean("persist.wm.debug.shell_transit_rotate", false);
```

`ENABLE_SHELL_TRANSITIONS`已经是个常量,`SHELL_TRANSITIONS_ROTATION`则是"旋转也走Shell过渡"的调试开关(`persist.wm.debug.shell_transit_rotate`)。

关掉时行为回落到`LegacyTransitions`(`libs/WindowManager/Shell/src/com/android/wm/shell/transition/LegacyTransitions.java`)。

动画倍速通过ContentObserver跟着Settings走:

```java
ContentResolver resolver = mContext.getContentResolver();
mTransitionAnimationScaleSetting = getTransitionAnimationScaleSetting();
dispatchAnimScaleSetting(mTransitionAnimationScaleSetting);

resolver.registerContentObserver(
        Settings.Global.getUriFor(Settings.Global.TRANSITION_ANIMATION_SCALE), false,
        new SettingsObserver());
```

倍速会被分发给所有Handler:

```java
private void dispatchAnimScaleSetting(float scale) {
    for (int i = mHandlers.size() - 1; i >= 0; --i) {
        mHandlers.get(i).setAnimScaleSetting(scale);
    }
}
```

这就是开发者选项里"动画时长缩放"能实时生效的原因——不需要重启SystemUI。

### 7.10 注册与外部接口

`Transitions`在`onInit`里做四件事:

```java
private void onInit() {
    if (Transitions.ENABLE_SHELL_TRANSITIONS) {
        mOrganizer.shareTransactionQueue();
    }
    mShellController.addExternalInterface(IShellTransitions.DESCRIPTOR,
            this::createExternalInterface, this);
    ...
    if (Transitions.ENABLE_SHELL_TRANSITIONS) {
        mIsRegistered = true;
        // Register this transition handler with Core
        try {
            mOrganizer.registerTransitionPlayer(mPlayerImpl);
        } catch (RuntimeException e) {
            mIsRegistered = false;
            throw e;
        }
        // Pre-load the instance.
        TransitionMetrics.getInstance();
    }

    mShellCommandHandler.addCommandCallback("transitions", this, this);
    mShellCommandHandler.addDumpCallback(this::dump, this);
}
```

1. `shareTransactionQueue()`:把Shell的`SyncTransactionQueue`分享给WMCore,让两边的事务能同步。
2. `addExternalInterface(IShellTransitions.DESCRIPTOR, ...)`:导出给Launcher。
3. `registerTransitionPlayer(mPlayerImpl)`:向WMCore注册`ITransitionPlayer`。**这是Shell Transition真正"接上线"的一步**。
4. 注册`transitions`命令和dump。

`ITransitionPlayer`的实现只有两个方法:

```java
private class TransitionPlayerImpl extends ITransitionPlayer.Stub {
    @Override
    public void onTransitionReady(IBinder iBinder, TransitionInfo transitionInfo,
            SurfaceControl.Transaction t, SurfaceControl.Transaction finishT)
            throws RemoteException {
        ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, "onTransitionReady(transaction=%d)",
                t.getId());
        mMainExecutor.execute(() -> Transitions.this.onTransitionReady(
                iBinder, transitionInfo, t, finishT));
    }

    @Override
    public void requestStartTransition(IBinder iBinder,
            TransitionRequestInfo request) throws RemoteException {
        mMainExecutor.execute(() -> Transitions.this.requestStartTransition(iBinder, request));
    }
}
```

Binder线程收到,立刻post到Shell主线程。这就是第5章讲的"接口提供方负责post"的典型例子。

**注意`mMainExecutor.execute`用的是`wmshell.main`,不是`wmshell.anim`。** 只有动画具体的每一帧计算才在`mAnimExecutor`上。`getAnimExecutor()`单独暴露出来给Handler用。

### 7.11 RemoteTransition:让Launcher来演

Launcher(Quickstep)要自己实现Recents的转场动画,它注册`RemoteTransition`:

```java
/**
 * Register a remote transition to be used for all operations except takeovers when `filter`
 * matches an incoming transition.
 */
public void registerRemote(@NonNull TransitionFilter filter,
        @NonNull RemoteTransition remoteTransition) {
    mRemoteTransitionHandler.addFiltered(filter, remoteTransition);
}
```

`TransitionFilter`是一个可序列化的匹配器,由Launcher在注册时给出:

```java
public final class TransitionFilter implements Parcelable {

    /** The associated requirement doesn't care about the z-order. */
    public static final int CONTAINER_ORDER_ANY = 0;
    /** The associated requirement only matches the top-most (z-order) container. */
    public static final int CONTAINER_ORDER_TOP = 1;

    /**
     * When non-null: this is a list of transition types that this filter applies to. This filter
     * will fail for transitions that aren't one of these types.
     */
    @Nullable public @TransitionType int[] mTypeSet = null;

    /** All flags must be set on a transition. */
    public @WindowManager.TransitionFlags int mFlags = 0;

    /** All flags must NOT be set on a transition. */
    public @WindowManager.TransitionFlags int mNotFlags = 0;

    /**
     * A list of required changes. To pass, a transition must meet all requirements.
     */
    @Nullable public Requirement[] mRequirements = null;
```

匹配逻辑:

```java
/** @return true if `info` meets all the requirements to pass this filter. */
public boolean matches(@NonNull TransitionInfo info) {
    if (mTypeSet != null) {
        // non-null typeset, so make sure info is one of the types.
        boolean typePass = false;
        for (int i = 0; i < mTypeSet.length; ++i) {
            if (info.getType() == mTypeSet[i]) {
                typePass = true;
                break;
            }
        }
        if (!typePass) return false;
    }
    if ((info.getFlags() & mFlags) != mFlags) {
        return false;
    }
    if ((info.getFlags() & mNotFlags) != 0) {
        return false;
    }
    // Make sure info meets all of the requirements.
    if (mRequirements != null) {
        for (int i = 0; i < mRequirements.length; ++i) {
            final boolean matches = mRequirements[i].matches(info);
            if (matches == mRequirements[i].mNot) {
                return false;
            }
        }
    }
    return true;
}
```

**为什么要把filter设计成Parcelable?** 类注释解释了:

> A parcelable filter that can be used for rerouting transitions to a remote. This is a local representation so that the transition system doesn't need to make blocking queries over binder.

如果filter是个Binder接口,每次匹配都要跨进程调用,而匹配发生在过渡分发的最热路径上。做成可序列化对象,注册时一次性传过来,之后纯本地判断。

`RemoteTransitionHandler`就是拿着这些filter逐条匹配的Handler,它在`mHandlers`里排索引1,优先级仅高于兜底。

**OneShotRemoteHandler**是另一个相关类,字面意思是"只演一次":

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/transition/OneShotRemoteHandler.java
```

它用于`RemoteAnimationAdapter`那套旧的接口——旧接口注册进来的时候没说要演哪次,wrapper就把它包成一个"谁匹配上就用掉,用掉即作废"的Handler。

### 7.12 LegacyTransitions与RemoteAnimationAdapter

Shell Transition是新路径,`LegacyTransitions`是旧路径:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/transition/LegacyTransitions.java
```

`RemoteAnimationAdapter`(`core/java/android/view/RemoteAnimationAdapter.java`)**在当前源码里没有标`@Deprecated`**,但它的定位已经变成"给还没迁移的应用/OEM用的兼容通道"。系统应用新代码应该用`RemoteTransition`。

两者的区别:

| 维度 | `RemoteAnimationAdapter` | `RemoteTransition` |
|---|---|---|
| 传的是什么 | 一组`RemoteAnimationTarget`(裸surface) | `TransitionInfo`(带窗口树结构) |
| 谁决定动画范围 | App自己 | 由filter匹配决定 |
| 能否收到取消/接管 | 不能 | 可以(`takeOverAnimation`) |
| 生命周期 | 一次性的 | 注册后可复用 |

### 7.13 一次转场的完整时序

把上面所有东西串起来,一次"从桌面点开App"的流程:

| 步骤 | 执行者 | 线程 | 动作 |
|---|---|---|---|
| 1 | Launcher | 自己的 | 调`startActivity`,带`ActivityOptions.setRemoteTransition(...)` |
| 2 | WMCore | `system_server` | 创建transition token,调`ITransitionPlayer.requestStartTransition` |
| 3 | `TransitionPlayerImpl` | Binder线程 | 收到请求,post到`wmshell.main` |
| 4 | `Transitions.requestStartTransition` | `wmshell.main` | `dispatchRequest`从后往前问每个Handler |
| 5 | 各Handler | `wmshell.main` | 返回`WindowContainerTransaction`或null |
| 6 | WMCore | `system_server` | 应用WCT,布局算好,发`onTransitionReady` |
| 7 | `TransitionPlayerImpl` | Binder线程 | post到`wmshell.main` |
| 8 | `Transitions.onTransitionReady` | `wmshell.main` | 从PENDING移到READY,调`dispatchReady` |
| 9 | `dispatchReady` | `wmshell.main` | 交给对应Track,尝试merge,否则排队 |
| 10 | `dispatchTransition` | `wmshell.main` | 从后往前问每个Handler要`startAnimation` |
| 11 | 选中的Handler | `wmshell.main` + `wmshell.anim` | 建Animator,每帧改surface |
| 12 | Handler | `wmshell.anim` | 动画结束,在`wmshell.main`上调`finishCallback` |
| 13 | `Transitions.onFinish` | `wmshell.main` | apply finishTransaction,通知Observer |
| 14 | WMCore | `system_server` | 收到finish,释放transition,回收surface |

第4步和第10步都发生在`wmshell.main`上,而第11步的逐帧计算在`wmshell.anim`上——这是为什么点击到动画开始之间的延迟必须控制得很短:第4到第10步全部是主线程上的同步工作。查看`wmshell.main`的`Slow dispatch`日志(阈值30ms)就是排查"点击后动画迟迟不动"的第一手段。

## 8 Shell功能模块

这一章按`libs/WindowManager/Shell/src/com/android/wm/shell/`下的目录,逐个说明各模块的装配方式和它导出的接口。

### 8.1 模块目录总览

实际目录列表(节选,按目录名排序):

| 目录 | 主要类 | 职责 |
|---|---|---|
| `activityembedding/` | `ActivityEmbeddingController` | Activity嵌入(大屏双栏) |
| `animation/` | 动画工具类 | 供各模块复用的动画辅助 |
| `apptoweb/` | 无 | 预留目录 |
| `automotive/` | 车机相关 | 车机定制 |
| `back/` | `BackAnimationController`、`BackAnimation`、`IBackAnimation.aidl` | 预测性返回手势动画 |
| `bubbles/` | `BubbleController`、`Bubbles`、`IBubbles.aidl` | 气泡通知 |
| `common/` | `DisplayController`、`SyncTransactionQueue`、`TransactionPool`、`ShellExecutor` | 公共设施 |
| `compatui/` | `CompatUIWindowManager` | 兼容模式提示UI |
| `dagger/` | `WMShellBaseModule`、`WMShellConcurrencyModule`等 | Dagger模块 |
| `desktopmode/` | `DesktopTasksController`、`DesktopMode`、`IDesktopMode.aidl` | 桌面模式 |
| `displayareahelper/` | `DisplayAreaHelperController`、`DisplayAreaHelper` | 显示区域助手 |
| `docs/` | 设计文档 | 不含代码 |
| `draganddrop/` | `DragAndDropController`、`IDragAndDrop.aidl` | 拖放 |
| `freeform/` | `FreeformTaskListener` | 自由窗口 |
| `fullscreen/` | `FullscreenTaskListener` | 全屏task接管 |
| `hidedisplaycutout/` | `HideDisplayCutoutController`、`HideDisplayCutoutOrganizer` | 隐藏刘海 |
| `keyguard/` | `KeyguardTransitionHandler`、`KeyguardTransitions` | 锁屏过渡 |
| `onehanded/` | `OneHandedController`、`OneHandedDisplayAreaOrganizer`、`OneHanded`、`IOneHanded.aidl` | 单手模式 |
| `performance/` | `PerfHintController.kt` | 性能提示 |
| `pip/` | `Pip.java`、`PipTransitionState`、`PipTaskOrganizer`、`PipAnimationController`;`pip/phone/PipController.java` | 画中画 |
| `pip2/` | 新版PiP实验实现 | 画中画(实验) |
| `protolog/` | `ShellProtoLogGroup`等 | ProtoLog定义 |
| `recents/` | `RecentTasksController`、`RecentTasks`、`IRecentTasks.aidl` | 最近任务 |
| `splitscreen/` | `SplitScreenController`、`StageCoordinator`、`SplitScreenTransitions`、`ISplitScreen.aidl` | 分屏 |
| `startingsurface/` | `StartingWindowController`、`StartingSurface`、`IStartingWindow.aidl` | 启动窗口/闪屏 |
| `sysui/` | `ShellInit`、`ShellController`、`ShellCommandHandler`、`ShellInterface` | 初始化与生命周期 |
| `taskview/` | `TaskViewFactoryController`、`TaskViewFactory`、`TaskView` | 任务快照视图 |
| `transition/` | `Transitions`、`DefaultTransitionHandler`、`RemoteTransitionHandler` | 过渡动画 |
| `unfold/` | `UnfoldAnimationController`、`UnfoldBackgroundController` | 折叠屏展开动画 |
| `windowdecor/` | `DragResizeInputListener`等 | 窗口装饰(桌面模式用) |

**几个容易记错的地方**:

- 分屏目录叫`splitscreen`,**不是`split`**。
- 启动窗口目录叫`startingsurface`,**不是`startingwindow`**;但类名是`StartingWindowController`。
- `PipController`不在`pip/`下,而在`pip/phone/`下,类名`PipController`前面没有`I`。
- 画中画的AIDL叫`IPip.aidl`,但它在`common/pip/`下,不在`pip/`下。

### 8.2 两类对外接口

Shell模块对外暴露接口有两条路径,**别搞混**:

| 路径 | 形式 | 给谁用 |
|---|---|---|
| `ShellController.addExternalInterface(descriptor, factory, target)` | AIDL(`IXxx.aidl`) | **跨进程**,主要是Launcher、其他系统App |
| 通过`WMComponent`直接返回 | 同进程Java/Kotlin接口(`@ExternalThread`) | **同进程**,SystemUI自己 |

第4章讲过,`SysUIComponent.Builder`上有13个`@BindsInstance`setter,注入的就是第二类:

```java
.setShell(mWMComponent.getShell())
.setPip(mWMComponent.getPip())
.setSplitScreen(mWMComponent.getSplitScreen())
.setOneHanded(mWMComponent.getOneHanded())
.setBubbles(mWMComponent.getBubbles())
.setTaskViewFactory(mWMComponent.getTaskViewFactory())
.setShellTransitions(mWMComponent.getShellTransitions())
.setKeyguardTransitions(mWMComponent.getKeyguardTransitions())
.setStartingSurface(mWMComponent.getStartingSurface())
.setDisplayAreaHelper(mWMComponent.getDisplayAreaHelper())
.setRecentTasks(mWMComponent.getRecentTasks())
.setBackAnimation(mWMComponent.getBackAnimation())
.setDesktopMode(mWMComponent.getDesktopMode());
```

这13个同进程接口都标了`@ExternalThread`,例如:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/pip/Pip.java
/**
 * Interface to engage picture in picture feature.
 */
@ExternalThread
public interface Pip {
    /**
     * Expand PIP, it's possible that specific request to activate the window via Alt-tab.
     */
    default void expandPip() {
    }

    /**
     * Called when SysUI state changed.
     */
    default void onSystemUiStateChanged(boolean isSysUiStateValid, long flag) {
    }

    default void addOnIsInPipStateChangedListener(@NonNull Consumer<Boolean> callback) {}
    default void removeOnIsInPipStateChangedListener(@NonNull Consumer<Boolean> callback) {}

    default void showPictureInPictureMenu() {}

    /**
     * Called by NavigationBar and TaskbarDelegate in order to listen in for PiP bounds change. This
     * is mostly used for times where the PiP bounds could conflict with SystemUI elements, such as
     * a stashed PiP and the Back-from-Edge gesture.
     */
    default void addPipExclusionBoundsChangeListener(Consumer<Rect> listener) { }
    default void removePipExclusionBoundsChangeListener(Consumer<Rect> listener) { }

    default void registerPipTransitionCallback(
            @NonNull PipTransitionController.PipTransitionCallback callback,
            @NonNull Executor executor) { }
}
```

注意`addPipExclusionBoundsChangeListener`的注释:NavigationBar(即SystemUI的导航栏)会调它,用于把PiP的边界告诉手势系统,免得返回手势扫过PiP区域时误触发。**这就是Shell和SystemUI在代码层面的直接耦合点之一**。

`OneHanded`接口很短:

```java
/**
 * Interface to engage one handed feature.
 */
@ExternalThread
public interface OneHanded {

    boolean sIsSupportOneHandedMode =  SystemProperties.getBoolean(
            OneHandedController.SUPPORT_ONE_HANDED_MODE, false);

    /** Enters one handed mode. */
    void startOneHanded();

    /** Exits one handed mode. */
    void stopOneHanded();

    /** Exits one handed mode with {@link OneHandedUiEventLogger}. */
    void stopOneHanded(int uiEvent);

    void setLockedDisabled(boolean locked, boolean enabled);

    /** Registers callback to notify WMShell when user tap shortcut to expand notification. */
    void registerEventCallback(OneHandedEventCallback callback);

    /** Registers callback to be notified after {@link OneHandedDisplayAreaOrganizer}
     * transition start or finish */
    void registerTransitionCallback(OneHandedTransitionCallback callback);
}
```

**这里有一个值得注意的写法**:接口里放了一个`public static final`字段`sIsSupportOneHandedMode`,而Java接口字段隐式是`public static final`,所以`OneHanded.sIsSupportOneHandedMode`可以直接读,不需要实例。这是把"设备是否支持单手模式"编译进接口的简单做法,判断依据是系统属性(`OneHandedController.SUPPORT_ONE_HANDED_MODE`)。

`RecentTasks`:

```java
/**
 * Interface for interacting with the recent tasks.
 */
@ExternalThread
public interface RecentTasks {
    /**
     * Gets the set of recent tasks.
     */
    default void getRecentTasks(int maxNum, int flags, int userId, Executor callbackExecutor,
            Consumer<List<GroupedTaskInfo>> callback) {
    }

    /**
     * Adds the listener to be notified of whether the recent task animation is running.
     */
    default void addAnimationStateListener(Executor listenerExecutor, Consumer<Boolean> listener) {
    }

    /**
     * Sets a background color on the transition root layered behind the outgoing task. {@code null}
     * may be used to clear any previously set colors to avoid showing a background at all. The
     * color is always shown at full opacity.
     */
    default void setTransitionBackgroundColor(@Nullable Color color) {
    }
}
```

注意返回类型是`List<GroupedTaskInfo>`,`GroupedTaskInfo`定义在`Shell/shared/src/com/android/wm/shell/shared/GroupedTaskInfo.java`,并且带一个AIDL伴生文件`GroupedTaskInfo.aidl`——因为跨进程传需要Parcelable。

**所有方法都是default空实现**,这是Shell接口的统一风格:方便测试替身,也方便OEM只覆盖需要的方法。

跨进程的一类,典型是`ISplitScreen.aidl`:

```aidl
/**
 * Interface that is exposed to remote callers to manipulate the splitscreen feature.
 */
interface ISplitScreen {

    /** Registers a split screen listener. */
    oneway void registerSplitScreenListener(in ISplitScreenListener listener) = 1;

    /** Unregisters a split screen listener. */
    oneway void unregisterSplitScreenListener(in ISplitScreenListener listener) = 2;

    /** Registers a split select listener. */
    oneway void registerSplitSelectListener(in ISplitSelectListener listener) = 20;

    /** Unregisters a split select listener. */
    oneway void unregisterSplitSelectListener(in ISplitSelectListener listener) = 21;

    /**
     * Removes the split-screen stages and leaving indicated task to top. Passing INVALID_TASK_ID
     * to indicate leaving no top task after leaving split-screen.
     */
    oneway void exitSplitScreen(int toTopTaskId) = 5;

    oneway void startTask(int taskId, int position, in Bundle options) = 7;

    oneway void startShortcut(String packageName, String shortcutId, int position,
            in Bundle options, in UserHandle user, in InstanceId instanceId) = 8;
    ...
}
```

方法编号不连续(1、2、20、21、5、7、8)是AIDL的常态——删掉方法后编号不复用,保证兼容性。

### 8.3 PipController:最完整的模块样板

`PipController`(`libs/WindowManager/Shell/src/com/android/wm/shell/pip/phone/PipController.java`)各要素齐全,拿它讲装配最合适。

**类声明**列出了它作为`RemoteCallable`要实现的四种监听:

```java
/**
 * Manages the picture-in-picture (PIP) UI and states for Phones.
 */
public class PipController implements PipTransitionController.PipTransitionCallback,
        RemoteCallable<PipController>, ConfigurationChangeListener, KeyguardChangeListener,
        UserChangeListener {
    private static final String TAG = "PipController";
```

`RemoteCallable<PipController>`是Shell里所有需要导出外部接口的类的统一接口,要求实现三个方法:

```java
@Override
public Context getContext() {
    return mContext;
}

@Override
public ShellExecutor getRemoteCallExecutor() {
    return mMainExecutor;
}
```

第三个是`createExternalInterface()`。

**创建走的是静态工厂,而不是`@Inject`构造**:

```java
public static Pip create(Context context,
        ShellInit shellInit,
        ShellCommandHandler shellCommandHandler,
        ShellController shellController,
        DisplayController displayController,
        PipAnimationController pipAnimationController,
        PipAppOpsListener pipAppOpsListener,
        PipBoundsAlgorithm pipBoundsAlgorithm,
        PipKeepClearAlgorithmInterface pipKeepClearAlgorithm,
        PipBoundsState pipBoundsState,
        PipDisplayLayoutState pipDisplayLayoutState,
        PipMotionHelper pipMotionHelper,
        PipMediaController pipMediaController,
        PhonePipMenuController phonePipMenuController,
        PipTaskOrganizer pipTaskOrganizer,
        PipTransitionState pipTransitionState,
        PipTouchHandler pipTouchHandler,
        PipTransitionController pipTransitionController,
        WindowManagerShellWrapper windowManagerShellWrapper,
        TaskStackListenerImpl taskStackListener,
        PipParamsChangedForwarder pipParamsChangedForwarder,
        DisplayInsetsController displayInsetsController,
        TabletopModeController pipTabletopController,
        Optional<OneHandedController> oneHandedController,
        ShellExecutor mainExecutor,
        Handler handler) {
    if (!context.getPackageManager().hasSystemFeature(FEATURE_PICTURE_IN_PICTURE)) {
        ProtoLog.w(ShellProtoLogGroup.WM_SHELL_PICTURE_IN_PICTURE,
                "%s: Device doesn't support Pip feature", TAG);
        return null;
    }

    return new PipController(context, shellInit, shellCommandHandler, shellController,
            displayController, pipAnimationController, pipAppOpsListener,
            pipBoundsAlgorithm, pipKeepClearAlgorithm, pipBoundsState,
            pipDisplayLayoutState, pipMotionHelper, pipMediaController, phonePipMenuController,
            pipTaskOrganizer, pipTransitionState, pipTouchHandler, pipTransitionController,
            windowManagerShellWrapper, taskStackListener, pipParamsChangedForwarder,
            displayInsetsController, pipTabletopController, oneHandedController, mainExecutor,
            handler)
            .mImpl;
}
```

两个要点:

1. **设备没有`FEATURE_PICTURE_IN_PICTURE`时直接返回`null`**。Dagger侧用`Optional<Pip>`接住,后面所有用到PiP的地方都先判optional。
2. **返回的是`.mImpl`,不是`PipController`本身**。`mImpl`是内部类`PipImpl implements Pip`,外面拿到的是接口不是实现。

**构造函数末尾注册初始化回调**:

```java
if (!PipUtils.isPip2ExperimentEnabled()) {
    shellInit.addInitCallback(this::onInit, this);
}
```

新版PiP2实验开着时,初始化路径走`pip2/`那边的实现。

**`onInit`里做六件事**:

```java
private void onInit() {
    mShellCommandHandler.addDumpCallback(this::dump, this);
    mPipInputConsumer = new PipInputConsumer(WindowManagerGlobal.getWindowManagerService(),
            INPUT_CONSUMER_PIP, mMainExecutor);
    mPipTransitionController.registerPipTransitionCallback(this, mMainExecutor);
    mPipTaskOrganizer.registerOnDisplayIdChangeCallback((int displayId) -> {
        mPipDisplayLayoutState.setDisplayId(displayId);
        onDisplayChanged(mDisplayController.getDisplayLayout(displayId),
                false /* saveRestoreSnapFraction */);
    });
    mPipTransitionState.addOnPipTransitionStateChangedListener((oldState, newState) -> {
        final boolean wasInPip = PipTransitionState.isInPip(oldState);
        final boolean nowInPip = PipTransitionState.isInPip(newState);
        if (nowInPip != wasInPip) {
            for (Consumer<Boolean> listener : mOnIsInPipStateChangedListeners) {
                listener.accept(nowInPip);
            }
        }
    });
    ...
    mMediaController.registerSessionListenerForCurrentUser();

    mShellController.addConfigurationChangeListener(this);
    mShellController.addKeyguardChangeListener(this);
    mShellController.addUserChangeListener(this);
    mShellController.addExternalInterface(IPip.DESCRIPTOR,
            this::createExternalInterface, this);
}
```

六件事分别是:注册dump、建输入消费者(接PiP窗口的触摸)、注册PiP过渡回调、注册显示变化回调、注册PiP状态变化监听、注册三类ShellController监听、导出外部接口。

`mShellController.addConfigurationChangeListener(this)`这类方法,对应第4章讲的`ShellController`,它把Shell的配置变化、锁屏变化、用户切换分发给各个模块,模块只需要实现对应的`onConfigurationChanged`等方法:

```java
@Override
public void onUserChanged(int newUserId, @NonNull Context userContext) {
    // Re-register the media session listener when switching users
    mMediaController.registerSessionListenerForCurrentUser();
}

@Override
public void onConfigurationChanged(Configuration newConfig) {
    mPipBoundsAlgorithm.onConfigurationChanged(mContext);
    mTouchHandler.onConfigurationChanged();
    mPipBoundsState.onConfigurationChanged();
    mPipDisplayLayoutState.onConfigurationChanged();
}
```

**外部接口实现是静态内部类,标`@BinderThread`**:

```java
/**
 * The interface for calls from outside the host process.
 */
@BinderThread
private static class IPipImpl extends IPip.Stub implements ExternalInterfaceBinder {
    private PipController mController;
    private final SingleInstanceRemoteListener<PipController,
            IPipAnimationListener> mListener;
    ...
    IPipImpl(PipController controller) {
        mController = controller;
    }
```

注意它**同时继承`IPip.Stub`并实现`ExternalInterfaceBinder`**,而且不是内部类而是`static`的——因为它只持有`mController`引用,不需要外部实例。`SingleInstanceRemoteListener`是Shell里处理"AIDL回调只能有一个"这个约束的工具类。

**`IPip.aidl`本身:**

```aidl
package com.android.wm.shell.common.pip;

/**
 * Interface that is exposed to remote callers to manipulate the Pip feature.
 */
interface IPip {

    /**
     * Notifies that Activity is about to be swiped to home with entering PiP transition and
     * queries the destination bounds for PiP depends on Launcher's rotation and shelf height.
     *
     * @return destination bounds the PiP window should land into
     */
    Rect startSwipePipToHome(in ComponentName componentName, in ActivityInfo activityInfo,
                in PictureInPictureParams pictureInPictureParams,
                int launcherRotation, in Rect hotseatKeepClearArea) = 1;

    /**
     * Notifies the swiping Activity to PiP onto home transition is finished
     */
    oneway void stopSwipePipToHome(int taskId, in ComponentName componentName,
            in Rect destinationBounds, in SurfaceControl overlay, in Rect appBounds,
            in Rect sourceRectHint) = 2;

    /**
     * Notifies the swiping Activity to PiP onto home transition is aborted
     */
    oneway void abortSwipePipToHome(int taskId, in ComponentName componentName) = 3;

    /**
     * Sets listener to get pinned stack animation callbacks.
     */
    oneway void setPipAnimationListener(IPipAnimationListener listener) = 4;

    /**
     * Sets the shelf height and visibility.
     */
    oneway void setShelfHeight(boolean visible, int shelfHeight) = 5;

    /**
     * Sets the next pip animation type to be the alpha animation.
     */
    oneway void setPipAnimationTypeToAlpha() = 6;

    /**
     * Sets the height and visibility of the Launcher keep clear area.
     */
    oneway void setLauncherKeepClearAreaHeight(boolean visible, int height) = 7;

    /**
     * Sets the app icon size in pixel used by Launcher
     */
    oneway void setLauncherAppIconSize(int iconSizePx) = 8;
}
```

**这份AIDL几乎全是给Launcher用的**,看方法名就知道:`startSwipePipToHome`(从App上滑进PiP)、`setShelfHeight`(Launcher任务栏高度)、`setLauncherKeepClearAreaHeight`(Launcher热区)、`setLauncherAppIconSize`(Launcher图标大小)。也就是说,PiP和Launcher的耦合非常深——上滑进PiP的动画需要Launcher和Shell一起完成。

注意`startSwipePipToHome`是**唯一没有`oneway`的**,因为它要返回`Rect`(目的地边界),必须同步等待。

### 8.4 SplitScreenController:分屏

目录`libs/WindowManager/Shell/src/com/android/wm/shell/splitscreen/`:

```text
ISplitScreen.aidl
ISplitScreenListener.aidl
ISplitSelectListener.aidl
SplitScreen.java
SplitScreenController.java
SplitScreenShellCommandHandler.java
SplitScreenTransitions.java
SplitscreenEventLogger.java
StageCoordinator.java
StageOrderOperator.kt
StageTaskListener.java
tv/
```

核心概念:

- **Stage**:分屏的一"半"。两个stage上下或左右排列。
- **`StageTaskListener`**:每个stage一个,继承`ShellTaskOrganizer.TaskListener`,接管该stage里的task。
- **`StageCoordinator`**:协调两个stage,处理divider(分隔条)拖动、stage交换、进入/退出分屏。
- **`SplitScreenTransitions`**:分屏自己的过渡处理,注册到`Transitions.mHandlers`。

`StageOrderOperator.kt`是个Kotlin类,负责决定"哪个stage在上/左"。用Kotlin写是因为涉及不少集合操作。

`tv/`子目录是Android TV的分屏实现。

分屏自定义的过渡类型在第7章列过:`TRANSIT_SPLIT_SCREEN_PAIR_OPEN`、`TRANSIT_SPLIT_SCREEN_OPEN_TO_SIDE`、`TRANSIT_SPLIT_DISMISS_SNAP`、`TRANSIT_SPLIT_DISMISS`、`TRANSIT_SPLIT_PASSTHROUGH`。这些类型由`StageCoordinator`发起,最终由它自己注册的Handler认领。

分屏还有一个独立的命令处理器:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/splitscreen/SplitScreenShellCommandHandler.java
```

对应`SplitScreenShellCommandHandler`提供的`adb shell wm shell splitscreen`子命令。

### 8.5 BubbleController:气泡

目录`bubbles/`:主要文件是`BubbleController.java`、`Bubbles.java`、`IBubbles.aidl`、`IBubblesListener.aidl`,以及大量子目录(气泡UI相关)。

气泡同时用三条通道:

1. **`ShellTaskOrganizer`**:注册为`TASK_LISTENER_TYPE_FREEFORM`,接管气泡的task。
2. **`WindowManagerShellWrapper`**:监听IME高度变化。这是第6.3节讲的旧通道:

```java
try {
    mWindowManagerShellWrapper.addPinnedStackListener(new BubblesImeListener());
} catch (RemoteException e) {
    e.printStackTrace();
}

/** PinnedStackListener that dispatches IME visibility updates to the stack. */
private class BubblesImeListener extends PinnedStackListenerForwarder.PinnedTaskListener {
    @Override
    public void onImeVisibilityChanged(boolean imeVisible, int imeHeight) {
        mBubblePositioner.setImeVisible(imeVisible, imeHeight);
        ...
    }
}
```

3. **`ShellController.addKeyguardChangeListener`**:锁屏时气泡要收起。

**为什么气泡需要IME高度?** 因为气泡是悬浮的小窗口,键盘弹出时会盖住它,`BubblePositioner`要据此把气泡往上挪。而IME的insets信息在WMCore,Shell又没有直接的insets订阅接口,只能挂在旧的`PinnedStackListener`上。这就是那个历史包袱一直没拆掉的实际原因。

### 8.6 OneHandedController:单手模式

目录`onehanded/`下的类:

```text
IOneHanded.aidl
OneHanded.java
OneHandedAnimationController.java
OneHandedController.java
OneHandedDisplayAreaOrganizer.java
(OneHandedEventCallback.java / OneHandedTransitionCallback.java 等)
```

`OneHandedDisplayAreaOrganizer extends DisplayAreaOrganizer`是本模块的核心——它把整个显示区往下拉一半,让单手能够到大屏顶部。

它的类注释说明了一个通用的动画套路:

> This class listens on `DisplayAreaOrganizer` callbacks for windowing mode change both to and from OneHanded and issues corresponding animation if applicable.
> Normally, we apply series of `SurfaceControl.Transaction` when the animator is running and files a final `WindowContainerTransaction` at the end of the transition.

**这是Shell里做"移动整块显示区"类动画的标准模式**:

- 动画过程中:直接改`SurfaceControl.Transaction`里的位置。轻量,不触发重新布局。
- 动画结束时:提交一个`WindowContainerTransaction`,让WMCore把窗口树的真实状态搬到最终位置。

如果反过来(过程中就改WCT),每一帧都要让WMCore重新算布局,根本跑不满60帧。

`OneHanded.java`接口只有开/关两个主要方法(见8.2节),因为**触发方式是从导航栏长按或系统设置里点**,调用方是SystemUI/NavigationBar。

### 8.7 BackAnimationController:返回手势动画

目录`back/`。`IBackAnimation.aidl`:

```aidl
package com.android.wm.shell.back;

import com.android.internal.view.AppearanceRegion;
import android.view.IRemoteAnimationRunner;
import android.window.IOnBackInvokedCallback;

/**
 * Interface for Launcher process to register back invocation callbacks.
 */
interface IBackAnimation {
    /**
     * Sets a {@link IOnBackInvokedCallback} and a {@link IRemoteAnimationRunner} to be invoked when
     * back navigation has type {@link BackNavigationInfo#TYPE_RETURN_TO_HOME}.
     */
    void setBackToLauncherCallback(in IOnBackInvokedCallback callback,
            in IRemoteAnimationRunner runner);

    /**
     * Clears the previously registered {@link IOnBackInvokedCallback}.
     */
    void clearBackToLauncherCallback();

    /**
     * Uses launcher flags to update the system bar color.
     */
    void customizeStatusBarAppearance(in AppearanceRegion appearance);
}
```

三个方法各对应一件事:

1. `setBackToLauncherCallback`:Launcher注册"从App返回桌面"时的动画runner。注意参数是**两个**:`IOnBackInvokedCallback`(接收返回事件)+`IRemoteAnimationRunner`(执行动画),不是一个。
2. `clearBackToLauncherCallback`:注销。
3. `customizeStatusBarAppearance`:让Launcher改系统栏颜色。这一条是**Shell主动找Launcher要**的——返回手势动画过程中系统栏颜色的过渡由Launcher决定,但它得告诉Shell。

`IRemoteAnimationRunner`出现在这里说明:即使在新架构下,预测性返回仍然复用旧的`RemoteAnimationAdapter`那一套runner接口。

### 8.8 RecentTasksController:最近任务

目录`recents/`:

```text
IRecentTasks.aidl
IRecentTasksListener.aidl
IRecentsAnimationController.aidl
IRecentsAnimationRunner.aidl
RecentTasksController.java
RecentsTransitionStateListener.java
```

`RecentTasksController`维护最近任务列表和它们的截图(task snapshot)。

**它和`ActivityTaskManagerService`的关系**:ATM是权威数据源,但每次查询都要跨进程,而且ATM的`getRecentTasks`有各种权限和flag限制。`RecentTasksController`在Shell侧做了一份缓存,给Launcher频繁访问用。

`IRecentTasks.aidl`和`RecentTasks.java`是同一个功能的两套接口:前者给Launcher(跨进程),后者给SystemUI(同进程)。

另外两个AIDL是旧的Recents动画接口,`RecentsTransitionStateListener`则是给新过渡用的。

### 8.9 DragAndDropController:拖放

目录`draganddrop/`:只有三个文件,`DragAndDropController.java`、`GlobalDragListener.kt`、`IDragAndDrop.aidl`。

拖放模块做的核心事情是:**在拖拽过程中创建一个Shell自己的surface**,也就是手指下面跟着的那个半透明视图。这个surface不是任何App的window,不能在App进程里创建,所以放在Shell。

`GlobalDragListener.kt`是个Kotlin接口,让SystemUI能全局感知拖拽开始/结束——比如拖拽时通知面板要收起。

### 8.10 DesktopTasksController:桌面模式

目录`desktopmode/`:含`DesktopTasksController`、`DesktopMode`、`IDesktopMode.aidl`、`IDesktopTaskListener.aidl`等。

`IDesktopMode.aidl`:

```aidl
/**
 * Interface that is exposed to remote callers to manipulate desktop mode features.
 */
interface IDesktopMode {

    /** Show apps on the desktop on the given display */
    void showDesktopApps(int displayId, in RemoteTransition remoteTransition);

    /** @deprecated use {@link #showDesktopApps} instead. */
    void stashDesktopApps(int displayId);

    /** @deprecated this is no longer supported. */
    void hideStashedDesktopApps(int displayId);

    /**
    * Bring task with the given id to front, using the given remote transition.
    *
    * <p> Note: beyond moving a task to the front, this method will minimize a task if we reach the
    * Desktop task limit, so {@code remoteTransition} should also handle any such minimize change.
    */
    oneway void showDesktopApp(int taskId, in @nullable RemoteTransition remoteTransition);

    /** Get count of visible desktop tasks on the given display */
    int getVisibleTaskCount(int displayId);

    /** Perform cleanup transactions after the animation to split select is complete */
    oneway void onDesktopSplitSelectAnimComplete(in RunningTaskInfo taskInfo);

    /** Set listener that will receive callbacks about updates to desktop tasks */
    oneway void setTaskListener(IDesktopTaskListener listener);

    /** Move a task with given `taskId` to desktop */
    void moveToDesktop(int taskId, in DesktopModeTransitionSource transitionSource,
                        in @nullable RemoteTransition remoteTransition);

    /** Remove desktop on the given display */
    oneway void removeDesktop(int displayId);

    /** Move a task with given `taskId` to external display */
    void moveToExternalDisplay(int taskId);
}
```

注意这个方法列表**大量接受`RemoteTransition`参数**。这意味着:Launcher要负责用哪个过渡动画来把App"放到桌面"上。Shell提供的是能力和状态,动画由Launcher演。这是Shell Transition架构下"策略归Launcher、机制归Shell"的具体体现。

`showDesktopApp`的注释里还有一句重要的约束:**如果到了桌面任务数量上限,这个方法会顺带最小化一个task,调用方传的`remoteTransition`必须能处理这种"最小化"变化**。也就是说,Launcher传进来的过渡不能是"只演打开一个App"的专用动画。

### 8.11 StartingWindowController:启动窗口

目录`startingsurface/`(不是`startingwindow/`):

```text
IStartingWindow.aidl
IStartingWindowListener.aidl
StartingWindowController.java
StartingSurface.java
```

它在**`wmshell.splashscreen`线程**上跑:

```java
HandlerThread shellSplashscreenThread = new HandlerThread("wmshell.splashscreen",
        THREAD_PRIORITY_TOP_APP_BOOST);
```

用`THREAD_PRIORITY_TOP_APP_BOOST`而不是`THREAD_PRIORITY_DISPLAY`,是因为启动闪屏的时序要求跟着**App启动**走,和App进程的启动优先级对齐才不会露白。

启动闪屏有两种数据来源:

- `StartingSurface`:Shell自己画一个纯色/图标闪屏(速度最快)。
- `StartingWindow`:App提供`windowBackground`,由Shell做成静态窗口。

`SplashScreenExitAnimationUtils`(在`Shell/shared/src/com/android/wm/shell/shared/startingsurface/`)提供退出动画的共用逻辑。

### 8.12 KeyguardTransitionHandler:锁屏过渡

`keyguard/`目录只有两个文件,很干净:

```text
KeyguardTransitionHandler.java
KeyguardTransitions.java
```

`KeyguardTransitions.java`是**同进程Java接口,不是AIDL**:

```java
/**
 * Interface exposed to SystemUI Keyguard to register handlers for running
 * animations on keyguard visibility changes.
 *
 * TODO(b/274954192): Merge the occludeTransition and occludeByDream handlers and just let the
 * keyguard handler make the decision on which version it wants to play.
 */
@ExternalThread
public interface KeyguardTransitions {
    /**
     * Registers a set of remote transitions for Keyguard.
     */
    default void register(
            @NonNull IRemoteTransition unlockTransition,
            @NonNull IRemoteTransition appearTransition,
            @NonNull IRemoteTransition occludeTransition,
            @NonNull IRemoteTransition occludeByDreamTransition,
            @NonNull IRemoteTransition unoccludeTransition) {}

    /**
     * Notify whether keyguard has created a remote animation runner for next app launch.
     */
    default void setLaunchingActivityOverLockscreen(boolean isLaunchingActivityOverLockscreen) {}

    /**
     * Notifies Shell to start a keyguard transition directly.
     * @param keyguardShowing whether keyguard is showing or not.
     * @param aodShowing whether aod is showing or not.
     */
    default void startKeyguardTransition(boolean keyguardShowing, boolean aodShowing) {}
}
```

**这是Shell和SystemUI耦合最深的一个接口**。SystemUI的锁屏(`KeyguardViewMediator`)通过`register`把5个`IRemoteTransition`交给Shell:

| Transition | 何时用 |
|---|---|
| `unlockTransition` | 解锁(锁屏消失) |
| `appearTransition` | 锁屏出现 |
| `occludeTransition` | 锁屏被遮挡(如有Activity显示在锁屏上) |
| `occludeByDreamTransition` | 被Dream遮挡 |
| `unoccludeTransition` | 遮挡解除 |

`IRemoteTransition`是AIDL,意味着**动画的实现在SystemUI进程**,Shell只是调用方。为什么?因为锁屏UI(bouncer、时钟、通知)都在SystemUI进程,动画需要操作这些View。

`setLaunchingActivityOverLockscreen`这一条是给"锁屏上启动Activity"用的:通知Shell"这次App启动盖在锁屏上,动画要按这个来"。

`startKeyguardTransition`则是让SystemUI**主动发起**一次锁屏过渡,而不是等WMCore触发。

### 8.13 FreeformTaskListener与其它task监听

`freeform/`: `FreeformTaskListener.java`、`TaskChangeListener.kt`。它注册`TASK_LISTENER_TYPE_FREEFORM`,接管自由窗口的task。

`fullscreen/`: `FullscreenTaskListener.java`,注册`TASK_LISTENER_TYPE_FULLSCREEN`,是**最基础的task监听器**,所有非分屏、非PiP、非自由窗口的普通App都在它的`SparseArray<State> mTasks`里。

`taskview/`: `TaskViewFactoryController`、`TaskViewFactory`、`TaskView`、`TaskViewBase`、`TaskViewTaskController`、`TaskViewTransitions`。它让SystemUI(桌面模式的任务栏)能显示一个"活的"任务快照视图——不是截图,而是真实的surface。这需要`TaskViewTransitions`配合把任务surface重定向到TaskView里。

### 8.14 activityembedding、unfold、compatui、windowdecor

**`activityembedding/`**:`ActivityEmbeddingController`。Activity嵌入是大屏上的"双栏",两个Activity并排显示在同一个task里。它注册成`TaskFragmentOrganizer`而不是`TaskOrganizer`——操作粒度是task fragment(同一task内部的容器),不是task。

**`unfold/`**:`UnfoldAnimationController`、`UnfoldBackgroundController`。折叠屏展开/折叠时的过渡动画。

**`compatui/`**:`CompatUIWindowManager`。App没适配大屏时的"兼容模式"提示(比如提示用户"这个应用不支持分屏,是否重启为全屏")。它是Shell里少数**直接创建窗口**的模块。

**`windowdecor/`**:`DragResizeInputListener`、`OnTaskRepositionAnimationListener.kt`、`OnTaskResizeAnimationListener.kt`。桌面模式下窗口的标题栏、拖拽改大小的输入处理。

### 8.15 模块骨架总结

把所有模块抽象成模板,一个Shell功能模块基本上是这样:

1. **静态工厂或Dagger `@Inject`构造**,依赖注入:
   - `ShellInit`(注册初始化回调)
   - `ShellCommandHandler`(命令与dump)
   - `ShellController`(外部接口、配置变化、锁屏变化、用户切换)
   - `DisplayController`(显示信息)
   - `ShellTaskOrganizer`(需要接管task时)
   - `ShellExecutor mainExecutor`(通常是`@ShellMainThread`)
2. **构造函数里只做赋值和注册回调**,`shellInit.addInitCallback(this::onInit, this);`。
3. **`onInit()`里**:
   - 需要task的:`shellTaskOrganizer.addListenerForType(this, TASK_LISTENER_TYPE_XXX);`
   - 需要显示区的:创建/注册`DisplayAreaOrganizer`子类。
   - 需要外部接口的:`shellController.addExternalInterface(X.DESCRIPTOR, this::createExternalInterface, this);`
   - 需要监听的:`shellController.addConfigurationChangeListener/ addKeyguardChangeListener / addUserChangeListener`。
   - 需要命令的:`shellCommandHandler.addCommandCallback("xxx", this, this); shellCommandHandler.addDumpCallback(this::dump, this);`
4. **实现`RemoteCallable<Self>`**的三个方法:`getContext()`、`getRemoteCallExecutor()`、`createExternalInterface()`。
5. **外部接口实现类**继承`X.Stub`并实现`ExternalInterfaceBinder`,标`@BinderThread`,每个方法里`mMainExecutor.execute(() -> ...)`。
6. **同进程接口**用`@ExternalThread`标记,方法写成`default`空实现。

对照`Transitions`的实现(第7.10节),它完全符合这个模板。

## 9 SystemUI架构

### 9.1 进程与APK

SystemUI是一个**独立的APK**,不是framework jar的一部分。`packages/SystemUI/Android.bp`里最终产物是:

```python
android_app {
    name: "SystemUI",
    defaults: ["platform_app_defaults", "SystemUI_optimized_defaults", "wmshell_defaults"],
    static_libs: ["SystemUI-core"],
    resource_dirs: [],
    use_resource_processor: true,
    platform_apis: true,
    system_ext_specific: true,
    certificate: "platform",
    privileged: true,
    kotlincflags: ["-Xjvm-default=all"],
    dxflags: ["--multi-dex"],
    optimize: { proguard_flags_files: ["proguard.flags"] },
    required: ["privapp_whitelist_com.android.systemui"],
}
```

几个关键属性:

| 属性 | 值 | 含义 |
|---|---|---|
| `platform_apis` | true | 可以用`@hide`的framework API |
| `system_ext_specific` | true | 装到`system_ext`分区 |
| `certificate` | `platform` | platform签名 |
| `privileged` | true | 特权应用,可以用`signature|privileged`权限 |
| `dxflags` | `--multi-dex` | 方法数超限,必须multidex |

业务代码编到静态库`SystemUI-core`里:

```python
android_library {
    name: "SystemUI-core",
    srcs: [
        "src/**/*.kt",
        "src/**/*.java",
        "src/**/I*.aidl",
        ":ReleaseJavaFiles",
        "compose/features/src/**/*.kt",
        "compose/facade/enabled/src/**/*.kt",
    ],
    ...
}
```

注意`"src/**/I*.aidl"`这一条——**文件名以`I`开头且`.aidl`结尾的会被自动编译**,这也是Android构建系统的AIDL约定。

`defaults: [..., "wmshell_defaults"]`这一项很关键:它把WMShell相关的编译配置带进来,包括第2章讲的那些aconfig标志。**WMShell不是独立产物,而是通过`static_libs`链编进`SystemUI.apk`的**。

进程信息在`packages/SystemUI/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:androidprv="http://schemas.android.com/apk/prv/res/android"
        package="com.android.systemui"
        android:sharedUserId="android.uid.systemui"
        xmlns:tools="http://schemas.android.com/tools"
        coreApp="true">

    <application
        android:name=".SystemUIApplication"
        android:persistent="true"
        android:allowClearUserData="false"
        android:backupAgent=".backup.BackupHelper"
        android:killAfterRestore="false"
        android:hardwareAccelerated="true"
        android:process="com.android.systemui"
        android:supportsRtl="true"
        android:theme="@style/Theme.SystemUI"
        android:defaultToDeviceProtectedStorage="true"
        android:directBootAware="true"
        tools:replace="android:appComponentFactory"
        android:appComponentFactory=".PhoneSystemUIAppComponentFactory">
```

四个要点:

- `android:sharedUserId="android.uid.systemui"`:共享UID是`android.uid.systemui`,**不是system**。
- `android:persistent="true"`:常驻进程,被杀了系统会立刻重启它。
- `android:directBootAware="true"` + `defaultToDeviceProtectedStorage="true"`:支持直接启动(Direct Boot),在用户解锁前就能运行——这是锁屏能工作的前提。
- `android:appComponentFactory=".PhoneSystemUIAppComponentFactory"`:**这是SystemUI的注入入口**,9.6节详解。

### 9.2 启动链路:SystemServer到SystemUI

SystemUI由`system_server`直接拉起。`services/java/com/android/server/SystemServer.java`在`startOtherServices()`末尾:

```java
t.traceBegin("StartSystemUI");
try {
    startSystemUi(context, windowManagerF);
} catch (Throwable e) {
    reportWtf("starting System UI", e);
}
t.traceEnd();

t.traceEnd(); // startOtherServices
```

`startSystemUi`:

```java
private static void startSystemUi(Context context, WindowManagerService windowManager) {
    PackageManagerInternal pm = LocalServices.getService(PackageManagerInternal.class);
    Intent intent = new Intent();
    intent.setComponent(pm.getSystemUiServiceComponent());
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //Slog.d(TAG, "Starting service: " + intent);
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
    windowManager.onSystemUiStarted();
}
```

三点:

1. 用的是`context.startServiceAsUser(intent, UserHandle.SYSTEM)`,**不是`ActivityManagerInternal`**。
2. 组件名从`PackageManagerInternal.getSystemUiServiceComponent()`拿:

```java
public final ComponentName getSystemUiServiceComponent() {
    return ComponentName.unflattenFromString(getContext().getResources().getString(
            com.android.internal.R.string.config_systemUIServiceComponent));
}
```

对应的资源在`core/res/res/values/config.xml`:

```xml
<string name="config_systemUIServiceComponent" translatable="false"
        >com.android.systemui/com.android.systemui.SystemUIService</string>
```

**所以换SystemUI实现只要改这个overlay资源**,不需要改SystemServer代码。

3. 最后调`windowManager.onSystemUiStarted()`,告诉WMS"SystemUI起来了"。WMS据此知道可以开始显示系统栏。

**`StatusBarManagerService`不参与启动SystemUI**。它(`services/core/java/com/android/server/statusbar/StatusBarManagerService.java`)只提供`IStatusBarService`的注册接口,让SystemUI反过来注册`IStatusBar`回调。

完整链路:

```text
SystemServer.startOtherServices()
  └─ startSystemUi(context, windowManagerF)
       ├─ pm.getSystemUiServiceComponent()
       │    └─ "com.android.systemui/com.android.systemui.SystemUIService"
       ├─ context.startServiceAsUser(intent, UserHandle.SYSTEM)   // 跨进程
       └─ windowManager.onSystemUiStarted()

[进程 com.android.systemui]
Application.onCreate() : SystemUIApplication.onCreate()
  ├─ mInitializer = mContextAvailableCallback.onContextAvailable(this)
  │    └─ SystemUIAppComponentFactoryBase
  │         └─ PhoneSystemUIAppComponentFactory.createSystemUIInitializer(context)
  │              └─ SystemUIInitializerImpl(context)
  ├─ mSysUIComponent = mInitializer.getSysUIComponent()
  ├─ mBootCompleteCache = mSysUIComponent.provideBootCacheImpl()
  └─ 注册 ACTION_LOCKED_BOOT_COMPLETED 接收器

Service.onCreate() : SystemUIService.onCreate()
  ├─ ((SystemUIApplication) getApplication()).startSystemUserServicesIfNeeded()
  │    └─ 拓扑排序后逐个 CoreStartable.start()
  └─ startServiceAsUser(SystemUIAuxiliaryDumpService, UserHandle.SYSTEM)
```

### 9.3 SystemUIApplication

`packages/SystemUI/src/com/android/systemui/SystemUIApplication.java`。

类声明:

```java
public class SystemUIApplication extends Application implements
        SystemUIAppComponentFactoryBase.ContextInitializer, HasWMComponent {

    public static final String TAG = "SystemUIService";
    private static final boolean DEBUG = false;

    private BootCompleteCacheImpl mBootCompleteCache;
    private CoreStartable[] mServices;
    private boolean mServicesStarted;
    private SystemUIAppComponentFactoryBase.ContextAvailableCallback mContextAvailableCallback;
    private SysUIComponent mSysUIComponent;
    private SystemUIInitializer mInitializer;
    private ProcessWrapper mProcessWrapper;
```

**注意两点**:

- 没有`mComponents`字段(旧资料常提),现在是`mServices`。
- 实现`HasWMComponent`接口,说明这个Application持有WMShell的能力,并且`ComponentName`可以获得。

`onCreate`:

```java
@SuppressLint("RegisterReceiverViaContext")
@Override
public void onCreate() {
    super.onCreate();
    Log.v(TAG, "SystemUIApplication created.");
    // This line is used to setup Dagger's dependency injection and should be kept at the
    // top of this method.
    TimingsTraceLog log = new TimingsTraceLog("SystemUIBootTiming", Trace.TRACE_TAG_APP);
    log.traceBegin("DependencyInjection");
    mInitializer = mContextAvailableCallback.onContextAvailable(this);
    mSysUIComponent = mInitializer.getSysUIComponent();
    mBootCompleteCache = mSysUIComponent.provideBootCacheImpl();
    log.traceEnd();

    GlobalRootComponent rootComponent = mInitializer.getRootComponent();
    rootComponent.getMainLooper().setTraceTag(Trace.TRACE_TAG_APP);
    mProcessWrapper = rootComponent.getProcessWrapper();
    setTheme(R.style.Theme_SystemUI);
    ...
    if (mProcessWrapper.isSystemUser()) {
        IntentFilter bootCompletedFilter = new
                IntentFilter(Intent.ACTION_LOCKED_BOOT_COMPLETED);
        bootCompletedFilter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);
        ...
        registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (mBootCompleteCache.isBootComplete()) return;
                if (DEBUG) Log.v(TAG, "BOOT_COMPLETED received");
                unregisterReceiver(this);
                mBootCompleteCache.setBootComplete();
                if (mServicesStarted) {
                    final int N = mServices.length;
                    for (int i = 0; i < N; i++) {
                        notifyBootCompleted(mServices[i]);
                    }
                }
            }
        }, bootCompletedFilter);
        ...
    } else {
        if (isSubprocess()) {
            return;
        }
        startSecondaryUserServicesIfNeeded();
    }
}
```

**最容易被写错的一点**:boot完成监听的是`Intent.ACTION_LOCKED_BOOT_COMPLETED`,**不是`ACTION_BOOT_COMPLETED`**。原因是SystemUI需要尽早知道可以干活了,而`LOCKED_BOOT_COMPLETED`比`BOOT_COMPLETED`早得多(用户还没解锁)。优先级设为`SYSTEM_HIGH_PRIORITY`。

Dagger初始化是`onCreate`的**第一件事**,注释里明确写了"should be kept at the top of this method"。

`isSubprocess()`的判断是看进程名里有没有冒号:

```java
// com.android.systemui:screenshot 这样的子进程
ActivityThread.currentProcessName().contains(":")
```

在主进程里,`mProcessWrapper.isSystemUser()`区分system user和secondary user。子进程直接return,不做任何初始化。

### 9.4 CoreStartable:SystemUI的模块化单元

`packages/SystemUI/src/com/android/systemui/CoreStartable.java`:

```java
/**
 * Code that needs to be run when SystemUI is started.
 *
 * Which CoreStartable modules are loaded is controlled via the dagger graph. Bind them into the
 * CoreStartable map with code such as:
 *
 *  <pre>
 *  &#64;Binds
 *  &#64;IntoMap
 *  &#64;ClassKey(FoobarStartable::class)
 *  abstract fun bind(impl: FoobarStartable): CoreStartable
 *  </pre>
 *
 * If your CoreStartable depends on different CoreStartables starting before it, you can specify
 * another map binding listing out its dependencies:
 *  <pre>
 *  &#64;Provides
 *  &#64;IntoMap
 *  &#64;Dependencies  // Important! com.android.systemui.startable.Dependencies.
 *  &#64;ClassKey(FoobarStartable::class)
 *  fun providesDeps(): Set&lt;Class&lt;out CoreStartable&gt;&gt; {
 *      return setOf(OtherStartable::class.java)
 *  }
 *  </pre>
 *
 * @see SystemUIApplication#startSystemUserServicesIfNeeded()
 */
public interface CoreStartable extends Dumpable {
    String STARTABLE_DEPENDENCIES = "startable_dependencies";

    /** Main entry point for implementations. Called shortly after SysUI startup. */
    void start();

    @Override
    default void dump(@NonNull PrintWriter pw, @NonNull String[] args) {
    }

    /** Called to determine if the dumpable should be registered as critical or normal priority */
    default boolean isDumpCritical() {
        return true;
    }

    /** Called immediately after the system broadcasts
     * {@link android.content.Intent#ACTION_LOCKED_BOOT_COMPLETED} or during SysUI startup if the
     * property {@code sys.boot_completed} is already set to 1. ... */
    default void onBootCompleted() {
    }

    /** No op implementation that can be used when feature flagging on the Dagger Module level. */
    CoreStartable NOP = new Nop();

    class Nop implements CoreStartable {
        @Override
        public void start() {}
    }
}
```

四个方法/字段:

| 成员 | 何时调用 | 用途 |
|---|---|---|
| `start()` | SystemUI启动后不久 | 主入口 |
| `onBootCompleted()` | 收到`LOCKED_BOOT_COMPLETED`时 | 延迟到boot完成才做的事 |
| `dump()` | `dumpsys`时 | 调试输出 |
| `isDumpCritical()` | 启动时查询一次 | 决定dump优先级 |
| `NOP` | — | 按flag关掉某模块时的替身 |

**`NOP`这个常量的用法**:当某个功能被aconfig关掉时,Dagger模块里可以`bind`一个`NOP`而不是真实现,这样调用方不用判空。

依赖声明用的qualifier定义在`packages/SystemUI/src/com/android/systemui/startable/Dependencies.kt`:

```kotlin
/**
 * Allows a [CoreStartable] to declare that it must be started after its dependencies.
 *
 * This creates a partial, topological ordering. See [com.android.systemui.SystemUIApplication] for
 * how this ordering is enforced at runtime.
 */
@Qualifier @Documented @Retention(AnnotationRetention.RUNTIME) annotation class Dependencies()
```

**"partial, topological ordering"**这个说法很准确:只声明了依赖关系的那些才有顺序保证,没声明的是任意顺序(实际按类名排序)。

### 9.5 Startables如何被收集与启动

绑定在`packages/SystemUI/src/com/android/systemui/dagger/SystemUICoreStartableModule.kt`。这个文件的开头写着:

> DEPRECATED: DO NOT ADD THINGS TO THIS FILE. Add a feature specific dagger module...

也就是说新代码**不应该**往这里加。它现在的形式:

```kotlin
@Binds
@IntoMap
@ClassKey(KeyboardUI::class)
abstract fun bindKeyboardUI(sysui: KeyboardUI): CoreStartable

@Binds
@IntoMap
@ClassKey(NotificationChannels::class)
@PerUser
abstract fun bindNotificationChannels(sysui: NotificationChannels): CoreStartable

@Binds
@IntoMap
@ClassKey(WMShell::class)
abstract fun bindWMShell(sysui: WMShell): CoreStartable
```

**第4章讲的`WMShell`就是在这里绑定的**——它是SystemUI侧的一个`CoreStartable`。

`@PerUser`标记的进`getPerUserStartables()`,只在次要用户启动。

启动流程在`SystemUIApplication.startSystemUserServicesIfNeeded()`:

```java
public void startSystemUserServicesIfNeeded() {
    if (!shouldStartSystemUserServices()) {
        Log.wtf(TAG, "Tried starting SystemUser services on non-SystemUser");
        return;  // Per-user startables are handled in #startSystemUserServicesIfNeeded.
    }
    final String vendorComponent = mInitializer.getVendorComponent(getResources());

    // Sort the startables so that we get a deterministic ordering.
    Map<Class<?>, Provider<CoreStartable>> sortedStartables = new TreeMap<>(
            Comparator.comparing(Class::getName));
    sortedStartables.putAll(mSysUIComponent.getStartables());
    sortedStartables.putAll(mSysUIComponent.getPerUserStartables());
    startServicesIfNeeded(
            sortedStartables, "StartServices", vendorComponent);
}
```

**用`TreeMap`按类名排序**,先得到确定性的初始顺序,再在这个顺序基础上做拓扑排序。

`startServicesIfNeeded`的拓扑排序部分:

```java
// Perform a form of topological sort:
// 1) Iterate through a queue of all non-started startables
//   If the startable has all of its dependencies met - > start it
//   Else -> enqueue it for the next iteration
// 2) If anything was started and the "next" queue is not empty, loop back to 1
// 3) If we're done looping and there are any non-started startables left, throw an error.
...
do {
    startedAny = false;
    queue = nextQueue;
    nextQueue = new ArrayDeque<>(startables.size());

    while (!queue.isEmpty()) {
        Map.Entry<Class<?>, Provider<CoreStartable>> entry = queue.removeFirst();
        Class<?> cls = entry.getKey();
        Set<Class<? extends CoreStartable>> deps =
                mSysUIComponent.getStartableDependencies().get(cls);
        if (deps == null || startedStartables.containsAll(deps)) {
            String clsName = cls.getName();
            int i = serviceIndex;  // Copied to make lambda happy.
            timeInitialization(
                    clsName,
                    () -> mServices[i] = startStartable(clsName, entry.getValue()),
                    log,
                    metricsPrefix);
            startedStartables.add(cls);
            startedAny = true;
            serviceIndex++;
        } else {
            nextQueue.add(entry);
        }
    }
    numIterations++;
} while (startedAny && !nextQueue.isEmpty());

if (!nextQueue.isEmpty()) {
    ... Log.e(TAG, "Failed to start " + cls.getName() + ". Missing dependencies: [...");
    throw new RuntimeException("Failed to start all CoreStartables. Check logcat!");
}
Log.i(TAG, "Topological CoreStartables completed in " + numIterations + " iterations");
```

这是一个**朴素的"多轮扫描"拓扑排序**,不是Kahn算法那种用入度队列的写法。每一轮把能启动的都启动,然后带着剩下的进入下一轮。循环结束时如果队列非空,说明有**循环依赖**,直接抛`RuntimeException("Failed to start all CoreStartables. Check logcat!")`。

日志里的`Topological CoreStartables completed in N iterations`,N通常在3以内。**如果看到N很大(比如10以上),说明依赖链很深,启动时间会明显变长**。

启动完之后:

```java
for (serviceIndex = 0; serviceIndex < mServices.length; serviceIndex++) {
    final CoreStartable service = mServices[serviceIndex];
    if (mBootCompleteCache.isBootComplete()) {
        notifyBootCompleted(service);
    }
    if (service.isDumpCritical()) {
        dumpManager.registerCriticalDumpable(service);
    } else {
        dumpManager.registerNormalDumpable(service);
    }
}
mSysUIComponent.getInitController().executePostInitTasks();
log.traceEnd();

mServicesStarted = true;
```

`startStartable`里有Perfetto trace:

```java
private static CoreStartable startStartable(CoreStartable startable) {
    if (Trace.isEnabled()) {
        Trace.traceBegin(Trace.TRACE_TAG_APP, startable.getClass().getSimpleName() + ".start()");
    }
    startable.start();
    Trace.endSection();
    return startable;
}
```

所以抓Perfetto可以看到每个CoreStartable的`start()`耗时,这是SystemUI启动优化的主要数据来源。

`InitController`(`packages/SystemUI/src/com/android/systemui/InitController.java`,62行)是个后置任务队列:

```java
@SysUISingleton
public class InitController {
    public void addPostInitTask(Runnable task) { ... }
    public void executePostInitTasks() { ... }
}
```

所有`start()`跑完后统一执行。用于"等大家都初始化好再做的事"。

### 9.6 Dagger三层:SysUI的图长什么样

第4章从WMShell视角讲了`@Singleton`/`@WMSingleton`/`@SysUISingleton`三个scope,这里从SystemUI视角再讲一遍完整的组件树。

**`GlobalRootComponent`**(`packages/SystemUI/src/com/android/systemui/dagger/GlobalRootComponent.java`,82行):

```java
/**
 * Base root component for Dagger injection.
 *
 * This class is not actually annotated as a Dagger component, since it is not used directly as one.
 * Doing so generates unnecessary code bloat.
 *
 * See {@link ReferenceGlobalRootComponent} for the one actually used by AOSP.
 */
public interface GlobalRootComponent {
    interface Builder {
        @BindsInstance Builder context(Context context);
        @BindsInstance Builder instrumentationTest(@InstrumentationTest boolean test);
        GlobalRootComponent build();
    }
    WMComponent.Builder getWMComponentBuilder();
    SysUIComponent.Builder getSysUIComponent();
    InitializationChecker getInitializationChecker();
    @Main Looper getMainLooper();
    SystemPropertiesHelper getSystemPropertiesHelper();
    ProcessWrapper getProcessWrapper();
}
```

**关键设计**:这个接口**故意不加`@Component`注解**。注释说明了原因:"Doing so generates unnecessary code bloat"。真正的Dagger组件是继承它的`ReferenceGlobalRootComponent`。

**这意味着OEM要换Dagger图时,只需要提供一个自己的`ReferenceGlobalRootComponent`等价物,而不动`GlobalRootComponent`这个契约。**

`ReferenceGlobalRootComponent`:

```java
@Singleton
@Component(modules = {GlobalModule.class})
public interface ReferenceGlobalRootComponent extends GlobalRootComponent {
    @Component.Builder
    interface Builder extends GlobalRootComponent.Builder {
        ReferenceGlobalRootComponent build();
    }
    ReferenceSysUIComponent.Builder getSysUIComponent();
}
```

`GlobalModule`提供全局单例,并include了插件模块:

```java
@Module(includes = {
        AndroidInternalsModule.class,
        FrameworkServicesModule.class,
        GlobalConcurrencyModule.class,
        GlobalCoroutinesModule.class,
        UnfoldTransitionModule.class,
        PluginsModule.class,
})
public class GlobalModule {
    @Provides @Application
    public Context provideApplicationContext(Context context) {
        return context.getApplicationContext();
    }
    ...
}
```

**`SysUIComponent`**(192行)是子组件:

```java
@SysUISingleton
@Subcomponent(modules = {
        DefaultComponentBinder.class,
        DependencyProvider.class,
        NotificationInsetsModule.class,
        QsFrameTranslateModule.class,
        SystemUIModule.class,
        SystemUICoreStartableModule.class,
        ReferenceSystemUIModule.class})
public interface SysUIComponent {
    @SysUISingleton
    @Subcomponent.Builder
    interface Builder {
        @BindsInstance Builder setShell(ShellInterface s);
        @BindsInstance Builder setPip(Optional<Pip> p);
        // ... 共 13 个 @BindsInstance setter
        SysUIComponent build();
    }

    @SysUISingleton BootCompleteCacheImpl provideBootCacheImpl();
    @SysUISingleton @GlobalConfig ConfigurationController getConfigurationController();
    @SysUISingleton @GlobalConfig ConfigurationForwarder getConfigurationForwarder();
    @SysUISingleton ContextComponentHelper getContextComponentHelper();
    @SysUISingleton Dependency createDependency();
    @SysUISingleton DumpManager createDumpManager();
    @SysUISingleton InitController getInitController();

    Map<Class<?>, Provider<CoreStartable>> getStartables();
    @PerUser Map<Class<?>, Provider<CoreStartable>> getPerUserStartables();
    @Dependencies Map<Class<?>, Set<Class<? extends CoreStartable>>> getStartableDependencies();

    void inject(SystemUIAppComponentFactoryBase factory);
    void inject(KeyguardSliceProvider keyguardSliceProvider);
    void inject(PeopleProvider peopleProvider);
}
```

`ReferenceSysUIComponent`(55行)继承它:

```java
@SysUISingleton
@Subcomponent(modules = {DefaultComponentBinder.class, DependencyProvider.class,
        NotificationInsetsModule.class, QsFrameTranslateModule.class, SystemUIModule.class,
        SystemUICoreStartableModule.class, SysUIUnfoldModule.class, ReferenceSystemUIModule.class})
public interface ReferenceSysUIComponent extends SysUIComponent {
    @Subcomponent.Builder
    interface Builder extends SysUIComponent.Builder {
        ReferenceSysUIComponent build();
    }
    void inject(CustomizationProvider customizationProvider);
}
```

三层的完整层次:

```text
@Singleton
ReferenceGlobalRootComponent  (modules = GlobalModule)
  │
  ├─ getWMComponentBuilder() ──> WMComponent          @WMSingleton  (第4章)
  │
  └─ getSysUIComponent() ──────> ReferenceSysUIComponent  @SysUISingleton
                                    (modules = SystemUIModule,
                                     SystemUICoreStartableModule, ...)
```

**`getStartables()`返回的是`Map<Class<?>, Provider<CoreStartable>>`**,靠Dagger的multibinding(`@IntoMap` + `@ClassKey`)填充。**不是反射扫描**,所以R8/ProGuard不会误删,启动也不需要遍历dex。

### 9.7 SystemUIInitializer:替代旧的SystemUIFactory

`packages/SystemUI/src/com/android/systemui/SystemUIInitializer.java`。注意`TAG`还是旧的:

```java
public abstract class SystemUIInitializer {
    private static final String TAG = "SystemUIFactory";
    private final Context mContext;
    private GlobalRootComponent mRootComponent;
    private WMComponent mWMComponent;
    private SysUIComponent mSysUIComponent;
    private InitializationChecker mInitializationChecker;

    protected abstract GlobalRootComponent.Builder getGlobalRootComponentBuilder();
```

`init()`是核心:

```java
public void init(boolean fromTest) throws ExecutionException, InterruptedException {
    mRootComponent = getGlobalRootComponentBuilder()
            .context(mContext)
            .instrumentationTest(fromTest)
            .build();
    mInitializationChecker = mRootComponent.getInitializationChecker();
    boolean initializeComponents = mInitializationChecker.initializeComponents();
    setupWmComponent(mContext);

    SysUIComponent.Builder builder = mRootComponent.getSysUIComponent();
    if (initializeComponents) {
        builder = prepareSysUIComponentBuilder(builder, mWMComponent)
                .setShell(mWMComponent.getShell())
                .setPip(mWMComponent.getPip())
                .setSplitScreen(mWMComponent.getSplitScreen())
                .setOneHanded(mWMComponent.getOneHanded())
                .setBubbles(mWMComponent.getBubbles())
                .setTaskViewFactory(mWMComponent.getTaskViewFactory())
                .setShellTransitions(mWMComponent.getShellTransitions())
                .setKeyguardTransitions(mWMComponent.getKeyguardTransitions())
                .setStartingSurface(mWMComponent.getStartingSurface())
                .setDisplayAreaHelper(mWMComponent.getDisplayAreaHelper())
                .setRecentTasks(mWMComponent.getRecentTasks())
                .setBackAnimation(mWMComponent.getBackAnimation())
                .setDesktopMode(mWMComponent.getDesktopMode());
        mWMComponent.init();
    } else { /* 测试环境:setShell(new ShellInterface(){}) 等空实现 */ }
    mSysUIComponent = builder.build();

    // Every other part of our codebase currently relies on Dependency, so we
    // really need to ensure the Dependency gets initialized early on.
    Dependency dependency = mSysUIComponent.createDependency();
    dependency.start();
}
```

**这是整个WMShell与SystemUI接合的核心代码**,逐条:

1. 建root component。
2. `setupWmComponent(mContext)`——**在build SysUIComponent之前**建好WMComponent。如果`config_enableShellMainThread`为true,这一步在Shell线程上跑:

```java
runWithScissors(..., 5000)  // 把WMComponent建在独立shell HandlerThread上
```

3. 把WMComponent的13个对象通过`setXxx()`注入SysUIComponent.Builder。
4. `mWMComponent.init()`——**注意顺序:先注入,后init**。`init()`会触发WMComponent内所有`ShellInit`回调。
5. `builder.build()`得到SysUIComponent。
6. 立即`createDependency().start()`。

第6步的注释解释了原因:"Every other part of our codebase currently relies on Dependency"——`Dependency`是一个老旧但被广泛依赖的静态持有类,必须最早初始化。

`getVendorComponent`:

```java
public String getVendorComponent(Resources resources) {
    return resources.getString(R.string.config_systemUIVendorServiceComponent);
}
```

OEM可以在这里塞一个额外的CoreStartable,它的启动在拓扑排序之后单独做(`startAdditionalStartable`用的是反射):

```java
private static CoreStartable startAdditionalStartable(String clsName) {
    // Class.forName(clsName).getDeclaredConstructor().newInstance()
}
```

**这是SystemUI里唯一还在用反射创建CoreStartable的地方**,因为它要加载的类来自OEM的overlay,编译期不存在。

`SystemUIInitializerImpl.kt`是AOSP默认实现:

```kotlin
class SystemUIInitializerImpl(context: Context) : SystemUIInitializer(context) {
    override fun getGlobalRootComponentBuilder(): GlobalRootComponent.Builder {
        return DaggerReferenceGlobalRootComponent.builder()
    }
}
```

OEM替换的完整机制在`packages/SystemUI/README.md`里说明:定制方重写`SystemUIAppComponentFactoryBase#createSystemUIInitializer(Context)`返回自己的`SystemUIInitializer`,就能换上自己的Dagger图。**不再需要像旧版那样反射式地替换`SystemUIFactory`的静态字段。**

### 9.8 AppComponentFactory:SystemUI的注入入口

Android的`AppComponentFactory`允许应用自己实例化Application、Activity、Service、Receiver、Provider。SystemUI用它来做构造函数注入。

`packages/SystemUI/src/com/android/systemui/SystemUIAppComponentFactoryBase.kt`(187行):

- 有个静态的`systemUIInitializer`单例。
- `createSystemUIInitializerInternal()`调用`initializer.init(false)`并`initializer.sysUIComponent.inject(this)`。
- 重写`instantiateApplicationCompat`、`instantiateActivityCompat`、`instantiateServiceCompat`、`instantiateReceiverCompat`、`instantiateProviderCompat`,通过`ContextComponentHelper`解析并实例化。
- 定义了`interface ContextInitializer { fun setContextAvailableCallback(callback) }`和`fun interface ContextAvailableCallback { fun onContextAvailable(context): SystemUIInitializer }`。

`SystemUIApplication`实现`ContextInitializer`,所以`onCreate`里那句`mContextAvailableCallback.onContextAvailable(this)`就是这条链路。

实现类`PhoneSystemUIAppComponentFactory.kt`只有23行:

```kotlin
class PhoneSystemUIAppComponentFactory : SystemUIAppComponentFactoryBase() {
    override fun createSystemUIInitializer(context: Context) = SystemUIInitializerImpl(context)
}
```

`SystemUIService`的构造函数因此可以用`@Inject`:

```java
@Inject
public SystemUIService(
        @Main Handler mainHandler,
        DumpHandler dumpHandler,
        BroadcastDispatcher broadcastDispatcher,
        LogBufferEulogizer logBufferEulogizer,
        LogBufferFreezer logBufferFreezer,
        BatteryStateNotifier batteryStateNotifier,
        UncaughtExceptionPreHandlerManager uncaughtExceptionPreHandlerManager) {
    super();
    mMainHandler = mainHandler;
    ...
}
```

**普通Android Service的构造函数是系统调用的,不能带参数。** SystemUI能这么做,是因为`AppComponentFactory`拦截了Service的创建过程。这是SystemUI代码风格和普通App最不一样的地方。

`SystemUIService.onCreate`:

```java
@Override
public void onCreate() {
    super.onCreate();

    // Start all of SystemUI
    ((SystemUIApplication) getApplication()).startSystemUserServicesIfNeeded();

    // Finish initializing dump logic
    mLogBufferFreezer.attach(mBroadcastDispatcher);

    // Attempt to dump all LogBuffers for any uncaught exception
    mUncaughtExceptionPreHandlerManager.registerHandler(
            (thread, throwable) -> mLogBufferEulogizer.record(throwable));

    // If configured, set up a battery notification
    if (getResources().getBoolean(R.bool.config_showNotificationForUnknownBatteryState)) {
        mBatteryStateNotifier.startListening();
    }

    if (Build.IS_DEBUGGABLE && SystemProperties.getBoolean("debug.crash_sysui", false)) {
        throw new RuntimeException();
    }

    if (Build.IS_DEBUGGABLE) {
        // b/71353150 - looking for leaked binder proxies
        BinderInternal.nSetBinderProxyCountEnabled(true);
        BinderInternal.nSetBinderProxyCountWatermarks(
                /* high= */ 1000, /* low= */ 900, /* warning= */ 950);
        BinderInternal.setBinderProxyCountCallback(
                new BinderInternal.BinderProxyCountEventListener() {
                    @Override
                    public void onLimitReached(int uid) {
                        Slog.w(SystemUIApplication.TAG,
                                "uid " + uid + " sent too many Binder proxies to uid "
                                + Process.myUid());
                    }
                }, mMainHandler);
    }

    // Bind the dump service so we can dump extra info during a bug report
    startServiceAsUser(
            new Intent(getApplicationContext(), SystemUIAuxiliaryDumpService.class),
            UserHandle.SYSTEM);
}
```

调试构建里有三条有用的东西:

- **Binder代理计数**:`nSetBinderProxyCountEnabled(true)`,超过900警告、1000报警。SystemUI是Binder代理泄漏的重灾区(BinderProxy对象不释放)。
- **`debug.crash_sysui`系统属性**:设为true时SystemUI启动即崩,用于测试RescueParty。
- **`LogBufferEulogizer`**:崩溃时把日志缓冲区落盘,否则进程死了日志也没了。

### 9.9 多用户

SystemUI的启动分三档:

| 触发 | 方法 | 启动哪些 |
|---|---|---|
| system user的SystemUIService | `startSystemUserServicesIfNeeded` | `getStartables()` + `getPerUserStartables()` |
| 非system user | `startSecondaryUserServicesIfNeeded` | 只有`@PerUser`的 |
| 非system user的辅助Service | `SystemUISecondaryUserService` | 同上 |

`startSecondaryUserServicesIfNeeded`从`SystemUIApplication.onCreate`的else分支调用(主进程且非system user)。

`SystemUISecondaryUserService`自己会检查身份:

```java
@Override
public void onCreate() {
    super.onCreate();
    if (mProcessWrapper.isSystemUser()) {
        Log.w(TAG, "SecondaryServices started for System User. Stopping it.");
        stopSelf();
        return;
    }
    ((SystemUIApplication) getApplication()).startSecondaryUserServicesIfNeeded();
}
```

### 9.10 插件框架

插件让OEM/第三方在不改SystemUI源码的情况下扩展SystemUI。

**目录结构**(注意:插件接口**不在**`src/com/android/systemui/plugins/`下):

| 路径 | 内容 |
|---|---|
| `packages/SystemUI/plugin_core/src/com/android/systemui/plugins/` | **插件接口定义**:`Plugin.java`、`PluginManager.java`、`PluginListener.java`、`PluginFragment.java`、`PluginLifecycleManager.java`、`PluginWrapper.kt`、`ProtectedPluginListener.kt` |
| `packages/SystemUI/plugin_core/src/com/android/systemui/plugins/annotations/` | `ProvidesInterface.java`、`Requires.java`、`Requirements.java`、`DependsOn.java`、`Dependencies.java`、`ProtectedInterface.kt` |
| `packages/SystemUI/plugin_core/processor/src/com/android/systemui/plugins/processor/` | `ProtectedPluginProcessor.kt`、`TabbedWriter.kt`(注解处理器) |
| `packages/SystemUI/plugin/` | `SystemUIPluginLib`(`java_library`)+ `bcsmartspace/` + `ExamplePlugin/` |
| `packages/SystemUI/src/com/android/systemui/plugins/` | `PluginDependencyProvider.java`、`PluginEnablerImpl.java`、`PluginsModule.java` |
| `packages/SystemUI/shared/src/com/android/systemui/shared/plugins/` | `PluginManagerImpl.java`、`PluginActionManager.java`、`PluginInstance.java`、`PluginEnabler.java`、`PluginPrefs.java`、`VersionInfo.java` |

`PluginManager`接口(`plugin_core`,58行):

```java
public interface PluginManager {

    String PLUGIN_CHANGED = "com.android.systemui.action.PLUGIN_CHANGED";
    String NOTIFICATION_CHANNEL_ID = "ALR";

    /** Returns plugins that don't get disabled when an exceptoin occurs. */
    String[] getPrivilegedPlugins();

    <T extends Plugin> void addPluginListener(PluginListener<T> listener, Class<T> cls);
    <T extends Plugin> void addPluginListener(PluginListener<T> listener, Class<T> cls,
            boolean allowMultiple);
    <T extends Plugin> void addPluginListener(String action, PluginListener<T> listener,
            Class<T> cls);
    <T extends Plugin> void addPluginListener(String action, PluginListener<T> listener,
            Class<T> cls, boolean allowMultiple);

    void removePluginListener(PluginListener<?> listener);

    <T> boolean dependsOn(Plugin p, Class<T> cls);

    class Helper {
        public static <P> String getAction(Class<P> cls) {
            ProvidesInterface info = cls.getDeclaredAnnotation(ProvidesInterface.class);
            if (info == null) {
                throw new RuntimeException(cls + " doesn't provide an interface");
            }
            if (TextUtils.isEmpty(info.action())) {
                throw new RuntimeException(cls + " doesn't provide an action");
            }
            return info.action();
        }
    }
}
```

插件接口上的注解(`annotations/ProvidesInterface.java`):

```java
/**
 * Should be added to all interfaces in plugin lib to specify their
 * current version and optionally their action to implement the plugin.
 */
@Retention(RetentionPolicy.RUNTIME)
public @interface ProvidesInterface {
    int version();
    String action() default "";
}
```

`Plugin`接口(`plugin_core`,132行):

```java
public interface Plugin {
    /** @deprecated @see Requires */
    @Deprecated
    @ProtectedReturn(statement = "return -1;")
    default int getVersion() {
        return -1;   // -1 表示支持新的 Requires 模型
    }
    default void onCreate(Context sysuiContext, Context pluginContext) {}
    default void onDestroy() {}
}
```

`getVersion()`返回`-1`表示"这个插件用新版的`@Requires`模型声明依赖",而不是老的版本号模型。

Dagger装配在`packages/SystemUI/src/com/android/systemui/plugins/PluginsModule.java`:

```java
@Module(includes = {GlobalConcurrencyModule.class})
public abstract class PluginsModule {
    public static final String PLUGIN_THREAD = "plugin_thread";
    public static final String PLUGIN_DEBUG = "plugin_debug";
    public static final String PLUGIN_PRIVILEGED = "plugin_privileged";

    @Provides @Named(PLUGIN_DEBUG)
    static boolean providesPluginDebug() { return Build.IS_DEBUGGABLE; }

    @Binds
    abstract PluginEnabler bindsPluginEnablerImpl(PluginEnablerImpl impl);

    @Provides @Singleton
    static PluginInstance.Factory providesPluginInstanceFactory(
            @Named(PLUGIN_PRIVILEGED) List<String> privilegedPlugins,
            @Named(PLUGIN_DEBUG) boolean isDebug) {
        return new PluginInstance.Factory(
                PluginModule.class.getClassLoader(),
                new PluginInstance.InstanceFactory<>(),
                new PluginInstance.VersionCheckerImpl(),
                privilegedPlugins,
                isDebug);
    }
    ...
    @Provides @Singleton @Named(PLUGIN_THREAD)
    static Executor providesPluginExecutor(ThreadFactory threadFactory) {
        return threadFactory.buildExecutorOnNewThread("plugin");
    }
    ...
    @Provides @Named(PLUGIN_PRIVILEGED)
    static List<String> providesPrivilegedPlugins(Context context) {
        return Arrays.asList(context.getResources().getStringArray(R.array.config_pluginAllowlist));
    }
}
```

**注意`PluginInstance.Factory`的base classloader是`PluginModule.class.getClassLoader()`**,也就是SystemUI自己的classloader。这样插件才能访问SystemUI的类。

`PluginManagerImpl`(`shared/plugins/`,320行)继承`BroadcastReceiver`:

```java
public class PluginManagerImpl extends BroadcastReceiver implements PluginManager {
```

它监听:

- `ACTION_PACKAGE_ADDED` / `CHANGED` / `REPLACED` / `REMOVED`
- `PLUGIN_CHANGED`(`com.android.systemui.action.PLUGIN_CHANGED`)
- `DISABLE_PLUGIN`(`com.android.systemui.action.DISABLE_PLUGIN`)
- `ACTION_USER_UNLOCKED`

packages相关的接收用`PluginActionManager.PLUGIN_PERMISSION`且`Context.RECEIVER_EXPORTED_UNAUDITED`。

**崩溃时的处理**是插件框架最有意思的部分:构造时如果`preHandlerManager != null`,注册一个`PluginExceptionHandler`。插件崩溃时按栈帧反查是哪个插件,如果定位到了就禁用那一个;定位不到就`disableAll()`,并抛`CrashWhilePluginActiveException`。

类加载的父加载器做了过滤(`PluginInstance.Factory#getParentClassLoader`):

```java
private ClassLoader getParentClassLoader(ClassLoader baseClassLoader) {
    return new PluginManagerImpl.ClassLoaderFilter(
            baseClassLoader,
            "androidx.constraintlayout.widget",
            "com.android.systemui.common",
            "com.android.systemui.log",
            "com.android.systemui.plugin");
}
```

`PluginManagerImpl.ClassLoaderFilter`是一个自定义`ClassLoader`,**只对列出的这几个包前缀委托给SystemUI的classloader,其他都走系统classloader**。这样插件自带的库不会和SystemUI的冲突。

插件APK的构建(`packages/SystemUI/plugin/ExamplePlugin/Android.bp`):

```python
android_app {
    name: "ExamplePlugin",
    use_resource_processor: true,
    libs: ["SystemUIPluginLib"],
    certificate: "platform",
    optimize: { enabled: false, },
    srcs: ["src/**/*.java"],
    platform_apis: true,
}
```

插件Manifest需要签名权限:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.android.systemui.plugin.testoverlayplugin">
    <uses-permission android:name="com.android.systemui.permission.PLUGIN" />
    <application>
        <service android:name=".SampleOverlayPlugin" android:label="@string/plugin_label" android:exported="false">
            <intent-filter>
                <action android:name="com.android.systemui.action.PLUGIN_OVERLAY" />
            </intent-filter>
        </service>
    </application>
</manifest>
```

`com.android.systemui.permission.PLUGIN`的protectionLevel是`signature`,所以**普通第三方App做不了SystemUI插件**,必须platform签名。这解释了为什么插件框架的注释里反复强调"privileged plugin"。

### 9.11 主题与RRO

**`packages/SystemUI/theme`这个目录不存在**(旧资料常写错)。主题代码在`packages/SystemUI/src/com/android/systemui/theme/`:

```text
ThemeModule.java
ThemeOverlayApplier.java
ThemeOverlayController.java   (874行)
```

```java
public class ThemeOverlayController implements CoreStartable, Dumpable {
    private final ThemeOverlayApplier mThemeManager;
    ...
    protected FabricatedOverlay newFabricatedOverlay(String name) {
        return new FabricatedOverlay.Builder("com.android.systemui", name, "android").build();
    }
```

它用**运行时`FabricatedOverlay`**(从壁纸取色得到的neutral/accent/dynamic色)+ RRO来给SystemUI上色。

主题类别定义在`ThemeOverlayApplier.java`:

```java
static final String OVERLAY_CATEGORY_DYNAMIC_COLOR = ...
static final String OVERLAY_CATEGORY_ACCENT_COLOR = ...
static final String OVERLAY_CATEGORY_SYSTEM_PALETTE = ...
static final String OVERLAY_CATEGORY_THEME_STYLE = ...
static final String OVERLAY_CATEGORY_FONT = "android.theme.customization.font";
static final String OVERLAY_CATEGORY_SHAPE = ...
static final String OVERLAY_CATEGORY_ICON_ANDROID = ...
static final String OVERLAY_CATEGORY_ICON_SYSUI = ...
static final List<String> THEME_CATEGORIES = Lists.newArrayList(
        OVERLAY_CATEGORY_SYSTEM_PALETTE, OVERLAY_CATEGORY_ICON_LAUNCHER, OVERLAY_CATEGORY_SHAPE,
        OVERLAY_CATEGORY_FONT, OVERLAY_CATEGORY_ACCENT_COLOR, OVERLAY_CATEGORY_DYNAMIC_COLOR,
        OVERLAY_CATEGORY_ICON_ANDROID, OVERLAY_CATEGORY_ICON_SYSUI, ...);
```

需要的权限在Manifest里:`MODIFY_THEME_OVERLAY`、`CHANGE_OVERLAY_PACKAGES`、`SET_THEME_OVERLAY_CONTROLLER_READY`。

编译期的RRO主题(导航栏模式切换等)在`frameworks/base/packages/overlays/`,第10章详述。

## 10 SystemUI核心UI

### 10.1 三大系统栏窗口

SystemUI在WMS眼里只有三个"系统栏窗口",由窗口类型区分。类型常量定义在`core/java/android/view/WindowManager.java`(`FIRST_SYSTEM_WINDOW = 2000`):

```java
/**
 * Window type: the status bar.  There can be only one status bar
 * window; it is placed at the top of the screen, and all other
 * windows are shifted down so they are below it.
 * In multiuser systems shows on all users' windows.
 */
public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;   // 2000
```

**`TYPE_STATUS_BAR`是public的,没有`@hide`**——这是历史遗留,它是最早公开的系统窗口类型。

```java
/**
 * Window type: Navigation bar (when distinct from status bar)
 * In multiuser systems shows on all users' windows.
 * @hide
 */
public static final int TYPE_NAVIGATION_BAR = FIRST_SYSTEM_WINDOW+19;    // 2019
```

```java
/**
 * Window type: the notification shade and keyguard. There can be only one status bar
 * window; it is placed at the top of the screen, and all other
 * windows are shifted down so they are below it.
 * In multiuser systems shows on all users' windows.
 * @hide
 */
public static final int TYPE_NOTIFICATION_SHADE = FIRST_SYSTEM_WINDOW + 40;   // 2040
```

三个类型的编号不是连续的(2000、2019、2040),中间夹着别的系统窗口类型。

相关的其它类型:

| 常量 | 值 | 说明 |
|---|---|---|
| `TYPE_STATUS_BAR_PANEL` | 2014 | 已`@removed` |
| `TYPE_STATUS_BAR_SUB_PANEL` | 2017 | `@hide` |
| `TYPE_NAVIGATION_BAR_PANEL` | 2024 | `@hide`,导航栏用于无缝旋转的透明面板 |
| `TYPE_QS_DIALOG` | 2035 | `@hide`,快捷设置对话框 |
| `TYPE_SCREENSHOT` | 2036 | 截图 |
| `TYPE_APPLICATION_OVERLAY` | 2038 | 应用悬浮窗 |
| `TYPE_STATUS_BAR_ADDITIONAL` | 2041 | `@hide`,把状态栏放到屏幕左侧/底部 |

**WMS怎么认出这三个窗口?** `DisplayPolicy.addWindowLw`只认这三种类型:

```java
private WindowState mStatusBar = null;
private volatile WindowState mNotificationShade;
private WindowState mNavigationBar = null;
...
void addWindowLw(WindowState win, WindowManager.LayoutParams attrs) {
    switch (attrs.type) {
        case TYPE_NOTIFICATION_SHADE:
            mNotificationShade = win;
            break;
        case TYPE_STATUS_BAR:
            mStatusBar = win;
            break;
        case TYPE_NAVIGATION_BAR:
            mNavigationBar = win;
            break;
    }
    if ((attrs.privateFlags & PRIVATE_FLAG_IMMERSIVE_CONFIRMATION_WINDOW) != 0) {
        mImmersiveConfirmationWindowExists = true;
    }
    if (attrs.providedInsets != null) {
        for (int i = attrs.providedInsets.length - 1; i >= 0; i--) {
            final InsetsFrameProvider provider = attrs.providedInsets[i];
            // The index of the provider and corresponding insets types cannot change at
            // runtime as ensured in WMS. ...
```

这段代码是**Shell与SystemUI协作面的核心**。两层含义:

1. WMS通过`attrs.type`认出三类系统栏窗口并分别记账。
2. WMS读取`attrs.providedInsets`,把SystemUI声明的inset提供者注册进insets系统。

**SystemUI能用这些type加窗,是因为它在Manifest里持有`INTERNAL_SYSTEM_WINDOW`等权限:**

```xml
<uses-permission android:name="android.permission.INTERNAL_SYSTEM_WINDOW" />
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
<uses-permission android:name="android.permission.STATUS_BAR_SERVICE" />
<uses-permission android:name="android.permission.STATUS_BAR" />
```

`TYPE_KEYGUARD`已经标`@removed`——**锁屏没有独立窗口类型了,它和通知面板共用`TYPE_NOTIFICATION_SHADE`窗口**。

### 10.2 状态栏窗口

状态栏的创建不在`CentralSurfaces`里,而在`statusbar/window/`目录:

```text
StatusBarWindowController.kt
StatusBarWindowControllerImpl.java
StatusBarWindowControllerStore.kt
StatusBarWindowModule.kt
StatusBarWindowStateController.kt
StatusBarWindowStateListener.kt
StatusBarWindowView.java
StatusBarWindowViewInflater.kt
```

`StatusBarWindowControllerImpl.attach()`:

```java
@Override
public void attach() {
    ...
    Trace.beginSection("StatusBarWindowController.getBarLayoutParams");
    mLp = getBarLayoutParams(mContext.getDisplay().getRotation());
    Trace.endSection();

    mWindowManager.addView(mStatusBarWindowView, mLp);
    mLpChanged.copyFrom(mLp);

    mContentInsetsProvider.addCallback(this::calculateStatusBarLocationsForAllRotations);
    calculateStatusBarLocationsForAllRotations();
    mIsAttached = true;
    apply(mCurrentState);
}
```

**窗口参数是这段(`getBarLayoutParamsForRotation`):**

```java
private WindowManager.LayoutParams getBarLayoutParamsForRotation(int rotation) {
    int height = SystemBarUtils.getStatusBarHeightForRotation(mContext, rotation);
    WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
            WindowManager.LayoutParams.MATCH_PARENT,
            height,
            WindowManager.LayoutParams.TYPE_STATUS_BAR,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                    | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
            PixelFormat.TRANSLUCENT);
    lp.privateFlags |= PRIVATE_FLAG_COLOR_SPACE_AGNOSTIC;
    lp.token = new Binder();
    lp.gravity = Gravity.TOP;
    lp.setFitInsetsTypes(0 /* types */);
    lp.setTitle("StatusBar");
    lp.packageName = mContext.getPackageName();
    lp.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
    final InsetsFrameProvider gestureInsetsProvider =
            new InsetsFrameProvider(mInsetsSourceOwner, 0, mandatorySystemGestures());
    ...
    lp.providedInsets = new InsetsFrameProvider[] {
            new InsetsFrameProvider(mInsetsSourceOwner, 0, statusBars())
                    .setInsetsSize(getInsets(height)),
            new InsetsFrameProvider(mInsetsSourceOwner, 0, tappableElement())
                    .setInsetsSize(getInsets(height)),
            gestureInsetsProvider
    };
    return lp;
}
```

逐个字段解释:

| 字段 | 值 | 为什么 |
|---|---|---|
| 宽 | `MATCH_PARENT` | 状态栏横跨整屏 |
| 高 | `getStatusBarHeightForRotation(...)` | 随旋转变化 |
| 类型 | `TYPE_STATUS_BAR` | WMS据此认账 |
| flags | `NOT_FOCUSABLE` | 状态栏不接收焦点,否则会抢走App的输入 |
| flags | `SPLIT_TOUCH` | 触摸事件分发给多个窗口 |
| flags | `DRAWS_SYSTEM_BAR_BACKGROUNDS` | SystemUI自己画系统栏背景,不是WMS画 |
| 格式 | `TRANSLUCENT` | 半透明,底下能看到App |
| `token` | `new Binder()` | 自己造一个token,不是Activity的token |
| `gravity` | `TOP` | 贴顶 |
| `setFitInsetsTypes(0)` | 0 | **不避开任何inset**,状态栏自己不被任何东西挤 |
| `layoutInDisplayCutoutMode` | `ALWAYS` | 允许延伸到刘海区域 |

**`providedInsets`是新版的关键变化**。旧写法是窗口加进去之后由WMS回调询问"你提供多少inset",新写法是**加窗时就把inset提供者声明清楚**:

```java
lp.providedInsets = new InsetsFrameProvider[] {
        new InsetsFrameProvider(mInsetsSourceOwner, 0, statusBars())
                .setInsetsSize(getInsets(height)),
        new InsetsFrameProvider(mInsetsSourceOwner, 0, tappableElement())
                .setInsetsSize(getInsets(height)),
        gestureInsetsProvider
};
```

三个provider分别声明:

- `statusBars()`:高度`getInsets(height)`,即`Insets.of(0, height, 0, 0)`。
- `tappableElement()`:状态栏区域内的可点击元素。
- `mandatorySystemGestures()`:必选系统手势区,防止App把返回手势区吃掉。

**强制显示状态栏**用的是另一个字段:

```java
mLpChanged.forciblyShownTypes |= WindowInsets.Type.statusBars();
```

状态栏还为旋转预准备了四份参数(`lp.paramsForRotation[4]`),旋转时不用重新算,直接换。

**状态栏窗口的可见性不由自己决定**,而是由`StatusBarWindowStateController`根据多处状态汇总。这就是为什么状态栏代码里到处是"readiness"判断。

### 10.3 通知面板(Shade)窗口

目录`shade/`下的相关文件:

```text
NotificationPanelView.java
NotificationPanelViewController.java        (5363行)
NotificationShadeWindowControllerImpl.java  (1067行)
NotificationShadeWindowState.kt
NotificationShadeWindowView.java
NotificationShadeWindowViewController.java
ShadeController.java
ShadeControllerImpl.java
ShadeControllerEmptyImpl.kt
ShadeSurface.kt
ShadeSurfaceImpl.kt
ShadeViewController.kt
ShadeViewControllerEmptyImpl.kt
ShadeViewProviderModule.kt
ShadeWindowLayoutParams.kt
ShadeWindowLogger.kt
ShadeModule.kt
StartShadeModule.kt
BaseShadeControllerImpl.kt
QuickSettingsControllerImpl.java
ShadeHeaderController.kt
ShadeDisplayAware.kt
ShadeDisplayAwareModule.kt
```

**注意`ShadeControllerImpl.java`和`BaseShadeControllerImpl.kt`并存**:前面的`extends`后面:

```java
public final class ShadeControllerImpl extends BaseShadeControllerImpl {
    private final NotificationShadeWindowController mNotificationShadeWindowController;
```

这是Kotlin/Java混合迁移期的常见形态。

**窗口参数集中到了`ShadeWindowLayoutParams.kt`**(较新引入):

```kotlin
object ShadeWindowLayoutParams {
    /**
     * Creates [LayoutParams] for the shade window.
     * This is extracted to a single place as those layout params will be used by several places:
     * - When sysui starts, and the shade is added the first time
     * - When the shade moves to a different window (e.g. while an external display is connected)
     */
    fun create(context: Context): LayoutParams {
        return LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT,
                LayoutParams.TYPE_NOTIFICATION_SHADE,
                LayoutParams.FLAG_NOT_FOCUSABLE or
                    LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING or
                    LayoutParams.FLAG_SPLIT_TOUCH or
                    LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
                    LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                PixelFormat.TRANSLUCENT,
            )
            .apply {
                token = Binder()
                gravity = Gravity.TOP
                fitInsetsTypes = 0
                title = "NotificationShade"
                packageName = context.packageName
                layoutInDisplayCutoutMode = LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS
                privateFlags = privateFlags or LayoutParams.PRIVATE_FLAG_OPTIMIZE_MEASURE
                if (SceneContainerFlag.isEnabled) {
                    softInputMode = LayoutParams.SOFT_INPUT_ADJUST_NOTHING
                }
            }
    }
}
```

注释说明了为什么要抽出来:窗口参数**在多处用**——第一次加窗时、以及**外接显示器导致shade搬到另一个窗口时**。

和状态栏窗口的差异:

| 字段 | 状态栏 | 通知面板 |
|---|---|---|
| 尺寸 | `MATCH_PARENT × height` | `MATCH_PARENT × MATCH_PARENT` |
| 类型 | `TYPE_STATUS_BAR` | `TYPE_NOTIFICATION_SHADE` |
| flags | 3个 | 5个,多了`TOUCHABLE_WHEN_WAKING`、`WATCH_OUTSIDE_TOUCH` |
| privateFlags | `COLOR_SPACE_AGNOSTIC` | `OPTIMIZE_MEASURE` |
| title | `"StatusBar"` | `"NotificationShade"` |
| `providedInsets` | 有 | **无** |

**通知面板不提供inset**,因为它是全屏的、可收起的遮罩层,不占任何"永久区域"。

`TOUCHABLE_WHEN_WAKING`让用户在屏幕刚亮起但还没完全唤醒时就能下拉;`WATCH_OUTSIDE_TOUCH`让面板能感知面板之外区域的触摸(用于点外部收起)。

**谁调用`addView`:**

```java
@Override
public void attach() {
    // ...
    // mLP is assigned here (instead of the constructor) as its null value is also used to check
    // if the shade window has been attached.
    mLp = mShadeWindowLayoutParams;
    mWindowManager.addView(mWindowRootView, mLp);

    // We use BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE here, however, there is special logic in
    // window manager which disables the transient show behavior.
    // TODO: Clean this up once that behavior moves into the Shell.
    if (mWindowRootView.getWindowInsetsController() != null) {
        mWindowRootView.getWindowInsetsController().setSystemBarsBehavior(
                BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
    }

    mLpChanged.copyFrom(mLp);
    onThemeChanged();

    // Make the state consistent with KeyguardViewMediator#setupLocked during initialization.
    if (mKeyguardViewMediator.isShowingAndNotOccluded()) {
        setKeyguardShowing(true);
    }
}
```

三个要点:

1. **`mLp`在构造函数里不赋值**,因为"null"被用来判断窗口是否已加。构造函数只保存`mShadeWindowLayoutParams`。
2. **`BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE`**——`TODO`注释坦承这个行为在WindowManager里有特殊处理会把它关掉,等搬进Shell才能清理。**这正是Shell与SystemUI边界还没划干净的一处**。
3. 加窗后立刻对齐keyguard状态,避免启动瞬间状态不一致。

根视图由`fetchWindowRootView()`拿到:

```java
@Override
public void fetchWindowRootView() {
    WindowRootViewComponent component = mWindowRootViewComponentFactory.create();
    mWindowRootView = component.getWindowRootView();
    ...
}
```

调用它的是`CentralSurfacesImpl.setUpShade()`:

```java
mNotificationShadeWindowController.fetchWindowRootView();
getNotificationShadeWindowViewController().setupExpandedStatusBar();
getNotificationShadeWindowViewController().setupCommunalHubLayout();
```

**视图从哪来?** `ShadeViewProviderModule.kt`:

```kotlin
@Provides @SysUISingleton
fun providesWindowRootView(...): WindowRootView {
    return if (SceneContainerFlag.isEnabled) {
        val sceneWindowRootView =
            layoutInflater.inflate(R.layout.scene_window_root, null) as SceneWindowRootView
        sceneWindowRootView.init(...)
        sceneWindowRootView
    } else {
        layoutInflater.inflate(R.layout.super_notification_shade, null)
    } as WindowRootView? ?: throw IllegalStateException(...)
}

@Provides @SysUISingleton
fun providesNotificationShadeWindowView(root: WindowRootView): NotificationShadeWindowView {
    if (SceneContainerFlag.isEnabled) { return root.requireViewById(R.id.legacy_window_root) }
    return root as NotificationShadeWindowView? ?: throw ...
}
```

**这里有个重要的架构演进**:传统布局的根是`R.layout.super_notification_shade`。当`SceneContainerFlag`打开时,根变成`R.layout.scene_window_root`里的`SceneWindowRootView`,而传统的`NotificationShadeWindowView`**退化成它的一个子视图**(`R.id.legacy_window_root`)。

也就是说,shade正在从"一个大View树"迁移到"Scene容器+独立场景"。因此`ShadeViewController.kt`里大量方法被标注:

```kotlin
@Deprecated("Does nothing when scene container is enabled.")
```

**动态改窗口属性**走`applyWindowLayoutParams()`:比较`mLp`和`mLpChanged`,有差异才`mWindowManager.updateViewLayout(...)`;批量场景用`batchApplyWindowLayoutParams(Runnable)`合并多次修改。

所有状态开关都是改`mLpChanged`再apply:

```java
setPanelVisible(boolean)
setKeyguardShowing(boolean)
setBouncerShowing(boolean)
setHeadsUpShowing(boolean)
setDozing(boolean)
setDreaming(boolean)
setForcePluginOpen(boolean, Object)
setNotTouchable(boolean)
setBackgroundBlurRadius(int)
```

它们的接口定义在`statusbar/NotificationShadeWindowController.java`(221行),**全部是default空实现**,方便测试替身。

### 10.4 导航栏

目录`navigationbar/`:

```text
NavBarHelper.java
NavigationBarComponent.java
NavigationBarController.java
NavigationBarControllerEmptyImpl.kt
NavigationBarControllerImpl.java
NavigationBarControllerModule.kt
NavigationBarModule.java
NavigationModeController.java
NoopNavigationBarControllerModule.kt
ScreenPinningNotify.java
TaskbarDelegate.java
gestural/
views/
```

**一个常见错误**:旧资料里导航栏的窗口根视图叫`NavBarWindowView`,文件在`navigationbar/views/window/`。**这两个现在都不存在**。现在的窗口根视图是`NavigationBarFrame`(`navigationbar/views/NavigationBarFrame.java`,60行)。

加窗在`NavigationBar.onInit()`:

```java
public void onInit() {
    ...
    mView.setVisibility(
            mStatusBarKeyguardViewManager.isNavBarVisible() ? View.VISIBLE : View.INVISIBLE);
    ...
    mViewCaptureAwareWindowManager.addView(mFrame,
            getBarLayoutParams(mContext.getResources().getConfiguration().windowConfiguration
                    .getRotation()));
}
```

窗口参数:

```java
WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
        width,
        height,
        WindowManager.LayoutParams.TYPE_NAVIGATION_BAR,
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                | WindowManager.LayoutParams.FLAG_SLIPPERY,
        PixelFormat.TRANSLUCENT);
lp.gravity = gravity;
lp.providedInsets = getInsetsFrameProvider(insetsHeight, userContext);

lp.token = new Binder();
lp.accessibilityTitle = userContext.getString(R.string.nav_bar);
lp.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COLOR_SPACE_AGNOSTIC
        | WindowManager.LayoutParams.PRIVATE_FLAG_LAYOUT_SIZE_EXTENDED_BY_CUTOUT;
lp.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
lp.windowAnimations = 0;
lp.setTitle("NavigationBar" + userContext.getDisplayId());
lp.setFitInsetsTypes(0 /* types */);
lp.setTrustedOverlay();
return lp;
```

和状态栏的差异:

| 字段 | 状态栏 | 导航栏 |
|---|---|---|
| flags | `NOT_FOCUSABLE\|SPLIT_TOUCH\|DRAWS_SYSTEM_BAR_BACKGROUNDS` | 另有`NOT_TOUCH_MODAL`、`WATCH_OUTSIDE_TOUCH`、**`SLIPPERY`** |
| `windowAnimations` | 默认 | **0** |
| title | `"StatusBar"` | `"NavigationBar" + displayId` |
| 其它 | — | `setTrustedOverlay()` |

**`FLAG_SLIPPERY`是导航栏特有的**:它让触摸事件"滑过"这个窗口传给下面的窗口,是手势导航能工作的基础。

**`title`带`displayId`**是因为导航栏**每个display一个**,不像状态栏那样是全局单例。

**`windowAnimations = 0`**是因为导航栏的显隐由SystemUI自己控制,不要系统默认动画。

**尺寸和重力随旋转变化**:竖屏时`height = navigation_bar_frame_height`、`gravity = BOTTOM`;`ROTATION_90`时`gravity = RIGHT`、宽为`navigation_bar_width`;`ROTATION_270`时为`Gravity.LEFT`。

**导航栏还额外加了一个透明窗口用于无缝旋转**:

```java
mOrientationHandle 用 TYPE_NAVIGATION_BAR_PANEL 加窗
    flags: FLAG_SLIPPERY | FLAG_LAYOUT_IN_SCREEN | FLAG_NOT_TOUCHABLE
```

旋转时这个窗口跟着转,让用户看到连续的旋转动画而不是闪一下。

**insets声明**由`getInsetsFrameProvider`给出,声明了`navigationBars`、`tappableElement`、`mandatorySystemGestures`、`systemGestures`四类,并带`FLAG_SUPPRESS_SCRIM`(是否要遮罩由`config_navBarNeedsScrim`决定)。

**三键/手势模式切换**由`NavigationModeController`读资源:

```java
private int getCurrentInteractionMode(Context context) {
    int mode = context.getResources().getInteger(
            com.android.internal.R.integer.config_navBarInteractionMode);
    ...
    return mode;
}
```

**布局字符串**由`NavigationBarInflaterView`决定:

```java
protected String getDefaultLayout() {
    final int defaultResource = QuickStepContract.isGesturalMode(mNavBarMode)
            ? R.string.config_navBarLayoutHandle
            : mOverviewProxyService.shouldShowSwipeUpUI()
                    ? R.string.config_navBarLayoutQuickstep
                    : R.string.config_navBarLayout;
    return getContext().getString(defaultResource);
}
```

**这三个字符串定义在SystemUI自己的资源里**(`packages/SystemUI/res/values/config.xml`),不是framework的:

```xml
<string name="config_navBarLayout" translatable="false">left[.5W],back[1WC];home;recent[1WC],right[.5W]</string>
<string name="config_navBarLayoutQuickstep" translatable="false">back[1.7WC];home;contextual[1.7WC]</string>
<string name="config_navBarLayoutHandle" translatable="false">back[70AC];home_handle;ime_switcher[70AC]</string>
```

**这是一套自带的布局DSL**。语法:

- `;`分隔三段(left/center/right),对应`GRAVITY_SEPARATOR`。
- `,`分隔同一段里的按钮,对应`BUTTON_SEPARATOR`。
- `按钮名[尺寸]`定义按钮和它的宽度。
- 尺寸后缀:`W`=weight(权重),`WC`=weight with... ,`A`=absolute,`AC`=absolute with...

`NavigationBarInflaterView.onFinishInflate()`先`inflateChildren()`(inflate水平组`R.layout.navigation_layout`和垂直组`R.layout.navigation_layout_vertical`),再`inflateLayout(getDefaultLayout())`,把解析结果填到`ends_group`/`center_group`。

**切换三键/手势的RRO overlay**在`frameworks/base/packages/overlays/`:

```text
NavigationBarMode3ButtonOverlay/
NavigationBarMode2ButtonOverlay/
NavigationBarModeGesturalOverlay/
TransparentNavigationBarOverlay/
NoCutoutOverlay/
DisplayCutoutEmulation*Overlay/   (7个)
```

`NavigationBarMode3ButtonOverlay/res/values/config.xml`:

```xml
<integer name="config_navBarInteractionMode">0</integer>
```

`NavigationBarModeGesturalOverlay/res/values/config.xml`(节选):

```xml
<integer name="config_navBarInteractionMode">2</integer>
<bool name="config_navBarCanMove">false</bool>
<bool name="config_navBarTapThrough">true</bool>
<bool name="config_imeDrawsImeNavBar">true</bool>
<dimen name="config_backGestureInset">30dp</dimen>
<bool name="config_navBarNeedsScrim">false</bool>
<integer name="config_navBarOpacityMode">2</integer>
...
```

**framework的`core/res/res/values/config.xml`只提供默认值**:

```xml
config_navBarInteractionMode=0
config_navBarCanMove=true
config_navBarTapThrough=false
config_navBarNeedsScrim=true
config_navBarOpacityMode=0
```

由overlay覆盖。所以"导航栏是手势还是三键"这件事,在运行时表现为一个资源整数值。`config_navBarInteractionMode`:0=三键,1=两键,2=手势。

### 10.5 锁屏

**锁屏UI在SystemUI进程,不在system_server。** 这一点经常被搞混。

**SystemUI侧**(`packages/SystemUI/src/com/android/systemui/keyguard/`):

```text
KeyguardViewMediator.java            (4133行)
KeyguardService.java
KeyguardLifecyclesDispatcher.java
KeyguardUnlockAnimationController.kt
WindowManagerLockscreenVisibilityManager.kt
WindowManagerOcclusionManager.kt
CustomizationProvider.kt
ScreenLifecycle.java
WakefulnessLifecycle.java
...
```

```java
public class KeyguardViewMediator implements CoreStartable,
        StatusBarStateController.StateListener {
```

它是个`CoreStartable`,消息常量`SHOW=1, HIDE=2, RESET=3, ... SET_OCCLUDED=9`。它不自己画UI,而是驱动`KeyguardViewController`的实现:

```java
mKeyguardViewControllerLazy.get().setOccluded(isOccluded, animate)
```

`KeyguardViewController`的实现就是`StatusBarKeyguardViewManager`。

**`StatusBarKeyguardViewManager`**(1884行)的类声明长得吓人:

```java
@ExperimentalCoroutinesApi @SysUISingleton
public class StatusBarKeyguardViewManager implements RemoteInputController.Callback,
        StatusBarStateController.StateListener, ConfigurationController.ConfigurationListener,
        ShadeExpansionListener, NavigationModeController.ModeChangedListener,
        KeyguardViewController, FoldAodAnimationController.FoldAodAnimationStatus {
```

它实现了7个接口,说明锁屏和"shade展开""显示配置""折叠屏""导航栏模式"全都耦合在一起。

它决定导航栏在锁屏下是否可见:

```java
public boolean isNavBarVisible() {
    boolean isWakeAndUnlockPulsing = mBiometricUnlockController != null
            && mBiometricUnlockController.getMode() == MODE_WAKE_AND_UNLOCK_PULSING;
    boolean keyguardVisible = mKeyguardStateController.isVisible();
    boolean hideWhileDozing = mDozing && !isWakeAndUnlockPulsing;
    boolean keyguardWithGestureNav = (keyguardVisible && !mDozing && !mScreenOffAnimationPlaying
            || mPulsing && !mIsDocked)
            && mGesturalNav;
    return (!keyguardVisible && !hideWhileDozing && !mScreenOffAnimationPlaying
            || primaryBouncerIsShowing()
            || mRemoteInputActive
            || keyguardWithGestureNav
            || mGlobalActionsVisible);
}
```

这段逻辑解释了几个现象:

- 锁屏时手势导航要显示导航栏(否则没法上滑解锁),但三键导航不显示。
- 睡眠显示(AOD)时不显示。
- 有远程输入(比如通知上直接回复)时显示。
- 关机菜单弹出来时要显示。

**`KeyguardStateController`在`statusbar/policy/`下,不在`keyguard/`下**:

```text
packages/SystemUI/src/com/android/systemui/statusbar/policy/KeyguardStateController.java      (287行,接口)
packages/SystemUI/src/com/android/systemui/statusbar/policy/KeyguardStateControllerImpl.java (507行)
```

它对外暴露`isVisible()` / `canDismissLockScreen()` / `isOccluded()` / `onUnlockedChanged()`,被`StatusBarKeyguardViewManager`、`NotificationShadeWindowControllerImpl`等消费。

**system_server侧**(`services/core/java/com/android/server/policy/keyguard/`)只有三个文件:

```text
KeyguardServiceDelegate.java
KeyguardServiceWrapper.java
KeyguardStateMonitor.java
```

```java
protected KeyguardServiceWrapper mKeyguardService;
public void bindService(Context context) {
    ...
        mKeyguardService = new KeyguardServiceWrapper(mContext,
                IKeyguardService.Stub.asInterface(service), mCallback);
        ...
            mKeyguardService.onSystemReady();
            ...
            mKeyguardService.setOccluded(mKeyguardState.occluded, false /* animate */);
            mKeyguardService.setKeyguardEnabled(mKeyguardState.enabled);
            mKeyguardService.onDreamingStarted();
```

**分工总结:**

| 事项 | 在哪 |
|---|---|
| 决定**何时**锁屏(`doKeyguardLocked`触发) | system_server(PowerManager/WindowManagerService) |
| 通过`IKeyguardService`通知SystemUI显示/隐藏/遮挡/启用 | system_server(`KeyguardServiceDelegate`) |
| 维护"是否可信"(trusted)、输入是否受限 | system_server(`KeyguardStateMonitor`实现`IKeyguardStateCallback`) |
| **实际绘制**锁屏UI(bouncer、钥匙孔、时钟、解锁动画) | SystemUI(`KeyguardViewMediator` + `StatusBarKeyguardViewManager`) |
| 回调`ViewMediatorCallback`报告解锁完成、保持wake lock | SystemUI |

`KeyguardServiceDelegate`用`bindServiceAsUser`绑定SystemUI的`KeyguardService`。

### 10.6 Scrim:前中后三层遮罩

`ScrimController`(`statusbar/phone/ScrimController.java`,1670行):

```java
public class ScrimController implements ViewTreeObserver.OnPreDrawListener, Dumpable {
    private ScrimState mState = ScrimState.UNINITIALIZED;
    private ScrimView mScrimInFront;
    private ScrimView mScrimBehind;
```

它管三层遮罩:`mScrimInFront`(前面)、`mScrimBehind`(后面)、通知层(`mNotificationsTint/mNotificationsAlpha`)。

**初始化时把两个ScrimView注入每个状态对象:**

```java
final ScrimState[] states = ScrimState.values();
for (int i = 0; i < states.length; i++) {
    states[i].init(mScrimInFront, mScrimBehind, mDozeParameters, mDockManager);
    states[i].setScrimBehindAlphaKeyguard(mScrimBehindAlphaKeyguard);
}
```

也就是**状态对象持有视图引用,视图不由Controller直接操作**——这是State模式。切换状态是`transitionTo(state, ...)` → `applyState()`:

```java
private void applyState() {
    mInFrontTint = mState.getFrontTint();
    mBehindTint = mState.getBehindTint();
    mNotificationsTint = mState.getNotifTint();

    mInFrontAlpha = mState.getFrontAlpha();
    mBehindAlpha = mState.getBehindAlpha();
    mNotificationsAlpha = mState.getNotifAlpha();

    assertAlphasValid();

    if (!mExpansionAffectsAlpha) { return; }

    if (mState == ScrimState.UNLOCKED || mState == ScrimState.DREAMING
            || mState == ScrimState.GLANCEABLE_HUB_OVER_DREAM) {
        ...
```

`ScrimState`(`statusbar/phone/ScrimState.java`,484行)是个枚举,含`UNINITIALIZED, OFF, KEYGUARD, ..., BOUNCER, SHADE_LOCKED, DREAMING, GLANCEABLE_HUB, GLANCEABLE_HUB_OVER_DREAM, PULSING, AOD, UNLOCKED`等。

**重点:`ScrimState`和`StatusBarState`是两套独立的状态机。** 名字里有重叠(`DREAMING`、`SHADE_LOCKED`),但互不相干。`ScrimState`管的是"哪层遮罩、什么颜色、多透明",`StatusBarState`管的是"状态栏处于什么模式"。

**`ScrimView`的路径也不是想当然的那个**:不在`statusbar/ScrimView.java`,而在:

```text
packages/SystemUI/src/com/android/systemui/scrim/ScrimView.java    (382行)
```

`ScrimController`通过`scrimBehind.setViewAlpha()`、`setScrimName()`、`dispatchBackScrimState(alpha)`驱动它。

### 10.7 LightBarController:状态栏图标深浅色

`statusbar/phone/LightBarController.kt`(接口,72行):

```kotlin
/** Controls how light status bar flag applies to the icons. */
interface LightBarController : CoreStartable {
    fun stop()
    fun setNavigationBar(navigationBar: LightBarTransitionsController)
    fun onNavigationBarAppearanceChanged(...)
    fun customizeStatusBarAppearance(appearance: AppearanceRegion)
    fun setScrimState(scrimState: ScrimState, scrimBehindAlpha: Float, scrimInFrontColor: ColorExtractor.GradientColors)
    fun interface Factory { fun create(context: Context): LightBarController }
}
```

实现在`statusbar/phone/LightBarControllerImpl.java`(542行):

```java
public class LightBarControllerImpl implements
        BatteryController.BatteryStateChangeCallback, LightBarController {
```

**注意它已被Kotlin化但实现仍是Java**,且**没有被删除**,仍然在SystemUI启动图里。

它的数据来源是`StatusBarModePerDisplayRepository`(订阅`StatusBarAppearance`变化)。核心逻辑:

```java
private void updateStatus(AppearanceRegion[] appearanceRegions) {
    final int numStacks = appearanceRegions.length;
    final ArrayList<Rect> lightBarBounds = new ArrayList<>();
    for (int i = 0; i < numStacks; i++) {
        final AppearanceRegion ar = appearanceRegions[i];
        if (isLight(ar.getAppearance(), mStatusBarMode, APPEARANCE_LIGHT_STATUS_BARS)) {
            lightBarBounds.add(ar.getBounds());
        }
    }
    if (lightBarBounds.isEmpty()) {
        mStatusBarIconController.getTransitionsController().setIconsDark(false, animateChange());
    } else if (lightBarBounds.size() == numStacks) {
        mStatusBarIconController.setIconsDarkArea(null);
        mStatusBarIconController.getTransitionsController().setIconsDark(true, animateChange());
    } else {
        // Not the same for every stack, magic!
        mStatusBarIconController.setIconsDarkArea(lightBarBounds);
        mStatusBarIconController.getTransitionsController().setIconsDark(true, animateChange());
    }
}
```

三种情况:

1. **没有任何区域要浅色图标** → 全部深色图标。
2. **所有区域都要浅色图标** → 全部浅色,不做分区。
3. **混合**(源码注释写着"magic!"):`setIconsDarkArea(lightBarBounds)`,只把指定区域内的图标变浅色。

第3种情况出现在**分屏**:上半屏是深色App(需要浅色图标),下半屏是浅色App(需要深色图标),状态栏横跨两半,只能分区处理。这也是为什么`AppearanceRegion[]`是数组而不是单个对象。

`LetterboxAppearanceCalculator.kt`(223行)负责把letterbox区域(黑边)的appearance也并入计算:

```text
statusbar/phone/LetterboxAppearanceCalculator.kt
statusbar/phone/LetterboxBackgroundProvider.kt
statusbar/phone/LetterboxModule.kt
```

**关于`SystemBarAttributesListener`**:这个类在当前源码里**不存在**(探测文件返回FAIL)。外观数据现在经`StatusBarAppearance` / `StatusBarModePerDisplayRepository`流入`LightBarControllerImpl`,没有叫这个名字的类。

### 10.8 外观数据从WMS怎么流过来

数据源头是WMS的`IStatusBar.onSystemBarAttributesChanged(...)`。SystemUI侧在`CentralSurfacesImpl.start()`里:

```java
mCommandQueueCallbacks.onSystemBarAttributesChanged(
        mDisplayId, result.mAppearance, result.mAppearanceRegions,
        result.mNavbarColorManagedByIme, result.mBehavior,
        result.mRequestedVisibleTypes, result.mPackageName, result.mLetterboxDetails);
```

WMS侧的计算在`DisplayPolicy.updateSystemBarAttributes()`(`services/core/java/com/android/server/wm/DisplayPolicy.java`):

```java
void updateSystemBarAttributes() {
    WindowState winCandidate = mFocusedWindow != null ? mFocusedWindow
            : mTopFullscreenOpaqueWindowState;
    if (winCandidate == null) { return; }
    ...
    final WindowState win = winCandidate;
    mSystemUiControllingWindow = win;

    final int displayId = getDisplayId();
    final int disableFlags = win.getDisableFlags();
    final int opaqueAppearance = updateSystemBarsLw(win, disableFlags);
    ...
    final int appearance = updateLightNavigationBarLw(win.mAttrs.insetsFlags.appearance,
            navColorWin) | opaqueAppearance;
    final WindowState navBarControlWin = topAppHidesSystemBar(Type.navigationBars())
            ? mTopFullscreenOpaqueWindowState : win;
    final int behavior = navBarControlWin.mAttrs.insetsFlags.behavior;
    final String focusedApp = win.mAttrs.packageName;
    final boolean isFullscreen = !win.isRequestedVisible(Type.statusBars())
            || !win.isRequestedVisible(Type.navigationBars());
    final AppearanceRegion[] statusBarAppearanceRegions =
            new AppearanceRegion[mStatusBarAppearanceRegionList.size()];
    mStatusBarAppearanceRegionList.toArray(statusBarAppearanceRegions);
    if (mLastDisableFlags != disableFlags) {
        mLastDisableFlags = disableFlags;
        ...
        callStatusBarSafely(statusBar -> statusBar.setDisableFlags(displayId, disableFlags, cause));
    }
    final @InsetsType int requestedVisibleTypes = win.getRequestedVisibleTypes();
    ...
    callStatusBarSafely(statusBar -> statusBar.onSystemBarAttributesChanged(displayId,
            appearance, statusBarAppearanceRegions, isNavbarColorManagedByIme, behavior,
            requestedVisibleTypes, focusedApp, letterboxDetails));
}
```

**关键的输入是"焦点窗口"或"最顶层全屏不透明窗口"**(`mFocusedWindow ?: mTopFullscreenOpaqueWindowState`)。也就是说,**系统栏的外观由"当前谁在屏幕上说了算"决定**。

计算出的东西:

| 输出 | 含义 |
|---|---|
| `appearance` | `APPEARANCE_LIGHT_STATUS_BARS` / `LIGHT_NAVIGATION_BARS` / `LOW_PROFILE_BARS` 的位或 |
| `statusBarAppearanceRegions` | 分区域的外观(分屏/letterbox用) |
| `behavior` | 系统栏行为(`BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE`等) |
| `requestedVisibleTypes` | 哪些系统栏类型的可见性被App请求了 |
| `disableFlags` | 单独通过`setDisableFlags`下发 |

**`WindowState`没有`getInsetsHint()`方法**。当前可用的insets查询接口是:

```java
@Override
public boolean isRequestedVisible(@InsetsType int types) {
    return (mRequestedVisibleTypes & types) != 0;
}

@Override
public @InsetsType int getRequestedVisibleTypes() {
    return mRequestedVisibleTypes;
}
```

加上`int getDisableFlags()`和`getControllableInsetProvider()`。

**临时显示系统栏**(手势上滑唤出):

```java
void requestTransientBars(WindowState swipeTarget, boolean isGestureOnSystemBar) {
    ...
    final InsetsSourceProvider provider = swipeTarget.getControllableInsetProvider();
    final InsetsControlTarget controlTarget = provider != null
            ? provider.getControlTarget() : null;
    if (controlTarget == null || controlTarget == getNotificationShade()) {
        // No transient mode on lockscreen (in notification shade window).
        return;
    }
    ...
        mDisplayContent.getInsetsPolicy().showTransient(SHOW_TYPES_FOR_SWIPE, isGestureOnSystemBar);
        controlTarget.showInsets(restorePositionTypes, false /* fromIme */, null /* statsToken */);
```

注意那句注释:"**锁屏上没有临时显示模式**",因为锁屏本身就在通知面板窗口里,不让App上滑唤出系统栏。这是全屏App在锁屏上行为不同的原因。

### 10.9 状态机:StatusBarState

`statusbar/StatusBarState.java`(58行):

```java
public class StatusBarState {
    /**
     * The status bar is in the "normal", unlocked mode or the device is still locked but we're
     * accessing camera from power button double-tap shortcut.
     */
    public static final int SHADE = 0;

    /**
     * Status bar is currently the Keyguard. In single column mode, when you swipe from the top of
     * the keyguard to expand QS immediately, it's still KEYGUARD state.
     */
    public static final int KEYGUARD = 1;

    /**
     * Status bar is in the special mode, where it was transitioned from lockscreen to shade.
     * ...
     */
    public static final int SHADE_LOCKED = 2;
```

**只有三个常量,没有`DREAMING`。** 这是一个高频误解。DREAMING是**独立的boolean**,在`StatusBarStateController`上:

`statusbar/SysuiStatusBarStateController.java`(169行,接口,`extends StatusBarStateController, CoreStartable`):

```java
@IntDef({RANK_STATUS_BAR, RANK_STATUS_BAR_WINDOW_CONTROLLER, RANK_STACK_SCROLLER, RANK_SHELF})
@interface SbStateListenerRank {}
int RANK_STATUS_BAR = 0;
int RANK_STATUS_BAR_WINDOW_CONTROLLER = 1;
int RANK_STACK_SCROLLER = 2;
int RANK_SHELF = 3;
...
boolean setState(int state, boolean force);
void setUpcomingState(int state);
int getCurrentOrUpcomingState();
boolean setIsDozing(boolean isDozing);
boolean setIsDreaming(boolean isDreaming);   // <- dreaming 在这里
void setAndInstrumentDozeAmount(View view, float dozeAmount, boolean animated);
```

**`RANK_*`是监听器回调顺序**。`RANK_STATUS_BAR_WINDOW_CONTROLLER`排第二,`NotificationShadeWindowControllerImpl`构造时用它注册:

```java
addCallback(mStateListener, RANK_STATUS_BAR_WINDOW_CONTROLLER)
```

**窗口控制器必须比其它listener先收到回调**,因为它要先把窗口属性(比如`setDozing`)改好,后面的listener才能基于正确的窗口状态做判断。

`StatusBarStateControllerImpl.java`(823行):

```java
private static final int MAX_STATE = StatusBarState.SHADE_LOCKED;
private static final int MIN_STATE = StatusBarState.SHADE;
...
public boolean setState(int state, boolean force) {
    ...
    // Record the to-be mState and mLastState
    recordHistoricalState(state /* newState */, mState /* lastState */, false);
    ...
    if (mState == StatusBarState.SHADE && state == StatusBarState.SHADE_LOCKED) {
        Log.e(TAG, "Invalid state transition: SHADE -> SHADE_LOCKED", new Throwable());
    }
```

注意这里**记了错误但没抛异常**(`Log.e`而已),因为实际运行中确实可能出现这种跳跃(比如锁屏解锁和面板展开几乎同时发生)。

dreaming是独立维护的:

```java
public boolean setIsDreaming(boolean isDreaming) {
    ...
    mIsDreaming = isDreaming;
    ...
        rl.mListener.onDreamingChanged(isDreaming);
```

`StatusBarStateControllerImpl`还会**根据keyguard和scene状态推导**`newState`。这意味着调用方不需要手动set,状态机会自己算。

**三个状态的实际含义:**

| 状态 | 场景 |
|---|---|
| `SHADE` | 正常解锁状态,或者锁屏下双击电源起相机 |
| `KEYGUARD` | 锁屏本体。单栏模式下从锁屏顶部下拉直接展QS,仍然是KEYGUARD |
| `SHADE_LOCKED` | 从锁屏滑到了通知面板,但还没解锁 |

### 10.10 加窗顺序

`CentralSurfacesImpl.createAndAddWindows`(`CentralSurfacesImpl.java:1881`):

```java
private void createAndAddWindows(@Nullable RegisterStatusBarResult result) {
    makeStatusBarView(result);
    mNotificationShadeWindowController.attach();
    // When the StatusBarConnectedDisplays flag is enabled, this logic will be done in
    // StatusBarOrchestrator
    if (!StatusBarConnectedDisplays.isEnabled()) {
        mStatusBarWindowControllerStore.getDefaultDisplay().attach();
    }
}
```

顺序是:

1. `makeStatusBarView(result)`——inflate状态栏的View树。
2. `mNotificationShadeWindowController.attach()`——**先加通知面板窗口**。
3. `mStatusBarWindowControllerStore.getDefaultDisplay().attach()`——**后加状态栏窗口**。

**为什么通知面板先加?** 因为锁屏在通知面板窗口里,而状态栏需要知道keyguard状态来决定初始可见性(`StatusBarWindowControllerImpl.attach()`末尾的`apply(mCurrentState)`)。

注意`StatusBarConnectedDisplays`这个flag——多屏互联显示场景下这块逻辑搬到`StatusBarOrchestrator`里了。

**导航栏不在这条链路里**,它是独立启动的:

```text
NavigationBarController.createNavigationBars()
  → NavigationBarControllerImpl.createNavigationBar()
  → NavigationBarComponent → NavigationBar.init() → onInit()
```

### 10.11 三大窗口的创建链路总图

```text
SystemUIApplication / CoreStartup (Dagger 启动图)
  └─ CentralSurfacesImpl.start()
       ├─ makeStatusBarView(result)
       └─ createAndAddWindows(result)
            ├─ mNotificationShadeWindowController.attach()
            │     ├─ fetchWindowRootView()  ← setUpShade() 先调用
            │     │     └─ WindowRootViewComponent
            │     │           → ShadeViewProviderModule.providesWindowRootView()
            │     │               ├─ flag关: inflate(R.layout.super_notification_shade)
            │     │               └─ flag开: inflate(R.layout.scene_window_root)
            │     ├─ mLp = mShadeWindowLayoutParams (ShadeWindowLayoutParams.create(ctx))
            │     └─ mWindowManager.addView(mWindowRootView, mLp)
            │            type = TYPE_NOTIFICATION_SHADE (2040)
            │            flags = NOT_FOCUSABLE|TOUCHABLE_WHEN_WAKING|SPLIT_TOUCH
            │                    |WATCH_OUTSIDE_TOUCH|DRAWS_SYSTEM_BAR_BACKGROUNDS
            │            format = TRANSLUCENT; token = Binder(); gravity = TOP
            │            fitInsetsTypes = 0; title = "NotificationShade"
            │            layoutInDisplayCutoutMode = ALWAYS
            │            (不提供 inset)
            └─ mStatusBarWindowControllerStore.getDefaultDisplay().attach()
                  ├─ mLp = getBarLayoutParams(rotation)
                  └─ mWindowManager.addView(mStatusBarWindowView, mLp)
                         type = TYPE_STATUS_BAR (2000)
                         flags = NOT_FOCUSABLE|SPLIT_TOUCH|DRAWS_SYSTEM_BAR_BACKGROUNDS
                         format = TRANSLUCENT; token = Binder(); gravity = TOP
                         fitInsetsTypes = 0; title = "StatusBar"
                         layoutInDisplayCutoutMode = ALWAYS
                         providedInsets = [statusBars(height),
                                           tappableElement(height),
                                           mandatorySystemGestures(minCutoutSafe)]

导航栏(独立路径,非CentralSurfaces):
NavigationBarController.createNavigationBars()
  → NavigationBarControllerImpl.createNavigationBar()
  → NavigationBarComponent → NavigationBar.init() → onInit()
        ├─ mView.setVisibility(isNavBarVisible() ? VISIBLE : INVISIBLE)
        └─ mViewCaptureAwareWindowManager.addView(mFrame, getBarLayoutParams(rotation))
               type = TYPE_NAVIGATION_BAR (2019)
               flags = NOT_FOCUSABLE|NOT_TOUCH_MODAL|WATCH_OUTSIDE_TOUCH|SPLIT_TOUCH|SLIPPERY
               format = TRANSLUCENT; token = Binder()
               gravity = BOTTOM/RIGHT/LEFT; fitInsetsTypes = 0
               title = "NavigationBar<displayId>"; setTrustedOverlay()
               providedInsets = [navigationBars, tappableElement,
                                 mandatorySystemGestures, systemGestures]
        (旋转时另加 TYPE_NAVIGATION_BAR_PANEL 的 mOrientationHandle 透明窗口)

WMS 侧:
WindowManagerService.addWindow
  → DisplayPolicy.addWindowLw() 按 attrs.type 记账
       (mStatusBar / mNotificationShade / mNavigationBar)
       并注册 attrs.providedInsets
  → 焦点/全屏窗口变化时 DisplayPolicy.updateSystemBarAttributes()
  → StatusBarManagerInternal.onSystemBarAttributesChanged / setDisableFlags
  → IStatusBar (SystemUI 的 CommandQueueCallbacks)
  → LightBarControllerImpl / CentralSurfaces 更新图标配色与禁用态
```

**这张图是整个教程的核心结论**:SystemUI加三个窗口,WMS按类型认账并读取insets声明;之后WMS根据焦点窗口算外观,再推回SystemUI。**两者不是主从关系,而是一个持续的闭环**:SystemUI提供窗口和inset,WMSCore决定外观和可见性,外观又推回SystemUI改变图标颜色。

## 11 两者协作面

前面各章分别讲了WMShell和SystemUI。这一章把它们之间的所有接口汇总起来,这是读源码时最该先搞清的一张表。

### 11.1 三层协作

WMShell和SystemUI在同一个进程里,靠三种方式交换信息:

| 层次 | 机制 | 例子 | 方向 |
|---|---|---|---|
| 同进程Dagger注入 | `SysUIComponent.Builder`的13个`@BindsInstance`setter | `ShellInterface`、`Pip`、`SplitScreen` | WMShell → SystemUI |
| 跨进程AIDL | `ShellController.addExternalInterface(...)` + `SystemUIProxy` | `ISplitScreen`、`IPip`、`IBubbles` | WMShell → Launcher/其它App |
| Bundle交接 | `IOverviewProxy.onInitialize(Bundle)` | 把AIDL和Binder对象一次性交给Launcher | SystemUI → Launcher |

第一层是"进程内但跨模块",第二层是"出进程",第三层是"把第二层的东西打包送给Launcher"。

### 11.2 Bundle交接:createExternalInterfaces

第4章提过`ShellController`里有`createExternalInterfaces`。它的调用点在`OverviewProxyService`。

**Launcher侧的接收端**是`TouchInteractionService.TISBinder.onInitialize(Bundle)`。**发送端**在:

```java
// packages/SystemUI/src/com/android/systemui/recents/OverviewProxyService.java
Bundle params = new Bundle();
addInterface(mSysUiProxy, params);
addInterface(mSysuiUnlockAnimationController, params);
addInterface(mUnfoldTransitionProgressForwarder.orElse(null), params);
// Add all the interfaces exposed by the shell
mShellInterface.createExternalInterfaces(params);
mOverviewProxy.onInitialize(params);
```

逐行说明:

1. `mSysUiProxy`(`ISystemUiProxy`):SystemUI自己的代理接口,给Launcher用。
2. `mSysuiUnlockAnimationController`(`ISysuiUnlockAnimationController`):解锁动画的接口。
3. `mUnfoldTransitionProgressForwarder`(`IUnfoldTransitionProgressForwarder`):折叠屏展开进度的转发。
4. `mShellInterface.createExternalInterfaces(params)`:**把WMShell注册的所有外部接口一次性塞进同一个Bundle**。
5. `mOverviewProxy.onInitialize(params)`:跨进程发给Launcher。

`addInterface`是个小工具方法,作用是把Binder以类名为key放进Bundle。

**`createExternalInterfaces`的实现**(`libs/WindowManager/Shell/src/com/android/wm/shell/sysui/ShellController.java`):

```java
/** Adds all the interfaces to the bundle that are registered as external interfaces. */
public void createExternalInterfaces(Bundle params) {
    ...
    for (int i = mExternalInterfaces.size() - 1; i >= 0; --i) {
        final ExternalInterfaceBinder binder = mExternalInterfaces.valueAt(i).get();
        if (binder == null) continue;
        params.putBinder(binder.getInterfaceName(), binder.asBinder());
    }
}
```

也就是**遍历所有通过`addExternalInterface`注册的接口,按名字放进Bundle**。

**这套设计的意义**:Launcher启动时不需要逐个`bindService`去拿各个系统服务,只需要一次`onInitialize(Bundle)`就拿到全部。而Shell注册了什么,Launcher就拿到什么——**新增一个Shell接口,Launcher侧不用改握手代码**。

### 11.3 WMShell导出的外部接口清单

在Shell源码里用`addExternalInterface`注册的接口(接口描述符 + 注册位置):

| 接口 | DESCRIPTOR | 注册位置 |
|---|---|---|
| `IShellTransitions` | `IShellTransitions.DESCRIPTOR` | `Transitions.java:onInit` |
| `ISplitScreen` | `ISplitScreen.DESCRIPTOR` | `SplitScreenController.java:284` |
| `IPip` | `IPip.DESCRIPTOR` | `PipController.java:729` |
| `IBubbles` | `IBubbles.DESCRIPTOR` | `BubbleController.java:526` |
| `IOneHanded` | `IOneHanded.DESCRIPTOR` | `OneHandedController.java:300` |
| `IRecentTasks` | `IRecentTasks.DESCRIPTOR` | `RecentTasksController.java:183` |
| `IDragAndDrop` | `IDragAndDrop.DESCRIPTOR` | `DragAndDropController.java:168` |
| `IDesktopMode` | `IDesktopMode.DESCRIPTOR` | `DesktopTasksController.kt:239` |
| `IStartingWindow` | `IStartingWindow.DESCRIPTOR` | `StartingWindowController.java:120` |
| `IBackAnimation` | `IBackAnimation.DESCRIPTOR` | `BackAnimationController` |

**注意`KeyguardTransitions`不在这张表里**——它是同进程接口,不走`addExternalInterface`。`ShellTransitions`两者都有(既有AIDL给Launcher,又有同进程`ShellTransitions.java`给SystemUI)。

### 11.4 SystemUI收到的13个同进程接口

`SystemUIInitializer.init()`里注入的13个(第9.7节列过完整代码):

| Setter | 接口 | 主要用途 |
|---|---|---|
| `setShell(ShellInterface)` | `ShellInterface` | 初始化握手、配置变化、IME变化、焦点过渡 |
| `setPip(Optional<Pip>)` | `Pip` | 展开PiP、监听PiP状态、排除区域 |
| `setSplitScreen(Optional<SplitScreen>)` | `SplitScreen` | 分屏状态与操作 |
| `setOneHanded(Optional<OneHanded>)` | `OneHanded` | 单手模式开关 |
| `setBubbles(Optional<Bubbles>)` | `Bubbles` | 气泡 |
| `setTaskViewFactory(Optional<TaskViewFactory>)` | `TaskViewFactory` | 任务快照视图 |
| `setShellTransitions(ShellTransitions)` | `ShellTransitions` | 注册远端过渡、监听home/focus过渡 |
| `setKeyguardTransitions(KeyguardTransitions)` | `KeyguardTransitions` | 注册锁屏的5个远端过渡 |
| `setStartingSurface(Optional<StartingSurface>)` | `StartingSurface` | 启动闪屏 |
| `setDisplayAreaHelper(Optional<DisplayAreaHelper>)` | `DisplayAreaHelper` | 挂载到root显示区 |
| `setRecentTasks(Optional<RecentTasks>)` | `RecentTasks` | 最近任务数据 |
| `setBackAnimation(Optional<BackAnimation>)` | `BackAnimation` | 返回手势动画 |
| `setDesktopMode(Optional<DesktopMode>)` | `DesktopMode` | 桌面模式 |

**只有`ShellInterface`、`ShellTransitions`、`KeyguardTransitions`三个是必需的**(没有`Optional`包装),其余十个都用`Optional`,因为设备可能不支持对应功能。

### 11.5 具体的耦合点清单

上面是接口层面的。实际代码里WMShell和SystemUI互相调用的具体位置有这么几组:

**WMShell → SystemUI:**

| WMShell侧 | SystemUI侧 | 做什么 |
|---|---|---|
| `ShellController.addConfigurationChangeListener` | 各模块实现`onConfigurationChanged` | 配置变化分发 |
| `ShellController.addKeyguardChangeListener` | 各模块实现`onKeyguardChanged` | 锁屏状态分发 |
| `ShellController.addUserChangeListener` | 各模块实现`onUserChanged` | 用户切换分发 |
| `ShellController.addDisplayImeChangeListener` | 实现`onDisplayImeChanged` | IME高度/可见性分发 |
| `Transitions.mObservers` | `HomeTransitionObserver`、`FocusTransitionObserver` | 过渡生命周期通知 |
| `Pip.addPipExclusionBoundsChangeListener` | `NavigationBar`、`TaskbarDelegate` | PiP边界告诉手势系统 |

**SystemUI → WMShell:**

| SystemUI侧 | WMShell侧 | 做什么 |
|---|---|---|
| `KeyguardViewMediator` | `KeyguardTransitions.register(...)` | 交5个锁屏过渡动画给Shell |
| `KeyguardViewMediator` | `KeyguardTransitions.startKeyguardTransition(...)` | 主动发起锁屏过渡 |
| `NavigationBar` | `Pip.addPipExclusionBoundsChangeListener` | 注册PiP边界监听 |
| `NavigationBar` | `OneHanded`(经`NavBarHelper`) | 长按触发单手模式 |
| `TaskbarDelegate` | `Pip`、`SplitScreen`、`DesktopMode` | 桌面模式任务栏 |
| `OverviewProxyService` | `ShellInterface.createExternalInterfaces` | Bundle交接 |
| SystemUI各Controller | `ShellTransitions.registerRemote(...)` | 注册自己的远端过渡 |

**特别提一下`ShellInterface`里的IME和焦点监听**,这是比较新的协作面:

```java
// libs/WindowManager/Shell/src/com/android/wm/shell/sysui/ShellInterface.java
default void addDisplayImeChangeListener(DisplayImeChangeListener listener,
        Executor executor) {}

default void setFocusTransitionListener(@NonNull FocusTransitionListener listener,
        Executor executor) {}
```

注意两个方法都**带`Executor`参数**。这是第5章讲的协作规则:跨模块回调必须让接收方指定线程。

### 11.6 线程边界:协作面的隐含约束

第5章讲过线程规则,这里结合协作面再总结一遍:**所有跨模块调用都要跨线程边界**。

| 调用方向 | 出发线程 | 目标线程 | 怎么过去 |
|---|---|---|---|
| SystemUI → Shell 同进程接口 | SystemUI主线程 | `wmshell.main` | 接口实现里`mMainExecutor.execute(...)` |
| Shell → SystemUI 回调 | `wmshell.main` | SystemUI主线程 | 调用方传`Executor`,自己post |
| WMShell → Launcher | `wmshell.main` | Launcher进程 | Binder |
| Launcher → WMShell | Launcher线程 | `wmshell.main` | Binder + `getRemoteCallExecutor()` |
| WMCore → Shell | Binder线程 | `wmshell.main` | `TransitionPlayerImpl`等里`mMainExecutor.execute(...)` |
| Shell → WMCore | `wmshell.main` | `system_server` | 同步Binder |

**`getRemoteCallExecutor()`是第四行的答案。** `RemoteCallable<T>`要求实现它,返回的就是Shell主线程的executor。AIDL桩在Binder线程收到调用后,要用它post到正确线程。

这也是为什么`ShellController.addExternalInterface`要传一个`target`参数:

```java
public void addExternalInterface(String name,
        Supplier<ExternalInterfaceBinder> supplier, Object target) {
    ...
}
```

`target`用来在dump时标明"这个接口属于哪个对象",但更重要的是`ExternalInterfaceBinder`的创建必须发生在正确的线程上。

### 11.7 谁拥有什么:一张对照表

最后用一张表收束,回答"某个功能到底在哪个进程里":

| 功能 | 归属 | 关键类 |
|---|---|---|
| 窗口的最终布局与层级 | WMCore | `WindowManagerService`、`RootWindowContainer` |
| 系统栏窗口的**内容与窗口本身** | SystemUI | `StatusBarWindowControllerImpl`、`NotificationShadeWindowControllerImpl` |
| insets的**声明**(谁提供多少) | SystemUI加窗时声明 | `LayoutParams.providedInsets` |
| insets的**仲裁**(最终给App多少) | WMCore | `DisplayPolicy`、`InsetsPolicy` |
| 系统栏外观(浅色/深色图标)的**决定** | WMCore | `DisplayPolicy.updateSystemBarAttributes` |
| 系统栏外观的**呈现** | SystemUI | `LightBarControllerImpl` |
| 过渡动画(**Shell Transition启用时**) | SystemUI(WMShell) | `Transitions`、各`TransitionHandler` |
| 过渡动画(**legacy**) | WMCore + App | `AppTransitionController` |
| 单次过渡有哪些窗口在动 | WMCore | `TransitionController`、`TransitionInfo` |
| 分屏/画中画/气泡的**状态与操作** | SystemUI(WMShell) | `SplitScreenController`、`PipController`、`BubbleController` |
| 分屏/画中画/气泡的**窗口管理** | WMCore | 各WindowingMode的`DisplayArea` |
| 锁屏UI | SystemUI | `KeyguardViewMediator`、`StatusBarKeyguardViewManager` |
| 何时锁屏 | WMCore / system_server | `WindowManagerService`、`PowerManagerService` |
| 通知列表的**数据** | WMCore | `NotificationManagerService` |
| 通知列表的**渲染** | SystemUI | `NotificationStackScrollLayout`、`NotificationPanelViewController` |
| Launcher(桌面、最近任务UI) | Launcher进程 | `Launcher`、`TouchInteractionService` |
| 任务快照、task状态 | 三处都有 | 权威在`ActivityTaskManagerService`,Shell有缓存,SystemUI有视图 |

**这张表里最容易被搞错的三行**:

1. **insets的声明和仲裁分属两个进程**。SystemUI说"我提供36dp的状态栏inset",WMCore根据当前App的请求决定最终给App多少。
2. **系统栏外观的决定和呈现分属两个进程**。WMCore算出`appearance`,SystemUI把图标染成对应颜色。
3. **通知的数据和渲染分属两个进程**。NMS管通知对象,SystemUI管ListView。

## 12 调试

### 12.1 先看dumpsys

SystemUI是一个Service,所以用`dumpsys`:

```shell
adb shell dumpsys activity service com.android.systemui
```

**WMShell的内容通过SystemUI这个Service一起dump**,不能单独dump:

```shell
adb shell dumpsys activity service SystemUIService WMShell
```

注意这里写的是`SystemUIService`(不带包名)。dumpsys的参数匹配是按Service的短类名/长类名做的,两种写法都可能匹配上,取决于框架版本的匹配逻辑。

`SystemUIService.dump`的实现:

```java
@Override
protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
    String[] massagedArgs = args;
    if (args.length == 0) {
        massagedArgs = new String[] {
                DumpHandler.PRIORITY_ARG,
                DumpHandler.PRIORITY_ARG_CRITICAL};
    }
    mDumpHandler.dump(fd, pw, massagedArgs);
}
```

**不带参数时默认只dump CRITICAL优先级的部分。** 原因是SystemUI的完整dump非常大,默认全打出来会拖慢adb。`DumpHandler`的常量:

```kotlin
PRIORITY_ARG = "--dump-priority"
PRIORITY_ARG_CRITICAL = "CRITICAL"
PRIORITY_ARG_NORMAL = "NORMAL"
```

所以要dump全部:

```shell
adb shell dumpsys activity service com.android.systemui --dump-priority NORMAL
```

还支持proto格式:

```shell
adb shell dumpsys activity service com.android.systemui --proto
```

每个`CoreStartable`在启动时被登记:

```java
if (service.isDumpCritical()) {
    dumpManager.registerCriticalDumpable(service);
} else {
    dumpManager.registerNormalDumpable(service);
}
```

`CoreStartable.isDumpCritical()`**默认返回true**。所以新写的模块默认都是CRITICAL,需要显式覆盖成false才进NORMAL。

**其它有用的dumpsys:**

```shell
# WindowManager的容器树(不用winscope就能看窗口层级)
adb shell dumpsys activity containers

# SurfaceFlinger的图层树(搜"Layer Hierarchy")
adb shell dumpsys SurfaceFlinger

# ActivityTaskManager的任务栈
adb shell dumpsys activity activities
```

`dumpsys activity containers`是排查"某个窗口挂在哪"的最快手段——它直接显示窗口容器树的层级,不用抓trace。

### 12.2 ProtoLog

WMShell的日志**基本不用`Log`/`Slog`**,用ProtoLog。

设计文档里说得很直接:

> Don't use Logs or Slogs except for error cases, Protologs are much more flexible, easy to add and easy to use

**为什么用ProtoLog?** 它有两个好处:

1. **编译期预处理**。`protologtool`在构建时把日志调用替换成"先检查是否启用"的形式,禁用时几乎零开销。普通`Log.v`即使不输出也要拼接字符串。
2. **可以打进Perfetto trace**。ProtoLog可以输出成二进制proto,在Winscope里和窗口信息对齐显示。

**代码里长这样:**

```java
ProtoLog.v(ShellProtoLogGroup.WM_SHELL_TRANSITIONS, "onTransitionReady (#%d) %s: %s",
        info.getDebugId(), transitionToken, info.toString("    " /* prefix */));
```

**注意这条代码单独是编译不过的**——`protologtool`会预处理`ShellProtoLogGroup`里的常量,把它变成编译期可判定的形式。

**日志分组定义**在`libs/WindowManager/Shell/src/com/android/wm/shell/protolog/ShellProtoLogGroup.java`。WMShell的组名都以`WM_SHELL_`开头,比如:

- `WM_SHELL_TRANSITIONS`
- `WM_SHELL_TASK_ORG`
- `WM_SHELL_PICTURE_IN_PICTURE`

**开启WMShell的ProtoLog**:

```shell
adb shell dumpsys activity service SystemUIService WMShell protolog enable-text WM_SHELL_TRANSITIONS
```

关闭把`enable-text`换成`disable-text`。

**开启WMCore的ProtoLog**用的是另一条通道:

```shell
adb shell wm logging enable-text TAG
adb shell wm logging disable-text TAG
```

TAG从`core/java/com/android/internal/protolog/ProtoLogGroup.java`里找。

**Kotlin的限制**:ProtoLog在Kotlin里**不做预处理**(`protologtool`还没有Kotlin支持),所以Kotlin代码里的ProtoLog没有内存收益,运行时要做字符串驻留,性能略差。日志仍然会进Perfetto。

**另一个限制**:Shell库里**不支持非文本ProtoLog**,所以没法在Winscope里以二进制形式查看Shell日志。

### 12.3 Shell命令

WMShell自己有一批命令,挂在SystemUI的dump入口下:

```shell
# 列出所有可用命令
adb shell dumpsys activity service SystemUIService WMShell help

# 执行某个命令
adb shell dumpsys activity service SystemUIService WMShell <cmd> <args> ...
```

这些命令来自各模块`onInit`里的注册:

```java
mShellCommandHandler.addCommandCallback("transitions", this, this);
```

已知的命令名(对应`addCommandCallback`的第一个参数):

| 命令 | 注册处 |
|---|---|
| `transitions` | `Transitions.onInit` |
| `pip` | `PipController.onInit` |
| 分屏 | `SplitScreenShellCommandHandler` |

`ShellCommandHandler`的接口里有三个概念要分清:

```java
public void addCommandCallback(String command, ShellCommandActionHandler handler, Object target);
public void addCommandCallback(String command, ShellCommandActionHandler handler, Object target, boolean ignoreOnTestHarness);
public void addCommandCallback(String command, ShellCommandActionHandler handler, Object target, boolean ignoreOnTestHarness, boolean ignoreOnBugReport);
public void addDumpCallback(Dumpable dumpable, Object target);
```

- `command` + `handler`:命令名和处理者。
- `target`:对象引用,用于dump时定位来源。
- `ignoreOnTestHarness` / `ignoreOnBugReport`:某些命令在自动化测试或bugreport里应该被忽略。

`ShellCommandActionHandler`有两个方法:

```java
interface ShellCommandActionHandler {
    boolean onCommand(String[] args, PrintWriter pw);
}
```

返回`true`表示"我处理了",`false`表示"不是我的"。框架会继续问下一个。

`ShellController`的`handleCommand`负责线程切换(第5章引过):

```java
@Override
public boolean handleCommand(String[] args, PrintWriter pw) {
    try {
        boolean[] result = new boolean[1];
        mMainExecutor.executeBlocking(() -> {
            result[0] = mShellCommandHandler.handleCommand(args, pw);
        });
        return result[0];
    } catch (InterruptedException e) {
        throw new RuntimeException("Failed to handle Shell command in 2s", e);
    }
}
```

**`executeBlocking`配合2秒超时**是因为命令要返回值(PrintWriter的内容),不能异步。异常信息里的"in 2s"就是实际超时时间。

**另外WMCore自己有`adb shell wm`命令**,和Shell的命令是两套:

```shell
adb shell wm size          # 改分辨率
adb shell wm density       # 改density
adb shell wm shell ...     # 转发给WMShell
adb shell wm logging ...   # WMCore的ProtoLog
```

### 12.4 Winscope

Winscope是看"屏幕上到底发生了什么"的工具。

**抓trace的方式:**

```shell
# 开始抓取(具体命令随版本变化,通常是layers/window两个trace)
adb shell su root cmd window tracing start
...
adb shell su root cmd window tracing stop > /data/misc/wmtrace/wm_trace.winscope
```

**WMShell的过渡信息也在里面**,这是排查"动画没播""动画播错"的主要手段。`Transitions`里专门有tracer:

```java
if (android.tracing.Flags.perfettoTransitionTracing()) {
    mTransitionTracer = new PerfettoTransitionTracer();
} else {
    mTransitionTracer = new LegacyTransitionTracer();
}
```

`PerfettoTransitionTracer`会把过渡信息直接写进Perfetto trace(比旧版的`LegacyTransitionTracer`多了和Perfetto时间轴的关联)。分发时记录:

```java
mTransitionTracer.logDispatched(info.getDebugId(), mHandlers.get(i));
```

**所以从Winscope里能看到每次过渡由哪个Handler认领了**(`logDispatched`的第二个参数就是Handler)。排查"分屏的动画为什么由DefaultTransitionHandler播了"这类问题,直接看这行。

### 12.5 追踪SurfaceControl事务

Winscope有时不够——它能看到surface变了,但看不出**是哪段代码改的**。这时可以打开事务调用栈追踪。

```shell
# 开启(按方法名匹配)
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha,setPosition
# 或者按surface名字匹配
adb shell setprop persist.wm.debug.sc.tx.log_match_name com.android.systemui
adb reboot
adb logcat -s "SurfaceControlRegistry"

# 关闭
adb shell setprop persist.wm.debug.sc.tx.log_match_call ""
adb shell setprop persist.wm.debug.sc.tx.log_match_name ""
```

**必须重启一次**才能开启(之后改属性不用重启)。

两个属性可以只设一个,但**不过滤的话日志会非常多**。

**追"这个事务什么时候被apply的"**也要打开`merge`和`apply`:

```shell
adb shell setprop persist.wm.debug.sc.tx.log_match_call setAlpha,merge,apply
adb reboot
adb logcat -s "SurfaceControlRegistry"
```

文档里的说明:

> apply will dump logs of each setAlpha or merge call on that tx

也就是apply会把之前所有合并进来的调用一起打出来。**Shell里事务合并很常见**(`TransactionPool`复用、`SyncTransactionQueue`排队),不打开merge就看不全。

### 12.6 其它有用的系统属性

**追踪App进程里的Activity启动:**

```shell
adb shell setprop persist.wm.debug.start_activity true
adb reboot
adb logcat -s "Instrumentation"
```

**追踪`finish()`的调用点:**

```shell
adb shell setprop persist.wm.debug.finish_activity true
adb reboot
adb logcat -s "Instrumentation"
```

**Shell相关:**

| 属性 | 作用 |
|---|---|
| `persist.wm.debug.shell_transit_rotate` | 让旋转也走Shell过渡 |
| `persist.wm.debug.pip_keep_clear_areas_delay` | PiP避让区域更新的延迟(默认200ms) |
| `debug.crash_sysui` | 启动即崩,测试RescueParty |

**`persist.wm.debug.pip_keep_clear_areas_delay`在源码里:**

```java
private static final long PIP_KEEP_CLEAR_AREAS_DELAY =
        SystemProperties.getLong("persist.wm.debug.pip_keep_clear_areas_delay", 200);
```

它控制PiP被其他元素"挤开"时的防抖延迟。调大可以看清PiP的移动过程。

### 12.7 关键日志TAG

排查时按TAG过滤最省时间:

| TAG | 来源 | 内容 |
|---|---|---|
| `ShellTransitions` | `Transitions.TAG` | 过渡生命周期 |
| `ShellTaskOrganizer` | `ShellTaskOrganizer.TAG` | task接管 |
| `PipController` | `PipController.TAG` | 画中画 |
| `SurfaceControlRegistry` | SurfaceFlinger | 事务调用栈 |
| `Instrumentation` | App进程 | Activity启动/finish |
| `SystemUIService` | `SystemUIApplication.TAG` | SystemUI启动 |
| `SystemUIFactory` | `SystemUIInitializer.TAG` | Dagger初始化(注意TAG名是旧的) |
| `CentralSurfaces` | `CentralSurfaces.TAG` | 状态栏 |
| `NotificationShade` | Shade相关 | 通知面板 |

**注意`SystemUIInitializer`的TAG还是`"SystemUIFactory"`**——类改名了,TAG没改。按`SystemUIFactory`过滤才能看到Dagger初始化日志。

**Shell主线程的慢消息**是另一个必看的信号(第5章讲过):

```java
mainThread.getLooper().setSlowLogThresholdMs(MSGQ_SLOW_DISPATCH_THRESHOLD_MS,
        MSGQ_SLOW_DELIVERY_THRESHOLD_MS);
```

阈值都是30ms,logcat里表现为`Slow dispatch took XXXms` / `Slow delivery took XXXms`,TAG是Looper。

### 12.8 在Android Studio里调试

AOSP提供了sysui-studio工程,可以把SystemUI当普通App在Android Studio里断点调试。设计文档里也提到:**Shell库是SystemUI的一部分,用这个工程就能直接调Shell代码**。

但有个限制:

> ProtoLogs are only fully supported from soong builds (ie. via make/mp). In SysUI-studio it falls back to log via Logcat

在Android Studio里ProtoLog会退化成普通Logcat输出,看不到ProtoLog的完整能力。

### 12.9 一个典型排查流程

以"从桌面点开App,动画没有从图标位置放大展开"为例:

| 步骤 | 命令/手段 | 看什么 |
|---|---|---|
| 1 | `dumpsys activity service SystemUIService WMShell transitions` | 这次过渡是否存在、type是什么、track是几 |
| 2 | 开启`WM_SHELL_TRANSITIONS` ProtoLog | 找`animated by XXXHandler`那行,确认谁认领了 |
| 3 | Winscope抓trace | 时序上动画是否被开始了、被merge了还是被中止 |
| 4 | 若应由Launcher演 | 检查`registerRemote`的filter是否匹配上,看`TransitionFilter.matches`的条件 |
| 5 | 若Shell主线程慢 | `logcat`搜`Slow dispatch` |

第2步最关键:**`Transitions.dispatchTransition`里那句`ProtoLog.v(..., " animated by %s", ...)`直接给出答案**。如果打出来的是`DefaultTransitionHandler`,说明Launcher的filter没匹配上,问题在Launcher侧。

## 13 总结

### 13.1 架构回顾

整个结构的核心是**一次拆分**:

```text
原来(Android 9及以前):
  WindowManagerService  ──  窗口管理 + 系统栏策略 + 动画 + 系统UI
  (都在 system_server)

现在:
  WMCore (system_server)         WMShell (com.android.systemui 进程)
  ─────────────────────          ────────────────────────────────────
  窗口树的真相                    task的surface组织(TaskOrganizer)
  布局计算                       显示区域组织(DisplayAreaOrganizer)
  insets仲裁                     过渡动画(Transitions)
  transition的创建与收集         分屏/PiP/气泡/单手/桌面模式
  系统栏外观的决定                系统栏外观的呈现
```

拆分的边界是**机制与策略**:

- **机制留在WMCore**:谁在哪个位置、哪个窗口盖着哪个、insets最终给App多少。这些是全局唯一的真相,不能被任一进程私自改。
- **策略移到WMShell**:这次变化该演什么动画、分屏怎么切、PiP怎么缩。这些是产品行为,改动频繁。

拆分的实现手段是**两条通道**:

1. `TaskOrganizer` / `DisplayAreaOrganizer`:WMCore把"surface的组织权"下放给Shell。
2. `ITransitionPlayer` + `TransitionInfo`:WMCore把"这次变化是什么"告诉Shell,Shell演完回调。

而SystemUI本身在依赖注入上是完全独立的(Dagger三层),在启动上是完全独立的(`CoreStartable` + 拓扑排序),它和WMShell只通过那13个接口和`ShellInterface`握手。

### 13.2 三个scope、三条线程、三个通道

把这个教程压缩成三句话:

- **三个Dagger scope**:`@Singleton`(`GlobalRootComponent`)、`@WMSingleton`(`WMComponent`)、`@SysUISingleton`(`SysUIComponent`)。
- **三条主要线程**:SystemUI主线程、`wmshell.main`(`wmshell.anim`/`wmshell.splashscreen`/`wmshell.background`是其辅助)。
- **三个通道**:`TaskOrganizer`(task)、`DisplayAreaOrganizer`(显示区)、`ITransitionPlayer`(过渡)。

### 13.3 常见误解清单

以下是读旧资料时最容易带进来的错误认知,**都已按当前`refs/heads/main`源码核对**:

| 常见说法 | 实际情况 |
|---|---|
| 状态栏的主类叫`StatusBar` | 文件不存在。现在是`CentralSurfaces`(接口)+`CentralSurfacesImpl`(3289行),另有`CentralSurfacesEmptyImpl.kt` |
| `StatusBarState`有`DREAMING` | 只有`SHADE`/`KEYGUARD`/`SHADE_LOCKED`。DREAMING是`StatusBarStateController`上的独立boolean,也是`ScrimState`里的一个枚举值 |
| `TransitionInfo.Change`的`getMode()`返回`CHANGE_*`常量 | `CHANGE_*`不存在,返回的是`TRANSIT_*` |
| `WindowState.getInsetsHint()` | 方法不存在。用`isRequestedVisible(int)` / `getRequestedVisibleTypes()` / `getDisableFlags()` |
| `packages/SystemUI/theme`目录 | 不存在。主题代码在`packages/SystemUI/src/com/android/systemui/theme/` |
| `SystemUIFactory` / `SystemUIFactoryImpl` | 不存在。现在是`SystemUIInitializer` + `SystemUIInitializerImpl.kt` |
| `SystemUIComponent.java` | 不存在。是`dagger/SysUIComponent.java` + `dagger/ReferenceSysUIComponent.java` |
| `SystemUIApplication.mComponents` | 没有此字段。是`CoreStartable[] mServices` + Dagger multibinding的`Map<Class<?>, Provider<CoreStartable>>` |
| `@PluginFactory`是注解 | 不是。`PluginFactory<T>`是`PluginInstance`的嵌套类;插件注解在`plugin_core/.../annotations/` |
| 插件接口在`src/com/android/systemui/plugins/` | 接口在`packages/SystemUI/plugin_core/src/...`;`src/.../plugins/`只放`PluginDependencyProvider`、`PluginEnablerImpl`、`PluginsModule` |
| `PluginManagerImpl`在`src/.../plugins/` | 在`packages/SystemUI/shared/src/com/android/systemui/shared/plugins/` |
| `navBarWindowView`是导航栏窗口根视图 | 类不存在。现在根视图是`NavigationBarFrame` |
| `SplitScreenController`在`split/`目录 | 目录叫`splitscreen/` |
| 启动窗口模块在`startingwindow/` | 目录叫`startingsurface/`(类名仍是`StartingWindowController`) |
| `IPip.aidl`在`pip/`目录 | 在`common/pip/`目录;`PipController`在`pip/phone/` |
| `TYPE_NOTIFICATION_SHADE`不存在 | 存在,值2040,标`@hide` |
| WMShell有`@ShellSingleton`注解 | 不存在。WMShell用的是`@WMSingleton` |
| WMShell有`ShellModule` | 不存在。模块是`WMShellConcurrencyModule`/`WMShellBaseModule`/`WMShellModule`等 |
| WMShell有一个叫`IOManager`的类 | 不存在 |
| 有`ShellTransition.java`(单数) | 文件不存在。类是`Transitions`,内部的接口实现类才叫`ShellTransitionImpl`(单数) |
| Shell的Handler有数字优先级 | 没有。优先级靠`mHandlers`列表顺序,从末尾往前扫描 |
| 通过aconfig的`SHELL_TRANSITIONS`标志控制 | 不是。是`Transitions.ENABLE_SHELL_TRANSITIONS`常量 + `persist.wm.debug.shell_transit_rotate`系统属性 |
| system_server里有`KeyguardViewMediator` | 没有。`KeyguardViewMediator`在SystemUI进程(`com.android.systemui.keyguard`) |
| `StatusBarManagerService`负责启动SystemUI | 不参与。SystemServer直接`startServiceAsUser(SystemUIService)` |
| SystemUI和system_server共享UID | SystemUI的UID是`android.uid.systemui`,不是system |
| `RemoteAnimationAdapter`已标`@Deprecated` | 没有标。它仍然可用,定位是兼容通道 |

### 13.4 怎么继续深入

按这几条线索读源码效率最高:

1. **想理解初始化**:从`SystemUIInitializer.init()`开始,一路跟到`WMComponent.init()`和`SystemUIApplication.startServicesIfNeeded()`。这两个是分岔点。
2. **想理解过渡**:从`Transitions.onTransitionReady`开始,打断点看`mHandlers`里都有谁、谁认领了。
3. **想理解某个功能模块**:找它的`onInit()`,那里集中了所有注册动作;然后看它的`createExternalInterface()`和接口AIDL。
4. **想理解SystemUI的某个UI**:从`CoreStartable`实现类入手,看它的`start()`;再找它的窗口Controller,看`LayoutParams`。
5. **想理解两者交互**:grep `addExternalInterface` 和 `setShell|setPip|setSplitScreen`,这两处能覆盖大部分交互点。

### 13.5 参考资料

| 资源 | 路径 / 链接 |
|---|---|
| WMShell设计文档 | `libs/WindowManager/Shell/src/com/android/wm/shell/docs/`(`overview.md`、`sysui.md`、`dagger.md`、`threading.md`、`changes.md`、`debugging.md`、`extending.md`) |
| SystemUI设计文档 | `packages/SystemUI/docs/`(`dagger.md`)和`packages/SystemUI/README.md` |
| AOSP源码(主分支) | `https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/` |
| AOSP源码浏览 | `https://cs.android.com/android/platform/superproject/+/android-latest-release` |

**读文档时注意**:`docs/`下的文档更新滞后于代码。本教程中凡与文档冲突处,均以`refs/heads/main`的实际源码为准,并在对应位置标明了差异。
