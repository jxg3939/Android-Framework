
## 1. 进入ZygoteInit()

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static void main(String[] argv) {
    ZygoteServer zygoteServer = null;
    Runnable caller;
    try {
        // ZygotePreload资源
        if (!enableLazyPreload) {
            preload(bootTimingsTraceLog);
        }
        // 负责建立并维护Socket服务端，监听并接收进程创建请求，孵化新进程
        zygoteServer = new ZygoteServer(isPrimaryZygote);
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
            // r==null是在parent，   r!=null就在systemServer了
            if (r != null) {
                r.run();
                return;
            }
        }
        Log.i(TAG, "Accepting command socket connections");
        // 在zygote中是一个无限循环，如果fork了子进程，会返回一个runable
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }
    // We're in the child process and have exited the select loop. Proceed to execute the command
    if (caller != null) {
        // 这里调用run方法保证了run是运行在fork出来的这个主进程中的
        // 对于大部分应用来说，就是ActivityThread.main()
        caller.run();
    }
}  
```

## 2. 由ZygoteServer循环监听，fork子进程

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
Runnable runSelectLoop(String abiList) {
    while (true) {
        ... // 这上面有个无限pool阻塞的流程  不是looper
        // ZygoteConnection是Zygote服务端与客户端之间的单次连接处理器
        // 已连接的客户端(AMS...)发送数据
        ZygoteConnection connection = peers.get(pollIndex);
        boolean multipleForksOK = !isUsapPoolEnabled()&& ZygoteHooks.isIndefiniteThreadSuspensionSafe();
        final Runnable command =connection.processCommand(this, multipleForksOK);
        if (mIsForkChild) {
            // We're in the child. We should always have a command to run at
            // this stage if processCommand hasn't called "exec".
            if (command == null) {
                throw new IllegalStateException("command == null");
            }
            return command;
        }
        ...
    }
}
```

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
Runnable processCommand(ZygoteServer zygoteServer, boolean multipleOK) {
    ...
    try {
        pid = Zygote.forkAndSpecialize(...);
        // 从这里返回开始 zygote和fork出的新进程开始不同的执行逻辑
        if (pid == 0) {
            // in child
            zygoteServer.setForkChild(); // mIsForkChild设置为true
            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            return handleChildProc(parsedArgs, childPipeFd,
                    parsedArgs.mStartChildZygote);
        } else {
            // In the parent. A pid < 0 indicates a failure and will be handled in
            // handleParentProc.
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            handleParentProc(pid, serverPipeFd);
            return null;
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```

```java
// frameworks/base/core/java/com/android/internal/os/Zygote.java
static int forkAndSpecialize(...) {
    ZygoteHooks.preFork();
    int pid = nativeForkAndSpecialize(...);
    if (pid == 0) {
        // postfork在handleChildProc中结束
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
        if (gids != null && gids.length > 0) {
            NetworkUtilsInternal.setAllowNetworkingForProcess(containsInetGid(gids));
        }
    }
    // Set the Java Language thread priority to the default value for new apps.
    Thread.currentThread().setPriority(Thread.NORM_PRIORITY);
    ZygoteHooks.postForkCommon();
    return pid;
}
```

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor pipeFd, boolean isZygote) {
    closeSocket();
    Zygote.setAppProcessName(parsedArgs, TAG);
    // postFork结束
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.mInvokeWith != null) {
        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.mRemainingArgs);
        // Should not get here.
        throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
    } else {
        // 普通应用进程
        if (!isZygote) {
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mDisabledCompatChanges,
                    parsedArgs.mEnabledCompatChanges,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        } else { // 子zygote进程
            return ZygoteInit.childZygoteInit(
                    parsedArgs.mRemainingArgs  /* classLoader */);
        }
    }
}
```

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static Runnable zygoteInit(...) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();
    RuntimeInit.commonInit();
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges,
            enabledCompatChanges, argv, classLoader);
}
```

```java
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        long[] enabledCompatChanges, String[] argv, ClassLoader classLoader) {
    // 当应用调用 System.exit()时，立即终止进程而不执行shutdown hooks
    nativeSetExitWithoutCleanup(true);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
    // For apps started via Zygote, we install AppCompatCallbacks here early
    // so that they are available during static initialization of the app.
    // ActivityThread will re-install them with more complete data later.
    if (disabledCompatChanges != null || enabledCompatChanges != null) {
        AppCompatCallbacks.install(disabledCompatChanges, enabledCompatChanges, null, false,
                targetSdkVersion);
    }
    final Arguments args = new Arguments(argv);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // 找到startClass的public static void main 程序入口
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

```java
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
protected static Runnable findStaticMain(String className, String[] argv, ClassLoader classLoader) {
    Class<?> cl;
    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }
    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }
    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }
    return new MethodAndArgsCaller(m, argv);
}
```

```java
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;
    /** argument array */
    private final String[] mArgs;
    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }
    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```
