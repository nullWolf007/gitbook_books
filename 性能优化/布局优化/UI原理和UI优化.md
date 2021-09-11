[TOC]

# UI原理UI优化

#### 转载

* [总结UI原理和高级的UI优化方式](https://juejin.cn/post/6844903974294781965)

### 前言

* 相信大家多多少少看过一些Activity启动源码分析的文章，也能大概说出**Activity启动流程**，例如这种回答：

> AMS负责管理系统所有Activity，所以应用startActivity 最终会通过Binder调用到AMS的startActivity方法，AMS启动一个Activity之前会做一些检查，例如权限、是否在清单文件注册等，然后就可以启动了，AMS是一个系统服务，在单独进程，所以要将生命周期告诉应用，又涉及到跨进程调用，这个跨进程同样采用Binder，媒介是通过ActivityThread的内部类ApplicationThread，AMS将生命周期跨进程传到ApplicationThread，然后ApplicationThread 再分发给ActivityThread内部的Handler，这时候生命周期已经回调到应用主线程了，回调Activity的各个生命周期方法。

* 还可以细分，比如**Activity、Window、DecorView之间的关系**，这个其实也应该难度不大，又突然想到，**setContentView为什么要放在onCreate中？**，放在其它方法里行不行，能不能放在onAttachBaseContext方法里？其实，这些问题可以在源码中找到答案。

* 本文参考Android 9.0源码，API 28。

* 写一个Activity，我们一般都是通过在onCreate方法中调用setContentView方法设置我们的布局，有没有想过这个问题，**设置完布局，界面就会开始绘制吗？**

## 一、从生命周期源码分析UI原理

* [根Activity启动过程](../../Android组件内核/FrameWork内核解析/根Activity启动过程.md) 
* 我们取出其中ActivityThread启动Activity过程进行分析 
* ![ActivityThread启动Activity过程](..\..\images\Android组件内核\四大组件\Activity\ActivityThread启动Activity过程.png)

### 1.1 ActivityThread

#### 1.1.1 内部类 ApplicationThread

* AMS暂且先不分析，AMS启动Activity会通过ApplicationThread通知到ActivityThread，启动Activity从ApplicationThread开始说起，看下 scheduleLaunchActivity 方法

#### 1.1.2 ApplicationThread#scheduleLaunchActivity

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        ActivityClientRecord r = new ActivityClientRecord();
        r.token = token;
        r.ident = ident;
        r.intent = intent;
        ...
        sendMessage(H.LAUNCH_ACTIVITY, r);
    }
```

* sendMessage 最终封装一个ActivityClientRecord对象到msg，调用`mH.sendMessage(msg);`,mH 是一个Handler，直接看处理部分吧，

### 1.2 ActivityThread的内部类H

```java
private class H extends Handler {

	public void handleMessage(Message msg) {
		switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
				//1、从msg.obj获取ActivityClientRecord 对象，调用handleLaunchActivity 处理消息
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
			...

	}


}
```

* 上面是Handler很基础的东西，应该都能看懂，注释1，启动Activity直接进入 `handleLaunchActivity` 方法

### 1.3 ActivityThread#handleLaunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    // Initialize before creating the activity //初始化WindowManagerGlobal
    WindowManagerGlobal.initialize();

    //1. 启动一个Activity，涉及到创建Activity对象，最终返回Activity对象
    Activity a = Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
	    //2. Activity 进入onResume方法
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        if (!r.activity.mFinished && r.startsNotResumed) {
	        //3. 如果没能在前台显示，就进入onPuse方法
            performPauseActivityIfNeeded(r, reason);

        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
	    //4. activity启动失败，则通知AMS finish掉这个Activity
        try {
            ActivityManagerNative.getDefault()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

* 注释1：performLaunchActivity，开始启动Activity了
* 注释2：Activity进入Resume状态，handleResumeActivity
* 注释3：如果没能在前台显示，那么进入pause状态，performPauseActivityIfNeeded
* 注释4，如果启动失败，通知AMS去finishActivity。

* 主要看performLaunchActivity（1.3.1） 和 handleResumeActivity（1.3.2）

#### 1.3.1 ActivityThread#performLaunchActivity

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

	//1 创建Activity对象
	activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);

	//2 调用Activity的attach方法
	activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);

	//3.回调onCreate
	mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
	
	//4.回调onStart
	activity.performStart();
	
	
	if (r.state != null || r.persistentState != null) {
	//5.如果有保存状态，则调用onRestoreInstanceState 方法，例如Activity被异常杀死，重写onSaveInstanceState保存的一些状态
        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
    }

	//6.回调 onPostCreate，这个方法基本没用过
	mInstrumentation.callActivityOnPostCreate(activity, r.state);

}
```

* 从这里可以看出Activity几个方法调用顺序：

1. Activity#attach
2. Activity#onCreate
3. Activity#onStart
4. Activity#nnRestoreInstanceState
5. Activity#onPostCreate

* 我们主要来分析 attach 方法 和 最熟悉的onCreate方法

##### 1.Activity#attach

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
	//1、回调 attachBaseContext
    attachBaseContext(context);

	//2、创建PhoneWindow
    mWindow = new PhoneWindow(this, window);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    ...
    
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
```

* attach方法主要关注两点：
  1.会调用attachBaseContext 方法；
  2.创建PhoneWindow，赋值给mWindow，这个后面会经常遇到。

##### 2. Activity#onCreate

* 大家有没有思考过，为什么 setContentView 要放在onCreate方法中？
* 不能放在onStart、onResume中我们大概是知道的，因为按Home键再切回来会回调生命周期onStart、onResume，setContentView多次调用是没必要的
* 那如果是放在attachBaseContext 里面行不行？
* 看下 setContentView 里面的逻辑

##### 3. Activity#setContentView

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

public Window getWindow() {
    return mWindow;
}
```

* getWindow 返回的是mWindow，mWindow是在 Activity 的 attach 方法初始化的，上面刚刚分析过attach方法，**mWindow是一个PhoneWindow对象**。所以setContentView 必须放在attachBaseContext之后。

##### 4. PhoneWindow#setContentView

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
		//1、 创建DecorView
        installDecor(); 
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
		//2、mContentParent是DecorView中的FrameLayout，将我们的布局添加到这个FrameLayout里
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
}
```

* installDecor 方法是创建DecorView，看一下主要代码

* 注释1：installDecor，创建根布局 DecorView，
* 注释2：mLayoutInflater.inflate(layoutResID, mContentParent); 将xml布局渲染到mContentParent里，第二节会重点分析LayoutInflater原理。

* 先看注释1

##### 5. PhoneWindow#installDecor

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
		//1.创建DecorView
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
		// 2. mDecor 不为空，就是创建过，只需设置window
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
		//3.找到DecorView 内容部分，findViewById(ID_ANDROID_CONTENT)
        mContentParent = generateLayout(mDecor);

        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeOptionalFitsSystemWindows();

		//4.找到DecorView的根View，给标题栏部分，设置图标和标题啥的
        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

		//5.设置标题图标啥的
        if (decorContentParent != null) {
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            if (mDecorContentParent.getTitle() == null) {
                mDecorContentParent.setWindowTitle(mTitle);
            }

            ...
            mDecorContentParent.setUiOptions(mUiOptions);

            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                    (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                mDecorContentParent.setIcon(mIconRes);
            } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                    mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                mDecorContentParent.setIcon(
                        getContext().getPackageManager().getDefaultActivityIcon());
                mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
            }
            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                    (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                mDecorContentParent.setLogo(mLogoRes);
            }

            ...
        } 
        
		...
    }
}
```

