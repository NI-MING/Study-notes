# Activity

## 1.什么是Activity

Activity是Android四大组件之一，是负责展示界面、与用户交互的一种组件，Activity是四大组件中唯一用户可以感知的组件。

## 2.进程优先级

Android平台的App，通常都是单进程的，根据App栈顶的活动状态，可将App分为以下五大进程模式：

* 前台进程：当前进程Activity正在与用户交互；当前Service正在与Activity进行交互，或者调用了`startForground()`或者此Service正在执行生命周期。
* 可见进程：此Activity失去焦点，但是处于可见状态；此Service与一个可见的Activity进行绑定。
* 服用进程：当前开启`startService()`启动一个Service服务就可以认为此进程是一个服务进程。
* 后台进程：Activity的onStop()被调用，但是onDestroy()没有调用的状态。该进程属于后台进程。
* 空进程：该进程没有任何数据，但是保留了内存空间，并没有被系统kill掉。

进程的优先级就是根据这五大进程模式逐一递减。当系统内存资源不足时，会根据此优先级进行回收操作。回收针对的是App，而不是App中的某个组件。

## 3.Activity的启动模式

在了解启动模式之前，我们有必要对Activity的两个概念拎清楚，稍后对启动模式的分析也会基于这两个概念：任务、返回栈。

这两个概念我在刚接触Android开发时接触过，当时看郭神那本第一行代码在讲解启动模式时提到过返回栈的概念，书中只是粗浅的介绍了一个Activity的启动（standard），伴随在入栈操作，这个栈就是返回栈。现在看来，介绍的不是很完整，当然对于当初学Android的我来说，介绍的很浅显易懂了。接下来就谈一谈对其的补充。

当我们按下手机中查看最近运行应用时，其实看到的是一个一个的任务（Task），任务是用户在执行某项工作时与之互动的一系列 Activity 的集合。这些 Activity 按照每个 Activity 打开的顺序排列在一个返回堆栈中（Google官方介绍）。每个任务都对应这一个返回堆栈，而任务是可以叠加的，这个叠加只针对于前台App，当App进入后台时，或者当按下查看最近任务列表键，前台App中Task的叠加就会被拆开。（稍后介绍如何叠加）。

Activity总共有四大启动模式：

* 标准模式（standard）
* 栈顶复用（singleTop）
* 栈内复用（singleTask）
* 单例模式（singleInstance）

前两种模式是最好理解的，网上许多博客也介绍的相当清楚：

### 标准模式

是Activity默认的启动模式，每当启动一个Activity时，就会在返回栈栈顶创建一个Activity的实例，不论这个Activity在返回栈中是否有实例。我们在开发中大多都用这种模式。

### 栈顶复用

如果这个Activity已经存在于返回栈的栈顶，那么当重新打开这个Activity时，并不会重新创建它的实例，而是去回调onNewIntent方法，接着执行Activity的onRestart->onStart->onResume。

这两种启动模式都不涉及两个App之间调用对方Activity的情况。即使打开了另一个App的Activity，也还是在当前App的Task里，返回栈还是当前Task的返回栈。如果多个App同时打开了一个App中的Activity，他们是不会相互影响的。

接下来介绍的两种启动模式，会更加有针对性的去考虑在一个App里打开另一个App中的Activity的情况。

### 栈内复用

在郭神的第一行代码中，是这样介绍这种启动模式的：每次启动Activity时，都会在返回栈内检查是否有该Activity的实例，如果发现有则直接使用该实例，并把在这个Activity之上的所有Activity统统出栈。如果没有，就会新创建一个Activity。

