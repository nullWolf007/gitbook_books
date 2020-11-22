[TOC]
- <!-- TOC -->
  - [ 根Activity的启动过程](#根activity的启动过程)
    - [ 一、简介](#一、简介)
      - [ 1.1 概述](#11-概述)
    - [ 二、Launcher进程请求AMS过程](#二、launcher进程请求ams过程)
      - [ 2.1 startActivitySafely](#21-startactivitysafely)
      - [ 2.2 startActivity](#22-startactivity)
      - [ 2.3 startActivityForResult](#23-startactivityforresult)
      - [ 2.4 execStartActivity](#24-execstartactivity)
      - [ 2.5 getService](#25-getservice)
      - [ 2.6 startActivity](#26-startactivity)
    - [ 三、 AMS到ApplicationThread的调用过程](#三、-ams到applicationthread的调用过程)
      - [ 3.1 startActivity](#31-startactivity)
      - [ 3.2 startActivityAsUser](#32-startactivityasuser)
      - [ 3.3 startActivityMayWait](#33-startactivitymaywait)
      - [ 3.4 startActivityLocked](#34-startactivitylocked)
      - [ 3.5 startActivity](#35-startactivity)
      - [ 3.6 startActivity](#36-startactivity)
      - [ 3.7 startActivityUnchecked](#37-startactivityunchecked)
      - [ 3.8 resumeFocusedStackTopActivityLocked](#38-resumefocusedstacktopactivitylocked)
      - [ 3.9 resumeTopActivityUncheckedLocked](#39-resumetopactivityuncheckedlocked)
      - [ 3.10 resumeTopActivityInnerLocked](#310-resumetopactivityinnerlocked)
      - [ 3.11 startSpecificActivityLocked](#311-startspecificactivitylocked)
      - [ 3.12 realStartActivityLocked](#312-realstartactivitylocked)
      - [ 3.13 scheduleLaunchActivity](#313-schedulelaunchactivity)
      - [ 3.14 进程关系](#314-进程关系)
    - [ 四、ActivityThread启动Activity的过程](#四、activitythread启动activity的过程)
      - [ 4.1 scheduleLaunchActivity](#41-schedulelaunchactivity)
      - [ 4.2 sendMessage](#42-sendmessage)
      - [ 4.3 H](#43-h)
      - [ 4.4 handleLaunchActivity](#44-handlelaunchactivity)
      - [ 4.5 performLaunchActivity](#45-performlaunchactivity)
      - [ 4.6 callActivityOnCreate](#46-callactivityoncreate)
      - [ 4.7 performCreate](#47-performcreate)
    - [ 五、总结](#五、总结)
      - [ 5.1 根Activity启动过程中涉及的进程之间的关系](#51-根activity启动过程中涉及的进程之间的关系)
  <!-- /TOC -->
## 根Activity的启动过程

### 一、简介

#### 1.1 概述

* 根Activity的启动过程一般情况下也可以理解成应用程序启动的过程或者App启动的过程。主要分成三部分
  * Launcher请求AMS过程
  * AMS到ApplicationThread的调用过程
  * ActivityThread启动Activity

### 二、Launcher进程请求AMS过程

![Launcher请求AMS过程](..\..\images\Android组件内核\四大组件\Activity\Launcher请求AMS过程.png)

#### 2.1 startActivitySafely

* 桌面点击应用程序的图标就会触发startActivitySafely方法
* packages\apps\Launcher3\src\com\android\launcher3\Launcher.java#startActivitySafely

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
	......
	//把根Activity的Flag设置为NEW_TASK，则会在新的任务栈中启动
	intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	if (v != null) {
    	intent.setSourceBounds(getViewBounds(v));
	}
    try {
    	if (Utilities.ATLEAST_MARSHMALLOW
        	&& (item instanceof ShortcutInfo)
            && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
            || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
			&& !((ShortcutInfo) item).isPromise()) {
            startShortcutIntentSafely(intent, optsBundle, item);
		} else if (user == null || user.equals(Process.myUserHandle())) {
            //去启动根Activity
			startActivity(intent, optsBundle);
		} else {
        	LauncherAppsCompat.getInstance(this).startActivityForProfile(
            	intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
		}
        return true;
	} catch......
	return false;
}
```

* 最终会调用startActivity(intent, optsBundle);去启动根Activity

#### 2.2 startActivity

* Activity.java#startActivity

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
	if (options != null) {
    	startActivityForResult(intent, -1, options);
	} else {
		startActivityForResult(intent, -1);
	}
}
```

* 最终会调用startActivityForResult方法

#### 2.3 startActivityForResult

* Activity.java#startActivityForResult

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
	@Nullable Bundle options) {
	if (mParent == null) {
		options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
        	mInstrumentation.execStartActivity(
            	this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
		......
	}else{
		.....
	}
}
```

* mParent是Activity类型的，表示当前Activity的父亲，所以这里肯定是null，因为连根Activity都没有。
* 最终会调用execStartActivity方法

#### 2.4 execStartActivity

* Instrumentation.java#execStartActivity

```java
public ActivityResult execStartActivity(
	Context who, IBinder contextThread, IBinder token, Activity target,
    Intent intent, int requestCode, Bundle options) {
	......
    try {
    	intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        //首先调用ActivityManager.getService()来获取AMS的代理对象，
        //然后调用AMS的代理对象的startActivity方法
        int result = ActivityManager.getService()
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

#### 2.5 getService

* ActivityManager.java#getService

```java
public static IActivityManager getService() {
	return IActivityManagerSingleton.get();
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
	new Singleton<IActivityManager>() {
    	@Override
        protected IActivityManager create() {
        	final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
            final IActivityManager am = IActivityManager.Stub.asInterface(b);
            return am;
		}
	};
```

* getService方法中调用了IActivityManagerSingleton.get()方法
* IActivityManagerSingleton是一个单例模式
* 这里使用了Binder机制，通过ServiceManager.getService(Context.ACTIVITY_SERVICE)拿到了IBinder类型的AMS的引用，然后通过IBinder并调用IActivityManager.Stub.asInterface把AMS的Ibinder对象转换成了当前进程可以使用的对象。ActivityManagerService实现了IActivityManager.Stub的抽象方法，所以此次跨进程调用AMS(SystemServer进程)其实就是服务端进程，Launcher进程就是客户端进程，跨进程调用AMS的startActivity方法。

```java
public class ActivityManagerService extends IActivityManager.Stub
```

#### 2.6 startActivity

* ActivityManagerService.java#startActivity
* 通过Binder跨进程通信的，先拿到AMS的Ibinder对象，然后转换成本进程可以使用的对象，然后跨进程调用了AMS的startActivity方法

### 三、 AMS到ApplicationThread的调用过程

![AMS到ApplicationThread的调用过程](..\..\images\Android组件内核\四大组件\Activity\AMS到ApplicationThread的调用过程.png)

#### 3.1 startActivity

* ActivityManagerService.java#startActivity

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
								Intent intent, String resolvedType, IBinder resultTo, 									String resultWho, int requestCode,int startFlags, 										ProfilerInfo profilerInfo, Bundle bOptions) {
	return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
    	resultWho, requestCode, startFlags, profilerInfo, bOptions,
        UserHandle.getCallingUserId());
}
```

* 调用了startActivityAsUser方法，多了一个参数UserHandle.getCallingUserId()来获取调用者的UserId。

#### 3.2 startActivityAsUser

* ActivityManagerService.java#startActivityAsUser

```java
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
		Intent intent, String resolvedType, IBinder resultTo, String resultWho,
		int requestCode,int startFlags, ProfilerInfo profilerInfo, Bundle bOptions,
		int userId) {
    //判断调用者进程是否被隔离 被隔离抛出SecurityException
	enforceNotIsolatedCaller("startActivity");
    //检查调用者权限 没有权限会抛出SecurityException
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), 						Binder.getCallingUid(),userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
    	resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
        profilerInfo, null, null, bOptions, false, userId, null, null,
        "startActivityAsUser");
}
```

* 最后会调用ActivityStarter的startActivityMayWait方法

#### 3.3 startActivityMayWait

* ActivityStarter.java#startActivityMayWait

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
	String callingPackage, Intent intent, String resolvedType,
    IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
    IBinder resultTo, String resultWho, int requestCode, int startFlags,
    ProfilerInfo profilerInfo, WaitResult outResult,
    Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity,
	int userId , IActivityContainer iContainer, TaskRecord inTask, String reason) {
    ...... 
	int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
				aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                inTask, reason);
	......
	return res;
}
```

* ActivityStarter是加载Activity的控制类，会收集所有的逻辑来决定如何将Intent和Flags转换为Activity，并将Activity和Task以及Stack相关联
* 最终调用了startActivityLocked方法

#### 3.4 startActivityLocked

* ActivityStarter.java#startActivityLocked

```java
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
		String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, 
		int callingUid,String callingPackage, int realCallingPid, int realCallingUid,
		int startFlags,ActivityOptions options, boolean ignoreTargetSecurity,
		boolean componentSpecified,ActivityRecord[] outActivity, 		
		ActivityStackSupervisor.ActivityContainer container,TaskRecord inTask, 
		String reason) {
	//判断启动的理由不为空
    if (TextUtils.isEmpty(reason)) {
    	throw new IllegalArgumentException("Need to specify a reason.");
	}
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, 
		resolvedType,aInfo, rInfo, voiceSession, voiceInteractor, resultTo,
		resultWho, requestCode,callingPid, callingUid, callingPackage, 
		realCallingPid, realCallingUid, startFlags,options, 
		ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
        container, inTask);
	if (outActivity != null) {
    	// mLastStartActivityRecord[0] is set in the call to startActivity above.
	    outActivity[0] = mLastStartActivityRecord[0];
	}
	return mLastStartActivityResult;
}   
```

* 最后又调用了startActivity方法

#### 3.5 startActivity

* ActivityStarter.java#startActivity

```java
private int startActivity(IApplicationThread caller, Intent intent, 
		Intent ephemeralIntent,String resolvedType, ActivityInfo aInfo, 
		ResolveInfo rInfo,IVoiceInteractionSession voiceSession,
		IVoiceInteractor voiceInteractor,IBinder resultTo, String resultWho, 
		int requestCode, int callingPid, int callingUid,String callingPackage,
		int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity,
		boolean componentSpecified,ActivityRecord[] outActivity, 
		ActivityStackSupervisor.ActivityContainer container,TaskRecord inTask) {
	int err = ActivityManager.START_SUCCESS;
    final Bundle verificationBundle
    	= options != null ? options.popAppVerificationBundle() : null;
	ProcessRecord callerApp = null;
    //IApplicationThread对象 也就是应用程序进程的ApplicationThread对象
    //使用的Ibinder机制
	if (caller != null) {
        //getRecordForAppLocked获取代表Launcher进程的callApp对象
        //callApp是ProcessRecord类型 描述了一个应用程序进程信息
    	callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            //获取Launcher进程的pid和uid
        	callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
		} else {
        	Slog.w(TAG, "Unable to find app for caller " + caller
					+ " (pid=" + callingPid + ") when starting: "
                    + intent.toString());
			err = ActivityManager.START_PERMISSION_DENIED;
		}
	}

	......
	//创建即将要启动的Activity的描述类ActivityRecord
	//ActivityRecord记录了一个Activity的所有信息
	ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
    	callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
	    resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
        mSupervisor, container, options, sourceRecord);
	if (outActivity != null) {
        //把值赋给outActivity[0] 接下来作为参数传递下去
    	outActivity[0] = r;
	}
    ......
	doPendingActivityLaunchesLocked(false);

	return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, 
			true,options, inTask, outActivity);
    }
```

* 最后会调用startActivity方法

#### 3.6 startActivity

* ActivityStarter.java#startActivity

```java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
		IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
	int result = START_CANCELED;
    try {
    	mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
        		startFlags, doResume, options, inTask, outActivity);
	}......
	return result;
}
```

* 紧接着调用startActivityUnchecked方法

#### 3.7 startActivityUnchecked

* ActivityStarter.java#startActivityUnchecked

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
		IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
	......
	if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
    		&& (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
		newTask = true;
        //创建一个新的Activity任务栈
        result = setTaskFromReuseOrCreateNewTask(
        			taskToAffiliate, preferredLaunchStackId, topStack);
	} else if (mSourceRecord != null) {
    	result = setTaskFromSourceRecord();
	} else if (mInTask != null) {
    	result = setTaskFromInTask();
	} else {
		setTaskToCurrentTopOrCreateNewTask();
	}
	......
	if (mDoResume) {
    	final ActivityRecord topTaskActivity =
        		mStartActivity.getTask().topRunningActivityLocked();
		if (!mTargetStack.isFocusable()
        		|| (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
			......
		} else {
			if (mTargetStack.isFocusable() 
                	&& !mSupervisor.isFocusedStack(mTargetStack)) {
				mTargetStack.moveToFront("startActivityUnchecked");
			}
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
            		mOptions);
		}
	} else {
    	mTargetStack.addRecentActivityLocked(mStartActivity);
	}
    ......
}
```

* startActivityUnchecked方法主要处理与栈管理相关的逻辑
* 最终会调用ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法

#### 3.8 resumeFocusedStackTopActivityLocked

* ActivityStackSupervisor.java#resumeFocusedStackTopActivityLocked

```java
boolean resumeFocusedStackTopActivityLocked(
		ActivityStack targetStack, ActivityRecord target, 
    	ActivityOptions targetOptions) {
	if (targetStack != null && isFocusedStack(targetStack)) {
    	return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
	}
    //获取要启动的Activity所在栈的栈顶的不是出于停止状态的ActivityRecord
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
	if (r == null || r.state != RESUMED) {
    	mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
	} else if (r.state == RESUMED) {
		mFocusedStack.executeAppTransition(targetOptions);
	}
    return false;
}
```

* 最后会调用ActivityStack的resumeTopActivityUncheckedLocked

#### 3.9 resumeTopActivityUncheckedLocked

* ActivityStack.java#resumeTopActivityUncheckedLocked

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
	if (mStackSupervisor.inResumeTopActivity) {
    	return false;
	}
	boolean result = false;
	try {
		mStackSupervisor.inResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);
	} finally {
    	mStackSupervisor.inResumeTopActivity = false;
	}
	mStackSupervisor.checkReadyForSleepLocked();
    return result;
}
```

* 最终会调用resumeTopActivityInnerLocked方法

#### 3.10 resumeTopActivityInnerLocked

* ActivityStack.java#resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {        
    ......
    	mStackSupervisor.startSpecificActivityLocked(next, true, true);
	}

    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```

* 最终会调用ActivityStackSupervisor的startSpecificActivityLocked方法

#### 3.11 startSpecificActivityLocked

* ActivityStackSupervisor.java#startSpecificActivityLocked

```java
void startSpecificActivityLocked(ActivityRecord r,
		boolean andResume, boolean checkConfig) {
    //获取即将启动的Activity所在的应用程序进程
	ProcessRecord app = mService.getProcessRecordLocked(r.processName,
    		r.info.applicationInfo.uid, true);
	r.getStack().setLaunchTime(r);
    
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
	mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
    		"activity", r.intent.getComponent(), false, false, true);
}
```

* 最终会调用realStartActivityLocked方法

#### 3.12 realStartActivityLocked

* ActivityStackSupervisor.java#realStartActivityLocked

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
		boolean andResume, boolean checkConfig) throws RemoteException {
    ......
	app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
    		System.identityHashCode(r), r.info,
			mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, !andResume,
            mService.isNextTransitionForward(), profilerInfo);
    ......
    retrun true;
}
```

* app.thread指的是IApplicationThread，是一个本进程用来操作IApplicationThread其实现类的方法。IApplicationThread的实现类是ActivityThread的内部类ApplicationThread。所以通过Binder机制，AMS所在的SystemServer进程拿到了应用程序进程的ApplicationThread的代理对象(类似AIDL中的代理对象，可以通过操作对象的方法操作ApplicationThread的方法)
* 跨进程通信：AMS所在的SystemServer进程，通过Binder机制，跨进程调用应用程序进程的ApplicationThread的scheduleLaunchActivity方法

```java
private class ApplicationThread extends IApplicationThread.Stub 
```

#### 3.13 scheduleLaunchActivity

* ActivityThread.java#ApplicationThread#scheduleLaunchActivity

#### 3.14 进程关系

 ![AMS与应用程序进程通信](..\..\images\Android组件内核\四大组件\Activity\AMS与应用程序进程通信.png)

* SystemServer进程和应用程序进程通过Binder机制进行通信，ApplicationThread就是那个服务器端具体的功能实现类。

### 四、ActivityThread启动Activity的过程

![ActivityThread启动Activity过程](..\..\images\Android组件内核\四大组件\Activity\ActivityThread启动Activity过程.png)

#### 4.1 scheduleLaunchActivity

* ActivityThread.java#ApplicationThread#scheduleLaunchActivity

```java
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
		ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
	updateProcessState(procState, false);
	ActivityClientRecord r = new ActivityClientRecord();
	r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
	......
	updatePendingConfiguration(curConfig);
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

* 将Activity参数封装成ActivityClientRecord
* sendMessage方法向H类发送LAUNCH_ACTIVITY的消息，并将ActivityClientRecord对象传递过去
* sendMessage有很多承载方法，最终会调用到如下sendMessage方法

#### 4.2 sendMessage

* ActivityThread.java#sendMessage

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
	if (DEBUG_MESSAGES) Slog.v(
    	TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
	Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
    	msg.setAsynchronous(true);
	}
    mH.sendMessage(msg);
}
```

* 最终调用mH.sendMessage(msg);
* mH是H类对象，它是ActivityThread的内部类继承自Handler，是应用程序进程中主线程的消息管理类。
* 因为ApplicationThread是一个Binder，所以调用逻辑运行在Binder线程中，需要切换到主线程中，所以使用H把代码切换到主线程中

#### 4.3 H

* ActivityThread.java#H

```java
private class H extends Handler {
	public static final int LAUNCH_ACTIVITY         = 100;
    ......
	public void handleMessage(Message msg) {
    	if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
        	case LAUNCH_ACTIVITY: {
            	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                //把msg.obj转换成ActivityClientRecord对象
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                //getPackageInfoNoCheck获得LoadedApk类型的对象
                //把此对象赋值给ActivityClientRecord的成员变量packageInfo
                r.packageInfo = getPackageInfoNoCheck(
                		r.activityInfo.applicationInfo, r.compatInfo);
				handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
			} break;
			case......
        }
    }
    ......
}
    
```

* Handler和Message和Looper消息机制，所以会进入H的handleMessage方法中，并且msg.what的值为LAUNCH_ACTIVITY。
* 应用程序进程启动Activity时需要将该Activity所属的APK加载进来，而LoadApk就是用来描述已加载的APK文件的
* 最终调用handleLaunchActivity方法

#### 4.4 handleLaunchActivity

* ActivityThread.java#handleLaunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ......
	WindowManagerGlobal.initialize();
    //启动Activity
	Activity a = performLaunchActivity(r, customIntent);
	if (a != null) {
    	r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        //将Activity的状态置为resume
        handleResumeActivity(r.token, false, r.isForward,
        		!r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
		if (!r.activity.mFinished && r.startsNotResumed) {
			performPauseActivityIfNeeded(r, reason);
			if (r.isPreHoneycomb()) {
            	r.state = oldState;
			}
		}
	} else {
		try {
            //停止Activity启动
        	ActivityManager.getService()
            	.finishActivity(r.token, Activity.RESULT_CANCELED, null,
                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
		} catch (RemoteException ex) {
        	throw ex.rethrowFromSystemServer();
		}
	}
}
```

* 调用performLaunchActivity来启动Activity

#### 4.5 performLaunchActivity

* ActivityThread.java#performLaunchActivity

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //获取ActivityInfo类
	ActivityInfo aInfo = r.activityInfo;
	if (r.packageInfo == null) {
        //获取APK文件的描述类LoadedApk
    	r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
        		Context.CONTEXT_INCLUDE_CODE);
	}
	ComponentName component = r.intent.getComponent();
	......
	//创建要启动Activity的上下文环境
	ContextImpl appContext = createBaseContextForActivity(r);
	Activity activity = null;
    try {
    	java.lang.ClassLoader cl = appContext.getClassLoader();
        //用类加载器来创建该Activity的实例
        activity = mInstrumentation.newActivity(
        		cl, component.getClassName(), r.intent);
		......
	} catch (Exception e) {
        ......
    }

	try {
        //创建Application
    	Application app = r.packageInfo.makeApplication(false, mInstrumentation);
		......
		if (activity != null) {
        	......
			//初始化Activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
            		r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
			......

			if (r.isPersistable()) {
            	mInstrumentation.
                    callActivityOnCreate(activity, r.state, r.persistentState);
			} else {
            	mInstrumentation.callActivityOnCreate(activity, r.state);
			}
            ......
        }    
		r.paused = true;
		mActivities.put(r.token, r);
	} catch (SuperNotCalledException e) {
    	throw e;
	} catch (Exception e) {
        ......
	}
    return activity;
}
```

* makeApplication方法内部会调用Application的onCreate方法
* attach方法初始化Activity，在attach方法中会创建Window对象（PhoneWindow）并于Activity自身关联
* 最后使用callActivityOnCreate方法来启动Activity

#### 4.6 callActivityOnCreate

* Instrumentation.java#callActivityOnCreate

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
		PersistableBundle persistentState) {
	prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

* 调用Activity的performCreate方法

#### 4.7 performCreate

* Activity.java#performCreate

```java
final void performCreate(Bundle icicle) {
	restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```

* 最终会调用Activity的onCreate方法，到此根Activity就启动起来了，程序也就运行起来了。

### 五、总结

#### 5.1 根Activity启动过程中涉及的进程之间的关系

![根Activity启动过程中涉及的进程之间的关系](..\..\images\Android组件内核\四大组件\Activity\根Activity启动过程中涉及的进程之间的关系.png)

