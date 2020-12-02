[TOC]

## Glide缓存原理之BitmapPool

#### 参考转载

* [Glide4.8源码拆解（五）BitmapPool从入门到放弃](https://juejin.cn/post/6844903761698095118#heading-0)

* [Glide 源码分析解读-缓存模块-基于最新版Glide 4.9.0](https://zhuanlan.zhihu.com/p/60426316)

### 一、Bitmap内存管理和复用

* 参考[Managing Bitmap Memory](https://developer.android.com/topic/performance/graphics/manage-memory)文档：
* [参考官方文档](https://developer.android.com/reference/android/graphics/BitmapFactory.Options#inBitmap)

#### 1.1 Android对Bitmap的内存管理

* 在Android2.2或更低的版本中，当发生垃圾回收时，线程都会暂停执行。这会导致延迟，降低程序性能。Android2.3增加了并发的垃圾回收机制，这意味着当图片对象不再被引用时所占用的内存空间立即就会被回收。

* 在Android2.3.3或更低版本中，Bitmap的像素数据是存储在Native内存中，它和Bitmap对象本身是分开来的，Bitmap对象本身是存储在Java虚拟机堆内存中。存储在本地内存中的像素数据的释放是不可预知的，这样就有可能导致应用短暂的超过其内存限制而崩溃。Android3.0之后，Bitmap像素数据和它本身都存储在Java虚拟机的堆内存中。到了Android8.0及其以后，Bitmap有重新存储到Native堆中。

* 在Android2.3.3或更低版本中，调用`recycle()`方法来尝试对Bitmap进行回收；

* 在Android3.0或者更高的版本中，使用`BitmapFactory.Options.inBitmap`来尝试对Bitmap进行复用，但是复用Bitmap是有条件限制的；

##### 复用Bitmap的限制

- 首先Bitmap必须是mutable，可以通过`BitmapFactory.Options`设置;
- 在Android4.4版本之前，仅支持jpg、png图片，且size大小一样，且 `inSampleSize=1`的Bitmap才可以复用，inPreferredConfig需要设置成与目标Config一致；
- 在Android4.4之后的版本，只要内存大小不小于需求的Bitmap都可以复用.
- Bitmap.Config.HARDWARE不需要复用；

### 二、Glide中的BitmapPool

#### 2.1 概述

* `BitmapPool`是Glide中对Bitmap复用进行统一管理的接口，原则上所有需要创建Bitmap的操作，都要经过它来进行获取。

#### 2.2 BitmapPool的类关系

- `BitmapPool`是一个接口，实现类有`BitmapPoolAdapter`和`LruBitmapPool`这两个;
- `BitmapPoolAdapter`是一个空壳子，根本没有做实际意义上的缓存操作；
- `LruBitmapPool`采用**策略模式**，它自身不处理具体逻辑，真正的逻辑在`LruPoolStrategy`中；

### 三、LruBitmapPool

#### 3.1 概述

* LruBitmapPool是策略的执行者，也是缓存大小的控制者；

#### 3.2 源码解析

* LruBitmapPool.java

```java
public class LruBitmapPool implements BitmapPool{
  //构造方法
  public LruBitmapPool(long maxSize) {
    this(maxSize, getDefaultStrategy(), getDefaultAllowedConfigs());
  }
 //构造方法
  LruBitmapPool(long maxSize, LruPoolStrategy strategy, Set<Bitmap.Config> allowedConfigs) {
    this.initialMaxSize = maxSize;
    this.maxSize = maxSize;
    this.strategy = strategy;
    this.allowedConfigs = allowedConfigs;
    this.tracker = new NullBitmapTracker();
  } 
  //put方法
  public synchronized void put(Bitmap bitmap) {
    if (bitmap == null) {
      throw new NullPointerException("Bitmap must not be null");
    }
    if (bitmap.isRecycled()) {
      throw new IllegalStateException("Cannot pool recycled bitmap");
    }
    //如果bitmap不是Mutable或者bitmap尺寸大于最大尺寸，或者不允许缓存
    if (!bitmap.isMutable() || strategy.getSize(bitmap) > maxSize
        || !allowedConfigs.contains(bitmap.getConfig())) {
        //直接回收bitmap
      bitmap.recycle();
      return;
    }
    //通过策略获取bitmap size
    final int size = strategy.getSize(bitmap);
    strategy.put(bitmap);//添加进lruCache
    tracker.add(bitmap);
    //计算put数量和currentSize
    puts++;
    currentSize += size;
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Put bitmap in pool=" + strategy.logBitmap(bitmap));
    }
    dump();
    evict();//可能会触发淘汰
  }
}
//获取能够缓存的config
private static Set<Bitmap.Config> getDefaultAllowedConfigs() {
    //获取所有config
    Set<Bitmap.Config> configs = new HashSet<>(Arrays.asList(Bitmap.Config.values()));
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      configs.add(null);
    }
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        //Bitmap.Config.HARDWARE是8.0之后才有的，不允许缓存
      configs.remove(Bitmap.Config.HARDWARE);
    }
    return Collections.unmodifiableSet(configs);
  }

//获取脏数据，可能返回空
private synchronized Bitmap getDirtyOrNull(
      int width, int height, @Nullable Bitmap.Config config) {
    assertNotHardwareConfig(config);
    //从策略中获取
    final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
    if (result == null) {
      //没有获取到，misses数量+1
      misses++;
    } else {
      hits++;//命中+1
      currentSize -= strategy.getSize(result);//currentSize减少
      tracker.remove(result);
      normalize(result);
    }
    dump();
    return result;
  }
//将缓存整理到size大小以内
private synchronized void trimToSize(long size) {
    while (currentSize > size) {//判断条件
      final Bitmap removed = strategy.removeLast();//调用策略的方法
      if (removed == null) {
        if (Log.isLoggable(TAG, Log.WARN)) {
          Log.w(TAG, "Size mismatch, resetting");
          dumpUnchecked();
        }
        currentSize = 0;
        return;
      }
      tracker.remove(removed);
      currentSize -= strategy.getSize(removed);
      evictions++;
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Evicting bitmap=" + strategy.logBitmap(removed));
      }
      dump();
      removed.recycle();//被淘汰的bitmap执行回收
    }
    
    //获取擦除掉像素的Bitmap
    public Bitmap get(int width, int height, Bitmap.Config config) {
    Bitmap result = getDirtyOrNull(width, height, config);
    if (result != null) {
      result.eraseColor(Color.TRANSPARENT);
    } else {
      result = createBitmap(width, height, config);
    }
    return result;
  }
    
    //直接获取携带脏数据Bitmap
    public Bitmap getDirty(int width, int height, Bitmap.Config config) {
    Bitmap result = getDirtyOrNull(width, height, config);
    if (result == null) {
      result = createBitmap(width, height, config);
    }
    return result;
  }
  }
```

* `LruBitmapPool`主要方法：

* `构造方法`：需要传入`maxSize`，这个是控制缓存大小的必要参数；

* `put()`：将Bitmap进行缓存，如果不满足缓存条件，执行回收；

* `getDirtyOrNull()`：从缓存中获取Bitmap，可能返回空；

* `trimToSize()`：对内存重新整理，防止超出目标size；

* `get()`：获取一个全透明像素的bitmap，不为空；

* `getDirty()`：直接获取，如果取自缓存，可能包含脏数据，不为空；

* 其中操作缓存的核心方法在`strategy`中，`LruPoolStrategy`也是一个抽象的策略接口，真正策略的实现类是`SizeConfigStrategy`和`AttributeStrategy`;

### 四、LruPoolStrategy

#### 4.1 源码解析

* LruPoolStrategy.java

```java
interface LruPoolStrategy {
  //put操作
  void put(Bitmap bitmap);
  
  //get操作
  @Nullable
  Bitmap get(int width, int height, Bitmap.Config config);

  @Nullable
  Bitmap removeLast();

  String logBitmap(Bitmap bitmap);

  String logBitmap(int width, int height, Bitmap.Config config);a
 
  int getSize(Bitmap bitmap);
}
```

#### 4.2 LruBitmapPool#getDefaultStrategy

* 在LruBitmapPool中会通过getDefaultStrategy方法来获取LruPoolStrategy实例对象
* 通过调用`getDefaultStrategy()`方法获得LruPoolStrategy实例：

```java
private static LruPoolStrategy getDefaultStrategy() {
    final LruPoolStrategy strategy;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      strategy = new SizeConfigStrategy();
    } else {
      strategy = new AttributeStrategy();
    }
    return strategy;
  }
```

* `SizeConfigStrategy`是针对Android4.4及其以上版本的策略，`AttributeStrategy`则是低版本的策略；

#### 4.3 低版本#AttributeStrategy

* AttributeStrategy.java

```java
class AttributeStrategy implements LruPoolStrategy {
  private final KeyPool keyPool = new KeyPool();//KeyPool
  private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();//真正的LRU CACHE
  @Override
  public void put(Bitmap bitmap) {
    //获取Key
    final Key key = keyPool.get(bitmap.getWidth(), bitmap.getHeight(), bitmap.getConfig());
    //保存
    groupedMap.put(key, bitmap);
  }

  @Override
  public Bitmap get(int width, int height, Bitmap.Config config) {
    //获取Key
    final Key key = keyPool.get(width, height, config);
    //取出
    return groupedMap.get(key);
  }
  //移除末尾的元素
  @Override
  public Bitmap removeLast() {
    return groupedMap.removeLast();
  }
 }
```

* `AttributeStrategy`重写接口定义的方法，其中`GroupedLinkedMap`是真正实现Lru逻辑的集合，值得注意的是`Key`的获取在`KeyPool`中，`Key`作为对入参`int width, int height, Bitmap.Config config`的封装，也是Lru缓存的键；

* AttributeStrategy.Key重写`equals()`方法和`hashCode()`方法，其中`hashCode()`是用来识别Lru内部`LinkedHashMap`中的`bucket`，`equal()`是真正的对比;

#### 4.4 AttributeStrategy.Key

```java
{
    @Override
    public boolean equals(Object o) {
      if (o instanceof Key) {
        Key other = (Key) o;
        //判断相等的条件是width、heigth一一对应相等且config相等
        return width == other.width && height == other.height && config == other.config;
      }
      return false;
    }

    @Override
    public int hashCode() {
      //hashCode的生成也是根据width、height、config.hashCode()
      int result = width;
      result = 31 * result + height;
      result = 31 * result + (config != null ? config.hashCode() : 0);
      return result;
    }
}
```

* `AttributeStrategy`这个策略的核心目的，就是在缓存的`Key`上面做对比，只有缓存中的Bitmap同时满足`width`、`height`、`config`相等才能命中；

#### 4.5 SizeConfigStrategy

* 上面说了`AttributeStrategy`是面向低于Android4.4版本的Bitmap缓存策略，`SizeConfigStrategy`则是面向高版本的，从文章开头的部分我们知道，高版本的`inBitmap`限制没有这么严格，至少在尺寸这一块是放开了，只有内存大小不小于需求就行；下面看看代码怎么实现的：

* SizeConfigStrategy.java

```java
public class SizeConfigStrategy implements LruPoolStrategy {

  private static final int MAX_SIZE_MULTIPLE = 8;
  private final KeyPool keyPool = new KeyPool();
  private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();
  private final Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes = new HashMap<>();

  @Override
  public void put(Bitmap bitmap) {
    //获取bitmap的像素占用字节数
    int size = Util.getBitmapByteSize(bitmap);
    //获取key
    Key key = keyPool.get(size, bitmap.getConfig());
    //保存到LRU
    groupedMap.put(key, bitmap);
    //获取sizeConfig Map
    NavigableMap<Integer, Integer> sizes = getSizesForConfig(bitmap.getConfig());
    //保存键值对，键是字节数大小，值是总共有多少个
    Integer current = sizes.get(key.size);
    sizes.put(key.size, current == null ? 1 : current + 1);
  }
  
  public Bitmap get(int width, int height, Bitmap.Config config) {
    //获取字节数
    int size = Util.getBitmapByteSize(width, height, config);
    //获取最优的key
    Key bestKey = findBestKey(size, config);
    //从LRU中获取
    Bitmap result = groupedMap.get(bestKey);
    if (result != null) {
      //操作sizeConfig集合，做减1操作或者移除
      decrementBitmapOfSize(bestKey.size, result);
      //重新计算Bitmap宽高和config
      result.reconfigure(width, height,
          result.getConfig() != null ? result.getConfig() : Bitmap.Config.ARGB_8888);
    }
    return result;
  }
  //获取SizesForConfig Map
  private NavigableMap<Integer, Integer> getSizesForConfig(Bitmap.Config config) {
    NavigableMap<Integer, Integer> sizes = sortedSizes.get(config);
    if (sizes == null) {
      //treeMap
      sizes = new TreeMap<>();
      sortedSizes.put(config, sizes);
    }
    return sizes;
  }
  
  //Key的组成
   static final class Key{
    @Override
    public boolean equals(Object o) {
      if (o instanceof Key) {
        Key other = (Key) o;
        return size == other.size
            && Util.bothNullOrEqual(config, other.config);
      }
      return false;
    }

    @Override
    public int hashCode() {
      int result = size;
      result = 31 * result + (config != null ? config.hashCode() : 0);
      return result;
    }
  }
  }
}
```

* `SizeConfigStrategy`和`AttributeStrategy`有很多相似之处，但是复杂的多，相同的是都是用`GroupedLinkedMap`作为Lru存储，不同之处是对于`Key`的获取以及多出一个辅助集合`NavigableMap`；`Key`的获取已经不依赖`Width`和`Height`了，而是`size`，它是Bitmap占用的字节数，`Key`的`hashCode()`和`equals()`依赖的是`size`和`config`;

* `SizeConfigStrategy`最关键的方法是`findBestKey`，它的作用是获取最合适的Key；

#### 4.6 SizeConfigStrategy.findBestKey()

```java
//获取最适合的Key
private Key findBestKey(int size, Bitmap.Config config) {
    //从pool里取出，肯定不为空
    Key result = keyPool.get(size, config);
    //获取匹配的Config,一般只有一个匹配
    for (Bitmap.Config possibleConfig : getInConfigs(config)) {
    //获取sizesForConfig
      NavigableMap<Integer, Integer> sizesForPossibleConfig = getSizesForConfig(possibleConfig);
      //获取不比size小的可能缓存的size，ceiling方法相当于是数学上的进一法
      Integer possibleSize = sizesForPossibleConfig.ceilingKey(size);
      //命中的size不能大于目标size的8倍，可能是担心浪费内存；
      if (possibleSize != null && possibleSize <= size * MAX_SIZE_MULTIPLE) {
        //`size`不相等或者`config`不相等，此处的判断等于是判断了`!Key.equals()`逻辑，这时候才降低维度获取相近的key
        if (possibleSize != size
            || (possibleConfig == null ? config != null : !possibleConfig.equals(config))) {
          //接受相近的缓存key，第一步创建的key放入队列
          keyPool.offer(result);
          //命中的key，他的size和目标相近但是肯定不完全一样
          result = keyPool.get(possibleSize, possibleConfig);
        }
        break;
      }
    }
    return result;
  }
  //获取能匹配上的config
  private static Bitmap.Config[] getInConfigs(Bitmap.Config requested) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      if (Bitmap.Config.RGBA_F16.equals(requested)) {
        return RGBA_F16_IN_CONFIGS;
      }
    }
    switch (requested) {
      case ARGB_8888:
        return ARGB_8888_IN_CONFIGS;
      case RGB_565:
        return RGB_565_IN_CONFIGS;
      case ARGB_4444:
        return ARGB_4444_IN_CONFIGS;
      case ALPHA_8:
        return ALPHA_8_IN_CONFIGS;
      default:
        return new Bitmap.Config[] { requested };
    }
  }
  
  //8888能匹配8888，大于等于Android O 能匹配RGBA_F16
  private static final Bitmap.Config[] ARGB_8888_IN_CONFIGS;
  static {
    Bitmap.Config[] result =
        new Bitmap.Config[] {
            Bitmap.Config.ARGB_8888,
            null,
        };
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      result = Arrays.copyOf(result, result.length + 1);
      result[result.length - 1] = Config.RGBA_F16;
    }
    ARGB_8888_IN_CONFIGS = result;
  }
  //RGBA_F16_IN_CONFIGS和ARGB_8888_IN_CONFIGS一样
  private static final Bitmap.Config[] RGBA_F16_IN_CONFIGS = ARGB_8888_IN_CONFIGS;
  //565匹配565
  private static final Bitmap.Config[] RGB_565_IN_CONFIGS =
      new Bitmap.Config[] { Bitmap.Config.RGB_565 };
  //4444匹配4444
  private static final Bitmap.Config[] ARGB_4444_IN_CONFIGS =
      new Bitmap.Config[] { Bitmap.Config.ARGB_4444 };
  //ALPHA_8匹配ALPHA_8
  private static final Bitmap.Config[] ALPHA_8_IN_CONFIGS =
      new Bitmap.Config[] { Bitmap.Config.ALPHA_8 };
```

* `getBestKey()`主要是通过`getInConfigs()`拿到能匹配到的`sizesForPossibleConfig`，通过辅助集合`NavigableMap`拿到size相近的`possibleSize`；
  * 能匹配的第一个条件是possibleSize要小于等于`size * MAX_SIZE_MULTIPLE`,`MAX_SIZE_MULTIPLE`默认是8；如果大于8对内存的利用率很低，没有必要强制匹配缓存；
  * 如果`sizesForPossibleConfig`和`possibleSize`有一个不和目标相等，就可以复用，否则说明两者的key肯定相等(参考`Key.equals()`方法)，两者相等没有必须再进行经纬度的匹配，直接返回就行；

#### 4.7 辅助Map

* 在回头看看这个`NavigableMap`，通过调用`getSizesForConfig()`得到一个`TreeMap`，这个Map保存了每个缓存的Bitmap的size和相同size的count，在`getBestKey()`方法中调用`ceilingKey(size)`方法，`TreeMap`默认会对key进行自然排序，`ceilingKey(size)`函数的意义是返回一个和size最接近的不小于size的key，正好符合内存复用的价值；

* 疑问：为啥要用Map来保存size和该size对应的count，count有何用？

* SizeConfigStrategy中有这么一个方法：`decrementBitmapOfSize()`；

```java
private void decrementBitmapOfSize(Integer size, Bitmap removed) {
    Bitmap.Config config = removed.getConfig();
    NavigableMap<Integer, Integer> sizes = getSizesForConfig(config);
    Integer current = sizes.get(size);
    if (current == null) {
    }
    if (current == 1) {
      //移除掉该条数据
      sizes.remove(size);
    } else {
      //减1
      sizes.put(size, current - 1);
    }
  }
```

* 该方法调用时机是当`Bitmap`从是缓存池中取出或者移除时，执行内容：操作该map，被移除的Bitmap对应的size减1或者把当前key移除，只有移除掉，在`getBestKey()`调用`ceilingKey(size)`时才知道该size在缓存中是否存在；

### 五、GroupedLinkedMap

#### 5.1 概述

* BitmapPool真正实现LruCache功能的是`GroupedLinkedMap`，这个类的功能跟`LinkedHashMap`很相似但又不同，相同的是都是利用链表来记住数据访问顺序，不同的是该类把相同key的value保存到一个数组中；

#### 5.2 源码分析

```java
class GroupedLinkedMap<K extends Poolable, V> {
  //LinkedEntry是存入Map的节点，同时是一个双向链表，同时还是持有一个数组
  private static class LinkedEntry<K, V> {
    @Synthetic final K key;//key
    private List<V> values;//value数组
    LinkedEntry<K, V> next;//链表下一个节点
    LinkedEntry<K, V> prev;//链表上一个节点 
    //构造
    LinkedEntry() {
      this(null);
    }
     //构造
    LinkedEntry(K key) {
      next = prev = this;
      this.key = key;
    }
    //移除数组的最后一个元素
    @Nullable
    public V removeLast() {
      final int valueSize = size();
      return valueSize > 0 ? values.remove(valueSize - 1) : null;
    }
    //数组的长度
    public int size() {
      return values != null ? values.size() : 0;
    }
    //添加到数组
    public void add(V value) {
      if (values == null) {
        values = new ArrayList<>();
      }
      values.add(value);
    }
  }

  //头节点
  private final LinkedEntry<K, V> head = new LinkedEntry<>();
  
  //存储key和entry的HashMap
  private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<>();
  
  //放入
  public void put(K key, V value) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
      //创建结点
      entry = new LinkedEntry<>(key);
      //放到链表尾部
      makeTail(entry);  
       //放到hashMap中
      keyToEntry.put(key, entry);
    } else {
      key.offer();//keyPool的操作
    }
    //放入entry数组中
    entry.add(value);
  }
  //获取操作
  public V get(K key) {
    //从HashMap中查找
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
    //如果不存在，创建结点，放到hashMap中
      entry = new LinkedEntry<>(key);
      keyToEntry.put(key, entry);
    } else {
      key.offer();//keyPool的操作
    }
    //放到链表头部
    makeHead(entry);
    return entry.removeLast();//返回数组的最后一个
  }
  //设成链表头(其实就是head的下一个)
  private void makeHead(LinkedEntry<K, V> entry) {
    removeEntry(entry);
    entry.prev = head;
    entry.next = head.next;
    updateEntry(entry);
  }
  //设成链表尾(其实就是head的上一个)
  private void makeTail(LinkedEntry<K, V> entry) {
    //把自己从链表中移除
    removeEntry(entry);
    //绑定自身的关系
    entry.prev = head.prev;
    entry.next = head;
    //绑定自身前后节点与自己的关系
    updateEntry(entry);
  }
  //更新节点，把当前节点的上一个的next指向自己，下一个的perv指向自己，完成双向链表
  private static <K, V> void updateEntry(LinkedEntry<K, V> entry) {
    entry.next.prev = entry;
    entry.prev.next = entry;
  }
  //删除当前节点，把自己上一个的next指向下一个，把自己下一个的prev指向上一个
  private static <K, V> void removeEntry(LinkedEntry<K, V> entry) {
    entry.prev.next = entry.next;
    entry.next.prev = entry.prev;
  }
  
  //移除队尾的元素
  public V removeLast() {
    //获取队尾节点
    LinkedEntry<K, V> last = head.prev;
    //这一块的whild循环有意思
    while (!last.equals(head)) {
      //移除改节点数组的最后一个
      V removed = last.removeLast();
      if (removed != null) {//如果不为空直接返回
        return removed;
      } else {
        //如果走到这里，说明last节点底下的数组为空，所以根本没有移除掉数据，第一件事就是干掉这个节点
        removeEntry(last);
        keyToEntry.remove(last.key);
        last.key.offer();
      }
      //走到这一步还是因为last节点底下的数组为空，继续探寻它的上一个节点，直到能return出去为止
      last = last.prev;
    }
    return null;
  }
}
```

* `GroupedLinkedMap`的代码量并不大，我在代码里做了比较详细的注释，如果有解释不当之处，还请留言交流；

### 六、BitmapPool缓存大小的计算

#### 6.1 概述

* 首先，`BitmapPool`相对Glide对象是单例，在`GlideBuilder.build()`中创建，构造方法中需要传`maxSize`，`maxSize`的计算规则是从`MemorySizeCalculator.getBitmapPoolSize()`获得；

#### 6.2 GlideBuilder

```java
//BitmapPool的创建
 if (bitmapPool == null) {
    //通过memorySizeCalculator获取bitmapPoolSize
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }
```

* 通过memorySizeCalculator获取`size`, 如果`size`等于0时，创建`BitmapPoolAdapter`，否则创建`LruBitmapPool`，什么时候情况下`size`等于0？我们还是看一些`memorySizeCalculator`的定义；

#### 6.3 MemorySizeCalculator

```java
//构造方法，Builder模式
MemorySizeCalculator(MemorySizeCalculator.Builder builder) {
    this.context = builder.context;
    //得到arrayPoolSize
    arrayPoolSize =
        isLowMemoryDevice(builder.activityManager)
            ? builder.arrayPoolSizeBytes / LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR
            : builder.arrayPoolSizeBytes;
    //最大总共内存缓存size
    int maxSize =
        getMaxSize(
            builder.activityManager, builder.maxSizeMultiplier, builder.lowMemoryMaxSizeMultiplier);
    //屏幕宽度
    int widthPixels = builder.screenDimensions.getWidthPixels();
     //屏幕高度
    int heightPixels = builder.screenDimensions.getHeightPixels();
     //屏幕像素数，一个像素按照4字节算
    int screenSize = widthPixels * heightPixels * BYTES_PER_ARGB_8888_PIXEL;
    //目标bitmap池缓存Size
    int targetBitmapPoolSize = Math.round(screenSize * builder.bitmapPoolScreens);
    //目标内存缓存size
    int targetMemoryCacheSize = Math.round(screenSize * builder.memoryCacheScreens);
    //可用内存size
    int availableSize = maxSize - arrayPoolSize;
    //如果算出来的size相加小于可用内存，直接赋值
    if (targetMemoryCacheSize + targetBitmapPoolSize <= availableSize) {
      memoryCacheSize = targetMemoryCacheSize;
      bitmapPoolSize = targetBitmapPoolSize;
    } else {
        //按比例重新分配memoryCacheSize和bitmapPoolSize
      float part = availableSize / (builder.bitmapPoolScreens + builder.memoryCacheScreens);
      memoryCacheSize = Math.round(part * builder.memoryCacheScreens);
      bitmapPoolSize = Math.round(part * builder.bitmapPoolScreens);
    }
  }
```

* 首先，`MemorySizeCalculator`是Builder模式，主要的参数是在`MemorySizeCalculator.Builder`中生成，在`MemorySizeCalculator`构造方法中对`Glide`所有内存缓存的计算，这包括`arrayPool`缓存的大小，`bitmapPool`缓存的大小，`memoryCache`缓存的大小。

* 我们主要讨论BitmapPool的size计算，在构造方法中，`targetBitmapPoolSize`的计算规则是屏幕尺寸的像素大小 * `builder.bitmapPoolScreens`； 其次`MemorySizeCalculator`还会根据`builder`的配置得到最大的缓存容量`maxSize`； 最后，会重新计算`targetBitmapPoolSize`,使其不超出最大容量；

* 接下来看一下`MemorySizeCalculator.Builder`中对`bitmapPoolScreens`的计算:

#### 6.4 MemorySizeCalculator.Builder

```java
public static final class Builder {

static final int BITMAP_POOL_TARGET_SCREENS =
        Build.VERSION.SDK_INT < Build.VERSION_CODES.O ? 4 : 1;

float bitmapPoolScreens = BITMAP_POOL_TARGET_SCREENS;

public Builder(Context context) {
      this.context = context;
      activityManager =
          (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
      screenDimensions =
          new DisplayMetricsScreenDimensions(context.getResources().getDisplayMetrics());

      // On Android O+ Bitmaps are allocated natively, ART is much more efficient at managing
      // garbage and we rely heavily on HARDWARE Bitmaps, making Bitmap re-use much less important.
      // We prefer to preserve RAM on these devices and take the small performance hit of not
      // re-using Bitmaps and textures when loading very small images or generating thumbnails.
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O && isLowMemoryDevice(activityManager)) {
        //在Android O上面，低内存的手机bitmapPoolScreens设置为0
        bitmapPoolScreens = 0;
      }
    }
}
```

* `bitmapPoolScreens`的值在这三种情况：
  * 如果设备小于Android O，取值**4**;
  * 如果设备大于等于Android O，低内存手机，取值**0**;
  * 如果设备大于等于Android O，非低内存手机，取值**1**;

* 至于为啥要取值**0**，Glide的解释是Android O上面Bitmap内存的申请在native，ART虚拟机对垃圾回收非常高效，而且我们可以用设置Bitmap`Config.HARDWARE`，所以对于Bitmap的缓存不是那么的重要。

### 七、总结

#### 7.1 小结

1. BitmapPool 大小通过 MemorySizeCalculator 设置；
2. 使用 LRU 算法维护 BitmapPool ；
3. Glide 会根据 Bitmap 的大小与 Config 生成一个 Key；
4. Key 也有自己对应的对象池，使用 Queue 实现；
5. 数据最终存储在 GroupedLinkedMap 中；
6. GroupedLinkedMap 使用哈希表、循环链表、List 来存储数据。

#### 7.2 整体己见

* 对于Bitmap而言，在使用完成，需要释放资源的时候，并不会释放bitmap对象，而是尝试把bitmap对象放入bitmappool中。当需要使用bitmap的时候就可以通过各种规则（如width/heigth/config的规则；如size/config的规则）去寻找有没有可以复用的bitmap，如果存在可以复用的bitmap，就直接从bitmappool中拿来使用，避免了不必要的创建和回收。