[TOC]

## 二、绘制前的准备：DecorView创建和显示

#### 转载

* [Android自定义View绘制前的准备：DecorView创建 & 显示](https://www.jianshu.com/p/ac3262d233af)

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0zNDk5MmViNDZiZGY5M2U3LnBuZw)

### 一、DecorView的创建

#### 1.1 setContentView

- 上面我们提到，`DecorView`是显示的顶层`View`，那么`View`的绘制准备从`DecorView`开始说起
- `DecorView`的开始 = 我们熟悉的 `setContentView()`

```csharp
/**
* 具体使用：Activity的setContentView()
*/
@Override
public void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}

/**
* 源码分析：Activity的setContentView()
*/
public void setContentView(int layoutResID) {
	// getWindow() 作用：获得Activity 的成员变量mWindow ->>分析1
    // Window类实例的setContentView（） ->>分析2
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

```

* getWindow获取的是Activity的成员变量mWindow

#### 1.2 成员变量mWindow

```java
/**
* 分析1：成员变量mWindow
*/
// 1. 创建一个Window对象（即 PhoneWindow实例）
// Window类 = 抽象类，其唯一实现类 = PhoneWindow
mWindow = new PhoneWindow(this, window);
  
// 2. 设置回调，向Activity分发点击或状态改变等事件
mWindow.setWindowControllerCallback(this);
mWindow.setCallback(this);

// 3. 为Window实例对象设置WindowManager对象
mWindow.setWindowManager(
		(WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
```

* mWindow是PhoneWindow实例对象
* 设置回调，向Activity分发点击或状态改变等事件
* 为Window实例对象设置WindowManager对象

#### 1.3 setContentView

* PhoneWindow#setContentView

```java
@Override
public void setContentView(int layoutResID) {
	// 1. 若mContentParent为空，创建一个DecroView
    // mContentParent即为内容栏（content）对应的ViewGroup
    if (mContentParent == null) {
    	installDecor();//分析3
	} else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
		// 若不为空，且没有动画，则删除其中的View
        mContentParent.removeAllViews();
	}

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        //如果有特别动画效果 执行下面的代码
    	final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
        		getContext());
		transitionTo(newScene);
	} else {
    	// 2. 为mContentParent添加子View
        // 即Activity中设置的布局文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
	}
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
    	//回调通知，内容改变
        cb.onContentChanged();
	}
    mContentParentExplicitlySet = true;
}

```

* 主要部分做了注释解释
* 分析3主要调用的installDecor方法

#### 1.4 installDecor

* PhoneWindow#installDecor

```java
/**
* 分析3：installDecor()
* 作用：创建一个DecroView;为DecorView设置布局格式
*/
private void installDecor() {
	if (mDecor == null) {
    	// 1. 生成DecorView ->>分析4
        mDecor = generateDecor(); 
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }else {
		mDecor.setWindow(this);
	}
    // 2. 为DecorView设置布局格式 & 返回mContentParent ->>分析5
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); 
        ...
        } 
    }
}
```

* 主要判断mDecor是否为空，为空的话，需要去生成DecorView
* 还需要为DecorView设置布局格式，并返回mContentParent

#### 1.5 generateDecor

* PhoneWindow#generateDecor

```java
/**
* 分析4：generateDecor()
* 作用：生成DecorView
*/    
protected DecorView generateDecor(int featureId) {
    //拿到上下文对象
	Context context;
    if (mUseDecorContext) {
    	Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
        	context = getContext();
		} else {
        	context = new DecorContext(applicationContext, getContext().getResources());
            if (mTheme != -1) {
            	context.setTheme(mTheme);
			}
		}
	} else {
    	context = getContext();
	}
    //创建DecorView对象
    return new DecorView(context, featureId, this, getAttributes());
}
```

#### 1.6 generateLayout

* PhoneWindow#generateLayout

