[TOC]

## Lifecycle解析

#### 转载参考

* [“终于懂了“系列：Jetpack AAC完整解析（一）Lifecycle 完全掌握！](https://juejin.cn/post/6893870636733890574)
* [Lifecycle官方文档](https://developer.android.com/topic/libraries/architecture/lifecycle)
* [Android Jetpack架构组件（三）一文带你了解Lifecycle（原理篇）](https://juejin.im/post/6844903977788817416)
* [Android架构组件（2）LifecycleRegistry 源码分析](https://blog.csdn.net/quiet_olivier/article/details/103384146)

## 一、Android Jetpack 介绍

### 1.1 Jetpack概述

* 官方定义如下:

> Jetpack 是一个由多个库组成的套件，可帮助开发者遵循最佳做法，减少样板代码并编写可在各种 Android 版本和设备中一致运行的代码，让开发者精力集中编写重要的代码。

* JetPack更多是一种概念和态度，它是谷歌开发的非Android Framework SDK自带、但同时是Android开发必备的/推荐的SDK/开发规范合集。相当于Google把自己的Android生态重新整理了一番，确立了Android未来的开发大方向。

* Jetpack原意为 喷气背包，Android背上Jetpack后就直冲云霄，这很形象了~

* 也就是，Jetpack是帮助开发者高效开发应用的工具集。那么这一工具包含了哪些内容呢？

### 1.2 Jetpack好处

- **遵循最佳做法**，Android Jetpack 组件采用最新的设计方法构建，具有向后兼容性，可以减少崩溃和内存泄露。
- **消除样板代码**，Android Jetpack 可以管理各种繁琐的 Activity（如后台任务、导航和生命周期管理），以便您可以专注于打造出色的应用。
- **减少不一致**，这些库可在各种 Android 版本和设备中以一致的方式运作，助您降低复杂性。

### 1.3 Jetpack分类

* 分类如下图（现在官网已经找不到这个图了）：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76302d9d9de643898fc539beb341af37~tplv-k3u1fbpfcp-watermark.image)

* Android Jetpack 组件覆盖以下 4 个方面：架构（Architecture）、基础（Foundation）、行为（Behavior） 、界面（UI）。

* 真正的精华主要是Architecture，全称是Android Architecture Component（AAC）， 即**Android架构组件**。

* 其包括比较成功的Lifecycle、LiveData、ViewModel，同时也是我们使用MVVM模式的最好框架工具，可以组合使用，也可以单独使用。

> 以上基本都是官网的介绍，我们主要目标就是掌握AAC的组件，深入理解进而运用到MVVM架构中。

* 如题，我们学习Jetpack的重点就是AAC，这篇就从基础的Lifecycle讲起。

## 二、Lifecycle

* Lifecycle，顾名思义，是用于帮助开发者管理Activity和Fragment 的生命周期，它是LiveData和ViewModel的基础。下面就先介绍为何及如何使用Lifecycle。

### 2.1 Lifecycle之前

* 官方文档有个例子 来说明使用Lifecycle之前是如何生命周期管理的：

* 假设我们有一个在屏幕上显示设备位置的 Activity。常见的实现可能如下所示：

```java
    class MyLocationListener {
        public MyLocationListener(Context context, Callback callback) {
            // ...
        }

        void start() {
            // 连接系统定位服务
        }

        void stop() {
            // 断开系统定位服务
        }
    }

    class MyActivity extends AppCompatActivity {
        private MyLocationListener myLocationListener;

        @Override
        public void onCreate(...) {
            myLocationListener = new MyLocationListener(this, (location) -> {
                // 更新 UI
            });
        }

        @Override
        public void onStart() {
            super.onStart();
            myLocationListener.start();
            // 管理其他需要响应activity生命周期的组件
        }

        @Override
        public void onStop() {
            super.onStop();
            myLocationListener.stop();
            // 管理其他需要响应activity生命周期的组件
        }
    }
```

* 虽然此示例看起来没问题，但在真实的应用中，最终会有太多管理界面和其他组件的调用，以响应生命周期的当前状态。管理多个组件会在生命周期方法（如 onStart() 和 onStop()）中放置大量的代码，这使得它们难以维护。

* 此外，无法保证组件会在 Activity 或 Fragment 停止之前启动myLocationListener。在我们需要执行长时间运行的操作（如 onStart() 中的某种配置检查）时尤其如此。在这种情况下，myLocationListener的onStop() 方法会在 onStart() 之前调用，这使得组件留存的时间比所需的时间要长，从而导致内次泄漏。如下：

```java
    class MyActivity extends AppCompatActivity {
        private MyLocationListener myLocationListener;

        public void onCreate(...) {
            myLocationListener = new MyLocationListener(this, location -> {
                // 更新 UI
            });
        }

        @Override
        public void onStart() {
            super.onStart();
            Util.checkUserStatus(result -> {
                //如果checkUserStatus耗时较长，在activity停止后才回调，那么myLocationListener启动后就没办法走stop()方法了，
                //又因为myLocationListener持有activity，所以会造成内存泄漏。
                if (result) {
                    myLocationListener.start();
                }
            });
        }

        @Override
        public void onStop() {
            super.onStop();
            myLocationListener.stop();
        }
    }
```

* 即2个问题点：
  * activity的生命周期内有大量管理组件的代码，难以维护。
  * 无法保证组件会在 Activity/Fragment停止后不执行启动

* Lifecycle库 则可以 以弹性和隔离的方式解决这些问题。

### 2.2 Lifecycle的使用

* Lifecycle是一个库，也包含Lifecycle这样一个类，Lifecycle类 用于存储有关组件（如 Activity 或 Fragment）的生命周期状态的信息，并允许其他对象观察此状态。

#### 2.2.1 引入依赖

##### **非androidX项目** 引入:

```groovy
implementation "android.arch.lifecycle:extensions:1.1.1"
```

* 添加这一句代码就依赖了如下的库:

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c389f0a56aba405facd1e457ceea3f76~tplv-k3u1fbpfcp-watermark.image)

##### **androidX项目** 引入:

* 如果项目已经依赖了AndroidX:

```java
implementation 'androidx.appcompat:appcompat:1.2.0'
```

* 那么我们就可以使用Lifecycle库了，因为appcompat依赖了androidx.fragment，而androidx.fragment下依赖了ViewModel和 LiveData，LiveData内部又依赖了Lifecycle。

* 如果想要单独引入依赖，则如下：

* 在项目根目录的build.gradle添加 google() 代码库，然后app的build.gradle引入依赖，官方给出的依赖如下：

```java
//根目录的 build.gradle
    repositories {
        google()
        ...
    }

	//app的build.gradle
    dependencies {
        def lifecycle_version = "2.2.0"
        def arch_version = "2.1.0"

        // ViewModel
        implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version"
        // LiveData
        implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
        // 只有Lifecycles (不带 ViewModel or LiveData)
        implementation "androidx.lifecycle:lifecycle-runtime:$lifecycle_version"
    
        // Saved state module for ViewModel
        implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:$lifecycle_version"

        // lifecycle注解处理器
        annotationProcessor "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
        // 替换 - 如果使用Java8,就用这个替换上面的lifecycle-compiler
        implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

		//以下按需引入
        // 可选 - 帮助实现Service的LifecycleOwner
        implementation "androidx.lifecycle:lifecycle-service:$lifecycle_version"
        // 可选 - ProcessLifecycleOwner给整个 app进程 提供一个lifecycle
        implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"
        // 可选 - ReactiveStreams support for LiveData
        implementation "androidx.lifecycle:lifecycle-reactivestreams:$lifecycle_version"
        // 可选 - Test helpers for LiveData
        testImplementation "androidx.arch.core:core-testing:$arch_version"
    }
```

* 看着有很多，实际上如果只使用Lifecycle，只需要引入lifecycle-runtime即可。但通常都是和 ViewModel、 LiveData 配套使用的，所以lifecycle-viewmodel、lifecycle-livedata 一般也会引入。

* 另外，lifecycle-process是给整个app进程提供一个lifecycle，后面也会提到。

#### 2.2.2 使用方法

- 1、生命周期拥有者 使用getLifecycle()获取Lifecycle实例，然后代用addObserve()添加观察者；
- 2、观察者实现LifecycleObserver，方法上使用OnLifecycleEvent注解关注对应生命周期，生命周期触发时就会执行对应方法；

##### 2.2.2.1 基本使用

* 在Activity（或Fragment）中 一般用法如下：

```java
public class LifecycleTestActivity extends AppCompatActivity {

    private String TAG = "Lifecycle_Test";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_lifecycle_test);
        //Lifecycle 生命周期
        getLifecycle().addObserver(new MyObserver());
        Log.i(TAG, "onCreate: ");
    }
    @Override
    protected void onResume() {
        super.onResume();
        Log.i(TAG, "onResume: ");
    }
    @Override
    protected void onPause() {
        super.onPause();
        Log.i(TAG, "onPause: ");
    }
}
```

* Activity（或Fragment）是生命周期的拥有者，通过getLifecycle()方法获取到生命周期Lifecycle对象，Lifecycle对象使用addObserver方法 给自己添加观察者，即MyObserver对象。当Lifecycle的生命周期发生变化时，MyObserver就可以感知到。

* MyObserver是如何使用生命周期的呢？看下MyObserver的实现：

```java
public class MyObserver implements LifecycleObserver {

    private String TAG = "Lifecycle_Test";
    
    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    public void connect(){
        Log.i(TAG, "connect: ");
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    public void disConnect(){
        Log.i(TAG, "disConnect: ");
    }
}
```

* 首先MyObserver实现了接口LifecycleObserver，LifecycleObserver用于标记一个类是生命周期观察者。 然后在connectListener()、disconnectListener()上 分别都加了@OnLifecycleEvent注解，且value分别是Lifecycle.Event.ON_RESUME、Lifecycle.Event.ON_PAUSE，这个效果就是：connectListener()会在ON_RESUME时执行，disconnectListener()会在ON_PAUSE时执行。

* 我们打开LifecycleTestActivity 然后退出，日志打印如下：

```java
2020-11-09 17:25:40.601 4822-4822/com.hfy.androidlearning I/Lifecycle_Test: onCreate: 

2020-11-09 17:25:40.605 4822-4822/com.hfy.androidlearning I/Lifecycle_Test: onResume: 
2020-11-09 17:25:40.605 4822-4822/com.hfy.androidlearning I/Lifecycle_Test: connect: 

2020-11-09 17:25:51.841 4822-4822/com.hfy.androidlearning I/Lifecycle_Test: disConnect: 
2020-11-09 17:25:51.841 4822-4822/com.hfy.androidlearning I/Lifecycle_Test: onPause: 
```

* 可见MyObserver的方法 确实是在对应关注的生命周期触发时调用。 当然注解中的value你也写成其它 你关注的任何一个生命周期，例如Lifecycle.Event.ON_DESTROY。

##### 2.2.2.2 MVP架构中的使用

* 如果是 **在MVP架构中**，那么就可以把presenter作为观察者：

```java
public class LifecycleTestActivity extends AppCompatActivity implements IView {
    private String TAG = "Lifecycle_Test";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_lifecycle_test);
        //Lifecycle 生命周期
//        getLifecycle().addObserver(new MyObserver());

        //MVP中使用Lifecycle
        getLifecycle().addObserver(new MyPresenter(this));
        Log.i(TAG, "onCreate: ");
    }

    @Override
    protected void onResume() {
        super.onResume();
        Log.i(TAG, "onResume: ");
    }
    @Override
    protected void onPause() {
        super.onPause();
        Log.i(TAG, "onPause: ");
    }

    @Override
    public void showView() {}
    @Override
    public void hideView() {}
}

//Presenter
class MyPresenter implements LifecycleObserver {
    private static final String TAG = "Lifecycle_Test";
    private final IView mView;

    public MyPresenter(IView view) {mView = view;}

    @OnLifecycleEvent(value = Lifecycle.Event.ON_START)
    private void getDataOnStart(LifecycleOwner owner){
        Log.i(TAG, "getDataOnStart: ");
        
        Util.checkUserStatus(result -> {
                //checkUserStatus是耗时操作，回调后检查当前生命周期状态
                if (owner.getLifecycle().getCurrentState().isAtLeast(STARTED)) {
                	start();
                    mView.showView();
                }
            });        
    }
    @OnLifecycleEvent(value = Lifecycle.Event.ON_STOP)
    private void hideDataOnStop(){
        Log.i(TAG, "hideDataOnStop: ");
        stop();
        mView.hideView();
    }
}

//IView
interface IView {
    void showView();
    void hideView();
}
```

* 这里是让Presenter实现LifecycleObserver接口，同样在方法上注解要触发的生命周期，最后在Activity中作为观察者添加到Lifecycle中。

* 这样做好处是啥呢？ 当Activity生命周期发生变化时，MyPresenter就可以感知并执行方法，不需要在MainActivity的多个生命周期方法中调用MyPresenter的方法了。
  * **所有方法调用操作都由组件本身管理**：Presenter类自动感知生命周期，如果需要在其他的Activity/Fragment也使用这个Presenter，只需添加其为观察者即可。
  * **让各个组件存储自己的逻辑，减轻Activity/Fragment中代码，更易于管理**；

* —— 上面提到的第一个问题点就解决了。

* 另外，注意到 getDataOnStart()中耗时校验回调后，对当前生命周期状态进行了检查：至少处于STARTED状态才会继续执行start()方法，也就是保证了Activity停止后不会走start()方法；

* —— 上面提到的第二个问题点也解决了。

#### 2.2.3 自定义LifecycleOwner

* 在Activity中调用getLifecycle()能获取到Lifecycle实例，那getLifecycle()是哪里定义的方法呢 ？是接口LifecycleOwner，顾名来思义，生命周期拥有者：

```java
/**
 * 生命周期拥有者
 * 生命周期事件可被 自定义的组件 用来 处理生命周期事件的变化，同时不会在Activity/Fragmen中写任何代码
 */
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

* Support Library 26.1.0及以上、AndroidX的 Fragment 和 Activity 已实现 LifecycleOwner 接口，所以我们在Activity中可以直接使用getLifecycle()。

* 如果有一个自定义类并希望使其成为LifecycleOwner，可以使用LifecycleRegistry类，它是Lifecycle的实现类，但需要将事件转发到该类：

```java
    public class MyActivity extends Activity implements LifecycleOwner {
        private LifecycleRegistry lifecycleRegistry;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            lifecycleRegistry = new LifecycleRegistry(this);
            lifecycleRegistry.markState(Lifecycle.State.CREATED);
        }
        @Override
        public void onStart() {
            super.onStart();
            lifecycleRegistry.markState(Lifecycle.State.STARTED);
        }
        @NonNull
        @Override
        public Lifecycle getLifecycle() {
            return lifecycleRegistry;
        }
    }
```

* MyActivity实现LifecycleOwner，getLifecycle()返回lifecycleRegistry实例。lifecycleRegistry实例则是在onCreate创建，并且在各个生命周期内调用markState()方法完成生命周期事件的传递。这就完成了LifecycleOwner的自定义，也即MyActivity变成了LifecycleOwner，然后就可以和 实现了LifecycleObserver的组件配合使用了。

* 补充一点，**观察者的方法可以接受一个参数LifecycleOwner**，就可以用来获取当前状态、或者继续添加观察者。 若注解的是ON_ANY还可以接收Event，用于区分是哪个事件。如下：

```java
    class TestObserver implements LifecycleObserver {
        @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
        void onCreated(LifecycleOwner owner) {
//            owner.getLifecycle().addObserver(anotherObserver);
//            owner.getLifecycle().getCurrentState();
        }
        @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
        void onAny(LifecycleOwner owner, Lifecycle.Event event) {
//            event.name()
        }
    }
```

### 2.3 Application生命周期 ProcessLifecycleOwner

* 之前对App进入前后台的判断是通过registerActivityLifecycleCallbacks(callback)方法，然后在callback中利用一个全局变量做计数，在onActivityStarted()中计数加1，在onActivityStopped方法中计数减1，从而判断前后台切换。

* 而使用ProcessLifecycleOwner可以直接获取应用前后台切换状态。（记得先引入lifecycle-process依赖）

* 使用方式和Activity中类似，只不过要使用ProcessLifecycleOwner.get()获取ProcessLifecycleOwner，代码如下：

```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

	//注册App生命周期观察者
        ProcessLifecycleOwner.get().getLifecycle().addObserver(new ApplicationLifecycleObserver());
    }
    
    /**
     * Application生命周期观察，提供整个应用进程的生命周期
     *
     * Lifecycle.Event.ON_CREATE只会分发一次，Lifecycle.Event.ON_DESTROY不会被分发。
     *
     * 第一个Activity进入时，ProcessLifecycleOwner将分派Lifecycle.Event.ON_START, Lifecycle.Event.ON_RESUME。
     * 而Lifecycle.Event.ON_PAUSE, Lifecycle.Event.ON_STOP，将在最后一个Activit退出后后延迟分发。如果由于配置更改而销毁并重新创建活动，则此延迟足以保证ProcessLifecycleOwner不会发送任何事件。
     *
     * 作用：监听应用程序进入前台或后台
     */
    private static class ApplicationLifecycleObserver implements LifecycleObserver {
        @OnLifecycleEvent(Lifecycle.Event.ON_START)
        private void onAppForeground() {
            Log.w(TAG, "ApplicationObserver: app moved to foreground");
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
        private void onAppBackground() {
            Log.w(TAG, "ApplicationObserver: app moved to background");
        }
    }
}
```

* 看到确实很简单，和前面Activity的Lifecycle用法几乎一样，而我们使用ProcessLifecycleOwner就显得很优雅了。 生命周期分发逻辑已在注释里说明。

## 三、 源码分析

* Lifecycle的使用很简单，接下来就是对Lifecycle原理和源码的解析了。

* 我们可以先猜下原理：LifecycleOwner（如Activity）在生命周期状态改变时（也就是生命周期方法执行时），遍历观察者，获取每个观察者的方法上的注解，如果注解是@OnLifecycleEvent且value是和生命周期状态一致，那么就执行这个方法。 这个猜测合理吧？下面你来看看。

### 3.1 Lifecycle类

* 先来瞅瞅Lifecycle：

```java
public abstract class Lifecycle {
    //添加观察者
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    //移除观察者
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    //获取当前状态
    public abstract State getCurrentState();

