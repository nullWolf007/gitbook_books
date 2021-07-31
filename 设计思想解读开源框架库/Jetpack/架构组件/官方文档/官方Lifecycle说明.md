[TOC]

# 官方Lifecycle说明

#### 转载

* [Android Developer 文档](https://developer.android.google.cn/topic/libraries/architecture/lifecycle#lco)

## 一、前言

### 1.1 背景

* 生命周期感知型组件可执行操作来响应另一个组件（如 Activity 和 Fragment）的生命周期状态的变化。这些组件有助于您写出更有条理且往往更精简的代码，这样的代码更易于维护。

* 一种常见的模式是在 Activity 和 Fragment 的生命周期方法中实现依赖组件的操作。但是，这种模式会导致代码条理性很差而且会扩散错误。通过使用生命周期感知型组件，您可以将依赖组件的代码从生命周期方法移入组件本身中。

### 1.2 lifecycle

* [`androidx.lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/package-summary) 软件包提供了可用于构建生命周期感知型组件的类和接口 - 这些组件可以根据 Activity 或 Fragment 的当前生命周期状态自动调整其行为。

* **注意：**要将 [`androidx.lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/package-summary) 导入 Android 项目，请参阅 [Lifecycle 版本说明](https://developer.android.google.cn/jetpack/androidx/releases/lifecycle#declaring_dependencies)中关于声明依赖项的说明。

* 在 Android 框架中定义的大多数应用组件都存在生命周期。生命周期由操作系统或进程中运行的框架代码管理。它们是 Android 运作方式的核心，应用必须遵循它们。如果不这样做，可能会引发内存泄露甚至应用崩溃。

### 1.3 实例

* 假设我们有一个在屏幕上显示设备位置的 Activity。常见的实现可能如下所示：

```java
    class MyLocationListener {
        public MyLocationListener(Context context, Callback callback) {
            // ...
        }

        void start() {
            // connect to system location service
        }

        void stop() {
            // disconnect from system location service
        }
    }

    class MyActivity extends AppCompatActivity {
        private MyLocationListener myLocationListener;

        @Override
        public void onCreate(...) {
            myLocationListener = new MyLocationListener(this, (location) -> {
                // update UI
            });
        }

        @Override
        public void onStart() {
            super.onStart();
            myLocationListener.start();
            // manage other components that need to respond
            // to the activity lifecycle
        }

        @Override
        public void onStop() {
            super.onStop();
            myLocationListener.stop();
            // manage other components that need to respond
            // to the activity lifecycle
        }
    }
```

* 虽然此示例看起来没问题，但在真实的应用中，最终会有太多管理界面和其他组件的调用，以响应生命周期的当前状态。管理多个组件会在生命周期方法（如 `onStart()` 和 `onStop()`）中放置大量的代码，这使得它们难以维护。

* 此外，无法保证组件会在 Activity 或 Fragment 停止之前启动。在我们需要执行长时间运行的操作（如 `onStart()` 中的某种配置检查）时尤其如此。这可能会导致出现一种竞争条件，在这种条件下，`onStop()` 方法会在 `onStart()` 之前结束，这使得组件留存的时间比所需的时间要长。

```java
    class MyActivity extends AppCompatActivity {
        private MyLocationListener myLocationListener;

        public void onCreate(...) {
            myLocationListener = new MyLocationListener(this, location -> {
                // update UI
            });
        }

        @Override
        public void onStart() {
            super.onStart();
            Util.checkUserStatus(result -> {
                // what if this callback is invoked AFTER activity is stopped?
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

* [`androidx.lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/package-summary) 软件包提供的类和接口可帮助您以弹性和隔离的方式解决这些问题。

## 二、生命周期

### 2.1 概述

* [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 是一个类，用于存储有关组件（如 Activity 或 Fragment）的生命周期状态的信息，并允许其他对象观察此状态。

### 2.2 两种方法

* [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 使用两种主要枚举跟踪其关联组件的生命周期状态：

#### 2.2.1 事件

* 从框架和 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 类分派的生命周期事件。这些事件映射到 Activity 和 Fragment 中的回调事件。

#### 2.2.2 状态

* 由 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对象跟踪的组件的当前状态。

### 2.3 生命周期

![生命周期状态示意图](https://developer.android.google.cn/images/topic/libraries/architecture/lifecycle-states.svg)

* **图 1.** 构成 Android Activity 生命周期的状态和事件

* 您可以将状态看作图中的节点，将事件看作这些节点之间的边。

* 类可以通过向其方法添加注解来监控组件的生命周期状态。然后，您可以通过调用 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 类的 [`addObserver()`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle#addObserver(androidx.lifecycle.LifecycleObserver)) 方法并传递观察者的实例来添加观察者，如以下示例中所示：

```java
    public class MyObserver implements LifecycleObserver {
        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        public void connectListener() {
            ...
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        public void disconnectListener() {
            ...
        }
    }

    myLifecycleOwner.getLifecycle().addObserver(new MyObserver());
```

* 在上面的示例中，`myLifecycleOwner` 对象实现了 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 接口，我们将在接下来的部分中对该接口进行说明。

## 三、LifecycleOwner

* [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 是单一方法接口，表示类具有 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle)。它具有一种方法（即 [`getLifecycle()`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner#getLifecycle())），该方法必须由类实现。如果您尝试管理整个应用进程的生命周期，请参阅 [`ProcessLifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/ProcessLifecycleOwner)。

* 此接口从各个类（如 `Fragment` 和 `AppCompatActivity`）抽象化 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 的所有权，并允许编写与这些类搭配使用的组件。任何自定义应用类均可实现 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 接口。

* 实现 [`LifecycleObserver`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleObserver) 的组件可与实现 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 的组件无缝协同工作，因为所有者可以提供生命周期，而观察者可以注册以观察生命周期。

* 对于位置跟踪示例，我们可以让 `MyLocationListener` 类实现 [`LifecycleObserver`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleObserver)，然后在 `onCreate()` 方法中使用 Activity 的 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对其进行初始化。这样，`MyLocationListener` 类便可以“自给自足”，这意味着，对生命周期状态的变化做出响应的逻辑会在 `MyLocationListener`（而不是在 Activity）中进行声明。让各个组件存储自己的逻辑，可使 Activity 和 Fragment 逻辑更易于管理。

```java
    class MyActivity extends AppCompatActivity {
        private MyLocationListener myLocationListener;

        public void onCreate(...) {
            myLocationListener = new MyLocationListener(this, getLifecycle(), location -> {
                // update UI
            });
            Util.checkUserStatus(result -> {
                if (result) {
                    myLocationListener.enable();
                }
            });
      }
    }
```

* 一个常见的用例是，如果 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 现在未处于良好的状态，则应避免调用某些回调。例如，如果回调在 Activity 状态保存后运行 Fragment 事务，就会引发崩溃，因此我们绝不能调用该回调。

* 为简化此用例，[`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 类允许其他对象查询当前状态。

```java
    class MyLocationListener implements LifecycleObserver {
        private boolean enabled = false;
        public MyLocationListener(Context context, Lifecycle lifecycle, Callback callback) {
           ...
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_START)
        void start() {
            if (enabled) {
               // connect
            }
        }

        public void enable() {
            enabled = true;
            if (lifecycle.getCurrentState().isAtLeast(STARTED)) {
                // connect if not connected
            }
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
        void stop() {
            // disconnect if connected
        }
    }
```

* 对于此实现，`LocationListener` 类可以完全感知生命周期。如果我们需要从另一个 Activity 或 Fragment 使用 `LocationListener`，只需对其进行初始化。所有设置和拆解操作都由类本身管理。

* 如果库提供了需要使用 Android 生命周期的类，我们建议您使用生命周期感知型组件。库客户端可以轻松集成这些组件，而无需在客户端进行手动生命周期管理。

## 四、实现自定义 LifecycleOwner

* Support Library 26.1.0 及更高版本中的 Fragment 和 Activity 已实现 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner) 接口。

* 如果您有一个自定义类并希望使其成为 [`LifecycleOwner`](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleOwner)，您可以使用 [LifecycleRegistry](https://developer.android.google.cn/reference/androidx/lifecycle/LifecycleRegistry) 类，但需要将事件转发到该类，如以下代码示例中所示：

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

## 五、生命周期感知型组件的最佳做法

- 使界面控制器（Activity 和 Fragment）尽可能保持精简。它们不应试图获取自己的数据，而应使用 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 执行此操作，并观察 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 对象以将更改体现到视图中。
- 设法编写数据驱动型界面，对于此类界面，界面控制器的责任是随着数据更改而更新视图，或者将用户操作通知给 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel)。
- 将数据逻辑放在 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 类中。[`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 应充当界面控制器与应用其余部分之间的连接器。不过要注意，[`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 不负责获取数据（例如，从网络获取）。[`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 应调用相应的组件来获取数据，然后将结果提供给界面控制器。
- 使用 [Data Binding](https://developer.android.google.cn/topic/libraries/data-binding) 在视图与界面控制器之间维持干净的接口。这样一来，您可以使视图更具声明性，并尽量减少需要在 Activity 和 Fragment 中编写的更新代码。如果您更愿意使用 Java 编程语言执行此操作，请使用诸如 [Butter Knife](http://jakewharton.github.io/butterknife/) 之类的库，以避免样板代码并实现更好的抽象化。
- 如果界面很复杂，不妨考虑创建 [presenter](http://www.gwtproject.org/articles/mvp-architecture.html#presenter) 类来处理界面的修改。这可能是一项艰巨的任务，但这样做可使界面组件更易于测试。
- 避免在 [`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel) 中引用 `View` 或 `Activity` 上下文。如果 `ViewModel` 存在的时间比 Activity 更长（在配置更改的情况下），Activity 将泄露并且不会由垃圾回收器妥善处置。
- 使用 [Kotlin 协程](https://developer.android.google.cn/topic/libraries/architecture/coroutines)管理长时间运行的任务和其他可以异步运行的操作。

## 生命周期感知型组件的用例

生命周期感知型组件可使您在各种情况下更轻松地管理生命周期。下面列举几个例子：

- 在粗粒度和细粒度位置更新之间切换。使用生命周期感知型组件可在位置应用可见时启用细粒度位置更新，并在应用位于后台时切换到粗粒度更新。借助生命周期感知型组件 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData)，应用可以在用户使用位置发生变化时自动更新界面。
- 停止和开始视频缓冲。使用生命周期感知型组件可尽快开始视频缓冲，但会推迟播放，直到应用完全启动。此外，应用销毁后，您还可以使用生命周期感知型组件终止缓冲。
- 开始和停止网络连接。借助生命周期感知型组件，可在应用位于前台时启用网络数据的实时更新（流式传输），并在应用进入后台时自动暂停。
- 暂停和恢复动画可绘制资源。借助生命周期感知型组件，可在应用位于后台时暂停动画可绘制资源，并在应用位于前台后恢复可绘制资源。

## 处理 ON_STOP 事件

如果 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 属于 `AppCompatActivity` 或 `Fragment`，那么调用 `AppCompatActivity` 或 `Fragment` 的 `onSaveInstanceState()` 时，[`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 的状态会更改为 [`CREATED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#CREATED) 并且会分派 [`ON_STOP`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.Event#ON_STOP) 事件。

通过 `onSaveInstanceState()` 保存 `Fragment` 或 `AppCompatActivity` 的状态后，其界面被视为不可变，直到调用 [`ON_START`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.Event#ON_START)。如果在保存状态后尝试修改界面，很可能会导致应用的导航状态不一致，因此应用在保存状态后运行 `FragmentTransaction` 时，`FragmentManager` 会抛出异常。如需了解详情，请参阅 `commit()`。

[`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 本身可防止出现这种极端情况，方法是在其观察者的关联 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 还没有至少处于 [`STARTED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#STARTED) 状态时避免调用其观察者。在后台，它会在决定调用其观察者之前调用 [`isAtLeast()`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#isAtLeast(androidx.lifecycle.Lifecycle.State))。

遗憾的是，`AppCompatActivity` 的 `onStop()` 方法会在 `onSaveInstanceState()` 之后调用，这样就会留下一个缺口，即不允许界面状态发生变化，但 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 尚未移至 [`CREATED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#CREATED) 状态。

为防止出现这个问题，`beta2` 及更低版本中的 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 类会将状态标记为 [`CREATED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#CREATED) 而不分派事件，这样一来，即使未分派事件（直到系统调用 `onStop()`），检查当前状态的任何代码也会获得实际值。

遗憾的是，此解决方案有两个主要问题：

- 在 API 23 及更低级别，Android 系统实际上会保存 Activity 的状态，即使它的一部分被另一个 Activity 覆盖。换句话说，Android 系统会调用 `onSaveInstanceState()`，但不一定会调用 `onStop()`。这样可能会产生很长的时间间隔，在此时间间隔内，观察者仍认为生命周期处于活动状态，虽然无法修改其界面状态。
- 要向 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 类公开类似行为的任何类都必须实现由 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 版本 `beta 2` 及更低版本提供的解决方案。

**注意**：为了简化此流程并让其与较低版本实现更好的兼容性，自 `1.0.0-rc1` 版本起，当调用 `onSaveInstanceState()` 时，会将 [`Lifecycle`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle) 对象标记为 [`CREATED`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.State#CREATED) 并分派 [`ON_STOP`](https://developer.android.google.cn/reference/androidx/lifecycle/Lifecycle.Event#ON_STOP)，而不等待调用 `onStop()` 方法。这不太可能影响您的代码，但您需要注意这一点，因为它与 API 26 及更低级别的 `Activity` 类中的调用顺序不符。

## 其他资源

要详细了解如何使用生命周期感知型组件处理生命周期，请参阅下面列出的其他资源：