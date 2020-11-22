[TOC]
# ActivityThread解析

### 参考链接

* [ActivityThread的main方法究竟做了什么？](https://www.jianshu.com/p/0efc71f349c8)

### 一、ActivityThread关联图

![ActivityThread关联图](..\..\images\Android组件内核\FrameWork内核解析\ActivityThread关联图.png)

* 从图中我们可以看到四大组件中的mActivities，mServices，mProviderMap都保存在ArrayMap中。但是唯独BroadcastReceiver没有进行保存，这是因为BroadcastReceiver对象的生命周期很短暂，属于调用它，再创建运行，所以不需要保存。

* mInitialApplication就是Application对象，一个进程只能有一个Application，但是一个Application可以占用多个进程
* 变量mResourcesManage管理着应用中的资源

### 二、ActivityThread的main方法

#### 2.1源码

```java
public static void main(String[] args) {
	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    SamplingProfilerIntegration.start();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
	//初始化应用中需要使用的系统路径
    Environment.initForCurrentUser();

    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());

    //为应用 设置当前用户的CA证书保存的位置
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
	TrustedCertificateStore.setDefaultUserDirectory(configDir);
	//设置进程名称
    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();
	//创建ActivityThread对象
    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
    	sMainThreadHandler = thread.getHandler();
	}

    if (false) {
    	Looper.myLooper().setMessageLogging(new
        	LogPrinter(Log.DEBUG, "ActivityThread"));
	}

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

* 上述代码就是main方法的代码，我们从上面取出几行代码进行单独讲解

```java
	Looper.prepareMainLooper();
	//创建ActivityThread对象
    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
    	sMainThreadHandler = thread.getHandler();
	}

    if (false) {
    	Looper.myLooper().setMessageLogging(new
        	LogPrinter(Log.DEBUG, "ActivityThread"));
	}

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

