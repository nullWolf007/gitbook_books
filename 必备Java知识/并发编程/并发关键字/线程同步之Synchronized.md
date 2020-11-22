[TOC]

### 线程同步之Synchronized

#### 参考

* [java多线程同步5种方法](https://blog.csdn.net/weixin_39214481/article/details/80489586)
* [让你彻底理解Synchronized](https://www.jianshu.com/p/d53bf830fa09)
* [Synchronized原理](https://juejin.im/post/6844903670933356551#heading-3)

### 一、基础知识

#### 1.1 概念

* synchronized 是 Java 内置的同步机制，它提供了互斥的语义和可见性，当一个线程已经获取当前锁时，其他试图获取的线程只能等待或者阻塞在那里。

#### 1.2 线程不同步实例

* 如果两个人向同一个0元余额账号存钱，A向这个账号存100，然后系统计算出结果为余额100元，但是还未来得及向账户写入100元；此时B向这个账号存100，由于A的结果还未写入，所以此时余额为0加上存入的100，还是100元，然后先后往账户里写入100元。所以最终账户余额为100元，而不是理论上的200元。下面是模拟代码
* Bank.java

```java
public class Bank {

    private int count = 0;//账户余额

    //存钱
    public void addMoney(int money) {
        count += money;
        System.out.println("存进：" + money);
    }

    //取钱
    public void subMoney(int money) {
        if (count - money < 0) {
            System.out.println("余额不足");
            return;
        }
        count -= money;
        System.out.println("取出：" + money);
    }

    //查询
    public void queryMoney() {
        System.out.println("账户余额：" + count);
    }
}
```

* Test.java

```java
public class Test {
    final static Bank bank = new Bank();

    public static void main(String args[]) {
        Thread thread1 = new Thread(new MyRunnable(), "thread1");
        Thread thread2 = new Thread(new MyRunnable(), "thread2");
        thread1.start();
        thread2.start();
    }

    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            boolean flag = true;
            int i = 0;
            while (flag) {
                i++;
                if (i == 100) {
                    flag = false;
                }
                bank.addMoney(100);
                bank.queryMoney();
                System.out.println();
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

* 输出结果：输出结果极有可能小于20000，理论上是20000，由于数据不同步的问题，导致出现了举例那种情况，导致数据小于20000

```java
存进：100
账户余额：13100
```

### 二、基本使用

#### 2.1 同步代码块

##### 2.1.1 概述

* 给特定对象加上synchronized，则当前的对象就是锁对象，如果当前有其他线程正持有该对象锁，那么新到的线程就必须等待。

* 有synchronized关键字修饰的语句块。被该关键字修饰的语句块会自动被加上内置锁，从而实现同步
* 同步是一种高开销的操作，因此应该尽量减少同步的内容。通常没有必要同步整个方法，使用synchronized代码块同步关键代码即可。
* 当一个线程访问对象的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该对象的非synchronized(this)同步代码块。
* synchronized可以同步this，也可以同步对象，也可以同步类。

##### 2.1.2 使用

```java
//instance,给特定对象加锁
synchronized(instance){
	......
}

//this,当前实例对象锁
synchronized(this){
	......
}

//给特定类加锁 类锁
synchronized(ClassName.class){
	......
}
```

##### 2.1.3 实例-当前实例

* 修改Bank.java的代码

```java
public class Bank {

    private int count = 0;//账户余额

    //存钱
    public void addMoney(int money) {
        synchronized (this) {
            count += money;
        }
        System.out.println("存进：" + money);
    }

    //取钱
    public void subMoney(int money) {
        synchronized (this) {
            if (count - money < 0) {
                System.out.println("余额不足");
                return;
            }
            count -= money;
        }
        System.out.println("取出：" + money);
    }

    //查询
    public void queryMoney() {
        System.out.println("账户余额：" + count);
    }
}
```

* 输出结果正常

```java
账户余额：20000
```

##### 2.1.4 实例-特定对象

```java
public class Test {
    final static Bank bank = new Bank();

    public static void main(String args[]) {
        Thread thread1 = new Thread(new MyRunnable());
        Thread thread2 = new Thread(new MyRunnable());
        thread1.start();
        thread2.start();
    }

    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            boolean flag = true;
            int i = 0;
            while (flag) {
                i++;
                if (i == 100) {
                    flag = false;
                }
                synchronized (bank) {
                    bank.addMoney(100);
                    bank.queryMoney();
                }
                System.out.println();
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

* 输出结果正常

```java
账户余额：20000
```

#### 2.2 同步实例方法

##### 2.2.1 概述

* 有synchronized关键字修饰的方法。 由于java的每个对象都有一个内置锁，当用此关键字修饰方法时，内置锁会保护整个方法。在调用该方法前，需要获得内置锁，否则就处于阻塞状态。

##### 2.2.2 注意点

* **同步实例方法是“对象锁”，锁的是对象，对于同一个对象而言只有一把锁，多个对象有多个锁**
* **synchronized关键字**不能继承，子类重写synchronized方法必须显式地在子类的方法上加上synchronized关键字，或者使用super调用父类的方法
* 定义接口中不能使用synchronized关键字
* 构造方法中不能使用synchronized关键字，但是可以使用synchronized代码块来进行同步

##### 2.2.3 同一对象实例

* 修改Bank.java的方法，给方法加上Synchronized关键字即可

```java
public class Bank {
	......
    //存钱
    public synchronized void addMoney(int money) {
        ......
    }

    //取钱
    public synchronized void subMoney(int money) {
        ......
    }

    //查询
    public synchronized void queryMoney() {
        ......
    }
}
```

* 输出结果：这样就是正确的了

```java
存进：100
账户余额：20000
```

##### 2.2.4 不同对象实例

* 修改Test.java代码

* ```java
  public class Test {
      final static Bank bank1 = new Bank();
      final static Bank bank2 = new Bank();
  
      public static void main(String args[]) {
          Thread thread1 = new Thread(new MyRunnable(1), "thread1");
          Thread thread2 = new Thread(new MyRunnable(2), "thread2");
          thread1.start();
          thread2.start();
      }
  
      static class MyRunnable implements Runnable {
          private int differFlag = 1;
  
          public MyRunnable(int differFlag) {
              this.differFlag = differFlag;
          }
  
          @Override
          public void run() {
              boolean flag = true;
              int i = 0;
              while (flag) {
                  i++;
                  if (i == 100) {
                      flag = false;
                  }
                  if (differFlag == 1) {
                      bank1.addMoney(100);
                      bank1.queryMoney();
                  } else {
                      bank2.addMoney(100);
                      bank2.queryMoney();
                  }
                  System.out.println();
              }
          }
      }
  }
  ```

* 输出结果：会有两个“账户余额：10000”的输出，因为是不同的对象，所以就会出现这种情况

```java
账户余额：10000
```

#### 2.3 同步静态方法

##### 2.3.1 概述

* 同步静态方法和同步实例方法类似，**但是同步静态方法作用的类的所有对象，因为静态方法属于类，而不属于类的对象。相对于“类锁”，所以对于不同的对象也可以起到锁的作用**

##### 2.3.2 实例

* 在不同对象实例的代码基础上，修改Bank.java代码

```java
public class Bank {

    private static int count = 0;//账户余额

    //存钱
    public static synchronized void addMoney(int money) {
        count += money;
        System.out.println("存进：" + money);
    }

    //取钱
    public static synchronized void subMoney(int money) {
        if (count - money < 0) {
            System.out.println("余额不足");
            return;
        }
        count -= money;
        System.out.println("取出：" + money);
    }

    //查询
    public static synchronized void queryMoney() {
        System.out.println("账户余额：" + count);
    }
}
```

* 输出结果：正确的结果，因为这是针对类的锁，而不是对象。

```java
存进：100
账户余额：20000
```

### 三、底层实现原理

#### 3.1 对象锁（monitor）机制

##### 3.1.1 概述

* 什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。
* 什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。

##### 3.1.2 实例

* 现在我们来看看synchronized的具体底层实现。先写一个简单的demo:

```java
public class SynchronizedDemo {
    public static void main(String[] args) {
        synchronized (SynchronizedDemo.class) {
        }
        method();
    }

    private synchronized static void method() {
    }
}
```

* 上面的代码中有一个同步代码块，锁住的是类对象，并且还有一个同步静态方法，锁住的依然是该类的类对象。编译之后，切换到SynchronizedDemo.class的同级目录之后，然后用**javap -v SynchronizedDemo.class**查看字节码文件：

![img](https:////upload-images.jianshu.io/upload_images/2615789-10e9e5d556d5214d.png?imageMogr2/auto-orient/strip|imageView2/2/w/778/format/webp)

* 如图，上面用黄色高亮的部分就是需要注意的部分了，这也是添Synchronized关键字之后独有的。执行同步代码块后首先要先执行**monitorenter**指令，退出的时候**monitorexit**指令。通过分析之后可以看出，使用Synchronized进行同步，其关键就是必须要对对象的监视器monitor进行获取，当线程获取monitor后才能继续往下执行，否则就只能等待。而这个获取的过程是**互斥**的，即同一时刻只有一个线程能够获取到monitor。上面的demo中在执行完同步代码块之后紧接着再会去执行一个静态同步方法，而这个方法锁的对象依然就这个类对象，那么这个正在执行的线程还需要获取该锁吗？答案是不必的，从上图中就可以看出来，执行静态同步方法的时候就只有一条monitorexit指令，并没有monitorenter获取锁的指令。这就是**锁的重入性**，即在同一锁程中，线程不需要再次获取同一把锁。Synchronized先天具有重入性。**每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一**。
* 编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时执行的释放monitor 的指令。

##### 3.1.3 小结

* 任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取该对象的监视器才能进入同步块和同步方法，如果没有获取到监视器的线程将会被阻塞在同步块和同步方法的入口处，进入到BLOCKED状态（关于线程的状态可以看[这篇文章](https://www.jianshu.com/p/f65ea68a4a7f)）

##### 3.1.4 对象，对象监视器，同步队列和线程状态的关系

![img](https:////upload-images.jianshu.io/upload_images/2615789-58bf5739c7c49c05.png?imageMogr2/auto-orient/strip|imageView2/2/w/993/format/webp)

* 该图可以看出，任意线程对Object的访问，首先要获得Object的监视器，如果获取失败，该线程就进入同步状态，线程状态变为BLOCKED，当Object的监视器占有者释放后，在同步队列中得线程就会有机会重新获取该监视器。

#### 3.2 synchronized的happens-before关系

* 在上一篇文章中讨论过[happens-before](https://www.jianshu.com/p/d52fea0d6ba5)规则，抱着学以致用的原则我们现在来看一看Synchronized的happens-before规则，即监视器锁规则：对同一个监视器的解锁，happens-before于对该监视器的加锁。继续来看代码：

```java
public class MonitorDemo {
    private int a = 0;

    public synchronized void writer() {     // 1
        a++;                                // 2
    }                                       // 3

    public synchronized void reader() {    // 4
        int i = a;                         // 5
    }                                      // 6
}
```

* 该代码的happens-before关系如图所示：



![img](https:////upload-images.jianshu.io/upload_images/2615789-d025c6be230f72a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/650/format/webp)

* 在图中每一个箭头连接的两个节点就代表之间的happens-before关系，黑色的是通过程序顺序规则推导出来，红色的为监视器锁规则推导而出：**线程A释放锁happens-before线程B加锁**，蓝色的则是通过程序顺序规则和监视器锁规则推测出来happens-befor关系，通过传递性规则进一步推导的happens-before关系。现在我们来重点关注2 happens-before 5，通过这个关系我们可以得出什么？

* 根据happens-before的定义中的一条:如果A happens-before B，则A的执行结果对B可见，并且A的执行顺序先于B。线程A先对共享变量A进行加一，由2 happens-before 5关系可知线程A的执行结果对线程B可见即线程B所读取到的a的值为1。

#### 3.3 锁获取和锁释放的内存语义

* 在上一篇文章提到过JMM核心为两个部分：happens-before规则以及内存抽象模型。我们分析完Synchronized的happens-before关系后，还是不太完整的，我们接下来看看基于java内存抽象模型的Synchronized的内存语义。
* 线程A写入共享变量

![img](https:////upload-images.jianshu.io/upload_images/2615789-8faace4c9e651d6e.png?imageMogr2/auto-orient/strip|imageView2/2/w/557/format/webp)

* 从上图可以看出，线程A会首先先从主内存中读取共享变量a=0的值然后将该变量拷贝到自己的本地内存，进行加一操作后，再将该值刷新到主内存，整个过程即为线程A 加锁-->执行临界区代码-->释放锁相对应的内存语义。
* 线程B读共享变量

![img](https:////upload-images.jianshu.io/upload_images/2615789-540462b1425e38d4.png?imageMogr2/auto-orient/strip|imageView2/2/w/564/format/webp)

* 线程B获取锁的时候同样会从主内存中共享变量a的值，这个时候就是最新的值1,然后将该值拷贝到线程B的工作内存中去，释放锁的时候同样会重写到主内存中。

* 从整体上来看，线程A的执行结果（a=1）对线程B是可见的，实现原理为：释放锁的时候会将值刷新到主内存中，其他线程获取锁时会强制从主内存中获取最新的值。另外也验证了2 happens-before 5，2的执行结果对5是可见的。

* 从横向来看，这就像线程A通过主内存中的共享变量和线程B进行通信，A 告诉 B 我们俩的共享数据现在为1啦，这种线程间的通信机制正好吻合java的内存模型正好是共享内存的并发模型结构。

### 四、Synchronized的优化

* 通过上面的讨论现在我们对Synchronized应该有所印象了，它最大的特征就是在同一时刻只有一个线程能够获得对象的监视器（monitor），从而进入到同步代码块或者同步方法之中，即表现为**互斥性（排它性）**。这种方式肯定效率低下，每次只能通过一个线程，既然每次只能通过一个，这种形式不能改变的话，那么我们能不能让每次通过的速度变快一点了。打个比方，去收银台付款，之前的方式是，大家都去排队，然后去纸币付款收银员找零，有的时候付款的时候在包里拿出钱包再去拿出钱，这个过程是比较耗时的，然后，支付宝解放了大家去钱包找钱的过程，现在只需要扫描下就可以完成付款了，也省去了收银员跟你找零的时间的了。同样是需要排队，但整个付款的时间大大缩短，是不是整体的效率变高速率变快了？这种优化方式同样可以引申到锁优化上，缩短获取锁的时间，伟大的科学家们也是这样做的，令人钦佩，毕竟java是这么优秀的语言（微笑脸）。
* 在早期，synchronized属于重量级锁，效率低下。在JDK6上，实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。
* 在聊到锁的优化也就是锁的几种状态前，有两个知识点需要先关注：（1）CAS操作 （2）Java对象头，这是理解下面知识的前提条件。

#### 4.1 CAS操作

##### 4.1.1 什么是CAS?

* 使用锁时，线程获取锁是一种**悲观锁策略**，即假设每一次执行临界区代码都会产生冲突，所以当前线程获取到锁的时候同时也会阻塞其他线程获取该锁。而CAS操作（又称为无锁操作）是一种**乐观锁策略**，它假设所有线程访问共享资源的时候不会出现冲突，既然不会出现冲突自然而然就不会阻塞其他线程的操作。因此，线程就不会出现阻塞停顿的状态。那么，如果出现冲突了怎么办？无锁操作是使用**CAS(compare and swap)**又叫做比较交换来鉴别线程是否出现冲突，出现冲突就重试当前操作直到没有冲突为止。

##### 4.1.2 CAS的操作过程

* CAS比较交换的过程可以通俗的理解为CAS(V,O,N)，包含三个值分别为：**V 内存地址存放的实际值；O 预期的值（旧值）；N 更新的新值**。当V和O相同时，也就是说旧值和内存中实际的值相同表明该值没有被其他线程更改过，即该旧值O就是目前来说最新的值了，自然而然可以将新值N赋值给V。反之，V和O不相同，表明该值已经被其他线程改过了则该旧值O不是最新版本的值了，所以不能将新值N赋给V，返回V即可。当多个线程使用CAS操作一个变量是，只有一个线程会成功，并成功更新，其余会失败。失败的线程会重新尝试，当然也可以选择挂起线程

* CAS的实现需要硬件指令集的支撑，在JDK1.5后虚拟机才可以使用处理器提供的**CMPXCHG**指令实现。

##### 4.1.3 Synchronized和CAS

* 元老级的Synchronized(未优化前)最主要的问题是：在存在线程竞争的情况下会出现线程阻塞和唤醒锁带来的性能问题，因为这是一种互斥同步（阻塞同步）。而CAS并不是武断的间线程挂起，当CAS操作失败后会进行一定的尝试，而非进行耗时的挂起唤醒的操作，因此也叫做非阻塞同步。这是两者主要的区别。

##### 4.1.4 CAS的应用场景

* 在J.U.C包中利用CAS实现类有很多，可以说是支撑起整个concurrency包的实现，在Lock实现中会有CAS改变state变量，在atomic包中的实现类也几乎都是用CAS实现，关于这些具体的实现场景在之后会详细聊聊，现在有个印象就好了（微笑脸）。

##### 4.1.5 CAS的问题

**1. ABA问题**

* 因为CAS会检查旧值有没有变化，这里存在这样一个有意思的问题。比如一个旧值A变为了成B，然后再变成A，刚好在做CAS时检查发现旧值并没有变化依然为A，但是实际上的确发生了变化。解决方案可以沿袭数据库中常用的乐观锁方式，添加一个版本号可以解决。原来的变化路径A->B->A就变成了1A->2B->3C。java这么优秀的语言，当然在java 1.5后的atomic包中提供了AtomicStampedReference来解决ABA问题，解决思路就是这样的。

**2. 自旋时间过长**

* 使用CAS时非阻塞同步，也就是说不会将线程挂起，会自旋（无非就是一个死循环）进行下一次尝试，如果这里自旋时间过长对性能是很大的消耗。如果JVM能支持处理器提供的pause指令，那么在效率上会有一定的提升。

**3. 只能保证一个共享变量的原子操作**

* 当对一个共享变量执行操作时CAS能保证其原子性，如果对多个共享变量进行操作,CAS就不能保证其原子性。有一个解决方案是利用对象整合多个共享变量，即一个类中的成员变量就是这几个共享变量。然后将这个对象做CAS操作就可以保证其原子性。atomic中提供了AtomicReference来保证引用对象之间的原子性。

#### 4.2 Java对象头

* HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。
* 普通对象的对象头包括两部分：Mark Word 和 Class Metadata Address （类型指针），如果是数组对象还包括一个额外的Array length数组长度部分。

***Mark Word***：用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，占用内存大小与虚拟机位长一致。

* Mark Word存储结构

![img](https:////upload-images.jianshu.io/upload_images/2615789-668194c20734e01f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1016/format/webp)

* 如图在Mark Word会默认存放hasdcode，年龄值以及锁标志位等信息。
* Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：**无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态**，这几个状态会随着竞争情况逐渐升级。**锁可以升级但不能降级**，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。对象的MarkWord变化为下图：

![img](https:////upload-images.jianshu.io/upload_images/2615789-4556662630b15159.png?imageMogr2/auto-orient/strip|imageView2/2/w/1010/format/webp)

***Class Metadata Address***：类型指针指向对象的类元数据，虚拟机通过这个指针确定该对象是哪个类的实例。

***Array length***：数组长度

#### 4.3 偏向锁

* HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

##### 4.3.1 偏向锁的获取

* 当一个线程访问同步块并获取锁时，会在**对象头**和**栈帧中的锁记录**里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程

##### 4.3.2 偏向锁的撤销

* 偏向锁使用了一种**等到竞争出现才释放锁**的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。



![img](https:////upload-images.jianshu.io/upload_images/2615789-5b183494bd1e145d.png?imageMogr2/auto-orient/strip|imageView2/2/w/567/format/webp)

* 如图，偏向锁的撤销，需要等待**全局安全点**（在这个时间点上没有正在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word**要么**重新偏向于其他线程，**要么**恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

##### 4.3.3 小结

* 下图线程1展示了偏向锁获取的过程，线程2展示了偏向锁撤销的过程。

![img](https:////upload-images.jianshu.io/upload_images/2615789-0b954fa67e8721c2.png?imageMogr2/auto-orient/strip|imageView2/2/w/791/format/webp)

##### 4.3.4 如何关闭偏向锁

* 偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟：**-XX:BiasedLockingStartupDelay=0**。如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：**-XX:-UseBiasedLocking=false**，那么程序默认会进入轻量级锁状态

#### 4.4 轻量级锁

##### 4.4.1 加锁

* 线程在执行同步块之前，JVM会先在当前线程的栈桢中**创建用于存储锁记录的空间**，并将对象头中的Mark Word复制到锁记录中，官方称为**Displaced Mark Word**。然后线程尝试使用CAS**将对象头中的Mark Word替换为指向锁记录的指针**。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁

##### 4.4.2 解锁

* 轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。下图是两个线程同时争夺锁，导致锁膨胀的流程图。

![img](https:////upload-images.jianshu.io/upload_images/2615789-0c92d94dad8bdc27.png?imageMogr2/auto-orient/strip|imageView2/2/w/794/format/webp)

* 因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

#### 4.5 各种锁的比较

![img](https:////upload-images.jianshu.io/upload_images/2615789-56647501fd77289f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1014/format/webp)

#### 4.6 锁膨胀流程

* 偏向锁->轻量级锁->重量级锁

* 偏向所锁，轻量级锁都是乐观锁，重量级锁是悲观锁。

##### 4.6.1 偏向锁->轻量级锁

* 一个对象刚开始实例化的时候，没有任何线程来访问它的时候。它是可偏向的，意味着，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问它的时候，它会偏向这个线程，此时，对象持有偏向锁。

* 偏向第一个线程，这个线程在修改对象头成为偏向锁的时候使用CAS操作，并将对象头中的ThreadID改成自己的ID，之后再次访问这个对象时，只需要对比ID，不需要再使用CAS在进行操作。

* 一旦有第二个线程访问这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到对象是偏向状态，这时表明在这个对象上已经存在竞争了，检查原来持有该对象锁的线程是否依然存活，如果挂了，则可以将对象变为无锁状态，然后重新偏向新的线程，如果原来的线程依然存活，则马上执行那个线程的操作栈，检查该对象的使用情况，如果仍然需要持有偏向锁，则偏向锁升级为轻量级锁，（偏向锁就是这个时候升级为轻量级锁的）。如果不存在使用了，则可以将对象回复成无锁状态，然后重新偏向。

##### 4.6.2 轻量级锁->重量级锁

* 轻量级锁认为竞争存在，但是竞争的程度很轻，一般两个线程对于同一个锁的操作都会错开，或者说稍微等待一下（自旋），另一个线程就会释放锁。 但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁膨胀为重量级锁，重量级锁使除了拥有锁的线程以外的线程都阻塞，防止CPU空转。

##### 4.6.3 小结

![img](https://user-gold-cdn.xitu.io/2018/9/6/165adaeab7580a64?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 4.7 锁消除

* 为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。 如果不存在竞争，为什么还需要加锁呢？所以锁消除可以节省毫无意义的请求锁的时间。变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是对于我们程序员来说这还不清楚么？我们会在明明知道不存在数据竞争的代码块前加上同步吗？但是有时候程序并不是我们所想的那样？我们虽然没有显示使用锁，但是我们在使用一些JDK的内置API时，如StringBuffer、Vector、HashTable等，这个时候会存在隐形的加锁操作。比如StringBuffer的append()方法，Vector的add()方法。