	//生命周期事件，对应Activity生命周期方法
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY  //可以响应任意一个事件
    }
    
    //生命周期状态. （Event是进入这种状态的事件）
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        //判断至少是某一状态
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

* Lifecycle 使用两种主要枚举跟踪其关联组件的生命周期状态：
  * Event，生命周期事件，这些事件对应Activity/Fragment生命周期方法。
  * State，生命周期状态，而Event是指进入一种状态的事件。

* Event触发的时机：
  * ON_CREATE、ON_START、ON_RESUME事件，是在LifecycleOwner对应的方法执行 之后 分发。
  * ON_PAUSE、ON_STOP、ON_DESTROY事件，是在LifecycleOwner对应的方法调用 之前 分发。

* 这保证了LifecycleOwner是在这个状态内。

* 官网有个图很清晰：

![构成 Android Activity 生命周期的状态和事件](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d73ca5de1494aff973f7aa166282794~tplv-k3u1fbpfcp-watermark.image)

### 3.2 Activity对LifecycleOwner的实现

* 前面提到Activity实现了LifecycleOwner，所以才能直接使用getLifecycle()，具体是在androidx.activity.ComponentActivity中:

```java
//androidx.activity.ComponentActivity，这里忽略了一些其他代码，我们只看Lifecycle相关
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner{
    ...
   
    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    ...
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ReportFragment.injectIfNeededIn(this); //使用ReportFragment分发生命周期事件
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
    @CallSuper
    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        Lifecycle lifecycle = getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
        }
        super.onSaveInstanceState(outState);
        mSavedStateRegistryController.performSave(outState);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

* 这里忽略了一些其他代码，我们只看Lifecycle相关。

* 看到ComponentActivity实现了接口LifecycleOwner，并在getLifecycle()返回了LifecycleRegistry实例。前面提到LifecycleRegistry是Lifecycle具体实现。

* 然后在onSaveInstanceState()中设置mLifecycleRegistry的状态为State.CREATED，然后怎么没有了？其他生命周期方法内咋没处理？what？和猜测的不一样啊。  别急，在onCreate()中有这么一行：**ReportFragment**.`injectIfNeededIn(this);`,这个就是关键所在。

### 3.3 生命周期事件分发——ReportFragment

```java
//专门用于分发生命周期事件的Fragment
public class ReportFragment extends Fragment {
    
    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            //在API 29及以上，可以直接注册回调 获取生命周期
            activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
        //API29以前，使用fragment 获取生命周期
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }

    @SuppressWarnings("deprecation")
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) {//这里废弃了，不用看
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                //使用LifecycleRegistry的handleLifecycleEvent方法处理事件
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatch(Lifecycle.Event.ON_CREATE);
    }
    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }
    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }
    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }
    ...省略onStop、onDestroy
    
    private void dispatch(@NonNull Lifecycle.Event event) {
        if (Build.VERSION.SDK_INT < 29) {
            dispatch(getActivity(), event);
        }
    }
    
    //在API 29及以上，使用的生命周期回调
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        ...
        @Override
        public void onActivityPostCreated(@NonNull Activity activity,@Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }
        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }
        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }
        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }
        ...省略onStop、onDestroy
    }
}
```

* 首先injectIfNeededIn()内进行了版本区分：在API 29及以上 直接使用activity的registerActivityLifecycleCallbacks 直接注册了生命周期回调，然后给当前activity添加了ReportFragment，注意这个fragment是没有布局的。

* 然后， 无论LifecycleCallbacks、还是fragment的生命周期方法 最后都走到了 dispatch(Activity activity, Lifecycle.Event event)方法，其内部使用LifecycleRegistry的handleLifecycleEvent方法处理事件。

* 而ReportFragment的作用就是获取生命周期而已，因为fragment生命周期是依附Activity的。好处就是把这部分逻辑抽离出来，实现activity的无侵入。如果你对图片加载库Glide比较熟，就会知道它也是使用透明Fragment获取生命周期的。

### 3.4 生命周期事件处理——LifecycleRegistry

* 到这里，生命中周期事件的处理有转移到了 **LifecycleRegistry** 中：

```java
//LifecycleRegistry.java
   //系统自定义的保存Observer的map，可在遍历中增删
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();
            
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);//获取event发生之后的将要处于的状态
        moveToState(next);//移动到这个状态
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;//如果和当前状态一致，不处理
        }
        mState = next; //赋值新状态
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            return;
        }
        mHandlingEvent = true;
        sync(); //把生命周期状态同步给所有观察者
        mHandlingEvent = false;
    }
    
        private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {  //isSynced()意思是 所有观察者都同步完了
            mNewEventOccurred = false;
            //mObserverMap就是 在activity中添加observer后 用于存放observer的map
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    ...
    
     static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
```

* 逻辑很清晰：使用getStateAfter()获取event发生之后的将要处于的状态（看前面那张图很好理解），moveToState()是移动到新状态，最后使用sync()把生命周期状态同步给所有观察者。

* 注意到sync()中有个while循环，很显然是在遍历观察者。并且很显然观察者是存放在mObserverMap中的，而mObserverMap对观察者的添加 很显然 就是 Activity中使用getLifecycle().addObserver()这里：

```java
//LifecycleRegistry.java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        //带状态的观察者，这个状态的作用：新的事件触发后 遍历通知所有观察者时，判断是否已经通知这个观察者了
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        //observer作为key，ObserverWithState作为value，存到mObserverMap

        if (previous != null) {
            return;//已经添加过，不处理
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            return;//lifecycleOwner退出了，不处理
        }
	//下面代码的逻辑：通过while循环，把新的观察者的状态 连续地 同步到最新状态mState。
    //意思就是：虽然可能添加的晚，但把之前的事件一个个分发给你(upEvent方法)，即粘性
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);//计算目标状态
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }
```

* 用observer创建带状态的观察者ObserverWithState，observer作为key、ObserverWithState作为value，存到mObserverMap。 接着做了安全判断，最后把新的观察者的状态 连续地 同步到最新状态mState，意思就是：虽然可能添加的晚，但会把之前的事件一个个分发给你，即粘性。

* 回到刚刚sync()的while循环，看看如何处理分发事件：

```java
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    
    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true; 
        }//最老的和最新的观察者的状态一致，都是ower的当前状态，说明已经同步完了
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }
    
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator = mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {//正向遍历，从老到新
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));//observer获取事件
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator = mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {//反向遍历，从新到老
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);//observer获取事件
                popParentState();
            }
        }
    }
