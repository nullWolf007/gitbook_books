[TOC]

# Java虚拟机

#### 参考

* [Android进阶解密](https://book.douban.com/subject/30358046/)

## 一、基础知识

### 1.1 概述

#### 1.1.1 含义

* Java虚拟机是整个Java平台的基石，是Java语言编译代码的运行平台，你可以把Java虚拟机看作一个抽象的计算机，它有各种指令集和各种运行时数据区域

#### 1.1.2 Java虚拟机执行流程

![Java虚拟机执行流程](..\..\images\必备Java知识\JVM\Java虚拟机执行流程.PNG)

* 分为编译时环境和运行时环境
* 从图中可以看出，Java虚拟机在乎的.class文件，而不关心源文件的格式，.java也行.kt也可以，只要最后编译成为.class文件就可以

#### 1.1.3 Class文件格式

```java
ClassFile { 
    u4 magic;  // 魔法数字，表明当前文件是.class文件，固定0xCAFEBABE
    u2 minor_version; // 分别为Class文件的副版本和主版本
    u2 major_version; 
    u2 constant_pool_count; // 常量池计数
    cp_info constant_pool[constant_pool_count-1];  // 常量池内容
    u2 access_flags; // 类访问标识
    u2 this_class; // 当前类
    u2 super_class; // 父类
    u2 interfaces_count; // 实现的接口数
    u2 interfaces[interfaces_count]; // 实现接口信息
    u2 fields_count; // 字段数量
    field_info fields[fields_count]; // 包含的字段信息 
    u2 methods_count; // 方法数量
    method_info methods[methods_count]; // 包含的方法信息
    u2 attributes_count;  // 属性数量
    attribute_info attributes[attributes_count]; // 各种属性
}
```

* u2：2字节，无符号类型
* u4：4字节，无符号类型

#### 1.1.4 类的生命周期

* 一个Java 文件被加载到 Java 虚拟机内存中到从内存中卸载的过程被称为类的生命周期。类的生命周期包括的阶段分别是：加载、链接、初始化、使用和卸载，其中链接包括 了三个阶段：验证、准备和解析，因此类的生命周期包括了7个阶段。广义上来说类的加载包括了类的生命周期的5个阶段，分别是加载、链接（验证、准备和解析）、初始化。

##### 1.1.4.1 类的加载

* 加载：查找并加载 Class 文件 

* 链接：包括验证、准备和解析
  * 验证 ：确保被导入类型的正确性。 
  * 准备：为类的静态字段分配字段，并用默认值初始化这些字段。 
  * 解析：虚拟机将常量池内的符号引用替换为直接引用。
* 初始化：将类变量初始化为正确初始值。

## 二、Java内存区域

* 详情请查看[Java内存区域](必备Java知识/JVM/Java内存区域.md)

## 三、GC

* 详情请查看[垃圾回收GC](必备Java知识/JVM/GC/垃圾回收GC.md)