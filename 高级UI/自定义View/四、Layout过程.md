[TOC]

## 四、Layout过程

#### 转载

* [（3）自定义View Layout过程 - 最易懂的自定义View原理系列](https://www.jianshu.com/p/158736a2549d)

### 一、概述

#### 1.1 Layout作用

* 计算视图`（View）`的位置
* 计算`View`的四个顶点位置：`Left`、`Top`、`Right` 和 `Bottom`

#### 1.2 Layout流程的分类

* 类似`measure`过程，`layout`过程根据**View的类型**分为2种情况：

![img](https://upload-images.jianshu.io/upload_images/944365-6e978f448667eb52.png?imageMogr2/auto-orient/strip|imageView2/2/w/590/format/webp)

* 接下来，我将详细分析这2种情况下的`layout`过程

### 二、单一View的Layout

#### 2.1 应用场景

*  在无现成的控件`View`满足需求、需自己实现时，则使用自定义单一`View`
  * 制作一个支持加载网络图片的`ImageView`控件
  * 注：自定义`View`在多数情况下都有替代方案：图片 / 组合动画，但二者可能会导致内存耗费过大，从而引起内存溢出等问题。

#### 2.2 具体使用 

-  继承自`View`、`SurfaceView` 或 其他`View`；不包含子`View`

#### 2.3 具体流程

![img](https:////upload-images.jianshu.io/upload_images/944365-05b688ab79b57ecf.png?imageMogr2/auto-orient/strip|imageView2/2/w/360/format/webp)

* 下面我将一个个方法进行详细分析

#### 2.4 源码分析

##### 2.4.1 View#layout

```java
/**
* 源码分析：layout（）
* 作用：确定View本身的位置，即设置View本身的四个顶点位置
*/ 
public void layout(int l, int t, int r, int b) {  
	......
    // 当前视图的四个顶点
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
      
    // 1. 确定View的位置：setFrame（） / setOpticalFrame（）
    // 即初始化四个顶点的值、判断当前View大小和位置是否发生了变化 & 返回 
    // ->>分析1、分析2
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    // 2. 若视图的大小 & 位置发生变化
    // 会重新确定该View所有的子View在父容器的位置：onLayout（）
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  

        // 对于单一View的laytou过程：由于单一View是没有子View的，故onLayout（）是一个空实现->>分析3
        // 对于ViewGroup的laytou过程：由于确定位置与具体布局有关，所以onLayout（）在ViewGroup为1个抽象方法，需重写实现（后面会详细说）
        onLayout(changed, l, t, r, b);  
        ......
    }   
	......
}  
```

* 调用setOpticalFrame或者setFrame判断当前的View大小和位置是否发生变化了
* 若视图的大小 & 位置发生变化，会调用onLayout方法，重新确定该View所有的子View在父容器的位置
* 由于单一View是没有子View的，所以onLayout方法是一个空实现

##### 2.4.2 View#setFrame

```java
/**
  * 分析1：setFrame（）
  * 作用：根据传入的4个位置值，设置View本身的四个顶点位置
  * 即：最终确定View本身的位置
*/ 
protected boolean setFrame(int left, int top, int right, int bottom) {
	......
    // 通过以下赋值语句记录下了视图的位置信息，即确定View的四个顶点
    // 从而确定了视图的位置
    mLeft = left;
    mTop = top;
    mRight = right;
    mBottom = bottom;
    mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
    ......
}
```

##### 2.4.3 View#setOpticalFrame

```java
/**
  * 分析2：setOpticalFrame（）
  * 作用：根据传入的4个位置值，设置View本身的四个顶点位置
  * 即：最终确定View本身的位置
  */ 
private boolean setOpticalFrame(int left, int top, int right, int bottom) {
	Insets parentInsets = mParent instanceof View ?
    					((View) mParent).getOpticalInsets() : Insets.NONE;

	Insets childInsets = getOpticalInsets();

    // 内部实际上是调用setFrame（）
    return setFrame(
    			left   + parentInsets.left - childInsets.left,
                top    + parentInsets.top  - childInsets.top,
                right  + parentInsets.left + childInsets.right,
                bottom + parentInsets.top  + childInsets.bottom);
}
```

* 内部实际上调用的也是setFrame方法

##### 2.4.4 View#onLayout

```java
/**
  * 分析3：onLayout（）
  * 注：对于单一View的laytou过程
  *    a. 由于单一View是没有子View的，故onLayout（）是一个空实现
  *    b. 由于在layout（）中已经对自身View进行了位置计算，所以单一View的layout过程在layout（）后就已完成了
  */ 
 protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

   // 参数说明
   // changed 当前View的大小和位置改变了 
   // left 左部位置
   // top 顶部位置
   // right 右部位置
   // bottom 底部位置
}  
```

##### 2.4.5 小结

* 单一`View`的`layout`过程解析如下：

![img](https://upload-images.jianshu.io/upload_images/944365-756f72f8ccc58d2c.png?imageMogr2/auto-orient/strip|imageView2/2/w/310/format/webp)

### 三、ViewGroup的Layout

#### 3.1 应用场景

- 利用现有的组件根据特定的布局方式来组成新的组件

#### 3.2 具体使用

- 继承自`ViewGroup` 或 各种`Layout`；含有子 `View`

#### 3.3 实例

* 如：底部导航条中的条目，一般都是上图标(ImageView)、下文字(TextView)，那么这两个就可以用自定义ViewGroup组合成为一个Veiw，提供两个属性分别用来设置文字和图片，使用起来会更加方便。

![img](https:////upload-images.jianshu.io/upload_images/944365-3eb8d5d9d7d9e9fd.png?imageMogr2/auto-orient/strip|imageView2/2/w/504/format/webp)

#### 3.4 原理（步骤）

- 计算自身`ViewGroup`的位置：`layout（）`
- 遍历子`View` & 确定自身子View在`ViewGroup`的位置（调用子`View` 的 `layout（）`->`onLayout（）`)
  * 步骤2 类似于 单一`View`的`layout`过程
  * 自上而下、一层层地传递下去，直到完成整个`View`树的`layout（）`过程

![img](https:////upload-images.jianshu.io/upload_images/944365-7133935cb1e56190.png?imageMogr2/auto-orient/strip|imageView2/2/w/661/format/webp)

#### 3.5 流程

![img](https:////upload-images.jianshu.io/upload_images/944365-7ebd03609c758d47.png?imageMogr2/auto-orient/strip|imageView2/2/w/498/format/webp)



* ` ViewGroup` 和 `View` 同样拥有`layout（）`和`onLayout()`，但二者不同的：

  * 一开始计算`ViewGroup`位置时，调用的是`ViewGroup`的`layout（）`和`onLayout()`；

  * 当开始遍历子`View` & 计算子`View`位置时，调用的是子`View`的`layout（）`和`onLayout()`

- 下面我将一个个方法进行详细分析：`layout`过程入口为`ViewGroup#layout()`

#### 3.6 源码解析

##### 3.6.1 ViewGroup#layout

```java
@Override
public final void layout(int l, int t, int r, int b) {
	if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
    	if (mTransition != null) {
        	mTransition.layoutChange(this);
		}
        super.layout(l, t, r, b);
	} else {
    	// record the fact that we noop'd it; request layout when transition finishes
        mLayoutCalledWhileSuppressed = true;
	}
}
```

* 最终会调用View#layout

##### 3.6.2 View#layout

* 接下来的代码和上面是一样的，就不具体说明了
* 最终会调用onLayout方法

##### 3.6.3 onLayout

* 在ViewGroup中，onLayout是抽象方法，所以必须重写

```java
@Override
protected abstract void onLayout(boolean changed,
		int l, int t, int r, int b);
```

* 其中onLayout方法是重写的View的onLayout方法，所以View的onLayout方法最终实现是我们重写的。
* 由于ViewGroup中把onLayout设置为抽象的了，所以View的onLayout方法最终会调用实现类的onLayout方法
* 重写步骤
  * 遍历子View
  * 计算当前子View的四个位置值 & 确定自身子View的位置，可以调用子View的layout方法

```java

/**
  * 分析3：onLayout（）
  * 作用：计算该ViewGroup包含所有的子View在父容器的位置（）
  */ 
// 参数说明
// changed 当前View的大小和位置改变了 
// left 左部位置
// top 顶部位置
// right 右部位置
// bottom 底部位置
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	// 1. 遍历子View：循环所有子View
    for (int i=0; i<getChildCount(); i++) {
    	View child = getChildAt(i);   
		// 2. 计算当前子View的四个位置值
        // 2.1 位置的计算逻辑
        ...// 需自己实现，也是自定义View的关键

        // 2.2 对计算后的位置值进行赋值
        int mLeft  = Left
        int mTop  = Top
        int mRight = Right
        int mBottom = Bottom

        // 3. 根据上述4个位置的计算值，设置子View的4个顶点：调用子view的layout() & 传递计算过的参数
        // 即确定了子View在父容器的位置
        child.layout(mLeft, mTop, mRight, mBottom);
        // 该过程类似于单一View的layout过程中的layout（）和onLayout（），此处不作过多描述
	}
}
```

##### 3.6.4 总结

![img](https:////upload-images.jianshu.io/upload_images/944365-6e27d40b50081d60.png?imageMogr2/auto-orient/strip|imageView2/2/w/647/format/webp)

### 四、实例1（LinearLayout）

* 为了更好理解`ViewGroup`的`layout`过程（特别是复写`onLayout（）`），我们通过实例去加深

#### 4.1 原理

* 计算出`LinearLayout`本身在父布局的位置
* 计算出`LinearLayout`中所有子`View`在容器中的位置

#### 4.2 具体流程

![img](https://upload-images.jianshu.io/upload_images/944365-7537ab3496115ac7.png?imageMogr2/auto-orient/strip|imageView2/2/w/675/format/webp)

#### 4.3 源码分析

* 由上面已经知道了，我们需要关注的是重写的`onLayout`方法

##### 4.3.1 LinearLayou#onLayout

```java
/**
  * 源码分析：LinearLayout复写的onLayout（）
  * 注：复写的逻辑 和 LinearLayout measure过程的 onMeasure()类似
*/ 
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
	// 根据自身方向属性，而选择不同的处理方式
    if (mOrientation == VERTICAL) {
    	layoutVertical(l, t, r, b);
	} else {
    	layoutHorizontal(l, t, r, b);
	}
}
```

* 会对垂直还是水平进行判断，调用两个不同的方法，但是是类似的，我们这里以垂直方向的举例说明，layoutVertical

##### 4.3.2 LinearLayou#layoutVertical

* 主要代码如下所示

```java
/**
* 分析1：layoutVertical(l, t, r, b)
*/
void layoutVertical(int left, int top, int right, int bottom) {
       
	// 子View的数量
    final int count = getVirtualChildCount();

    // 1. 遍历子View
    for (int i = 0; i < count; i++) {
    	final View child = getVirtualChildAt(i);
        if (child == null) {
        	childTop += measureNullChild(i);
		} else if (child.getVisibility() != GONE) {
			// 2. 计算子View的测量宽 / 高值
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();

            // 3. 确定自身子View的位置
            // 即：递归调用子View的setChildFrame()，实际上是调用了子View的layout() ->>分析2
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);

            // childTop逐渐增大，即后面的子元素会被放置在靠下的位置
            // 这符合垂直方向的LinearLayout的特性
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
		}
	}
}
```

* 主要就是遍历子View，对其每个子View进行layout操作，计算出对应的位置，然后使用childTop去垂直排序，每次遍历加上子View的高度，其他的子View都要在此值的下面位置。
* 调用setChildFrame去进行layout操作的

##### 4.3.3 setChildFrame

```java
/**
  * 分析2：setChildFrame()
*/
private void setChildFrame( View child, int left, int top, int width, int height){
        
	// setChildFrame（）仅仅只是调用了子View的layout（）而已
    child.layout(left, top, left ++ width, top + height);
}
// 在子View的layout（）又通过调用setFrame（）确定View的四个顶点，即确定了子View的位置
// 如此不断循环确定所有子View的位置，最终确定ViewGroup的位置
```

### 五、常见问题

#### 5.1 getWidth() （ getHeight()）与 getMeasuredWidth() （getMeasuredHeight()）获取的宽 （高）有什么区别？

- `getWidth()` / `getHeight()`：获得`View`最终的宽 / 高
- `getMeasuredWidth()` / `getMeasuredHeight()`：获得 `View`测量的宽 / 高

* 首先，从上文中我们得知，`getWidth()`和`getHeight()`函数的相关信息实际上是在`setFrame()`函数执行完毕才准备完毕的——我们大致可以认为是这两个函数 **只有布局流程(layout)执行完毕才能调用**，而在父控件的`onLayout()`函数中，获取子控件宽度和高度时，子控件还并未开始进行布局流程，因此此时不能调用`getWidth()`函数，而只能通过`getMeasuredWidth()`函数获取控件测量阶段结果的宽度。
* 那么当控件绘制流程执行完毕后，`getWidth()`和`getMeasuredWidth()`函数的值有什么区别呢？从上述`setChildFrame()`函数中的源码可以得知，布局流程执行后，`getWidth()`返回值的本质其实就是`getMeasuredWidth()`——因此本质上，当我们没有手动调用`layout()`函数强制修改控件的布局信息的话，两个函数的返回值大小是完全一致的。

### 六、总结

#### 6.2 两种情况

![img](https://upload-images.jianshu.io/upload_images/944365-bb11305f1e40a8fb.png?imageMogr2/auto-orient/strip|imageView2/2/w/560/format/webp)

#### 6.2 流程图

![img](https://upload-images.jianshu.io/upload_images/944365-6baebb31c56040dc.png?imageMogr2/auto-orient/strip|imageView2/2/w/970/format/webp)