```

* 循环条件是!isSynced()，若最老的和最新的观察者的状态一致，且都是ower的当前状态，说明已经同步完了。

* 没有同步完就进入循环体：
  * mState比最老观察者状态小，走backwardPass(lifecycleOwner)：从新到老分发，循环使用downEvent()和observer.dispatchEvent()，连续分发事件；
  * mState比最新观察者状态大，走forwardPass(lifecycleOwner)：从老到新分发，循环使用upEvent()和observer.dispatchEvent()，连续分发事件。

* 接着ObserverWithState类型的observer就获取到了事件，即observer.dispatchEvent(lifecycleOwner, event)，下面来看看它是如何让加了对应注解的方法执行的。

### 3.5 事件回调后 方法执行

* 我们继续看下 **ObserverWithState**:

```java
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

* mState的作用是：新的事件触发后 遍历通知所有观察者时，判断是否已经通知这个观察者了，即防止重复通知。

* mLifecycleObserver是使用Lifecycling.getCallback(observer)获取的GenericLifecycleObserver实例。GenericLifecycleObserver是接口，继承自LifecycleObserver：

```java
//接受生命周期改变并分发给真正的观察者
public interface LifecycleEventObserver extends LifecycleObserver {
    //生命周期状态变化
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

* 也就说，LifecycleEventObserver 给 LifecycleObserver 增加了感知生命周期状态变化的能力。

* 看看Lifecycling.getCallback(observer)：

```java
    @NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {
        ...省略很多类型判断的代码
        return new ReflectiveGenericLifecycleObserver(object);
    }