在我们的测试中发现情况确实如此，但是这个描述是不太准确的。我们看singleTask这个名字，一定是与Task有一定关系的。我们考虑一种情况。在App-B中有一个ActivityB，它的启动模式是singleTask，那么，我们在App-A中启动这个ActivityB，这个ActivityB在哪里创建？在App-A上面吗？我们在App-A上面确实看到了这个ActivityB。但是，如果我们按下回退键，如果App-B之前被你启动过，里面还有其他Activity，我们会看到回退到了App-B的Activity上。那......这个ActivityB是在App-B上？事实确实如此，当我们从App-A启动App-B的ActivityB时（前提是singleTask模式），ActivityB会在App-B的任务中入栈，然后将App-B的任务压在App-A的上面，这样，给我们造成的体验就是App-A上面打开了App-B的ActivityB。在这种情况下，一旦我们按home键返回桌面或者查看最近任务列表，两个压在一起的Task就会被拆开，这时再回到App-A就不会看到ActivityB了。（当作者使用微信向QQ转发文件时，发现QQ选择联系人页面使用的就是这个启动逻辑，有兴趣的小伙伴可以试一试）。

那么问题来了，为什么App-B的ActivityB会在App-B上启动，我们明明是在App-A里启动Activity的呀？这是因为每个Activity都会有一个taskAffinity属性，这个属性可以在AndroidManifest.xml中设置（设置的字符串中必须加`.`,否则会报错），这个属性时Task相似、相近的意思，taskAffinity属性相同的Activity会归属同一个Task栈。（如过没设置这个属性，Activity的taskAffinity默认取自Application的taskAffinity，而Application的taskAffinity默认取自包名，而Task的taskAffinity的值取自Task栈底Activity的taskAffinity）。如果启动模式是standard或singleTop，会忽略taskAffinity属性，而singleTask或singleInstance则会检查这个属性，把启动的Activity归属于属于它的Task中。这样，我们遇到的问题就迎刃而解了。

同时这个启动模式有ClearTop属性，它会清空此Activity上的所有Activity，以便此Activity的复用。

### 单例模式

单例模式和栈内复用的工作机制是相同的，但是它比栈内复用做的更绝，它不仅要求该Task中仅有此Activity的一个实例，还要此Activity独占一个Task。那么这种启动模式就具有很强的跨App交互意图。

总结：在同一个App中，往往使用前两种启动模式，当你得App想要对外提供服务，可以考虑单例模式，而栈内复用是兼容派，既可以在一个App中使用，也可以跨App启动。总之，根据具体场景具体分析。

启动模式的设置方式

```xml
<activity    
	android:name=".MainActivity"    
	android:launchMode="standard" /> 
```

## 4.Activity生命周期

![Untitled](C:\Users\wlk\Desktop\Untitled.png)

针对Activity的生命周期，我们可以分两类进行讨论：一种是正常情况下的生命周期，一种是异常情况下的生命周期。

### 正常情况下的生命周期：

分析生命周期，先要了解Activity的生命周期函数：

* onCreate：表示Activity正在被创建，这时可以进行一些初始化工作，如我们的界面布局的加载，控件的绑定。`onCreate`方法不是一个常驻方法，完成工作后就会进入`onStart`方法。
* onRestart：Activity由onStop重新变为可见状态时调用。
* onStart：在Activity由不可见变为即将可见时调用。
* onResume：在Activity准备好与用户进行交互时调用。此时的Activity一定位于返回栈的栈顶，并且处于运行状态。
* onPause：当系统准备去启动或者恢复另一个Activity时调用，在这个方法中将一些消耗CPU的资源都释放掉，以及保存一些关键的数据，这个方法执行的速度一定要快，不然会影响栈顶活动的使用。
* onStop：在Activity完全不可见时调用。
* onDestroy：在Activity被销毁前调用，之后活动的状态变为销毁状态。

#### 情景分析：

Activity A 是 singleTask 模式，Activity B 启动模式为常规模式时 ， 多种不同情况下执行方法：

1. Activity A 跳转到 Activity B：

   ```java
   A.onCreat() -> A.onStart() -> A.onResume() -> A.onPause() -> B.onCreat() -> B.onStart() -> B.onResume() -> A.onStop()
   ```

2. Activity B 跳转到 Activity A：

   ```java
   B.onPause() ->  A.onRestart() -> A.onStart() -> A.onResume() -> B.onStop()
   ```