* installDecor 主要做了两件事，一个是创建DecorView，一个是根据主题，填充DecorView中的一些属性，比如默认就是标题栏+内容部分

* 先看注释1：generateDecor 创建DecorView

###### 6. PhoneWindow#generateDecor

```java
protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```

###### 7. DecorView 构造方法

```java
DecorView(Context context, int featureId, PhoneWindow window,
        WindowManager.LayoutParams params) {
    super(context);
    ...

    updateAvailableWidth();

    //设置window
    setWindow(window);
}
```

* 创建DecorView传了一个PhoneWindow进去 。

* installDecor 方法里面注释2，如果判断DecorView已经创建过的情况下，直接调用`mDecor.setWindow(this);`

* 再看 installDecor 方法注释3

```java
if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
		...
```

* 给 mContentParent 赋值，看下 generateLayout 方法

##### 8. PhoneWindow#generateLayout

```java
 protected ViewGroup generateLayout(DecorView decor) {

    // 1.窗口各种属性设置，例如 requestFeature(FEATURE_NO_TITLE);
	...
    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    }
	...
	if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
        setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
    }
	...
	
	//2.获取 windowBackground 属性，默认窗口背景
	if (mBackgroundDrawable == null) {
            if (mBackgroundResource == 0) {
                mBackgroundResource = a.getResourceId(
                        R.styleable.Window_windowBackground, 0);
            }
			...
	}

	...
	// 3.获取DecorView里面id为 R.id.content的布局
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	...
	if (getContainer() == null) {
		//4.在注释2的时候已经获取了窗口背景属性id，这里转换成drawable，并且设置给DecorView
        final Drawable background;
        if (mBackgroundResource != 0) {
            background = getContext().getDrawable(mBackgroundResource);
        } else {
            background = mBackgroundDrawable;
        }
        mDecor.setWindowBackground(background);

		

	}


	return contentParent;

}
```

* 注释1： 首先是窗口属性设置，例如我们的主题属性设置了没有标题，则走：`requestFeature(FEATURE_NO_TITLE);`, 全屏则走`setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));`

* 还有其他很多属性判断，这里只列出两个代表性的，为什么说代表性呢，因为我我们平时要让Activity全屏，去掉标题栏，可以在主题里设置对应属性，还可以通过代码设置，也就是在onCreate方法里setContentView之前调用

```java
getWindow().requestFeature(Window.FEATURE_NO_TITLE);
getWindow().setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN );
```

* 注释2：获取窗口背景属性id，这个id就是style.xml 里面我们定义的主题下的一个属性

```xml
<style name="AppThemeWelcome" parent="Theme.AppCompat.Light.DarkActionBar">
	<item name="colorPrimary">@color/colorPrimary</item>
    ...

    <item name="android:windowBackground">@mipmap/logo</item> //就是这个窗口背景属性

</style>
```

* 注释3，这个ID_ANDROID_CONTENT 定义在Window中，
  `public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;` 找到id对应的ViewGroup叫contentParent，最终是要返回回去。

* 注释4：在注释2的时候已经获取了窗口背景属性id，这里转换成drawable，并且设置给DecorView，DecorView是一个FrameLayout，**所以我们在主题中设置的默认背景图，最终是设置给DecorView**

* generateLayout 方法分析完了，主要做的事情是读取主题里配置的一堆属性，然后给DecorView 设置一些属性，例如背景，还有获取DecorView 里的内容布局，作为方法返回值。

* 回到 installDecor 方法注释4，填充DecorContentParent 内容

##### 9. 填充 DecorContentParent

```java
//这个是DecorView的外层布局，也就是titlebar那一层
final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
	
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            
			//有该属性的话就进行相关设置，例如标题、图标
			mDecorContentParent.setWindowTitle(mTitle);
			...
			mDecorContentParent.setIcon(mIconRes);
			...
			mDecorContentParent.setIcon(
                        getContext().getPackageManager().getDefaultActivityIcon());
			...
			mDecorContentParent.setLogo(mLogoRes);
			...
```

* 这是installDecor 的最后一步，设置标题、图标等属性。

* setContentView如果发现DecorView不存在则创建DecorView，现在DecorView 创建完成，那么要将我们写的布局添加到DecorView中去了，怎么做呢，看 `PhonwWPhonwWindowindow#setContentView` 方法的注释2，`mLayoutInflater.inflate(layoutResID, mContentParent);`，这个mContentParent 上面已经分析过了，是 generateLayout 方法返回的，就是DecorView里面id为content 的布局。

* 所以接下来的一步`LayoutInflater.inflate`就是将我们写的布局添加到DecorView的内容部分，怎么添加？这个就涉及到解析xml，转换成对应的View对象了，怎么转换？

* LayoutInflater 原理放在第二节，这样章节比较清晰。 `mLayoutInflater.inflate(layoutResID, mContentParent);`执行完，xml布局就被解析成View对象，添加到DecorView中去了。画一张图



![Activity、Window、DecorView层级关系](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de98108940f2a6~tplv-t2oaga2asx-watermark.awebp)



* 此时DecorView并不能显示到屏幕，因为PhoneWindow并没有被使用，只是准备好了而已。1.3 继续分析第二点

#### 1.3.2 ActivityThread#handleResumeActivity

```java
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
	ActivityClientRecord r = mActivities.get(token);	        
	...
	//1、先回调Activity的onResume方法
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;

		...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
			//2、wm是当前Activity的WindowManager
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            ...
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
				// 3.添加decor到WindowManager
                wm.addView(decor, l);
            }
```

* 主要关注的就是上面的代码，
* 注释1：performResumeActivity，主要是回调Activity的onResume方法。
* 注释2： 从Activity开始跟踪这个wm，其实是`WindowManagerImpl`,
* 注释3： 调用WindowManagerImpl的`addView`方法，最终调用的是`WindowManagerGlobal`的addView方法，将DecorView添加到 WindowManagerGlobal

* 从这个顺序可以看出，**在Activity的onResume方法回调之后，才会将decorView添加到window**，看下addView 的逻辑

##### 1 WindowManagerImpl#addView

```java
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
...    
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

* 看 `WindowManagerGlobal#addView`

##### 2 WindowManagerGlobal#addView

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {


	ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ... 
		//1、创建 ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

		//2、添加到ArrayList进行管理
        mViews.add(view);
		//3.mRoots 存放ViewRootImpl对象
        mRoots.add(root);
        mParams.add(wparams);
    }

    // do this last because it fires off messages to start doing things
    try {
		// 4.调用ViewRootImpl的setView
        root.setView(view, wparams, panelParentView);
    }
