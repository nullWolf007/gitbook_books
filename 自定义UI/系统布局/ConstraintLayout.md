[TOC]

# ConstraintLayout

#### 参考链接

* [android ConstraintLayout使用详解](https://www.jianshu.com/p/f86f800964d2)

## 一、前言

### 1.1 优点：

* 避免过度绘制
* 自适应布局

## 二、基本使用

### 2.1 文字对齐

*  ```
	app:layout_constraintBaseline_toBaselineOf=""
	```

### 2.2 margin

* 只有存在约束了，margin才能起效果
* goneMargin：当其他控件gone的时候，此时依赖此控件的约束条件将会失效，goneMargin将会起作用

### 2.3 bias偏移量

* ```xml
  app:layout_constraintHorizontal_bias="0.8"
  app:layout_constraintVertical_bias="0.5"
  ```

* 只有存在约束条件时，才能起作用。如左右偏移，则左右都需要约束。上下偏移，则上下都需要约束

### 2.4 Circle环形约束

* ```xml
  app:layout_constraintCircle="" <!--以哪个控件为圆心-->
  app:layout_constraintCircleAngle="" <!--角度，如30表示30°-->
  app:layout_constraintCircleRadius="" <!--半径-->
  ```

### 2.5 chainStyle链式风格

* ```xml
  app:layout_constraintHorizontal_chainStyle="" <!--spread/spread_inside/packed-->
  app:layout_constraintVertical_chainStyle="" <!--spread/spread_inside/packed-->
  ```

* spread(默认)：元素被分散开来

![spread](..\..\images\自定义UI\系统布局\ConstraintLayout\spread.jpg)

* spread_inside：与spread类似，但是链条两边的端点不会分散

![spread_inside](..\..\images\自定义UI\系统布局\ConstraintLayout\spread_inside.jpg)

* packed：链条的元素将被捆在一起

![packed](..\..\images\自定义UI\系统布局\ConstraintLayout\packed.jpg)



### 2.6 weight

* 权重

### 2.7 default：默认风格

* 此属性只有在width/height设置为0dp的时候起作用

* ```xml
  app:layout_constraintWidth_default="spread" <!--spread/percent/wrap_parent-->
  app:layout_constraintHeight_default="spread" <!--spread/percent/wrap_parent-->
  ```

* spread(默认)：等价于match_parent

* percent(百分比)：

  * ```xml
    android:layout_width="0dp"
    app:layout_constraintWidth_default="percent"
    app:layout_constraintWidth_percent="0.8"
    android:layout_height="wrap_content"
    ```

* wrap：等价于wrap_parent

### 2.8 Ratio布局宽高比

* ```xml
  app:layout_constraintDimensionRatio="3:1"
  ```

### 2.9 GuideLine：辅助线

* ```xml
  <androidx.constraintlayout.widget.Guideline
  	android:id="@+id/guideline"
      android:layout_width="wrap_content"
      android:layout_height="0dp"
      android:orientation="vertical"
      app:layout_constraintGuide_percent="0.55" />
  ```

* 这是辅助工具，帮助开发者开发使用的，并不会绘制到屏幕上

### 2.10 Barrier屏障

* 这是辅助工具，帮助开发者开发使用的，并不会绘制到屏幕上
* 一般来说是为了有界限，防止别的控件越界，占用了当前控件的位置
* `constraint_referenced_ids`将一些View包裹在一起形成一个屏障，然后通过属性`barrierDirection`向左上右下四个方向给某个View提供约束条件，或者叫做屏障方向。使用这些约束条件(屏障方向)的View可以防止屏障内的View覆盖自己，当屏障内的某个View要覆盖自己的时候，屏障会自动移动，避免自己被覆盖。

### 2.11 Group：组

* 主要功能：将不同的控件放在一个组里，可以一起隐藏和显示

* ```xml
  <android.support.constraint.Group
  	android:id="@+id/group1"
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      app:constraint_referenced_ids="button3, textView2"/>
  ```

* 可以对group进行setVisibility，将会队组里所有控件生效

### 2.12 PlaceHolder

## 三、实例

### 3.1 按比例缩放

* 代码

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      xmlns:tools="http://schemas.android.com/tools"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      tools:context=".MainActivity">
  
      <TextView
          android:layout_width="0dp"
          android:layout_height="0dp"
          android:background="#00BFFF"
          android:gravity="center"
          android:text="Test"
          android:textColor="#FFFFFF"
          android:textSize="40sp"
          app:layout_constraintDimensionRatio="16:9"
          app:layout_constraintEnd_toStartOf="@id/guideline"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toTopOf="parent" />
  
  
      <androidx.constraintlayout.widget.Guideline
          android:id="@+id/guideline"
          android:layout_width="wrap_content"
          android:layout_height="0dp"
          android:orientation="vertical"
          app:layout_constraintGuide_percent="0.55" />
  
  </androidx.constraintlayout.widget.ConstraintLayout>
  ```

* 效果图

  ![demo1](..\..\images\自定义UI\系统布局\ConstraintLayout\deno1.gif)