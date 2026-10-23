## 1 Activity启动到showStartingWindow

```java
// frameworks/base/core/java/android/app/Activity.java
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

public void startActivity(Intent intent, @Nullable Bundle options) {
    getAutofillClientController().onStartActivity(intent, mIntent);
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(        // 关键跳板
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        ...
    } else {
        ...
    }
}
```

```java
// frameworks/base/core/java/android/app/Instrumentation.java
public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token,
        Activity target, Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    ...
    try {
        // ActivityTaskManager.getService()拿到ATMS的代理，然后通过这个代理对象调用其startActivity
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        ...
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

// 重载跳转
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, @CanBeCURRENT @UserIdInt int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

private int startActivityAsUser(...) {
    ...
    return getActivityStartController()
            .obtainStarter(intent, "startActivityAsUser")   // 拿一个"启动器"
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setRequestCode(requestCode)
            ...
            .execute();                                      // 执行启动
}
```

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
int execute() {
    ...
    res = executeRequest(mRequest);
    ...
}

private int executeRequest(Request request) {
    ...
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            .setCaller(callerApp)               // 谁发起的
            .setLaunchedFromPid(callingPid)     // 启动方pid
            .setLaunchedFromUid(callingUid)     // 启动方uid
            .setLaunchedFromPackage(callingPackage)
            .setIntent(intent)
            .setActivityInfo(aInfo)     // PMS解析出的ActivityInfo
            .setConfiguration(mService.getGlobalConfiguration())
            .setResultTo(resultRecord)
            .setRequestCode(requestCode)
            ...
            .build();
    ...
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, ...);
    ...
}

private int startActivityUnchecked(...) {
    ...
    result = startActivityInner(r, sourceRecord, ...);
    ...
}

