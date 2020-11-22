[TOC]

## SystemServer进程解析

#### 参考

* [Android进阶解密](https://book.douban.com/subject/30358046/)

### 一、简介

#### 1.1 含义

* SystemServer进程主要用于创建系统服务，我们熟知AMS、WMS和PMS都是由他创建的

#### 1.2 主要工作

* 启动Binder线程池，这样就可以于其他进程进行通信
* 创建SysetmServiceManager，其用于对系统的服务进行创建、启动和生命周期的管理
* 启动各种服务：引导服务、核心服务、其他服务

#### 1.3 引导服务

* ActivityManagerService ：负责四大组件的启动，切换，调度
* PackageManagerService：用来对APK进行安装、解析、删除、卸载等操作

#### 1.4 核心服务

* 

#### 1.5 其他服务

* CameraService：摄像头相关服务
* AlarmManagerService：全局定时器管理服务

### 二、流程

![Zygote处理SystemServer进程](..\..\images\Android组件内核\FrameWork内核解析\Zygote处理SystemServer进程.png)

### 三、Zygote启动SystemServer源码

#### 3.1 startSystemServer

* ZygoteInit.java#startSystemServer

```java
private static boolean startSystemServer(String abiList, String socketName, ZygoteServer zygoteServer)throws Zygote.MethodAndArgsCaller, RuntimeException {

	......
	//创建args数组，这个数组用来保存启动SystemServer的启动参数
    String args[] = {
    	"--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer",
};
	ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
    	parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        //fork一个子进程，也就是SystemServer进程
        pid = Zygote.forkSystemServer(
        	parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
            null,
            parsedArgs.permittedCapabilities,
            parsedArgs.effectiveCapabilities);
	} catch (IllegalArgumentException ex) {
    	throw new RuntimeException(ex);
	}

     //当前逻辑运行在子进程中
     if (pid == 0) {
     	if (hasSecondZygote(abiList)) {
        	waitForSecondaryZygote(socketName);
		}

        zygoteServer.closeServerSocket();
         //处理SystemServer进程
        handleSystemServerProcess(parsedArgs);
	}

    return true;
}
```

* args中保存了“com.android.server.SystemServer”值，用于后面进行反射
* 我们可以找到handleSystemServerProcess方法。

#### 3.2 handleSystemServerProcess

* ZygoteInit.java

```java
private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs)
	throws Zygote.MethodAndArgsCaller {
	if (parsedArgs.invokeWith != null) {
		......
	} else {
    	ClassLoader cl = null;
        if (systemServerClasspath != null) {
            //创建PathClassLoader
        	cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);

            Thread.currentThread().setContextClassLoader(cl);
		}
		//重点
        ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
	}
}
```

* 最终会调用ZygoteInit.zygoteInit方法

#### 3.3 zygoteInit

* ZygoteInit.java

```java
public static final void zygoteInit(int targetSdkVersion, String[] argv,
	ClassLoader classLoader) throws Zygote.MethodAndArgsCaller {
    if (RuntimeInit.DEBUG) {
    	Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
	}

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    //启动Binder线程池
    ZygoteInit.nativeZygoteInit();
    //进入SystemServer的main方法
    RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```

#### 3.4 nativeZygoteInit启动线程池

* 详细步骤可以参考[应用程序进程启动过程](应用程序进程启动过程.md)中线程池的启动过程

* nativeZygoteInit是一个native的方法
* AndroidRuntime.cpp
* 最终调用了app_main.cpp的onZygoteInit()方法
* app_main.cpp的onZygoteInit()最终调用startThreadPool()方法启动了Binder线程池

#### 3.5 applicationInit启动SystemServer的main

* RuntimeInit.java#applicationInit

```java
protected static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
	throws Zygote.MethodAndArgsCaller {
	......
    
	invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```

* 最终会调用invokeStaticMain方法

#### 3.6 invokeStaticMain

* RuntimeInit.java#invokeStaticMain

* className为"com.android.server.SystemServer"，层层传递过来的

```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
	throws Zygote.MethodAndArgsCaller {
    Class<?> cl;

    try {
        //通过反射找到SystemServer类
    	cl = Class.forName(className, true, classLoader);
	} catch (ClassNotFoundException ex) {
    	throw new RuntimeException(
        	"Missing class when invoking static main " + className,
            ex);
	}

	Method m;
    try {
        //找到SystemServer的main方法
    	m = cl.getMethod("main", new Class[] { String[].class });
	} catch (NoSuchMethodException ex) {
    	throw new RuntimeException(
        	"Missing static main on " + className, ex);
	} catch (SecurityException ex) {
    	throw new RuntimeException(
        	"Problem getting static main on " + className, ex);
	}

	......
	//执行SystemServer的main方法
	throw new Zygote.MethodAndArgsCaller(m, argv);
}
```

* 通过反射的技术找到SystemServer的main方法
* 然后通过抛出异常的方式去执行SystemServer的main方法
* 为什么要用抛出异常的方式？这种抛出异常的处理会清除所有的设置过程需要的堆栈帧，并让SystemServer的main方法看起来像是SystemServer进程的入口方式

#### 3.7 main

* ZygoteInit.java#main

```java
public static void main(String argv[]) {
	......
	try {
		......
	} catch (Zygote.MethodAndArgsCaller caller) {
    	caller.run();
	} catch (Throwable ex) {
    	Log.e(TAG, "System zygote died with exception", ex);
        zygoteServer.closeServerSocket();
        throw ex;
	}
}
```

* 通过捕获异常的方式最终会调用MethodAndArgsCaller的run方法

#### 3.8 run

* Zygote.java#MethodAndArgsCaller#run

```java
public static class MethodAndArgsCaller extends Exception
	implements Runnable {
    private final Method mMethod;
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
    	mMethod = method;
        mArgs = args;
	}

    public void run() {
    	try {
            //执行SystemServer的main方法
        	mMethod.invoke(null, new Object[] { mArgs });
		}
        ......
	}
}
```

* 通过Method的invoke方法去执行SystemServer的main方法

#### 3.9 main

* SystemServer.java#main

### 四、SystemServer进程源码解析

#### 4.1 main

* SystemServer.java#main

```java
public static void main(String[] args) {
	new SystemServer().run();
}
```

* SystemServer.java#run

```java
private void run() {
	try {
         ......//虚拟机相关设置等

		//设置线程优先级
        android.os.Process.setThreadPriority(
        	android.os.Process.THREAD_PRIORITY_FOREGROUND);
		android.os.Process.setCanSelfBackground(false);
        //创建消息Looper
        Looper.prepareMainLooper();
        //加载动态库android_servers
		System.loadLibrary("android_servers");
		performPendingShutdown();
        //创建系统的Context
		createSystemContext();
		//创建SystemServiceManager实例
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
		mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
		SystemServerInitThreadPool.get();
	} finally {
    	traceEnd();  // InitBeforeStartServices
	}

	try {
		traceBeginAndSlog("StartServices");
        //启动引导服务
        startBootstrapServices();
        //启动核心服务
        startCoreServices();
        //启动其他服务
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
	} ......
    //进入Loop循环，处理消息循环
	Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