3. Activity B 点 击 back 键 ：

   ```java
   B.onPause() -> A.onRestart() -> A.onStart() -> A.onResume() ->B.onStop() -> B.onDestroy()
   ```

4. Activity B 按 home 键，然后打开应用：

   ```java
   B.onPause() -> B.onStop() -> B.onRestart() ->B.onStart() -> B.onResume()
   ```

5. Activity A 启动一个完全透明的 Activity B 时会执行：

   ```java
   A.onCreat() -> A.onStart() -> A.onResume()-> A.onPause() -> B.onCreat() -> B.onStart() -> B.onResume()
   ```

Activity A 是常规模式，Activity B 启动模式为常规模式时，多种不同情况下执行方法：

1. Activity A 跳转到 Activity B：

   ```java
   A.onCreat() -> A.onStart() -> A.onResume() -> A.onPause() ->B.onCreat() -> B.onStart() -> B.onResume() -> A.onStop()
   ```

2. Activity B 跳转到 Activity A：

   ```java
   B.onPause() -> A.onCreat() -> A.onStart() -> A.onResume() -> B.onStop()
   ```

3. Activity B 点 击 back 键 ：

   ```java
   B.onPause() -> A.onRestart() -> A.onStart() -> A.onResume() -> B.onStop() -> B.onDestroy()
   ```

4. Activity B 按 home 键，然后打开应用：

   ```java
   B.onPause() -> B.onStop() -> B.onRestart() -> B.onStart() -> B.onResume()
   ```

### 异常情况下的生命周期

#### 为什么会有异常情况？

当系统资源回收或者配置发生改变时，就会导致Activity重建。系统资源回收通常发生在内存不足，配置发生变化通常为屏幕方向发生变化、系统语言更改等情况。

#### 为什么要重建？

对于资源回收的情况，保存状态到使用时再恢复，远比后台保留进程所占的资源小的多。

对于配置情况的改变，只有重建，才能加载不同的布局。

#### 重建机制：onSaveInstanceState & onRestoreInstanceState

重建机制就是当Activity因为异常情况需要重建时，通过onSaveInstanceState方法保留活动当前的状态，如：EditText文本内容、CheckBox选项框状态等，当重建Activity时，通过onRestoreInstanceState方法恢复其状态。

onSaveInstanceState方法在onStop方法之前被调用，但是和onPause方法没有时序关系。

onRestoreInstanceState方法执行在onStart之后。

#### 重建流程

Activity 在 onSaveInstanceState 时，会 自动收集 View Hierachy 中每一个 “实现了状态保存和恢复方法” 的 View 的状态，这些状态数据会在 onRestoreInstanceState 时回传给每一个 View。并且回传时是依据 View 的 id 来逐一匹配的。

#### 状态保存和恢复的注意事项

1. 为了成功保存状态，要求在view内部实现保存和恢复方法

   原生的 View 都有做到；如果是自定义 View，务必记住这一点；如果第三方 View 没做到，可以通过继承其来实现保存和恢复方法。

2. 为了成功的恢复状态，需要给View赋予对应的ID

3. 如果需要保存Activity的成员变量，需要手动重写onSaveInstanceState 和 onRestoreInstanceState 方法。

   重写仅仅是为了额外地保存成员变量。重写方法时，记得要保留基类（super）的实现，Activity 正是在基类中完成 View 状态的保存和恢复。

## 4.Activity启动流程

![Activity启动流程](C:\Users\wlk\Desktop\Activity启动流程.png)

分析Activity的启动流程，我们先看如何启动一个Activity

```java
Intent intent = new Intent(ActivityA.this,ActivityB.class);
startActivity(intent);
```

那我们就进入startActivity这个方法一探究竟。

通过对源码的阅读，我发现通过方法调用链最终调用startActivityForResult这个方法：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //重点
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
                mStartedActivity = true;
            }
            cancelInputsAndStartExitTransition(options);
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

