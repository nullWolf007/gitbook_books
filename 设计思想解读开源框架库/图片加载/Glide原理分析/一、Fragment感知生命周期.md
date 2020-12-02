[TOC]

## 一、Fragment感知生命周期

#### 参考

* [如何绑定页面生命周期（一）－Glide实现](https://juejin.cn/post/6844903647877267463)

### 一、前言

#### 1.1 感知特性

* Glide中一个重要的特性，就是Request可以随着Activity或Fragment的onStart而resume，onStop而pause，onDestroy而clear。从而节约流量和内存，并且防止内存泄露，这一切都由Glide在内部实现了。用户唯一要注意的是，Glide.with()方法中尽量传入Activity或Fragment，而不是Application，不然没办法进行生命周期管理。

### 二、总体实现原理

#### 2.1 概述

* 基于当前Activity添加无UI的Fragment，通过Fragment接收Activity传递的生命周期。Fragment和RequestManager基于Lifecycle建立联系，并传递生命周期事件，实现生命周期感知。分析上述的原理，可以归纳为两个方面：
  * 如何基于当前传入Activity生成无UI的Fragment，即如何实现对页面的周期绑定。
  * 无UI的fragment如何将生命周期传递给RequestManager，即如何实现生命周期传递。

### 三、如何绑定生命周期

#### 3.1 Glide#with

* 使用Glide时，我们通过`Glide.with(Activity activity)`的方式传入页面引用，让我们看下`with(Activity activity)`方法的实现:

```java
@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
	return getRetriever(activity).get(activity);
}
```

* 调用了getRetriever()对象的get方法，返回了一个RequestManager对象
* getRetriever()返回的是RequestManagerRetriever对象

#### 3.2 Glide#getRetriever

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
	// Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}
```

* 返回了一个RequestManagerRetriever对象

#### 3.3 RequestManagerRetriever#get

* Glide.with最终调用的是RequestManagerRetriever的get方法

```java
@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
    //当应用在后台，则直接绑定应用的生命周期。
	if (Util.isOnBackgroundThread()) {
		return get(activity.getApplicationContext());
	} else {
        assertNotDestroyed(activity);
        //获取当前activity的FragmentManager实例
        FragmentManager fm = activity.getSupportFragmentManager();
        return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
	}
}
```

* 当应用在后台，则直接绑定应用的生命周期。
* 主要代码还是else中的部分，首先获取当前activity的FragmentManager实例，然后调用了supportFragmentGet方法

#### 3.4 RequestManagerRetriever#supportFragmentGet

```java
@NonNull
private RequestManager supportFragmentGet(
	@NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
    //分析1
	SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    //分析2
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        //分析3
		Glide glide = Glide.get(context);
      	requestManager =factory.build(glide, current.getGlideLifecycle(), 
                                      current.getRequestManagerTreeNode(), context);
      	current.setRequestManager(requestManager);
    }
    return requestManager;
}
```

* 分析1：通过当前activity的FragmentManager对象，调用getSupportRequestManagerFragment创建一个基于当前activity的无UI的SupportRequestManagerFragment
* 分析2：获取当前SupportRequestManagerFragment对象的RequestManager实例
* 分析3：通过current.getGlideLifecycle()获取fragment的lifecycle，传入requestManager，将fragment和requestManager建立联系

#### 3.5 RequestManagerRetriever#getSupportRequestManagerFragment

* 在3.4的分析1中调用的getSupportRequestManagerFragment

```java
@NonNull
private SupportRequestManagerFragment getSupportRequestManagerFragment(
	@NonNull final FragmentManager fm, @Nullable Fragment parentHint, 
    boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      	current = pendingSupportRequestManagerFragments.get(fm);
      	if (current == null) {
            //分析1
        	current = new SupportRequestManagerFragment();
        	current.setParentFragmentHint(parentHint);
        	if (isParentVisible) {
          		current.getGlideLifecycle().onStart();
        	}
        	pendingSupportRequestManagerFragments.put(fm, current);
        	fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        	handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      	}
    }
    return current;
}
```

* 分析1：通过判断，最终调用了SupportRequestManagerFragment代码，创建SupportRequestManagerFragment实例
* 同时如果当前的Activity是visible的话，还会调用当前SupportRequestManagerFragment实例的getGlideLifecycle方法去获取ActivityFragmentLifecycle对象，然后调用此对象的onStart方法。

#### 3.6 SupportRequestManagerFragment#构造方法

```java
public SupportRequestManagerFragment() {
	this(new ActivityFragmentLifecycle());
}

@VisibleForTesting
@SuppressLint("ValidFragment")
public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
	this.lifecycle = lifecycle;
}
```

* 创建的时候还会调用new ActivityFragmentLifecycle()创建ActivityFragmentLifecycle对象，来初始化成员变量lifecycle

#### 3.7 RequestManagerFactory#build

* 在3.4中调用了factory的build方法

```java
requestManager =factory.build(glide, current.getGlideLifecycle(), 
                                      current.getRequestManagerTreeNode(), context);
```

* factory是RequestManagerFactory对象

* RequestManagerFactory是一个接口类，它在RequestManagerRetriever有一个实现的匿名内部类

```java
private static final RequestManagerFactory DEFAULT_FACTORY =
	new RequestManagerFactory() {
    	@NonNull
        @Override
        public RequestManager build(
        	@NonNull Glide glide,
            @NonNull Lifecycle lifecycle,
            @NonNull RequestManagerTreeNode requestManagerTreeNode,
            @NonNull Context context) {
          return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
        }
      };
