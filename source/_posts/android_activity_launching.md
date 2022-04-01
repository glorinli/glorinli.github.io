---
title: 【Android】Android Activity 启动流程简析(基于Android 10 代码)
date: 2020-07-27
categories:
- Android
tags:
- Android
- Framework
- ActivityManagerService
---
## 简介
本文基于 Android 10 代码，分析了 Activity 的启动流程，Activity 启动流程可以分为三大块：
1. 发起，在调用者进程中执行
2. 管理，在 system_server 进程中执行，主要是 ActivityManagerService 在做工作
3. 执行，在被启动的 Activity 所在的进程中执行

## 发起
我们知道，要启动一个 Activity，常见的有两种方式
1. 调用 Activity.startActivity
2. 调用 Context.startActivity

下面我们来分析一下这两种方式。
### Activity.startActivity
``` java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // ...
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        // ... 最终也是调用到Instrumentation.execStartActivity
    }
}
```

可见最终调用到了 Instrumentation.execStartActivity 方法。
### Context.startActivity
``` java
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
    // maintain this for backwards compatibility.
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```

可见最终也是调用到 Instrumentation.execStartActivity 方法。

### Instrumentation.execStartActivity
``` java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // ...
    try {
        // ...
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    }
    // ...
}
```
可以看到，这边调用了 ActivityTaskManager.getService().startActivity 方法