```

* 入参view是`DecorView`，
* 注释1：创建 ViewRootImpl，
* 注释2：将decorView添加进到ViewRootImpl中去，ViewRootImpl 和 DecorView 的关系建立。
* 注释3：将ViewRootImpl保存到 ArrayList
* 注释4：**调用ViewRootImpl的setView 方法，这个很关键。**

* ViewRootImpl的setView 方法，第一个参数是DecorView，第二个参数是PhoneWindow的属性，第三个参数是父窗口，如果当前是子窗口，那么需要传父窗口，否则传null，默认null即可。

* 看晕了没，重新梳理一下resume的整个流程：

1. ActivityThread 调用 handleResumeActivity（这个方法源头是AMS回调的）
2. 回调 Activity 的 onResume
3. 将 DecorView添加到 WindowManagerGlobal
4. **创建 ViewRootImpl，调用setView方法**，跟DecorView绑定

* 接下来主要关注 ViewRootImpl 的setView方法

### 1.4 ViewRootImpl 分析

* 看下构造方法

#### 1.4.1 ViewRootImpl 构造

```java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    //1、WindowSession 是IWindowSession，binder对象，可跨进程跟WMS通信
    mWindowSession = WindowManagerGlobal.getWindowSession();
    mDisplay = display;
    ...
    // 2、创建window，是一个binder对象
    mWindow = new W(this);
    ...
    //3、mAttachInfo 很熟悉吧，在这里创建的，View.post跟这个息息相关
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
    ...
    //4、Choreographer在这里初始化
    mChoreographer = Choreographer.getInstance();
    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
    loadSystemProperties();
}
```

* 注释1：创建 mWindowSession，这是一个Binder对象，可以跟WMS通信。
* 注释2：mWindow 实例化，主要是接收窗口状态改变的通知。
* 注释3：mAttachInfo 创建，view.post(runable) 能获取宽高的问题跟这个mAttachInfo的创建有关。
* 注释4：mChoreographer 初始化，之前文章讲过 Choreographer，不再重复讲。

* mWindowSession 的创建简单来看一下，因为接下来很多地方都要用到它来跟WMS通信。

#### 1.4.2 mWindowSession 是干嘛的

```java
mWindowSession = WindowManagerGlobal.getWindowSession();
```

* 看下WindowManagerGlobal 是怎么创建Session的

##### 1. WindowManagerGlobal.getWindowSession()

```java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
				// 这里获取的是WMS的Binder代理
                IWindowManager windowManager = getWindowManagerService();
				// 通过代理调用 WMS 的openSession 方法返回一个 Session（也是一个Binder代理）
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

* 拿到WMS的binder代理，通过代理调用WMS 的 openSession 方法。看下WMS的 openSession 方法

##### 2. WindowManagerService#openSession

```java
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
	//创建一个 Session，把WMS传给Session，Session就有WMS的引用
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

* 创建一个Session返回， 它也是一个Binder对象，拥有跨进程传输能力

##### 3. Session

```java
final class Session extends IWindowSession.Stub
    implements IBinder.DeathRecipient {
```

* 通过分析，**mWindowSession 是一个 拥有跨进程能力的 Session 对象，拥有WMS的引用，后面ViewRootImpl跟WMS交互都是通过这个mWindowSession**。

* 看下ViewRootImpl的setView方法

#### 1.4.2 ViewRootImpl#setView

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
	    	// 1.DecorView赋值给mView
            mView = view;
            ...
	    	//2.会调用requestLayout
            requestLayout();
            ...
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
				//3.WindowSession,将window添加到屏幕，有返回值
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            } 
            ...
	    	//4.添加window失败处理
            if (res < WindowManagerGlobal.ADD_OKAY) {
                ...
                switch (res) {
                    case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                    case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not valid; is your activity running?");
                    case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not for an application");
                    case WindowManagerGlobal.ADD_APP_EXITING:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- app for token " + attrs.token
                                + " is exiting");
                    case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- window " + mWindow
                                + " has already been added");
                    case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                        // Silently ignore -- we would have just removed it
                        // right away, anyway.
                        return;
                    case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- another window of type "
                                + mWindowAttributes.type + " already exists");
                    case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- permission denied for window type "
                                + mWindowAttributes.type);
                    case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified display can not be found");
                    case WindowManagerGlobal.ADD_INVALID_TYPE:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified window type "
                                + mWindowAttributes.type + " is not valid");
                }
                throw new RuntimeException(
                        "Unable to add window -- unknown error code " + res);
            }

            ...
			//5.这里，给DecorView设置parent为ViewRootImpl
            view.assignParent(this);
            ...
        }
    }
}
```

* setView 整理如下：
  注释1. 给mView赋值为DecorView
  注释2. 调用 requestLayout，主动发起绘制请求
  注释3. WindowSession 是一个Binder对象，mWindowSession.addToDisplay(...)，将PhoneWindow添加到WMS去了
  注释4. 添加window失败的处理
  注释5： view.assignParent(this); 给DecorView设置parent为ViewRootimpl

* 注释2调用requestLayout，会请求vsyn信号，然后下一次vsync信号来的时候，会调用 performTraversals 方法，这个上一篇文章分析过了，performTraversals 主要是执行View的measure、layout、draw。

* 注释3，mWindowSession.addToDisplay 这一句要重点分析，

##### 1.4.2.1 mWindowSession.addToDisplay

* mWindowSession 的创建上面已经分析过了。

##### 1.4.2.2 Session#addToDisplay

```java
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
	//调用WMS 的 addWindow
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

* Session里面有WMS的应用，这里可以看到 addToDisplay 方法就是调用了 WMS 的 addWindow 方法。

##### 1.4.2.3 WindowManagerService#addWindow

* WMS 的 addWindow 方法代码比较多，不打算详细分析，内部涉及到窗口**权限校验、token校验、创建窗口对象、添加窗口到Windows列表、确定窗口位置**等。读者可以自行查看源码。

* 到这里可以知道的是，ViewRootImpl跟WMS通信的媒介就是Session:

**ViewRootImpl -> Session -> WMS**

* 回来ViewRootImpl的 setView 方法，注释2调用 requestLayout方法，最终会请求vsync信号，然后等下一个vsync信号来临，就会通过Handler执行performTraversals方法。所以按照执行顺序，performTraversals方法是在添加Window之后才调用。

#### 1.5 ViewRootImpl#performTraversals

```java
private void performTraversals() {

	... // 1、relayoutWindow 方法是将mSurface跟WMS关联
	relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
	 ...
	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	...
	performLayout(lp, mWidth, mHeight);
	 ...
	performDraw();
}
```

* performTraversals 里面执行View的三个方法，这个大家可能都知道，可是怎么跟屏幕关联上的呢，performDraw 方法里面能找到答案~

* 看下performDraw

```java
private void performDraw() {      
        draw(fullRedrawNeeded);
...
}


private boolean draw(boolean fullRedrawNeeded) {
        //这个 mSurface是在定义的时候new的 
        Surface surface = mSurface;
        // 只分析软件绘制
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
         }
        return useAsyncReport;
    }
```

1. mSurface 在定义的时候就new了`public final Surface mSurface = new Surface();`，但是只是一个空壳，其实在 performTraversals 的注释1里面做了一件事，调用 relayoutWindow 这个方法

##### relayoutWindow

```java
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {

        ...

        int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                mPendingMergedConfiguration, mSurface);

        return relayoutResult;
    }
```

* 里面调用了mWindowSession.relayout 方法，将mSurface传过去，直接看Session的relayout 方法

```java
public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
            Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
            Surface outSurface) {
        //调用WMS的relayoutWindow方法
        int res = mService.relayoutWindow(this, window, seq, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
                outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
                outStableInsets, outsets, outBackdropFrame, cutout,
                mergedConfiguration, outSurface);
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
                + Binder.getCallingPid());
        return res;
    }
