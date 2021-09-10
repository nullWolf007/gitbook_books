[TOC]
# AndroidAutoSize

## 一、使用

* [AndroidAutoSize使用说明](https://github.com/JessYanCoding/AndroidAutoSize/blob/master/README-zh.md)

## 二、源码解析

### 2.1 入口

#### 2.1.1 InitProvider

```java
public class InitProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        Context application = getContext().getApplicationContext();
        if (application == null) {
            application = AutoSizeUtils.getApplicationByReflect();
        }
        AutoSizeConfig.getInstance()
                .setLog(true)
                .init((Application) application)
                .setUseDeviceSize(false);
        return true;
    }
    //省略
}    
```

* 由于`ContentProvider`的`onCreate`是先于`Application`的`onCreate`执行的。所以可以做一些初始化的工作。
* 通过`getApplicationByReflect`反射的的方法去拿到了`Application`对象
* `init((Application) application)`这个方法就是进行初始化的主要代码，那我们跟踪一下这个方法。

#### 2.1.2 AutpSizeConfig#init()

```java
AutoSizeConfig init(Application application) {
	return init(application, true, null);
}

AutoSizeConfig init(final Application application, boolean isBaseOnWidth, AutoAdaptStrategy strategy) {
	//检查是否第一次进行init，只能初始化一次，不然会报错
	Preconditions.checkArgument(mInitDensity == -1, "AutoSizeConfig#init() can only be called once");
    Preconditions.checkNotNull(application, "application == null");
    this.mApplication = application;
    this.isBaseOnWidth = isBaseOnWidth;
    final DisplayMetrics displayMetrics = Resources.getSystem().getDisplayMetrics();
    final Configuration configuration = Resources.getSystem().getConfiguration();

    //设置一个默认值, 避免在低配设备上因为获取 MetaData 过慢, 导致适配时未能正常获取到设计图尺寸
    //建议使用者在低配设备上主动在 Application#onCreate 中调用 setDesignWidthInDp 替代以使用 AndroidManifest 配置设计图尺寸的方式
    if (AutoSizeConfig.getInstance().getUnitsManager().getSupportSubunits() == Subunits.NONE) {
    	mDesignWidthInDp = 360;
        mDesignHeightInDp = 640;
	} else {
		mDesignWidthInDp = 1080;
        mDesignHeightInDp = 1920;
	}
		
    //从AndroidManifest.xml中获取设置的宽高数据
    getMetaData(application);
    //判断屏幕方向
	isVertical = application.getResources().getConfiguration().orientation == Configuration.ORIENTATION_PORTRAIT;
    //获取屏幕宽高
    int[] screenSize = ScreenUtils.getScreenSize(application);
    mScreenWidth = screenSize[0];
    mScreenHeight = screenSize[1];
    mStatusBarHeight = ScreenUtils.getStatusBarHeight();

    //当前屏幕的density，densityDpi，scaledDensity信息
	mInitDensity = displayMetrics.density;
    mInitDensityDpi = displayMetrics.densityDpi;
    mInitScaledDensity = displayMetrics.scaledDensity;
    mInitXdpi = displayMetrics.xdpi;
    mInitScreenWidthDp = configuration.screenWidthDp;
    mInitScreenHeightDp = configuration.screenHeightDp;
    //监听系统的字体切换操作
    application.registerComponentCallbacks(new ComponentCallbacks() {
    	@Override
        public void onConfigurationChanged(Configuration newConfig) {
        	if (newConfig != null) {
            	if (newConfig.fontScale > 0) {
                mInitScaledDensity =
                	Resources.getSystem().getDisplayMetrics().scaledDensity;
				}
				isVertical = newConfig.orientation == Configuration.ORIENTATION_PORTRAIT;
                int[] screenSize = ScreenUtils.getScreenSize(application);
                mScreenWidth = screenSize[0];
                mScreenHeight = screenSize[1];
			}
		}

        @Override
        public void onLowMemory() {

        }
	});
    
    //添加ActivityLifecycleCallbacks，对Activity进行监听，然后和今日头条方案一样去修改density等值
	mActivityLifecycleCallbacks = new ActivityLifecycleCallbacksImpl(new WrapperAutoAdaptStrategy(strategy == null ? new DefaultAutoAdaptStrategy() : strategy));
    //注册监听Activity的生命周期
    application.registerActivityLifecycleCallbacks(mActivityLifecycleCallbacks);
    
    //兼容行处理Miui小米系统
    if ("MiuiResources".equals(application.getResources().getClass().getSimpleName()) || "XResources".equals(application.getResources().getClass().getSimpleName())) {
    	isMiui = true;
        try {
        	mTmpMetricsField = Resources.class.getDeclaredField("mTmpMetrics");
            mTmpMetricsField.setAccessible(true);
		} catch (Exception e) {
        	mTmpMetricsField = null;
		}
	}
    return this;
}
```

* 从上述代码中，我们可以知道对屏幕适配的主要逻辑应该在`ActivityLifecycleCallbacksImpl`这个类中去实现的，那我们去跟踪一下这个类

#### 2.1.3 ActivityLifecycleCallbacksImpl

```java
public class ActivityLifecycleCallbacksImpl implements Application.ActivityLifecycleCallbacks {
    //省略代码
	@Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        //区分出来Androidx和Andorid
        if (AutoSizeConfig.getInstance().isCustomFragment()) {
           //省略代码
        }

        //Activity 中的 setContentView(View) 一定要在 super.onCreate(Bundle); 之后执行
        if (mAutoAdaptStrategy != null) {
            mAutoAdaptStrategy.applyAdapt(activity, activity);
        }
    }
    //省略代码
}    
```

* 通过上面代码我们知道主要的逻辑还是在`mAutoAdaptStrategy`中，我们方向这是`AutoAdaptStrategy`类的实例，那我们就愮继续跟踪`AutoAdaptStrategy`类。

#### 2.1.4 AutoAdaptStrategy

```java
public interface AutoAdaptStrategy {

    /**
     * 开始执行屏幕适配逻辑
     *
     * @param target   需要屏幕适配的对象 (可能是 {@link Activity} 或者 Fragment)
     * @param activity 需要拿到当前的 {@link Activity} 才能修改 {@link DisplayMetrics#density}
     */
    void applyAdapt(Object target, Activity activity);
}
```

* 我们可以发现这是一个接口类，找到其实现类。我们发现是两个实现类`WrapperAutoAdaptStrategy`和`DefaultAutoAdaptStrategy`两个实现类。

* 由上面的`AutpSizeConfig#init()`代码可知

  ```java
  mActivityLifecycleCallbacks = new ActivityLifecycleCallbacksImpl(new WrapperAutoAdaptStrategy(strategy == null ? new DefaultAutoAdaptStrategy() : strategy));
  
  ```

* `WrapperAutoAdaptStrategy`只是传入了`DefaultAutoAdaptStrategy`对象进去，所以`WrapperAutoAdaptStrategy`知识相当于包装类，真正实现类还是`DefaultAutoAdaptStrategy`类。所以我们一起看看`DefaultAutoAdaptStrategy`类。

#### 2.1.5 DefaultAutoAdaptStrategy

```java
public class DefaultAutoAdaptStrategy implements AutoAdaptStrategy {
    @Override
    public void applyAdapt(Object target, Activity activity) {
        //检查是否开启了外部三方库的适配模式,
        //只要不主动调用 ExternalAdaptManager 的方法, 下面的代码就不会执行
        if (AutoSizeConfig.getInstance().getExternalAdaptManager().isRun()) {
			//省略代码
        }

        //如果 target 实现 CancelAdapt 接口表示放弃适配, 所有的适配效果都将失效
        if (target instanceof CancelAdapt) {
            AutoSize.cancelAdapt(activity);
            return;
        }

        //如果 target 实现 CustomAdapt 接口
        //表示该 target 想自定义一些用于适配的参数, 从而改变最终的适配效果
        if (target instanceof CustomAdapt) {
            AutoSize.autoConvertDensityOfCustomAdapt(activity, (CustomAdapt) target);
        } else {
            AutoSize.autoConvertDensityOfGlobal(activity);
        }
    }
}
```

* 上面的`CustomAdapt`就相当于自定义的`Adapte`，是为了解决当某个`Activity`的设计尺寸与 AndroidManifest 中填写的全局设计图尺寸不同时，可以实现 CustomAdapt 接口扩展适配参数。这个`CustomAdapt`的使用说明，官方说明文档已经很清楚了，这里就不说明了。
* 那默认的就是`AutoSize.autoConvertDensityOfGlobal(activity);`这行代码了，我们需要继续追踪代码。查看`AutoSize#autoConvertDensityOfGlobal`方法

#### 2.1.6 AutoSize

**autoConvertDensityOfGlobal**

```java
public static void autoConvertDensityOfGlobal(Activity activity) {
    //判断是以width为基准还是height为基准
	if (AutoSizeConfig.getInstance().isBaseOnWidth()) {
    	autoConvertDensityBaseOnWidth(activity, AutoSizeConfig.getInstance().getDesignWidthInDp());
	} else {
    	autoConvertDensityBaseOnHeight(activity, AutoSizeConfig.getInstance().getDesignHeightInDp());
	}
}
```

* 一般是以width为基准的，所以我们继续跟踪`autoConvertDensityBaseOnWidth`方法

**autoConvertDensityBaseOnWidth**

```java
public static void autoConvertDensityBaseOnWidth(Activity activity, float designWidthInDp) {
	autoConvertDensity(activity, designWidthInDp, true);
}
```

* 继续跟踪`autoConvertDensity(activity, designWidthInDp, true);`方法

**autoConvertDensity**

```java
public static void autoConvertDensity(Activity activity, float sizeInDp, boolean isBaseOnWidth) {
	Preconditions.checkNotNull(activity, "activity == null");
    Preconditions.checkMainThread();
	
    //获取base的值，如果是BaseOnWidth就获取设计图基准width的值
    float subunitsDesignSize = isBaseOnWidth ? AutoSizeConfig.getInstance().getUnitsManager().getDesignWidth()
    	: AutoSizeConfig.getInstance().getUnitsManager().getDesignHeight();
	subunitsDesignSize = subunitsDesignSize > 0 ? subunitsDesignSize : sizeInDp;

    //获取屏幕的尺寸
    int screenSize = isBaseOnWidth ? AutoSizeConfig.getInstance().getScreenWidth()
    	: AutoSizeConfig.getInstance().getScreenHeight();

    //获取key值，因为存在mCache中，key的生成规则和put的时候一致即可
	int key = Math.round((sizeInDp + subunitsDesignSize + screenSize) * AutoSizeConfig.getInstance().getInitScaledDensity()) & ~MODE_MASK;
    key = isBaseOnWidth ? (key | MODE_ON_WIDTH) : (key & ~MODE_ON_WIDTH);
    key = AutoSizeConfig.getInstance().isUseDeviceSize() ? (key | MODE_DEVICE_SIZE) : (key & ~MODE_DEVICE_SIZE);
	
    //根据key中从mCache中获取DisplayMetricsInfo对象
    DisplayMetricsInfo displayMetricsInfo = mCache.get(key);

    //目标变量
    float targetDensity = 0;
    int targetDensityDpi = 0;
    float targetScaledDensity = 0;
    float targetXdpi = 0;
    int targetScreenWidthDp;
    int targetScreenHeightDp;

    //如果mCache中没有缓存，我们需要自己去处理，生成然后put到mCache中去
    if (displayMetricsInfo == null) {
        //计算density等值，和今日头条方案一致
        //计算targetDensity
    	if (isBaseOnWidth) {
            //屏幕的总宽度px/Mainfest中配置的dp值 其他类似
        	targetDensity = AutoSizeConfig.getInstance().getScreenWidth() * 1.0f / sizeInDp;
		} else {
        	targetDensity = AutoSizeConfig.getInstance().getScreenHeight() * 1.0f / sizeInDp;
		}
        //计算targetScaledDensity
        if (AutoSizeConfig.getInstance().getPrivateFontScale() > 0) {
        	targetScaledDensity = targetDensity * AutoSizeConfig.getInstance().getPrivateFontScale();
		} else {
        	float systemFontScale = AutoSizeConfig.getInstance().isExcludeFontScale() ? 1 : AutoSizeConfig.getInstance().
            getInitScaledDensity() * 1.0f / AutoSizeConfig.getInstance().getInitDensity();
            targetScaledDensity = targetDensity * systemFontScale;
		}
        //计算targetDensityDpi
        targetDensityDpi = (int) (targetDensity * 160);
		//目标总宽度dp
        targetScreenWidthDp = (int) (AutoSizeConfig.getInstance().getScreenWidth() / targetDensity);
        //目标总高度dp
        targetScreenHeightDp = (int) (AutoSizeConfig.getInstance().getScreenHeight() / targetDensity);
        
        if (isBaseOnWidth) {
            //屏幕宽度px/基准的宽度dp
        	targetXdpi = AutoSizeConfig.getInstance().getScreenWidth() * 1.0f / subunitsDesignSize;
		} else {
        	targetXdpi = AutoSizeConfig.getInstance().getScreenHeight() * 1.0f / subunitsDesignSize;
		}
		
        //把数据放入缓存中
        mCache.put(key, new DisplayMetricsInfo(targetDensity, targetDensityDpi, targetScaledDensity, targetXdpi, targetScreenWidthDp, targetScreenHeightDp));
	} else {//如果mCache缓存中有的话，直接去除缓存中的数据
    	targetDensity = displayMetricsInfo.getDensity();
        targetDensityDpi = displayMetricsInfo.getDensityDpi();
        targetScaledDensity = displayMetricsInfo.getScaledDensity();
        targetXdpi = displayMetricsInfo.getXdpi();
        targetScreenWidthDp = displayMetricsInfo.getScreenWidthDp();
        targetScreenHeightDp = displayMetricsInfo.getScreenHeightDp();
	}
	
    //设置density等值
    setDensity(activity, targetDensity, targetDensityDpi, targetScaledDensity, targetXdpi);
    setScreenSizeDp(activity, targetScreenWidthDp, targetScreenHeightDp);
    }
```

* 从代码中我们可以看到已经计算出来各种目标值，并且放入缓存中了，现在就差进行设置了，继续跟踪`setDesnity方法`

**setDensity**

```java
private static void setDensity(Activity activity, float density, int densityDpi, float scaledDensity, float xdpi) {
    //对Activity的DisplayMetrics进行设置
	DisplayMetrics activityDisplayMetrics = activity.getResources().getDisplayMetrics();
    setDensity(activityDisplayMetrics, density, densityDpi, scaledDensity, xdpi);
	//对Application的DisplayMetrics进行设置
    DisplayMetrics appDisplayMetrics = AutoSizeConfig.getInstance().getApplication().getResources().getDisplayMetrics();
    setDensity(appDisplayMetrics, density, densityDpi, scaledDensity, xdpi);

    //兼容 MIUI
    DisplayMetrics activityDisplayMetricsOnMIUI = getMetricsOnMiui(activity.getResources());
    DisplayMetrics appDisplayMetricsOnMIUI = getMetricsOnMiui(AutoSizeConfig.getInstance().getApplication().getResources());

    if (activityDisplayMetricsOnMIUI != null) {
    	setDensity(activityDisplayMetricsOnMIUI, density, densityDpi, scaledDensity, xdpi);
	}
    if (appDisplayMetricsOnMIUI != null) {
    	setDensity(appDisplayMetricsOnMIUI, density, densityDpi, scaledDensity, xdpi);
	}
}
```

```java
private static void setDensity(DisplayMetrics displayMetrics, float density, int densityDpi, float scaledDensity, float xdpi) {
    //对density和densityDpi进行设置，也就是dp
	if (AutoSizeConfig.getInstance().getUnitsManager().isSupportDP()) {
    	displayMetrics.density = density;
        displayMetrics.densityDpi = densityDpi;
	}
	//对scaledDensity进行设置，也就是sp，字体大小
    if (AutoSizeConfig.getInstance().getUnitsManager().isSupportSP()) {
    	displayMetrics.scaledDensity = scaledDensity;
	}
    switch (AutoSizeConfig.getInstance().getUnitsManager().getSupportSubunits()) {
    	case NONE:
        	break;
		case PT:
        	displayMetrics.xdpi = xdpi * 72f;
            break;
		case IN:
        	displayMetrics.xdpi = xdpi;
            break;
		case MM:
        	displayMetrics.xdpi = xdpi * 25.4f;
            break;
		default:
	}
}
```

* 我们终于找到了入口的流程，一不不的跟踪代码，发现了真正修改**density,densityDpi,scaledDesnity**的地方了。其核心代码也是和今日头条适配方案是一致的，提供了许多的封装处理，用户直接操作api即可，方便了用户的使用。

### 2.2 获取Meta数据

```java
private void getMetaData(final Context context) {
	new Thread(new Runnable() {
    	@Override
        public void run() {
        	PackageManager packageManager = context.getPackageManager();
            ApplicationInfo applicationInfo;
            try {
            	applicationInfo = packageManager.getApplicationInfo(context
                	.getPackageName(), PackageManager.GET_META_DATA);
                if (applicationInfo != null && applicationInfo.metaData != null) {
                    if (applicationInfo.metaData.containsKey(KEY_DESIGN_WIDTH_IN_DP)) {
                    	mDesignWidthInDp = (int) applicationInfo.metaData.get(KEY_DESIGN_WIDTH_IN_DP);
					}
                    if (applicationInfo.metaData.containsKey(KEY_DESIGN_HEIGHT_IN_DP)) {
                    	mDesignHeightInDp = (int) applicationInfo.metaData.get(KEY_DESIGN_HEIGHT_IN_DP);
					}
				}
			} catch (PackageManager.NameNotFoundException e) {
            	e.printStackTrace();
			}
		}
	}).start();
}
```

* 通过`applicationInfo.metaData.get`来获取对应的值，并存入到`mDesignWidthInDp`和`mDesignHeightInDp`中。

