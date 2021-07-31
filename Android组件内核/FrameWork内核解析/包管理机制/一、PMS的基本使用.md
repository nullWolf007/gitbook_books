[TOC]

# PMS的基本使用

#### 转载

* [通过 PackageManager 获得你想要的 App 信息](https://juejin.cn/post/6844903504788602894)

## 一、通过PackageManager获取App信息

### 1. 前言

* 有些场景下，我们会需要获取一些其它 App 的各项信息，例如：App 名称，包名、Icon 等。这个时候就需要使用到 PackageManager 这个类了。
* 通过PackageManager 的使用，拿到各项 App 的信息，当然也包括一些未安装的 App 的信息。

### 2. 相关类

* 当你需要获取到指定 App 的各项信息的时候，你需要操作一些 Android 为我们提供的对应的 Api。

* 你首先需要获取 PackageManager（以下简称 PM） 对象，通过 PM 对象，你就可以获取到你需要的各项 App 的信息类。

* 这里涉及到的 App 信息类包括：PackageInfo、ApplicationInfo、ActivityInfo、ServiceInfo、ProviderInfo 等，还有一个 ResolveInfo 类，它比较特殊一点，不和前面的结构为从属关系。

* 这些类，都可以在根据 AndroidManifest.xml 中定义的组件进行划分，大概的结构如下。

  ![/manifest.png](https://user-gold-cdn.xitu.io/2017/10/19/7de6c948682813fde5e8ad1697fcc0f1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 可以看到，它们之间的关系还它挺复杂的。

* 一个 PackageInfo 对应一个 ApplicationInfo，而其中又包含若干个 ActivityInfo、ServiceInfo、ProviderInfo。

#### 2.1 PackageManager

* PackageManager 是一个抽象类，我们一般操作的 PackageManager ，实际上是它的实现类 ApplicationPackageManager 这个对象。

* 在 Context 中，就有获取 PM 对象的方法，`getPackageManager()`，所以四大组件想要获取它是非常简单的。

  ![/getPackageManager.png](https://user-gold-cdn.xitu.io/2017/10/19/bf67a1654b1bffe2f330e33e1f37c45a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* PM 中，提供了非常多的方法，供我们通过不同的条件，获取到 PackageInfo 对象、ApplicationInfo 对象等，它是本文的基础。

#### 2.2 PackageInfo

* PackageInfo 从名称上也可以看出来，它主要用于存储获取到的 Package 的一些信息，包括：包名（packageName）、版本号（versionCode）、版本名（versionName）。
* 基本上拿到了 PackageInfo ，你就可以拿到大部分此 Apk 相关的信息了。
* 并且，PackageInfo 中有一个 applicationInfo 的字段，是可以直接获取到与它相关的 ApplicationInfo 对象的。
* 这里介绍几个 PackageInfo 中，比较常用的字段：
  - packageName：包名。
  - versionCode：版本号
  - versionName：版本名。
  - firstInstallTime：首次安装时间。
  - lastUpdateTime：最后一次覆盖安装时间。

#### 2.3 ApplicationInfo

* ApplicationInfo 相对于 PackageInfo 用的会比较少一些。它主要用于获取 Apk 定义在 AndroidManifest.xml 中的一些信息。
* 这里介绍几个比较常用的：
  - packageName：包名
  - targetSdkVersion：目标 SDK 版本。
  - minSdkVersion：最小支持 SDK 版本，有 Api 限制，最低在 Api Level 24 及以上支持。
  - sourceDir：App 的 Apk 源文件存放的目录。
  - dataDir：data 目录的全路径。
  - metaData：Manifest 中定义的 meta 标签数据。
  - uid：当前 App 分配的 uid。
- 可以看到 ApplicationInfo 涵盖的信息，基本上都是在 AndroidManifest.xml 中定义的信息，并且有一些属性是有 Api Level 限制的，所以不确定的属性，提前看一下文档，确定是否全版本支持。

#### 2.4 ActivityInfo

* ActivityInfo、ServiceInfo、ProviderInfo 这三个是平级的，熟悉的一眼就能看出来，它们就是 Android 定义的四大组件中的几个。各自涵盖了一部分信息。一般在外部获取其他 App 的信息的时候，不会获取到这么细致的数据，如果有，看看这几个类准没错。

### 3. 基本操作

#### 3.1 获取所有安装的app

* 如果想要获取当前设备上已经安装的所有 App，可以使用 `getInstalledPackages()` 方法获取到它所有的已安装 App 的 PackageInfo 。

```java
   /**
     * 从设备中获取安装的app的PackageInfo
     *
     * @param context
     * @return
     */
    public static List<PackageInfo> getInstalledAppFromDevice(Context context) {
        PackageManager pm = context.getPackageManager();
        List<PackageInfo> packageInfoList = pm.getInstalledPackages(PackageManager.GET_ACTIVITIES);
        return packageInfoList;
    }
```

* PackageManager 中，很多方法都会需要传递一个 flags 的字段，它表示你当前需要获取到的 App 的信息。取值范围有挺多的，获取不同的信息使用不同的 Flags，通常如果没有额外的要求，直接使用 GET_ACTIVITYS 即可。

#### 3.2 判断App是否安装

* 这里主要说的是通过包名，判断 App 是否安装在当前设备上。
* 最简单的逻辑就是去获取 PackageInfo ，如果能拿回来数据，就说明是有安装的。

```java
    /**
     * 判断App是否已安装
     * @param context
     * @param packageName 对应app的包名
     * @return
     */
    public static boolean isInstalledApp(Context context, String packageName) {
        if (TextUtils.isEmpty(packageName)) {
            return false;
        }
        PackageManager pm = context.getPackageManager();

        try {
            PackageInfo packageInfo = pm.getPackageInfo(packageName, PackageManager.GET_ACTIVITIES);
            return packageInfo != null;
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return false;
    }
```

#### 3.3 通过包名获取PackageInfo

* 其实前面判断 App 是否安装的时候，就已经获取到了 PackageInfo 信息，这里只需要将它返回出去即可。

```java
    /**
     * 通过包名获取app的PackageInfo
     *
     * @param context
     * @param packageName app的包名
     * @return
     */
    public static PackageInfo getPackageInfoFromPackageName(Context context, String packageName) {
        PackageInfo packageInfo = null;
        if (TextUtils.isEmpty(packageName)) {
            return packageInfo;
        }
        PackageManager pm = context.getPackageManager();
        try {
            packageInfo = pm.getPackageInfo(packageName, PackageManager.GET_ACTIVITIES);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return packageInfo;
    }
```

#### 3.4 获取版本号和版本名

* 这里只需要根据 PackageInfo 中的字段，获取到对应的值就好了。

* 获取版本号

```java
    /**
     * 通过包名获取app的versionCode
     * @param context
     * @param packageName
     * @return 获取不到返回-1
     */
    public static int getVersionCodeFromPackageName(Context context, String packageName) {
        int res = -1;
        PackageInfo packageInfo = getPackageInfoFromPackageName(context, packageName);
        if (packageInfo != null) {
            res = packageInfo.versionCode;
        }
        return res;
    }
```

* 获取版本名

```java
    /**
     * 通过包名获取app的versionName
     *
     * @param context
     * @param packageName
     * @return 获取不到返回 null
     */
    public static String getVersionNameFromPackageName(Context context, String packageName) {
        String res = null;
        PackageInfo packageInfo = getPackageInfoFromPackageName(context, packageName);
        if (packageInfo != null) {
            res = packageInfo.versionName;
        }
        return res;
    }
```

#### 3.5 获取App的名称

* 获取 App 的名称，就需要用到 ApplicationInfo 这个对象了，其中 `loadLabel()` 方法返回的，就是 App 的名称。

```java
    /**
     * 通过包名获取app的name
     * @param context
     * @param packageName
     * @return
     */
    public static String getAppNameFromPackageName(Context context, String packageName) {
        String res = null;
        if (TextUtils.isEmpty(packageName)) {
            return res;
        }
        PackageManager pm = context.getPackageManager();
        PackageInfo packageInfo = getPackageInfoFromPackageName(context,packageName);
        if(packageInfo!=null){
            res = packageInfo.applicationInfo.loadLabel(pm).toString();
        }
        return res;
    }
```

#### 3.6 获取App的Icon

* 在 ApplicationInfo 中，还可以通过 `loadIcon()` 获取到 App 的 Icon。它返回的是一个 Drawable 对象，可以直接使用。

```java
    /**
     * 通过包名获取app的icon
     * @param context
     * @param packageName
     * @return
     */
    public static Drawable getAppIconFromPackageName(Context context, String packageName) {
        Drawable res = null;
        if (TextUtils.isEmpty(packageName)) {
            return res;
        }
        PackageManager pm = context.getPackageManager();
        PackageInfo packageInfo = getPackageInfoFromPackageName(context,packageName);
        if(packageInfo!=null){
            res = packageInfo.applicationInfo.loadIcon(pm);
        }
        return res;
    }
```

#### 3.7 根据Apk文件，获取PackageInfo

* 前面介绍的方法，都是基于一个已安装的 App 的包名，来获取额外的信息。

* 但是有时候，我们只有一个未安装的 Apk 文件，想要解析出 Apk 文件中的额外信息，PM 中，也有对应的 Api。非常的方便，直接使用 `getPackageArchiveInfo()` 即可。

```java
    /**
     * 通过Apk的路径，获取PackageInfo
     *
     * @param context
     * @param apkPath
     * @return
     */
    public static PackageInfo getPackageInfoFromApkPath(Context context, String apkPath) {
        PackageInfo packageInfo = context.getPackageManager().getPackageArchiveInfo(apkPath, PackageManager.GET_ACTIVITIES);
        return packageInfo;
    }
```

* 只要拿到这个 Apk 文件相关的 PackageInfo 信息，就有办法拿到其他的名称、icon 、版本号、版本名、包名等信息。和前面介绍的例子类似，这里就不再一一列举了。