```

* 最终调用WMS的relayoutWindow方法，WMS会对这个mSurface做一些事情，使得这个mSurface跟当前的PhoneWindow关联起来。

* performDraw -> draw -> drawSoftware

* 继续看ViewRootImpl的drawSoftware方法

##### ViewRootImpl#drawSoftware

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {

        // Draw with software renderer.
        final Canvas canvas;
        //1、从mSurface获取一个Canvas
        canvas = mSurface.lockCanvas(dirty);
        //2、View绘制
        mView.draw(canvas);
        //3、绘制完提交到Surfece，并且释放画笔
        surface.unlockCanvasAndPost(canvas);

    }
```

* drawSoftware 方法的核心就三句代码：

1. lockCanvas从Surface 获取一个canvas。
2. draw，用这个canvas去执行View的绘制。
3. unlockCanvasAndPost，将绘制的内容提交到Surface。

* mView.draw(canvas); 大家都比较熟悉了，mView 是DecorView，是一个FrameLayout，这里就是遍历子view，调用draw方法了，最终就是调用Canvas.draw。
* 整个View树draw完，就会将绘制的内容提交到Surface，之后这个Surface会被SurfaceFlinger 管理。



![软件绘制流程](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de9810896314c7~tplv-t2oaga2asx-watermark.awebp)

* 解读一下这种图：
  * Surface ：每个View都是由某一个窗口管理，每个窗口关联一个Surface
  * Canvas：通过Surface的lockCanvas方法获取一个Canvas，Canvas可以理解为Skia底层接口的封装，调用Canvas 的draw方法，相当于调用Skia库的绘制方法。
  * Graphic Buffer：SurfaceFlinger 会帮我们托管一个Buffer Queue，我们从Buffer Queue获取一个Graphic Buffer，然后通过Canvas以及Skia将绘制的数据栅格化到上面。
  * SurfaceFlinger：通过Swap Buffer 把前台Graphic Buffer 提交给SurfaceFlinger，最后通过硬件合成器 Hardware Composer合成并输出到显示屏。

### UI原理总结

1. Activity的attach 方法里创建PhoneWindow。
2. onCreate方法里的 setContentView 会调用PhoneWindow的setContentView方法，创建DecorView并且把xml布局解析然后添加到DecorView中。
3. 在onResume方法执行后，会创建ViewRootImpl，它是最顶级的View，是DecorView的parent，创建之后会调用setView方法。
4. ViewRootImpl 的 setView方法，会将PhoneWindow添加到WMS中，通过 Session作为媒介。 setView方法里面会调用requestLayout，发起绘制请求。
5. requestLayout 一旦发起，最终会调用 performTraversals 方法，里面将会调用View的三个measure、layout、draw方法，其中View的draw 方法需要一个传一个Canvas参数。
6. 最后分析了软件绘制的原理，通过relayoutWindow 方法将Surface跟当前Window绑定，通过Surface的lockCanvas方法获取Surface的的Canvas，然后View的绘制就通过这个Canvas，最后通过Surface的unlockCanvasAndPost 方法提交绘制的数据，最终将绘制的数据交给SurfaceFlinger去提交给屏幕显示。



![软件绘制流程](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de9810896314c7~tplv-t2oaga2asx-watermark.awebp)



## 二、LayoutInflater 原理

* 前面分析setContentView的时候，最终会调用 `mLayoutInflater.inflate(layoutResID, mContentParent);` 将布局文件的View添加到DecorView的content部分。这一节就来分析LayoutInflater的原理~

* LayoutInflater 是在 PhoneWindow 构造方法里创建的

```java
public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
}
```

* LayoutInflater是一个系统服务

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    return LayoutInflater;
}
```

* inflate 方法有很多重载，看带有资源id参数的

### 2.1 LayoutInflater#inflate(@LayoutRes int resource, @Nullable ViewGroup root)

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}


//root 不为空，attachToRoot就为true，表示最终解析出来的View要add到root中去。
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
	//1.将资源id转换成 XmlResourceParser   
    final XmlResourceParser parser = res.getLayout(resource);
    try {
		//2.调用另一个重载方法
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}	
```

* 注释1是将资源id转换成 XmlResourceParser，XmlResourceParser 是啥？

```java
public interface XmlResourceParser extends XmlPullParser, AttributeSet, AutoCloseable {
    /**
     * Close this interface to the resource.  Calls on the interface are no
     * longer value after this call.
     */
    public void close();
}
```

* 继承 XmlPullParser ，这是一个标准XmlPullParser接口，同时继承AttributeSet接口以及当用户完成资源读取时调用的close接口，简单来说就是一个xml资源解析器，用于解析xml资源。

* 关于 XmlPullParser 原理可以参考文末链接。

### 2.2 【扩展】XML解析有哪些方式

* SAX（Simple API XML）

> 是一种基于事件的解析器，事件驱动的流式解析方式是，从文件的开始顺序解析到文档的结束，不可暂停或倒退

* DOM

> 即对象文档模型，它是将整个XML文档载入内存(所以效率较低，不推荐使用)，每一个节点当做一个对象

* Pull

> Android解析布局文件所使用的方式。Pull与SAX有点类似，都提供了类似的事件，如开始元素和结束元素。不同的是，SAX的事件驱动是回调相应方法，需要提供回调的方法，而后在SAX内部自动调用相应的方法。而Pull解析器并没有强制要求提供触发的方法。因为他触发的事件不是一个方法，而是一个数字。它使用方便，效率高。

* 没用过没关系，但是要知道Pull解析的优点：
  Pull解析器小巧轻便，解析速度快，简单易用，非常适合在Android移动设备中使用，Android系统内部在解析各种XML时也是用Pull解析器，Android官方推荐开发者们使用Pull解析技术。

* 上面inflate方法注释1解析完xml，得到XmlResourceParser 对象，

> 解析一个xml，我们可以直接使用 `getContext().getResources().getLayout(resource)；`，返回一个 XmlResourceParser 对象。

* 注释2，调用另一个inflate 的重载方法，XmlResourceParser 作为入参

