[TOC]

## Zygote进程解析

#### 参考

* [Zygote进程浅析](https://blog.csdn.net/fengluoye2012/article/details/80023051)
* [Android进程系列第二篇---Zygote进程的启动流程](https://www.jianshu.com/p/ab9b83a77af6)
* [Android进阶解密](https://book.douban.com/subject/30358046/)

### 一、简介

#### 1.1 含义

* Zygote进程又称受精卵进程，它是由init进程以Service的方式启动的。Zygote进程最大意义是作为一个Socket的Server端，接收着四面八方的进程创建请求。
* Android中所有的应用进程的创建都是一个应用进程通过Binder方式请求SystemServer进程，SystemServer进程发送socket消息给Zygote进程，统一由Zygote进程创建出来的。

#### 1.2 为什么所有进程需要Zygote创建

* Zygote进程在启动的时候会创建一个虚拟机实例，因此通过Zygote进程在父进程，创建的子进程都会继承这个虚拟机实例，App中的JAVA代码可以得到翻译执行。

#### 1.3 主要工作

* 创建Java虚拟机并Java虚拟机注册JNI方法
* 通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层
* 通过registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程
* 启动SystemServer进程

### 二、流程

![Zygote进程启动过程](..\..\images\Android组件内核\FrameWork内核解析\Zygote进程启动过程.png)

* app_main.cpp的main函数中的AppRuntime的start方法来启动Zygote进程的
* AndroidRuntime.cpp中创建了Java虚拟机并注册了JNI方法。并通过JNI的方式调用了ZygoteInit.java的main方法，实现了从c到java的过程。
* ZygoteInit.java的main方法中，通过调用registerServerSocker的方法创建了一个Server端的Socket.
* ZygoteInit.java使用了preload预加载类和资源
* ZygoteInit.java的startSystemServer的方法创建了SystemServer进程
* ZygoteInit.java的runSelectLoop等待AMS请求来创建新的应用程序进程

### 三、源码记忆

#### 3.1 ZygoteInit#main

* ZygoteInit.java#main

```java
public static void main(String argv[]) {
    ......
    try{
        ......
        //创建了一个Server端的Socket.socketName的值为“zygote”
        zygoteServer.registerServerSocket(socketName);
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            //预加载类和资源
            preload(bootTimingsTraceLog);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                    bootTimingsTraceLog.traceEnd(); // ZygotePreload
        } else {
            Zygote.resetNicePriority();
        }
        ......
        if (startSystemServer) {
            //启动SystemServer进程
            startSystemServer(abiList, socketName, zygoteServer);
        }

        Log.i(TAG, "Accepting command socket connections");
		//等待AMS请求
        zygoteServer.runSelectLoop(abiList);
        zygoteServer.closeServerSocket();
	}......
}
```

* 通过调用registerServerSocker的方法创建了一个Server端的Socket.
* 使用了preload预加载类和资源
* startSystemServer的方法创建了SystemServer进程
* runSelectLoop等待AMS请求来创建新的应用程序进程

#### 3.2 registerServerSocket

* ZygoteInit.java#registerServerSocket
* socketName="zygote"

```java
void registerServerSocket(String socketName) {
	if (mServerSocket == null) {
    	int fileDesc;
        //拼接socket名称
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            //得到Socket的环境变量的值
        	String env = System.getenv(fullSocketName);
            //将Socket环境变量的值转换为文件描述符的参数
            fileDesc = Integer.parseInt(env);
		} catch (RuntimeException ex) {
        	throw new RuntimeException(fullSocketName + " unset or invalid", ex);
		}

        try {
            //创建文件描述符
        	FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);
            //创建服务器端Socket
            mServerSocket = new LocalServerSocket(fd);
		} catch (IOException ex) {
        	throw new RuntimeException(
            	"Error binding to local socket '" + fileDesc + "'", ex);
		}
	}
}
```

* LocalServerSocket

  ```java
  public LocalServerSocket(String name) throws IOException
  {
      //1、创建服务端socket对象
  	impl = new LocalSocketImpl();
      impl.create(LocalSocket.SOCKET_STREAM);
  
      //2、设置地址
      localAddress = new LocalSocketAddress(name);
      //3、绑定地址
      impl.bind(localAddress);
  	  //4、监听
      impl.listen(LISTEN_BACKLOG);
  }
  ```

#### 3.3 preload预加载

##### 3.3.1 预加载

* android系统资源加载分两种方式，预加载和使用进程中加载。 预加载是指在zygote进程启动的时候就加载，这样系统只在zygote执行一次加载操作，所有APP用到该资源不需要再重新加载，减少资源加载时间，加快了应用启动速度，一般情况下，系统中App共享的资源会被列为预加载资源。

##### 3.3.2 预加载原理

* 预加载的原理很简单，就是在zygote进程启动后将资源读取出来，保存到Resources一个全局静态变量中，下次读取系统资源的时候优先从静态变量中查找。主要代码在zygoteInit.java类中方法preloadResources()

#### 3.4 startSystemServer

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

#### 3.4 runSelectLoop

* ZygoteServer.java#runSelectLoop

```java
void runSelectLoop(String abiList) throws Zygote.MethodAndArgsCaller {
	ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
	//获取该Socket的fd字段的值并添加fd到fds中
    fds.add(mServerSocket.getFileDescriptor());
    peers.add(null);

    //无限循环等待AMS请求
	while (true) {
    	StructPollfd[] pollFds = new StructPollfd[fds.size()];
        //遍历fds将fds数组复制到pollFds数组中
        for (int i = 0; i < pollFds.length; ++i) {
        	pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
		}
        try {
        	Os.poll(pollFds, -1);
		} catch (ErrnoException ex) {
        	throw new RuntimeException("poll failed", ex);
		}
        //对pollFds数组进行遍历
        for (int i = pollFds.length - 1; i >= 0; --i) {
        	if ((pollFds[i].revents & POLLIN) == 0) {
            	continue;
			}

            ////等于0表示服务器端Socket与客户端连接上了
            if (i == 0) {
                //获取ZygoteConnection实例 并添加到Socket连接列表peers中
            	ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
			} else {
                //如果不等于0则表示AMS向Zygote进程发送了一个创建应用程序进程的请求
                //调用runOnce创建一个新的应用程序进程
            	boolean done = peers.get(i).runOnce(this);
                if (done) {
                	peers.remove(i);
                    fds.remove(i);
				}
			}
		}
	}
}
```

