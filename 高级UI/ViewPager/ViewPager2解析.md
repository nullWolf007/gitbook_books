[TOC]

## ViewPager2解析

#### 转载

* [ViewPager2重大更新，支持offscreenPageLimit](https://juejin.cn/post/6844903843658989581)

### 一、基础知识

#### 1.1 ViewPager顽疾

* 顽疾是什么鬼，没有这么严重吧。`ViewPager`有两个毛病：`不能关闭预加载`和`更新Adapter不生效`，所以开头我为什么说`offscreenPageLimit`在`ViewPager`上十分不友好；本质上是因为`offscreenPageLimit`不能设置成0(设置成0就是想象中的关闭预加载)；

![img](https://user-gold-cdn.xitu.io/2019/5/14/16ab4b7775ef7b93?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 上面是ViewPager默认情况下的加载示意图，当切换到当前页面时，会默认预加载左右两侧的布局到`ViewPager`中，尽管两侧的View并不可见的，我们称这种情况叫`预加载`；由于`ViewPager`对`offscreenPageLimit`设置了限制，页面的预加载是不可避免；

* **ViewPager#setOffscreenPageLimit**

```java
private static final int DEFAULT_OFFSCREEN_PAGES = 1;

public void setOffscreenPageLimit(int limit) {
    if (limit < DEFAULT_OFFSCREEN_PAGES) {//不允许小于1
        Log.w(TAG, "Requested offscreen page limit " + limit + " too small; defaulting to "
                + DEFAULT_OFFSCREEN_PAGES);
        limit = DEFAULT_OFFSCREEN_PAGES;
    }
    if (limit != mOffscreenPageLimit) {
        mOffscreenPageLimit = limit;
        populate();
    }
}
```

* 由于limit默认设置最小值为1，即使传入的参数小于1，也会赋值为1。
* ViewPager强制预加载的逻辑在`Fragment`配合`ViewPager`使用时依然存在

### 二、ViewPager2的基本使用

#### 2.1 常用方法

- `setAdapter()` 设置适配器
- `setOrientation()` 设置布局方向
- `setCurrentItem()` 设置当前Item下标
- `beginFakeDrag()` 开始模拟拖拽
- `fakeDragBy()` 模拟拖拽中
- `endFakeDrag()` 模拟拖拽结束
- `setUserInputEnabled()` 设置是否允许用户输入/触摸
- `setOffscreenPageLimit()`设置屏幕外加载页面数量
- `registerOnPageChangeCallback()` 注册页面改变回调
- `setPageTransformer()` 设置页面滑动时的变换效果

#### 2.2 实例

##### 2.2.1 build.gradle引入

```groovy
implementation "androidx.viewpager2:viewpager2:1.0.0"
```

##### 2.2.2 布局文件添加

* activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewpager2"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

##### 2.2.3 第一种：类似RecyclerView的形式

* item_card_layout.xml

```java
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/label_center"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:textColor="@color/colorAccent"
        android:textSize="24sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

* MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ViewPager2 viewPager2 = findViewById(R.id.viewpager2);
        viewPager2.setAdapter(new RecyclerView.Adapter<ViewHolder>() {
            @NonNull
            @Override
            public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, 
                                                 int viewType) {
                View itemView = LayoutInflater.from(parent.getContext()).
                    inflate(R.layout.item_card_layout, parent, false);
                ViewHolder viewHolder = new ViewHolder(itemView);
                return viewHolder;
            }

            @Override
            public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
                holder.labelCenter.setText(String.valueOf(position));
            }

            @Override
            public int getItemCount() {
                return 10;
            }
        });
    }

    static class ViewHolder extends RecyclerView.ViewHolder {
        private final TextView labelCenter;

        public ViewHolder(@NonNull View itemView) {
            super(itemView);
            labelCenter = itemView.findViewById(R.id.label_center);
        }
    }
}

```

##### 2.2.4 第二种：Fragment+ViewPager

* 创建需要的Fragment，如TestFragment/Test2Fragment/Test3Fragment

* MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ViewPager2 viewPager2 = findViewById(R.id.viewpager2);
        viewPager2.setAdapter(new FragmentStateAdapter(this) {
            @NonNull
            @Override
            public Fragment createFragment(int position) {
                switch (position){
                    case 0:
                        return new TestFragment();
                    case 1:
                        return new Test2Fragment();
                    default:
                        return new Test3Fragment();
                }
            }

            @Override
            public int getItemCount() {
                return 3;
            }
        });
    }
}
```

### 三、ViewPager2的新特性

#### 3.1 声明

* 在上文说`ViewPager`预加载时，我就在想`offscreenPageLimit`能不能称之为`预加载`，如果在`ViewPager`上可以，那么在`ViewPager2`上可能就要混淆了，因为`ViewPager2`拥有`RecyclerView`的一整套缓存策略，包括`RecyclerView`的预加载；为了避免混淆，在下面的文章中我把`offscreenPageLimit`定义为`离屏加载`，`预加载`只代表`RecyclerView`的预加载；
* `setOffscreenPageLimit`定义为`离屏加载`
* `预加载`只代表`RecyclerView`的预加载；

#### 3.2 ViewPager2的离屏加载

##### 3.2.1 源码分析#setOffscreenPageLimit

```java
public static final int OFFSCREEN_PAGE_LIMIT_DEFAULT = -1;
private @OffscreenPageLimit int mOffscreenPageLimit = OFFSCREEN_PAGE_LIMIT_DEFAULT;