### ActivityTaskManager.getService()
``` java
public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
}

@UnsupportedAppUsage(trackingBug = 129726065)
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

这边getService方法，拿到的就是一个IActivityTaskManager对象，这个IActivityTaskManager.Stub.asInterface(b)很熟悉，是AIDL的写法，于是我们就猜测这边其实是远程调用了某个系统服务了，猜测名字是ActivityTaskManagerService，于是我们果然发现有一个ActivityManagerService类，在 framework\base\services\core\java\com\android\server\wm\ActivityTaskManagerService.java 下，这就是一个系统服务了，这边发生了IPC，于是转入第二阶段：管理。
## 管理
### ActivityTaskManagerService.startActivity
查看ActivityTaskManagerService的代码，发现最终调用是来到了startActivityAsUser方法：
``` java
int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivityAsUser");

    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();

}
```
这边有很多参数，我们注意第一个参数caller，类型是IApplicationThread，这是一个AIDL接口，实现类是ActivityThread的内部类ApplicationThread：
``` java
private class ApplicationThread extends IApplicationThread.Stub
```

getActivityStartController().obtainStarter这边是用了工厂模式，最终是获取一个ActivityStarter，顾名思义，这个ActivityStarter就是用来执行startActivity操作的。
### ActivityStarter
``` java
/**
 * Starts an activity based on the request parameters provided earlier.
 * @return The starter result.
 */
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                    mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        }
    } finally {
        onExecutionComplete();
    }
}
```

上面我们看到，Stater的setMaywait被调用了，因此mayWait为true，所以就会走到startActivityMayWait中。

### ActivityStarter.startActivityMayWait
``` java
private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }
    
    // .. Calling pid uid 设置 省略

    // Save a copy in case ephemeral needs it
    final Intent ephemeralIntent = new Intent(intent);
    // Don't modify the client's object!
    intent = new Intent(intent);
    if (componentSpecified
            && !(Intent.ACTION_VIEW.equals(intent.getAction()) && intent.getData() == null)
            && !Intent.ACTION_INSTALL_INSTANT_APP_PACKAGE.equals(intent.getAction())
            && !Intent.ACTION_RESOLVE_INSTANT_APP_PACKAGE.equals(intent.getAction())
            && mService.getPackageManagerInternalLocked()
                    .isInstantAppInstallerComponent(intent.getComponent())) {
        // intercept intents targeted directly to the ephemeral installer the
        // ephemeral installer should never be started with a raw Intent; instead
        // adjust the intent so it looks like a "normal" instant app launch
        intent.setComponent(null /*component*/);
        componentSpecified = false;
    }

    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
            0 /* matchFlags */,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, mRequest.filterCallingUid));
    if (rInfo == null) {
        UserInfo userInfo = mSupervisor.getUserInfo(userId);
        if (userInfo != null && userInfo.isManagedProfile()) {
            // Special case for managed profiles, if attempting to launch non-cryto aware
            // app in a locked managed profile from an unlocked parent allow it to resolve
            // as user will be sent via confirm credentials to unlock the profile.

            // ...
        }
    }
    // Collect information about the target of the Intent.
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

    synchronized (mService.mGlobalLock) {
        final ActivityStack stack = mRootActivityContainer.getTopDisplayFocusedStack();
        stack.mConfigWillChange = globalConfig != null
                && mService.getGlobalConfiguration().diff(globalConfig) != 0;
        if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                "Starting activity when config will change = " + stack.mConfigWillChange);

        final long origId = Binder.clearCallingIdentity();

        if (aInfo != null &&
                (aInfo.applicationInfo.privateFlags
                        & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0 &&
                mService.mHasHeavyWeightFeature) {
            // This may be a heavy-weight process!  Check to see if we already
            // have another, different heavy-weight process running.

            // ... 省略
        }

        final ActivityRecord[] outRecord = new ActivityRecord[1];
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);

        Binder.restoreCallingIdentity(origId);

        if (stack.mConfigWillChange) {
            // If the caller also wants to switch to a new configuration,
            // do so now.  This allows a clean switch, as we are waiting
            // for the current activity to pause (so we will not destroy
            // it), and have not yet started the next activity.
            mService.mAmInternal.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                    "updateConfiguration()");
            stack.mConfigWillChange = false;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Updating to new configuration after starting activity.");
            mService.updateConfigurationLocked(globalConfig, null, false);
        }

        // Notify ActivityMetricsLogger that the activity has launched. ActivityMetricsLogger
        // will then wait for the windows to be drawn and populate WaitResult.
        // ... 省略

        return res;
    }
}
```
这个方法很长，做了一些特殊情况的处理，最终是调用到了 startActivity 方法。
接下来是一系列检查，例如能否切换app等等，然后走到了startActivityUnchecked方法。
### ActivityStarter.startActivityUnchecked
``` java
// Note: This method should only be called from {@link startActivity}.
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor, restrictedBgActivity);

    final int preferredWindowingMode = mLaunchParams.mWindowingMode;

    computeLaunchingTaskFlags();

    computeSourceStack();

    mIntent.setFlags(mLaunchFlags);

    ActivityRecord reusedActivity = getReusableIntentActivity();

    // ... Reused 逻辑，也就是说启动了一个已存在的Activity，这部分先不考虑

    // 错误处理...

    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    // 刚好启动了当前Activity，省略

    boolean newTask = false;
    final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
            ? mSourceRecord.getTaskRecord() : null;

    // Should this be considered a new task?
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(taskToAffiliate);
    } else if (mSourceRecord != null) {
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        result = setTaskFromInTask();
    } else {
        // This not being started from an existing activity, and not part of a new task...
        // just put it in the top task, though these days this case should never happen.
        result = setTaskToCurrentTopOrCreateNewTask();
    }
    if (result != START_SUCCESS) {
        return result;
    }

    // ...
    if (newTask) {
        EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, mStartActivity.mUserId,
                mStartActivity.getTaskRecord().taskId);
    }
    ActivityStack.logStartActivity(
            EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTaskRecord());
    mTargetStack.mLastPausedActivity = null;

    mRootActivityContainer.sendPowerHintForLaunchStartIfNeeded(
            false /* forceSend */, mStartActivity);

    // 通知Stack
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
            mOptions);
    // Resume 逻辑
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTaskRecord().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            // If the activity is not focusable, we can't resume it, but still would like to
            // make sure it becomes visible as it starts (this will also trigger entry
            // animation). An example of this are PIP activities.
            // Also, we don't want to resume activities in a task that currently has an overlay
            // as the starting activity just needs to be in the visible paused state until the
            // over is removed.
            mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
            // Go ahead and tell window manager to execute app transition for this activity
            // since the app transition will not be triggered through the resume channel.
            mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
        } else {
            // If the target stack was not previously focusable (previous top running activity
            // on that stack was not visible) then any prior calls to move the stack to the
            // will not update the focused stack.  If starting the new activity now allows the
            // task stack to be focusable, then ensure that we now update the focused stack
            // accordingly.
            if (mTargetStack.isFocusable()
                    && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            mRootActivityContainer.resumeFocusedStacksTopActivities(
                    mTargetStack, mStartActivity, mOptions);
        }
    } else if (mStartActivity != null) {
        mSupervisor.mRecentTasks.add(mStartActivity.getTaskRecord());
    }
    mRootActivityContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTaskRecord(),
            preferredWindowingMode, mPreferredDisplayId, mTargetStack);

    return START_SUCCESS;
}
```

这个方法主要就是处理Activity重用的逻辑，以及确定要启动Activity的Task，启动模式相关的问题都可以在这里找到答案，此处我们先略过。
看代码，一开始以为下一步是在 mTargetStack.startActivityLocked 里面，查看了代码之后，发现并不是这样，于是往下看，跟踪mRootActivityContainer.resumeFocusedStacksTopActivities 方法，发现调用路径如下：
mRootActivityContainer.resumeFocusedStacksTopActivities -> ActivityStack.resumeTopActivityUncheckedLocked -> ActivityStack.resumeTopActivityInnerLocked
### ActivityStack.resumeTopActivityInnerLocked
这部分只贴最后关键的代码：
```
if (next.attachedToProcess()) {
    // 省略，我们跟踪的是一个全新的App启动，所以此时进程还没起来
} else {
    // Whoops, need to restart this activity!
    if (!next.hasBeenLaunched) {
        next.hasBeenLaunched = true;
    } else {
        if (SHOW_APP_STARTING_PREVIEW) {
            next.showStartingWindow(null /* prev */, false /* newTask */,
                    false /* taskSwich */);
        }
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
    }
    if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
    mStackSupervisor.startSpecificActivityLocked(next, true, true);
}
```

可以发现，走到了StackSupervisor.startSpecificActivityLocked方法。
### StackSupervisor.startSpecificActivityLocked
``` java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    // 1. 如果有进程了，则走到realStartActivityLocked方法
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    // Suppress transition until the new activity becomes ready, otherwise the keyguard can
    // appear for a short amount of time before the new process with the new activity had the
    // ability to set its showWhenLocked flags.
    if (getKeyguardController().isKeyguardLocked()) {
        r.notifyUnknownVisibilityLaunched();
    }

    try {
        if (Trace.isTagEnabled(TRACE_TAG_ACTIVITY_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "dispatchingStartProcess:"
                    + r.processName);
        }
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        // 没有进程，启动进程
        final Message msg = PooledLambda.obtainMessage(
                ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
        mService.mH.sendMessage(msg);
    } finally {
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```
由于是启动一个未启动的App，我们分析一下没有进程的情况，可以发现本质上是走到了ActivityManagerInternal::startProcess方法。
那么这个ActivityManagerInternal又是何方神圣呢，我们发现，它是一个abstract的类，而子类则是ActivityManagerService.LocalService，因此我们转移到，而LocalService的startProcess方法，实际上是调用到了ActivityManagerService的startProcessLocked方法:
``` java
final ProcessRecord startProcessLocked(String processName,
        ApplicationInfo info, boolean knownToBeDead, int intentFlags,
        HostingRecord hostingRecord, boolean allowWhileBooting,
        boolean isolated, boolean keepIfLarge) {
    return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
            hostingRecord, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
            null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
            null /* crashHandler */);
}
```

由此可见，ActivityManagerInternal本质上是ActivityManagerService的一个封装，但是仅用于system_server进程内部使用，方便其他的系统服务调用AMS。

### 进程启动
顺藤摸瓜，ProcessList.startProcessLocked 最终的调用如下：
ProcessList.startProcessLocked -> ProcessList.startProcessLocked(重载方法) -> startProcessLocked(重载方法2，这里面进行了一系列启动前检查) ->startProcessLocked(重载方法3) ->ProcessList.startProcess
### ProcessList.startProcess
``` java
private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
        long startTime) {
    try {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                app.processName);
        checkSlow(startTime, "startProcess: asking zygote to start proc");
        final Process.ProcessStartResult startResult;
        if (hostingRecord.usesWebviewZygote()) {
            startResult = startWebView(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null, app.info.packageName,
                    new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        } else if (hostingRecord.usesAppZygote()) {
            final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);

            startResult = appZygote.getProcess().start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null, app.info.packageName,
                    /*useUsapPool=*/ false,
                    new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        } else {
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith, app.info.packageName,
                    new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        }
        checkSlow(startTime, "startProcess: returned from zygote!");
        return startResult;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```
这边其实是调用到了Process.start方法。
### Process.start
Process启动过程比较复杂，后续再分析，此处可以简单总结一下：
Process.start -> ZYGOTE_PROCESS.start -> ZygoteProcess.startViaZygote -> ZygoteProcess.zygoteSendArgsAndGetResult(此处发送启动命令到Zygote进程，通过LocalSocket方式实现进程通信)
Zygote进程接收命令后，通过fork操作，复制出一个新进程，然后进入新进程执行（fork操作调用一次，在新旧进程分别返回不同的resultCode，从而可以在新旧进程中执行不同的逻辑），这部分逻辑如下:
ZygoteServer.runSelectLoop -> ZygoteConnection.processOneCommand
### ZygoteConnection.processOneCommand
``` java
pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
            parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mTargetSdkVersion);

    try {
        if (pid == 0) {
            // in child
            zygoteServer.setForkChild();

            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;

            return handleChildProc(parsedArgs, descriptors, childPipeFd,
                    parsedArgs.mStartChildZygote);
        } else {
            // In the parent. A pid < 0 indicates a failure and will be handled in
            // handleParentProc.
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            handleParentProc(pid, descriptors, serverPipeFd);
            return null;
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
```
此处便是fork的逻辑，可以看到，返回值pid可能为0，也可能不为0，这就是fork执行一次，返回两次的奥秘，对于新fork出来的子进程，pid == 0，
于是便进入了 handleChildProc 方法。
在handleChildProc中，又调用了 ZygoteInit.childZygoteInit 方法，这里面就是去查找进程的main方法了。
``` java
static final Runnable childZygoteInit(
        int targetSdkVersion, String[] argv, ClassLoader classLoader) {
    RuntimeInit.Arguments args = new RuntimeInit.Arguments(argv);
    return RuntimeInit.findStaticMain(args.startClass, args.startArgs, classLoader);
}
```
我们知道，Android App的main方法，就是ActivityThread.main方法，于是来到了ActivityThread.main，main方法中，又调用了ActivityThread.attach方法，然后调用到了IActivityManager.attachApplication方法，这时候又回到了AMS中。
### ActivityManagerService.attachApplication
attachApplication 又会调用attachApplicationLocked，在attachApplicationLocked中，首先会调用ActivityThread.bindApplication（IPC，运行在app进程），然后调用ActivityTaskManagerService.attachApplication:
``` java
// See if the top visible activity is waiting to run in this process...
if (normalMode) {
    try {
        didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
    } catch (Exception e) {
        Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
        badApp = true;
    }
}
```
而ATM.attachApplication又会调用RootActivityContainer.attachApplication
### RootActivityContainer.attachApplication

``` java
boolean attachApplication(WindowProcessController app) throws RemoteException {
    final String processName = app.mName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        final ActivityDisplay display = mActivityDisplays.get(displayNdx);
        final ActivityStack stack = display.getFocusedStack();
        if (stack != null) {
            stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
            final ActivityRecord top = stack.topRunningActivityLocked();
            final int size = mTmpActivityList.size();
            for (int i = 0; i < size; i++) {
                final ActivityRecord activity = mTmpActivityList.get(i);
                if (activity.app == null && app.mUid == activity.info.applicationInfo.uid
                        && processName.equals(activity.processName)) {
                    try {
                        if (mStackSupervisor.realStartActivityLocked(activity, app,
                                top == activity /* andResume */, true /* checkConfig */)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Exception in new application when starting activity "
                                + top.intent.getComponent().flattenToShortString(), e);
                        throw e;
                    }
                }
            }
        }
    }
    if (!didSomething) {
        ensureActivitiesVisible(null, 0, false /* preserve_windows */);
    }
    return didSomething;
}
```

这个方法的核心逻辑就是把待启动的Activity取出来，调用StackSupervisor.realStartActivityLocked方法。
### StackSupervisor.realStartActivityLocked
核心逻辑如下：
``` java
// Create activity launch transaction.
final ClientTransaction clientTransaction = ClientTransaction.obtain(
        proc.getThread(), r.appToken);

final DisplayContent dc = r.getDisplay().mDisplayContent;
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
        System.identityHashCode(r), r.info,
        // TODO: Have this take the merged configuration instead of separate global
        // and override configs.
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
        r.icicle, r.persistentState, results, newIntents,
        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                r.assistToken));

