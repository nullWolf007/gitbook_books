[TOC]
- <!-- TOC -->
  - [ Binder](#binder)
    - [ 参考文章](#参考文章)
    - [ 一、前提知识](#一、前提知识)
      - [ 1.1 内核空间和用户空间](#11-内核空间和用户空间)
        - [ 1.1.1 内核空间](#111-内核空间)
        - [ 1.1.2 用户空间](#112-用户空间)
      - [ 1.2 进程隔离](#12-进程隔离)
        - [ 1.2.1 含义](#121-含义)
        - [ 1.2.2 结论](#122-结论)
      - [ 1.3 传统跨进程通信——管道队列模式](#13-传统跨进程通信——管道队列模式)
      - [ 1.4 内存映射](#14-内存映射)
        - [ 1.4.1 定义](#141-定义)
        - [ 1.4.2 mmap()](#142-mmap)
        - [ 1.4.3 作用](#143-作用)
    - [ 二、Binder简介](#二、binder简介)
      - [ 2.1 介绍](#21-介绍)
      - [ 2.2 概述](#22-概述)
      - [ 2.3 Android为什么选择Binder?](#23-android为什么选择binder)
        - [ 2.3.1 性能](#231-性能)
        - [ 2.3.2 稳定性](#232-稳定性)
        - [ 2.3.3 安全性](#233-安全性)
      - [ 2.4 Binder IPC原理](#24-binder-ipc原理)
        - [ 2.4.1 概述](#241-概述)
        - [ 2.4.2 总结](#242-总结)
    - [ 三、Binder架构](#三、binder架构)
      - [ 3.1 Binder通信模型](#31-binder通信模型)
        - [ 3.1.1 Client进程](#311-client进程)
        - [ 3.1.2 Server进程](#312-server进程)
        - [ 3.1.3 Binder驱动](#313-binder驱动)
        - [ 3.1.4 ServiceManager](#314-servicemanager)
        - [ 3.1.5 总结](#315-总结)
      - [ 3.2 通讯流程](#32-通讯流程)
      - [ 3.4 模型通讯原理](#34-模型通讯原理)
      - [ 3.5 Java层的Binder](#35-java层的binder)
    - [ 四、 Binder整体概述图](#四、-binder整体概述图)
    - [ 五、手写AIDL实例](#五、手写aidl实例)
      - [ 5.1 AIDL定义](#51-aidl定义)
      - [ 5.2 Android开发之AIDL](#52-android开发之aidl)
      - [ 5.3 AIDL支持的数据类型](#53-aidl支持的数据类型)
      - [ 5.4 函数说明](#54-函数说明)
        - [ 5.4.1 DESCRIPTION](#541-description)
        - [ 5.4.2 asInterface()](#542-asinterface)
        - [ 5.4.3 asBinder](#543-asbinder)
        - [ 5.4.4 onTransact()](#544-ontransact)
      - [ 5.5 Person](#55-person)
      - [ 5.6 IPersonManager](#56-ipersonmanager)
      - [ 5.7 Stub](#57-stub)
      - [ 5.8 Proxy](#58-proxy)
      - [ 5.9 RemoteService](#59-remoteservice)
      - [ 5.10 ClientActivity](#510-clientactivity)
      - [ 5.11 AndroidManifest.xml](#511-androidmanifestxml)
  <!-- /TOC -->
## Binder解析

### 参考文章

* [写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)
* [安卓开发艺术探索](https://book.douban.com/subject/26599538/)
* [Android binder全解析](https://blog.csdn.net/u013835855/article/details/78908856)
* [Android Binder](https://www.jianshu.com/p/4c4dcf80d412)
* [binder驱动-------之内存映射篇](https://blog.csdn.net/xiaojsj111/article/details/31422175)
* [Binder驱动之内存映射全解](https://blog.csdn.net/zhanshenwu/article/details/106458188?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)

### 一、前提知识

#### 1.1 内核空间和用户空间

##### 1.1.1 内核空间

* Linux 操作系统和驱动程序运行的空间
* 独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限
* 所有的系统资源管理都是在内核空间完成的，比如读写磁盘文件，分配回收内存，从网络读写数据等，用户空间通过系统调用让内核空间完成这些功能

##### 1.1.2 用户空间

* 应用程序运行的空间

* 表示进程运行在一个特定的操作模式中，没有接触物理内存或设备的权限

#### 1.2 进程隔离

##### 1.2.1 含义

* 我们知道进程之间是无法直接进行交互的，每个进程独享自己的数据，而且操作系统为了保证自身的安全稳定性，将系统内核空间和用户空间分离开来，保证用户程序进程崩溃时不会影响到整个系统。在Linux系统中，虚拟内存机制为每个进程分配了线性连续的内存空间，操作系统将这种虚拟内存空间映射到物理内存空间，每个进程有自己的虚拟内存空间，进而不能操作其他进程的内存空间，每个进程只能操作自己的虚拟内存空间，只有操作系统才有权限操作物理内存空间。

##### 1.2.2 结论

* 进程间，用户空间的数据不可共享
* 为了保证**安全性**和**独立性**，一个进程不能直接操作或者访问另一个进程，即Android的进程是相互独立、隔离的

* 用户空间和内核空间进行交互需要通过系统调用，主要函数

  > copy_from_user()：将用户空间的数据拷贝到内核空间
  >
  > copy_to_user()：将内核空间的数据拷贝到用户空间


#### 1.3 传统跨进程通信——管道队列模式

![管道队列模式跨进程](..\..\images\Android组件内核\跨进程通信IPC\管道队列模式跨进程.png)

* 从上面我们知道此时需要两次拷贝，才能把用户空间的数据传递到另外一个用户空间。那么有没有只需要一次拷贝的方法呢？其实是有的那就是内存映射。

#### 1.4 内存映射

##### 1.4.1 定义

* Linux通过将一个虚拟内存区域与一个磁盘上的对象关联起来，以初始化这个虚拟内存区域的内容，这个过程称为内存映射(memory mapping)

##### 1.4.2 mmap()

* 内存映射的实现过程主要是通过Linux系统下的函数：mmap()。
* 该函数的作用就是**创建虚拟内存区域**和**与共享对象建立映射关系**
* 实现这样的映射关系后，就可以采用指针的方式读写操作这一段内存，而系统会自动回写到对应的文件磁盘上

##### 1.4.3 作用

* 实现内存共享：如跨进程通信

* 提高数据读/写效率：如文件读/写操作

### 二、Binder简介

#### 2.1 介绍

* Binder是Andorid的一个类，实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有，从AndroidFramework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager，WindowManager，等等）和相对应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

#### 2.2 概述

* 是一种Android实现跨进程通讯的方式
* Binder继承了IBinder
* Binder采用了内存映射的方式，从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，因此只需要1次拷贝即可。

#### 2.3 Android为什么选择Binder?

* Android是基于Liunx内核的，所以Android为了实现进程间通信，有liunx的许多方法，如管道、socket、共享内存、消息队列等方式。Android选择Binder，主要是基于**性能、稳定性、安全**三个方面考虑的。

##### 2.3.1 性能

* Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。Binder 只需要一次数据拷贝，性能上仅次于共享内存。

##### 2.3.2 稳定性

* Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度讲，Binder 机制是优于内存共享的。

##### 2.3.3 安全性

* Android 作为一个开放性的平台，市场上有各类海量的应用供用户选择安装，因此安全性对于 Android 平台而言极其重要。作为用户当然不希望我们下载的 APP 偷偷读取我的通信录，上传我的隐私数据，后台偷跑流量、消耗手机电量。传统的 IPC 没有任何安全措施，完全依赖上层协议来确保。首先传统的 IPC 接收方无法获得对方可靠的进程用户ID/进程ID（UID/PID），从而无法鉴别对方身份。Android 为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志。传统的 IPC 只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。其次传统的 IPC 访问接入点是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。同时 Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。
* 实名Binder---系统服务 ；匿名Binder----自己创建的

#### 2.4 Binder IPC原理

##### 2.4.1 概述

Binder跨进程通信是基于内存映射(mmap)来实现的，

* 为进程的用户和内核空间分配了一段虚拟地址，这段虚拟地址将用于binder内存的访问。binder的物理内存页由binder驱动负责分配，然后这些物理页的访问，将分为进程的用户空间和进程内核空间。由于进程的用户空间和内核空间的虚拟地址范围是不一样的，所以这里分配的一段虚拟地址将包括进程的用户空间地址和内核空间地址。
* 并且在映射建立初始阶段，只是映射了一个物理页，而不是为整个虚拟地址空间都建立相应的物理映射，目的是充分高效的利用内存，只是到需要用到的时候才建立映射，用完后马上释放映射，这样可以充分高效的利用系统的物理内存页。

* 内核刚开始只是分配了一个物理页，并且分别将这个物理页映射到进程的内核虚拟地址空间V1（修改内核空间的页表映射）和进程的用户虚拟地址空间V2（修改用户空间的页表映射）。在用户空间访问V1和在内核空间访问V2，其实都是访问的是同一个物理内存块，从而实现进程的内核和用户空间共享同一块物理内存的目的。这样binder驱动在内核空间，将一段数据拷贝到这个物理页，则该进程的用户空间则不需要copy_to_user()即可以同步看到内核空间的修改，并能够访问这段物理内存。

##### 2.4.2 总结

* 参数检查。这里要注意的Binder驱动中，通信的内存映射大小上限是4MB，超过4MB，会截断到4MB。在用户态的`ProccessStat.cpp`实现中，调用`mmap`进行地址映射的大小是`BINDER_VM_SIZE`，它的值为1M - 2 * PAGE_SIZE，PAGE_SIZE一般为4KB，所以`BINDER_VM_SIZE`是**1016KB**，且同一时间只能进行一次`mmap`。即普通APP一次Binder通信最大的数据量是1016KB，要突破这个限制，就需要先unmap在`ProcessStat`映射的内存，再自行调用mmap进行内存映射，但是这样最大也只能进行扩大到4MB。

* 分配物理内存，并将它**同时映射到内核地址空间和用户态地址空间**，即用户态地址空间和用户态地址空间都是指向同一块物理内存空间。这是Binder通信**只需要进行一次数据拷贝的精髓**所在，当一个进程将其要进行Binder传输的数据从用户态传输态拷贝到内核态地址空间时，此时数据所在物理内存也是通信对方用户态虚拟地址所映射的区域，无需再从内核态再到用户态的数据拷贝。

### 三、Binder架构

#### 3.1 Binder通信模型

##### 3.1.1 Client进程

* 跨进程通讯的客户端（运行在某个进程）
* 运行在用户空间

##### 3.1.2 Server进程

* 跨进程通讯的服务端（运行在某个进程）
* 运行在用户空间

##### 3.1.3 Binder驱动

* 跨进程通讯的介质
* 运行在内核空间

##### 3.1.4 ServiceManager

* 跨进程通讯中提供服务的注册和查询
* 运行在用户空间
* 只要是IBinder对象的注册和获取，各个进程都把IBinder对象存储到ServiceManager中，然后进程之间就可以互相拿到对方的IBinder对象。然后跨进程通讯。

##### 3.1.5 总结

* Client、Server、ServiceManager运行在用户空间；Binder 驱动运行在内核空间
* ServiceManager和Binder驱动由系统提供；而Client、Server由应用程序来实现。
* Client、Server和ServiceManager均是通过系统调用open、mmap和ioctl来访问设备文件/dev/binder，从而实现与Binder驱动的交互来间接的实现跨进程通信。

* Client、Server、ServiceManager、Binder驱动这几个组件在通信过程中扮演的角色就如同互联网中服务器（Server）、客户端（Client）、DNS域名服务器（ServiceManager）以及路由器（Binder 驱动）之前的关系。

![Binder通信模型](..\..\images\Android组件内核\跨进程通信IPC\Binder通信模型.png)

#### 3.2 通讯流程

* Server端通过Binder驱动在ServiceManager中注册
* Client端通过Binder驱动获取ServerManager中注册的Server端
* Client端通过Binder驱动和Server端进行通讯

#### 3.4 模型通讯原理

![Binder模型通讯原理](..\..\images\Android组件内核\跨进程通信IPC\Binder模型通讯原理.png)

* Service端通过Binder驱动在ServiceManager的查找表中注册Object对象的add方法
* Client端通过Binder驱动在在ServiceManager的查找表中找到Object对象的add方法，并返回proxy对象的add方法，add方法是个空实现，proxy对象也不是真正的Object对象，而是通过Binder驱动封装好的代理类的add方法
* 当Client端调用add方法时，Client端会调用proxy对象的add方法，通过Binder驱动去请求ServiceManager来找到Server端真正对象，然后调用Server端的add方法

#### 3.5 Java层的Binder

- **IBinder** : IBinder 是一个接口，代表了一种跨进程通信的能力。只要实现了这个借口，这个对象就能跨进程传输。
- **IInterface** : IInterface是一个接口， 代表的就是 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口）
- **Binder** : Java 层的 Binder 类，代表的其实就是 Binder 本地对象。BinderProxy 类是 Binder 类的一个内部类，它代表远程进程的 Binder 对象的本地代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。
- **Stub** : AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder, 说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。

* **Parcel**：是一个容器，它主要用于存储序列化数据，然后通过Binder在进程间传递这些数据

### 四、 Binder整体概述图

![](..\..\images\Android组件内核\跨进程通信IPC\Binder跨进程通信.png)

### 五、手写AIDL实例

* 因为AIDL底层使用的就是Binder，所以我们创建一个AIDL实例，来进行讲解说明
* 查看代码实例请点击[AIDL实例](https://github.com/nullWolf007/ToolProject/tree/master/AIDL%E7%9A%84demo%E7%9A%84%E6%A0%B8%E5%BF%83%E4%BB%A3%E7%A0%81/app/src/main)

#### 5.1 AIDL定义

* AIDL(Android Interface Definition Language)是一种接口定义语言，用于生成可以在Android设备上两个进程之间进行进程间通信的代码。其内部是通过Binder机制来实现的

#### 5.2 Android开发之AIDL

* Binder类和BinderProxy类都继承自IBinder，因此都具有跨进程传输的能力，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。IBinder是远程对象的基本接口，是为高性能设计的轻量级远程调用机制的核心部分，但它不仅用于远程调用，也用于进程间调用。IBinder接口定义了与远程对象交互的协议，建议不要直接实现这个接口，而应该从Binder派生。Binder实现了IBinder接口，但是一般不需要直接实现此类，而是根据你的需要由开发包中的工具生成，也就是**AIDL**，通过AIDL语言定义远程对象的方法，然后使用AIDL工具生成Binder的派生类，然后使用它。

#### 5.3 AIDL支持的数据类型

* 基本数据类型（int,long,char,boolean,double等）

* String和CharSequence

* List:只支持ArrayList，里面每个元素都必须能够被AIDL支持

* Map：只支持HashMap,里面每个元素都必须被AIDL支持，包括key和value

* Parcelable:所有实现了Parcelable接口的对象

* AIDL：所有的AIDL接口本身也可以在AIDL文件中使用；AIDL中除了基本数据类型，其他数据类型必须标上方向：in out inout ；AIDL接口只支持方法，不支持静态变量

#### 5.4 函数说明

##### 5.4.1 DESCRIPTION

* Binder的唯一标识，一般用当前Binder的类名表示

##### 5.4.2 asInterface()

* asInterface(android.os.IBinder obj)

* 将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象；如果客户端和服务端位于同一进程，返回的是服务端的Stub对象本身；如果不在同一进程，返回的是系统封装后的Stub.proxy对象

##### 5.4.3 asBinder

* 返回当前Binder对象

##### 5.4.4 onTransact()

* onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
* 运行在服务端中的Binder线程中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法进行处理；服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数，然后执行目标方法；执行完毕后，像reply中写入返回值，如果此方法返回false，则客户端的请求会失败。

#### 5.5 Person

* 基类，需要实现Parcelable，只有进行序列化才能进行传输

```java
public class Person implements Parcelable {
    private String name;

    public Person(String name) {
        this.name = name;
    }

    protected Person(Parcel in) {
        name = in.readString();
    }

    public static final Creator<Person> CREATOR = new Creator<Person>() {
        @Override
        public Person createFromParcel(Parcel in) {
            return new Person(in);
        }

        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
    }
}
```

#### 5.6 IPersonManager

* 定义接口继承IInterface，用来表示服务器端具有的功能，如下有添加人数和获取人数的功能

```java
/**
 * 这个类用来定义服务端具有什么样的能力
 */
public interface IPersonManager extends IInterface {
    /**
     * 添加人数
     *
     * @throws RemoteException
     */
    void addPerson(Person personBean) throws RemoteException;

    /**
     * 获取人数
     *
     * @throws RemoteException
     */
    List<Person> getPersons() throws RemoteException;
}
```

#### 5.7 Stub

* Stub继承自Binder，说明Stub具有跨进程通讯的能力，是一个Binder本地对象
* Stub实现了 IInterface接口，表明它具有Server承诺给Client的能力
* Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。

* asInterface(android.os.IBinder obj)方法将将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象；如果客户端和服务端位于同一进程，返回的是服务端的Stub对象本身；如果不在同一进程，返回的是系统封装后的proxy对象
* onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)方法运行在服务端中的Binder线程中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法进行处理；服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数，然后执行目标方法；执行完毕后，像reply中写入返回值，如果此方法返回false，则客户端的请求会失败。当code为TRANSAVTION_getpersons时就是getPersons的方法

```java
/**
 * 继承自binder实现了PersonManager的方法，说明它可以跨进程传输，并可进行服务端相关的数据操作
 */
public abstract class Stub extends Binder implements IPersonManager {
    private static final String DESCRIPTOR = "com.example.learn.common.IPersonManager";

    public Stub() {
        this.attachInterface(this, DESCRIPTOR);
    }

    //传入的是服务器server的binder
    public static IPersonManager asInterface(IBinder binder) {
        if (binder == null) {
            return null;
        }
        IInterface iin = binder.queryLocalInterface(DESCRIPTOR);//通过DESCRIPTOR查询本地binder，如果存在则说明调用方和service在同一进程间，直接本地调用
        if (iin != null && iin instanceof IPersonManager) {
            return (IPersonManager) iin;
        }
        return new Proxy(binder);//本地没有，返回一个远程代理对象
    }

    @Override
    public IBinder asBinder() {
        return this;
    }

    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION:
                reply.writeString(DESCRIPTOR);
                return true;
            case TRANSAVTION_getpersons:
                data.enforceInterface(DESCRIPTOR);
                List<Person> result = this.getPersons();
                reply.writeNoException();
                reply.writeTypedList(result);
                return true;
            case TRANSAVTION_addperson:
                data.enforceInterface(DESCRIPTOR);
                Person arg0 = null;
                if (data.readInt() != 0) {
                    arg0 = Person.CREATOR.createFromParcel(data);
                }
                this.addPerson(arg0);
                reply.writeNoException();
                return true;

        }
        return super.onTransact(code, data, reply, flags);
    }

    public static final int TRANSAVTION_getpersons = IBinder.FIRST_CALL_TRANSACTION;
    public static final int TRANSAVTION_addperson = IBinder.FIRST_CALL_TRANSACTION + 1;
}
```

#### 5.8 Proxy

* 远程代理类Proxy，里面主要工作时打把数据，真正进行发送使用的是transact方法
* transact调用的是server端的binder对象的transact方法，会根据code值进行判断具体是什么方法，然后调用相对应的方法，如addPerson方法。由于Stub是抽象类，并没有实现addPerson方法，所以最后会找到对应的实现类的addPerson方法。
* transact会挂起当前线程
* reply返回数据

```java
/**
 * 远程代理类
 */
public class Proxy implements IPersonManager {
    /**
     * server服务端的binder对象
     */
    IBinder remote;
    private static final String DESCRIPTOR = "com.example.learn.common.IPersonManager";

    /**
     * 构造方法传入ibinder
     *
     * @param remote
     */
    public Proxy(IBinder remote) {
        this.remote = remote;
    }

    public String getInterfaceDescriptor() {
        return DESCRIPTOR;
    }

    @Override
    public void addPerson(Person personBean) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel replay = Parcel.obtain();
        try {
            data.writeInterfaceToken(DESCRIPTOR);
            if (personBean != null) {
                data.writeInt(1);
                personBean.writeToParcel(data, 0);
            } else {
                data.writeInt(0);
            }
            //最终会调用remote即server端的ibinder的transact方法
            remote.transact(Stub.TRANSAVTION_addperson, data, replay, 0);
            replay.readException();
        } finally {
            replay.recycle();
            data.recycle();
        }
    }

    @Override
    public List<Person> getPersons() throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel replay = Parcel.obtain();
        List<Person> result;
        try {
            data.writeInterfaceToken(DESCRIPTOR);
			//最终会调用remote即server端的ibinder的transact方法
            remote.transact(Stub.TRANSAVTION_getpersons, data, replay, 0);
            replay.readException();
            result = replay.createTypedArrayList(Person.CREATOR);
        } finally {
            replay.recycle();
            data.recycle();
        }
        return result;
    }

    @Override
    public IBinder asBinder() {
        return remote;
    }
}
```

#### 5.9 RemoteService

* 对抽象类Stub进行了实现，完成了addPerson和getPersons方法，所以如果不在一个进程然后通过代理的方式最终会调用到RemoteService中的实现方法，并返回数据
* 服务器的接受运行在binder线程中
* log显示com.example.learn:remote E/RemoteService: addPerson:Thread[Binder:3197_2,5,main]

```java
public class RemoteService extends Service {
    private List<Person> persons;

    public RemoteService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        persons = new ArrayList<>();
        Log.e("RemoteService", "onBind");
        return iBinder;
    }

    private IBinder iBinder = new Stub() {
        @Override
        public void addPerson(Person personBean) throws RemoteException {
            Log.e("RemoteService", "addPerson:" + Thread.currentThread());
            persons.add(personBean);
        }

        @Override
        public List<Person> getPersons() throws RemoteException {
            return persons;
        }
    };
}
```

#### 5.10 ClientActivity

* 通过Service的方式建立连接，然后获取到了Server端的binder对象，当然了如果是同一个进程那就是Stub对象，不在同一个进程的话就是proxy代理对象。然后对其使用方法，最终就会调用到Server端的实现方法。如上解释。
* 比如不在同一个进程，然后iPersonManager就是一个代理对象proxy，iPersonManager#addPerson就会调用proxy的addPerson，接着调用Stub的transact方法，接着调用实现类服务端的addPerson。

```java
public class ClientActivity extends AppCompatActivity {

    private IPersonManager iPersonManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent intent = new Intent(this, RemoteService.class);
        intent.setAction("com.example.learn.remoteservice");
        bindService(intent, connection, Context.BIND_AUTO_CREATE);

        findViewById(R.id.main_tv_aidl).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    Log.e("ClientActivity:", "Current Thread:" + Thread.currentThread());
                    iPersonManager.addPerson(new Person("hn"));
                    List<Person> list = iPersonManager.getPersons();
                    Log.e("ClientActivity:", "list:" + list.toString() + "Current Thread:" + Thread.currentThread());
                } catch (RemoteException e) {
                    e.printStackTrace();
                }

            }
        });

    }

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e("ClientActivity:", "onServiceConnected");
            iPersonManager = Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.e("ClientActivity:", "onServiceDisconnected");
            iPersonManager = null;
        }
    };

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(connection);
    }
}
```

#### 5.11 AndroidManifest.xml

* 使用process把service放在另一个进程

```xml
<service
	android:name=".server.RemoteService"
    android:enabled="true"
    android:process=":remote"
    android:exported="true">
    	<intent-filter>
        	<action android:name="com.example.learn.remoteservice"/>
            <category android:name="android.intent.category.LAUNCHER" />
		</intent-filter>
</service>
```






​     