核心逻辑就是调用Instrumentation.execStartActivity：

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ......
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            //重点
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

看这个方法名，猜测是得到ActivityTaskManagerService，然后调用其的startActivity方法：

注：网上大部分博客讲解Activity启动流程中这里介绍为AMS，这是Android10之前的写法，从Android10，将Activity启动服务的逻辑转移到ActivityTaskManagerService中，没有本质差别。

```Java
public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```

startActivityAsUser：

```java
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    assertPackageMatchesCallingUid(callingPackage);
    enforceNotIsolatedCaller("startActivityAsUser");

    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();

}
```

这里通过工厂模式创建ActivityStarter实例。

```java
ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
```

通过obtain进行内存复用，避免频繁发创建ActivityStarter对象，导致内存不足触发GC造成的内存抖动。

ActivityStarter.execute:

```java
int execute() {
        try {
            if (mRequest.intent != null && mRequest.intent.hasFileDescriptors()) {
                throw new IllegalArgumentException("File descriptors passed in Intent");
            }

            final LaunchingState launchingState;
            synchronized (mService.mGlobalLock) {
                final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
                launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                        mRequest.intent, caller);
            }

            if (mRequest.activityInfo == null) {
                mRequest.resolveActivity(mSupervisor);
            }

            int res;
            synchronized (mService.mGlobalLock) {
                final boolean globalConfigWillChange = mRequest.globalConfig != null
                        && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
                final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();
                if (stack != null) {
                    stack.mConfigWillChange = globalConfigWillChange;
                }
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG_CONFIGURATION, "Starting activity when config will change = "
                            + globalConfigWillChange);
                }

                final long origId = Binder.clearCallingIdentity();

                res = resolveToHeavyWeightSwitcherIfNeeded();
                if (res != START_SUCCESS) {
                    return res;
                }
                //开始请求
                res = executeRequest(mRequest);

                Binder.restoreCallingIdentity(origId);

                if (globalConfigWillChange) {

                    mService.mAmInternal.enforceCallingPermission(
                            android.Manifest.permission.CHANGE_CONFIGURATION,
                            "updateConfiguration()");
                    if (stack != null) {
                        stack.mConfigWillChange = false;
                    }
                    if (DEBUG_CONFIGURATION) {
                        Slog.v(TAG_CONFIGURATION,
                                "Updating to new configuration after starting activity.");
                    }
                    mService.updateConfigurationLocked(mRequest.globalConfig, null, false);
                }

                mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(launchingState, res,
                        mLastStartActivityRecord);
                return getExternalResult(mRequest.waitResult == null ? res
                        : waitForResult(res, mLastStartActivityRecord));
            }
        } finally {
            onExecutionComplete();
        }
    }
```

executeRequest：

```java
private int executeRequest(Request request) {
    
    ......
	//每个Activity都对应一个ActivityRecord，存放于ActivityStack中
    final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, callingFeatureId, intent, resolvedType, aInfo,
            mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
            request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
            sourceRecord);
    mLastStartActivityRecord = r;

    if (r.appTimeTracker == null && sourceRecord != null) {
        r.appTimeTracker = sourceRecord.appTimeTracker;
    }

    final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();

    if (voiceSession == null && stack != null && (stack.getResumedActivity() == null
            || stack.getResumedActivity().info.applicationInfo.uid != realCallingUid)) {
        if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                realCallingPid, realCallingUid, "Activity start")) {
            if (!(restrictedBgActivity && handleBackgroundActivityAbort(r))) {
                mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                        sourceRecord, startFlags, stack, callerApp, intentGrants));
            }
            ActivityOptions.abort(checkedOptions);
            return ActivityManager.START_SWITCHES_CANCELED;
        }
    }

    mService.onStartActivitySetDidAppSwitch();
    mController.doPendingActivityLaunches(false);
	//重点
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);

    if (request.outActivity != null) {
        request.outActivity[0] = mLastStartActivityRecord;
    }

    return mLastStartActivityResult;
}
```

