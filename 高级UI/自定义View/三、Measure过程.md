[TOC]

## Measure过程

#### 转载

* [自定义View Measure过程 - 最易懂的自定义View原理系列（2）](https://www.jianshu.com/p/1dab927b2f36)

![img](https:////upload-images.jianshu.io/upload_images/944365-64faaebdceacd3ba.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 一、概述

#### 1.1 Measure的作用

* 测量`View`的宽 / 高
  * 在某些情况下，需要多次测量`（measure）`才能确定`View`最终的宽/高；
  * 该情况下，`measure`过程后得到的宽 / 高可能不准确；
  * 此处建议：在`layout`过程中`onLayout()`去获取最终的宽 / 高

### 二、 储备知识

了解`measure`过程前，需要先了解传递尺寸（宽 / 高测量值）的2个类：

- `ViewGroup.LayoutParams`类（）
- `MeasureSpecs` 类（父视图对子视图的测量要求）

#### 2.1 ViewGroup.LayoutParams

##### 2.1.1 简介

- 布局参数类

* `ViewGroup` 的子类`（RelativeLayout、LinearLayout）`有其对应的 `ViewGroup.LayoutParams` 子类，如：`RelativeLayout`的 `ViewGroup.LayoutParams`子类 `RelativeLayoutParams`

##### 2.1.2 作用

- 指定视图`View` 的高度`（height）`  和 宽度`（width）`等布局参数。

##### 2.1.3 具体使用

- 
  通过以下参数指定

| 参数         |                             解释                             |
| ------------ | :----------------------------------------------------------: |
| 具体值       |                           dp / px                            |
| fill_parent  |  强制性使子视图的大小扩展至与父视图大小相等（不含 padding )  |
| match_parent |        与fill_parent相同，用于Android 2.3 & 之后版本         |
| wrap_content | 自适应大小，强制性地使视图扩展以便显示其全部内容(含 padding ) |

```xml
android:layout_height="wrap_content"   //自适应大小  
android:layout_height="match_parent"   //与父视图等高  
android:layout_height="fill_parent"    //与父视图等高  
android:layout_height="100dp"         //精确设置高度值为 100dp  
```

##### 2.1.4 构造函数

- 构造函数 = `View`的入口，可用于初始化 & 获取自定义属性

```java
// 如果View是在Java代码里面new的，则调用第一个构造函数
public CarsonView(Context context) {
	super(context);
}

// 如果View是在.xml里声明的，则调用第二个构造函数
// 自定义属性是从AttributeSet参数传进来的
public CarsonView(Context context, AttributeSet attrs) {
	super(context, attrs);
}

// 不会自动调用
// 一般是在第二个构造函数里主动调用
// 如View有style属性时
public CarsonView(Context context, AttributeSet attrs, int defStyleAttr) {
	super(context, attrs, defStyleAttr);
}

//API21之后才使用
// 不会自动调用
// 一般是在第二个构造函数里主动调用
// 如View有style属性时
public  CarsonView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
	super(context, attrs, defStyleAttr, defStyleRes);
}
```

#### 2.2 MeasureSpec

##### 2.2.1 简介

![img](https:////upload-images.jianshu.io/upload_images/944365-0cf0a1ffd083cad1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

##### 2.2.2 组成

* 测量规格`（MeasureSpec）` = 测量模式`（mode）` + 测量大小`（size）`

![img](https:////upload-images.jianshu.io/upload_images/944365-7d0f873cee3912bb.png?imageMogr2/auto-orient/strip|imageView2/2/w/380/format/webp)

* 其中，测量模式`（Mode）`的类型有3种：UNSPECIFIED、EXACTLY 和 AT_MOST。具体如下：

![img](https:////upload-images.jianshu.io/upload_images/944365-e631b96ea1906e34.png?imageMogr2/auto-orient/strip|imageView2/2/w/960/format/webp)

##### 2.2.3 具体使用

- `MeasureSpec` 被封装在`View`类中的一个内部类里：`MeasureSpec`类
- MeasureSpec类 用1个变量封装了2个数据`（size，mode）`：**通过使用二进制**，将测量模式`（mode）` & 测量大小`(size）`打包成一个`int`值来，并提供了打包 & 解包的方法
  * 该措施的目的 = 减少对象内存分配
- 实际使用

```cpp
/**
  * MeasureSpec类的具体使用
  **/
// 1. 获取测量模式（Mode）
int specMode = MeasureSpec.getMode(measureSpec)

// 2. 获取测量大小（Size）
int specSize = MeasureSpec.getSize(measureSpec)

// 3. 通过Mode 和 Size 生成新的SpecMode
int measureSpec=MeasureSpec.makeMeasureSpec(size, mode);
```

- 源码分析

```java
/**
  * MeasureSpec类的源码分析
  **/
public class MeasureSpec {

	// 进位大小 = 2的30次方
    // int的大小为32位，所以进位30位 = 使用int的32和31位做标志位
    private static final int MODE_SHIFT = 30;  
          
    // 运算遮罩：0x3为16进制，10进制为3，二进制为11
    // 3向左进位30 = 11 00000000000(11后跟30个0)  
    // 作用：用1标注需要的值，0标注不要的值。因1与任何数做与运算都得任何数、0与任何数做与运算都得0
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
  
    // UNSPECIFIED的模式设置：0向左进位30 = 00后跟30个0，即00 00000000000
    // 通过高2位
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
        
    // EXACTLY的模式设置：1向左进位30 = 01后跟30个0 ，即01 00000000000
    public static final int EXACTLY = 1 << MODE_SHIFT;  

    // AT_MOST的模式设置：2向左进位30 = 10后跟30个0，即10 00000000000
    public static final int AT_MOST = 2 << MODE_SHIFT;  
  
    /**
    * makeMeasureSpec（）方法
    * 作用：根据提供的size和mode得到一个详细的测量结果吗，即measureSpec
    **/ 
    public static int makeMeasureSpec(int size, int mode) {  
		return size + mode;  
        // measureSpec = size + mode；此为二进制的加法 而不是十进制
        // 设计目的：使用一个32位的二进制数，其中：32和31位代表测量模式（mode）、后30位代表测量大小（size）
        // 例如size=100(4)，mode=AT_MOST，则measureSpec=100+10000...00=10000..00100  
	}  
      
    /**
    * getMode（）方法
    * 作用：通过measureSpec获得测量模式（mode）
    **/    
	public static int getMode(int measureSpec) {  
		return (measureSpec & MODE_MASK);  
        // 即：测量模式（mode） = measureSpec & MODE_MASK;  
        // MODE_MASK = 运算遮罩 = 11 00000000000(11后跟30个0)
        //原理：保留measureSpec的高2位（即测量模式）、使用0替换后30位
        // 例如10 00..00100 & 11 00..00(11后跟30个0) = 10 00..00(AT_MOST)，这样就得到了mode的值
	}  
    
    /**
    * getSize方法
    * 作用：通过measureSpec获得测量大小size
    **/       
    public static int getSize(int measureSpec) {  
		return (measureSpec & ~MODE_MASK);  
        // size = measureSpec & ~MODE_MASK;  
        // 原理类似上面，即 将MODE_MASK取反，也就是变成了00 111111(00后跟30个1)，将32,31替换成0也就是去掉mode，保留后30位的size  
	} 
}  
```

##### 2.2.4 MeasureSpec值的计算

- 上面讲了那么久`MeasureSpec`，那么`MeasureSpec`值到底是如何计算得来?
- 结论：子View的`MeasureSpec`值根据**子View的布局参数（LayoutParams）和父容器的MeasureSpec值**计算得来的，具体计算逻辑封装在`getChildMeasureSpec()`里。如下图：

![img](https:////upload-images.jianshu.io/upload_images/944365-d059b1afdeae0256.png?imageMogr2/auto-orient/strip|imageView2/2/w/470/format/webp)

* 对于普通View
  * 即：子`view`的大小由父`view`的`MeasureSpec`值 和 子`view`的`LayoutParams`属性 共同决定
* 下面，我们来看`getChildMeasureSpec()`的源码分析：

```dart
/**
* 源码分析：getChildMeasureSpec（）
* 作用：根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
* 注：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定
**/
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  
	//参数说明
    * @param spec 父view的详细测量值(MeasureSpec) 
    * @param padding view当前尺寸的的内边距和外边距(padding,margin) 
    * @param childDimension 子视图的布局参数（宽/高）

    //父view的测量模式
    int specMode = MeasureSpec.getMode(spec);     

    //父view的大小
    int specSize = MeasureSpec.getSize(spec);     
          
    //通过父view计算出的子view = 父大小-边距（父要求的大小，但子view不一定用这个值）   
    int size = Math.max(0, specSize - padding);  
          
    //子view想要的实际大小和模式（需要计算）  
    int resultSize = 0;  
    int resultMode = 0;  
          
    //通过父view的MeasureSpec和子view的LayoutParams确定子view的大小  
     switch (specMode) {  
     	case MeasureSpec.EXACTLY:  
            //当父view的模式为EXACITY时，父view强加给子view确切的值
            //一般是父view设置为match_parent或者固定值的ViewGroup  
            if (childDimension >= 0) {  
                //当子view的LayoutParams>0，即有确切的值  
            	//子view大小为子自身所赋的值，模式大小为EXACTLY  
                resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
			} else if (childDimension == LayoutParams.MATCH_PARENT) {  
                // 当子view的LayoutParams为MATCH_PARENT时(-1)  
            	//子view大小为父view大小，模式为EXACTLY  
                resultSize = size;  
                resultMode = MeasureSpec.EXACTLY;  
			} else if (childDimension == LayoutParams.WRAP_CONTENT) {  
				//当子view的LayoutParams为WRAP_CONTENT时(-2)      
				//子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
			}  
            break;  
  
		case MeasureSpec.AT_MOST:  
             // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）
			// 道理同上  
            if (childDimension >= 0) {  
            	resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
			} else if (childDimension == LayoutParams.MATCH_PARENT) {  
            	resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
			} else if (childDimension == LayoutParams.WRAP_CONTENT) {  
            	resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
			}  
			break;  
             
		case MeasureSpec.UNSPECIFIED:  
             // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
             // 多见于ListView、GridView  
			if (childDimension >= 0) {  
				// 子view大小为子自身所赋的值  
                resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
			} else if (childDimension == LayoutParams.MATCH_PARENT) {  
            	// 因为父view为UNSPECIFIED，所以MATCH_PARENT的话子类大小为0  
                resultSize = 0;  
                resultMode = MeasureSpec.UNSPECIFIED;  
			} else if (childDimension == LayoutParams.WRAP_CONTENT) {  
            	// 因为父view为UNSPECIFIED，所以WRAP_CONTENT的话子类大小为0  
                resultSize = 0;  
                resultMode = MeasureSpec.UNSPECIFIED;  
			}  
            break;  
	}  
	return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
}  
```

- 关于`getChildMeasureSpec()`里对子`View`的测量模式 & 大小的判断逻辑有点复杂；
- 别担心，我已帮大家总结好。具体请看下表：

![img](https:////upload-images.jianshu.io/upload_images/944365-76261325e6576361.png?imageMogr2/auto-orient/strip|imageView2/2/w/751/format/webp)

* 其中的规律总结：（以子`View`为标准，横向观察）

![img](https:////upload-images.jianshu.io/upload_images/944365-6088d2d291bbae09.png?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)



* 由于`UNSPECIFIED`模式适用于系统内部多次`measure`情况，很少用到，故此处不讨论

#### 2.3 补充/总结

##### 2.3.1 顶级View

- 顶级`View`（即`DecorView`）的测量规格`MeasureSpec`计算逻辑：取决于 **自身布局参数 & 窗口尺寸**

![img](https:////upload-images.jianshu.io/upload_images/944365-560312570515aef4.png?imageMogr2/auto-orient/strip|imageView2/2/w/470/format/webp)

### 三、Measure过程

#### 3.1 分类

- `measure`过程 根据**View的类型**分为2种情况：

![img](https://upload-images.jianshu.io/upload_images/944365-556bf094df91b9de.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

* 下面将分析这两种情况

### 四、单一View的Measure过程

#### 4.1 应用场景

- 在无现成的控件`View`满足需求、需自己实现时，则使用自定义单一`View`
  * 如：制作一个支持加载网络图片的`ImageView`控件
  * 注：自定义`View`在多数情况下都有替代方案：图片 / 组合动画，但二者可能会导致内存耗费过大，从而引起内存溢出等问题。

#### 4.2 具体使用 

*  继承自`View`、`SurfaceView` 或 其他`View`；不包含子`View`

#### 4.3 具体流程

![img](https:////upload-images.jianshu.io/upload_images/944365-6fd614936d045071.png?imageMogr2/auto-orient/strip|imageView2/2/w/150/format/webp)

#### 4.4  源码分析

##### 4.4.1 View#measure

```java
/**
  * 源码分析：measure（）
  * 定义：Measure过程的入口；属于View.java类 & final类型，即子类不能重写此方法
  * 作用：基本测量逻辑的判断
**/ 
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 参数说明：View的宽 / 高测量规格
    ......
	final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
	......
	int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
	if (cacheIndex < 0 || sIgnoreMeasureCache) {
    	//计算视图大小
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	} else {
        .....
	}
    ......
}
```

* 最重要的是调用了onMeasure方法，此方法主要是计算视图大小的

##### 4.4.2 View#onMeasure

```java
/**
  * 分析1：onMeasure（）
  * 作用：a. 根据View宽/高的测量规格计算View的宽/高值：getDefaultSize()
  *      b. 存储测量后的View宽 / 高：setMeasuredDimension()
**/ 
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
	// 参数说明：View的宽 / 高测量规格
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
    // setMeasuredDimension() ：获得View宽/高的测量值 ->>分析2
    // 传入的参数通过getDefaultSize()获得 ->>分析3
}
```

* 主要就是调用了getDefaultSize方法去获取参数，然后调用setMeasuredDimension去设置

##### 4.4.3 View#getDefaultSize

```java
/**
  * 分析3：getDefaultSize()
  * 作用：根据View宽/高的测量规格计算View的宽/高值
**/
public static int getDefaultSize(int size, int measureSpec) {  
	// 参数说明：
    // size：提供的默认大小
    // measureSpec：宽/高的测量规格（含模式 & 测量大小）

    // 设置默认大小
    int result = size; 
            
    // 获取宽/高测量规格的模式 & 测量大小
    int specMode = MeasureSpec.getMode(measureSpec);  
    int specSize = MeasureSpec.getSize(measureSpec);  
          
    switch (specMode) {  
    	// 模式为UNSPECIFIED时，使用提供的默认大小 = 参数Size
        case MeasureSpec.UNSPECIFIED:  
        	result = size;  
            break;  
		// 模式为AT_MOST,EXACTLY时，使用View测量后的宽/高值 = measureSpec中的Size
        case MeasureSpec.AT_MOST:  
        case MeasureSpec.EXACTLY:  
        	result = specSize;  
            break;  
	}  

	// 返回View的宽/高值
    return result;  
}  
```

##### 4.4.4 View#setMeasuredDimension

```java
/**
  * 分析2：setMeasuredDimension()
  * 作用：存储测量后的View宽 / 高
  * 注：该方法即为我们重写onMeasure()所要实现的最终目的
  **/
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
	boolean optical = isLayoutModeOptical(this);
    //判断是否需要对measuredWidth和measuredHeight进行处理
    if (optical != isLayoutModeOptical(mParent)) {
    	Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;
		
        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
	}
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

//参数说明：测量后子View的宽 / 高值
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    // 将测量后子View的宽 / 高值进行传递
	mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

##### 4.4.5 View#getSuggestedMinimumWidth

* 在4.4.2中调用了此方法获取默认的宽度，获取默认的高度同理，就不进行说明了
* 在4.4.3中，当specMode为UNSPECIFIED不约束的时候，就是用此方法获取的默认值大小

```java
protected int getSuggestedMinimumWidth() {
	return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

* 若mBackground等于null，即此View没有背景的话，默认宽度就是mMinWidth
  * mMinWidth= `android:minWidth`属性所指定的值；
  * 若`android:minWidth`没指定，则默认为0
* 若设置了背景，此View的宽度为mMinWidth和mBackground.getMinimumWidth()中的较大值
* mBackground是Drawable对象

##### 4.4.6 Drawable#getMinimumWidth

```java
public int getMinimumWidth() {
	final int intrinsicWidth = getIntrinsicWidth();
    //返回背景图Drawable的原始宽度
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

* 由源码可知：mBackground.getMinimumWidth()的大小 = 背景图Drawable的原始宽度
*  若无原始宽度，则为0；
*  注：BitmapDrawable有原始宽度，而ShapeDrawable没有

##### 4.4.7 `getDefaultSize()`计算View的宽/高值的逻辑

![img](https://upload-images.jianshu.io/upload_images/944365-bf6b3dc2261012dc.png?imageMogr2/auto-orient/strip|imageView2/2/w/970/format/webp)

### 五、ViewGroup的Measure过程

#### 5.1 应用场景

* 利用现有的组件根据特定的布局方式来组成新的组件 

#### 5.2 具体使用

- 继承自`ViewGroup` 或 各种`Layout`；含有子 `View`

#### 5.3 实例

* 如：底部导航条中的条目，一般都是上图标(ImageView)、下文字(TextView)，那么这两个就可以用自定义ViewGroup组合成为一个Veiw，提供两个属性分别用来设置文字和图片，使用起来会更加方便。

![img](https:////upload-images.jianshu.io/upload_images/944365-3eb8d5d9d7d9e9fd.png?imageMogr2/auto-orient/strip|imageView2/2/w/504/format/webp)

#### 5.4 原理

* 遍历 测量所有子`View`的尺寸

* 合并将所有子`View`的尺寸进行，最终得到`ViewGroup`父视图的测量值
  * 自上而下、一层层地传递下去，直到完成整个`View`树的`measure（）`过程

![img](https:////upload-images.jianshu.io/upload_images/944365-7133935cb1e56190.png?imageMogr2/auto-orient/strip|imageView2/2/w/661/format/webp)

#### 5.5 流程

![img](https:////upload-images.jianshu.io/upload_images/944365-1438a7fbd93d0987.png?imageMogr2/auto-orient/strip|imageView2/2/w/569/format/webp)

#### 5.6 源码

* 若需进行自定义`ViewGroup`，则需重写`onMeasure()`
* 入口 = `measure（）`

##### 5.6.1 View#measure

* 由于ViewGroup继承自View，同时ViewGroup中没有实现measure方法而且也无法重写(由于final)，所以调用的是View的measure方法

```java
@UiThread
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
```

```java
/**
  * 源码分析：measure（）
  * 定义：Measure过程的入口；属于View.java类 & final类型，即子类不能重写此方法
  * 作用：基本测量逻辑的判断
**/ 
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 参数说明：View的宽 / 高测量规格
    ......
	final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
	......
	int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
	if (cacheIndex < 0 || sIgnoreMeasureCache) {
    	//计算视图大小
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	} else {
        .....
	}
    ......
}
```

* 最终会调用onMeasure方法

##### 5.6.2 View#onMeasure

* ViewGroup也没有重写onMeasure方法，所以还是使用的View的onMeasure方法

* 但是LinearLayout等布局继承自ViewGroup都自己重写了onMeasure方法的

```java
/**
  * 分析1：onMeasure（）
  * 作用：a. 根据View宽/高的测量规格计算View的宽/高值：getDefaultSize()
  *      b. 存储测量后的View宽 / 高：setMeasuredDimension()
**/ 
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
	// 参数说明：View的宽 / 高测量规格
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
    // setMeasuredDimension() ：获得View宽/高的测量值 ->>分析2
    // 传入的参数通过getDefaultSize()获得 ->>分析3
}
```

* onMeasure在ViewGroup中是无法使用的，因为不同的`ViewGroup`子类（`LinearLayout`、`RelativeLayout` / 自定义`ViewGroup`子类等）具备不同的布局特性，这导致他们子`View`的测量方法各有不同，所以没有办法同一规定onMeasure方法

##### 5.6.3 ViewGroup和View的onMeasure区别

* **ViewGroup无法对onMeasure（）作统一实现。这个也是单一View的measure过程与ViewGroup过程最大的不同。**

* 单一`View measure`过程的`onMeasure（）`具有统一实现，而`ViewGroup`则没有

* 其实，在单一`View measure`过程中，`getDefaultSize()`只是简单的测量了宽高值，在实际使用时有时需更精细的测量。所以有时候也需重写`onMeasure（）`

##### 5.6.4 重写onMeasure

* 在自定义`ViewGroup`中，关键在于：**根据需求重写`onMeasure()`从而实现你的子View测量逻辑**。重写`onMeasure()`的套路如下：

```java
/**
  * 根据自身的测量逻辑复写onMeasure（），分为3步
  * 1. 遍历所有子View & 测量：measureChildren（）
  * 2. 合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）
  * 3. 存储测量后View宽/高的值：调用setMeasuredDimension()  
  **/ 

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  

	// 定义存放测量后的View宽/高的变量
    int widthMeasure ;
    int heightMeasure ;

    // 1. 遍历所有子View & 测量(measureChildren（）)
    // ->> 分析1
    measureChildren(widthMeasureSpec, heightMeasureSpec)；

    // 2. 合并所有子View的尺寸大小，最终得到ViewGroup父视图的测量值
    void measureCarson{
    	... // 自身实现
	}

    // 3. 存储测量后View宽/高的值：调用setMeasuredDimension()  
    // 类似单一View的过程，此处不作过多描述
    setMeasuredDimension(widthMeasure,  heightMeasure);  
}
```

* 步骤

  1.遍历所有子View & 测量：measureChildren（）

  2.合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）

  3.存储测量后View宽/高的值：调用setMeasuredDimension()  

* 其中第一步和第三步调用的都是系统的api，第二步是需要自己实现的

##### 5.6.5 ViewGroup#measureChildren

```java
/**
  * 分析1：measureChildren()
  * 作用：遍历子View & 调用measureChild()进行下一步测量
  **/ 
// 参数说明：父视图的测量规格（MeasureSpec）
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
	final int size = mChildrenCount;
    final View[] children = mChildren;

    // 遍历所有子view
    for (int i = 0; i < size; ++i) {
    	final View child = children[i];
        // 调用measureChild()进行下一步的测量 ->>分析1
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
        	measureChild(child, widthMeasureSpec, heightMeasureSpec);
		}
	}
}
```

* 遍历所有的子view，然后调用measureChild方法

##### 5.6.7 ViewGroup#measureChild

```java
/**
  * 分析2：measureChild()
  * 作用：a. 计算单个子View的MeasureSpec
  *      b. 测量每个子View最后的宽 / 高：调用子View的measure()
  **/ 
protected void measureChild(View child, int parentWidthMeasureSpec,
		int parentHeightMeasureSpec) {
	// 1. 获取子视图的布局参数
	final LayoutParams lp = child.getLayoutParams();

    // 2. 根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
    // getChildMeasureSpec() 请看上面第2节储备知识处
    // 获取 ChildView 的 widthMeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
    		mPaddingLeft + mPaddingRight, lp.width);
    // 获取 ChildView 的 heightMeasureSpec
	final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
    		mPaddingTop + mPaddingBottom, lp.height);

	// 3. 将计算好的子View的MeasureSpec值传入measure()，进行最后的测量
    // 下面的流程即类似单一View的过程，此处不作过多描述
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

* 至此，`ViewGroup`的`measure`过程分析完毕

#### 5.7 小结

![img](https:////upload-images.jianshu.io/upload_images/944365-c9ea47e8b5e325bf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

- 为了让大家更好地理解`ViewGroup`的`measure`过程（特别是复写`onMeasure()`），下面，我将用`ViewGroup`的子类`LinearLayout`来分析下`ViewGroup`的`measure`过程

### 六、重写onMeasure实例之LinearLayout

#### 6.1 LinearLayout#onMeasure

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	if (mOrientation == VERTICAL) {
    	measureVertical(widthMeasureSpec, heightMeasureSpec);
	} else {
    	measureHorizontal(widthMeasureSpec, heightMeasureSpec);
	}
}
```

* 对于两种不同的布局方式，调用不同的方法，我们这里就研究measureVertical方法把，垂直排序

#### 6.2 LinearLayout#measureVertical

```java
/**
* 分析1：measureVertical()
* 作用：测量LinearLayout垂直方向的测量尺寸
**/ 
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	......//其他测量逻辑

	// 获取垂直方向上的子View个数
    final int count = getVirtualChildCount();
	......
    // 遍历子View获取其高度，并记录下子View中最高的高度数值
	for (int i = 0; i < count; ++i) {
    	final View child = getVirtualChildAt(i);
		......
        // 子View不可见，直接跳过该View的measure过程，getChildrenSkipCount()返回值恒为0
        // 注：若view的可见属性设置为VIEW.INVISIBLE，还是会计算该view大小
        if (child.getVisibility() == View.GONE) {
        	i += getChildrenSkipCount(child, i);
            continue;
		}

        // 记录子View是否有weight属性设置，用于后面判断是否需要二次measure
        totalWeight += lp.weight;

        if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
        	// 如果LinearLayout的specMode为EXACTLY且子View设置了weight属性，在这里会跳过子View的measure过程
            // 同时标记skippedMeasure属性为true，后面会根据该属性决定是否进行第二次measure
            // 若LinearLayout的子View设置了weight，会进行两次measure计算，比较耗时
            // 这就是为什么LinearLayout的子View需要使用weight属性时候，最好替换成RelativeLayout布局
                
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            skippedMeasure = true;
		} else {
        	int oldHeight = Integer.MIN_VALUE;
			
            //步骤1 遍历调用measureChildren
            //注：该方法内部，最终会调用measureChildren（），从而 遍历所有子View & 测量
            measureChildBeforeLayout(
                   child, i, widthMeasureSpec, 0, heightMeasureSpec,
                   totalWeight == 0 ? mTotalLength : 0);
                   ...
		}

		//步骤2：合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）     
		final int childHeight = child.getMeasuredHeight();

		// 1. mTotalLength用于存储LinearLayout在竖直方向的高度
        final int totalLength = mTotalLength;

        // 2. 每测量一个子View的高度， mTotalLength就会增加
        mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                     lp.bottomMargin + getNextLocationOffset(child));
      
        // 3. 记录LinearLayout占用的总高度
        // 即除了子View的高度，还有本身的padding属性值
        mTotalLength += mPaddingTop + mPaddingBottom;
        int heightSize = mTotalLength;
        ......
    }
	......
	//步骤3：存储测量后View宽/高的值：调用setMeasuredDimension()  
	setMeasuredDimension(resolveSizeAndState(maxWidth,width))
    ...
}
```

* 至此，自定义`View`的中最重要、最复杂的`measure`过程讲解完毕。

### 七、总结

#### 7.1 两种measure过程

- 本文对自定义View中最重要、最复杂的`measure`过程进行了详细分析，具体如下图：

![img](https://upload-images.jianshu.io/upload_images/944365-29a36501ce27a5cb.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

#### 7.2 单一View的measure过程和ViewGroup的measure过程

![img](https://upload-images.jianshu.io/upload_images/944365-1250b5f61c90147f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