* 首先通过了Looper.prepareMainLooper();为主线程创建了Looper，然后thread.getHandler()保存了主线程的Handler，最后Looper.loop()进入消息循环。请点击查看[Handler和Message和MessageQueue和Looper](https://github.com/nullWolf007/Android/blob/master/%E8%BF%9B%E9%98%B6/Handler%E5%92%8CMessage%E5%92%8CMessageQueue%E5%92%8CLooper.md)
* 最后看下thread.attach(false)

```java
private void attach(boolean system) {
	sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
    	ViewRootImpl.addFirstDrawHandler(new Runnable() {
        	@Override
            public void run() {
            	ensureJitEnabled();
			}
		});
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
        								UserHandle.myUserId());
		//将mAppThread放到RuntimeInit类中的静态变量
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        final IActivityManager mgr = ActivityManager.getService();
        try {
        	//将mAppThread传入ActivityThreadManager中
            mgr.attachApplication(mAppThread);
		} catch (RemoteException ex) {
        	throw ex.rethrowFromSystemServer();
		}
        // Watch for getting close to heap limit.
        BinderInternal.addGcWatcher(new Runnable() {
        	@Override public void run() {
            	if (!mSomeActivitiesChanged) {
                	return;
				}
                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                	if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                    	+ " total=" + (runtime.totalMemory()/1024)
                        + " used=" + (dalvikUsed/1024));
					mSomeActivitiesChanged = false;
                    try {
                    	mgr.releaseSomeActivities(mAppThread);
					} catch (RemoteException e) {
                    	throw e.rethrowFromSystemServer();
					}
				}
			}
		});
	} else {
    	// Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        android.ddm.DdmHandleAppName.setAppName("system_process",
        	UserHandle.myUserId());
		try {
        	mInstrumentation = new Instrumentation();
            ContextImpl context = ContextImpl.createAppContext(
            	this, getSystemContext().mPackageInfo);
			mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
		} catch (Exception e) {
        	throw new RuntimeException(
            	"Unable to instantiate Application():" + e.toString(), e);
		}
	}

    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());

    ViewRootImpl.ConfigChangedCallback configChangedCallback
    	= (Configuration globalConfig) -> {
		synchronized (mResourcesManager) {
        	// We need to apply this change to the resources immediately, because upon returning
            // the view hierarchy will be informed about it.
            if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
            	null /* compat */)) {
                    						  updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());
                
                // This actually changed the resources! Tell everyone about it.
                if (mPendingConfiguration == null
                	|| mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                    mPendingConfiguration = globalConfig;
                    sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
				}
			}
		}
	};
	ViewRootImpl.addConfigCallback(configChangedCallback);
}
```

* 对于system值为false的情况，上述代码主要完成两件事情。

  * 1.调用RuntimeInit.setApplicationObject(mAppThread.asBinder());方法，把对象mAppThread的Binder放到了RuntimeInit类中的静态变量mApplicationObject中。mAppThread就是ApplicationThread对象
  
    ```java
    public static final void setApplicationObject(IBinder app) {
    	mApplicationObject = app;
    }
    ```
  
  * 2.调用**ActivityManagerService**的attachApplication()方法，将mAppThread 作为参数传入ActivityManagerService，这样ActivityManagerService就可以调用**ApplicaitonThread**的接口了，这样就把AMS和ActivityThread通过Binder联系起来了

### 三、ActivityThread关联时机

#### 3.1 APP的启动过程

* 可以点击查看[APP的启动过程](https://github.com/nullWolf007/Android/blob/master/进阶/启动相关(Context%EF%BC%8C跨进程等)/APP的启动过程.md)

#### 3.2 ActivityThread中的handlerMessage

* 从APP的启动过程中，我们可以了解到ApplicationThread和ActivityThread之间是通过handler及性能通信的，所以我们可以在ActivityThread中发现handlerMessage()方法，该方法会对Message消息进行判断做相应的处理，如下图代码

* ```java
      private class H extends Handler {
          public static final int LAUNCH_ACTIVITY         = 100;
          public static final int PAUSE_ACTIVITY          = 101;
          public static final int PAUSE_ACTIVITY_FINISHING= 102;
          public static final int STOP_ACTIVITY_SHOW      = 103;
          //省略了 太多了
          
          String codeToString(int code) {
              if (DEBUG_MESSAGES) {
                  switch (code) {
                      case LAUNCH_ACTIVITY: return "LAUNCH_ACTIVITY";
                      case PAUSE_ACTIVITY: return "PAUSE_ACTIVITY";
                      case PAUSE_ACTIVITY_FINISHING: return "PAUSE_ACTIVITY_FINISHING";
                      case STOP_ACTIVITY_SHOW: return "STOP_ACTIVITY_SHOW";
                      case STOP_ACTIVITY_HIDE: return "STOP_ACTIVITY_HIDE";
                      //省略了 太多了
                  }
              }
              return Integer.toString(code);
          }
          
          
  		public void handleMessage(Message msg) {
              if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
              switch (msg.what) {
                  case LAUNCH_ACTIVITY: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                      final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
  
                      r.packageInfo = getPackageInfoNoCheck(
                              r.activityInfo.applicationInfo, r.compatInfo);
                      handleRelaunchActivity(r, null, "LAUNCH_ACTIVITY");
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case RELAUNCH_ACTIVITY: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                      ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                      handleRelaunchActivity(r);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case PAUSE_ACTIVITY: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                      SomeArgs args = (SomeArgs) msg.obj;
                      handlePauseActivity((IBinder) args.arg1, false,
                              (args.argi1 & USER_LEAVING) != 0, args.argi2,
                              (args.argi1 & DONT_REPORT) != 0, args.argi3);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case PAUSE_ACTIVITY_FINISHING: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                      SomeArgs args = (SomeArgs) msg.obj;
                      handlePauseActivity((IBinder) args.arg1, true, (args.argi1 & USER_LEAVING) != 0,
                              args.argi2, (args.argi1 & DONT_REPORT) != 0, args.argi3);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case STOP_ACTIVITY_SHOW: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStop");
                      SomeArgs args = (SomeArgs) msg.obj;
                      handleStopActivity((IBinder) args.arg1, true, args.argi2, args.argi3);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case STOP_ACTIVITY_HIDE: {
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStop");
                      SomeArgs args = (SomeArgs) msg.obj;
                      handleStopActivity((IBinder) args.arg1, false, args.argi2, args.argi3);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                  } break;
                  case SHOW_WINDOW:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityShowWindow");
                      handleWindowVisibility((IBinder)msg.obj, true);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case HIDE_WINDOW:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityHideWindow");
                      handleWindowVisibility((IBinder)msg.obj, false);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case RESUME_ACTIVITY:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
                      SomeArgs args = (SomeArgs) msg.obj;
                      handleResumeActivity((IBinder) args.arg1, true, args.argi1 != 0, true,
                              args.argi3, "RESUME_ACTIVITY");
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case SEND_RESULT:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityDeliverResult");
                      handleSendResult((ResultData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case DESTROY_ACTIVITY:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityDestroy");
                      handleDestroyActivity((IBinder)msg.obj, msg.arg1 != 0,
                              msg.arg2, false);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case BIND_APPLICATION:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                      AppBindData data = (AppBindData)msg.obj;
                      handleBindApplication(data);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case EXIT_APPLICATION:
                      if (mInitialApplication != null) {
                          mInitialApplication.onTerminate();
                      }
                      Looper.myLooper().quit();
                      break;
                  case NEW_INTENT:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityNewIntent");
                      handleNewIntent((NewIntentData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case RECEIVER:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
                      handleReceiver((ReceiverData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case CREATE_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                      handleCreateService((CreateServiceData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case BIND_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                      handleBindService((BindServiceData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case UNBIND_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceUnbind");
                      handleUnbindService((BindServiceData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case SERVICE_ARGS:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
                      handleServiceArgs((ServiceArgsData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case STOP_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStop");
                      handleStopService((IBinder)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case CONFIGURATION_CHANGED:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "configChanged");
                      mCurDefaultDisplayDpi = ((Configuration)msg.obj).densityDpi;
                      mUpdatingSystemConfig = true;
                      try {
                          handleConfigurationChanged((Configuration) msg.obj, null);
                      } finally {
                          mUpdatingSystemConfig = false;
                      }
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case CLEAN_UP_CONTEXT:
                      ContextCleanupInfo cci = (ContextCleanupInfo)msg.obj;
                      cci.context.performFinalCleanup(cci.who, cci.what);
                      break;
                  case GC_WHEN_IDLE:
                      scheduleGcIdler();
                      break;
                  case DUMP_SERVICE:
                      handleDumpService((DumpComponentInfo)msg.obj);
                      break;
                  case LOW_MEMORY:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "lowMemory");
                      handleLowMemory();
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case ACTIVITY_CONFIGURATION_CHANGED:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityConfigChanged");
                      handleActivityConfigurationChanged((ActivityConfigChangeData) msg.obj,
                              INVALID_DISPLAY);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case ACTIVITY_MOVED_TO_DISPLAY:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityMovedToDisplay");
                      handleActivityConfigurationChanged((ActivityConfigChangeData) msg.obj,
                              msg.arg1 /* displayId */);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case PROFILER_CONTROL:
                      handleProfilerControl(msg.arg1 != 0, (ProfilerInfo)msg.obj, msg.arg2);
                      break;
                  case CREATE_BACKUP_AGENT:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "backupCreateAgent");
                      handleCreateBackupAgent((CreateBackupAgentData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case DESTROY_BACKUP_AGENT:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "backupDestroyAgent");
                      handleDestroyBackupAgent((CreateBackupAgentData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case SUICIDE:
                      Process.killProcess(Process.myPid());
                      break;
                  case REMOVE_PROVIDER:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "providerRemove");
                      completeRemoveProvider((ProviderRefCount)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case ENABLE_JIT:
                      ensureJitEnabled();
                      break;
                  case DISPATCH_PACKAGE_BROADCAST:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastPackage");
                      handleDispatchPackageBroadcast(msg.arg1, (String[])msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case SCHEDULE_CRASH:
                      throw new RemoteServiceException((String)msg.obj);
                  case DUMP_HEAP:
                      handleDumpHeap((DumpHeapData) msg.obj);
                      break;
                  case DUMP_ACTIVITY:
                      handleDumpActivity((DumpComponentInfo)msg.obj);
                      break;
                  case DUMP_PROVIDER:
                      handleDumpProvider((DumpComponentInfo)msg.obj);
                      break;
                  case SLEEPING:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "sleeping");
                      handleSleeping((IBinder)msg.obj, msg.arg1 != 0);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case SET_CORE_SETTINGS:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "setCoreSettings");
                      handleSetCoreSettings((Bundle) msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case UPDATE_PACKAGE_COMPATIBILITY_INFO:
                      handleUpdatePackageCompatibilityInfo((UpdateCompatibilityData)msg.obj);
                      break;
                  case TRIM_MEMORY:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "trimMemory");
                      handleTrimMemory(msg.arg1);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
                  case UNSTABLE_PROVIDER_DIED:
                      handleUnstableProviderDied((IBinder)msg.obj, false);
                      break;
                  case REQUEST_ASSIST_CONTEXT_EXTRAS:
                      handleRequestAssistContextExtras((RequestAssistContextExtras)msg.obj);
                      break;
                  case TRANSLUCENT_CONVERSION_COMPLETE:
                      handleTranslucentConversionComplete((IBinder)msg.obj, msg.arg1 == 1);
                      break;
                  case INSTALL_PROVIDER:
                      handleInstallProvider((ProviderInfo) msg.obj);
                      break;
                  case ON_NEW_ACTIVITY_OPTIONS:
                      Pair<IBinder, ActivityOptions> pair = (Pair<IBinder, ActivityOptions>) msg.obj;
                      onNewActivityOptions(pair.first, pair.second);
                      break;
                  case ENTER_ANIMATION_COMPLETE:
                      handleEnterAnimationComplete((IBinder) msg.obj);
                      break;
                  case START_BINDER_TRACKING:
                      handleStartBinderTracking();
                      break;
                  case STOP_BINDER_TRACKING_AND_DUMP:
                      handleStopBinderTrackingAndDump((ParcelFileDescriptor) msg.obj);
                      break;
                  case MULTI_WINDOW_MODE_CHANGED:
                      handleMultiWindowModeChanged((IBinder) ((SomeArgs) msg.obj).arg1,
                              ((SomeArgs) msg.obj).argi1 == 1,
                              (Configuration) ((SomeArgs) msg.obj).arg2);
                      break;
                  case PICTURE_IN_PICTURE_MODE_CHANGED:
                      handlePictureInPictureModeChanged((IBinder) ((SomeArgs) msg.obj).arg1,
                              ((SomeArgs) msg.obj).argi1 == 1,
                              (Configuration) ((SomeArgs) msg.obj).arg2);
                      break;
                  case LOCAL_VOICE_INTERACTION_STARTED:
                      handleLocalVoiceInteractionStarted((IBinder) ((SomeArgs) msg.obj).arg1,
                              (IVoiceInteractor) ((SomeArgs) msg.obj).arg2);
                      break;
                  case ATTACH_AGENT:
                      handleAttachAgent((String) msg.obj);
                      break;
                  case APPLICATION_INFO_CHANGED:
                      mUpdatingSystemConfig = true;
                      try {
                          handleApplicationInfoChanged((ApplicationInfo) msg.obj);
                      } finally {
                          mUpdatingSystemConfig = false;
                      }
                      break;
              }
              Object obj = msg.obj;
              if (obj instanceof SomeArgs) {
                  ((SomeArgs) obj).recycle();
              }
              if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
          }
      }
  ```

* 从上图可以看到ActivityThread中存在一个内部类H，里面定义了很多Message消息，如Activity、Service、Application等的创建、销毁等。由此可以知道Activity、Service、Applicaiton等创建、销毁都是由ActivityThread来操控的。ActivityThread就相当于一个领导，管理者下面很多的员工（Activity等）

#### 3.3 关联

**3.3.1 Application关联**

* 由上面的handlerMessage()方法，我们可以找到BIND_APPLICATION，这个就是和Application关联的case。里面的方法就是handleBindApplication()。所以我们一起看下handleBindApplication()这个方法

* 核心代码（太多了，有删减）

* ```java
     // ActivityThread.java
     private void handleBindApplication(AppBindData data) { 
        ......//太多了 省略了
        try {
              // If the app is being launched for full backup or restore, bring it up in
              // a restricted environment with the base application class.
              Application app = data.info.makeApplication(data.restrictedBackupMode, null);
              mInitialApplication = app;
              ......//太多了 省略了
          } finally {
              // If the app targets < O-MR1, or doesn't change the thread policy
              // during startup, clobber the policy to maintain behavior of b/36951662
              if (data.appInfo.targetSdkVersion <= Build.VERSION_CODES.O
                      || StrictMode.getThreadPolicy().equals(writesAllowedPolicy)) {
                  StrictMode.setThreadPolicy(savedPolicy);
              }
          }
     }

     // LoadedApk.java
     public Application makeApplication(boolean forceDefaultAppClass,
              Instrumentation instrumentation) {
          if (mApplication != null) {
              return mApplication;
          }
      
          Application app = null;
      
          String appClass = mApplicationInfo.className;
          if (forceDefaultAppClass || (appClass == null)) {
              appClass = "android.app.Application";
          }
      
          try {
               java.lang.ClassLoader cl = getClassLoader();
              if (!mPackageName.equals("android")) {
                  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                          "initializeJavaContextClassLoader");
                  initializeJavaContextClassLoader();
                  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              }
              ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this); // 创建ContextImpl实例
              app = mActivityThread.mInstrumentation.newApplication(
                      cl, appClass, appContext);
              appContext.setOuterContext(app);// 将Application实例传递给Context实例
          } catch (Exception e) {
              ...
          }
          mActivityThread.mAllApplications.add(app);
          mApplication = app;
      
          return app;
      }
  ```

* 对于应用程序只有一个进程而言，都会首先创建Application对象。

**3.3.2 Activity关联**

* 我们同样也可以找到handleLaunchActivity()方法

* ```java
        private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
          ...
          Activity a = performLaunchActivity(r, customIntent); // 到下一步
    
          if (a != null) {
              r.createdConfig = new Configuration(mConfiguration);
              Bundle oldState = r.state;
              handleResumeActivity(r.token, false, r.isForward,
                      !r.activity.mFinished && !r.startsNotResumed);
              ...
          }
          ...
       }

      private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
          ...    
          Activity activity = null;
          try {
              java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
              activity = mInstrumentation.newActivity(
                      cl, component.getClassName(), r.intent);
              StrictMode.incrementExpectedActivityCount(activity.getClass());
              r.intent.setExtrasClassLoader(cl);
              if (r.state != null) {
                  r.state.setClassLoader(cl);
              }
          } catch (Exception e) {
              ...
          }
      
          try {
              Application app = r.packageInfo.makeApplication(false, mInstrumentation);
      
              if (activity != null) {
                  Context appContext = createBaseContextForActivity(r, activity); // 创建Context
                  CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                  Configuration config = new Configuration(mCompatConfiguration);
                  if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                          + r.activityInfo.name + " with config " + config);
                  activity.attach(appContext, this, getInstrumentation(), r.token,
                          r.ident, app, r.intent, r.activityInfo, title, r.parent,
                          r.embeddedID, r.lastNonConfigurationInstances, config);
      
                  if (customIntent != null) {
                      activity.mIntent = customIntent;
                  }
                  r.lastNonConfigurationInstances = null;
                  activity.mStartedActivity = false;
                  int theme = r.activityInfo.getThemeResource();
                  if (theme != 0) {
                      activity.setTheme(theme);
                  }
    
              mActivities.put(r.token, r);
      
          } catch (SuperNotCalledException e) {
              ...
      
          } catch (Exception e) {
              ...
          }
      
          return activity;
      }

      private Context createBaseContextForActivity(ActivityClientRecord r,
              final Activity activity) {
          ContextImpl appContext = new ContextImpl();  // 创建ContextImpl实例
          appContext.init(r.packageInfo, r.token, this);
          appContext.setOuterContext(activity);
      
          // For debugging purposes, if the activity's package name contains the value of
          // the "debug.use-second-display" system property as a substring, then show
          // its content on a secondary display if there is one.
          Context baseContext = appContext;
          String pkgName = SystemProperties.get("debug.second-display.pkg");
          if (pkgName != null && !pkgName.isEmpty()
                  && r.packageInfo.mPackageName.contains(pkgName)) {
              DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
              for (int displayId : dm.getDisplayIds()) {
                  if (displayId != Display.DEFAULT_DISPLAY) {
                      Display display = dm.getRealDisplay(displayId);
                      baseContext = appContext.createDisplayContext(display);
                      break;
                  }
              }
          }
          return baseContext;
      }
    ```
    
* 通过startActivity()或startActivityForResult()请求启动一个Activity时，如果系统检测需要新建一个Activity对象时，就会回调handleLaunchActivity()方法，该方法继而调用performLaunchActivity()方法，去创建一个Activity实例，并且回调onCreate()，onStart()方法等，函数位于 ActivityThread.java类。

**3.3.3 Service的关联**

```java
    private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

            ContextImpl context = new ContextImpl(); // 创建ContextImpl实例
            context.init(packageInfo, null, this);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            context.setOuterContext(service);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, 0, 0, 0);
            } catch (RemoteException e) {
                // nothing to do.
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```

* 通过startService或者bindService时，如果系统检测到需要新创建一个Service实例，就会回调handleCreateService()方法，完成相关数据操作。handleCreateService()函数位于ActivityThread.java类