public void setOffscreenPageLimit(@OffscreenPageLimit int limit) {
    //分析1
	if (limit < 1 && limit != OFFSCREEN_PAGE_LIMIT_DEFAULT) {
    	throw new IllegalArgumentException(
        	"Offscreen page limit must be OFFSCREEN_PAGE_LIMIT_DEFAULT or a number > 0");
	}
    //分析2
    mOffscreenPageLimit = limit;
    // Trigger layout so prefetch happens through getExtraLayoutSize()
    mRecyclerView.requestLayout();
}
```

* 分析1：可以通过判断发现，我们可以设置limit的值为OFFSCREEN_PAGE_LIMIT_DEFAULT即是-1。如果设置其他小于1的值会抛出异常。其实-1就是我们用来去除预加载的标志位，mOffscreenPageLimit其默认值就是-1，也就说明默认没有预加载功能。

##### 3.2.2 calculateExtraLayoutSpace

* 实际上`OffscreenPageLimit`本质上是重写`LinearLayoutManager`的`calculateExtraLayoutSpace`方法，该方法是最新的`recyclerView`包加入的功能；

```java
@Override
protected void calculateExtraLayoutSpace(@NonNull RecyclerView.State state,
		@NonNull int[] extraLayoutSpace) {
	int pageLimit = getOffscreenPageLimit();
    if (pageLimit == OFFSCREEN_PAGE_LIMIT_DEFAULT) {
    	// Only do custom prefetching of offscreen pages if requested
        super.calculateExtraLayoutSpace(state, extraLayoutSpace);
        return;
    }
	final int offscreenSpace = getPageSize() * pageLimit;
    extraLayoutSpace[0] = offscreenSpace;
    extraLayoutSpace[1] = offscreenSpace;
}
```

* getOffscreenPageLimit方法获取的就是上面的mOffscreenPageLimit值，然后判断是否等于-1，如果等于的话就不进行额外的操作
* 如果不等于1的话，就获取getPageSize() * pageLimit的空间大小，然后放入到数组中去。

* getPageSize就是获取当前RecyclerView的宽/高长度（根据水平/垂直方向获取对应的）

```java
int getPageSize() {
	final RecyclerView rv = mRecyclerView;
    return getOrientation() == ORIENTATION_HORIZONTAL
    	? rv.getWidth() - rv.getPaddingLeft() - rv.getPaddingRight()
        : rv.getHeight() - rv.getPaddingTop() - rv.getPaddingBottom();
}
```

**小结**

* `calculateExtraLayoutSpace`方法定义了布局额外的空间，何为布局额外的空间？默认空间等于RecyclerView的宽高空间，定义这个意在可以放大可布局的空间，该方法参数`extraLayoutSpace`是一个长度为2的int数组，第一条数据接受左边/上边的额外空间，第二条数据接受右边/下边的额外空间，故上诉代码是表明左右/上下各扩大`offscreenSpace`；
* `OffscreenPageLimit`其实就是放大了`LinearLayoutManager`的布局空间

#### 3.3 ViewPager2预加载和缓存

##### 3.3.1 概述

* `ViewPager2预加载`即`RecyclerView`的预加载，代码在`RecyclerView`的`GapWorker`中，这个知识可能有些同学不是很了解，推荐先看这篇博客[medium.com/google-deve…](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710)；

* 在`ViewPager2`上默认开启预加载，表现形式是在拖动控件或者`Fling`时，可能会预加载一条数据；下面是预加载的示意图：

![img](https://user-gold-cdn.xitu.io/2019/5/13/16ab12a07ee251aa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 3.3.2 如何关闭预加载？

```java
((RecyclerView)viewPager.getChildAt(0)).getLayoutManager().setItemPrefetchEnabled(false);
```

* 预加载的开关在`LayoutManager`上，只需要获取`LayoutManager`并调用`setItemPrefetchEnabled()`即可控制开关；

##### 3.3.3 自定义缓存条数

* `ViewPager2`默认会缓存2条`ItemView`，而且在最新的`RecyclerView`中可以自定义缓存Item的个数；

* RecyclerView.java

```
public void setItemViewCacheSize(int size) {
    mRecycler.setViewCacheSize(size);
}
```

##### 3.3.4 小结

* `预加载`和`缓存`在`View`层面没有本质的区别，都是已经准备了布局，但是没有加载到parent上； 
* `预加载`和`离屏加载`在`View`层面有本质的区别，`离屏加载`的View已经添加到parent上；

### 四、提前加载对Adapter影响

#### 4.1 提前加载

* 所谓的提前加载，是指当前`position`不可见但加载了布局，包括上面说的`预加载`和`离屏加载`。

#### 4.2 Adapter

##### 4.2.1 常用方法

`ViewPager2`的`Adapter`本质上是`RecyclerView.Adapter`，下面列举常用方法：

- `onCreateViewHolder(ViewGroup parent, int viewType)`：创建ViewHolder
- `onBindViewHolder(VH holder, int position)`：绑定ViewHolder
- `onViewRecycled(VH holder)`：当View被回收
- `onViewAttachedToWindow(VH holder)`：当前View加载到窗口
- `onViewDetachedFromWindow(VH holder)`：当前View从窗口移除
- `getItemCount()`：获取Item个数

##### 4.2.2 离屏加载和预加载调用Adapter方法

下面主要针对`ItemView`的创建来说，暂不讨论回收的情况；

- `onBindViewHolder`：预加载和离屏加载都会调用
- `onViewAttachedToWindow` ：离屏加载ItemView会调用，可见ItemView会调用
- `onViewDetachedFromWindow` ：从可见到不可见的ItemView(除离屏加载中)必定调用

##### 4.2.3 小结

* `预加载`和`缓存`在`Adapter`层面没有区别，都会调用`onBindViewHolder`方法；
*  `预加载`和`离屏加载`在`Adapter`层面有本质的区别，`离屏加载`的View会调用`onViewAttachedToWindow`；
* 最主要的区别就是离屏加载此时ItemView已经被加载到窗口了，而预加载此时没有加载到窗口。

### 五、ViewPager2对Fragment支持

#### 5.1 FragmentStateAdapter

* 目前，`ViewPager2`对`Fragment`的支持只能使用`FragmentStateAdapter`，使用起来也是非常简单

```java
ViewPager2 viewPager2 = findViewById(R.id.viewpager2);
viewPager2.setAdapter(new FragmentStateAdapter(this) {
	@NonNull
    @Override
    public Fragment createFragment(int position) {
    	switch (position){
        	case 0:
            	return new TestFragment();
			case 1:
            	return new Test2Fragment();
			default:
            	return new Test3Fragment();
		}
    }

	@Override
    public int getItemCount() {
            return 3;
	}
});
```

#### 5.2 实例

##### 5.2.1 代码

```java
ViewPager2 viewPager2 = findViewById(R.id.viewpager2);
viewPager2.setAdapter(new FragmentStateAdapter(this) {
	@NonNull
    @Override
    public Fragment createFragment(int position) {
    	switch (position){
        	case 0:
            	return new TestFragment();
			case 1:
            	return new Test2Fragment();
			case 2:
            	return new Test3Fragment();
			case 3:
				return new Test4Fragment();
			default:
            	return new Test5Fragment();
		}
    }

	@Override
    public int getItemCount() {
    	return 5;
	}
});
```

##### 5.2.2 默认情况下生命周期

```java
//第一页
生命周期：: TestFragment:onCreate
生命周期：: TestFragment:onCreateView
生命周期：: TestFragment:onStart
生命周期：: TestFragment:onResume

