## Hook

#### 参考

* [Android进阶解密](https://book.douban.com/subject/30358046/)
* [Android：Hook技术之Hook Activity](https://juejin.im/post/6844903778148155400)

#### 一、Hook技术概述

* Hook技术的核心实际上是动态分析技术，动态分析是指在程序运行时对程序进行调试的技术。众所周知，Android系统的代码和回调是按照一定的顺序执行的，这里举一个简单的例子，如图所示。

![img](https://user-gold-cdn.xitu.io/2019/2/16/168f5c907c29cc3f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 对象A调用类对象B，对象B处理后将数据回调给对象A。接下来看看采用Hook的调用流程，如下图：

![img](https://user-gold-cdn.xitu.io/2019/2/16/168f5cd58cfff014?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 上图中的Hook可以是一个方法或者一个对象，它就想一个钩子一样，始终连着AB，在AB之间互传信息的时候，hook会在中间做一些处理，比如修改方法的参数和返回值等，就这样hook起到了欺上瞒下的作用，我们把hook的这种行为称之为劫持。同理，大家知道，系统进程和应该进程之间是相互独立的，应用进程要想直接去修改系统进程，这个是很难实现的，有了hook技术，就可以在进程之间进行行为更改了。如图所示：![img](https://user-gold-cdn.xitu.io/2019/2/16/168f5d999425ccb1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 可见，hook将自己融入到它所劫持的对象B所在的进程中，成为系统进程的一部分，这样我们就可以通过hook来更改对象B的行为了，对象B就称为hook点。

### 二、基础知识

#### 2.1 Hook的分类

##### 2.1.1 根据API语言

* Hook Java 主要通过反射和代理来实现，应用于在 SDK 开发环境中修改 Java 代码 
* Hook Native 则应用于在 NDK 开发环境和系统开发中修改 Native 代码。

##### 2.1.2 根据进程

* 应用程序进程 Hook 只能 Hook 当前所在的应用程序进程。
* 应用程序进程是 Zygote 进程 fork 出来的，如果对 Zygote 进行 Hook ，就可以实现 Hook 系统所有的应用程序进程，这就是全局 Hook

##### 2.1.3 根据实现

* 通过反射和代理实现，只能 Hook 当前的应用程序进程。 
* 通过 Hook 框架来实现，比如 Xposed ，可以实现全局 Hook ，但是需要 root

#### 2.2 代理模式

* 详情请查看[代理模式](Android必备技能/Hook/代理模式.md)

#### 2.3 Hook的思路

- **寻找 Hook 点**
  * 原则是静态变量和单例，因为一旦创建对象，它们不容易变化，非常容易定位。
  * 尽量 Hook public 的对象和方法。
- 选择合适的代理方式，如果是接口可以用动态代理。
- 偷梁换柱——用代理对象替换原始对象。

#### 2.4 注意点

* Android 的 API 版本比较多，方法和类可能不一样，所以要做好 API 的兼容工作。

### 三、实例1-Hook Activity的startActivity

* 我们要去Hook的话，最重要的就是要找到Hook点，所以我们需要通过源码去查看到一个合适的Hook点

#### 3.1 startActivity

* Activity#startActivity

```java
@Override
public void startActivity(Intent intent) {
	this.startActivity(intent, null);
}

@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
	if (options != null) {
    	startActivityForResult(intent, -1, options);
	} else {
        startActivityForResult(intent, -1);
	}
}
```

* startActivity方法最终会调用startActivityForResult方法

#### 3.2 startActivityForResult

```java
@Override
public void startActivityForResult(
		String who, Intent intent, int requestCode, @Nullable Bundle options) {
	Uri referrer = onProvideReferrer();
    if (referrer != null) {
    	intent.putExtra(Intent.EXTRA_REFERRER, referrer);
	}
    options = transferSpringboardActivityOptions(options);
    Instrumentation.ActivityResult ar =
    	mInstrumentation.execStartActivity(
        	this, mMainThread.getApplicationThread(), mToken, who,
            intent, requestCode, options);
	if (ar != null) {
    	mMainThread.sendActivityResult(
        	mToken, who, requestCode,
            ar.getResultCode(), ar.getResultData());
	}
    cancelInputsAndStartExitTransition(options);
}
```

* 最终使用mInstrumentation的execStartActivity方法来调用
* mInstrumentation是Instrumentation的实例，mInstrumentation是Activity的成员变量

* 我们这里就选择Instrumentation位Hook点，进行处理。用代理的Instrumentation类去替代原始的Instrumentation类

#### 3.3 InstrumentationProxy

```java
public class InstrumentationProxy extends Instrumentation {

    Instrumentation instrumentation;

    public InstrumentationProxy(Instrumentation instrumentation) {
        this.instrumentation = instrumentation;
    }

    public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,
                                            Intent intent, int requestCode, Bundle options) {
        Log.e("InstrumentationProxy", "Hook成功：" + "who-" + who);
        //通过反射找到Instrumentation的execStartActivity方法
        try {
            @SuppressLint("DiscouragedPrivateApi") Method execStartActivity = Instrumentation.class.
                    getDeclaredMethod("execStartActivity",
                            Context.class, IBinder.class, IBinder.class, Activity.class, Intent.class, int.class, Bundle.class);
            return (ActivityResult) execStartActivity.invoke(instrumentation, who, contextThread, token, target, intent, requestCode, options);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

* InstrumentationProxy继承自Instrumentation，同时创建同名execStartActivity方法，以便Activity调用的时候，调用的是自己实现的execStartActivity方法
* execStartActivity方法中通过反射调用了Instrumentation的execStartActivity方法
* InstrumentationProxy的execStartActivity方法相当于一个中间量，做了传递的效果。但是在传递的过程中，我们可以添加自己的操作，这就是Hook的本质。我们在此添加了一行打印的代码，标志我们的hook成功了

#### 3.4 MainActivity

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        replaceActivityInstrumentation(this);
        Intent intent = new Intent(this, TestActivity.class);
        startActivity(intent);
    }


    public void replaceActivityInstrumentation(Activity activity) {
        try {
            //得到Activity的mInstrumentation字段
            Field field = Activity.class.getDeclaredField("mInstrumentation");
            //取消Java的权限控制检查
            field.setAccessible(true);
            //得到Activity的Instrumentation对象
            Instrumentation instrumentation = (Instrumentation) field.get(activity);
            //创建代理类
            InstrumentationProxy instrumentationProxy = new InstrumentationProxy(instrumentation);
            //替换Activity的Instrumentation对象位代理类对象
            field.set(activity, instrumentationProxy);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

* 这里主要的就是replaceActivityInstrumentation方法，我们主要的操作就是找到Activity的Instrumentation对象instrumentation，然后使用反射的技术，把instrumentation替换成InstrumentationProxy的对象。

#### 3.5 注意点

* 这里的实例只是为了方便展示，其实还有更好的Hook点





