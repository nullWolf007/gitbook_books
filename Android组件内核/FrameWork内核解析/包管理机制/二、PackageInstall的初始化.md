[TOC]

### PackageInstall的初始化

#### 转载

* [Android包管理机制（一）PackageInstaller的初始化](http://liuwangshu.cn/framework/pms/1-packageinstaller-initialize.html)

### 1.前言

* 包管理机制是Android中的重要机制，是应用开发和系统开发需要掌握的知识点之一。
* 包指的是Apk、jar和so文件等等，它们被加载到Android内存中，由一个包转变成可执行的代码，这就需要一个机制来进行包的加载、解析、管理等操作，这就是包管理机制。包管理机制由许多类一起组成，其中核心为PackageManagerService（PMS）,它负责对包进行管理，如果直接讲PMS会比较难以理解，因此我们需要一个切入点，这个切入点就是常见的APK的安装。
* 讲到APK的安装之前，先了解下PackageManager、APK文件结构和安装方式。

### 2.PackageManager简介

* 与ActivityManager和AMS的关系类似，PMS也有一个对应的管理类PackageManager，用于向应用程序进程提供一些功能。PackageManager是一个抽象类，它的具体实现类为ApplicationPackageManager，ApplicationPackageManager中的方法会通过IPackageManager与AMS进行进程间通信，因此PackageManager所提供的功能最终是由PMS来实现的，这么设计的主要用意是为了避免系统服务PMS直接被访问。PackageManager提供了一些功能，主要有以下几点：

1. 获取一个应用程序的所有信息（ApplicationInfo）。
2. 获取四大组件的信息。
3. 查询permission相关信息。
4. 获取包的信息。
5. 安装、卸载APK.

### 3.APK文件结构和安装方式

* APK是AndroidPackage的缩写，即Android安装包，它实际上是zip格式的压缩文件，一般情况下，解压后的文件结构如下表所示。

| 目录/文件           | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| assert              | 存放的原生资源文件,通过AssetManager类访问。                  |
| lib                 | 存放库文件。                                                 |
| META-INF            | 保存应用的签名信息，签名信息可以验证APK文件的完整性。        |
| res                 | 存放资源文件。res中除了raw子目录，其他的子目录都参与编译，这些子目录下的资源是通过编译出的R类在代码中访问。 |
| AndroidManifest.xml | 用来声明应用程序的包名称、版本、组件和权限等数据。 apk中的AndroidManifest.xml经过压缩，可以通过AXMLPrinter2工具解开。 |
| classes.dex         | Java源码编译后生成的Java字节码文件。                         |
| resources.arsc      | 编译后的二进制资源文件。                                     |

* APK的安装场景主要有以下几种：
  * 通过adb命令安装：adb 命令包括adb push/install
  * 用户下载的Apk，通过系统安装器packageinstaller安装该Apk。packageinstaller是系统内置的应用程序，用于安装和卸载应用程序。
  * 系统开机时安装系统应用。
  * 电脑或者手机上的应用商店自动安装。

* 这4种方式最终都是由PMS来进行处理，在此之前的调用链是不同的，本篇文章会介绍第二种方式，对于用户来说，这是比较常用的安装方式；对于开发者来说，这是调用链比较长的安装方式，能学到的更多。其他的安装场景会在本系列的后续文章进行讲解。

### 4.寻找PackageInstaller入口

* 在Android7.0之前我们可以通过如下代码安装指定路径中的APK。

```JAVA
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.parse("file://" + path),"application/vnd.android.package-archive");
context.startActivity(intent);
```

* 但是Android7.0或更高版本再这么做，就会报FileUriExposedException异常。这是因为StrictMode API 政策禁止应用程序将file:// Uri暴露给另一个应用程序，如果包含file:// Uri的 intent 离开你的应用，就会报FileUriExposedException 异常。为了解决这个问题，谷歌提供了FileProvider，FileProvider继承自ContentProvider ，使用它可以将file://Uri替换为content://Uri，具体怎么使用FileProvider并不是本文的重点，只要知道无论是Android7.0之前还是Android7.0以及更高版本，都会调用如下代码：

```JAVA
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(xxxxx, "application/vnd.android.package-archive");
```

* Intent的Action属性为ACTION_VIEW，Type属性指定Intent的数据类型为application/vnd.android.package-archive。

* 能隐式匹配的Activity为InstallStart，需要注意的是，这里分析的源码基于Android8.0，7.0能隐式匹配的Activity为PackageInstallerActivity。

#### 4.1 packages/apps/PackageInstaller/AndroidManifest.xml

```XML
<activity android:name=".InstallStart"
               android:exported="true"
               android:excludeFromRecents="true">
           <intent-filter android:priority="1">
               <action android:name="android.intent.action.VIEW" />
               <action android:name="android.intent.action.INSTALL_PACKAGE" />
               <category android:name="android.intent.category.DEFAULT" />
               <data android:scheme="file" />
               <data android:scheme="content" />
               <data android:mimeType="application/vnd.android.package-archive" />
           </intent-filter>
        ...
       </activity>
```

* InstallStart是PackageInstaller中的入口Activity，其中PackageInstaller是系统内置的应用程序，用于安装和卸载应用。当我们调用PackageInstaller来安装应用时会跳转到InstallStart，并调用它的onCreate方法：

#### 4.2 packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStart.java

```java
@Override
  protected void onCreate(@Nullable Bundle savedInstanceState) {
         if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {//1
          nextActivity.setClass(this, PackageInstallerActivity.class);
      } else {
          Uri packageUri = intent.getData();
          if (packageUri == null) {//2
              Intent result = new Intent();
              result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                      PackageManager.INSTALL_FAILED_INVALID_URI);
              setResult(RESULT_FIRST_USER, result);
              nextActivity = null;
          } else {
              if (packageUri.getScheme().equals(SCHEME_CONTENT)) {//3
                  nextActivity.setClass(this, InstallStaging.class);
              } else {
                  nextActivity.setClass(this, PackageInstallerActivity.class);
              }
          }
      }
      if (nextActivity != null) {
          startActivity(nextActivity);
      }
      finish();
  }
```

* 注释1处判断Intent的Action是否为CONFIRM_PERMISSIONS，根据本文的应用情景显然不是，接着往下看，
* 注释2处判断packageUri 是否为空也不成立，
* 注释3处，判断Uri的Scheme协议是否是content，如果是就跳转到InstallStaging，如果不是就跳转到PackageInstallerActivity。本文的应用情景中，Android7.0以及更高版本我们会使用FileProvider来处理URI ，FileProvider会隐藏共享文件的真实路径，将路径转换成content://Uri路径，这样就会跳转到InstallStaging。InstallStaging的onResume方法如下所示。

#### 4.3 packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java#onResume

```JAVA
@Override
  protected void onResume() {
      super.onResume();
      if (mStagingTask == null) {
          if (mStagedFile == null) {
              try {
                  mStagedFile = TemporaryFileManager.getStagedFile(this);//1
              } catch (IOException e) {
                  showError();
                  return;
              }
          }
          mStagingTask = new StagingAsyncTask();
          mStagingTask.execute(getIntent().getData());//2
      }
  }
```

* 注释1处如果File类型的mStagedFile 为null，则创建mStagedFile ，mStagedFile用于存储临时数据。

*  注释2处启动StagingAsyncTask，并传入了content协议的Uri，如下所示。

#### 4.4 packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java#StagingAsyncTask

```JAVA
 private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
        @Override
        protected Boolean doInBackground(Uri... params) {
            if (params == null || params.length <= 0) {
                return false;
            }
            Uri packageUri = params[0];
            try (InputStream in = getContentResolver().openInputStream(packageUri)) {
                if (in == null) {
                    return false;
                }
                try (OutputStream out = new FileOutputStream(mStagedFile)) {
                    byte[] buffer = new byte[4096];
                    int bytesRead;
                    while ((bytesRead = in.read(buffer)) >= 0) {
                        if (isCancelled()) {
                            return false;
                        }
                        out.write(buffer, 0, bytesRead);
                    }
                }
            } catch (IOException | SecurityException e) {
                Log.w(LOG_TAG, "Error staging apk from content URI", e);
                return false;
            }
            return true;
        }
        @Override
        protected void onPostExecute(Boolean success) {
            if (success) {
                Intent installIntent = new Intent(getIntent());
                installIntent.setClass(InstallStaging.this, PackageInstallerActivity.class);
                installIntent.setData(Uri.fromFile(mStagedFile));
                installIntent
                        .setFlags(installIntent.getFlags() & ~Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                installIntent.addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
                startActivityForResult(installIntent, 0);
            } else {
                showError();
            }
        }
    }
}
```

* doInBackground方法中将packageUri（content协议的Uri）的内容写入到mStagedFile中，如果写入成功，onPostExecute方法中会跳转到PackageInstallerActivity中，并将mStagedFile传进去。绕了一圈又回到了PackageInstallerActivity，
* 这里可以看出InstallStaging主要起了转换的作用，将content协议的Uri转换为File协议，然后跳转到PackageInstallerActivity，这样就可以像此前版本（Android7.0之前）一样启动安装流程了。

### 5.PackageInstallerActivity解析

* packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java

* 从功能上来说，PackageInstallerActivity才是应用安装器PackageInstaller真正的入口Activity，PackageInstallerActivity的onCreate方法如下所示。

#### 5.1 PackageInstallerActivity.java#onCreate

```JAVA
@Override
protected void onCreate(Bundle icicle) {
    super.onCreate(icicle);
    if (icicle != null) {
        mAllowUnknownSources = icicle.getBoolean(ALLOW_UNKNOWN_SOURCES_KEY);
    }
    mPm = getPackageManager();
    mIpm = AppGlobals.getPackageManager();
    mAppOpsManager = (AppOpsManager) getSystemService(Context.APP_OPS_SERVICE);
    mInstaller = mPm.getPackageInstaller();
    mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);
    ...
    //根据Uri的Scheme进行预处理
    boolean wasSetUp = processPackageUri(packageUri);//1
    if (!wasSetUp) {
        return;
    }
    bindUi(R.layout.install_confirm, false);
    //判断是否是未知来源的应用，如果开启允许安装未知来源选项则直接初始化安装
    checkIfAllowedAndInitiateInstall();//2
}
```

* 首先初始话安装所需要的各种对象，比如PackageManager、IPackageManager、AppOpsManager和UserManager等等，它们的描述如下表所示。

| 类名             | 描述                                                      |
| ---------------- | --------------------------------------------------------- |
| PackageManager   | 用于向应用程序进程提供一些功能，最终的功能是由PMS来实现的 |
| IPackageManager  | 一个AIDL的接口，用于和PMS进行进程间通信                   |
| AppOpsManager    | 用于权限动态检测,在Android4.3中被引入                     |
| PackageInstaller | 提供安装、升级和删除应用程序功能                          |
| UserManager      | 用于多用户管理                                            |

* 注释1处的processPackageUri方法如下所示。

#### 5.2 PackageInstallerActivity.java#processPackageUri

```JAVA
private boolean processPackageUri(final Uri packageUri) {
     mPackageURI = packageUri;
     final String scheme = packageUri.getScheme();//1
     switch (scheme) {
         case SCHEME_PACKAGE: {
             try {
              ...
         } break;
         case SCHEME_FILE: {
             File sourceFile = new File(packageUri.getPath());//2
             //得到sourceFile的包信息
             PackageParser.Package parsed = PackageUtil.getPackageInfo(this, sourceFile);//3
             if (parsed == null) {
                 Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                 showDialogInner(DLG_PACKAGE_ERROR);
                 setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                 return false;
             }
             //对parsed进行进一步处理得到包信息PackageInfo
             mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                     PackageManager.GET_PERMISSIONS, 0, 0, null,
                     new PackageUserState());//4
             mAppSnippet = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
         } break;
         default: {
             Log.w(TAG, "Unsupported scheme " + scheme);
             setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
             finish();
             return false;
         }
     }
     return true;
 }
```

* 首先在注释1处得到packageUri的Scheme协议，接着根据这个Scheme协议分别对package协议和file协议进行处理，如果不是这两个协议就会关闭PackageInstallerActivity并return false。
* 我们主要来看file协议的处理，注释2处根据packageUri创建一个新的File。
* 注释3处的内部会用PackageParser的parsePackage方法解析这个File（这个File其实是APK文件），得到APK的包信息Package ，Package包含了该APK的所有信息。
* 注释4处会将Package根据uid、用户状态信息和PackageManager的配置等变量对包信息Package做进一步处理得到PackageInfo。

#### 5.3 PackageInstallerActivity.java#checkIfAllowedAndInitiateInstall

* 回到PackageInstallerActivity的onCreate方法的注释2处，checkIfAllowedAndInitiateInstall方法如下所示。


```JAVA
private void checkIfAllowedAndInitiateInstall() {
       //判断如果允许安装未知来源或者根据Intent判断得出该APK不是未知来源
       if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {//1
           //初始化安装
           initiateInstall();//2
           return;
       }
       // 如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面
       if (isUnknownSourcesDisallowed()) {//3
           if ((mUserManager.getUserRestrictionSource(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES,
                   Process.myUserHandle()) & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {    
               showDialogInner(DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER);
               return;
           } else {
               startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
               finish();
           }
       } else {
           handleUnknownSources();//4
       }
   }
```

* 注释1处判断允许安装未知来源或者根据Intent判断得出该APK不是未知来源
* 调用注释2处的initiateInstall方法来初始化安装。
* 注释3：如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面，否则就调用注释4处的handleUnknownSources方法来处理未知来源的APK。注释2处的initiateInstall方法如下所示。

#### 5.4 PackageInstallerActivity.java#initiateInstall

```java
private void initiateInstall() {
      String pkgName = mPkgInfo.packageName;//1
      String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
      if (oldName != null && oldName.length > 0 && oldName[0] != null) {
          pkgName = oldName[0];
          mPkgInfo.packageName = pkgName;
          mPkgInfo.applicationInfo.packageName = pkgName;
      }
      try {
          //根据包名获取应用程序信息
          mAppInfo = mPm.getApplicationInfo(pkgName,
                  PackageManager.MATCH_UNINSTALLED_PACKAGES);//2
          if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
              mAppInfo = null;
          }
      } catch (NameNotFoundException e) {
          mAppInfo = null;
      }
      //初始化安装确认界面
      startInstallConfirm();//3
  }
```

* 注释1处得到包名
* 注释2处根据包名获取获取应用程序信息ApplicationInfo。
* 注释3处的startInstallConfirm方法如下所示。

#### 5.5 PackageInstallerActivity.java#startInstallConfirm

```JAVA
private void startInstallConfirm() {
    //省略初始化界面代码
     ...
     AppSecurityPermissions perms = new AppSecurityPermissions(this, mPkgInfo);//1
     final int N = perms.getPermissionCount(AppSecurityPermissions.WHICH_ALL);
     if (mAppInfo != null) {
         msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                 ? R.string.install_confirm_question_update_system
                 : R.string.install_confirm_question_update;
         mScrollView = new CaffeinatedScrollView(this);
         mScrollView.setFillViewport(true);
         boolean newPermissionsFound = false;
         if (!supportsRuntimePermissions) {
             newPermissionsFound =
                     (perms.getPermissionCount(AppSecurityPermissions.WHICH_NEW) > 0);
             if (newPermissionsFound) {
                 permVisible = true;
                 mScrollView.addView(perms.getPermissionsView(
                         AppSecurityPermissions.WHICH_NEW));//2
             }
         }
     ...
 }
```

* startInstallConfirm方法中首先初始化安装确认界面，就是我们平常安装APK时出现的界面，界面上有确认和取消按钮并会列出安装该APK需要访问的系统权限。需要注意的是，不同厂商定制的Android系统会有不同的安装确认界面。
* 注释1处会创建AppSecurityPermissions，它会提取出APK中权限信息并展示出来，这个负责展示的View是AppSecurityPermissions的内部类PermissionItemView。
* 注释2处调用AppSecurityPermissions的getPermissionsView方法来获取PermissionItemView，并将PermissionItemView添加到CaffeinatedScrollView中，这样安装该APK需要访问的系统权限就可以全部的展示出来了，PackageInstaller的初始化工作就完成了。

### 6.总结

* 现在来总结下PackageInstaller初始化的过程：

1. 根据Uri的Scheme协议不同，跳转到不同的界面，content协议跳转到InstallStart，其他的跳转到PackageInstallerActivity。本文应用场景中，如果是Android7.0以及更高版本会跳转到InstallStart。
2. InstallStart将content协议的Uri转换为File协议，然后跳转到PackageInstallerActivity。
3. PackageInstallerActivity会分别对package协议和file协议的Uri进行处理，如果是file协议会解析APK文件得到包信息PackageInfo。
4. PackageInstallerActivity中会对未知来源进行处理，如果允许安装未知来源或者根据Intent判断得出该APK不是未知来源，就会初始化安装确认界面，如果管理员限制来自未知源的安装, 就弹出提示Dialog或者跳转到设置界面。

* PackageInstaller的初始化就讲到这，关于PackageInstaller的安装APK的过程会在本系列的下一篇文章进行讲解。