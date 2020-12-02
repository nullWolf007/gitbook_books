[TOC]

## Glide缓存原理之LruCache

#### 7.1 MemoryCache

```java
public interface MemoryCache {
```

* MemoryCache这是一个接口，真正使用的应该是它的实现类

#### 7.2 LruResourceCache

* MemoryCache的实现类

```java
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
```

* 可以看到他同时继承了LruCache类，其实这个类就是我们这节重点要将的LruCache机制

#### 7.3 LruCache#整体思路

* Lru是Least Recently Used的缩写，即最近最少使用，是一种常用的[页面置换算法](https://baike.baidu.com/item/页面置换算法/7626091)，选择最近最久未使用的页面予以淘汰。
* 通过LinkedHashMap数据结构存储，往里面put数据，每次put都是放入队尾，当超过size的时候，则从队首删除数据，因为队首数据是最早放入的，所以是最老的数据，予以删除。以实现最近最少使用机制。
* 有关LinkedHashMap详细信息可以查看[LinkedHashMap解析](数据结构及算法/数据结构/Hash/LinkedHashMap解析.md)

#### 7.3 LruCache#成员变量

```java
public class LruCache<T, Y> {
  private final Map<T, Y> cache = new LinkedHashMap<>(100, 0.75f, true);
  private final long initialMaxSize;
  private long maxSize;
  private long currentSize;
	......
}
```

* LruCache中采用了LinkedHashMap的数据结构来存储数据，与HashMap相比LinkedHashMap维护的是一个具有双重链表的HashMap，使其有序。有关HashMap详细信息请点击[HashMap解析](数据结构及算法/数据结构/Hash/HashMap解析.md)。
* new LinkedHashMap<>(100, 0.75f, true)
  * 100表示初始容量100
  * 0.75f表示碰撞因子0.75（即真实容量到达总容量的75%就开始扩容）
  * true：链表顺序按访问顺序
* initialMaxSize：初始大小
* maxSize：最大的大小
* currentSize：当前的大小

#### 7.3 LruCache#构造函数

```java
public LruCache(long size) {
    this.initialMaxSize = size;
    this.maxSize = size;
}
```

* 可以看到创建时initialMaxSize和maxSize相等

#### 7.4 LruCache#setSizeMultiplier

```java
public synchronized void setSizeMultiplier(float multiplier) {
    if (multiplier < 0) {
      throw new IllegalArgumentException("Multiplier must be >= 0");
    }
    maxSize = Math.round(initialMaxSize * multiplier);
    evict();
}
```

* 通过setSizeMultiplier改变maxSize的大小，其中multiplier为变化乘数，initialMaxSize * multiplier就是最新的maxSize
* 调用evict

#### 7.5 LruCache#evict

```java
private void evict() {
    trimToSize(maxSize);
}
```

* 调用trimToSize

#### 7.6 LruCache#trimToSize

```java
  protected synchronized void trimToSize(long size) {
    Map.Entry<T, Y> last;
    Iterator<Map.Entry<T, Y>> cacheIterator;
      //分析1
    while (currentSize > size) {
        //分析2
      cacheIterator = cache.entrySet().iterator();
      last = cacheIterator.next();
      final Y toRemove = last.getValue();
      currentSize -= getSize(toRemove);
      final T key = last.getKey();
      cacheIterator.remove();
        //用于重写的保留方法
      onItemEvicted(key, toRemove);
    }
  }
```

* 分析1：当currentSize大于size就一直循环
* 分析2：通过Iterator进行获取最后的一个元素，修改currentSize，并把其remove掉

* 就是把数据清除出缓存，把容量缩减到小于等于maxSize。

#### 7.7 LruCache#put

* 这个方法时重点

```java
  @Nullable
  public synchronized Y put(@NonNull T key, @Nullable Y item) {
      //分析1
    final int itemSize = getSize(item);
      //分析2
    if (itemSize >= maxSize) {
        //用于重写的保留方法
      onItemEvicted(key, item);
      return null;
    }
	//分析3
    if (item != null) {
      currentSize += itemSize;
    }
      //分析4
    @Nullable final Y old = cache.put(key, item);
      //分析5
    if (old != null) {
      currentSize -= getSize(old);

      if (!old.equals(item)) {
          //用于重写的保留方法
        onItemEvicted(key, old);
      }
    }
      //分析6
    evict();

    return old;
  }
```

* 分析1：首先获取item的size，如果没有重写，默认返回1
* 分析2：如果itemSize大于等于maxSize，那说明肯定放不进去，则就return null
* 分析3：如果item不等于null，则说明需要放入，则需要修改其currentSize
* 分析4：返回此key的旧值value，如果没有此key的话，返回null
* 分析5：如果old不等于null，说明已经存在了此值，则把currentSize值恢复原样，如果旧值old不等于新值item
* 分析6：清除最老的数据，就是在Map中最后的数据，使其满足currentSize <= maxSize

#### 7.8 LruCache#get

```java
  @Nullable
  public synchronized Y get(@NonNull T key) {
    return cache.get(key);
  }
```

* 调用LinkedHashMap#get获取的

#### 7.9 LruCache#remove

```java
  @Nullable
  public synchronized Y remove(@NonNull T key) {
    final Y value = cache.remove(key);
    if (value != null) {
      currentSize -= getSize(value);
    }
    return value;
  }
```

* 使用从LinkedHashMap中移除对应的数据，然后修改currentSize的值。这个流程比较简单

#### 7.10 如果put存在同样的key？

```java
private final Map<T, Y> cache = new LinkedHashMap<>(100, 0.75f, true);
```

* LinkedHashMap提供两种排序方式
  * 按照访问的次序来排序的含义：当调用LinkedHashMap的get(key)或者put(key, value)时，碰巧key在map中被包含，那么LinkedHashMap会将key对象的entry放在线性结构的最后。 
  * 按照插入顺序来排序的含义：调用get(key), 或者put(key, value)并不会对线性结构产生任何的影响
* 对于LruCache采用的是true，表示的意思是访问排序，所以当put的时候即使存在相同的key值，也会把其数据放在最后（最后表示最近的数据），
* 其实现在LinkedHashMap的afterNodeAccess方法中，详情请查看[LinkedHashMap解析](数据结构及算法/数据结构/Hash/LinkedHashMap解析.md)