```

```java
public RequestManagerRetriever(@Nullable RequestManagerFactory factory) {
    this.factory = factory != null ? factory : DEFAULT_FACTORY;
    handler = new Handler(Looper.getMainLooper(), this /* Callback */);
}
```

* factory的值要么是传入的factory，要么是DEFAULT_FACTORY。最终得到的都是RequestManager对象。
* 其中把current.getGlideLifecycle()的ActivityFragmentLifecycle对象作为参数传递了build方法得到的RequestManager对象，把两者关联了起来。

#### 3.8 RequestManager#构造函数

* 调用构造函数

```java
public RequestManager(
	@NonNull Glide glide,
    @NonNull Lifecycle lifecycle,
    @NonNull RequestManagerTreeNode treeNode,
    @NonNull Context context) {
    this(
        glide,
        lifecycle,
        treeNode,
        new RequestTracker(),
        glide.getConnectivityMonitorFactory(),
        context);
}
```

* 调用this方法，方法如下所示

```java
// Our usage is safe here.
@SuppressWarnings("PMD.ConstructorCallsOverridableMethod")
RequestManager(
	Glide glide,
    Lifecycle lifecycle,
    RequestManagerTreeNode treeNode,
    RequestTracker requestTracker,
    ConnectivityMonitorFactory factory,
    Context context) {
    this.glide = glide;
    this.lifecycle = lifecycle;
    this.treeNode = treeNode;
    this.requestTracker = requestTracker;
    this.context = context;

    connectivityMonitor =
        factory.build(
            context.getApplicationContext(),
            new RequestManagerConnectivityListener(requestTracker));

    // If we're the application level request manager, we may be created on a background thread.
    // In that case we cannot risk synchronously pausing or resuming requests, so we hack around the
    // issue by delaying adding ourselves as a lifecycle listener by posting to the main thread.
    // This should be entirely safe.
    if (Util.isOnBackgroundThread()) {
      	mainHandler.post(addSelfToLifecycle);
    } else {
      	lifecycle.addListener(this);
    }
    lifecycle.addListener(connectivityMonitor);

    defaultRequestListeners =
        new CopyOnWriteArrayList<>(glide.getGlideContext().getDefaultRequestListeners());
    setRequestOptions(glide.getGlideContext().getDefaultRequestOptions());

    glide.registerRequestManager(this);
  }
```

* 如果是Application级别的请求的话，创建一个background的thread
* 调用lifecycle.addListener(this)方法，将自己的引用存入lifecycle
* 同时会把此RequestManager放入到Glide的成员变量List<RequestManager> managers中去

### 四、如何传递生命周期

#### 4.1 概述

* 通过上面生命周期绑定的流程，我们已经知道通过ActivityFragmentLifecycle，将空白Fragment和RequestManager建立了联系。因为空白fragment注册在页面上，其可以感知页面的生命周期。下面我们来看下如何从空白fragment，将生命周期传递给RequestManager，从而对Request进行管理。

#### 4.2 SupportRequestManagerFragment的生命周期方法

##### 4.2.1 onStart

```java
@Override
public void onStart() {
	super.onStart();
    lifecycle.onStart();
}
```

* 调用lifecycle的onStart方法

##### 4.2.2 onStop

```java
@Override
public void onStop() {
    super.onStop();
    lifecycle.onStop();
}
```

* 调用lifecycle的onStop方法

##### 4.2.3 onDestroy

```java
@Override
public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
}
```

* 调用lifecycle的onDestroy方法

#### 4.3 ActivityFragmentLifecycle#onStart

```java
void onStart() {
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
}
```

* 调用lifecycleListener的onStart方法
* 因为RequestManager实现了lifeCycleListener接口。且在绑定阶段，在RequestManager的构造方法中，将RequestManager加入到了lifeCycle中。故回调lifeCycleListener中的相关方法，可以调用到它里面的对request生命周期进行管理的方法。由此，实现了Request对生命周期的感知。

```java
public class RequestManager
    implements ComponentCallbacks2, LifecycleListener, ModelTypes<RequestBuilder<Drawable>> {
    ......
}
```

#### 4.4 RequestManager#onStart

* 由于RequestManager是LifecycleListener的实现类，所以调用LifecycleListener的onStart方法就是调用RequestManager的onStart方法

```java
@Override
public synchronized void onStart() {
    resumeRequests();
    targetTracker.onStart();
}
```

* 调用了resumeRequests，这样就实现了Request可以随着Activity或Fragment的onStart而resume。

### 五、小结

#### 5.1 流程

* Activity的onXXX
* SupportRequestManagerFragment的onXXX
* ActivityFragmentLifecycle的onXXX
* RequestManager的onXXX
* requestTracker的resume/pause/clear和targetTracker的onXXX方法

#### 5.2 核心类介绍

* Glide：库提供对外调用方法的类，传入页面引用。

* RequestManagerRetriever：一个处理中间类，获取RequestManager和SupportRequestManagerFragment，并将两者绑定

* SupportRequestManagerFragment：无UI的fragment，与RequestManager绑定，感知并传递页面的生命周期

* RequestManager：实现了LifeCycleListener，主要作用为结合Activity或Fragment生命周期，对Request进行管理，如pauseRequests(), resumeRequests(), clearRequests()。

* LifecycleListener：接口，定义生命周期管理方法，onStart(), onStop(), onDestroy()。RequestManager实现了它。

* ActivityFragmentLifecycle：保存Fragment和RequestManager映射关系的类，管理LifecycleListener， 空白Fragment会回调它的onStart(), onStop(), onDestroy()。