startActivityUnchecked：

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.deferWindowLayout();
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            //重点
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityStack = handleStartResult(r, result);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }
```

startActivityInner：

```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    //设置初始化信息
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor, restrictedBgActivity);
	//判断启动模式，并追加标记
    computeLaunchingTaskFlags();
	//获取到Activity的启动栈
    computeSourceStack();
	//根据上面的计算，设置应用识别到的flags
    mIntent.setFlags(mLaunchFlags);

    ......

    mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
            newTask, mKeepCurTransition, mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isTopActivityFocusable()
                || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                && mStartActivity != topTaskActivity)) {

            mTargetStack.ensureActivitiesVisible(null /* starting */,
                    0 /* configChanges */, !PRESERVE_WINDOWS);

            mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
        } else {

            if (mTargetStack.isTopActivityFocusable()
                    && !mRootWindowContainer.isTopDisplayFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityInner");
            }
            //重点
            mRootWindowContainer.resumeFocusedStacksTopActivities(
                    mTargetStack, mStartActivity, mOptions);
        }
    }
    mRootWindowContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

    mSupervisor.mRecentTasks.add(mStartActivity.getTask());
    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(),
            mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetStack);

    return START_SUCCESS;
}
```

`mRootWindowContainer.resumeFocusedStacksTopActivities` -> `focusedStack.resumeTopActivityUncheckedLocked` -> `resumeTopActivityInnerLocked` -> `mStackSupervisor.startSpecificActivity`:

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
   	//如果该进程存在则直接进入realStartActivityLocked
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        knownToBeDead = true;
    }

    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

    final boolean isTop = andResume && r.isTopRunningActivity();
    //不存在则去异步开启进程
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```

进入到这里，我们也看到了一行Google给的注释`Is this activity's application already running?`，我们假设已经有进程存在，直接进入`realStartActivityLocked`，进程不存在就不继续分析了，逻辑就是调用系统孵化进程在进入`realStartActivityLocked`方法。

realStartActivityLocked：

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
    	......
        try {
            ......
            try {
                ......
				//重点1
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);
				//重点2
                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
         		......
            } catch (RemoteException e) {
        } finally {
			......
        }
		......
        if (andResume && readyToResume()) {
            ......
        } else {
			......
        }
        return true;
    }
```

在重点1处，我们看到创建了ClientTransaction对象，在重点2处，调用了ClientLifecycleManagerscheduleTransaction()，通过给它传递参数，调用了ClientTransaction.schedule():

```java
public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
```

mClient.scheduleTransaction:

```java
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

ActivityThread继承自ClientTransactionHandler：

```java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

在ActivityThread的handleMessage中：

```java
case EXECUTE_TRANSACTION:
    final ClientTransaction transaction = (ClientTransaction) msg.obj;
    mTransactionExecutor.execute(transaction);
    if (isSystem()) {
        // Client transactions inside system process are recycled on the client side
        // instead of ClientLifecycleManager to avoid being cleared before this
        // message is handled.
        transaction.recycle();
    }
    // TODO(lifecycler): Recycle locally scheduled transactions.
    break;
```

mTransactionExecutor.execute：

```java
public void execute(ClientTransaction transaction) {
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

    ......

    if (DEBUG_RESOLVER) Slog.d(TAG, transactionToString(transaction, mTransactionHandler));
	//重点
    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
}
```

executeCallbacks:

```java
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();


    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
    final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
            : UNDEFINED;
    // Index of the last callback that requests some post-execution state.
    final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);

        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
    }
}
```

这个item是LaunchActivityItem：

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

我们看到熟悉的handleLaunchActivity，这个方法里就会调用到Activity的onCreate方法，Activity启动流程至此结束。

Activity启动流程的分析我参照了网上的博客，源码我是逐个逐个的翻了一边，大致的捋清了方法之间的调用，但是对于每个方法具体做了什么，由于本人水平有限，暂时没有搞明白，我相信，随着不断深入的学习，这些都终将不会是问题。