[TOC]
# APP的启动过程

### 参考文章

* [**ActivityThread的理解和APP的启动过程**](https://blog.csdn.net/hzwailll/article/details/85339714)

### 一、系统的启动过程

* APP的启动简单的流程图

  > 加载BootLoader-->初始化内核-->启动init进程-->init进程fork出Zygote进程-->Zygote进程fork出SystemServer进程

![APP系统启动过程](..\..\..\images\android\进阶\APP系统启动过程.png)

* SystemServer进程是系统进程。很多系统服务，如ActivityManagerService、PackageManagerService、WindowManagerService等都是存在该进程并创建成功后启动
* ActivityManagerServices(AMS)：是一个服务端对象，负责所有Activity的生命周期，AMS通过Binder与Activity通信，而AMS与Zygote之间是通过Socket通信
* ActivityThread：UI线程/主线程，它的main()方法是APP的真正入口
* ApplicationThread：实现IBinder接口的ActivityThread内部类，用于ActivityThread和AMS的所在进程间通信
* Instrumentation：可以理解为ActivityThread的一个工具类，在ActivityThread中初始化，一个进程只存在一个Instrumentation对象，在每个Activity初始化时，会通过Activity的Attach方法，将该引用传递给Activity。Activity所有生命周期的方法都由该类来执行

### 二、APP的启动过程

![APP启动过程](..\..\..\images\android\进阶\APP启动过程.png)

1. 点击桌面APP图标时，Launcher的startActivity()方法，通过Binder通信，调用SystemServer进程中的AMS服务
2. SystemServer进程收到请求后，向Zygote进程发送创建进程的请求
3. Zygote进程fork出App进程，通过反射来执行ActivityThread的main方法，进行初始化。初始化MainLooper，主线程的Handler以及初始化ApplicationThread用于和AMS通信交互
4. App进程，通过Binder向SystemServer进程发起attachApplication请求，这里实际上就是App进程通过Binder调用SystemServer进程中AMS的attachApplication方法，AMS的attachApplication方法的作用是将ApplicationThread对象和AMS绑定
5. SystemServer进程在收到attachApplication的请求，进行一些准备工作后，再通过binder IPC向App进程发送handleBindApplication请求（初始化Application并调用onCreate方法）和scheduleLaunchActivity请求（创建启动Activity）
6. App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送BIND_APPLICATION和LAUNCH_ACTIVITY消息，这里注意的是AMS和主线程并不直接直接通信，因为是不同的进程，而是AMS和主线程的内部类ApplicationThread通过Binder通信，ApplicationThread再和主线程通过Handler消息交互
7. 主线程在收到Message后，创建Application并调用onCreate方法，再通过反射机制创建目标Activity，并调用Activity.onCreate()等方法
8. 至此，App便正式启动，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染后显示App主界

