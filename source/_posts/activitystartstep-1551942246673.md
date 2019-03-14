---
title: Android App/Activity 启动流程分析
tags: [Activity 启动流程分析]
categories: [Android]
date: 2019-03-07 15:04:06
---

#  Android App/Activity 启动流程分析
首先我们带着问题来看:

1. 点击了图标之后系统道理做了哪些工作呢？
2. 应用进程是怎么被启动的呢？
3. Activity 的生命周期是什么时候被谁调用的呢？

本文将继续基于 **Android Nougat** 的 Frameworks 层源码的解答这些问题。

阅读建议：
如果你是首次阅读这个过程的源码，建议你忽略一些细枝末节的代码，先抓主干代码，从整体上理解代码的执行流程（右下角文章目录视图中可以点击跳转到相应章节），否则将会被细节的代码扰乱思路。最后可以回头多看几遍，这时候如果有需要可以追踪一些枝干代码，做到融会贯通。
## 1.1 调用过程分析
### 1.1.1 Launcher.onClick
在 Launcher app 的主 Activity —— Launcher.java 中，App 图标的点击事件最终会回调 Launcher.java 中的 `onClick` 方法，
[packages/apps/Launcher3/src/com/android/launcher3/Launcher.java：](https://android.googlesource.com/platform/packages/apps/Launcher3/+/nougat-release/src/com/android/launcher3/Launcher.java?autodive=0%2F)


```JAVA
public void onClick(View v) {
    ...
    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        // 从快捷方式图标启动
        onClickAppShortcut(v);
    } else if (tag instanceof FolderInfo) {
        // 文件夹
        if (v instanceof FolderIcon) {
           onClickFolderIcon(v);
        }
    } else if (v == mAllAppsButton) {
        // “所有应用”按钮
        onClickAllAppsButton(v);
    } else if (tag instanceof AppInfo) {
        // 从“所有应用”中启动的应用
        startAppShortcutOrInfoActivity(v);
    } else if (tag instanceof LauncherAppWidgetInfo) {
        // 组件
        if (v instanceof PendingAppWidgetHostView) {
            onClickPendingWidget((PendingAppWidgetHostView) v);
        }
    }
}

```

### 1.1.2 Launcher.onClickAppShortcut
如果是快捷方式图标，则调用 `onClickAppShortcut` 方法进而调用 `startAppShortcutOrInfoActivity` 方法：


```JAVA
@Thunk void startAppShortcutOrInfoActivity(View v) {
    Object tag = v.getTag();
    final ShortcutInfo shortcut;
    final Intent intent;
    if (tag instanceof ShortcutInfo) {
        shortcut = (ShortcutInfo) tag;
        // 去除对应的 Intent 对象
        intent = shortcut.intent;
        int[] pos = new int[2];
        v.getLocationOnScreen(pos);
        intent.setSourceBounds(new Rect(pos[0], pos[1],
                pos[0] + v.getWidth(), pos[1] + v.getHeight()));

    } else if (tag instanceof AppInfo) {
        shortcut = null;
        intent = ((AppInfo) tag).intent;
    } else {
        throw new IllegalArgumentException("Input must be a Shortcut or AppInfo");
    }

    // 调用 startActivitySafely 方法
    boolean success = startActivitySafely(v, intent, tag);
    mStats.recordLaunch(v, intent, shortcut);

    if (success && v instanceof BubbleTextView) {
        mWaitingForResume = (BubbleTextView) v;
        mWaitingForResume.setStayPressed(true);
    }
}

```

### 1.1.3 Launcher.startActivity
获取相应 App 的 **Intent** 信息之后，调用 `startActivity` 方法：
并设置Flags为**Intent.FLAG_ACTIVITY_NEW_TASK**,启动新的任务栈

```JAVA
private boolean startActivity(View v, Intent intent, Object tag) {
    // 启动新的任务栈
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    try {
        ...
        if (user == null || user.equals(UserHandleCompat.myUserHandle())) {
            StrictMode.VmPolicy oldPolicy = StrictMode.getVmPolicy();
            try {            
                StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder().detectAll()
                        .penaltyLog().build());
                // 调用 Activity 的 startActivity 方法
                startActivity(intent, optsBundle);
            } finally {
                StrictMode.setVmPolicy(oldPolicy);
            }
        } else {
            launcherApps.startActivityForProfile(intent.getComponent(), user,
                    intent.getSourceBounds(), optsBundle);
        }
        return true;
    } catch (SecurityException e) {      
        ...
    }
    return false;
}

```

1.1.4 Activity.startActivity
这里最终调用了 `Activity` 中的 `startActivity` 方法，并且设置 Flag 为 **FLAG_ACTIVITY_NEW_TASK**。到此为止，已经跟启动普通的 `Activity` 流程汇合起来了，继续往下分析。
[frameworks/base/core/java/android/app/Activity.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/android/app/Activity.java)


```JAVA
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    // 第二个参数为 -1 表示不需要回调 onActivityResult 方法
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```
### 1.1.5 Activity.startActivityForResult
调用 `Activity` 的 `startActivityForResult` 方法


```JAVA
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
           @Nullable Bundle options) {
    // mParent 是当前 Activity 的父类，此时条件成立
    if (mParent == null) {
        // 调用 Instrumentation 的 execStartActivity 方法
        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this,
               mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
        ...
    } else {
        ...
    }
}
```

### 1.1.6 Instrumentation.execStartActivity

[frameworks/base/core/java/android/app/Instrumentation.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/android/app/Instrumentation.java)


```JAVA
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
        ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        // 获取 AMS 的代理对象并调用其 startActivity 方法
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

### 1.1.7 ActivityManagerProxy.startActivity

以上过程是在 `Launcher App `所在的进程中发生的，**由于远程 **`AMS(ActivityManagerService) `跟使用 `Service` 的 `Activity` **不在同一个进程中**，因此他们之间交互需要通过 **Binder IPC 机制**的支持，在这个过程中`Client` 首先获取到 `Server` 端的代理对象，在 `Client` 看来 `ActivityManagerProxy` 对象同样具有 `ActivityManagerService` 本地对象承诺的能力，因此 `Client` 可以调用 `ActivityManagerProxy` 跟 `ActivityManagerService` 对象进行数据交互，**`Binder` 驱动**作为桥梁在他们中间起到中间人的作用。
同样，`AMS` 是运行在 `system_server` 线程中的，这时 `AMS` 就相当于 AIDL 中的远程 服务端，App 进程要与 AMS 交互，需要通过 **AMS的代理**对象 `ActivityManagerProxy` 来完成，来看 `ActivityManagerNative.getDefault()` 拿到的是什么：
[frameworks/base/core/java/android/app/ActivityManagerNative.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/android/app/ActivityManagerNative.java)

```JAVA
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

`getDefault` 是一个静态变量：


```JAVA
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        // 向 ServiceManager 查询一个 key 为 "activity" 的引用
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```


`ServiceManager` **是 Binder IPC通信过程的核心**，是上下文的管理者，`Binder` 服务端必须先向 `ServerManager` 注册才能够为客户端提供服务，`Binder` 客户端在与服务端通信之前需要从 `ServerManager` **中查找并获取 `Binder` 服务端的引用**。

这里通过 "**activity**" 这个名字向 `ServiceManager` 查询 `AMS` 的引用，获取 `AMS` 的引用后，调用 `asInterface` 方法：


```JAVA
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    // 根据 descriptor 查询 obj 是否为 Binder 本地对象，具体过程请看前文中提到的文章
    IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    // 如果 obj 不是 Binder 本地对象，则将其包装成代理对象并返回
    return new ActivityManagerProxy(obj);
}
```

因为 `AMS` 与 `Launcher App` 不在同一个进程中，这里返回的 `IBinder` 对象是一个 `Binder` 代理对象，因此这类将其包装成 `ActivityManagerProxy`对象并返回，`ActivityManagerProxy` 是`ActivityManagerNative` 的内部类，查看 `ActivityManagerProxy` 类 ：

```JAVA
class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        ...
        // 调用号为 START_ACTIVITY_TRANSACTION
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    ...
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId) throws RemoteException
    {
        ...
        mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        ComponentName res = ComponentName.readFromParcel(reply);
        data.recycle();
        reply.recycle();
        return res;
    }
    ...
}
```
可以看到，`ActivityManagerProxy` 里面将客户端的请求通过 `mRemote.transact`  进行转发，`mRemote` 对象正是 `Binder` 驱动返回来的 **Binder 服务端的 Proxy** 对象，通过 这个`Binder Proxy`，`Binder` 驱动最终将调用处于 `Binder Server` 端 `ActivityManagerNative` 中的 `onTransact` 方法：


```JAVA
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    // 根据方法调用号 code 决定调用哪个方法
    switch (code) {
    case START_ACTIVITY_TRANSACTION:
    {
        ...
        // 调用 startActivity 方法
        int result = startActivity(app, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
        reply.writeNoException();
        reply.writeInt(result);
        return true;
    }
    ...
    case START_SERVICE_TRANSACTION: {
        ...
        ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
            reply.writeNoException();
            ComponentName.writeToParcel(cn, reply);
            return true;
        }
        ...
    }
}
```
### 1.1.8 ActivityManagerService.startActivity
`ActivityManagerNative` 是一个抽象类，它的 `startActivity` 为抽象方法，具体的实现在 [frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/services/core/java/com/android/server/am/ActivityManagerService.java) 中：


```JAVA
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```
## 1.2 小结
你应该可以发现，相对于 `AIDL` 的调用过程，调用方 `Launcher App` 相当于 AIDL 过程中的 `Clinent`端；AMS 相当于远程 Service 的角色，充当 Server 端角色，他们的调用过程总体上都是一样的。
从 Launcher App 到 AMS 的时序图如下：

![](https://ws1.sinaimg.cn/large/007lnl1egy1g0u54pqf70j30sk0gqabi.jpg)


# 2. AMS —— zygote

## 2.1 调用过程分析
### 2.1.1 ActivityManagerService.startActivityAsUser
接着从 `AMS` 的 `startActivityAsUser` 方法开始分析：


```JAVA
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    // 调用 ActivityStarter 的 startActivityMayWait 方法
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, null);
}
```

### 2.1.2 ActivityStarter.startActivityMayWait
继续跟进 [frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/services/core/java/com/android/server/am/ActivityStarter.java)


```JAVA
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
        Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask) {
   ...
   synchronized (mService) {
        ...
        // 调用 startActivityLocked 方法
        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                inTask);
        ...
        return res;
    }
}
```

### 2.1.3 ActivityStarter.startActivityLocked
查看 `startActivityLocked` 方法：


```JAVA
final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
        TaskRecord inTask) {
    ...
    // 调用 doPendingActivityLaunchesLocked 方法，传入 false 参数
    doPendingActivityLaunchesLocked(false);
    ...
    return err;
}

```

### 2.1.4 ActivityStarter.doPendingActivityLaunchesLocked
查看 `doPendingActivityLaunchesLocked` 方法：


```JAVA
final void doPendingActivityLaunchesLocked(boolean doResume) {
    while (!mPendingActivityLaunches.isEmpty()) {
        final PendingActivityLaunch pal = mPendingActivityLaunches.remove(0);
        final boolean resume = doResume && mPendingActivityLaunches.isEmpty();
        try {
            // 调用 startActivityUnchecked 方法
            final int result = startActivityUnchecked(pal.r, pal.sourceRecord, null, null,
                pal.startFlags, resume, null, null);
            postStartActivityUncheckedProcessing(pal.r, result, mSupervisor.mFocusedStack.mStackId, 
                mSourceRecord, mTargetStack);
        } catch (Exception e) {
            Slog.e(TAG, "Exception during pending activity launch pal=" + pal, e);
            pal.sendErrorResult(e.getMessage());
        }
    }
}
```
### 2.1.5 ActivityStarter.startActivityUnchecked
查看 `startActivityUnchecked` 方法：


```JAVA
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
    ...  
    // 调用 ActivityStackSupervisor 的 resumeFocusedStackTopActivityLocked 方法
    mSupervisor.resumeFocusedStackTopActivityLocked();  
    ... 
    return START_SUCCESS;
}

```

### 2.1.6 ActivityStackSupervisor.resumeFocusedStackTopActivityLocked

[frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/services/core/java/com/android/server/am/ActivityStackSupervisor.java)


```JAVA
boolean resumeFocusedStackTopActivityLocked(ActivityStack targetStack, ActivityRecord target,
            ActivityOptions targetOptions) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || r.state != RESUMED) {
        // 调用 ActivityStack 的 resumeTopActivityUncheckedLocked 方法
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    }
    return false;
}
```


### 2.1.7 ActivityStack.resumeTopActivityUncheckedLocked
查看 `ActivityStack` 的 `resumeTopActivityUncheckedLocked` 方法：
```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    try {
        ...
        // 调用 resumeTopActivityInnerLocked 方法
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    return result;
}
```

### 2.1.8 ActivityStack.resumeTopActivityInnerLocked
查看 `resumeTopActivityInnerLocked` 方法：
```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    final ActivityRecord next = topRunningActivityLocked();
    ...
    if (next.app != null && next.app.thread != null) {
        ...
    } else {
        ...
        if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
        // 调用 ActivityStackSupervisor 的 startSpecificActivityLocked 方法
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
    
    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```
### 2.1.9 ActivityStackSupervisor.startSpecificActivityLocked
回到 `ActivityStackSupervisor` 的 `startSpecificActivityLocked` 方法：
```java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // 当前 Activity 附属的 Application
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);
    r.task.stack.setLaunchTime(r);
    // 如果 Application 已经运行
    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                        mService.mProcessStats);
            }
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
    }
    // 如果 Application 没有运行,调用AMS,启动新进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```
首先，在方法中获取了当前 `Activity` 附属的 `Application`，如果已经在运行了，说明这个 App 是已经被启动过了的，这时候调用 `realStartActivityLocked` 方法就可以进入下一步的流程了，同一个 App 中不同 `Activity` 的相互启动就是走的这个流程。当 `Application` 没有运行的时候，就需要调用 `AMS` 的 `startProcessLocked` 方法启动一个进程去承载它然后完成后续的工作，顺便铺垫一下，当新进程被启动完成后还会调用回到这个方法，查看 `AMS` 的 `startProcessLocked` 方法：
### 2.1.10 ActivityManagerService.startProcessLocked

```java
final ProcessRecord startProcessLocked(String processName,
        ApplicationInfo info, boolean knownToBeDead, int intentFlags,
        String hostingType, ComponentName hostingName, boolean allowWhileBooting,
        boolean isolated, boolean keepIfLarge) {
    return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
            hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
            null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
            null /* crashHandler */);
}
```
### 2.1.11 ActivityManagerService.startProcessLocked
调用 `startProcessLocked` 方法：
```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
        boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler){
    ...
    startProcessLocked(app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
    checkTime(startTime, "startProcess: done starting proc!");
    return (app.pid != 0) ? app : null;
}
```
### 2.1.12 ActivityManagerService.startProcessLocked
调用 `startProcessLocked` 的重载方法：
```java
private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs){
    ...
    try {
        ...
        // 调用 Process 的 start 方法
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);
        ...
    } catch (RuntimeException e) {
        ...
    }
}
```
### 2.1.13 Process.start
[frameworks/base/services/core/java/android/os/Process.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/android/os/Process.java)
```java
public static final ProcessStartResult start(final String processClass,
                                final String niceName,
                                int uid, int gid, int[] gids,
                                int debugFlags, int mountExternal,
                                int targetSdkVersion,
                                String seInfo,
                                String abi,
                                String instructionSet,
                                String appDataDir,
                                String[] zygoteArgs) {
    try {
        // 调用 startViaZygote 方法
        return startViaZygote(processClass, niceName, uid, gid, gids,
                debugFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}
```
### 2.1.14 Process.startViaZygote
查看 `startViaZygote` 方法：
```java
private static ProcessStartResult startViaZygote(final String processClass,
                                final String niceName,
                                final int uid, final int gid,
                                final int[] gids,
                                int debugFlags, int mountExternal,
                                int targetSdkVersion,
                                String seInfo,
                                String abi,
                                String instructionSet,
                                String appDataDir,
                                String[] extraArgs)
                                throws ZygoteStartFailedEx {
    synchronized(Process.class) {
        ...
        // 调用 zygoteSendArgsAndGetResult 方法，传入 openZygoteSocketIfNeeded 的返回值
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
}
```
### 2.1.15 Process.zygoteSendArgsAndGetResult、Process.openZygoteSocketIfNeeded
查看 `zygoteSendArgsAndGetResult` 方法：
```java
private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
    try {
        ...
        final BufferedWriter writer = zygoteState.writer;
        final DataInputStream inputStream = zygoteState.inputStream;

        writer.write(Integer.toString(args.size()));
        writer.newLine();

        for (int i = 0; i < sz; i++) {
            String arg = args.get(i);
            writer.write(arg);
            writer.newLine();
        }

        writer.flush();

        // Should there be a timeout on this?
        ProcessStartResult result = new ProcessStartResult();

        // 等待 socket 服务端（即zygote）返回新创建的进程pid;
        result.pid = inputStream.readInt();
        result.usingWrapper = inputStream.readBoolean();

        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }
        return result;
    } catch (IOException ex) {
        zygoteState.close();
        throw new ZygoteStartFailedEx(ex);
    }
}
```
在 `zygoteSendArgsAndGetResult` 中等待 `Socket` 服务端，也就是 `zygote` 进程返回创建新进程的结果，这里 `zygoteState` 参数是由 `openZygoteSocketIfNeeded` 方法返回的，`openZygoteSocketIfNeeded` 方法则负责根据 `abi` 向 `Zygote` 进程发起连接请求：
```java
private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
        try {
            // 向主zygote发起connect()操作
            primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
        }
    }

    if (primaryZygoteState.matches(abi)) {
        return primaryZygoteState;
    }

    if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
        try {
            // 当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
        }
    }

    if (secondaryZygoteState.matches(abi)) {
        return secondaryZygoteState;
    }

    throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}
```
## 2.2 小结
如果是从桌面新启动一个 App 中的 `Activity`，此时是没有进程去承载这个 App 的，因此需要通过 `AMS` 向 `zygote` 继承发起请求去完成这个任务，AMS 运行在 `system_server` 进程中，它通过 `Socket` 向 `zygote` 发起 `fock` 进程的请求，从 `AMS` 开始的调用时序图如下：

![](https://ws1.sinaimg.cn/large/007lnl1egy1g0u5m53eloj30t50p7tcl.jpg)



# 3. zygote —— ActivityThread
## 3.1 调用过程分析
### 3.1.1 ZygoteInit.main

`zygote` 进程的其中一项任务就是：

调用 `registerZygoteSocket() `函数建立 `Socket` 通道，使 `zygote` 进程成为 `Socket` 服务端，并通过` runSelectLoop()` 函数等待 `ActivityManagerService` 发送请求创建新的应用程序进程。

`zygote` 终于要再次上场了！接下来从 **ZygoteInit.java** 的 `main` 方法开始回顾一下 `zygote` 进程的工作：

[frameworks/base/core/java/com/android/internal/os/ZygoteInit.java：](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/com/android/internal/os/RuntimeInit.java)
```java
public static void main(String argv[]) {
    try {
        ...
        runSelectLoop(abiList);
        ....
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}
```
### 3.1.2 ZygoteInit.runSelectLoop
查看 `runSelectLoop` 方法：
```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ...
    // 循环读取状态
    while (true) {
        ...
        for (int i = pollFds.length - 1; i >= 0; --i) {
            // 读取的状态不是客户端连接或者数据请求时，进入下一次循环
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {// i = 0 表示跟客户端 Socket 连接上了
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {// i > 0 表示接收到客户端 Socket 发送过来的请求
                // runOnce 方法创建一个新的应用程序进程
                boolean done = peers.get(i).runOnce();
                if (done) {
                    peers.remove(i);
                    fds.remove(i);
                }
            }
        }
    }
}
```
### 3.1.3 ZygoteConnection.runOnce
查看 [frameworks/base/core/java/com/android/internal/os/](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/com/android/internal/os/RuntimeInit.java)
**ZygoteConnection.java** 的 `runOnce` 方法：
```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;

    try {
        // 读取 socket 客户端发送过来的参数列表
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    } catch (IOException ex) {
        // EOF reached.
        closeSocket();
        return true;
    }
    ...
    try {
        // 将 socket 客户端传递过来的参数，解析成 Arguments 对象格式
        parsedArgs = new Arguments(args);
        ...
        // 同样调用 Zygote.java 的 forkAndSpecialize 方法 fock 出子进程
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                parsedArgs.appDataDir);
    } catch (Exception e) {
        ...
    }

    try {
        if (pid == 0) {
            // 子进程执行
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            // 进入子进程流程
            handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            return true;
        } else {
            // 父进程执行
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```
### 3.1.4 ZygoteConnection.handleChildProc
首先解析 `Socket` 客户端传过来的参数，`Zygote.java` 的 `forkAndSpecialize` 返回的 `pid == 0` 的时候表示此时在 `fock` 出来的子进程中执行，继续调用 `handleChildProc` 方法，并将参数继续层层传递：
```JAVA
private void handleChildProc(Arguments parsedArgs, FileDescriptor[] 
    descriptors, FileDescriptor pipeFd, PrintStream newStderr) throws ZygoteInit.MethodAndArgsCaller {
    /*由于 fock 出来的 system_server 进程会复制 zygote 进程的地址空间，因此它也得到了 zygote
    进程中的 Socket，这个 Socket 对它来说并无用处，这里将其关闭 
    */
    closeSocket();
    ZygoteInit.closeServerSocket();
    ...
    if (parsedArgs.niceName != null) {
        // 设置进程名
        Process.setArgV0(parsedArgs.niceName);
    }

    if (parsedArgs.invokeWith != null) {
        ...
    } else {
        // 调用 RuntimeInit 的 zygoteInit 方法
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                parsedArgs.remainingArgs, null);
    }
}
```
### 3.1.5 RuntimeInit.zygoteInit
查看 [frameworks/base/core/java/com/android/internal/os/RuntimeInit.java ](https://android.googlesource.com/platform/frameworks/base/+/nougat-release/core/java/com/android/internal/os/RuntimeInit.java)的 `zygoteInit` 方法：
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, 
            ClassLoader classLoader) throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    // 重定向 log 输出
    redirectLogStreams();
    // 初始化一些通用的设置
    commonInit(); 
    /**
     *通过 Native 层中 AndroidRuntime.cpp 的 JNI 方法最终调用 app_main.cpp 的 
     *onZygoteInit 方法启动 Binder 线程池， 使 system_server 进程可以使用 Binder  *与其他进程通信
     **/
    nativeZygoteInit(); 
    applicationInit(targetSdkVersion, argv, classLoader);
}
```
### 3.1.6 RuntimeInit.applicationInit
继续调用 `applicationInit` 方法：
```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    ...
    // 提取出参数里面的要启动的类的名字
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```
### 3.1.7 RuntimeInit.invokeStaticMain
主要调用了 `invokeStaticMain` 方法：
```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
         throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;
    try {
        /** className 为通过 Socket 客户端（AMS）传递过来的一系列参数中的其中一个，这里获取到的值为传"com.android.app.ActivityThread"，然后通过反射得到 ActivityThread 类 **/
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
            "Missing class when invoking static main " + className, ex);
    }
    Method m;
    try {
        // 找到 ActivityThread 类的 main 方法
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
    /** 将 main 方法包装在 ZygoteInit.MethodAndArgsCaller 类中并作为异常抛出
    捕获异常的地方在上一小节中 ZygoteInit.java 的 main 方法 **/
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```
### 3.1.8 MethodAndArgsCaller.run
回到 `ZygoteInit` 的 `main` 方法：
```java
public static void main(String argv[]) {
    ...
    closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        // 接收到 caller 对象后调用它的 run 方法
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```
跟 `system_server` 进程的启动过程一样，这里同样通过抛出异常的方式来清空调用 `ActivityThread.main `之前的方法栈帧。
`ZygoteInit` 的 `MethodAndArgsCaller` 类是一个 `Exception` 类，同时也实现了 `Runnable` 接口：
```java
public static class MethodAndArgsCaller extends Exception
        implements Runnable {
        
    private final Method mMethod;
    private final String[] mArgs;
        
    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }
    public void run() {
        try {
            // 调用传递过来的 mMethod
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            ...
        }
    }
}
```
### 3.1.9 ActivityThread .main
最后通过反射调用到 `ActivityThread` 的 `main` 方法：
```java
public static void main(String[] args) {
    ...
    Environment.initForCurrentUser();
    ...
    Process.setArgV0("<pre-initialized>");
    // 创建主线程 Looper
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    // attach 到系统进程
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    // 主线程进入轮询状态
    Looper.loop();

    // 抛出异常说明轮询出现问题
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
## 3.2 小结
`zygote` 进程作为 `Socket` 服务端在接收到作为客户端的 `AMS` 发送过来的请求和参数之后，`fock` 出新的进程并根据各种参数进程了初始化的工作，这个过程和 `zygote` 启动 `system_server` 进程的过程如出一辙，时序图如下所示：
![](https://ws1.sinaimg.cn/large/007lnl1egy1g0u80ukv7ej30q30ivmzb.jpg)


# 4. ActivityThread —— Activity
##4.1 调用过程分析
### 4.1.1 ActivityThread.attach
上一小节的最后，`ActivityThread` 的 `main` 通过反射被运行起来了，接着会调用 `ActivityThread` 的 `attach` 方法：


```JAVA
private void attach(boolean system) {
    ...
    mSystemThread = system;
    if (!system) {
        ...
        // 获取 ActivityManagerProxy 对象
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            // 通过 Binder 调用 AMS 的 attachApplication 方法
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    } else {
        ...
    }
    ...
}
```

这里，我们再一次通过 `Binder IPC` 机制跟 `AMS` 通信，通信模型跟前面` Launcher App `调用 `AMS` 的 `startActivity` 方法一样，getDefault 过程不重复分析，这次是调用了 `AMS` 的 `attachApplication` 方法，注意这里将 `ApplicationThead` 类型的 `mAppThread` 对象作为参数传递了过去，`ApplicationThead` 是 `ActivityThread` 的一个内部类，后面我们会讲到，先查看 `AMP` 的 `attachApplication` 方法：
### 4.1.2 ActivityManagerProxy.attachApplication
```java
public void attachApplication(IApplicationThread app) throws RemoteException {
    ...
    // 调用 asBinder 方法使其能够跨进程传输
    data.writeStrongBinder(app.asBinder());
    // 通过 transact 方法将数据交给 Binder 驱动
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0); 
    reply.readException();
    data.recycle();
    reply.recycle();
}
```
### 4.1.3 ActivityManagerNative.onTransact
```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
        ...
        case ATTACH_APPLICATION_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            // 获取 ApplicationThread 的代理对象，这里返回的是 ApplicationThreadNative(ATN)
            // 的内部类：ApplicationThreadProxy(ATP) 对象
            IApplicationThread app = ApplicationThreadNative.asInterface(data.readStrongBinder());
            if (app != null) {
                // 委托给 AMS 执行
                attachApplication(app);
            }
            reply.writeNoException();
            return true;
        }
        ...
    }
}
```
`asInterface` 将 `ActivityThread` 对象**转换**成了 `ApplicationThreadNative` 的 `Binder` 代理对象 `ApplicationThreadProxy`，并作为参数传给 `attachApplication` 方法，其中 `ApplicationThreadProxy` 是 `ApplicationThreadNative` 的**内部类**。
### 4.1.4 ActivityManagerService.attachApplication
```java
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
```
### 4.1.5 ActivityManagerService.attachApplicationLocked
```java
private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
    ProcessRecord app;
    ...
    try {
        // 绑定死亡通知
        AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        // 如果 system_server 进程死亡则重新启动进程
        startProcessLocked(app, "link fail", processName); 
        return false;
    }
    ...
    try {
        ...
        // 获取应用appInfo
        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        ...
        // 绑定应用
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked());
        ...
    } catch (Exception e) {
        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        // bindApplication 失败也要重启进程
        startProcessLocked(app, "bind fail", processName);
        return false;
    }
    // 如果是 Activity: 检查最顶层可见的Activity是否等待在该进程中运行
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            badApp = true;
        }
    }
    // 如果是 Service: 寻找所有需要在该进程中运行的服务
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            badApp = true;
        }
    }

    // 如果是 BroadcastReceiver: 检查是否在这个进程中有下一个广播接收者
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            badApp = true;
        }
    }
    // 检查是否在这个进程中有下一个 backup 代理
    if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
        ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
        try {
            thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                    compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                    mBackupTarget.backupMode);
        } catch (Exception e) {
            badApp = true;
        }
    }
    if (badApp) { 
        // 杀掉 badApp
        app.kill("error during init", true);
        handleAppDiedLocked(app, false, true);
        return false;
    }
    if (!didSomething) {
        // 更新 adj(组件的权值)
        updateOomAdjLocked(); 
    }
    return true;
}
```

首先，通过 `ApplicationThreadProxy` 使用 `Binder` 向 `ApplicationThreadProxy` 发起 `bindApplication` 请求，然后通过 `normalMode` 字段判断是否为 `Activity`，如果是则执行 `ActivityStackSupervisor` 的 `attachApplicationLocked` 方法。
#### 4.1.5.1 ActivityThread.java::ApplicationThread.bindApplication
`thread` 对象类型是 `ApplicationThreadProxy`，通过 `Binder` 驱动调到了 `ApplicationThreadNative` 的方法，`ApplicationThreadNative` 是一个抽象类，它的实现都委托给了 `ApplicationThread`(这跟 AMS 跟 AMN 的关系一样)，ApplicationThread 作为 ActivityThread 的内部类存在，它的 binderApplication 方法如下：
```java
ActivityThread.java::ApplicationThread：
public final void bindApplication(String processName, ApplicationInfo appInfo,
    List<ProviderInfo> providers, ComponentName instrumentationName, ProfilerInfo profilerInfo,
    Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,
    IUiAutomationConnection instrumentationUiConnection, int debugMode, boolean
    enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent, Configuration
    config, CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

    if (services != null) {
        // 将services缓存起来, 减少binder检索服务的次数
        ServiceManager.initServiceCache(services);
    }
    ...
    // 发送消息 H.BIND_APPLICATION 给 Handler 对象
    sendMessage(H.BIND_APPLICATION, data);
}
```
**H** 是 **ActivityThread** 中的一个 **Handler** 对象，用于处理发送过来的各种消息：
```java
private class H extends Handler {
    public static final int BIND_APPLICATION        = 110;
 
    public void handleMessage(Message msg) {
        ...
        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        ...
    }
}
```
调用了 `handleBindApplication` 方法：
```
private void handleBindApplication(AppBindData data) {
    // 获取 LoadedApk 对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    // 创建 ContextImpl 上下文
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    ...
    // 创建 Instrumentation 对象
    if (data.instrumentationName != null) {
        ...
    } else {
        mInstrumentation = new Instrumentation();
    }

    try {
        // 调用 LoadedApk 的 makeApplication 方法创建 Application
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
        ...
        mInstrumentation.onCreate(data.instrumentationArgs);
        // 调用 Application.onCreate 方法
        mInstrumentation.callApplicationOnCreate(app);
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```
#### 4.1.5.2 ActivityStackSupervisor.attachApplicationLocked
在 **4.1.4** 小节中通过 **Binder** 向 **ActivityThread** 发起 `bindApplication` 请求后，会根据启动组件的类型去做相应的处理，如果是 `Acitivity`，则会调用 **ActivityStackSupervisor** 的 `attachApplicationLocked` 方法：
```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFrontStack(stack)) {
                continue;
            }
            // 获取前台stack中栈顶第一个非 finishing 状态的 Activity
            ActivityRecord hr = stack.topRunningActivityLocked(null);
            if (hr != null) {
                if (hr.app == null && app.uid == hr.info.applicationInfo.uid && processName.equals(hr.processName)) {
                    try {
                        // 真正的启动 Activity
                        if (realStartActivityLocked(hr, app, true, true)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        throw e;
                    }
                }
            }
        }
    }
    ...
    return didSomething;
}
##### 4.1.5.2.1 ActivityStackSupervisor.realStartActivityLocked

前面 **2.1.8ActivityStackSupervisor.startSpecificActivityLocked**  小节中分析过，如果当前 `Activity` 依附的 `Application` 已经被启动，则调用 `realStartActivityLocked` 方法，否则创建新的进程，再创建新的进程之后，两个流程的在这里合并起来了：
```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app, boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    final ActivityStack stack = task.stack;
    try {
        ...
        app.forceProcessStateUpTo(mService.mTopProcessState);
        // 通过 Binder 调用 ApplicationThread 的 scheduleLaunchActivity 方法
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
        ...
    } catch (RemoteException e) {
        if (r.launchFailed) {
            // 第二次启动失败，则结束该 Activity
            mService.appDiedLocked(app);
            stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                    "2nd-crash", false);
            return false;
        }
        // 第一个启动失败，则重启进程
        app.activities.remove(r);
        throw e;
    }
    ...
    return true;
}
``` 
这里有一次使用 `Binder` 调用 `ApplicationThread` 的 `scheduleLaunchActivity` 方法。

##### 4.1.5.2.2 ApplicationThread.scheduleLaunchActivity

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo 
        info, Configuration curConfig, Configuration overrideConfig, CompatibilityInfo 
        compatInfo, String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle 
        state, PersistableBundle persistentState, List<ResultInfo> pendingResults, 
        List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward, 
        ProfilerInfo profilerInfo) {
    ...
    updateProcessState(procState, false);
    ActivityClientRecord r = new ActivityClientRecord();
    ...
    sendMessage(H.LAUNCH_ACTIVITY, r);
 }
```

上面提到过，**H** 是 **ActivityThread** 中一个 **Handler** 类，它接收到 `LAUNCH_ACTIVITY` 消息后会调用 `handleLaunchActivity` 方法。
##### 4.1.5.2.3 ActivityThread.handleLaunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    // 初始化 WMS
    WindowManagerGlobal.initialize();
    // 执行 performLaunchActivity 方法
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // 执行 handleResumeActivity 方法，最终调用 onStart 和 onResume 方法
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);

        if (!r.activity.mFinished && r.startsNotResumed) {
            r.activity.mCalled = false;
            mInstrumentation.callActivityOnPause(r.activity);
            r.paused = true;
        }
    } else {
        // 停止该 Activity
        ActivityManagerNative.getDefault()
            .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
    }
}
```

##### 4.1.4.2.4 ApplicationThread.performLaunchActivity

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        // Instrumentation 中使用反射创建 Activity
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        ...
    } catch (Exception e) {
        ...
    }

    try {
        // 创建 Application 对象并调用 Application 的 onCreate 方法
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            ...
            // attach 到 Window 上
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                // 设置主题
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                // 重新创建的 Activity
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                // 第一次创建的 Activity
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
        }
        ...
    }  catch (Exception e) {
        ...
    }
    return activity;
}
```