```

* 方法内有很多对observer进行类型判断的代码，我们这里关注的是ComponentActivity，所以LifecycleEventObserver的实现类就是ReflectiveGenericLifecycleObserver了：

```java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());//存放了event与加了注解方法的信息
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);//执行对应event的观察者的方法
    }
}
```

* 它的onStateChanged()方法内部使用CallbackInfo的invokeCallbacks方法，这里应该就是执行观察者的方法了。

* ClassesInfoCache内部用Map存了 所有观察者的回调信息，CallbackInfo是当前观察者的回调信息。

* 先看下CallbackInfo实例的创建，ClassesInfoCache.sInstance.getInfo(mWrapped.getClass())：

```java
//ClassesInfoCache.java
    private final Map<Class, CallbackInfo> mCallbackMap = new HashMap<>();//所有观察者的回调信息
    private final Map<Class, Boolean> mHasLifecycleMethods = new HashMap<>();//观察者是否有注解了生命周期的方法
    
    CallbackInfo getInfo(Class<?> klass) {
        CallbackInfo existing = mCallbackMap.get(klass);//如果已经存在当前观察者回调信息 直接取
        if (existing != null) {
            return existing;
        }
        existing = createInfo(klass, null);//没有就去收集信息并创建
        return existing;
    }
    
    private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
        Class<?> superclass = klass.getSuperclass();
        Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();//生命周期事件到来 对应的方法
        ...
        Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);//反射获取观察者的方法
        boolean hasLifecycleMethods = false;
        for (Method method : methods) {//遍历方法 找到注解OnLifecycleEvent
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation == null) {
                continue; //没有注解OnLifecycleEvent 就return
            }
            hasLifecycleMethods = true;//有注解OnLifecycleEvent
            Class<?>[] params = method.getParameterTypes(); //获取方法参数
            int callType = CALL_TYPE_NO_ARG;
            if (params.length > 0) { //有参数
                callType = CALL_TYPE_PROVIDER;
                if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                    throw new IllegalArgumentException(//第一个参数必须是LifecycleOwner
                            "invalid parameter type. Must be one and instanceof LifecycleOwner");
                }
            }
            Lifecycle.Event event = annotation.value();

            if (params.length > 1) {
                callType = CALL_TYPE_PROVIDER_WITH_EVENT;
                if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                    throw new IllegalArgumentException(//第二个参数必须是Event
                            "invalid parameter type. second arg must be an event");
                }
                if (event != Lifecycle.Event.ON_ANY) {
                    throw new IllegalArgumentException(//有两个参数 注解值只能是ON_ANY
                            "Second arg is supported only for ON_ANY value");
                }
            }
            if (params.length > 2) { //参数不能超过两个
                throw new IllegalArgumentException("cannot have more than 2 params");
            }
            MethodReference methodReference = new MethodReference(callType, method);
            verifyAndPutHandler(handlerToEvent, methodReference, event, klass);//校验方法并加入到map handlerToEvent 中
        }
        CallbackInfo info = new CallbackInfo(handlerToEvent);//获取的 所有注解生命周期的方法handlerToEvent，构造回调信息实例
        mCallbackMap.put(klass, info);//把当前观察者的回调信息存到ClassesInfoCache中
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);//记录 观察者是否有注解了生命周期的方法
        return info;
    }