// Set desired final state.
final ActivityLifecycleItem lifecycleItem;
if (andResume) {
    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
} else {
    lifecycleItem = PauseActivityItem.obtain();
}
clientTransaction.setLifecycleStateRequest(lifecycleItem);

// Schedule transaction.
mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```
就是通过ClientTransition的方式，调用client的方法（client其实就是对应app进程），我们发现，其实是创建了一个LaunchActivityItem
### LaunchActivityItem.execute
``` java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```
可以发现是调用了client.handleLaunchActivity方法。这个client为何物呢？这是一个ClientTransactionHandler，通过分析我们发现这个ClientTransactionHandler的实现类就是ActivityThread，也就是说，最终走到了ActivityThread.handleLaunchActivity，这也就是回到了app进程，进入执行阶段。
## 执行
### ActivityThread.handleLaunchActivity
``` java
/**
 * Extended implementation of activity launch. Used when server requests a launch or relaunch.
 */
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    // 省略

    // Initialize before creating the activity
    if (!ThreadedRenderer.sRendererDisabled
            && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
        HardwareRenderer.preload();
    }
    WindowManagerGlobal.initialize();

    // Hint the GraphicsEnvironment that an activity is launching on the process.
    GraphicsEnvironment.hintActivityLaunch();

    final Activity a = performLaunchActivity(r, customIntent);

    // 省略
}
```

可见走进了performLaunchActivity方法。
### ActivityThread.performLaunchActivity
``` java
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // 准备工作

    // 创建Activity
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        // ...
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        // ...

        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            // Attach，一些组件的赋值
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            // 调用Activity.onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
        }
        r.setState(ON_CREATE);

        // updatePendingActivityConfiguration() reads from mActivities to update
        // ActivityClientRecord which runs in a different thread. Protect modifications to
        // mActivities to avoid race.
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);
        }

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}
```

从注释 Core implementation of activity launch. 就可以看出来，这是Activity启动的核心逻辑。
最终，走到了mInstrumentation.callActivityOnCreate，顾名思义，这里面就会调用到Activity.onCreate方法，也就到了我们熟悉的Activity的第一个生命周期方法了。

## 总结
由以上分析可以看出，启动一个Activity（准确地说是冷启动），过程十分复杂，主要分为三个部分
1. 发起：这部分在调用者进程中执行，相对比较简单
2. 管理：这部分主要在system_server进程中执行，是最复杂的部分，包含了各种情况的处理
3. 执行：这部分在被调用者进程中执行

从以上分析我们也可以看出，Binder跨进程调用是一个非常优秀的机制，可以让我们像同一个进程一样调用其他进程的方法。

本文只是对Activity启动流程做了一个粗浅的分析，其中还有非常多的细节是被略过的，读者若想更深入地了解，建议亲自阅读源码，而这篇文章可以作为一个指路者，亲自阅读才能更深刻地理解其中的原理。

另外，由于本人才疏学浅，难免会有错误，希望读者给予指正，共同学习，共同进步。

参考文章：http://gityuan.com/2016/03/12/start-activity/