### 2.3 LayoutInflater#inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {

        final Context inflaterContext = mContext;
		//获取xml中的属性，存到一个Set里
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            // Look for the root node.
            int type;
	   		//1、寻找开始标签
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

	    	//2.遍历一遍没找到开始标签，抛异常
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

	    	//3.开始标签的名字，例如根布局是个LinearLayout，那么这个name就是LinearLayout
            final String name = parser.getName();
            
	    	//4、如果根布局是merge标签
            if (TAG_MERGE.equals(name)) {
				//merge标签必须要有附在一个父布局上，rootView是空，那么抛异常
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

				// 遇到merge标签调用这个rInflate方法（下面会分析）
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
				//5、不是merge标签，调用createViewFromTag，通过标签创建一个根View
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    
		    		//这个方法里面是解析layout_width和layout_height属性，创建一个LayoutParams
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                    	//如果只是单纯解析，不附加到一个父View去，那么根View设置LayoutParams，否则，使用外部View的宽高属性
                        temp.setLayoutParams(params);
                    }
                }

				//6.调用这个方法，解析子View
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);


                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

				//如果不附加到root，那么返回这个temp，result 默认是 root
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        }


        return result;
    }
}
```

* 这个方法按顺序来分析，不是很复杂，一步步看下，

* 注释1到3：从XmlPullParser 里面解析出开始标签，例如，然后取出标签的名字 LinearLayout

* 注释4：**如果根标签是 merge标签**，那么再判断，merge标签必须要依附到rootview，否则抛异常。merge标签的使用就不用说了，必须放在根布局。

* 根布局merge标签的解析是调用 `rInflate(parser, root, inflaterContext, attrs, false);`方法，看一下

### 2.4 merge标签解析：rInflate

```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();
        
		
        if (TAG_REQUEST_FOCUS.equals(name)) { 
	    	// 1、requestFocus 标签
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {     
	    	// 2、tag 标签
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {  
	    	// 3、include 标签
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {  
	    	//4、merge 标签下不能有merge标签
            throw new InflateException("<merge /> must be the root element");
        } else {                             
	    	//5、其它正常标签，调用createViewFromTag 创建View对象
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
	    	//6、递归解析子View
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

* merge标签的解析，主要是以下这些情况：

```xml
<?xml version="1.0" encoding="utf-8"?>
<merge
    xmlns:android="http://schemas.android.com/apk/res/android">

    <requestFocus></requestFocus>  //requestFocus

    <tag android:id="@+id/tag"></tag>  //tag

    <include layout="@layout/activity_main"/>  //include

    <merge>  //错误，merge标签必须在根布局
        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:textColor="@color/colorAccent"
            android:text="123"/>
    </merge>

	<TextView  //正常标签
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:textColor="@color/colorAccent"
        android:text="123"/>

</merge>
```

#### 2.4.1 解析requestFocus标签

* 调用父View的 requestFocus 方法

```java
private void parseRequestFocus(XmlPullParser parser, View view)
        throws XmlPullParserException, IOException {
    view.requestFocus(); //这个view是parent，也就是父布局请求聚焦

    consumeChildElements(parser);  //跳过requestFocus标签
}


final static void consumeChildElements(XmlPullParser parser)
        throws XmlPullParserException, IOException {
    int type;
    final int currentDepth = parser.getDepth();
    //遇到 END_TAG 退出循环，在此处也就是跳过 requestFocus 标签
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > currentDepth) && type != XmlPullParser.END_DOCUMENT) {
        // Empty
    }
}
```

#### 2.4.2 解析tag标签

* 给父布局设置tag

```java
private void parseViewTag(XmlPullParser parser, View view, AttributeSet attrs)
        throws XmlPullParserException, IOException {
    final Context context = view.getContext();
    final TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.ViewTag);
    final int key = ta.getResourceId(R.styleable.ViewTag_id, 0);
    final CharSequence value = ta.getText(R.styleable.ViewTag_value);
    view.setTag(key, value); //这个view是parent，给父布局设置tag
    ta.recycle();

    consumeChildElements(parser);  //跳过tag标签
}
```

* 可以看到requestFocus标签和tag标签只是给父View设置属性。

#### 2.4.3 解析include标签

```java
private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    int type;

	// 父View必须是ViewGroup
    if (parent instanceof ViewGroup) {
       
        ...
		//1、解析layout属性值id，解析不到默认0
        int layout = attrs.getAttributeResourceValue(null, ATTR_LAYOUT, 0);
        ...
		//2、检查layout属性值，是0就抛异常
        if (layout == 0) {
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            throw new InflateException("You must specify a valid layout "
                    + "reference. The layout ID " + value + " is not valid.");
        } else {
	    	//3、解析layout属性对应的布局
            final XmlResourceParser childParser = context.getResources().getLayout(layout);
	    	//之后就跟解析前面一样，判断是否有merge标签，有就执行 rInflate，没有就执行 createViewFromTag
            try {
                final AttributeSet childAttrs = Xml.asAttributeSet(childParser);

                while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty.
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(childParser.getPositionDescription() +
                            ": No start tag found!");
                }

                final String childName = childParser.getName();

                if (TAG_MERGE.equals(childName)) {
                    // The <merge> tag doesn't support android:theme, so
                    // nothing special to do here.
                    rInflate(childParser, parent, context, childAttrs, false);
                } else {
                    final View view = createViewFromTag(parent, childName,
                            context, childAttrs, hasThemeOverride);
		    		//父View，最终解析子View会添加到父View里。
                    final ViewGroup group = (ViewGroup) parent;

                    final TypedArray a = context.obtainStyledAttributes(
                            attrs, R.styleable.Include);
                    final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
		    		//获取visibility属性
                    final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                    a.recycle();
                    ViewGroup.LayoutParams params = null;
                    try {
                        params = group.generateLayoutParams(attrs);
                    } catch (RuntimeException e) {
                        // Ignore, just fail over to child attrs.
                    }
                    if (params == null) {
                        params = group.generateLayoutParams(childAttrs);
                    }
                    view.setLayoutParams(params);

                    //解析子View
                    rInflateChildren(childParser, view, childAttrs, true);

                    if (id != View.NO_ID) {
                        view.setId(id);
                    }
		    		//设置 visibility 属性
                    switch (visibility) {
                        case 0:
                            view.setVisibility(View.VISIBLE);
                            break;
                        case 1:
                            view.setVisibility(View.INVISIBLE);
                            break;
                        case 2:
                            view.setVisibility(View.GONE);
                            break;
                    }

                    group.addView(view);
                }
            } finally {
                childParser.close();
            }
        }
    } else {
        throw new InflateException("<include /> can only be used inside of a ViewGroup");
    }

    LayoutInflater.consumeChildElements(parser);
}
```

* 解析include标签步骤比较多，以 `<include layout="@layout/activity_main"/>`为例：

* 首先，include标签的父View必须是ViewGroup，
  注释1和注释2是检查include标签有没有layout属性，属性值不能是0，属性值对应`layout/activity_main`这个布局id，

* 注释3，xml解析，解析`layout/activity_main`布局，`context.getResources().getLayout(layout)`很熟悉了， 然后就跟最开始的解析布局差不多了，如果遇到`merge`标签，就调用`rInflate`方法去解析merge标签，否则，就跟inflate方法逻辑一样，调用`createViewFromTag`创建一个View对象，然后再调用`rInflateChildren`解析子View，还有就是给View设置visibility属性。 这部分逻辑跟inflate方法里面一样，所以放到后面分析。

#### 2.4.4 子标签是merge

* merge标签下不能有merge标签，**merge标签必须在xml根节点**

#### 2.4.5 递归解析子View，rInflateChildren

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    //递归调用解析merge标签的方法，parent换了而已
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

* merge标签解析完了，回到 2.3 LayoutInflater#inflate 看注释5

### 2.5 LayoutInflater#inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)

```java
				//5、不是merge标签，调用createViewFromTag，通过标签创建一个根View
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    
		    		//这个方法里面是解析layout_width和layout_height属性，返回一个LayoutParams
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        //如果只是单纯解析，不附加到一个父View去，那么根View设置LayoutParams，否则，使用外部View的宽高属性
                        temp.setLayoutParams(params);
                    }
                }

				//6、调用这个方法，解析子View
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);


                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

				//如果不附加到root，那么返回这个temp，result 默认是 root
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
```

* 注释5，这里的逻辑跟解析include标签里的layout逻辑基本是一样的，通过`createViewFromTag` 方法,传View的名称，去创建一个View对象

### 2.6 LayoutInflater#createViewFromTag

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {

    //1、如果是view标签，取出class属性
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    ...

    //2、TAG_1995 ：blink，如果是blink标签，就返回一个 BlinkLayout，会闪烁的意思
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
        if (mFactory2 != null) {
	    	// 3、通过 mFactory2 创建View
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
	    	// 4、通过 mFactory 创建View
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
	    	// 5、通过 mPrivateFactory 创建View
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
				// 6、通过 onCreateView 方法创建View
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } 
}
```

* 注释1：如果是view标签，例如`<view class="android.widget.TextView"></view>`，则取出class属性，也就是`android.widget.TextView`，

* 注释2： 如果标签是blink，则创建 `BlinkLayout` 返回，看下BlinkLayout 是啥？

#### 2.6.1 BlinkLayout

```java
private static class BlinkLayout extends FrameLayout {
    private static final int MESSAGE_BLINK = 0x42;
    private static final int BLINK_DELAY = 500;

    private boolean mBlink;
    private boolean mBlinkState;
    private final Handler mHandler;

    public BlinkLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        mHandler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                if (msg.what == MESSAGE_BLINK) {
                    if (mBlink) {
                        mBlinkState = !mBlinkState;
                        makeBlink(); //循mBlink为true则循环调用
                    }
                    invalidate();
                    return true;
                }
                return false;
            }
        });
    }

    private void makeBlink() {
        Message message = mHandler.obtainMessage(MESSAGE_BLINK);
        mHandler.sendMessageDelayed(message, BLINK_DELAY); //延时500毫秒
    }

	@Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();

        mBlink = true; //标志位true
        mBlinkState = true;

        makeBlink();
    }

	@Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();

        mBlink = false;
        mBlinkState = true;

        mHandler.removeMessages(MESSAGE_BLINK);
    }