##### 4.1.5.2.5 Instrumentation.callActivityOnCreate


```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
    prePerformCreate(activity);
    // 调用 Activity 的 performCreate 方法
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

##### 4.1.5.2.6 Activity.performCreate
```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle, persistentState);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```

终于，onCreate 方法被调用了！！！

## 4.2 小结
从 `ActivityThread` 到最终 `Activity` 被创建及生命周期被调用，核心过程涉及到了三次** Binder IPC** 过程，分别是：
    
    1. ActivityThread 调用 AMS 的 attachApplication 方法
    
    2. AMS 调用 ApplicationThread 的 bindApplication 方法
    
    3. ActivityStackSupervisor 调用 Application 的 attachApplicationLocked 方法


整个过程的时序图如下：

![](https://ws1.sinaimg.cn/large/007lnl1egy1g0u8r1k48wj30t30qo43w.jpg)

5. 总结
纵观整个过程，从 Launcher 到 AMS、从 AMS 再到 Zygote、再从 Zygote 到 ActivityThread，最后在 ActivitThread 中层层调用到 Activity 的生命周期方法，中间涉及到了无数的细节，但总体上脉络还是非常清晰的，各个 Android 版本的 Framework 层代码可以某些过程的实现不太一样，但是整个调用流程大体上也是相同的，借用 [Gityuan](http://gityuan.com/android/) 大神的一张图作为结尾：

![](https://ws1.sinaimg.cn/large/007lnl1egy1g0u8shvqsuj30qo0k0gt0.jpg)

