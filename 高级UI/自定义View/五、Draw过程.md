## 五、Draw过程

#### 转载

* [（4）自定义View Draw过程- 最易懂的自定义View原理系列](https://www.jianshu.com/p/95afeb7c8335)

### 一、概述

#### 1.1 作用

* 绘制`View`视图

#### 1.2 分类

* 类似`measure`过程、`layout`过程，`draw`过程根据**View的类型**分为2种情况：

![img](https://upload-images.jianshu.io/upload_images/944365-2dc3a798a3039bb5.png?imageMogr2/auto-orient/strip|imageView2/2/w/410/format/webp)

* 接下来，我将详细分析这2种情况下的`draw`过程

### 二、单一View的Draw

#### 2.1 应用场景

- 在无现成的控件`View`满足需求、需自己实现时，则使用自定义单一`View`
  * 如：制作一个支持加载网络图片的`ImageView`控件
  * 注：自定义`View`在多数情况下都有替代方案：图片 / 组合动画，但二者可能会导致内存耗费过大，从而引起内存溢出等问题。

#### 2.2 具体使用 

*  继承自`View`、`SurfaceView` 或 其他`View`；不包含子`View`

#### 2.3 原理（步骤）

* `View`绘制自身（含背景、内容）；

* 绘制装饰（滚动指示器、滚动条、和前景）

#### 2.4 具体流程

![img](https://upload-images.jianshu.io/upload_images/944365-1554a862ed0c3f95.png?imageMogr2/auto-orient/strip|imageView2/2/w/330/format/webp)

#### 2.5 源码解析

* `View#draw`过程的入口 = `View#draw()`

##### 2.5.1 View#draw

* 主要代码如下

```java
/**
  * 源码分析：draw（）
  * 作用：根据给定的 Canvas 自动渲染 View（包括其所有子 View）。
  * 绘制过程：
  *   1. 绘制view背景
  *   2. 绘制view内容
  *   3. 绘制子View
  *   4. 绘制装饰（渐变框，滑动条等等）
  * 注：
  *    a. 在调用该方法之前必须要完成 layout 过程
  *    b. 所有的视图最终都是调用 View 的 draw （）绘制视图（ ViewGroup 没有复写此方法）
  *    c. 在自定义View时，不应该复写该方法，而是复写 onDraw(Canvas) 方法进行绘制
  *    d. 若自定义的视图确实要复写该方法，那么需先调用 super.draw(canvas)完成系统的绘制，然后再进行自定义的绘制
  */ 
public void draw(Canvas canvas) {  
	int saveCount;

    // 步骤1： 绘制本身View背景
    if (!dirtyOpaque) {
    	drawBackground(canvas);
	}

    // 若有必要，则保存图层（还有一个复原图层）
    // 优化技巧：当不需绘制 Layer 时，“保存图层“和“复原图层“这两步会跳过
    // 因此在绘制时，节省 layer 可以提高绘制效率
    final int viewFlags = mViewFlags;
    if (!verticalEdges && !horizontalEdges) {

    // 步骤2：绘制本身View内容
    if (!dirtyOpaque) 
    	onDraw(canvas);
	// View 中：默认为空实现，需复写
    // ViewGroup中：需复写

    // 步骤3：绘制子View
    // 由于单一View无子View，故View 中：默认为空实现
    // ViewGroup中：系统已经复写好对其子视图进行绘制我们不需要复写
    dispatchDraw(canvas);
        
    // 步骤4：绘制装饰，如滑动条、前景色等等
    onDrawScrollBars(canvas);

    return;
}
```

* 上述主要调用了4个方法`drawBackground（）`、 `onDraw()`、`dispatchDraw()`、`onDrawScrollBars()`，接下来会一一讲解

##### 2.5.2 View#drawBackground

```java
/**
  * 步骤1：drawBackground(canvas)
  * 作用：绘制View本身的背景
*/
private void drawBackground(Canvas canvas) {
	// 获取背景 drawable
    final Drawable background = mBackground;
    if (background == null) {
    	return;
	}
    // 根据在 layout 过程中获取的 View 的位置参数，来设置背景的边界
    setBackgroundBounds();

    .....

    // 获取 mScrollX 和 mScrollY值 
    final int scrollX = mScrollX;
    final int scrollY = mScrollY;
    if ((scrollX | scrollY) == 0) {
    	background.draw(canvas);
	} else {
    	// 若 mScrollX 和 mScrollY 有值，则对 canvas 的坐标进行偏移
        canvas.translate(scrollX, scrollY);

		// 调用 Drawable 的 draw 方法绘制背景
        background.draw(canvas);
        canvas.translate(-scrollX, -scrollY);
	}
} 
```

* 主要就是绘制View本身的背景

##### 2.5.3 onDraw

```java

/**
  * 步骤2：onDraw(canvas)
  * 作用：绘制View本身的内容
  * 注：
  *   a. 由于 View 的内容各不相同，所以该方法是一个空实现
  *   b. 在自定义绘制过程中，需由子类去实现复写该方法，从而绘制自身的内容
  *   c. 谨记：自定义View中 必须 且 只需复写onDraw（）
  */
protected void onDraw(Canvas canvas) {
	... // 复写从而实现绘制逻辑
}
```

##### 2.5.4 dispatchDraw

```java

/**
  * 步骤3： dispatchDraw(canvas)
  * 作用：绘制子View
  * 注：由于单一View中无子View，故为空实现
*/
protected void dispatchDraw(Canvas canvas) {
	... // 空实现
}
```

##### 2.5.5 onDrawScrollBars

```java
/**
* 步骤4： onDrawScrollBars(canvas)
* 作用：绘制装饰，如 滚动指示器、滚动条、和前景等
*/
public void onDrawForeground(Canvas canvas) {
	onDrawScrollIndicators(canvas);
    onDrawScrollBars(canvas);

    final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
    if (foreground != null) {
    	if (mForegroundInfo.mBoundsChanged) {
        	mForegroundInfo.mBoundsChanged = false;
            final Rect selfBounds = mForegroundInfo.mSelfBounds;
            final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

            if (mForegroundInfo.mInsidePadding) {
            	selfBounds.set(0, 0, getWidth(), getHeight());
			} else {
            	selfBounds.set(getPaddingLeft(), getPaddingTop(),
                            getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
			}

            final int ld = getLayoutDirection();
            Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                        foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
            foreground.setBounds(overlayBounds);
		}

        foreground.draw(canvas);
	}
}
```

* 至此，单一`View`的`draw`过程已分析完毕。

##### 2.5.6 总结

![img](https://upload-images.jianshu.io/upload_images/944365-b63b64086782a217.png?imageMogr2/auto-orient/strip|imageView2/2/w/330/format/webp)

### 三、View Group的Draw

#### 3.1 应用场景

- 利用现有的组件根据特定的布局方式来组成新的组件

#### 3.2 具体使用

- 继承自`ViewGroup` 或 各种`Layout`；含有子 `View`

#### 3.3 实例

* 如：底部导航条中的条目，一般都是上图标(ImageView)、下文字(TextView)，那么这两个就可以用自定义ViewGroup组合成为一个Veiw，提供两个属性分别用来设置文字和图片，使用起来会更加方便。

![img](https:////upload-images.jianshu.io/upload_images/944365-3eb8d5d9d7d9e9fd.png?imageMogr2/auto-orient/strip|imageView2/2/w/504/format/webp)

#### 3.4 原理（步骤）

* `ViewGroup`绘制自身（含背景、内容）；
* `ViewGroup`遍历子`View` & 绘制其所有子View；
  * 类似于单一`View`的`draw`过程
* `ViewGroup`绘制装饰（滚动指示器、滚动条、和前景）

#### 3.5 流程

![img](https://upload-images.jianshu.io/upload_images/944365-ff799e17e24e4a3b.png?imageMogr2/auto-orient/strip|imageView2/2/w/424/format/webp)

#### 3.6 源码分析

* `ViewGroup#draw`过程的入口 = `ViewGroup#draw（）`

##### 3.6.1 ViewGroup#draw

* 由于ViewGroup没有重写draw方法，所以ViewGroup.draw调用的是父类的方法View#draw

##### 3.6.2 View#draw

```java
/**
  * 源码分析：draw（）
  * 作用：根据给定的 Canvas 自动渲染 View（包括其所有子 View）。
  * 绘制过程：
  *   1. 绘制view背景
  *   2. 绘制view内容
  *   3. 绘制子View
  *   4. 绘制装饰（渐变框，滑动条等等）
  * 注：
  *    a. 在调用该方法之前必须要完成 layout 过程
  *    b. 所有的视图最终都是调用 View 的 draw （）绘制视图（ ViewGroup 没有复写此方法）
  *    c. 在自定义View时，不应该复写该方法，而是复写 onDraw(Canvas) 方法进行绘制
  *    d. 若自定义的视图确实要复写该方法，那么需先调用 super.draw(canvas)完成系统的绘制，然后再进行自定义的绘制
  */ 
public void draw(Canvas canvas) {  
	int saveCount;

    // 步骤1： 绘制本身View背景
    if (!dirtyOpaque) {
    	drawBackground(canvas);
	}

    // 若有必要，则保存图层（还有一个复原图层）
    // 优化技巧：当不需绘制 Layer 时，“保存图层“和“复原图层“这两步会跳过
    // 因此在绘制时，节省 layer 可以提高绘制效率
    final int viewFlags = mViewFlags;
    if (!verticalEdges && !horizontalEdges) {

    // 步骤2：绘制本身View内容
    if (!dirtyOpaque) 
    	onDraw(canvas);
	// View 中：默认为空实现，需复写
    // ViewGroup中：需复写

    // 步骤3：绘制子View
    // 由于单一View无子View，故View 中：默认为空实现
    // ViewGroup中：系统已经复写好对其子视图进行绘制我们不需要复写
    dispatchDraw(canvas);
        
    // 步骤4：绘制装饰，如滑动条、前景色等等
    onDrawScrollBars(canvas);

    return;
}
```

* 由于drawBackground和onDraw和onDrawScrollBars方法于单一View类似，这里就不再说明了
* 主要的不同是dispatchDraw方法，这里坐一下说明

##### 3.6.3 ViewGroup#dispatchDraw

* 主要代码如下所示

```java
/**
  * 源码分析：dispatchDraw（）
  * 作用：遍历子View & 绘制子View
  * 注：
  *   a. ViewGroup中：由于系统为我们实现了该方法，故不需重写该方法
  *   b. View中默认为空实现（因为没有子View可以去绘制）
*/ 
protected void dispatchDraw(Canvas canvas) {
	......
	// 1. 遍历子View
    final int childrenCount = mChildrenCount;
    ......

    for (int i = 0; i < childrenCount; i++) {
    	......
        if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
        		transientChild.getAnimation() != null) {
			// 2. 绘制子View视图 ->>分析1
            more |= drawChild(canvas, transientChild, drawingTime);
		}
                ....
	}
}
```

* 遍历子View，然后对单个的View调用drawChild方法进行绘制

##### 3.6.4 ViewGroup#drawChild

```java
/**
  * 分析1：drawChild（）
  * 作用：绘制子View
*/
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
	// 最终还是调用了子 View 的 draw （）进行子View的绘制
    return child.draw(canvas, this, drawingTime);
}
```

* 最终调用View的draw方法进行绘制，这就是单个View的绘制流程

* 至此，`ViewGroup`的`draw`过程已分析完毕。

##### 3.6.5 总结

`ViewGroup`的`draw`过程如下：

![img](https://upload-images.jianshu.io/upload_images/944365-cf42edcab0a206fa.png?imageMogr2/auto-orient/strip|imageView2/2/w/985/format/webp)

### 四、常见问题

#### 4.1 View.setWillNotDraw()

```java
/**
  * 源码分析：setWillNotDraw()
  * 定义：View 中的特殊方法
  * 作用：设置 WILL_NOT_DRAW 标记位；
  * 注：
  *   a. 该标记位的作用是：当一个View不需要绘制内容时，系统进行相应优化
  *   b. 默认情况下：View 不启用该标记位（设置为false）；ViewGroup 默认启用（设置为true）
  */ 

public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}

// 应用场景
// a. setWillNotDraw参数设置为true：当自定义View继承自 ViewGroup 、且本身并不具备任何绘制时，设置为 true 后，系统会进行相应的优化。
// b. setWillNotDraw参数设置为false：当自定义View继承自 ViewGroup 、且需要绘制内容时，那么设置为 false，来关闭 WILL_NOT_DRAW 这个标记位。
```

### 五、总结

#### 2.1 两种情况

![img](https://upload-images.jianshu.io/upload_images/944365-2c28edbbb3f1936f.png?imageMogr2/auto-orient/strip|imageView2/2/w/500/format/webp)

#### 2.2 流程

![img](https://upload-images.jianshu.io/upload_images/944365-c9d3cd1d746be319.png?imageMogr2/auto-orient/strip|imageView2/2/w/1010/format/webp)