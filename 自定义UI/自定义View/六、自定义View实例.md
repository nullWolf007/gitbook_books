[TOC]

## 六、自定义View实例

#### 转载

* [手把手教你写一个完整的自定义View](https://www.jianshu.com/p/e9d8420b1b9c)

### 一、自定义View的分类

* 自定义View一共分为两大类，具体如下图：

![img](https:////upload-images.jianshu.io/upload_images/944365-3b9e7aa2039f7075.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 二、具体介绍 & 使用场景

* 对于自定义View的类型介绍及使用场景如下图：

![img](https:////upload-images.jianshu.io/upload_images/944365-0b3407e5debd137e.png?imageMogr2/auto-orient/strip|imageView2/2/w/860/format/webp)

### 三、 使用注意点

* 在使用自定义View时有很多注意点（坑），希望大家要非常留意：

![img](https:////upload-images.jianshu.io/upload_images/944365-ab0d19c418ffeeaa.png?imageMogr2/auto-orient/strip|imageView2/2/w/900/format/webp)

#### 3.1 支持特殊属性

- 支持wrap_content
  *  如果不在`onMeasure（）`中对`wrap_content`作特殊处理，那么`wrap_content`属性将失效

> 具体原因请看文章：[为什么你的自定义View wrap_content不起作用？](https://www.jianshu.com/p/ca118d704b5e)

- 支持padding & margin
  * 如果不支持，那么`padding`和`margin`（ViewGroup情况）的属性将失效

> 1. 对于继承View的控件，padding是在draw()中处理
> 2. 对于继承ViewGroup的控件，padding和margin会直接影响measure和layout过程

#### 3.2 多线程应直接使用post方式

* View的内部本身提供了post系列的方法，完全可以替代Handler的作用，使用起来更加方便、直接。

#### 3.3 避免内存泄露

* 主要针对View中含有线程或动画的情况：**当View退出或不可见时，记得及时停止该View包含的线程和动画，否则会造成内存泄露问题**。

> 启动或停止线程/ 动画的方式：
>
> 1. 启动线程/ 动画：使用`view.onAttachedToWindow（）`，因为该方法调用的时机是当包含View的Activity启动的时刻
> 2. 停止线程/ 动画：使用`view.onDetachedFromWindow（）`，因为该方法调用的时机是当包含View的Activity退出或当前View被remove的时刻

#### 3.4 处理好滑动冲突

* 当View带有滑动嵌套情况时，必须要处理好滑动冲突，否则会严重影响View的显示效果。

### 四、 具体实例

* 接下来，我将用自定义View中最常用的**继承View**来说明自定义View的具体应用和需要注意的点

#### 4.1 继承View的介绍

![img](https:////upload-images.jianshu.io/upload_images/944365-502bad8cac77b8f5.png?imageMogr2/auto-orient/strip|imageView2/2/w/840/format/webp)

在下面的例子中，我将讲解：

- 如何实现一个基本的自定义View（继承View）
- 如何自身支持padding属性
- 如何为自定义View提供自定义属性（如颜色等等）
- 实例说明：画一个实心圆

#### 4.2 具体步骤

1. 创建自定义View类（继承View类）
2. 布局文件添加自定义View组件
3. 注意点设置（支持padding属性自定义属性等等）

下面我将逐个步骤进行说明：

#### 4.3 创建自定义View类

* CircleView.java

```java
public class CircleView extends View {
    // 设置画笔变量
    Paint mPaint = null;

    public CircleView(Context context) {
        super(context);
        // 在构造函数里初始化画笔的操作
        init();
    }

    public CircleView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        // 在构造函数里初始化画笔的操作
        init();
    }

    public CircleView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        // 在构造函数里初始化画笔的操作
        init();
    }

    @RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
    public CircleView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        // 在构造函数里初始化画笔的操作
        init();
    }

    // 画笔初始化
    private void init() {
        // 创建画笔
        mPaint = new Paint();
        // 设置画笔颜色为蓝色
        mPaint.setColor(Color.BLUE);
        // 设置画笔宽度为10px
        mPaint.setStrokeWidth(5f);
        //设置画笔模式为填充
		mPaint.setStyle(Paint.Style.FILL);
    }


    // 复写onDraw()进行绘制
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        // 获取控件的高度和宽度
        int width = getWidth();
        int height = getHeight();

        // 设置圆的半径 = 宽,高最小值的2分之1
        int r = Math.min(width, height) / 2;

        // 画出圆（蓝色）
        // 圆心 = 控件的中央,半径 = 宽,高最小值的2分之1
        canvas.drawCircle(width / 2f, height / 2f, r, mPaint);
    }
}
```

* 主要就是重写了onDraw()方法

#### 4.4 在xml中使用

* activity_main.xml

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <com.example.study.CircleView
        android:id="@+id/main_tv_start"
        android:layout_width="match_parent"
        android:layout_height="150dp"
        android:background="@color/colorAccent"
        android:gravity="center" />
</FrameLayout>
```

#### 4.5 在Activity中设置显示

* MainActivity.java

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

* 输出结果

<img src="..\..\images\自定义UI\自定义View\自定义View实例显示1.PNG" alt="自定义View实例显示1.PNG" style="zoom: 50%;" />

* 好了，至此，一个基本的自定义View已经实现了。
  * 如何手动支持padding属性
  * 如何为自定义View提供自定义属性（如颜色等等）

#### 4.6 支持padding属性

* `padding`属性：用于设置控件内容相对控件边缘的边距；
* 如果不手动设置支持padding属性，那么padding属性在自定义View中是不会生效的。

##### 4.6.1 解决方案

* 绘制时考虑传入的padding属性值（四个方向）。
  * 在自定义View类的复写onDraw()进行设置
* CircleView.java#onDraw

```java
public class CircleView extends View {
    ......
    // 复写onDraw()进行绘制
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        // 获取传入的padding值
        final int paddingLeft = getPaddingLeft();
        final int paddingRight = getPaddingRight();
        final int paddingTop = getPaddingTop();
        final int paddingBottom = getPaddingBottom();

        // 获取绘制内容的高度和宽度（考虑了四个方向的padding值）
        int width = getWidth() - paddingLeft - paddingRight;
        int height = getHeight() - paddingTop - paddingBottom;


        // 设置圆的半径 = 宽,高最小值的2分之1
        int r = Math.min(width, height) / 2;

        // 画出圆（蓝色）
        // 圆心 = 控件的中央,半径 = 宽,高最小值的2分之1
        canvas.drawCircle(paddingLeft + width / 2f, paddingTop + height / 2f, r, mPaint);
    }
}
```

* activity_main.xml

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <com.example.study.CircleView
        android:id="@+id/main_tv_start"
        android:layout_width="match_parent"
        android:layout_height="150dp"
        android:background="@color/colorAccent"
        android:gravity="center"
        android:padding="50dp" />
</FrameLayout>
```

* 输出结果

<img src="..\..\images\自定义UI\自定义View\自定义View实例显示2.PNG" alt="自定义View实例显示2.PNG" style="zoom: 50%;" />

#### 4.7 提供自定义属性

##### 4.7.1 使用步骤

* 在values目录下创建自定义属性的xml文件
* 在自定义View的构造方法中解析自定义属性的值
* 在布局文件中使用自定义属性

##### 4.7.2 在values目录下创建自定义属性的xml文件

* attr_circle_view.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!--自定义属性集合:CircleView-->
    <!--在该集合下,设置不同的自定义属性-->
    <declare-styleable name="CircleView">
        <!--在attr标签下设置需要的自定义属性-->
        <!--此处定义了一个设置图形的颜色:circle_color属性,格式是color,代表颜色-->
        <!--格式有很多种,如资源id(reference)等等-->
        <attr name="circle_color" format="color" />
        <!--此处定义了一个设置图形的style:circle_style属性,格式是enum,代表style-->
        <attr name="circle_style" format="integer" />

    </declare-styleable>
</resources>
```

* 自定义了两个属性

##### 4.7.3 在自定义View的构造方法中解析自定义属性的值

* CircleView.java

```java
public class CircleView extends View {
    // 设置画笔变量
    Paint mPaint = null;
    //画笔颜色
    int mColor = Color.BLUE;
    //圆的style
    int mStyle = 0;

    public CircleView(Context context) {
        this(context, null);
    }

    public CircleView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs,0);
    }

    public CircleView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        // 加载自定义属性集合CircleView
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleView);

        // 解析集合中的属性circle_color属性
        // 该属性的id为:R.styleable.CircleView_circle_color
        // 将解析的属性传入到画圆的画笔颜色变量当中（本质上是自定义画圆画笔的颜色）
        // 第二个参数是默认设置颜色（即无指定circle_color情况下使用）
        mColor = a.getColor(R.styleable.CircleView_circle_color, Color.RED);
        mStyle = a.getInt(R.styleable.CircleView_circle_style, 0);

        // 解析后释放资源
        a.recycle();
        // 在构造函数里初始化画笔的操作
        init();
    }

    // 画笔初始化
    private void init() {
        // 创建画笔
        mPaint = new Paint();
        // 设置画笔颜色为蓝色或自定义颜色
        mPaint.setColor(mColor);
        // 设置画笔宽度为10px
        mPaint.setStrokeWidth(5f);
        //设置画笔模式为填充或自定义格式
        if (mStyle == 0) {
            mPaint.setStyle(Paint.Style.FILL);
        } else if (mStyle == 1) {
            mPaint.setStyle(Paint.Style.STROKE);
        } else {
            mPaint.setStyle(Paint.Style.FILL_AND_STROKE);
        }
    }
	......
}
```

* 主要修改了构造函数，去解析了xml中的属性

##### 4.7.4 在布局文件中使用自定义属性

* activity_main.xml

```xml
 <com.example.study.CircleView
        android:id="@+id/main_tv_start"
        android:layout_width="match_parent"
        android:layout_height="150dp"
        android:background="@color/colorAccent"
        android:gravity="center"
        app:circle_color="#FFFFFF"
        app:circle_style="1"
        android:padding="50dp" />
```

* 显示结果

<img src="..\..\images\自定义UI\自定义View\自定义View实例显示3.PNG" alt="自定义View实例显示3.PNG" style="zoom: 50%;" />

* 至此，一个较为规范的自定义View已经完成了。