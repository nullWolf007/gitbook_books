## 零、引言

* [说明](README.md)

## 第一章 必备Java知识

* 1.1 并发编程

  * 1.1.1 基础知识
    
    * [线程状态及基本操作](必备Java知识/并发编程/并发原理/线程状态及基本操作.md)
  * 1.1.2 并发原理
    
    * [Java内存模型以及happens-before](必备Java知识/并发编程/并发原理/Java内存模型以及happens-before.md)
    * [原子性,有序性,可见性](必备Java知识/并发编程/并发原理/原子性,有序性,可见性)
  * 1.1.3 并发关键字
    * [线程同步之Synchronized](必备Java知识/并发编程/并发关键字/线程同步之Synchronized.md)
    * [线程同步之volatile](必备Java知识/并发编程/并发关键字/线程同步之volatile.md)
  * 1.1.4 Lock
    * [线程同步之Lock](必备Java知识/并发编程/Lock/线程同步之Lock.md)
  * 1.1.5 原子操作类
    * [原子操作类](必备Java知识/并发编程/原子操作类/原子操作类.md)
  * 1.1.6 线程协作
    * [线程间协作](必备Java知识/并发编程/线程协作/线程间协作.md)
  * 1.1.7 线程池
    * [线程池基本解析](必备Java知识/并发编程/线程池/线程池基本解析.md)
    * [Callable,Future和FutureTask](必备Java知识/并发编程/线程池/Callable,Future和FutureTask.md)
  * 1.1.8 线程相关类
    * [ThreadLocal解析](必备Java知识/并发编程/线程相关类/ThreadLocal解析.md)
  * 1.1.9 死锁
    * [死锁](必备Java知识/并发编程/死锁/死锁.md)
  
  * [多线程解析](必备Java知识/并发编程/多线程解析.md)
  
  * [多线程同步解析](必备Java知识/并发编程/多线程同步解析.md)
  * [并发常见面试题](必备Java知识/并发编程/并发常见面试题.md)
* 1.2 JVM

  * 1.2.1GC 
    * [强引用、软引用、弱引用、虚引用](必备Java知识/JVM/GC/强引用、软引用、弱引用、虚引用.md)
    * [Java内存区域](必备Java知识/JVM/Java内存区域.md)
    * [垃圾回收GC](必备Java知识/JVM/GC/垃圾回收GC.md)
* 1.3 反射与类加载
  * 1.3.1 类
    * [内部类详解](必备Java知识/反射与类加载/类/内部类详解.md)
  * 1.3.2 反射
    * [Java反射机制](必备Java知识/反射与类加载/反射/Java反射机制.md)

  * 1.3.3 类加载
    * [Java类加载](必备Java知识/反射与类加载/类加载/Java类加载.md)
* 1.4 关键字
  * 1.4.1 [static](必备Java知识/关键字/static.md)
  * 1.4.2 [final](必备Java知识/关键字/final.md)
* 1.5 注解
  * 1.5.1 [Java注解](必备Java知识/注解/Java注解.md)

## 第2章 数据结构及算法

* 2.1 数据结构 
  * 2.1.1 线性表
  * 2.1.2 链表
  * 2.1.3 栈
  * 2.1.4 队列
  * 2.1.5 树
    * [二叉树排序](数据结构及算法/数据结构/树/二叉树排序.md)
  * 2.1.6 图
  * 2.1.7 Hash
    * [HashMap解析](数据结构及算法/数据结构/Hash/HashMap解析.md)
* 2.2 排序算法
  * [常见排序算法](数据结构及算法/排序算法/常见排序算法.md)
  * [线性排序](数据结构及算法/排序算法/线性排序.md)
* 2.3 查找算法
* 2.4 算法思想和技巧
  * [五大算法思想](数据结构及算法/算法思想和技巧/五大算法思想.md)
* 2.5 常见问题
  * [字符串匹配](数据结构及算法/常见问题/字符串匹配.md)
  * [常见算法技巧和题型](数据结构及算法/常见问题/常见算法技巧和题型.md)

## 第3章 Android必备技能

* 3.1 数据传输及序列化
  * [Serializable和Parcelable比较](Android必备技能/数据传输及序列化/Serializable和Parcelable比较.md)
* 3.2 并发
  * [AsyncTask](Android必备技能/并发/AsyncTask.md)
* 3.3 Hook
  * [代理模式](Android必备技能/Hook/代理模式.md)
  * [Hook](Android必备技能/Hook/Hook.md)