```

- 如果不存在当前观察者回调信息，就使用createInfo()方法收集创建
- 先反射获取观察者的方法，遍历方法 找到注解了OnLifecycleEvent的方法，先对方法的参数进行了校验。
- 第一个参数必须是LifecycleOwner；第二个参数必须是Event；有两个参数 注解值只能是ON_ANY；参数不能超过两个
- 校验方法并加入到map，key是方法，value是Event。map handlerToEvent是所有的注解了生命周期的方法。
- 遍历完，然后用 handlerToEvent来构造 当前观察者回调信息CallbackInfo，存到ClassesInfoCache的mCallbackMap中，并记录 观察者是否有注解了生命周期的方法。

整体思路还是很清晰的，继续看CallbackInfo的invokeCallbacks方法：

```java
    static class CallbackInfo {
        final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;//Event对应的多个方法
        final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;//要回调的方法

        CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
            mHandlerToEvent = handlerToEvent;
            mEventToHandlers = new HashMap<>();
            //这里遍历mHandlerToEvent来获取mEventToHandlers
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
                Lifecycle.Event event = entry.getValue();
                List<MethodReference> methodReferences = mEventToHandlers.get(event);
                if (methodReferences == null) {
                    methodReferences = new ArrayList<>();
                    mEventToHandlers.put(event, methodReferences);
                }
                methodReferences.add(entry.getKey());
            }
        }

        @SuppressWarnings("ConstantConditions")
        void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
            invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);//执行对应event的方法
            invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,target);//执行注解了ON_ANY的方法
        }

        private static void invokeMethodsForEvent(List<MethodReference> handlers,
                LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
            if (handlers != null) {
                for (int i = handlers.size() - 1; i >= 0; i--) {//执行Event对应的多个方法
                    handlers.get(i).invokeCallback(source, event, mWrapped);
                }
            }
        }
    }
