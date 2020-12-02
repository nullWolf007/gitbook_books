[TOC]

## Glide缓存原理之ActiveResources

#### 6.1 概述

* ActiveResources表示当前正在活动中的资源。
* Engine#load 方法中构建好 Key 之后第一件事就是去这个缓存中获取资源，获取到则直接返回，获取不到才继续从其他缓存中寻找。
* 当资源加载成功，或者通过缓存中命中资源后都会将其放入 ActivityResources 中，资源被释放时移除出 ActivityResources 。
* 由于其中的生命周期较短，所以**没有大小限制**。
* ActiveResources 中通过一个 Map 来存储数据，数据保存在一个**弱引用**（WeakReference）中。

#### 6.2 Engine#loadFromActiveResources

* 在上述的2.4中调用了Engine#loadFromActiveResources方法，这里我们作具体分析

```java
@Nullable
private EngineResource<?> loadFromActiveResources(Key key) {
	EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      	active.acquire();
    }

    return active;
}
```

* 调用activeResources.get(key)方法取获取数据的，其中activeResources是ActiveResources对象，那么我们先看一下ActiveResources的构造函数以及机制，最后看看ActiveResources#get方法以及acquire方法

#### 6.3 ActiveResources#成员变量

```java
final class ActiveResources {
    .......
	@VisibleForTesting final Map<Key, ResourceWeakReference> activeEngineResources = 
        new HashMap<>();
  	private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = 
        new ReferenceQueue<>();
    ......
}
```

* activeEngineResources对象是用来存储缓存对象的，采用了HashMap的数据结构
* 其中ResourceWeakReference继承自WeakReference

```java
@VisibleForTesting
static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
```

* 其中使用了弱引用活跃缓存机制，其中缓存的数据就存储在ResourceWeakReference对象中，下节将详细介绍

#### 6.4 弱引用+引用队列

##### 6.4.1 弱引用

* 只具有**弱引用**的对象，更容易被回收。只要`GC`发现了**弱引用**对象，就会对其进行回收，无论当前**内存空间是否充足**。
* 垃圾回收器是一个优先级很低的线程，所以弱引用对象不一定很快的就能被回收，需要一定的时间。
* **被弱引用对象关联的对象会自动被垃圾回收器回收，但是弱引用对象本身也是一个对象，这些创建的弱引用并不会自动被垃圾回收器回收掉。**
* 如需了解弱引用详情可查看[强引用、软引用、弱引用、虚引用](必备Java知识/JVM/GC/强引用、软引用、弱引用、虚引用.md)

##### 6.4.2 引用队列

* 引用队列，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到该队列中

* ReferenceQueue是一个引用队列，是在对象被回收后，会把弱引用对象，也就是WeakReference对象或者其子类的对象，放入队列ReferenceQueue中，这样我们就可以控制把弱引用对象本身给清除掉。

* Engine#构造函数

```java
if (activeResources == null) {
  	activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
}
```

* 最终会调用ActiveResources#cleanReferenceQueue

```java
@SuppressWarnings("WeakerAccess")
@Synthetic
void cleanReferenceQueue() {
	while (!isShutdown) {
    	try {
        	ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        	cleanupActiveReference(ref);

        	// This section for testing only.
        	DequeuedResourceCallback current = cb;
        	if (current != null) {
          	current.onResourceDequeued();
        	}
        	// End for testing only.
      	} catch (InterruptedException e) {
        	Thread.currentThread().interrupt();
      	}
	}
}
```

* 此方法通过resourceReferenceQueue的remove方法，取清除弱引用对象本身

#### 6.5 ActiveResources#get

* 在Engine#load方法中生成了key，然后通过key去调用此方法获取缓存数据

```java
@Nullable
synchronized EngineResource<?> get(Key key) {
	ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      	return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
      	cleanupActiveReference(activeRef);
    }
    return active;
}
```

* 调用了activeEngineResources的get方法获取缓存对象，其中activeEngineResources就是Map<Key, ResourceWeakReference>对象，也是我们的成员变量，所以activeEngineResources.get拿到的就是ResourceWeakReference对象，再通过ResourceWeakReference的get方法获取到我们存储进去的EngineResource<?>数据。此数据就是我们真正缓存的数据

#### 6.6 移除无用的弱引用对象

* 在第3部分做了部分说明，这里详细解释一下

##### 6.6.1 Engine#构造函数

* 在new Engine构造函数的时候，最终会调用如下的代码取创建ActiveResources对象

```java
if (activeResources == null) {
  	activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
}
```

##### 6.6.2 ActiveResources#构造函数