## 第4章 Android组件内核

* 4.1 四大组件
    * 4.1.1 Activity

    * 4.1.2 Service
      
    * 4.1.3 BroadcastReceiver
      
    * 4.1.4 ContentProvider
* 4.2 FrameWork内核解析
  
  * [Android系统架构](Android组件内核/FrameWork内核解析/Android系统架构.md)
  * [Android系统启动](Android组件内核/FrameWork内核解析/Android系统启动.md)
  * [Zygote进程解析](Android组件内核/FrameWork内核解析/Zygote进程解析.md)
  * [SystemServer进程解析](Android组件内核/FrameWork内核解析/SystemServer进程解析.md)
  * [应用程序进程启动过程](Android组件内核/FrameWork内核解析/应用程序进程启动过程.md)
  * [根Activity启动过程](Android组件内核/FrameWork内核解析/根Activity启动过程.md)
  * [ActivityThread解析](Android组件内核/FrameWork内核解析/ActivityThread解析.md)
* 4.3 Android消息机制
  * [ThreadLocal解析](必备Java知识/并发编程/线程/ThreadLocal解析.md)
  * [Handler和Message和MessageQueue和Looper](Android组件内核/Android消息机制/Handler和Message和MessageQueue和Looper.md)
* 4.4 跨进程通信IPC
  * [Binder解析](Android组件内核/跨进程通信IPC/Binder解析.md)
* 4.5 Intent
    * 

## 第5章 高级UI

* 5.1 系统布局
  * [ConstraintLayout](高级UI/系统布局/ConstraintLayout.md)
  * [RemoteViews解析](高级UI/系统布局/RemoteViews解析.md)
* 5.2 自定义View
  
  * [一、基础知识](高级UI/自定义View/一、基础知识.md)
  * [二、绘制前的准备：DecorView创建和显示](高级UI/自定义View/二、绘制前的准备：DecorView创建和显示.md)
  * [三、Measure过程](高级UI/自定义View/三、Measure过程.md)
  * [四、Layout过程](高级UI/自定义View/四、Layout过程.md)
  * [五、Draw过程](高级UI/自定义View/五、Draw过程.md)
  * [六、自定义View实例](高级UI/自定义View/六、自定义View实例.md)
  * [View工作原理](高级UI/自定义View/View工作原理.md)
  * [View的事件分发机制](高级UI/自定义View/View的事件分发机制.md)
* 5.3 Webview
  * [Webview与JS交互](高级UI/Webview/Webview与JS交互.md)
  * [Webview](高级UI/Webview/Webview.md)

## 第6章 数据存储

* 6.1 文件存储

* 6.2 轻量级KV
* 6.3 SharedPreference
	
* 6.4 MMKV
	
* 6.5 Sqlite

## 第7章 常见机制

## 第8章 设计思想解读开源库框架

* 8.1 热修复
* 8.2 图片加载

  * 8.2.1 Bitmap
    * [Bitmap解析](设计思想解读开源框架库\图片加载\Bitmap\Bitmap解析.md)
  * 8.2.2 Glide
* 8.3 网络访问框架
* 8.3.1 网络基础
	
* 8.3.2 OkHttp
	
* 8.3.3 Retrofit
* 8.4 RxJava响应式编程
* 8.5 IOC框架设计
  * 8.5.1 AOP和IOC
  * 8.5.2 ButterKnife
* 8.6 屏幕适配
  * [屏幕适配基础知识](设计思想解读开源框架库\屏幕适配\屏幕适配基础知识.md)
  * [宽高限定符适配](设计思想解读开源框架库\屏幕适配\宽高限定符适配.md)
  * [今日头条适配方案](设计思想解读开源框架库\屏幕适配\今日头条适配方案.md)
  * [基于头条适配的AndroidAutoSize](设计思想解读开源框架库\屏幕适配\基于头条适配的AndroidAutoSize.md)

## 第9章 性能优化

* OOM
  * 

## 第10章 NDK开发

## 第11章 架构能力

* 11.1 架构设计
* 

## 第12章 提高开发效率工具

* 12.1 Groovy
* 12.2 Gradle
* 12.3 Transform 
* 12.4 自定义插件开发
* 12.5 插件实战

## 第13章 功能开发

* 13.1 即时通讯

* 13.2 串口开发

## 第14章 工具使用

* 14.1 Android Studio

* 14.2 Git

## 第15章 语言

* 15.1 Kotlin

## 第16章 面试