int startActivityInner(...) {
    ...
    // 正式启动Activity（生命周期、可见性、动画）
    mTargetRootTask.startActivityLocked(...);
    ...
    // 恢复焦点Task并执行Resume
    if (mDoResume) {
        if (可聚焦) mRootWindowContainer.resumeFocusedTasksTopActivities(...);
        else ensureActivitiesVisible(...);
    }
    ...
}
```

```java
// frameworks/base/services/core/java/com/android/server/wm/Task.java
void startActivityLocked(@NonNull ActivityRecord r, @Nullable Task topTask, boolean newTask,
            boolean isTaskSwitch, @Nullable ActivityOptions options,
            @Nullable ActivityRecord sourceRecord) {
    ...
    // 默认要显示StartingWindow
    boolean doShow = true;
    // 如果是新建任务
    if (newTask) {
        // Intent带了FLAG_ACTIVITY_RESET_TASK_IF_NEEDED，会把有相同affinity的Activity归拢
        if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
            resetTaskIfNeeded(r, r);
            // 重置后只有该Activity是栈顶运行Activity时才显示StartingWindow
            doShow = topRunningActivity(null) == r;
        }
    // 由Scene Transition接管，不需要StartingWindow
    } else if (options != null && options.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
        doShow = false;
    }
    // 要求不显示StartingWindow
    if (options != null && options.getDisableStartingWindow()) {
        doShow = false;
    }
    // 是在后台任务中启动
    if (r.mLaunchTaskBehind) {
        // 保证WindowManager知道r是visible的
        r.setVisibility(true);
        ensureActivitiesVisible(null /* starting */);
        // 需要特殊处理，通知系统发生了一次可见性状态不确定的启动
        if (!r.isVisibleRequested()) {
            r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
        }
        // 后台启动，不走resume来触发应用过渡动画，手动调用executeAppTransition()强制执行过渡
        mDisplayContent.executeAppTransition();
    // 确认要显示StartingWindow
    } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
        // 查找 (存在StartingWindow相关的启动数据 且 允许向当前用户显示) 的activity，后面传入showStartingWindow中看能否复用
        final ActivityRecord prev = activityTask.getActivity(
                a -> a.mStartingData != null && a.showToCurrentUser());
        mWmService.mStartingSurfaceController.showStartingWindow(r, prev, newTask,
                isTaskSwitch, sourceRecord);
    }
}
```

```java
// frameworks/base/services/core/java/com/android/server/wm/StartingSurfaceController.java
void showStartingWindow(ActivityRecord target, ActivityRecord prev,
        boolean newTask, boolean isTaskSwitch, ActivityRecord source) {
    if (mDeferringAddStartingWindow) {
        addDeferringRecord(target, prev, newTask, isTaskSwitch, source);
    } else {
        target.showStartingWindow(prev, newTask, isTaskSwitch, true /* startActivity */,
                source);
    }
}
```

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java    
// 重载调用
void showStartingWindow(ActivityRecord prev, boolean newTask, boolean taskSwitch,
        boolean startActivity, ActivityRecord sourceRecord) {
    showStartingWindow(prev, newTask, taskSwitch, isProcessRunning(), startActivity,
            sourceRecord, null /* candidateOptions */);
}

void showStartingWindow(ActivityRecord prev, boolean newTask, boolean taskSwitch,
            boolean processRunning, boolean startActivity, ActivityRecord sourceRecord,
            ActivityOptions candidateOptions) {
    if (mTaskOverlay) {
        // We don't show starting window for overlay activities.
        return;
    }
    // 优先使用传入的candidateOptions
    final ActivityOptions startOptions = candidateOptions != null
            ? candidateOptions : mPendingOptions;
    if (startOptions != null && startOptions.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
        // Don't show starting window when using shared element transition.
        return;
    }
    // 从启动选项等相关信息中获取SplashScreen主题
    final int splashScreenTheme = startActivity ? getSplashscreenTheme(startOptions) : 0;
    // 确定最终启动窗口使用的主题
    final int resolvedTheme = evaluateStartingWindowTheme(prev, packageName, theme, splashScreenTheme);
    mSplashScreenStyleSolidColor = shouldUseSolidColorSplashScreen(sourceRecord, startActivity,
            startOptions, resolvedTheme, newTask);
    // 判断Activity是否已经创建
    final boolean activityCreated = mState.ordinal() >= STARTED.ordinal() && mState.ordinal() <= STOPPED.ordinal();
    // 判断为特殊的热启动场景
    final boolean newSingleActivity = !newTask && !activityCreated
            && task.getActivity((r) -> !r.finishing && r != this) == null;
    final boolean scheduled = addStartingWindow(packageName, resolvedTheme,
            prev, newTask || newSingleActivity, taskSwitch, processRunning,
            allowTaskSnapshot(), activityCreated, mSplashScreenStyleSolidColor, allDrawn);
    if (DEBUG_STARTING_WINDOW_VERBOSE && scheduled) {
        Slog.d(TAG, "Scheduled starting window for " + this);
    }
}


boolean addStartingWindow(String pkg, int resolvedTheme, ActivityRecord from, boolean newTask,
        boolean taskSwitch, boolean processRunning, boolean allowTaskSnapshot,
        boolean activityCreated, boolean isSimple, boolean activityAllDrawn) {
    // 如果当前显示器尚不允许显示窗口
    if (!okToDisplay()) {
        return false;
    }
    // 如果当前Activity已经有StartingWindow，不重复创建
    if (hasStartingWindow()) {
        return false;
    }
    // 查找当前Activity的主窗口，不包含StartingWindow
    final WindowState mainWin = findMainWindow(false /* includeStartingApp */);
    if (mainWin != null && mainWin.isDrawn()) {
        // 主窗口已经完成绘制，说明应用窗口已经可用，没有必要再显示StartingWindow
        return false;
    }
    // 获取当前Task的历史截图，用于判断是否可以使用Snapshot作为StartingWindow
    final TaskSnapshot snapshot = mWmService.mTaskSnapshotController.getSnapshot(task.mTaskId,
            Flags.onlyCacheLowResTaskSnapshot()
                    ? TaskSnapshotManager.RESOLUTION_ANY
                    : TaskSnapshotManager.RESOLUTION_HIGH);
    // 决定StartingWindow的类型，Snapshot或SplashScreen
    final int type = getStartingWindowType(newTask, taskSwitch, processRunning,
            allowTaskSnapshot, activityCreated, activityAllDrawn, snapshot);
    // 兼容性处理：如果最终选择的是 SplashScreen，
    // 并且目标应用属于需要特殊处理的应用，则使用旧版启动窗口实现。
    final boolean useLegacy = type == STARTING_WINDOW_TYPE_SPLASH_SCREEN
            && mWmService.mStartingSurfaceController.isExceptionApp(packageName, mTargetSdk,
                () -> {
                    // 查询目标 Activity 对应的 ActivityInfo，
                    // 再获取其 ApplicationInfo，供例外应用判断逻辑使用。
                    ActivityInfo activityInfo = intent.resolveActivityInfo(
                            mAtmService.mContext.getPackageManager(),
                            PackageManager.GET_META_DATA);
                    return activityInfo != null ? activityInfo.applicationInfo : null;
                });
    // 包装参数供StartingWindow创建使用
    final int typeParameter = StartingSurfaceController
            .makeStartingWindowTypeParameter(newTask, taskSwitch, processRunning,
                    allowTaskSnapshot, activityCreated, isSimple, useLegacy, activityAllDrawn,
                    type, isIconStylePreferred(resolvedTheme), packageName, mUserId);

    // 如果用Snapshot作StartingWindow。
    if (type == STARTING_WINDOW_TYPE_SNAPSHOT) {
        if (isActivityTypeHome()) {
            // Home 的 Snapshot 只使用一次，因为屏幕亮起期间，
            // 它不会持续更新（参见 TaskSnapshotController.screenTurningOff）。
            final Transition transition = mTransitionController.getCollectingTransition();
            // 如果当前正在进行窗口过渡，且不是带有
            // KEYGUARD_GOING_AWAY_NO_ANIMATION 标志的解锁过渡，
            // 则不使用 Home 的 Snapshot 作为启动窗口。
            if (transition != null && (transition.getFlags()
                    & WindowManager.TRANSIT_FLAG_KEYGUARD_GOING_AWAY_NO_ANIMATION) == 0) {
                // 从 Snapshot 缓存中移除 Home 的截图，
                // 避免后续继续使用这张不再更新的截图。
                mWmService.mTaskSnapshotController.removeSnapshotCache(task.mTaskId);
                // Home 的 Snapshot 只适用于直接解锁场景；
                // 其他过渡场景不使用它，直接结束本次处理。
                return false;
            }
            // 在移除缓存前增加引用计数，确保 Snapshot 后续仍可被使用。
            snapshot.addReference(TaskSnapshot.REFERENCE_WRITE_TO_PARCEL);
            // 移除缓存中的 Home Snapshot，避免重复使用。
            mWmService.mTaskSnapshotController.removeSnapshotCache(task.mTaskId);
        } else if (from != null && (mActivityComponent.equals(from.mActivityComponent)
                // 如果来源 Activity 与目标 Activity 属于同一个组件，
                // 或者来源 Activity 当前持有 SplashScreenStartingData，
                // 则尝试将来源 Activity 的 StartingWindow 转移给当前 Activity。
                || from.mStartingData instanceof SplashScreenStartingData)
                && transferStartingWindow(from)) {
            // 转移成功，直接返回，不再创建新的 Snapshot 启动窗口。
            return true;
        }
        // 创建基于 Task Snapshot 的启动窗口。
        // 如果是 Home，前面已处理 Snapshot 的引用计数和缓存移除；
        // 其他 Activity 则在必要时尝试复用来源 Activity 的 StartingWindow。
        return createSnapshot(snapshot, typeParameter);
    }
    // resolvedTheme为 0，但Activity原始主题不为0 应用最终没有可用的启动窗口主题，应用不希望显示StartingWindow
    if (resolvedTheme == 0 && theme != 0) {
        return false;
    }
    // 如果来源 Activity 存在，尝试将其 StartingWindow 转移给当前 Activity。
    // 转移成功则直接复用，不再创建新的启动窗口。
    if (from != null && transferStartingWindow(from)) {
        return true;
    }
    // 如果既不使用 Snapshot，也没有选择 SplashScreen，
    // 则不需要创建其他类型的启动窗口。
    if (type != STARTING_WINDOW_TYPE_SPLASH_SCREEN) {
        return false;
    }

    // 记录调试日志：即将创建 SplashScreen 启动窗口数据。
    ProtoLog.v(WM_DEBUG_STARTING_WINDOW, "Creating SplashScreenStartingData");

    // 创建 SplashScreenStartingData，保存 SplashScreen 的主题和类型参数。
    // 注意：这里创建的是启动窗口数据对象，并非直接创建 Surface 或完成绘制。
    mStartingData = new SplashScreenStartingData(mWmService, resolvedTheme, typeParameter);

    // 安排后续添加 StartingWindow 的操作。
    // 后续流程会根据 StartingData 创建对应的启动窗口及其 Surface。
    scheduleAddStartingWindow();

    // 已成功安排 SplashScreen 启动窗口的创建流程。
    return true;
}
```