```

* 内部通过Handler，onAttachedToWindow的时候mBlink是true，调用makeBlink方法后，500毫秒之后Handler处理，因为mBlink为true，所以又会调用makeBlink方法。所以这是一个每500毫秒就会改变状态刷新一次的FrameLayout，表示没用过这玩意。

* 继续看 createViewFromTag方法的注释3、通过 mFactory2 创建View

#### 2.6.2 Factory2

```java
public interface Factory2 extends Factory {
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}

public interface Factory {
    public View onCreateView(String name, Context context, AttributeSet attrs);
}
```

* Factory2是一个接口，mFactory2 不为空，那么最终会通过这个接口的实现类去创建View。

* AppCompatDelegateImpl 中实现了这个Factory2接口，什么时候使用 AppCompatDelegateImpl？还有就是mFactory2 是什么时候在哪里赋值的？

* 看下 AppCompatActivity 源码就明白了

#### 2.6.3 AppCompatActivity

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    AppCompatDelegate delegate = this.getDelegate(); //1、获取代理
    delegate.installViewFactory(); //2、初始化Factory
    delegate.onCreate(savedInstanceState);
    ..
}
```

* 注释1：获取代理

```
public AppCompatDelegate getDelegate() {
    if (this.mDelegate == null) {
        this.mDelegate = AppCompatDelegate.create(this, this);
    }

    return this.mDelegate;
}
```

* AppCompatDelegate.create 其实返回的就是 AppCompatDelegateImpl

```java
public static AppCompatDelegate create(Activity activity, AppCompatCallback callback) {
    return new AppCompatDelegateImpl(activity, activity.getWindow(), callback);
}
```

* onCreate 注释2：delegate.installViewFactory()，看 AppCompatDelegateImpl 里面

```java
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(this.mContext);
	//如果 getFactory 空，就设置Factory2
    if (layoutInflater.getFactory() == null) {
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    }

}
```

* LayoutInflaterCompat#setFactory2

```java
public static void setFactory2(@NonNull LayoutInflater inflater, @NonNull Factory2 factory) {
    inflater.setFactory2(factory); //设置factory2
    //AppCompact 是兼容包，sdk 21以下应该是没有提供setFactory2 这个方法，需要强制通过反射设置factory2，
    if (VERSION.SDK_INT < 21) {
        Factory f = inflater.getFactory();
        if (f instanceof Factory2) {
	    //强制设置Factory2
            forceSetFactory2(inflater, (Factory2)f);
        } else {
            forceSetFactory2(inflater, factory);
        }
    }

}
```

* LayoutInflaterCompat#forceSetFactory2

* 只看关键代码如下：

```java
private static void forceSetFactory2(LayoutInflater inflater, Factory2 factory) {
    //反射拿到LayoutInflater 的mFactory2 字段
    sLayoutInflaterFactory2Field = LayoutInflater.class.getDeclaredField("mFactory2");
    sLayoutInflaterFactory2Field.setAccessible(true);

	//强制设置
	sLayoutInflaterFactory2Field.set(inflater, factory);

    
}
```

* 这里简单说一下AppCompact 是干嘛的，简单来说就是高版本的一些特性，正常情况下低版本设备是看不到的，所以谷歌提供兼容包，v4、v7、以及现在推荐替换androidx，目的是为了让高api的一些特性，在低版本也能享受到，只要继承AppCompactActivity，低版本设备也能有高版本api的特性。

* 经过上面分析，针对 `view = mFactory2.onCreateView(parent, name, context, attrs);` 这句代码，
  我们来看 `AppCompatDelegateImpl`的`onCreateView` 方法就对了，有理有据，你不要说你写界面都是继承Activity，我会鄙视你。

#### 2.6.4 AppCompatDelegateImpl#onCreateView

```JAVA
public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return this.createView(parent, name, context, attrs);
}

public View createView(View parent, String name, @NonNull Context context, @NonNull AttributeSet attrs) {
    if (this.mAppCompatViewInflater == null) {
        TypedArray a = this.mContext.obtainStyledAttributes(styleable.AppCompatTheme);
		//1、我们可以在主题中配置自定义的 Inflater
        String viewInflaterClassName = a.getString(styleable.AppCompatTheme_viewInflaterClass);
        if (viewInflaterClassName != null && !AppCompatViewInflater.class.getName().equals(viewInflaterClassName)) {
            try {
                Class viewInflaterClass = Class.forName(viewInflaterClassName);
                this.mAppCompatViewInflater = (AppCompatViewInflater)viewInflaterClass.getDeclaredConstructor().newInstance();
            } catch (Throwable var8) {
                Log.i("AppCompatDelegate", "Failed to instantiate custom view inflater " + viewInflaterClassName + ". Falling back to default.", var8);
                this.mAppCompatViewInflater = new AppCompatViewInflater();
            }
        } else {
	    	// 2、如果没有指定，默认创建 AppCompatViewInflater
            this.mAppCompatViewInflater = new AppCompatViewInflater();
        }
    }
	...
    //3、最后通过AppCompatViewInflater 的 createView 方法创建View
    return this.mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext, IS_PRE_LOLLIPOP, true, VectorEnabledTintResources.shouldBeUsed());
}
```

* 注释1和注释2，说明我们可以在主题中配置Inflater，可以自定义创建View的规则

```
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="viewInflaterClass">android.support.v7.app.AppCompatViewInflater</item> //指定viewInflaterClass
</style>
```

* 注释3 AppCompatViewInflater 的 createView 方法就是真正去创建View对象了。

