[TOC]

# AAPT2的解析

### 参考链接

* [Android官网-AAPT2](https://developer.android.google.cn/studio/command-line/aapt2?hl=zh_cn)
* [**APK打包流程**](https://blog.csdn.net/loongago/article/details/89646920)

## 一、前言

### 1.1 什么是AAPT2？

* AAPT2（Android Asset Packing Tool2）是Android资源打包工具也是一种编译工具。负责给Android Studio和Android Gradle Plugin用于编译和打包应用资源。AAPT2会解析资源、为资源编索引并将资源编译为针对Android平台进行过优化的二进制格式
* Android Gradle Plugin 3.0.0及更高版本默认会启用AAPT2。以前使用的是AAPT

### 1.2 说明

* 由于AAPT2是对资源进行编译的工具，所以我们有必要了解apk的打包流程

## 二、APK打包流程

### 1.1 初略流程

![apk简易版本打包流程图](..\..\images\工具使用\AS自带工具\apk简易版本打包流程图.png)

* 大体流程：将你本身项目的资源和所依赖的资源景行编译生成文件(DEX)，然后加上你的Keystore/jks(签名)，通过APK Packager打包工具打包成APK

### 1.2 细化流程

![apk细化版本打包流程图](..\..\images\工具使用\AS自带工具\apk细化版本打包流程图.png)

* 七个打包流程

#### 1.2.1 打包资源文件，生成R.java文件

aapt/aapt2来打包资源文件，生成R.java、resoucres.arsc和res文件

* R.java文件是我们在编写代码的时候会用到的，里面有静态内部类，资源等
* resources.arcs这个文件记录了所有的应用程序资源目录的信息，包括每一个资源名称、类型、值、ID以及所配置的纬度信息。是一个资源索引表，在给定资源ID和设备信息的情况下能快速找到资源

#### 1.2.2 处理aidl文件，生成相应的Java文件

aidl位于android-sdk/build-tools目录下。aidl工具解析接口定义文件然后生成相应的Java代码接口供程序调用，如果项目没有aidl则跳过这步

#### 1.2.3 编译项目源代码，生成class文件

Java Compiler阶段，项目中所有的Java代码，包括R.java和aidl文件，都会被Java编译器编译成.class文件

#### 1.2.4 转换所有的class文件，生成class.dex文件

dex阶段。通过dx工具，将.class文件和第三方库中的.class文件处理生成classes.dex文件。该工具位于android-sdk/build-tools目录下。dx工具的主要工作是将Java字节码转成Dalvik字节码、压缩常量池、消除冗余信息等

#### 1.2.5 打包生成APK文件

apkbuilder阶段。通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk。打包工具apkbuilder位于android-sdk/tools目录下。**当前版本已经删除了apkbuilder工具**

#### 1.2.6 对APK文件进行签名

#### 1.2.7 对签名后的APK文件进行对齐处理 

### 1.3 详细流程

![apk详细版本打包流程图](..\..\images\工具使用\AS自带工具\apk详细版本打包流程图.png)