```java
/**
* 分析5：generateLayout(mDecor)
* 作用：为DecorView设置布局格式
*/
protected ViewGroup generateLayout(DecorView decor) {
	// 1. 从主题文件中获取样式信息
    TypedArray a = getWindowStyle();
	......
	// 2. 根据主题样式，加载窗口布局
    int layoutResource;
    int features = getLocalFeatures();
	......
	// 3. 加载layoutResource
    View in = mLayoutInflater.inflate(layoutResource, null);
	......
	// 4. 往DecorView中添加子View
    // 即文章开头介绍DecorView时提到的布局格式，那只是一个例子，根据主题样式不同，加载不同的布局。
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT)); 
    mContentRoot = (ViewGroup) in;
	......
	// 5. 这里获取的是mContentParent = 即为内容栏（content）对应的DecorView = FrameLayout子类
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT); 
    
	if (getContainer() == null) {
        ......
		//设置标题 标题字体 标题子颜色等
    }    	
      
    return contentParent;
}
```

#### 1.7 总结

* mDecor是DecorView对象，对应的是最外层的顶层View

* mContentParent是ViewGroup对象，是content部分的顶层View。
* mDecor包括了mContentParent。

![img](https:////upload-images.jianshu.io/upload_images/944365-e451602c76fead54.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

* 此时，顶层`View`（`DecorView`对象mDecor）已创建
* 同时mDecor内部的mContentParent已经加载进了Activity中设置的布局文件

### 二、DecorView的显示

#### 2.1 handleResumeActivity

![根Activity启动过程中涉及的进程之间的关系](..\..\images\Android组件内核\四大组件\Activity\根Activity启动过程中涉及的进程之间的关系.png)

* 通过上面的步骤，最终会调用ActivityThread.java#handleLaunchActivity中调用ActivityThread.java#handleResumeActivity
* 详细情况可查看[根Activity启动过程](Android组件内核/FrameWork内核解析/根Activity启动过程.md)
* ActivityThread.java#handleResumeActivity主要部分代码如下所示

~~~java
final void handleResumeActivity(IBinder token,
		boolean clearHide, boolean isForward, boolean reallyResume) {

	ActivityClientRecord r = performResumeActivity(token, clearHide);
	if (r != null) {
    	final Activity a = r.activity;
        if (r.window == null && !a.mFinished && willBeVisible) {
        	// 1. 获取Window实例中的Decor对象
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();

            // 2. DecorView对用户不可见
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
              
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;

            // 3. DecorView被添加进WindowManager了
            // 此时，还是不可见
            if (a.mVisibleFromClient) {
            	a.mWindowAdded = true;
                    
                wm.addView(decor, l);
			}

            // 4. 此处设置DecorView对用户可见
            if (!r.activity.mFinished && willBeVisible
            	&& r.activity.mDecor != null && !r.hideForNow) {
                if (r.activity.mVisibleFromClient) {
                	r.activity.makeVisible();
                    // —>>分析1
				}
			}
		}
    }
}
~~~

* 最终通过r.activity.makeVisible()方法将DecorView对用户可见的，也就是Activity的makeVisible方法

#### 2.2 makeVisible

* Activity#makeVisible

```java
/**
* 分析1：Activity.makeVisible()
*/
void makeVisible() {
	if (!mWindowAdded) {
    	ViewManager wm = getWindowManager();
        // 1. 将DecorView添加到WindowManager ->>分析2
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
	}
    // 2. DecorView可见
    mDecor.setVisibility(View.VISIBLE);
}
```

* 主要功能：通过ViewManager对象wm的addView方法，将将DecorView添加到WindowManager中去

* 然后设置了mDecor为VISIBLE
* getWindowManager获取的是WindowManager对象，但是它是一个接口，WindowManager接口的实现类是WindowManagerImpl，所以调用的WindowManagerImpl的addView

#### 2.3 addView

* WindowManagerImpl#addView

```java

/**
* 分析2：wm.addView
* 作用：WindowManager = 1个接口，由WindowManagerImpl类实现
*/
public final class WindowManagerImpl implements WindowManager {    
	private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...
    @Override
    public void addView(View view, ViewGroup.LayoutParams params) {
        mGlobal.addView(view, params, mDisplay, mParentWindow); ->>分析3
    }
}
```

* 调用WindowManagerGlobal对象的addView方法

#### 2.4 addView

* WindowManagerGlobal#addView

```java
/**
  * 分析3：WindowManagerGlobal 的addView()
  */
public void addView(View view, ViewGroup.LayoutParams params,Display display, Window parentWindow) {

	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    ......

	synchronized (mLock) {
        ......
    	// 1. 实例化一个ViewRootImpl对象
        ViewRootImpl root;
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
	}

    // 2. WindowManager将DecorView实例对象交给ViewRootImpl 绘制View
    try {
    	root.setView(view, wparams, panelParentView);
        // ->> 分析4
	}catch (RuntimeException e) {
        ......
    }

}
```

* 实例化一个ViewRootImpl对象root，然后调用root.setView方法，并把DecorView实例对象作为参数传递进去

#### 2.5 setView

* ViewRootImpl#setView

```java
/**
  * 分析4：ViewRootImpl.setView（）
  */
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	......
	requestLayout(); // ->>分析5
	......
}
```

* 重点看一下requestLayout方法

#### 2.6 requestLayout

* ViewRootImpl#requestLayout

```java
/**
  * 分析5：ViewRootImpl.requestLayout()
  */
@Override
public void requestLayout() {
	if (!mHandlingLayoutInLayoutRequest) {
    	// 1. 检查是否在主线程
        checkThread();
        mLayoutRequested = true;//mLayoutRequested 是否measure和layout布局。
        // 2. ->>分析6
        scheduleTraversals();
	}
}
```

#### 2.7 scheduleTraversals

* ViewRootImpl#scheduleTraversals

```java
/**
  * 分析6：ViewRootImpl.scheduleTraversals()
  */
void scheduleTraversals() {
	if (!mTraversalScheduled) {
    	mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

        // 通过mHandler.post()发送一个runnable，在run()方法中去处理绘制流程
        // 与ActivityThread的Handler消息传递机制相似
        // ->>分析7
        mChoreographer.postCallback(
        		Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ......
	}
}
```

* 主要调用 mChoreographer.postCallback方法，最终会调用mTraversalRunnable的run方法

#### 2.8 run

* ViewRootImpl#TraversalRunnable#run

```java
/**
  * 分析7：Runnable类的子类对象mTraversalRunnable
  * 作用：在run()方法中去处理绘制流程
*/
final class TraversalRunnable implements Runnable {
	@Override
    public void run() {
    	doTraversal(); // ->>分析8
	}
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

* 最终会调用doTraversal

#### 2.8 doTraversal

* ViewRootImpl#doTraversal

```java
/**
* 分析8：doTraversal()
*/
void doTraversal() {
	mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
	performTraversals(); 
    // 最终会调用performTraversals()，从而开始View绘制的3大流程：Measure、Layout、Draw
}
// 注：
//    a. 我们知道ViewRootImpl中W类是Binder的Native端，用于接收WmS处理操作
//    b. 因W类的接收方法是在线程池中的，故可通过Handler将事件处理切换到主线程中
```

* 最终会调用performTraversals方法

#### 2.9 performTraversals

* 在此方法中开始View绘制的3大流程：Measure、Layout、Draw

#### 2.10 小结

##### 2.10.1 流程

上面的流程如下：

1. 将 `DecorView`对象添加到`WindowManager`中
2. 创建`ViewRootImpl`对象
3. `WindowManager` 将 `DecorView`对象交给`ViewRootImpl`对象
4. `ViewRootImpl`对象通过`Handler`向主线程发送了一条触发遍历操作的消息：`performTraversals()`；该方法用于执行`View`的绘制流程（`measure`、`layout`、`draw`）

##### 2.10.2 ViewRootImpl

* `ViewRootImpl`对象中接收的各种变化（如来自`WmS`的窗口属性变化、来自控件树的尺寸变化 & 重绘请求等都引发`performTraversals()`的调用 & 在其中完成处理

##### 2.10.3 performTraversals

从上可看出：

- 一次次`performTraversals()`的调用驱动着控件树有条不紊的工作
- 一旦此方法无法正常执行，整个控件树都将处于僵死状态
- 因此`performTraversals()`可以说是`ViewRootImpl`类对象的核心

#### 2.11 接下来的问题

* 而`View`的绘制则是在`performTraversals()`中执行，故下面从`performTraversals()`开始，讲解View绘制的三大流程（`measure`、`layout`、`draw`）

### 三、 总结

#### 3.1 源码分析流程图

![img](https:////upload-images.jianshu.io/upload_images/944365-7628f5c6bdc57a0c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



