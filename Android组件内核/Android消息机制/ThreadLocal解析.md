[TOC]
- <!-- TOC -->
- [ ThreadLocal解析](#threadlocal解析)
  - [ 参考链接](#参考链接)
  - [ 一、思维导图](#一、思维导图)
  - [ 二、原理](#二、原理)
    - [ 2.1 原理猜测](#21-原理猜测)
    - [ 2.2 实际原理](#22-实际原理)
    - [ 2.3 实际优势](#23-实际优势)
  - [ 三、源码分析(JDK1.8)](#三、源码分析jdk18)
    - [ 3.1 set方法](#31-set方法)
      - [ 3.1.1 源码](#311-源码)
      - [ 3.2 注意点](#32-注意点)
    - [ 3.2 get方法](#32-get方法)
      - [ 3.2.1 源码](#321-源码)
    - [ 3.3 remove方法](#33-remove方法)
    - [ 3.4 ThreadLocalMap对象](#34-threadlocalmap对象)
      - [ 3.4.1 ThreadLocalMap的定义](#341-threadlocalmap的定义)
      - [ 3.4.2 getEntry方法](#342-getentry方法)
      - [ 3.4.3 set方法](#343-set方法)
      - [ 3.4.4 remove方法](#344-remove方法)
  - [ 四、常见问题](#四、常见问题)
    - [ 4.1 Hash冲突](#41-hash冲突)
      - [ 4.1.1 ThreadLocalMap如何解决hash冲突的？](#411-threadlocalmap如何解决hash冲突的)
    - [ 4.2 内存泄漏](#42-内存泄漏)
      - [ 4.2.1 强引用和弱引用](#421-强引用和弱引用)
      - [ 4.2.2 ThreadLocalMa中Entry为什么要用弱引用而不是强引用？](#422-threadlocalma中entry为什么要用弱引用而不是强引用)
      - [ 4.2.3 ThreadLocalMap为什么可能会导致内存泄漏？](#423-threadlocalmap为什么可能会导致内存泄漏)
      - [ 4.2.4 如何避免内存泄漏](#424-如何避免内存泄漏)
    - [ 4.3 线程安全](#43-线程安全)
      - [ 4.3.1 ThreadLocal和Synchronized的区别](#431-threadlocal和synchronized的区别)
  - [ 五、实例](#五、实例)
    - [ 5.1 代码](#51-代码)
    - [ 5.2 说明](#52-说明)
  <!-- /TOC -->
# ThreadLocal解析

### 参考链接

* [ThreadLocal原理分析](https://juejin.im/post/5ca0d6e8e51d4563c3505505)
* [JAVA并发-自问自答学ThreadLocal](https://juejin.im/post/5a0e985df265da430e4ebb92)
* [Java面试必问，ThreadLocal终极篇](https://juejin.im/post/5a64a581f265da3e3b7aa02d)

## 一、思维导图

![ThreadLocal思维导图](..\..\images\必备Java知识\并发编程\线程\ThreadLocal思维导图.png)

## 二、原理

### 2.1 原理猜测

* 对于这种变量线程间互不干扰，我们可以比较迅速的想到，对于每一个`ThreadLocal`变量内部都维持了一个`Map`对象。利用`key:value`的形式，使用线程的ID作为`key`来操作对应线程的变量，把值放在对应的`value`中。这样就可以很简单的实现互不干扰的功能。

### 2.2 实际原理

* 上述的猜测其实是早期JDK的实现方法。
* JDK8原理：每个`Thread`维护了一个`ThreadLocalMap`哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，value就是存储的值。举例如下

```java
ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal<>();
threadLocal.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
```

* threadLocal就是实例本身，set的就是值

### 2.3 实际优势

* 实际设计的话，现有的`ThreadLocalMap`的长度会变小，因为`ThreadLocalMap`不会受线程数量的影响，这受当前线程`ThreadLocal`的数量，一般情况下，该数量并不会很大。而我们猜测的原理，会受线程数量的影响，对于大量线程的情况下，`Map`的长度会很大。
* 实际情况下，当`Thread`销毁时，内部的`ThreadLocalMap`对象也会被销毁，能够减少内存的使用

## 三、源码分析(JDK1.8)

### 3.1 set方法

#### 3.1.1 源码

```java
public void set(T value) {
    //获取当前线程
	Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    //如果该对象不为null直接set就行
    //如果该对象为null的话new一个该对象
    if (map != null)
    	map.set(this, value);
	else
    	createMap(t, value);
}

//创建一个ThreadLocalMap对象 并且把当前的ThreadLocal实例this以及firstValue存储进去
void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

* 调用的`ThreadLocalMap`的`set`方法

#### 3.2 注意点

* 上述代码中的`this`代表的是`ThreadLocal`实例本身，而不是`Thread`对象。

### 3.2 get方法

#### 3.2.1 源码

```java
public T get() {
    //获取当前线程
	Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    //如果ThreadLocalMap不为null，那就去map里面找
    if (map != null) {
        //使用ThreadLocalMap的getEntry方法去获取
    	ThreadLocalMap.Entry e = map.getEntry(this);
        //如果获取的对象不是空，那么e对应的value就是我们所需要的值
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //返回null
    return setInitialValue();
}

//默认的设置初始值为null 并返回
private T setInitialValue() {
	T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
    	map.set(this, value);
	else
    	createMap(t, value);
	return value;
}

//返回null 开发的时候也可以重写这个方法 即设置初始值
protected T initialValue() {
	return null;
}
```

* 调用的`ThreadLocalMap`的`getEntry`方法

### 3.3 remove方法

```java
public void remove() {
    //获取当前线程的ThreadLocalMap对象
	ThreadLocalMap m = getMap(Thread.currentThread());
    //如果不为null的话 remove掉
    if (m != null)
    	m.remove(this);
}

//获取当前线程的ThreadLocalMap对象
ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}
```

* 调用的`ThreadLocalMap`的`remove`方法

### 3.4 ThreadLocalMap对象

* 从上面的`set,get,remove`方法中，我们发现和`ThreadLocalMap`对象密不可分，所以我们需要了解`ThreadLocalMap`对象

#### 3.4.1 ThreadLocalMap的定义

```java
public class ThreadLocal<T> {
    static class ThreadLocalMap {
        
        //ThreadLocalMap的内部类Entry
        static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
            value = v;
        }
            
		private Entry[] table;//用来保存Entry
    }
}
```

* 从上面定义来看，数据其实也是`key:value`的形式存储起来的，只不过都放在`Entry`的内部类中，并且`Entry`的内部类的`key`必须是`ThreadLocal`对象。
* 需要注意的是`Entry`继承的是`WeakReference`，是为了尽可能的避免内存泄漏

#### 3.4.2 getEntry方法

```java
private Entry getEntry(ThreadLocal<?> key) {
    //求出index
	int i = key.threadLocalHashCode & (table.length - 1);
    //拿到Entry对象
    Entry e = table[i];
    //存在且key一致
    if (e != null && e.get() == key)
    	return e;
	else
        //不存在或key不一致，则通过调用getEntryAfterMiss继续查找
    	return getEntryAfterMiss(key, i, e);
}
```

* 可以看到是通过`key.threadLocalHashCode & (table.length - 1)`的方式来获取`index`，这和`HashMap`的方式几乎是一样的，然后数据存储在table中，通过`index`去table数组中去找到对应的`Entry`，然后就可以拿到对应的`value`。如果不是的话就得通过`getEntryAfterMiss(key, i, e)`方法去获取`Entry`。
* 这样我们很容易想到了和HashMap存在的相同的问题，即hash冲突。所以当key不一致的时候会进入`getEntryAfterMiss(key, i, e)`方法去寻找。

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
	Entry[] tab = table;
    int len = tab.length;
	
    //循环遍历所有的Entry
	while (e != null) {
        ThreadLocal<?> k = e.get();
        //找到key相等的话 返回就行
        if (k == key)
            return e;
        //k为null的话 则调用expungeStaleEntry()方法
        //删除过期的Entry，原因：弱引用
        if (k == null)
            expungeStaleEntry(i);
        //key不相等且不为null 接着往下找nextIndex
        else
            i = nextIndex(i, len);
    	e = tab[i];
	}
    //遍历完毕 找不到就返回null
    return null;
}
```

* `expungeStaleEntry`方法，删除对应位置的过期实体，并删除此位置后对应相关联位置`key = null`的实体。
* 大体流程：遍历所有的`Entry`，对`key`进行比较，寻找是否有当前的`key`的`Entry`，没有返回`null`。

#### 3.4.3 set方法

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    //通过hash的方式 来求出index(和get方法求index是一样的)
    int i = key.threadLocalHashCode & (len-1);

    //如果table[i]不为null，说明里面有数据，则说明需要覆盖或者存在hash冲突
    for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
        	ThreadLocal<?> k = e.get();
			
        	//就是已经存在的key 直接覆盖值 并返回
            if (k == key) {
            	e.value = value;
                return;
			}

        	//替换该位置key == null的实体为当前要设置的实体
            if (k == null) {
            	replaceStaleEntry(key, value, i);
                return;
			}
	}

    //如果都存在key 并且key不等于当前的key 需要放进新的
    tab[i] = new Entry(key, value);
    int sz = ++size;
	//由于弱引用带来了这个问题，所以先要清除无用数据，才能判断现在的size有没有达到阀值threshhold
    //如果没有要清除的数据，存储元素个数仍然 大于 阈值 则扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        //扩容
    	rehash();
}
```

* `replaceStaleEntry`方法：使用当前的key和value替代过期的key和value
* `cleanSomeSlots`方法：查找过期的实体并清除
* `rehash`方法：扩容
* 大体流程就是：先计算出`index`，然后对`index`进行判断。如果当前的`Entry`存在且`key`相同的话，直接覆盖就行。如果当前`Entry`存在且`key==null`的话，说明当前的`Entry`是过期的，则替换就行。如果当前的`Entry`都存在且`key!=null`且`key`不相等的话，则放入新的`Entry`。

#### 3.4.4 remove方法

```java
private void remove(ThreadLocal<?> key) {
	Entry[] tab = table;
    int len = tab.length;
    //计算出index
    int i = key.threadLocalHashCode & (len-1);
    //从index位置 开始进行遍历
    for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
        //判断key是否相等 相等的话 clear并且清除对应的实体
		if (e.get() == key) {
        	e.clear();
            expungeStaleEntry(i);
            return;
		}
	}
}
```

## 四、常见问题

### 4.1 Hash冲突

#### 4.1.1 ThreadLocalMap如何解决hash冲突的？

* `set`的时候，如果`key.threadLocalHashCode & (table.length - 1)`求出的`index`位置的Entry已经存在且`key`不同且`key`不等于`null`的时候，就说明发生了hash冲突。`ThreadLocalMap`采取的方法比较简答粗暴，就是直接循环向后寻找一个空的位置去放置，也被成为**开放寻址法**。
* `get`的时候，找到`index`，如果此`index`的`key`不同的话，也是遍历`Entry`，找到相同`key`的`Entry`，找不到的话就返回`null`。

### 4.2 内存泄漏

#### 4.2.1 强引用和弱引用

* 强引用

  ```java
  A a = new A();
  B b = new B();
  ```

* 当`a=null`和`b=null`的时候，a和b就会被GC给回收掉

* 特例

  ```java
  C c = new C(b);
  b = null;
  ```

* 此时即使`b=null`了，但是c持有对b的强引用，所以b就无法被GC回收，这样的话就会造成内存泄漏

* 解决方法：可以把`c=null`，也可以使用`WeakReference w = new WeakReference(b)`，使用弱引用。这两种情况都可以是b被GC回收。

#### 4.2.2 ThreadLocalMa中Entry为什么要用弱引用而不是强引用？

* 我们分两种情况进行讨论
* 强引用：引用的`ThreadLocal`的对象被回收了，但是`ThreadLocalMap`还持有`ThreadLocal`的强引用，如果没有手动删除，`ThreadLocal`不会被回收，导致`Entry`内存泄漏(包括`ThreadLocal`和`value`)。
* 弱引用：引用的`ThreadLocal`的对象被回收了，由于`ThreadLocalMap`持有`ThreadLocal`的弱引用，即使没有手动删除，`ThreadLocal`也会被回收。`value`在下一次`ThreadLocalMap`调用`get()`,`set()`,`remove()`的时候会被清除。
* 总结：使用弱引用可以有效的解决`ThreadLocal`的内存泄漏问题。而对应的`value`也会在`get`,`set`,`remove`的方法中进行清除，大大降低了内存泄漏的概率。

#### 4.2.3 ThreadLocalMap为什么可能会导致内存泄漏？

* 假如`ThreadLocal对象`等于`null`的时候，意味着`ThreadLocal对象`失去强引用，相应的`Entry`中对应的软引用变量就会被回收，即`key`将会被回收。此时`value`由于是强引用，所以存在引用链`Thread Reference`->`Thread`->`ThreadLocalMap`->`Entry`->`value`，就不会被回收。如果`Thread`等于`null`的时候，`value`才会被回收。
* 当`key`被回收，且由于`Thread`没被回收时，`value`也不会被回收。这个时间段就造成了内存泄漏。
* 发生常见场景：使用了线程池，此时线程不会被回收，就会导致`value`不会被回收，很容易造成内存泄漏。

#### 4.2.4 如何避免内存泄漏

* 尽管源码在`set`,`get`,`remove`的时候都会对过期的`Entry`进行清除，但是这只是尽可能的避免内存泄漏。我们使用的`ThreadLocal`的时候，应该在不使用的时候，手动调用`ThreadLocal`的`remove`方法，最终会调用`ThreadLocalMap`的`remove`方法，对过期的`Entry`进行清除，避免了内存泄漏。

### 4.3 线程安全

#### 4.3.1 ThreadLocal和Synchronized的区别

* `ThreadLocal`是一个Java类,通过对当前线程中的局部变量的操作来解决不同线程的变量访问的冲突问题。所以，`ThreadLocal`提供了线程安全的共享对象机制，每个线程都拥有其副本。
* Java中的`synchronized`是一个保留字，它依靠JVM的锁机制来实现临界区的函数或者变量的访问中的原子性。在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。此时，被用作“锁机制”的变量时多个线程共享的。

- 同步机制(`synchronized`关键字)采用了以“时间换空间”的方式，提供一份变量，让不同的线程排队访问。而`ThreadLocal`采用了“以空间换时间”的方式，为每一个线程都提供一份变量的副本，从而实现同时访问而互不影响。

## 五、实例

### 5.1 代码

```java
public class ParseDateThreadLocal {

    static ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal<>();
//    static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static class ParseDate implements Runnable {
        int i = 0;

        ParseDate(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            if (threadLocal.get() == null) {
                threadLocal.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
            }
            try {
                Date t = threadLocal.get().parse("2020-5-14 12:00:" + i % 60);
//                Date t = simpleDateFormat.parse("2020-5-14 12:00:" + i % 60);
                System.out.println(t);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(new ParseDate(i));
        }
    }
}
```

```text
Thu May 14 12:00:02 CST 2020
Thu May 14 12:00:04 CST 2020
Thu May 14 12:00:03 CST 2020
Thu May 14 12:00:05 CST 2020
Thu May 14 12:00:08 CST 2020
Thu May 14 12:00:01 CST 2020
Thu May 14 12:00:07 CST 2020
Thu May 14 12:00:06 CST 2020
Thu May 14 12:00:00 CST 2020
Thu May 14 12:00:09 CST 2020
```

### 5.2 说明

* 上述注释的代码，运行的话，会抛出` java.lang.NumberFormatException`异常。因为`SimpleDateFormat`在多线程下是不安全的。所以为了线程安全，我们使用了`ThreadLocal`来保证是安全的。