#### 2.6.5 AppCompatViewInflater#createView

```java
final View createView(View parent, String name, @NonNull Context context, @NonNull AttributeSet attrs, boolean inheritContext, boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    ...
    switch(name.hashCode()) {
	...
    case -938935918:
        if (name.equals("TextView")) {
            var12 = 0;
        }
        break;

    case 2001146706:
        if (name.equals("Button")) {
            var12 = 2;
        }
    }
	...省略很多case

    //上面已经对var12赋值，走对应的方法
    switch(var12) {
    case 0:
		// 0 是 TextView标签
        view = this.createTextView(context, attrs);
        this.verifyNotNull((View)view, name);
        break;
    case 1:
        view = this.createImageView(context, attrs);
        this.verifyNotNull((View)view, name);
        break;
    case 2:
        view = this.createButton(context, attrs);
        this.verifyNotNull((View)view, name);
        break;
    ...省略很多case

    default:
        view = this.createView(context, name, attrs);
    }

    if (view == null && originalContext != context) {
        //如果上面都没有创建，createViewFromTag 挽救一些
        view = this.createViewFromTag(context, name, attrs);
    }

    if (view != null) {
        this.checkOnClickListener((View)view, attrs);
    }

    return (View)view;
}
```

* 对应View走对应的创建方法，遇到TextView，调用 createTextView 方法，

```java
protected AppCompatTextView createTextView(Context context, AttributeSet attrs) {
    return new AppCompatTextView(context, attrs);
}
```

* 看到这里就明白了，**只要使用AppCompactActivity，布局里写TextView，最终创建的是 AppCompatTextView。**

* 其它ImageView、Button同理，就不再分析，读者可以自己查看源码。

* 如果走完所有case没有创建成功，没关系，还可以挽救一下，调用 createViewFromTag，添加前缀，然后通过反射创建，看一下

#### 2.6.6 AppCompatViewInflater#createViewFromTag

```
private View createViewFromTag(Context context, String name, AttributeSet attrs) {
    //1、如果是view标签，获取class属性，就是类的全路径
    if (name.equals("view")) {
        name = attrs.getAttributeValue((String)null, "class");
    }

    View view;
    try {
        this.mConstructorArgs[0] = context;
        this.mConstructorArgs[1] = attrs;
        View var4;
        //如果是API提供的控件，这个条件会成立
        if (-1 == name.indexOf(46)) {
	    //sClassPrefixList 定义： private static final String[] sClassPrefixList = new String[]{"android.widget.", "android.view.", "android.webkit."};
            for(int i = 0; i < sClassPrefixList.length; ++i) {
				//createViewByPrefix，传View标签名和前缀
                view = this.createViewByPrefix(context, name, sClassPrefixList[i]);
                if (view != null) {
                    View var6 = view;
                    return var6;
                }
            }

            var4 = null;
            return var4;
        }

		//自定义的控件，不需要前缀，传null
        var4 = this.createViewByPrefix(context, name, (String)null);
        return var4;
    } catch (Exception var10) {
        view = null;
    } finally {
        this.mConstructorArgs[0] = null;
        this.mConstructorArgs[1] = null;
    }

    return view;
}
```

* 在布局xml里写的 TextView，类全路径是android.widget.TextView，所以要添加前缀 android.widget.才能通过全路径反射创建TextView对象， sClassPrefixList 中定义的前缀有三个： `private static final String[] sClassPrefixList = new String[]{"android.widget.", "android.view.", "android.webkit."};`，原生控件定义在这几个目录下。

* 通过View名字加上前缀（如果是我们的自定义View，前缀null）来创建View，调用createViewByPrefix方法

#### 2.6.7 AppCompatViewInflater#createViewByPrefix

```java
private View createViewByPrefix(Context context, String name, String prefix) throws ClassNotFoundException, InflateException {
    Constructor constructor = (Constructor)sConstructorMap.get(name);

    try {
		//如果没有缓存
        if (constructor == null) {
	    	//最终通过ClassLoader去加载一个类，类有前缀的话会在这里做拼接
            Class<? extends View> clazz = context.getClassLoader().loadClass(prefix != null ? prefix + name : name).asSubclass(View.class);
            constructor = clazz.getConstructor(sConstructorSignature);
	    	//缓存类构造器
            sConstructorMap.put(name, constructor);
        }

        constructor.setAccessible(true);
        return (View)constructor.newInstance(this.mConstructorArgs);
    } catch (Exception var6) {
        return null;
    }
}
```

* 最终通过ClassLoader去加载一个类，asSubclass(View.class); 是强转换成View对象，

* 这里用到反射去创建一个View对象，考虑到反射的性能，所以缓存了类的构造器，key是类名，也就是说第一次需要反射加载Class对象，之后创建相同的View，直接从缓存中取出该View的构造器，调用 newInstance 方法实例化。

* 看到这里，可能大家会有一个问题了，为什么只分析AppCompat，如果主题不使用AppCompat，那么流程又是怎样的？

* 这里之所以分析AppCompat，因为它其实已经包含了默认的处理在里面，比如TextView，如果创建AppCompatTextView失败，则通过反射区创建`android.widget.TextView`，默认流程也差不多是这样的，不信？

* 再回到 LayoutInflater#createViewFromTag

### 2.7 LayoutInflater#createViewFromTag

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {

    //1、如果是view标签，取出class属性
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    ...

    //2、TAG_1995 ：blink，如果是blink标签，就返回一个 BlinkLayout，会闪烁的意思
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
        if (mFactory2 != null) {
	    	// 3、通过 mFactory2 创建View
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
	    	// 4、通过 mFactory 创建View
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
	    	// 5、通过 mPrivateFactory 创建View
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
				// 6、通过 onCreateView 方法创建View
                if (-1 == name.indexOf('.')) {
                    //原生控件没有.  调用这个 onCreateView方法创建
                    view = onCreateView(parent, name, attrs);
                } else {
                    //自定义控件全名有. 所以前缀传null
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } 
}
```

* 注释3、4、5 都是通过代理类去创建，分析了mFactory2，mFactory也是类似的，这里就不再分析了，
  如果没有代理类或者代理类没能力创建，那么走默认的创建方法，也就就是 注释6，LayoutInflater#onCreateView(String name, AttributeSet attrs)，简单看下吧

#### 2.7.1 LayoutInflater#onCreateView(String name, AttributeSet attrs)

```java
protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    //1、传前缀，不考虑support包，android提供的原生控件都是android.view. 开头
    return createView(name, "android.view.", attrs);
}

