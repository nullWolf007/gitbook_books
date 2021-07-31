[TOC]

### PackageInstall安装APK

#### 转载

* [Android包管理机制（二）PackageInstaller安装APK](http://liuwangshu.cn/framework/pms/2-packageinstaller-apk.html)

#### 说明

* 本系列文章的源码基于Android8.0。

### 1.PackageInstaller中的处理

* 紧接着上一篇的内容，在PackageInstallerActivity调用startInstallConfirm方法初始化安装确认界面后，这个安装确认界面就会呈现给用户，用户如果想要安装这个应用程序就会点击确定按钮，就会调用PackageInstallerActivity的onClick方法，如下所示。

#### 1.1 PackageInstallerActivity.java#onClick

```java
public void onClick(View v) {
        if (v == mOk) {
            if (mOk.isEnabled()) {
                if (mOkCanInstall || mScrollView == null) {
                    if (mSessionId != -1) {
                        mInstaller.setPermissionsResult(mSessionId, true);
                        finish();
                    } else {
                        startInstall();//1
                    }
                } else {
                    mScrollView.pageScroll(View.FOCUS_DOWN);
                }
            }
        } else if (v == mCancel) {
            ...
            finish();
        }
    }
```

* onClick方法中分别对确定和取消按钮做处理，主要查看对确定按钮的处理
* 注释1处调用了startInstall方法：

#### 1.2 PackageInstallerActivity.java#startInstall

```java
private void startInstall() {
     Intent newIntent = new Intent();
     newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
             mPkgInfo.applicationInfo);
     newIntent.setData(mPackageURI);//1
     newIntent.setClass(this, InstallInstalling.class);
     String installerPackageName = getIntent().getStringExtra(
             Intent.EXTRA_INSTALLER_PACKAGE_NAME);
     if (mOriginatingURI != null) {
         newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
     }
     ...
     if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
     startActivity(newIntent);
     finish();
 }
```

* startInstall方法用于跳转到InstallInstalling这个Activity，并关闭掉当前的PackageInstallerActivity。
* InstallInstalling主要用于向包管理器发送包的信息并处理包管理的回调。 InstallInstalling的onCreate方法如下所示。

#### 1.3 InstallInstalling.java#onCreate

* packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java

```JAVA
@Override
 protected void onCreate(@Nullable Bundle savedInstanceState) {
     super.onCreate(savedInstanceState);
     setContentView(R.layout.install_installing);
     ApplicationInfo appInfo = getIntent()
             .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
     mPackageURI = getIntent().getData();
     if ("package".equals(mPackageURI.getScheme())) {
         try {
             getPackageManager().installExistingPackage(appInfo.packageName);
             launchSuccess();
         } catch (PackageManager.NameNotFoundException e) {
             launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
         }
     } else {
     //根据mPackageURI创建一个对应的File 
         final File sourceFile = new File(mPackageURI.getPath());
         PackageUtil.initSnippetForNewApp(this, PackageUtil.getAppSnippet(this, appInfo,
                 sourceFile), R.id.app_snippet);
         //如果savedInstanceState不为null，获取此前保存的mSessionId和mInstallId       
         if (savedInstanceState != null) {//1
             mSessionId = savedInstanceState.getInt(SESSION_ID);
             mInstallId = savedInstanceState.getInt(INSTALL_ID);
           //向InstallEventReceiver注册一个观察者
             try {
                 InstallEventReceiver.addObserver(this, mInstallId,
                         this::launchFinishBasedOnResult);//2
             } catch (EventResultPersister.OutOfIdsException e) {
      
             }
         } else {
             PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                     PackageInstaller.SessionParams.MODE_FULL_INSTALL);//3
             params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
             params.originatingUri = getIntent()
                     .getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
             params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                     UID_UNKNOWN);
             File file = new File(mPackageURI.getPath());//4
             try {
                 PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);//5
                 params.setAppPackageName(pkg.packageName);
                 params.setInstallLocation(pkg.installLocation);
                 params.setSize(
                         PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
             } catch (PackageParser.PackageParserException e) {
                ...
             }
             try {
                 mInstallId = InstallEventReceiver
                         .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                 this::launchFinishBasedOnResult);//6
             } catch (EventResultPersister.OutOfIdsException e) {
                 launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
             }
             try {
                 mSessionId = getPackageManager().getPackageInstaller().createSession(params);//7
             } catch (IOException e) {
                 launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
             }
         }
          ...
         mSessionCallback = new InstallSessionCallback();
     }
 }
```

* onCreate方法中会分别对package和content协议的Uri进行处理，我们来看content协议的Uri处理部分。
* 注释1处如果savedInstanceState不为null，获取此前保存的mSessionId和mInstallId，其中mSessionId是安装包的会话id，mInstallId是等待的安装事件id。
* 注释2处根据mInstallId向InstallEventReceiver注册一个观察者，launchFinishBasedOnResult会接收到安装事件的回调，无论安装成功或者失败都会关闭当前的Activity(InstallInstalling)。
* 如果savedInstanceState为null，代码的逻辑也是类似的.
* 注释3处创建SessionParams，它用来代表安装会话的参数，
* 注释4、5处根据mPackageUri对包（APK）进行轻量级的解析，并将解析的参数赋值给SessionParams。
* 注释6处和注释2处类似向InstallEventReceiver注册一个观察者返回一个新的mInstallId，其中InstallEventReceiver继承自BroadcastReceiver，用于接收安装事件并回调给EventResultPersister。 
* 注释7处PackageInstaller的createSession方法内部会通过IPackageInstaller与PackageInstallerService进行进程间通信，最终调用的是PackageInstallerService的createSession方法来创建并返回mSessionId。
* InstallInstalling的onCreate方法就分析到这，接着查看InstallInstalling的onResume方法：

#### 1.4 InstallInstalling.java#onResume

* packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java

```java
@Override
 protected void onResume() {
     super.onResume();
     if (mInstallingTask == null) {
         PackageInstaller installer = getPackageManager().getPackageInstaller();
         PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);//1
         if (sessionInfo != null && !sessionInfo.isActive()) {//2
             mInstallingTask = new InstallingAsyncTask();
             mInstallingTask.execute();
         } else {
             mCancelButton.setEnabled(false);
             setFinishOnTouchOutside(false);
         }
     }
 }
```

* 注释1处根据mSessionId得到SessionInfo，SessionInfo代表安装会话的详细信息。
* 注释2处如果sessionInfo不为Null并且不是活动的，就创建并执行InstallingAsyncTask。InstallingAsyncTask的doInBackground方法中会根据包(APK)的Uri，将APK的信息通过IO流的形式写入到PackageInstaller.Session中。InstallingAsyncTask的onPostExecute方法如下所示。

#### 1.5 InstallInstalling.java#InstallingAsyncTask

* packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java

```java
@Override
 protected void onPostExecute(PackageInstaller.Session session) {
     if (session != null) {
         Intent broadcastIntent = new Intent(BROADCAST_ACTION);
         broadcastIntent.setPackage(
                 getPackageManager().getPermissionControllerPackageName());
         broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);
         PendingIntent pendingIntent = PendingIntent.getBroadcast(
                 InstallInstalling.this,
                 mInstallId,
                 broadcastIntent,
                 PendingIntent.FLAG_UPDATE_CURRENT);
         session.commit(pendingIntent.getIntentSender());//1
         mCancelButton.setEnabled(false);
         setFinishOnTouchOutside(false);
     } else {
         getPackageManager().getPackageInstaller().abandonSession(mSessionId);
         if (!isCancelled()) {
             launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
         }
     }
 }
```

* 创建了一个PendingIntent，并将该PendingIntent的IntentSender通过注释1处的PackageInstaller.Session的commit方法发送出去，发送去哪了呢？接着查看PackageInstaller.Session的commit方法。

#### 1.6 PackageInstaller.java#commit

* frameworks/base/core/java/android/content/pm/PackageInstaller.java

```java
public void commit(@NonNull IntentSender statusReceiver) {
           try {
               mSession.commit(statusReceiver);
           } catch (RemoteException e) {
               throw e.rethrowFromSystemServer();
           }
       }
```

* mSession的类型为IPackageInstallerSession，这说明要通过IPackageInstallerSession来进行进程间的通信，最终会调用PackageInstallerSession的commit方法，这样代码逻辑就到了Java框架层的。

### 2.Java框架层的处理

#### 2.1 PackageInstallerSession.java#commit

* frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java

```java
@Override
   public void commit(IntentSender statusReceiver) {
       Preconditions.checkNotNull(statusReceiver);
       ...
       mActiveCount.incrementAndGet();
       final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(mContext,
               statusReceiver, sessionId, mIsInstallerDeviceOwner, userId);
       mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();//1
   }
```

* commit方法中会将包的信息封装为PackageInstallObserverAdapter ，它在PMS中被定义。
* 在注释1处会向Handler发送一个类型为MSG_COMMIT的消息，其中`adapter.getBinder()`会得到IPackageInstallObserver2.Stub类型的观察者，从类型就知道这个观察者是可以跨进程进行回调的。处理该消息的代码如下所示。

#### 2.2 PackageInstallerSession.java

* frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java

```java
private final Handler.Callback mHandlerCallback = new Handler.Callback() {
      @Override
      public boolean handleMessage(Message msg) {
          final PackageInfo pkgInfo = mPm.getPackageInfo(
                  params.appPackageName, PackageManager.GET_SIGNATURES
                          | PackageManager.MATCH_STATIC_SHARED_LIBRARIES /*flags*/, userId);
          final ApplicationInfo appInfo = mPm.getApplicationInfo(
                  params.appPackageName, 0, userId);
          synchronized (mLock) {
              if (msg.obj != null) {
                  mRemoteObserver = (IPackageInstallObserver2) msg.obj;//1
              }
              try {
                  commitLocked(pkgInfo, appInfo);//2
              } catch (PackageManagerException e) {
                  final String completeMsg = ExceptionUtils.getCompleteMessage(e);
                  Slog.e(TAG, "Commit of session " + sessionId + " failed: " + completeMsg);
                  destroyInternal();
                  dispatchSessionFinished(e.error, completeMsg, null);//3
              }
              return true;
          }
      }
  };
```

* 注释1处获取IPackageInstallObserver2类型的观察者mRemoteObserver
* 注释2处的commitLocked方法如下所示。

#### 2.3 PackageInstallerSession.java#commitLocked

```java
private void commitLocked(PackageInfo pkgInfo, ApplicationInfo appInfo)
          throws PackageManagerException {
     ...
      mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
              installerPackageName, installerUid, user, mCertificates);
  }
```

* commitLocked方法比较长，这里截取最主要的信息，会调用PMS的installStage方法，这样代码逻辑就进入了PMS中。
* 回到mHandlerCallback的handleMessage方法，如果commitLocked方法出现PackageManagerException异常，就会调用注释3处的dispatchSessionFinished方法，它的实现如下所示：

#### 2.4 PackageInstallerSession.java#dispatchSessionFinished

* frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java

```java
private void dispatchSessionFinished(int returnCode, String msg, Bundle extras) {
       mFinalStatus = returnCode;
       mFinalMessage = msg;
       if (mRemoteObserver != null) {
           try {
               mRemoteObserver.onPackageInstalled(mPackageName, returnCode, msg, extras);//1
           } catch (RemoteException ignored) {
           }
       }
       ...
   }
```

* 注释1处会调用IPackageInstallObserver2的onPackageInstalled方法，具体是实现在PackageInstallObserver类中：

#### 2.5 PackageInstallObserver.java#onPackageInstalled

* frameworks/base/core/java/android/app/PackageInstallObserver.java

```java
public class PackageInstallObserver {
    private final IPackageInstallObserver2.Stub mBinder = new IPackageInstallObserver2.Stub() {
        ...
        @Override
        public void onPackageInstalled(String basePackageName, int returnCode,
                String msg, Bundle extras) {
            PackageInstallObserver.this.onPackageInstalled(basePackageName, returnCode, msg,
                    extras);//1
        }
    };
```

* 注释1处调用了PackageInstallObserver的onPackageInstalled方法，实现这个方法的类为PackageInstallObserver的子类、前面提到的PackageInstallObserverAdapter。
* 总结一下就是dispatchSessionFinished方法会通过mRemoteObserver的onPackageInstalled方法，将Complete方法出现的PackageManagerException的异常信息回调给PackageInstallObserverAdapter。

### 3.总结

* 本篇文章讲解了PackageInstaller安装APK的过程，简单来说就两步：

1. 将APK的信息通过IO流的形式写入到PackageInstaller.Session中。
2. 调用PackageInstaller.Session的commit方法，将APK的信息交由PMS处理。

* 由于PMS中对APK安装的处理比较复杂，因此关于PMS的处理部分会在本系列的下一篇文章进行讲解。