```java
ActiveResources(boolean isActiveResourceRetentionAllowed) {
	this(
    	isActiveResourceRetentionAllowed,
        java.util.concurrent.Executors.newSingleThreadExecutor(
            new ThreadFactory() {
              @Override
              public Thread newThread(@NonNull final Runnable r) {
                return new Thread(
                    new Runnable() {
                      @Override
                      public void run() {
                        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                        r.run();
                      }
                    },
                    "glide-active-resources");
              }
            }));
  }
```

* 其中调用了this方法，并且创建了一个单线程的线程池对象

```java
@VisibleForTesting
ActiveResources(
      boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
    this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
    this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

    monitorClearedResourcesExecutor.execute(
        new Runnable() {
          @Override
          public void run() {
            cleanReferenceQueue();
          }
        });
  }
```

* 最终会执行匿名Runnable对象的run方法，即cleanReferenceQueue方法

##### 6.6.3 ActiveResources#cleanReferenceQueue

```java
@Synthetic
void cleanReferenceQueue() {
    //分析1
    while (!isShutdown) {
      try {
          //分析2
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        cleanupActiveReference(ref);

        DequeuedResourceCallback current = cb;
        if (current != null) {
          current.onResourceDequeued();
        }
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
}
```

* 分析1：可以看到这是一个死循环，只要没有Shutdown，就一直循环着
* 分析2：resourceReferenceQueue.remove()在队列中没有元素时会阻塞
* 调用了cleanupActiveReference方法

##### 6.6.4 ActiveResources#cleanupActiveReference

```java
  @Synthetic
  void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
        //分析1
      activeEngineResources.remove(ref.key);

      if (!ref.isCacheable || ref.resource == null) {
        return;
      }
    }

    EngineResource<?> newResource =
        new EngineResource<>(
            ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
    listener.onResourceReleased(ref.key, newResource);
  }
```

* 分析1：调用activeEngineResources.remove方法，去删除对应的Map<Key, ResourceWeakReference>结构中无需的数据

#### 6.7 缓存对象到弱引用中

##### 6.7.1 Engine#loadFromCache

```java
  private EngineResource<?> loadFromCache(Key key) {
      //分析1
    EngineResource<?> cached = getEngineResourceFromCache(key);
      //分析2
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
  }
```

* 分析1：从MemoryCache中获取缓存数据
* 分析2：如果获取到数据了，调用acquire方法，并且调用activeResources的activate方法，把该数据存储到activeResources中去

##### 6.7.2 ActiveResources#activate

```java
  synchronized void activate(Key key, EngineResource<?> resource) {
    ResourceWeakReference toPut =
        new ResourceWeakReference(
            key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    if (removed != null) {
      removed.reset();
    }
  }
```

* 调用activeEngineResources的put方法把数据存储进去，其中activeEngineResources是Map<Key, ResourceWeakReference>对象

#### 6.8 小结

* 至此弱引用活跃缓存就基本讲解完毕了，主要就是利用了弱引用+引用队列

#### 6.9 引用计数法

##### 6.9.1 含义

* 给对象中添加一个引用计数器，每当有一个地方引用它，计数器就加1；当引用失效，计数器就减1；用对象计数器是否为0来判断对象是否可被回收。
* 详细请查看[垃圾回收GC](必备Java知识/JVM/GC/垃圾回收GC.md)

##### 6.9.2 EngineResource#acquire

* 可以看到上面很多地方都使用了此方法，那么它究竟是干嘛的呢？

```java
  synchronized void acquire() {
    if (isRecycled) {
      throw new IllegalStateException("Cannot acquire a recycled resource");
    }
    ++acquired;
  }
```

* 可以看到主要的就是对int数据acquired进行了++操作，其实这就是一个int常量，用来计数的，相对应的有一个release方法去对数据进行--操作

##### 6.9.3 EngineResource#release

```java
  void release() {
    boolean release = false;
    synchronized (this) {
      if (acquired <= 0) {
        throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
      }
      if (--acquired == 0) {
        release = true;
      }
    }
    if (release) {
      listener.onResourceReleased(key, this);
    }
  }
```

* 主要操作就是对acquired进行--操作

##### 6.9.4 小结

* 其实这里就是使用了引用计数法，其中acquired就是引用计数器。当EngineResource对象被引用的时候引用计数器就加1，当EngineResource对象引用失效时就减1。当引用计数器acquired大于0时，就说明存在引用，则说明正在使用；当等于0的时候，就说明当前没有使用，就可以释放资源了

#### 6.10 总结

* 其实最主要的就是对两种机制进行了源码解读
  * 引用计数法
  * 弱引用活跃缓存