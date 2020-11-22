[TOC]

## 自定义View

#### 转载

* [Android：手把手带你清晰梳理自定义View的工作全流程！](https://blog.csdn.net/carson_ho/article/details/98477394)
  ![示意图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NDQzNjUtNzM1NTIwODVhNTA3ZWI1NS5wbmc)

------

### 一、 储备知识

#### 1.1 自定义View基础

* 了解自定义View流程前，需了解一定的自定义View基础，具体请看文章：[一、基础知识](自定义UI/自定义View/一、基础知识.md)

#### 1.2 ViewRoot

##### 1.2.1 定义
* 连接器，对应于`ViewRootImpl`类

##### 1.2.2 作用

* 连接`WindowManager` 和 `DecorView`

* 完成`View`的三大流程： `measure`、`layout`、`draw`

##### 1.2.3 特别注意

```java
// 在主线程中，Activity对象被创建后：
// 1. 自动将DecorView添加到Window中 & 创建ViewRootImpll对象
root = new ViewRootImpl(view.getContent(),display);

// 3. 将ViewRootImpll对象与DecorView建立关联
root.setView(view,wparams,panelParentView)
```

#### 1.3 DecorView

##### 1.3.1 定义

- 顶层`View`

* 即 `Android` 视图树的根节点；同时也是 `FrameLayout` 的子类

##### 1.3.2 作用

- 显示 & 加载布局

* `View`层的事件都先经过`DecorView`，再传递到`View`

##### 1.3.3 特别说明

- 内含1个竖直方向的`LinearLayout`，分为2部分：上 = 标题栏`（titlebar）`、下 = 内容栏`（content）`

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS00OTIzYjYzNzdiMDMyMjU2LnBuZw)

* 在`Activity`中通过 `setContentView（）`所设置的布局文件其实是被加到内容栏之中的，成为其唯一子View = id为content的`FrameLayout`中

```java
// 在代码中可通过content得到对应加载的布局

// 1. 得到content
ViewGroup content = (ViewGroup)findViewById(android.R.id.content);
// 2. 得到设置的View
ViewGroup rootView = (ViewGroup) content.getChildAt(0);
```

#### 1.4 Window、Activity、DecorView 与 ViewRoot的关系

##### 1.4.1 简介

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS1iOWM0MWFhOTk0ZThkZGY0LnBuZw)

##### 1.4.2 之间的关系
![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0zNDk5MmViNDZiZGY5M2U3LnBuZw)

### 二、绘制准备

* 在绘制前，系统会有一些绘制准备，即前面几个步骤：创建`PhoneWindow`类、`DecorView`类、`ViewRootmpl`类等

#### 2.1 DecorView的创建和显示

* [二、绘制前的准备：DecorView创建和显示](自定义UI/自定义View/二、绘制前的准备：DecorView创建和显示.md)

### 三、绘制流程概述

* 从上可知，`View`的绘制流程开始于：`ViewRootImpl`对象的`performTraversals()`

#### 3.1 源码

```java
/**
  * 源码分析：ViewRootImpl.performTraversals()
  */
private void performTraversals() {
	......
	// 1. 执行measure流程
    // 内部会调用performMeasure()
	measureHierarchy(host, lp, res,desiredWindowWidth, desiredWindowHeight);
	......
    // 2. 执行layout流程
    performLayout(lp, mWidth, mHeight);
	......
    // 3. 执行draw流程
    performDraw();
}
```

* 自上而下遍历、由父视图到子视图、每一个 `ViewGroup` 负责测绘它所有的子视图，而最底层的 View 会负责测绘自身

#### 3.2 绘制流程

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS1jMWFkYjlkZDJkMjJjMDU2LnBuZw)

![示意图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NDQzNjUtODU4ZGUxZmFhMzhkZjFiMi5wbmc)

* 下面，我将详细讲解`View`绘制的三大流程：`measure`过程、`layout`过程、`draw`过程

### 四、Measure过程

* 详情请查看[三、Measure过程](自定义UI/自定义View/三、Measure过程.md)

### 五、Layout过程

* 详情请查看[四、Layout过程](自定义UI/自定义View/四、Layout过程.md)

### 六、Draw过程

* 详情请查看[五、Draw过程](自定义UI/自定义View/五、Draw过程.md)

### 七、自定义View的步骤

#### 7.1 步骤1：实现Measure、Layout、Draw流程

- 从View的工作流程（`measure`过程、`layout`过程、`draw`过程）来看，若要实现自定义`View`，根据自定义View的种类不同（单一`View` / `ViewGroup`），需自定义实现不同的方法
- 主要是：`onMeasure()`、`onLayout()`、`onDraw()`，具体如下

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0wMDgyZGU0ZjQ3ZjJkMGMzLnBuZw)

#### 7.2 步骤2：自定义属性

1. 在values目录下创建自定义属性的xml文件
2. 在自定义View的构造方法中加载自定义XML文件 & 解析属性值
3. 在布局文件中使用自定义属性

### 八、自定义View实例

* 详情请查看[六、自定义View实例](自定义UI/自定义View/六、自定义View实例.md)