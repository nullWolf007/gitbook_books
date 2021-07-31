[TOC]

## PMS处理APK的安装

#### 转载

* [Android包管理机制（三）PMS处理APK的安装](http://liuwangshu.cn/framework/pms/3-pms-install.html)

### 1.前言

* 在上一篇文章[Android包管理机制（二）PackageInstaller安装APK](http://liuwangshu.cn/framework/pms/2-packageinstaller-apk.html)中，我们学习了PackageInstaller是如何安装APK的，最后会将APK的信息交由PMS处理。那么PMS是如何处理的呢？这篇文章会给你答案。

### 2.PackageHandler处理安装消息

#### 2.1 时序图

* APK的信息交由PMS后，PMS通过向PackageHandler发送消息来驱动APK的复制和安装工作。
* 先来查看PackageHandler处理安装消息的调用时序图。

[![VeClRJ.png](https://s2.ax1x.com/2019/05/27/VeClRJ.png)](https://s2.ax1x.com/2019/05/27/VeClRJ.png)

#### 2.2 PackageManagerService.java#installStage

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

* 接着上一篇文章的代码逻辑来查看PMS的installStage方法。

```java
void installStage(String packageName, File stagedDir, String stagedCid,
           IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
           String installerPackageName, int installerUid, UserHandle user,
           Certificate[][] certificates) {
       ...
       final Message msg = mHandler.obtainMessage(INIT_COPY);//1
       final int installReason = fixUpInstallReason(installerPackageName, installerUid,
               sessionParams.installReason);
       final InstallParams params = new InstallParams(origin, null, observer,
               sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
               verificationInfo, user, sessionParams.abiOverride,
               sessionParams.grantedRuntimePermissions, certificates, installReason);//2
       params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
       msg.obj = params;
       ...
       mHandler.sendMessage(msg);//3
   }
```

* 注释2处创建InstallParams，它对应于包的安装数据。
* 注释1处创建了类型为INIT_COPY的消息，
* 在注释3处将InstallParams通过消息发送出去。

####  2.3 对INIT_COPY的消息的处理

##### 2.3.1 PackageManagerService.java#PackageHandler

* 处理INIT_COPY类型的消息的代码如下所示
* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#PackageHandler

```java
	void doHandleMessage(Message msg) {
           switch (msg.what) {
               case INIT_COPY: {
                   HandlerParams params = (HandlerParams) msg.obj;
                   int idx = mPendingInstalls.size();
                   if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
                   //mBound用于标识是否绑定了服务，默认值为false
                   if (!mBound) {//1
                       Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                               System.identityHashCode(mHandler));
                       //如果没有绑定服务，重新绑定，connectToService方法内部如果绑定成功会将mBound置为true
                       if (!connectToService()) {//2
                           Slog.e(TAG, "Failed to bind to media container service");
                           params.serviceError();
                           Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                                   System.identityHashCode(mHandler));
                           if (params.traceMethod != null) {
                               Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                       params.traceCookie);
                           }
                           //绑定服务失败则return
                           return;
                       } else {
                           //绑定服务成功，将请求添加到ArrayList类型的mPendingInstalls中，等待处理
                           mPendingInstalls.add(idx, params);
                       }
                   } else {
                   //已经绑定服务
                       mPendingInstalls.add(idx, params);
                       if (idx == 0) {
                           mHandler.sendEmptyMessage(MCS_BOUND);//3
                       }
                   }
                   break;
               }
               ....
               }
   }
}
```

* PackageHandler继承自Handler，它被定义在PMS中，doHandleMessage方法用于处理各个类型的消息，来查看对INIT_COPY类型消息的处理。
* 注释1处的mBound用于标识是否绑定了DefaultContainerService，默认值为false。DefaultContainerService是用于检查和复制可移动文件的服务，这是一个比较耗时的操作，因此DefaultContainerService没有和PMS运行在同一进程中，它运行在com.android.defcontainer进程，通过IMediaContainerService和PMS进行IPC通信，如下图所示。
  [![VeCQG4.png](https://s2.ax1x.com/2019/05/27/VeCQG4.png)](https://s2.ax1x.com/2019/05/27/VeCQG4.png)

* 注释2处的connectToService方法用来绑定DefaultContainerService，注释3处发送MCS_BOUND类型的消息，触发处理第一个安装请求。

* 查看注释2处的connectToService方法：

##### 2.3.2 PackageManagerService.java#PackageHandler#connectToService

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#PackageHandler 

```java
private boolean connectToService() {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "Trying to bind to" +
                  " DefaultContainerService");
          Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
          Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          if (mContext.bindServiceAsUser(service, mDefContainerConn,
                  Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {//1
              Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
              mBound = true;//2
              return true;
          }
          Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
          return false;
      }
```

* 注释2处如果绑定DefaultContainerService成功，mBound会置为ture 。
* 注释1处的bindServiceAsUser方法会传入mDefContainerConn，bindServiceAsUser方法的处理逻辑和我们调用bindService是类似的，服务建立连接后，会调用onServiceConnected方法：

##### 2.3.3 PackageManagerService#DefaultContainerConnection

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java 

```java
class DefaultContainerConnection implements ServiceConnection {
      public void onServiceConnected(ComponentName name, IBinder service) {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
          final IMediaContainerService imcs = IMediaContainerService.Stub
                  .asInterface(Binder.allowBlocking(service));
          mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, Object));//1
      }
      public void onServiceDisconnected(ComponentName name) {
          if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceDisconnected");
      }
  }
```

* 注释1处发送了MCS_BOUND类型的消息，与`PackageHandler.doHandleMessage`方法的注释3处不同的是，这里发送消息带了Object类型的参数，这里会对这两种情况来进行讲解，一种是消息不带Object类型的参数，一种是消息带Object类型的参数。

#### 2.4 对MCS_BOUND类型的消息的处理

##### 2.4.1 消息不带Object类型的参数

* 查看对MCS_BOUND类型消息的处理：
* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```java
case MCS_BOUND: {
            if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
            if (msg.obj != null) {//1
                mContainerService = (IMediaContainerService) msg.obj;//2
                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                        System.identityHashCode(mHandler));
            }
            if (mContainerService == null) {//3
                if (!mBound) {//4
                      Slog.e(TAG, "Cannot bind to media container service");
                      for (HandlerParams params : mPendingInstalls) {
                          params.serviceError();//5
                          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                        System.identityHashCode(params));
                          if (params.traceMethod != null) {
                          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER,
                           params.traceMethod, params.traceCookie);
                          }
                          return;
                      }   
                          //绑定失败，清空安装请求队列
                          mPendingInstalls.clear();
                   } else {
                          //继续等待绑定服务
                          Slog.w(TAG, "Waiting to connect to media container service");
                   }
            } else if (mPendingInstalls.size() > 0) {
              ...
              else {
                   Slog.w(TAG, "Empty queue");
                   }
            break;
        }
```

* 如果消息不带Object类型的参数，就无法满足注释1处的条件，注释2处的IMediaContainerService类型的mContainerService也无法被赋值，这样就满足了注释3处的条件。
* 如果满足注释4处的条件，说明还没有绑定服务，而此前已经在`PackageHandler.doHandleMessage`方法的注释2处调用绑定服务的方法了，这显然是不正常的，因此在注释5处负责处理服务发生错误的情况。如果不满足注释4处的条件，说明已经绑定服务了，就会打印出系统log，告知用户等待系统绑定服务。

##### 2.4.2 消息带Object类型的参数

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```java
case MCS_BOUND: {
            if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
            if (msg.obj != null) {
            ...
            }
            if (mContainerService == null) {//1
             ...
            } else if (mPendingInstalls.size() > 0) {//2
                          HandlerParams params = mPendingInstalls.get(0);//3
                        if (params != null) {
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                    System.identityHashCode(params));
                            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                            if (params.startCopy()) {//4
                                if (DEBUG_SD_INSTALL) Log.i(TAG,
                                        "Checking for more work or unbind...");
                                 //如果APK安装成功，删除本次安装请求
                                if (mPendingInstalls.size() > 0) {
                                    mPendingInstalls.remove(0);
                                }
                                if (mPendingInstalls.size() == 0) {
                                    if (mBound) {
                                    //如果没有安装请求了，发送解绑服务的请求
                                        if (DEBUG_SD_INSTALL) Log.i(TAG,
                                                "Posting delayed MCS_UNBIND");
                                        removeMessages(MCS_UNBIND);
                                        Message ubmsg = obtainMessage(MCS_UNBIND);
                                        sendMessageDelayed(ubmsg, 10000);
                                    }
                                } else {
                                    if (DEBUG_SD_INSTALL) Log.i(TAG,
                                            "Posting MCS_BOUND for next work");
                                   //如果还有其他的安装请求，接着发送MCS_BOUND消息继续处理剩余的安装请求       
                                    mHandler.sendEmptyMessage(MCS_BOUND);//5
                                }
                            }
                            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                        }else {
                        Slog.w(TAG, "Empty queue");//6
                    }
            break;
        }
```

* 如果MCS_BOUND类型消息带Object类型的参数就不会满足注释1处的条件，就会调用注释2处的判断，如果安装请求数不大于0就会打印出注释6处的log，说明安装请求队列是空的。
* 安装完一个APK后，就会在注释5处发出MSC_BOUND消息，继续处理剩下的安装请求直到安装请求队列为空。
* 注释3处得到安装请求队列第一个请求HandlerParams ，如果HandlerParams 不为null就会调用注释4处的HandlerParams的startCopy方法，用于开始复制APK的流程。

### 3.复制APK

#### 3.1 时序图

* 先来查看复制APK的时序图。

[![VeCuIU.png](https://s2.ax1x.com/2019/05/27/VeCuIU.png)](https://s2.ax1x.com/2019/05/27/VeCuIU.png)

#### 3.2 HandlerParams#startCopy

* HandlerParams是PMS中的抽象类，它的实现类为PMS的内部类InstallParams。HandlerParams的startCopy方法如下所示
* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#HandlerParams

```java
final boolean startCopy() {
           boolean res;
           try {
               if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
               //startCopy方法尝试的次数，超过了4次，就放弃这个安装请求
               if (++mRetries > MAX_RETRIES) {//1
                   Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                   mHandler.sendEmptyMessage(MCS_GIVE_UP);//2
                   handleServiceError();
                   return false;
               } else {
                   handleStartCopy();//3
                   res = true;
               }
           } catch (RemoteException e) {
               if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
               mHandler.sendEmptyMessage(MCS_RECONNECT);
               res = false;
           }
           handleReturnCode();//4
           return res;
       }
```

* 注释1处的mRetries用于记录startCopy方法调用的次数，调用startCopy方法时会先自动加1，如果次数大于4次就放弃这个安装请求：
* 在注释2处发送MCS_GIVE_UP类型消息，将第一个安装请求（本次安装请求）从安装请求队列mPendingInstalls中移除掉。
* 注释4处用于处理复制APK后的安装APK逻辑，第3小节中会再次提到它。
* 注释3处调用了抽象方法handleStartCopy，它的实现在InstallParams中，如下所示。

#### 3.3PackageManagerService.java#InstallParams#handleStartCopy

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#InstallParams

```java
public void handleStartCopy() throws RemoteException {
       ...
       //确定APK的安装位置。onSd：安装到SD卡， onInt：内部存储即Data分区，ephemeral：安装到临时存储（Instant Apps安装）            
       final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
       final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
       final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
       PackageInfoLite pkgLite = null;
       if (onInt && onSd) {
         // APK不能同时安装在SD卡和Data分区
           Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
           ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
         //安装标志冲突，Instant Apps不能安装到SD卡中
       } else if (onSd && ephemeral) {
           Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
           ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
       } else {
            //获取APK的少量的信息
           pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                   packageAbiOverride);//1
           if (DEBUG_EPHEMERAL && ephemeral) {
               Slog.v(TAG, "pkgLite for install: " + pkgLite);
           }
       ...
       if (ret == PackageManager.INSTALL_SUCCEEDED) {
            //判断安装的位置
           int loc = pkgLite.recommendedInstallLocation;
           if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
               ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
           } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
               ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
           } 
           ...
           }else{
             loc = installLocationPolicy(pkgLite);//2
             ...
           }
       }
       //根据InstallParams创建InstallArgs对象
       final InstallArgs args = createInstallArgs(this);//3
       mArgs = args;
       if (ret == PackageManager.INSTALL_SUCCEEDED) {
              ...
           if (!origin.existing && requiredUid != -1
                   && isVerificationEnabled(
                         verifierUser.getIdentifier(), installFlags, installerUid)) {
                 ...
           } else{
               ret = args.copyApk(mContainerService, true);//4
           }
       }
       mRet = ret;
   }
```

* handleStartCopy方法的代码很多，这里截取关键的部分。
* 注释1处通过IMediaContainerService跨进程调用DefaultContainerService的getMinimalPackageInfo方法，该方法轻量解析APK并得到APK的少量信息，轻量解析的原因是这里不需要得到APK的全部信息，APK的少量信息会封装到PackageInfoLite中。
* 接着在注释2处确定APK的安装位置。
* 注释3处创建了InstallArgs，InstallArgs 是一个抽象类，定义了APK的安装逻辑，比如复制和重命名APK等，它有3个子类，都被定义在PMS中，如下图所示。
  [![VeCMiF.png](https://s2.ax1x.com/2019/05/27/VeCMiF.png)](https://s2.ax1x.com/2019/05/27/VeCMiF.png)
* 其中FileInstallArgs用于处理安装到非ASEC的存储空间的APK，也就是内部存储空间（Data分区），AsecInstallArgs用于处理安装到ASEC中（mnt/asec）即SD卡中的APK。MoveInstallArgs用于处理已安装APK的移动的逻辑。
* 对APK进行检查后就会在注释4处调用InstallArgs的copyApk方法进行安装。
* 不同的InstallArgs子类会有着不同的处理，这里以FileInstallArgs为例。FileInstallArgs的copyApk方法中会直接return FileInstallArgs的doCopyApk方法：

#### 3.4 PackageManagerService.java#FileInstallArgs#doCopyApk

* 不同的InstallArgs子类会有着不同的处理，这里以FileInstallArgs为例。FileInstallArgs的copyApk方法中会直接return FileInstallArgs的doCopyApk方法：

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#FileInstallArgs

```java
private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
        ...
         try {
             final boolean isEphemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
             //创建临时文件存储目录
             final File tempDir =
                     mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);//1
             codeFile = tempDir;
             resourceFile = tempDir;
         } catch (IOException e) {
             Slog.w(TAG, "Failed to create copy file: " + e);
             return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
         }
         ...
         int ret = PackageManager.INSTALL_SUCCEEDED;
         ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);//2
         ...
         return ret;
     }
```

* 注释1处用于创建临时存储目录，比如/data/app/vmdl18300388.tmp，其中18300388是安装的sessionId。
* 注释2处通过IMediaContainerService跨进程调用DefaultContainerService的copyPackage方法，这个方法会在DefaultContainerService所在的进程中将APK复制到临时存储目录，比如/data/app/vmdl18300388.tmp/base.apk。目前为止APK的复制工作就完成了，接着就是APK的安装过程了。

### 4.安装APK

#### 4.1 时序图

* 照例先来查看安装APK的时序图。
  [![VeCnaT.png](https://s2.ax1x.com/2019/05/27/VeCnaT.png)](https://s2.ax1x.com/2019/05/27/VeCnaT.png)
* 我们回到APK的复制调用链的头部方法：HandlerParams的startCopy方法，在注释4处会调用handleReturnCode方法，它的实现在InstallParams中，如下所示。

#### 4.2 InstallParams#handleReturnCode

* HandlerParams是PMS中的抽象类，它的实现类为PMS的内部类InstallParams。

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java#InstallParams

```java
void handleReturnCode() {
    if (mArgs != null) {
        processPendingInstall(mArgs, mRet);
    }
}

    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus);
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    //安装前处理
                    args.doPreInstall(res.returnCode);//1
                    synchronized (mInstallLock) {
                        installPackageTracedLI(args, res);//2
                    }
                    //安装后收尾
                    args.doPostInstall(res.returnCode, res.uid);//3
                }
              ...
            }
        });
    }
```

* handleReturnCode方法中只调用了processPendingInstall方法
* 注释1处用于检查APK的状态的，在安装前确保安装环境的可靠，如果不可靠会清除复制的APK文件，
* 注释3处用于处理安装后的收尾操作，如果安装不成功，删除掉安装相关的目录与文件。
* 主要来看注释2处的installPackageTracedLI方法，其内部会调用PMS的installPackageLI方法。

#### 4.3 PackageManagerService.java#installPackageLI

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setDisplayMetrics(mMetrics);
    pp.setCallback(mPackageParserCallback);
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final PackageParser.Package pkg;
    try {
        //解析APK
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);//1
    } catch (PackageParserException e) {
        res.setError("Failed parse during installPackageLI", e);
        return;
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
    ...
    pp = null;
    String oldCodePath = null;
    boolean systemApp = false;
    synchronized (mPackages) {
        // 检查APK是否存在
        if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
            String oldName = mSettings.getRenamedPackageLPr(pkgName);//获取没被改名前的包名
            if (pkg.mOriginalPackages != null
                    && pkg.mOriginalPackages.contains(oldName)
                    && mPackages.containsKey(oldName)) {
                pkg.setPackageName(oldName);//2
                pkgName = pkg.packageName;
                replace = true;//设置标志位表示是替换安装
                if (DEBUG_INSTALL) Slog.d(TAG, "Replacing existing renamed package: oldName="
                        + oldName + " pkgName=" + pkgName);
            } 
            ...
        }
        PackageSetting ps = mSettings.mPackages.get(pkgName);
        //查看Settings中是否存有要安装的APK的信息，如果有就获取签名信息
        if (ps != null) {//3
            if (DEBUG_INSTALL) Slog.d(TAG, "Existing package: " + ps);
            PackageSetting signatureCheckPs = ps;
            if (pkg.applicationInfo.isStaticSharedLibrary()) {
                SharedLibraryEntry libraryEntry = getLatestSharedLibraVersionLPr(pkg);
                if (libraryEntry != null) {
                    signatureCheckPs = mSettings.getPackageLPr(libraryEntry.apk);
                }
            }
            //检查签名的正确性
            if (shouldCheckUpgradeKeySetLP(signatureCheckPs, scanFlags)) {
                if (!checkUpgradeKeySetLP(signatureCheckPs, pkg)) {
                    res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                            + pkg.packageName + " upgrade keys do not match the "
                            + "previously installed version");
                    return;
                }
            } 
            ...
        }

        int N = pkg.permissions.size();
        for (int i = N-1; i >= 0; i--) {
           //遍历每个权限，对权限进行处理
            PackageParser.Permission perm = pkg.permissions.get(i);
            BasePermission bp = mSettings.mPermissions.get(perm.info.name);
         
            }
        }
    }
    if (systemApp) {
        if (onExternal) {
            //系统APP不能在SD卡上替换安装
            res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                    "Cannot install updates to system apps on sdcard");
            return;
        } else if (instantApp) {
            //系统APP不能被Instant App替换
            res.setError(INSTALL_FAILED_INSTANT_APP_INVALID,
                    "Cannot update a system app with an instant app");
            return;
        }
    }
    ...
    //重命名临时文件
    if (!args.doRename(res.returnCode, pkg, oldCodePath)) {//4
        res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
        return;
    }

    startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);

    try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
            "installPackageLI")) {
       
        if (replace) {//5
         //替换安装   
           ...
            replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                    installerPackageName, res, args.installReason);
        } else {
        //安装新的APK
            installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                    args.user, installerPackageName, volumeUuid, res, args.installReason);
        }
    }

    synchronized (mPackages) {
        final PackageSetting ps = mSettings.mPackages.get(pkgName);
        if (ps != null) {
            //更新应用程序所属的用户
            res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            ps.setUpdateAvailable(false /*updateAvailable*/);
        }
        ...
    }
}
```

* installPackageLI方法的代码有将近500行，这里截取主要的部分，主要做了几件事：

1. 创建PackageParser解析APK。
2. 检查APK是否存在，如果存在就获取此前没被改名前的包名并在注释1处赋值给PackageParser.Package类型的pkg，在注释3处将标志位replace置为true表示是替换安装。
3. 注释3处，如果Settings中保存有要安装的APK的信息，说明此前安装过该APK，则需要校验APK的签名信息，确保安全的进行替换。
4. 在注释4处将临时文件重新命名，比如前面提到的/data/app/vmdl18300388.tmp/base.apk，重命名为/data/app/包名-1/base.apk。这个新命名的包名会带上一个数字后缀1，每次升级一个已有的App，这个数字会不断的累加。
5. 系统APP的更新安装会有两个限制，一个是系统APP不能在SD卡上替换安装，另一个是系统APP不能被Instant App替换。
6. 注释5处根据replace来做区分，如果是替换安装就会调用replacePackageLIF方法，其方法内部还会对系统APP和非系统APP进行区分处理，如果是新安装APK会调用installNewPackageLIF方法。

* 这里我们以新安装APK为例，会调用PMS的installNewPackageLIF方法。

#### 4.4 PackageManagerService.java#installNewPackageLIF

* frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```java
private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
           int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
           PackageInstalledInfo res, int installReason) {
       ...
       try {
           //扫描APK
           PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                   System.currentTimeMillis(), user);
           //更新Settings信息
           updateSettingsLI(newPackage, installerPackageName, null, res, user, installReason);
           if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
               //安装成功后，为新安装的应用程序准备数据
               prepareAppDataAfterInstallLIF(newPackage);

           } else {
               //安装失败则删除APK
               deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                       PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
           }
       } catch (PackageManagerException e) {
           res.setError("Package couldn't be installed in " + pkg.codePath, e);
       }
       Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
   }
```

* installNewPackageLIF主要做了以下3件事：

1. 扫描APK，将APK的信息存储在PackageParser.Package类型的newPackage中，一个Package的信息包含了1个base APK以及0个或者多个split APK。
2. 更新该APK对应的Settings信息，Settings用于保存所有包的动态设置。
3. 如果安装成功就为新安装的应用程序准备数据，安装失败就删除APK。

* 安装APK的过程就讲到这里，就不再往下分析下去，有兴趣的同学可以接着深挖。

### 5.总结

* 本文主要讲解了PMS是如何处理APK安装的，主要有几个步骤：

1. PackageInstaller安装APK时会将APK的信息交由PMS处理，PMS通过向PackageHandler发送消息来驱动APK的复制和安装工作。
2. PMS发送INIT_COPY和MCS_BOUND类型的消息，控制PackageHandler来绑定DefaultContainerService，完成复制APK等工作。
3. 复制APK完成后，会开始进行安装APK的流程，包括安装前的检查、安装APK和安装后的收尾工作。