public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    //2、读取缓存
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;

    try {

        //没有有缓存，通过反射创建View
        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            //有缓存的逻辑
			...

        }

        Object[] args = mConstructorArgs;
        args[1] = attrs;

        final View view = constructor.newInstance(args);
		//如果是 ViewStub
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;

    }
}
```

1、传前缀android.view. ，原生控件在这个包下。
2、通过反射创建View，反射耗性能，所以采用缓存机制。
3、如果遇到viewStub，则给它设置一个 LayoutInflater

* 上面分析的是一层View的创建流程，第二层View怎么创建的呢？

* 看 rInflateChildren(parser, temp, attrs, true)

### 2.8 rInflateChildren(parser, temp, attrs, true);

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

* rInflateChildren 方法内部调用 rInflate，rInflate方法上面其实已经分析过了，跟解析merge标签流程是一样的。到这里基本把整个xml解析成View对象的流程分析完了，需要简单小结一下，不然都忘了前面讲啥了。

### LayoutInflater 小结

* xml布局的解析，文字总结如下：

1、 采用 Pull解析，将布局id解析成 XmlResourceParser 对象，简单介绍几种xml解析方式：SAX、DOM和Pull解析。
2、 因为第一步已经将xml解析出来，所以直接取出标签的View名字，如果是merge标签，调用rInflate 方法，如果是普通的正常的View，就通过createViewFromTag 这个方法创建一个View对象。
3、介绍rInflate这个方法是如何解析各个标签的

- requestFocus标签
- tag 标签
- include 标签
- view 标签
- blink 标签
- merge 标签

主要是对以上这些标签（不分先后）的处理，处理完

xml中第二层View的创建最终也是调用rInflate这个方法，属于递归创建，创建View的顺序是深度优先。

3、 介绍 createViewFromTag 方法创建View的方式，如果是blink标签，直接new new BlinkLayout() 返回，简单介绍了BlinkLayout，虽然没用过它。

4、 正常情况下，createViewFromTag 方法创建View的几个逻辑

- mFactory2 不为空就调用mFactory2.onCreateView 方法创建
- mFactory 不为空就调用 mFactory.onCreateView 方法创建
- 上述创建失败，就用默认 mPrivateFactory 创建
- 上述依然没创建成功，就调用LayoutInflater的CreateView方法，给View添加 android.view. 前缀，然后通过反射去创建一个View对象，考虑到反射耗性能，所以第一次创建成功就会缓存类的构造器 Constructor，之后创建相同View只要调用构造器的newInstance方法。

5、介绍 Factory2 接口的实现类 AppCompatDelegateImpl，AppCompatActivity 内部几乎所有生命周期方法都交给AppCompatDelegateImpl这个代理去做，并且给LayoutInflater设置了Factory2，所以创建View的时候优先通过 Factory2 接口去创建，也就是通过AppCompatDelegateImpl 这个实现类去创建。

6、介绍 AppCompatDelegateImpl 创建View的方式，如果布局文件是TextView，则创建的是兼容类AppCompactTextView，其它控件同理。创建失败则回退到正常的创建流程，添加前缀，例如 android.widget.TextView，然后通过反射去创建。

 7、如果没有使用兼容包，例如继承Activity，那么也是通过反射去创建View对象，自定义View不加前缀。

## 三、UI 优化

* 所谓UI优化，就是拆解渲染过程的耗时，找到瓶颈的地方，加以优化。

* 前面分析了UI原理，Activity、Window、DecorView、ViewRootImpl之间的关系，以及XML布局文件是如何解析成View对象的。

* 耗时的地方：
  * **View的创建在主线程，包括measure、layout、draw，界面复杂的时候，这一部分可能会很耗时。**
  * **解析XML，反射创建VIew对象，这一过程的耗时。**

* 下面介绍一些常用的UI优化方式

### 3.1 常规方式

1. 减少UI层级、使用merge、Viewstub标签优化
2. 优化layout开销、RelativeLayout和带有weight的Linearlayout会测量多次,可以尝试使用ConstraintLayout 来代替。
3. 背景优化，分析DecorView创建的时候，发现DecorView会设置一个默认背景，可以统一将DecorView背景设置为通用背景，其它父控件就无需设置背景，避免重复绘制。

### 3.2 xml转换成代码

* 使用xml编写布局，很方便，但是最终要通过LayoutInflater的inflate方法，将xml解析出来并递归+反射去创建View对象，布局比较复杂的时候，这一部分会非常耗时。

* **使用代码创建可以减少xml递归解析和反射创建View的这部分耗时。** 当然，如果将xml都换成代码来写，开发效率将不忍直视，并且代码可读性也是个问题。

* 掌阅开源的一个库，编译期自动将xml转换成java代码，[X2C](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FiReaderAndroid%2FX2C%2Fblob%2Fmaster%2FREADME_CN.md)

* 它的原理是采用APT（Annotation Processor Tool）+ JavaPoet技术来完成编译期间【注解】-【解注解】->【翻译xml】->【生成java】整个流程的操作

> 即在编译生成APK期间，将需要翻译的layout翻译生成对应的java文件，这样对于开发人员来说写布局还是写原来的xml，但对于程序来说，运行时加载的是对应的java文件。



![编译期xml转java代码](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de9807f364b2f5~tplv-t2oaga2asx-watermark.awebp)



* 侵入性极低，去除注解则回退到原生的运行时解析方式。当然，有些情况是不支持转换的，比如merge标签，编译期没法确定它的parent。

### 3.3 异步创建View

* 通过子线程创建View，减少主线程耗时。

```java
private void threadNewView() {
        new Thread(){
            @Override
            public void run() {
                mSplashView = LayoutInflater.from(MainActivity.this).inflate(R.layout.activity_splash,null);
            }
        }.start();
    }

```

* 当然，这种方式需要处理同步问题，并且没有从源头上解决创建View耗时，只是将这部分耗时放到线程去做。UI更新的操作还是要切换到主线程，不然会触发ViewRootImpl的checkThread检测。

```java
void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

### 3.4 复用View

* 复用View这个应该比较常见了，像RecyclerView的四级缓存，目的就是通过复用View减少创建View的时间。

* 我们可以在onDestroy方法将View的状态清除，然后放入缓存。在onCreate的时候去命中缓存，设置状态。



![View缓存池](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de980815a25876~tplv-t2oaga2asx-watermark.awebp)



### 3.5 异步布局： Litho

* 正常情况下measure、layout、draw都是在主线程执行的，最终绘制操作是在draw方法，而measure、layout只是做一些数据准备，完全可以放到子线程去做。

* Litho 的原理就是将measure、layout 放到子线程： [github.com/facebook/li…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Flitho)



![Litho](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de980815b5de49~tplv-t2oaga2asx-watermark.awebp)



* 优点：

1. 将measure、layout、放到子线程去做，减少主线程耗时。
2. 使用自己的布局引擎，减少View层级，界面扁平化。
3. 优化RecyclerView，提高缓存命中率。

* 缺点：

1. 无法在AS中预览。
2. 使用自己的布局引擎，有一点的使用成本。

### 3.6 Flutter：自绘引擎

* Flutter 是一个跨平台UI框架，内部集成了Skia图像库，自己接管了图像绘制流程，性能直逼原生，是时候制定计划学习一波了~

### UI优化总结

* 前面解析了UI原理之后，我们知道UI渲染主要涉及到xml的解析、View的measure、layout、draw这些耗时的阶段。UI优化方式总结如下：
  * **常规方式。**
  * **xml的解析和反射创建View耗时，采用new 代替xml。**
  * **复用View**
  * **异步创建View，将创建View的这部分时间放到子线程。**
  * **将measure、layout放到子线程，代表： `Litho`。**

## 全文总结

* 本文分为三节：

1. 第一节通过源码分析了UI原理，主要是介绍Activity、Window、DecorView、ViewRootImpl之间的关系，以及创建的流程。
2. 第二节分析LayoutInflater解析xml布局的原理，反射创建View是耗时的，内部使用了缓存。
3. 第三节在前两节的原理基础上，介绍了了几种UI优化方式。