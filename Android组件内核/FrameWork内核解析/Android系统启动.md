[TOC]

### 参考文章

* [Android进阶解密](https://book.douban.com/subject/30358046/)

### 流程图

<img src="..\..\images\Android组件内核\FrameWork内核解析\Android系统启动.png"  />



## 一、Android系统启动整体流程

### 1. 启动电源以及系统启动

* 当电源按下时引导芯片代码从预定义的地方（固化在ROM）开始执行。加载引导程序BootLoader到RAM中，然后执行

### 2. 引导程序BootLoader

* 引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行

### 3. Linux内核启动

* 当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。在内核完成系统设置后，它首先在系统文件中寻找init.rc文件，并启动init进程

### 4. init进程启动

* init进程做的工作比较多，主要用来初始化和启动属性服务，也用来启动Zygote进程

#### (1) 创建和挂载启动所需的文件目录

#### (2) 初始化和启动属性服务

#### (3) 解析init.rc配置文件并启动Zygote

#### (4) 解析init.rc配置文件并启动ServiceManager

### 5. Zygote进程

#### (1) 创建AppRuntime并调用其start方法，启动Zygote进程

* AppRuntime继承自AndroidRuntime

#### (2) 创建Java虚拟机并为Java虚拟机注册JNI方法

#### (3) 通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层

#### (4) 通过registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等待ASM的请求来创建新的应用程序进程

#### (5) 启动SystemServer进程

### 6. SystemServer进程

#### (1) 启动Binder线程池，这样就可以与其他进程进行通信

#### (2) 创建SystemServiceManager，其用于对系统的服务进行创建、启动和生命周期管理

* SystemServiceManager会创建AMS，PMS，WMS等系统服务。

#### (3) 启动各种系统服务

### 7. ActivityManagerService

#### (1) 将Launcher启动起来

### 8. Launcher

#### (1) 作为Android系统的启动器，用于启动应用程序

#### (2) 作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其他桌面组件

