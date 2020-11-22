[TOC]
- <!-- TOC -->
- [ RemoteViews解析](#remoteviews解析)
  - [ 参考链接](#参考链接)
  - [ 一、RemoteViews的应用](#一、remoteviews的应用)
    - [ 1.1 前言](#11-前言)
    - [ 1.2 RemoteViews在通知栏的应用](#12-remoteviews在通知栏的应用)
    - [ 1.3 RemoteViews在桌面小部件的应用](#13-remoteviews在桌面小部件的应用)
      - [ 1.3.1 主要步骤](#131-主要步骤)
      - [ 1.3.2 AppWidgetProvider的常用方法](#132-appwidgetprovider的常用方法)
      - [ 1.3.3 实例代码](#133-实例代码)
  - [ 二、RemoteViews的内部机制](#二、remoteviews的内部机制)
    - [ 2.1 RemoteViews支持的View类型](#21-remoteviews支持的view类型)
    - [ 2.2 RemoteViews常用的方法](#22-remoteviews常用的方法)
    - [ 2.3 整体通信流程](#23-整体通信流程)
    - [ 2.4 源码解析](#24-源码解析)
      - [ 2.4.1 setTextViewText](#241-settextviewtext)
      - [ 2.4.2 setCharSequence](#242-setcharsequence)
      - [ 2.4.3 addAction](#243-addaction)
      - [ 2.4.4 apply(Remoteviews)](#244-applyremoteviews)
      - [ 2.4.5 performApply](#245-performapply)
      - [ 2.4.6 apply(Action的子类ReflectionAction)](#246-applyaction的子类reflectionaction)
      - [ 2.4.7 总结](#247-总结)
    - [ 2.5 说明](#25-说明)
      - [ 2.5.1 ListView和StackView](#251-listview和stackview)
  <!-- /TOC -->
# RemoteViews解析

### 参考链接

* [**Android开发艺术探索**](https://book.douban.com/subject/26599538/)

## 一、RemoteViews的应用

### 1.1 前言

* RemoteViews主要用在通知栏和桌面小部件的开发过程中。通知栏主要通过NotificationManager的notify方法来实现，它除了默认效果外，还可以另外定义布局；桌面小部件则是通过AppWidgetProvider来实现，AppWidgetProvider本质上是一个广播。通知栏和桌面小部件的开发过程中都会用到RemoteViews，他们在更新界面时无法像在Activity里面那样去直接更新View，因为这两者都运行在SystemServer进程

### 1.2 RemoteViews在通知栏的应用

* 注意Android8.0前后通知栏的区别，做好适配。8.0之后需要CHANNEL_ID并且需要createNotificationChannel()

* 使用RemoteViews实现自定义通知栏

* 查看自定义通知栏请点击[通知栏的demo核心代码](https://github.com/nullWolf007/ToolProject/tree/master/通知栏的demo核心代码) 

### 1.3 RemoteViews在桌面小部件的应用

#### 1.3.1 主要步骤

* 定义小部件界面
* 定义小部件配置信息
* 定义小部件的实现类
* 在AndroidManifest.xml中声明小部件

#### 1.3.2 AppWidgetProvider的常用方法

* onEnable：当该窗口小部件第一次添加到桌面时调用该方法，可添加多次但只在第一次调用
* onUpdate：小部件被添加时或者每次小部件更新时都会调用一次该方法，小部件的更新时机由updatePeriodMills来指定，每个周期小部件都会自动更新一次。updatePeriodMills最低更新时间周期为30分钟
* onDeleted：每删除一次桌面小部件就调用一次
* onDisabled：当最后一个该类型的桌面小部件被删除时调用该方法，注意是最后一个
* onReceive：这是广播的内置方法，用于分发具体的事件给其他方法

#### 1.3.3 实例代码

* 查看桌面小部件实例代码请点击[桌面小部件demo的核心代码](https://github.com/nullWolf007/ToolProject/tree/master/桌面小部件demo的核心代码) 
* updatePeriodMills最低更新时间周期为30分钟

## 二、RemoteViews的内部机制

### 2.1 RemoteViews支持的View类型

**Layout**

* FrameLayout、LinearLayout、RelativeLayout、GridLayout

**View**

* Button、Chronometer、Image Button、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewStub

**使用其他的View类型，会抛出异常**

### 2.2 RemoteViews常用的方法

|                            方法名                            |                      作用                       |
| :----------------------------------------------------------: | :---------------------------------------------: |
|        setTextViewText(int viewId,CharSequence text)         |               设置TextView的文本                |
|     setTextViewTextSize(int viewId,int units,float size)     |             设置TextView的字体大小              |
|              setTextColor(int viewId,int color)              |             设置TextView的字体颜色              |
|          setImageViewResource(int viewId,int srcId)          |             设置ImageView的图片资源             |
|         setImageViewBitmap(int viewId,Bitmap bitmap)         |               设置ImageView的图片               |
|        setInt(int viewId,String methodName,int value)        |      反射调用View对象的参数类型为int的方法      |
|       setLong(int viewId,String methodName,long value)       |     反射调用View对象的参数类型为long的方法      |
|    setBoolean(int viewId,String methodName,boolean value)    |    反射调用View对象的参数类型为boolean的方法    |
| setOnClickPendingIntent(int viewId,PendingIntent pendingIntent) | 为View添加点击事件，事件类型只能为PendingIntent |

 ### 2.3 整体通信流程

* 通知栏和桌面小部件分别由NotificationManager和AppWidgetManager管理，而NotificationManager和AppWidgetManager通过**Binder**分别和**SystemServer进程**中的NotificationManagerService以及AppWidgetService进行通信。所以通知栏和桌面小部件的布局文件都是在SystemServer进程中进行加载的，所以实际上是**跨进程通信**
* 首先RemoteViews会通过Binder传递到SystemServer进程，这是因为RemoteViews实现了Parcelable接口。系统会根据remoteViews中的包名等信息取得到应用的资源，然后通过LayoutInflater去加载RemoteViews的布局文件。系统会等RemoteViews被加载以后进行界面更新的任务，也就是我们通过set方法来设置的布局。当需要更新RemoteViews的时候，需要调用一系列set方法，并通过NotificationManager和AppWidgetManager来提交更新任务，具体的更新操作也是在SystemServer中进行
* 理论上系统可以直接通过Binder去支持所有的View和View操作。但是这样的话代价太大，因为View的方法太多了，另外就是大量的IPC操作会影响效率。为了解决这个问题，系统提供了一个Action的概念。Action代表一个View操作，Action实现了Parcelable接口。系统先把View操作封装到Action对象中，传递Action对象而不是View操作，远程中取出Action对象，然后得到View的操作去执行。远程对象通过RemoteViews的apply方法来进行View的更新操作。

![](..\..\images\自定义UI\系统布局\RemoteViews解析\RemoteViews内部机制.png)

### 2.4 源码解析

#### 2.4.1 setTextViewText

```java
public void setTextViewText(int viewId, CharSequence text) {
	setCharSequence(viewId, "setText", text);
}
```

#### 2.4.2 setCharSequence

```java
public void setCharSequence(int viewId, String methodName, CharSequence value) {
	addAction(new ReflectionAction(viewId, methodName, ReflectionAction.CHAR_SEQUENCE, value));
}
```

* 从ReflectionAction这个名称，我们就能了解到这是一个反射类型的动作

#### 2.4.3 addAction

```java
private void addAction(Action a) {
	if (hasLandscapeAndPortraitLayouts()) {
    	throw new RuntimeException("RemoteViews specifying separate landscape and portrait" +
                    " layouts cannot be modified. Instead, fully configure the landscape and" +
                    " portrait layouts individually before constructing the combined layout.");
	}
    if (mActions == null) {
    	mActions = new ArrayList<Action>();
	}
    mActions.add(a);

    // update the memory usage stats
    a.updateMemoryUsageEstimate(mMemoryUsageCounter);
}
```

* 从上面的代码，可以知道RemoteViews内部有一个mActions成员，是一个ArrayList，外界每调用一次set方法，RemoteViews就会创建一个Action对象并加入到这个ArrayList中。这里仅仅把Action对象保存起来，并未对View进行实际的操作。跟随代码，我们需要了解ReflectionAction，但是在此之前，先看下RemoteViews的apply方法

#### 2.4.4 apply(Remoteviews)

```java
public View apply(Context context, ViewGroup parent, OnClickHandler handler) {
	RemoteViews rvToApply = getRemoteViewsToApply(context);

    View result = inflateView(context, rvToApply, parent);
    loadTransitionOverride(context, handler);

    rvToApply.performApply(result, parent, handler);

    return result;
}

private View inflateView(Context context, RemoteViews rv, ViewGroup parent) {
	// RemoteViews may be built by an application installed in another
    // user. So build a context that loads resources from that user but
    // still returns the current users userId so settings like data / time formats
    // are loaded without requiring cross user persmissions.
    final Context contextForResources = getContextForResources(context);
    Context inflationContext = new RemoteViewsContextWrapper(context, contextForResources);

    LayoutInflater inflater = (LayoutInflater)
   		context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

	// Clone inflater so we load resources from correct context and
    // we don't add a filter to the static version returned by getSystemService.
    inflater = inflater.cloneInContext(inflationContext);
    inflater.setFilter(this);
    View v = inflater.inflate(rv.getLayoutId(), parent, false);
    v.setTagInternal(R.id.widget_frame, rv.getLayoutId());
    return v;
}

private static void loadTransitionOverride(Context context,
	RemoteViews.OnClickHandler handler) {
    if (handler != null && context.getResources().getBoolean(
		com.android.internal.R.bool.config_overrideRemoteViewsActivityTransition)) {
        TypedArray windowStyle = context.getTheme().obtainStyledAttributes(
        	com.android.internal.R.styleable.Window);
		int windowAnimations = windowStyle.getResourceId(
        	com.android.internal.R.styleable.Window_windowAnimationStyle, 0);
		TypedArray windowAnimationStyle = context.obtainStyledAttributes(
        	windowAnimations, com.android.internal.R.styleable.WindowAnimation);
		handler.setEnterAnimationId(windowAnimationStyle.getResourceId(
        	com.android.internal.R.styleable.
            	WindowAnimation_activityOpenRemoteViewsEnterAnimation, 0));
		windowStyle.recycle();
        windowAnimationStyle.recycle();
	}
}
```

* 从上述代码可以看到，通过LayoutInflater去加载RemoteViews的布局文件。通过getLayoutId()方法获取布局文件。然后使用performApply方法去进行更新操作

#### 2.4.5 performApply

```java
private void performApply(View v, ViewGroup parent, OnClickHandler handler) {
	if (mActions != null) {
    	handler = handler == null ? DEFAULT_ON_CLICK_HANDLER : handler;
        final int count = mActions.size();
        for (int i = 0; i < count; i++) {
        	Action a = mActions.get(i);
            a.apply(v, parent, handler);
		}
	}
}
```

* 遍历之前放入的action集合mActions，然后去调用Action对象的apply方法

#### 2.4.6 apply(Action的子类ReflectionAction)

```java
@Override
public void apply(View root, ViewGroup rootParent, OnClickHandler handler) {
	final View view = root.findViewById(viewId);
    if (view == null) return;

    Class<?> param = getParameterType();
    if (param == null) {
    	throw new ActionException("bad type: " + this.type);
	}

    try {
    	getMethod(view, this.methodName, param).invoke(view, wrapArg(this.value));
	} catch (ActionException e) {
    	throw e;
	} catch (Exception ex) {
    	throw new ActionException(ex);
	}
}
```

* 从上述代码可以知道，ReflectionAction表示的是一个反射动作，通过它对View的操作会以反射的方式来调用，其中getMethod就是根据方法名来得到反射所需的Method对象。

#### 2.4.7 总结

* 这里是以setTextViewText举例的，其他方法大致上和这个类似，就不一一说明了，主要就是把操作放入mActions集合中，最后一块从集合中取出来，通过ReflectionAction反射的机制，对View进行更新。
* 也有一些方法不通过反射去完成的。

### 2.5 说明

#### 2.5.1 ListView和StackView

* 对于ListView和StackView，如果需要给item添加点击事件，则必须将setPendingIntentTemplate和setOnClickFillInIntent组合使用才可以