//第二页
生命周期：: Test2Fragment:onCreate
生命周期：: Test2Fragment:onCreateView
生命周期：: Test2Fragment:onStart
生命周期：: TestFragment:onPause
生命周期：: Test2Fragment:onResume

//第三页
生命周期：: Test3Fragment:onCreate
生命周期：: Test3Fragment:onCreateView
生命周期：: Test3Fragment:onStart
生命周期：: Test2Fragment:onPause
生命周期：: Test3Fragment:onResume

//第四页
生命周期：: Test4Fragment:onCreate
生命周期：: Test4Fragment:onCreateView
生命周期：: Test4Fragment:onStart
生命周期：: TestFragment:onStop
生命周期：: TestFragment:onDestroyView
生命周期：: TestFragment:onDestroy
生命周期：: Test3Fragment:onPause
生命周期：: Test4Fragment:onResume

//第五页
生命周期：: Test5Fragment:onCreate
生命周期：: Test5Fragment:onCreateView
生命周期：: Test5Fragment:onStart
生命周期：: Test4Fragment:onPause
生命周期：: Test5Fragment:onResume
```

* 从上面可以看到默认是没有开启离屏加载的，但是onStop这些方法有时候有有时候没有，暂时未知什么原因

##### 5.2.3 设置离屏加载生命周期

```
viewPager2.setOffscreenPageLimit(1);
```

* 输出结果

```java
//第一页
生命周期：: TestFragment:onCreate
生命周期：: TestFragment:onCreateView
生命周期：: TestFragment:onStart
生命周期：: TestFragment:onResume
生命周期：: Test2Fragment:onCreate
生命周期：: Test2Fragment:onCreateView
生命周期：: Test2Fragment:onStart
    
//第二页
生命周期：: Test3Fragment:onCreate
生命周期：: Test3Fragment:onCreateView
生命周期：: Test3Fragment:onStart
生命周期：: TestFragment:onPause
生命周期：: Test2Fragment:onResume

//第三页
生命周期：: Test4Fragment:onCreate
生命周期：: Test4Fragment:onCreateView
生命周期：: Test4Fragment:onStart
生命周期：: Test2Fragment:onPause
生命周期：: Test3Fragment:onResume
    
//第四页
生命周期：: Test5Fragment:onCreate
生命周期：: Test5Fragment:onCreateView
生命周期：: Test5Fragment:onStart
生命周期：: Test3Fragment:onPause
生命周期：: Test4Fragment:onResume
    
//第五页
生命周期：: Test4Fragment:onPause
生命周期：: Test5Fragment:onResume
```

* 可以看到第一页的时候就会调用第二页的onCreate等，第二页会调用第三页的onCreate等。
* 实例上测试，切换没有调用onStop等方法，只有Activity finsish的时候，才会调用所有Fragment的onStop等方法，暂时未知。



