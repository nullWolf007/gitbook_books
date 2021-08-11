[TOC]

## ViewStub

#### 转载

* [ViewStub的实现深入解析](https://www.jianshu.com/p/16ebb86c6c23)

### 一、概述

* 布局优化是性能优化中一项不可缺失的工作，而`ViewStub`是性能布局优化中很有必要的一项，使用`ViewStub`可以把类似空白页、错误页等不需要马上显示的View实现懒加载的效果，而且内存占有量非常的少，它是一个宽高为0、不执行draw方法且本身设置了`View.GONE`所以基本上不参与layout，非常适合用于做懒加载的布局优化。

```java
public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
	// 在构造函数中就设置成了GONE
    setVisibility(GONE);
}
    
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	setMeasuredDimension(0, 0);
}

@Override
public void draw(Canvas canvas) {
}

@Override
protected void dispatchDraw(Canvas canvas) {
}
```

* 是吧，这些常见的方法都是空实现的且在初始化中就设置成GONE了、所以这就很好奇是怎么实现懒加载的。

### 二、ViewStub简单用法

* **当需要的时候调用inflater()或者是setVisible()方法显示这些View组件。**

* 先看下如何使用ViewStub：

```objectivec
<ViewStub
	android:id="@+id/id_vs"
    android:layout="@layout/layout_test"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />

ViewStub mViewStub = (ViewStub) findViewById(R.id.id_vs);
View mRootView = mViewStub.inflate();
// 接下来就可以各种的findViewById了...
mRootView.findViewById(R.id.xxx);
```

* 在布局的时候我们需要给`android:layout`传入一个我们定义好的布局文件，这个文件只有在调用了`mViewStub.inflate()`才会被显示出来。对于传入的的布局文件会赋值给`ViewStub`的成员变量`mLayoutResource`，Ok，一切的准备就绪了。

### 三、ViewStub实现的原理

* 原理其实不难，只要看下下面这个方法就一目了然了。

```dart
public View inflate() {
    final ViewParent viewParent = getParent();
    // 首先要求父控件是ViewGroup才可以
    if (viewParent != null && viewParent instanceof ViewGroup) {
        // 其次要给mLayoutResource赋值，因为mLayoutResource就是要懒加载显示的界面对应的布局
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
            final LayoutInflater factory;
            if (mInflater != null) {
                factory = mInflater;
            } else {
                factory = LayoutInflater.from(mContext);
            }
            // 这就是重点了，直接调用常见的LayoutInflater.from().inflate系列方法来初始化需要懒加载的View
            final View view = factory.inflate(mLayoutResource, parent,
                false);

            if (mInflatedId != NO_ID) {
                view.setId(mInflatedId);
            }

            final int index = parent.indexOfChild(this);
            parent.removeViewInLayout(this);

            final ViewGroup.LayoutParams layoutParams = getLayoutParams();
            if (layoutParams != null) {
                parent.addView(view, index, layoutParams);
            } else {
                parent.addView(view, index);
            }

            mInflatedViewRef = new WeakReference<View>(view);

            if (mInflateListener != null) {
                mInflateListener.onInflate(this, view);
            }
            // 通过返回的这个View  我们就可以拿来各种findViewById 就能显示需要显示的View了
            return view;
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

* ViewStub这个类很短去掉注释估计也就一百行的代码，原理其实就是把对应的布局文件当做值传入给mLayoutResource变量，当调用`inflate()`的时候调用`LayoutInflater.from(xx).inflate(xx)`方法把对应懒加载的View初始化出来，没错、这样子之后就能显示了。

### 四、最后有一个疑问

* `android:visibility="gone"` 和 `ViewStub` 之间的区别，为什么GONE就没有ViewStub的功效呢，因为同样在界面上是不显示的。

```css
setContentView(R.layout.activity_main);
```

* 这一切还得从这个方法说起，都知道这个方法最后是会跳转到`LayoutInflater` 这个类中去然后同样执行`inflate`方法。

```kotlin
final XmlResourceParser parser = res.getLayout(resource);
try {
	return inflate(parser, root, attachToRoot);
} finally {
	parser.close();
}
```

* 在inflate中有这么一段方法，会根据传入的布局资源然后调用`XmlResourceParser`实现xml文件解析从而得到一个个的View，而这时候如果你把不需要马上显示的View设置成GONE一样会被解析一样会被加载到内存中去，当然你放到ViewStub中去那么就只会加载ViewStub并不会把相对应的View也加载进去，所以从而可以起到懒加载的效果。

* 所以、ViewStub用起来..