```

* 很好理解，执行对应event的方法、执行注解了ON_ANY的方法。其中mEventToHandlers是在创建CallbackInfo时由遍历mHandlerToEvent来获取，存放了每个Event对应的多个方法。

* 最后看看handlers.get(i).invokeCallback，即MethodReference中：

```java
    static class MethodReference {
        ...

        void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            try {
                switch (mCallType) {
                    case CALL_TYPE_NO_ARG:
                        mMethod.invoke(target);//没有参数的
                        break;
                    case CALL_TYPE_PROVIDER:
                        mMethod.invoke(target, source);//一个参数的：LifecycleOwner
                        break;
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                        mMethod.invoke(target, source, event);//两个参数的：LifecycleOwner，Event
                        break;
                }
            } 
           ...
        }
...
    }
```

* 根据不同参数类型，执行对应方法。

* 到这里，整个流程就完整了。实际看了这么一大圈，基本思路和我们的猜想是一致的。

* 这里借[Android Jetpack架构组件（三）一文带你了解Lifecycle（原理篇）](https://juejin.im/post/6844903977788817416)的图总结下：

![借刘望舒的图总结下](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfeed64e896c49ddb57b7ff1f1a099d1~tplv-k3u1fbpfcp-watermark.image)

## 四、总结

* 本文先介绍了Jetpack和AAC的概念，这是Android官方推荐的通用开发工具集。其中AAC是架构组件，是本系列文章的介绍内容。接着介绍了AAC的基础组件Lifecycle，它能让开发者更好的管理Activity/Fragment生命周期。最后详细分析了Lifecycle源码及原理。

* Jetpack的AAC是我们后续开发Android必备知识，也是完成MVVM架构的基础。Lifecycle更是AAC中的基础，所以完整掌握本篇内容十分必要。

