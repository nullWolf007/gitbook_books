[TOC]

## Glide缓存原理之磁盘缓存

#### 参考转载

* [探索 Glide 原理](https://juejin.cn/post/6882536990400020494#heading-87)
* [Glide源码分析（一）——DiskLruCache磁盘缓存的实现](https://blog.csdn.net/yxz329130952/article/details/65447541)

### 一、Glide 磁盘缓存策略

#### 1.1 概述

* 在加载图片时，我们可以用 diskCacheStratgy() 方法设置图片在磁盘的缓存策略，这个选项传入的参数类型为抽象类 DiskCacheStrategy。

* 磁盘缓存策略涉及到 Glide 的数据源类型 DataSource 和编码策略 EncodeStratefy.

#### 1.2 实例

```java
Glide.with(this)
	.load(R.drawable.ic_launcher_background)
    .diskCacheStrategy(DiskCacheStrategy.AUTOMATIC)
    .into(imageView);
```

#### 1.3 五种数据源

* Glide 中定义了下面 5 种数据源 DataSource。

##### 1.3.1 LOCAL

- 设备上有的数据，比如 App 内置的 Drawable 也属于 LOCAL ；

##### 1.3.2 REMOTE

- 从服务端拿到的数据；

##### 1.3.3 DATA_DISK_CACHE

- 从缓存中取出来的原始数据；

##### 1.3.4 RESOURCE_DISK_CACHE

- 从缓存中取出来的图片资源；

##### 1.3.5 MEMORY_CACHE

- 从内存缓存中取出来的数据；

#### 1.4 四个抽象方法

* DiskCacheStrategy 有下面 4 个抽象方法，这个 4 个方法的返回值都是布尔值。

##### 1.4.1 isDataCacheable()

* 是否保存图片的原始数据。
* DecodeJob 中用到的 SourceGenerator 在从图片来源获取到数据后，会根据这个方法判断是否保存图片的原始数据。

##### 1.4.2 isResourceCacheable()

* 是否保存解码后的图片数据。

* 当资源解码器对图片数据进行解码后，DecodeJob 就会根据这个方法的返回值决定是否保存该 Resource 。

##### 1.4.3 decodeCachedResource()

* 是否对缓存的解码后的图片数据进行解码。

* 在 DecodeJob 的 getNextStage() 中，会根据这个方法的返回值判断，如果返回值为 false，意味着跳过 RESOURCE_CACHE 步骤，也就是不对缓存中处理过的图片资源进行处理。

##### 1.4.4 decodeCachedData()

* 是否对缓存的原始数据进行解码。

* 在 DecodeJob 的 getNextStage() 方法中，会根据这个方法的返回值判断，如果该值为 false，意味着跳过 DATA_CACHE 步骤，也就是不对缓存中的原始图片数据进行处理。

#### 1.5 五种缓存策略

* Glide 定义好的磁盘缓存策略有下面 5 种，默认为 AUTOMATIC。

##### 1.5.1 AUTOMATIC

```java
  public static final DiskCacheStrategy AUTOMATIC =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return dataSource == DataSource.REMOTE;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return ((isFromAlternateCacheKey && dataSource == DataSource.DATA_DISK_CACHE)
                  || dataSource == DataSource.LOCAL)
              && encodeStrategy == EncodeStrategy.TRANSFORMED;
        }

        @Override
        public boolean decodeCachedResource() {
          return true;
        }

        @Override
        public boolean decodeCachedData() {
          return true;
        }
      };
```

- isDataCacheable()：只保存网络图片(DataSource.REMOTE)的原始数据；

- isResourceCacheable()：只保存数据源为缓存原始数据(DATA_DISK_CACHE) 或设备上数据(LOCAL)，并且编码策略为 TRANSFORMED 的图片资源；

- decodeCachedResource()：true；

- decodeCachedData()：true；


##### 1.5.2 ALL

```java
  public static final DiskCacheStrategy ALL =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return dataSource == DataSource.REMOTE;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return dataSource != DataSource.RESOURCE_DISK_CACHE
              && dataSource != DataSource.MEMORY_CACHE;
        }

        @Override
        public boolean decodeCachedResource() {
          return true;
        }

        @Override
        public boolean decodeCachedData() {
          return true;
        }
      };
```

- isDataCacheable()：只保存网络图片(REMOTE)的原始数据；

- isResourceCacheable()：不保存数据源为 缓存图片数据(RESOURCE_DISK_CACHE) 和 内存数据(MEMORY_CACHE) 的图片资源；

- decodeCachedResource()：true；

- decodeCachedData()：true；


##### 1.5.3 DATA

```java
  public static final DiskCacheStrategy DATA =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return dataSource != DataSource.DATA_DISK_CACHE && dataSource != DataSource.MEMORY_CACHE;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return false;
        }

        @Override
        public boolean decodeCachedResource() {
          return false;
        }

        @Override
        public boolean decodeCachedData() {
          return true;
        }
      };
```

- isDataCacheable()：不保存数据源为 缓存原始数据(DATA_DISK_CACHE) 或 内存数据(MEMORY_CACHE) 的图片资源；

- isResourceCacheable()：false；

- decodeCachedResource()：false；

- decodeCachedData()：true；


##### 1.5.4 RESOURCE

```java
  public static final DiskCacheStrategy RESOURCE =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return false;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return dataSource != DataSource.RESOURCE_DISK_CACHE
              && dataSource != DataSource.MEMORY_CACHE;
        }

        @Override
        public boolean decodeCachedResource() {
          return true;
        }

        @Override
        public boolean decodeCachedData() {
          return false;
        }
      };
```

- isDataCacheable()：false；

- isResourceCacheable()：不保存数据源为 RESOURCE_DISK_CACHE 和 MEMORY_CACHE 的图片资源；

- decodeCachedResource()：true；

- decodeCachedData()：false；


##### 5. NONE

```java
  public static final DiskCacheStrategy NONE =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return false;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return false;
        }

        @Override
        public boolean decodeCachedResource() {
          return false;
        }

        @Override
        public boolean decodeCachedData() {
          return false;
        }
      };
```

* 所有方法的返回值都为 false。

### 二、流程

* 先查看[二、启动解码任务流程](设计思想解读开源框架库\图片加载\Glide原理分析\二、启动解码任务流程.md)

#### 2.1 整体流程

* 构造函数中创建LazyDiskCacheProvider遍历diskCacheProvider，并作为参数传递给DecodeJobFactory对象decodeJobFactory作为其内部的成员变量
* Engine#load方法是起始
* Engine#load中调用Engine#waitForExistingOrStartNewJob方法
* Engine#waitForExistingOrStartNewJob方法中调用engineJob.start(decodeJob)方法，其中decodeJob是DecodeJobFactory对象decodeJobFactorybuild出来的
* EngineJob#start方法中运行线程池，最终调用DecodeJob对象decodeJob的run方法
* DecodeJob#run调用DecodeJob#runWrapped方法
* DecodeJob#runWrapped方法依次调用DecodeJob#getNextStage，DecodeJob#getNextGenerator方法和DecodeJob#runGenerators方法
* DecodeJob#getNextStage获得stage状态
* DecodeJob#getNextGenerator根据stage获取到ResourceCacheGenerator或者DataCacheGenerator或者SourceGenerator对象
* DecodeJob#runGenerators方法调用DataFetcherGenerator对象的startNext方法
* ResourceCacheGenerator/DataCacheGenerator/SourceGenerator都是继承自DataFetcherGenerator，调用其startNext方法
* 举例DataCacheGenerator#startNext方法中调用helper.getDiskCache().get(originalKey);即DecodeHelper#getDiskCache
* DecodeHelper#getDiskCache方法调用DecodeJob.DiskCacheProvider#getDiskCache
* DecodeJob.DiskCacheProvider是一个接口类，它的实现类就是Engine#LazyDiskCacheProvider，所以等同于Engine#LazyDiskCacheProvider#getDiskCache
* Engine#LazyDiskCacheProvider#getDiskCache获取的是DiskCache的diskCache对象
* 所以helper.getDiskCache().get(originalKey)等同于DiskCache#get方法
* 其中ResourceCacheGenerator也调用了类似的方法，从DiskCache中获取数据

#### 2.2 DecodeJob#getNextStage

```java
  private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```

* 主要就是通过diskCacheStrategy的方法去进行判断，然后返回不同的stage。主要是decodeCachedResource和decodeCachedData。这两个方法都是上面介绍的，这里就不赘述了。

#### 2.3 DecodeJob#getNextGenerator

```java
  private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```

* 通过stage，进入不同的CacheGenerator，其实stage表示的就是能否对缓存的解码图片数据解码或者缓存原始数据解码。都不行那自然就是源数据了。所以对应三个不同的CacheGenerator

#### 2.4 DecodeJob#runGenerators

```java
  private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }

  }
```

* 可以看到这里调用了currentGenerator的startNext方法，则对应三个ResourceCacheGenerator/DataCacheGenerator/SourceGenerator的startNext方法

### 三、磁盘缓存原理

#### 3.1 DiskCache

* Glide 是用 DiskCache 保存图片文件的。

```java
public interface DiskCache {

  interface Factory {
    int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;

    String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";

    @Nullable
    DiskCache build();
  }

  interface Writer {
    boolean write(@NonNull File file);
  }
	......
}
```

* DiskCache 是一个接口，这个接口中还定义了 Factory 和 Writer 两个接口，Writer 只是对 ResourceEncoder 的封装。
* DiskCache 有两个实现类， DiskCacheAdapter 和 DiskLruCacheWrapper。

#### 3.2 DiskCacheAdapter 

* DiskCacheAdapter 只是一个空实现。

#### 3.3 DiskLruCacheWrapper

* 从名字可以看得出来 DiskLruCacheWrapper 是对 DiskLruCache 的封装，具体的实现是在 DiskLruCache 中，DataCacheGenerator 和 ResourceCacheGenerator 都是用的 DiskLruCache 来获取磁盘缓存数据的。
* 成员变量

```java
public class DiskLruCacheWrapper implements DiskCache {

  private static final int APP_VERSION = 1;
  private static final int VALUE_COUNT = 1;
  private static DiskLruCacheWrapper wrapper;

  private final SafeKeyGenerator safeKeyGenerator;
  private final File directory;
  private final long maxSize;
  private final DiskCacheWriteLocker writeLocker = new DiskCacheWriteLocker();
  private DiskLruCache diskLruCache;
```

* 其中真正的实现类是DiskLruCache

#### 3.4 DiskLruCache

* 这部分将在下一节详细介绍

#### 3.5  DiskCache.Factory

##### 3.5.1 DiskCache#Factory

```java
 interface Factory {
    int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;

    String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";

    @Nullable
    DiskCache build();
  }
```

* 在 Factory 接口中，定义了默认的磁盘缓存大小为 250M，默认的缓存目录名称为 "image_manager_disk_cache" 。

##### 3.5.2 LazyDiskCacheProvider 

* 在 DiskCache 中有一个 Factory 工厂接口，这个接口用在了 Engine 的 LazyDiskCacheProvider 中。

```java
  private static class LazyDiskCacheProvider implements DecodeJob.DiskCacheProvider {

    private final DiskCache.Factory factory;
    private volatile DiskCache diskCache;

    LazyDiskCacheProvider(DiskCache.Factory factory) {
      this.factory = factory;
    }

    @VisibleForTesting
    synchronized void clearDiskCacheIfCreated() {
      if (diskCache == null) {
        return;
      }
      diskCache.clear();
    }

    @Override
    public DiskCache getDiskCache() {
      if (diskCache == null) {
        synchronized (this) {
          if (diskCache == null) {
            diskCache = factory.build();
          }
          if (diskCache == null) {
            diskCache = new DiskCacheAdapter();
          }
        }
      }
      return diskCache;
    }
  }

```

* 会调用DiskCache.Factory的build方法的

##### 3.5.3 实现类

* Factory 主要有下面 2 个实现类。

**ExternalPreferredCacheDiskCacheFactory**

* 用的是 getExternalCacheDir() 。

* 对应的目录是 /data/user/0/包名/cache/image_manager_disk_cache。

**InternalCacheDiskCacheFactory**

* 用的是 context.getCacheDir() 。

* 对应的目录是 /data/user/0/包名/cache 。

**默认情况下 Glide 用的是 InternalCacheDiskCacheFactory ，如果想把图片放在外部缓存目录的话，可以在自定义的 GlideModule 设置 DiskCache 。**

```java
public class GlideConfig extends AppGlideModule {
    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        super.applyOptions(context, builder);
        builder.setDiskCache(new ExternalPreferredCacheDiskCacheFactory(context));
    }
}
```

### 四、DiskLruCacheWrapper

* 这是DiskLruCache 的包装类

#### 4.1 DiskLruCacheWrapper#get

* 在ResourceCacheGenerator/DataCacheGenerator中都会调用此方法

```java
  @Override
  public File get(Key key) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
    }
    File result = null;
    try {
		//分析1
      final DiskLruCache.Value value = getDiskCache().get(safeKey);
        //分析2
      if (value != null) {
        result = value.getFile(0);
      }
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to get from disk cache", e);
      }
    }
    return result;
  }

  private synchronized DiskLruCache getDiskCache() throws IOException {
      //分析3
    if (diskLruCache == null) {
      diskLruCache = DiskLruCache.open(directory, APP_VERSION, VALUE_COUNT, maxSize);
    }
    return diskLruCache;
  }
```

* 分析1：调用getDiskCache获取DiskLruCache对象，分析3为此方法，会调用DiskLruCache的open方法进行获取。然后调用DiskLruCache的get方法获取DiskLruCache#value对象
* 分析2：通过DiskLruCache#value对象的getFile方法获取文件
* DiskLruCache#open方法见5.8
* DiskLruCache#get方法见5.9
* DiskLruCache#value#getFile方法见5.9.3

#### 4.2 DiskLruCacheWrapper#put

* DecodeJob#decodeFromRetrievedData的方法中，最终会调用此put方法

```java
  @Override
  public void put(Key key, Writer writer) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    writeLocker.acquire(safeKey);
    try {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Put: Obtained: " + safeKey + " for for Key: " + key);
      }
      try {
          //分析1
        DiskLruCache diskCache = getDiskCache();
        Value current = diskCache.get(safeKey);
        if (current != null) {
          return;
        }
		//分析2
        DiskLruCache.Editor editor = diskCache.edit(safeKey);
        if (editor == null) {
          throw new IllegalStateException("Had two simultaneous puts for: " + safeKey);
        }
        try {
          File file = editor.getFile(0);
          if (writer.write(file)) {
            editor.commit();
          }
        } finally {
          editor.abortUnlessCommitted();
        }
      } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.WARN)) {
          Log.w(TAG, "Unable to put to disk cache", e);
        }
      }
    } finally {
      writeLocker.release(safeKey);
    }
  }
```

* 分析1：获取DiskLruCache对象，然后调用DiskLruCache#get方法，看是否由对应的值，如果有，则说明已经缓存过了，则直接return
* 分析2：如果没有缓存过，则需要缓存，首先获取DiskLruCache.Editor对象，然后根据DiskLruCache.Editor获取File对象，然后把需要缓存的写入进去，然后调用DiskLruCache.Editor的commit方法进行提交，最后调用DiskLruCache.Editor的abortUnlessCommitted方法
* DiskLruCache#edit方法见5.11
* DiskLruCache#Editor#getFile见5.11，返回是临时文件，所以写入的也是临时文件
* DiskLruCache#Editor#commit见5.12
* DiskLruCache#Editor#abortUnlessCommitted见5.13

### 五、DiskLruCache 

#### 5.1 概述

* DiskLruCache并非针对Glide编写的，而是一个通用的磁盘缓存实现，虽然并非Google官方的代码，但是已经在很多应用中得到了引入使用。

#### 5.2 journal日志

* DiskLruCache通过日志来辅助保证磁盘缓存的有效性。在应用程序运行阶段，可以通过内存数据来保证缓存的有效性，但是一旦应用程序退出或者被意外杀死，下次再启动的时候就需要通过journal日志来重新构建磁盘缓存数据记录，保证上次的磁盘缓存是有效和可用的。

#### 5.3 journal日志基本数据

* 为了理解journal日志是如何起作用的，首先需要理解journal日志的结构。以下是journal日志的基本数据：

```java
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

* 其中第一行固定为libcore.io.DiskLruCache；第二行是DiskLruCache的版本，目前固定为1；第三行表示所属应用的版本号；第四行valueCount表示一个缓存key可能对应多少个缓存文件，它决定了后面一个CLEAN状态记录最多可以size大小数据；第五行是空行。此后记录的就是DiskLruCache针对磁盘缓存的操作记录了。其中几个状态表示如下：
  * CLEAN 表示缓存处于一个稳定状态，即当前没有对该缓存数据进行写操作，在该状态下，对缓存文件的读写都是安全的。
  * DIRTY 表示当前该key对应的缓存文件正在被修改，该状态下对缓存文件的读写都是不安全的，需要阻塞到对文件的修改完成，使该key对应的状态转变成CLEAN为止。
  * REMOVE 表示该key对应的缓存文件被删除了，在缓存整理的过程中可能会出现多条这样的记录。
  * READ 表示一个对key对应的缓存文件进行读取的操作记录。
* 每个操作记录状态后面都有一个字符串，表示缓存的key，其中CLEAN状态在后面还会有一个或者多个数字，这些数字表示对应缓存文件的大小。之所以允许一个key对应多个文件，主要是考虑到满足类似于一张图片可能存在多个大小和分辨率不同的缓存的功能。

#### 5.4 Entry

* 一个Entry对应一条缓存，DiskLruCache通过Entry对相应的缓存进行操作。其主要的成员为：

```java
private final class Entry {
    //缓存的key
    private final String key;
    //表示缓存文件的大小
    //因为一个Key也可能存在多个大小不同的缓存文件，例如尺寸或者分辨率不同的图片，所以可能会存在多个length
    private final long[] lengths;
    //缓存文件
    File[] cleanFiles;
    //临时文件
    File[] dirtyFiles;
    //表示该缓存是否能够被读取
    private boolean readable;
    //正在操作entry的editor 
    //是Entry用来修改缓存文件的类
    private Editor currentEditor;
	//是一个唯一的序号，每当对应的缓存文件被改变后，该序号就改变
    private long sequenceNumber;

  }
```

* key：缓存的key
* lengths：表示缓存文件的大小；因为一个Key也可能存在多个大小不同的缓存文件，例如尺寸或者分辨率不同的图片，所以可能会存在多个length
* cleanFiles：缓存文件
* dirtyFiles：临时文件
* readable：表示该缓存是否能够被读取，只有在缓存状态为CLEAN的时候才为true
* currentEditor：是Entry用来修改缓存文件的类
* sequenceNumber：是一个唯一的序号，每当对应的缓存文件被改变后，该序号就改变，在通过snapshot获取缓存映像的时候，snapshot内部的sequenceNumber和当时刻的Entry的sequenceNumber相同，如果在此后Entry对应的缓存文件被改变，那么通过snapshot获取缓存的时候就能发现二者的sequenceNumber不相同，因此snapshot就失效了，这样避免通过snapshot获取到旧的缓存数据信息。

#### 5.5 Editor

* Editor是用来对缓存文件进行修改的操作的封装，他和Entry一一对应，一个entry如果其中的currentEditor不为空，表示这个Entry对应的缓存文件正在被修改，即该Entry处于DIRTY状态。总体来说，Editor的功能单一实现简单。唯一需要说明一下的是在使用它对文件进行操作后，需要调用它的commit方法，以使它对缓存文件的更改记录更新到内存lruEntries和日志当中，否则对应的缓存将会始终处于DIRTY状态而无法被再次修改和读取。

#### 5.6 Snapshot

* Snapshot就是一个缓存记录的映像，它代表在某一时刻缓存的状态，它和某一时刻的Entry一一对应，前面也提到过，一旦Snapshot对应的Entry被修改了，那么虽然该Snapshot虽然还和这个Entry有对应关系，但是因为缓存文件的内容已经发生了改变，所以该Snapshot处于失效状态，不能被使用。
* Snapshot是封装给外部使用的，它封装了接口专门用来对缓存文件进行读写，方便外部通过它直接访问缓存文件而不管具体的缓存处理细节。

#### 5.7 DiskLruCache#成员变量

```java
public final class DiskLruCache implements Closeable {
	private final LinkedHashMap<String, Entry> lruEntries =
		new LinkedHashMap<String, Entry>(0, 0.75f, true);
    
}
```

* 优先阅读[Glide缓存原理之LruCache](设计思想解读开源框架库\图片加载\Glide原理分析\Glide缓存原理之LruCache)
* DiskLruCache和LruCache 采用的LinkedHashMap 去存储数据，同时是按照访问顺序进行排序的。
* LinkedHashMap 的Key的类型为String，value类型为Entry。
* lruEntries是一个LinkedHashMap，DiskLruCache用它来实现LRU的功能。它是缓存在内存中的记录，能够反映当前磁盘中的缓存文件的状态，通过读取这个数据结构，能够快速对磁盘中是否有对应key的缓存文件做出判断。

#### 5.8 DiskLruCache#open

##### 5.8.1 DiskLruCache#open

```java
  public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
      throws IOException {
    if (maxSize <= 0) {
      throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
      throw new IllegalArgumentException("valueCount <= 0");
    }

    File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    if (backupFile.exists()) {
      File journalFile = new File(directory, JOURNAL_FILE);
      if (journalFile.exists()) {
        backupFile.delete();
      } else {
        renameTo(backupFile, journalFile, false);
      }
    }

    DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    if (cache.journalFile.exists()) {
      try {
          //分析1：读取Journal日志
        cache.readJournal日志();
          //分析2：删除不必要Journal日志
        cache.processJournal();
        return cache;
      } catch (IOException journalIsCorrupt) {
        System.out
            .println("DiskLruCache "
                + directory
                + " is corrupt: "
                + journalIsCorrupt.getMessage()
                + ", removing");
        cache.delete();
      }
    }

    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    cache.rebuildJournal();
    return cache;
  }
```

* 主要就是在DiskLruCache初始化的时候，根据journal日志来初始化其中的lruEntries
* 分析1：读取Journal日志
* 分析2：删除不必要Journal日志

##### 5.8.2 DiskLruCache#readJournal

```java
  private void readJournal() throws IOException {
    StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
    try {
      String magic = reader.readLine();
      String version = reader.readLine();
      String appVersionString = reader.readLine();
      String valueCountString = reader.readLine();
      String blank = reader.readLine();
      if (!MAGIC.equals(magic)
          || !VERSION_1.equals(version)
          || !Integer.toString(appVersion).equals(appVersionString)
          || !Integer.toString(valueCount).equals(valueCountString)
          || !"".equals(blank)) {
        throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
            + valueCountString + ", " + blank + "]");
      }

      int lineCount = 0;
      while (true) {
        try {
            //分析1
          readJournalLine(reader.readLine());
          lineCount++;
        } catch (EOFException endOfJournal) {
          break;
        }
      }
      redundantOpCount = lineCount - lruEntries.size();

      if (reader.hasUnterminatedLine()) {
        rebuildJournal();
      } else {
        journalWriter = new BufferedWriter(new OutputStreamWriter(
            new FileOutputStream(journalFile, true), Util.US_ASCII));
      }
    } finally {
      Util.closeQuietly(reader);
    }
  }

```

* 读取日志文件中的内容，根据日志文件中的记录来初始化对应entry的状态并构建lruEntries
* 分析1：调用readJournalLine方法

##### 5.8.3 DiskLruCache#readJournalLine

```java
  private void readJournalLine(String line) throws IOException {
    int firstSpace = line.indexOf(' ');
    if (firstSpace == -1) {
      throw new IOException("unexpected journal line: " + line);
    }

    int keyBegin = firstSpace + 1;
    int secondSpace = line.indexOf(' ', keyBegin);
    final String key;
    if (secondSpace == -1) {
      key = line.substring(keyBegin);
      if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
        lruEntries.remove(key);
        return;
      }
    } else {
      key = line.substring(keyBegin, secondSpace);
    }

    Entry entry = lruEntries.get(key);
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }

    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
      String[] parts = line.substring(secondSpace + 1).split(" ");
      entry.readable = true;
      entry.currentEditor = null;
      entry.setLengths(parts);
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
      entry.currentEditor = new Editor(entry);
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
      // This work was already done by calling lruEntries.get().
    } else {
      throw new IOException("unexpected journal line: " + line);
    }
  }
```

* 读取每行日志，初始化日志对应的entry状态
* 读取key以及创建entry，然后通过lruEntries.put(key, entry);放入到lruEntries中，然后对Entry的成员变量进行初始化

##### 5.8.4 DiskLruCache#processJournal

* 删除不必要Journal日志

```java
private void processJournal() throws IOException {
    deleteIfExists(journalFileTmp);
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
        Entry entry = i.next();
        // currenEditor为null，表示当前为clean状态
        if (entry.currentEditor == null) {      
            for (int t = 0; t < valueCount; t++) {
                // 累计大小
                size += entry.lengths[t];       
            }
        } else {        
            // 在初始化阶段才调用该函数
            //如果此时日志文件中记录该项为dirty，那么就直接放弃这条缓存
            //删除其相关的文件以及在lruEntries中的记录
            entry.currentEditor = null;
            for (int t = 0; t < valueCount; t++) {
                deleteIfExists(entry.getCleanFile(t));
                deleteIfExists(entry.getDirtyFile(t));
            }
            i.remove();
        }
    }
}
```

##### 5.8.5 小结

* 在open函数中通过readJournal和processJournal两个函数来完成从journal日志到lruEntries的构建。其中readJournal和readJournalLine分别通过解析日志的每一行内容，通过简单的容错逻辑来初步构建lruEntries的内容，包括缓存key和其状态和缓存文件大小等。其中解析是从第一行到最后一行，因此一个key可能在lruEntries中被操作多次，特别是对缓存状态的改变。在readJournal完成后，lruEntries基本上就是上次引用结束后的DiskLruCache的缓存状态，之后再调用processJournal对异常状态进行处理后就得到了一个有效且能反映当前磁盘缓存状态的lruEntries记录了。

#### 5.9 DiskLruCache#get

##### 5.9.1 DiskLruCache#get

```java
 public synchronized Value get(String key) throws IOException {
    checkNotClosed();
     //分析1
    Entry entry = lruEntries.get(key);
    if (entry == null) {
      return null;
    }

    if (!entry.readable) {
      return null;
    }
	
     //分析2
    for (File file : entry.cleanFiles) {
        // A file must have been deleted manually!
        if (!file.exists()) {
            return null;
        }
    }
	
     //分析3
    redundantOpCount++;
    journalWriter.append(READ);
    journalWriter.append(' ');
    journalWriter.append(key);
    journalWriter.append('\n');
     //判断是否需要重新构建日志（当缓存大小达到上限的时候）
    if (journalRebuildRequired()) {
      executorService.submit(cleanupCallable);
    }

    return new Value(key, entry.sequenceNumber, entry.cleanFiles, entry.lengths);
  }
```

* 分析1：通过lruEntries获取到Entry对象
* 分析2：cleanFiles是缓存文件的对象，遍历cleanFiles，判断文件是否存在
* 分析3：添加记录到journal日志中，并判断是否需要重新构建日志，重建会调用cleanupCallable的call方法
* 分析4：返回一个Value对象，也就是上文提到的Snapshot

##### 5.9.2 DiskLruCache#Value

```java
  public final class Value {
    private final String key;
    private final long sequenceNumber;
    private final long[] lengths;
    private final File[] files;
```

* key：缓存的key
* sequenceNumber：是一个唯一的序号，每当对应的缓存文件被改变后，该序号就改变，在通过snapshot获取缓存映像的时候，snapshot内部的sequenceNumber和当时刻的Entry的sequenceNumber相同，如果在此后Entry对应的缓存文件被改变，那么通过snapshot获取缓存的时候就能发现二者的sequenceNumber不相同，因此snapshot就失效了，这样避免通过snapshot获取到旧的缓存数据信息。
* sequenceNumber：读取的时候，数据没有改变，所以也不会变
* lengths：缓存文件的大小（可能多个缓存文件）
* files：缓存文件（可能多个）

##### 5.9.3 DiskLruCache#value#getFile

* getFile(index)，就是从缓存文件数组中拿缓存文件

#### 5.10 重新构建日志

##### 5.10.1 DiskLruCache#journalRebuildRequired

```java
  private boolean journalRebuildRequired() {
    final int redundantOpCompactThreshold = 2000;
    return redundantOpCount >= redundantOpCompactThreshold //
        && redundantOpCount >= lruEntries.size();
  }
```

* 当记录条数大于200，或者记录条数大于lruEntries的size就需要重建
* 当记录条数大于200，说明大于了缓存大小，需要重建

* 记录条数大于lruEntries的size说明到达了一定的大小，需要重建

##### 5.10.2 Callable#call

```java
  private final Callable<Void> cleanupCallable = new Callable<Void>() {
    public Void call() throws Exception {
      synchronized (DiskLruCache.this) {
        if (journalWriter == null) {
          return null; // Closed.
        }
          //分析1
        trimToSize();
          //分析2
        if (journalRebuildRequired()) {
            //分析3
          rebuildJournal();
          redundantOpCount = 0;
        }
      }
      return null;
    }
  };
```

* 日志清理任务，会调用call方法
* 分析1：调用trimToSize
* 分析2：判断是否需要重建
* 分析3：调用rebuildJournal方法重建，并把记录条数置为0，重新计数

##### 5.10.3 DiskLruCache#trimToSize

```java
private void trimToSize() throws IOException {
    while (size > maxSize) {
    //            Map.Entry<String, Entry> toEvict = lruEntries.eldest();
        final Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
        remove(toEvict.getKey());
    }
}
```

* maxSize是lruEntries最大的size，保证当前lruEntries的size在最大限制之内。由于访问顺序排序，所以最老的就在最前面，所以从前往后删除即可，删除的是lruEntries数据

##### 5.10.4 DiskLruCache#rebuildJournal

* 根据内部数据结构lruEntries来构造新的日志文件，然后替换原来的日志文件。
   * 在初始化阶段，一般构建的日志文件中没有dirty记录，如果是日志清理线程调用的，那么存在dirty的记录
   * 注意是同步方法

```java
private synchronized void rebuildJournal() throws IOException {
    if (journalWriter != null) {
        journalWriter.close();
    }

    Writer writer = new BufferedWriter(new FileWriter(journalFileTmp), IO_BUFFER_SIZE);     // 先写到tmp文件中，注意该方法是同步方法
    writer.write(MAGIC);
    writer.write("\n");
    writer.write(VERSION_1);
    writer.write("\n");
    writer.write(Integer.toString(appVersion));
    writer.write("\n");
    writer.write(Integer.toString(valueCount));
    writer.write("\n");
    writer.write("\n");

    for (Entry entry : lruEntries.values()) {
        if (entry.currentEditor != null) {
            writer.write(DIRTY + ' ' + entry.key + '\n');
        } else {
            writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        }
    }

    writer.close();
    journalFileTmp.renameTo(journalFile);       // 重命名成journal文件
    journalWriter = new BufferedWriter(new FileWriter(journalFile, true), IO_BUFFER_SIZE);
}
```

#### 5.11 DiskLruCache#edit

##### 5.11.1 DiskLruCache#edit

* 4.2的DiskLruCacheWrapper#put最终会调用此方法

```java
  public Editor edit(String key) throws IOException {
    return edit(key, ANY_SEQUENCE_NUMBER);
  }

//分析1
  private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    checkNotClosed();
      //分析2
    Entry entry = lruEntries.get(key);
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        || entry.sequenceNumber != expectedSequenceNumber)) {
        // entry为空，表示该缓存已经被删除，
        // sequenceNumber和snapshot不匹配，说明entry对应的内容已经改变，不能通过该snapshot来获取缓存内容
      return null; // Value is stale.
    }
    if (entry == null) {
        //分析3
      entry = new Entry(key);
      lruEntries.put(key, entry);
    } else if (entry.currentEditor != null) {
        // 已经有editor正在进行修改
      return null; // Another edit is in progress.
    }

    Editor editor = new Editor(entry);
    entry.currentEditor = editor;

    // 分析4
    journalWriter.append(DIRTY);
    journalWriter.append(' ');
    journalWriter.append(key);
    journalWriter.append('\n');
    flushWriter(journalWriter);
    return editor;
  }
```

* 分析1：采用的是同步方法来写缓存和日志文件
* 分析2：先获取Entry对象，然后判断是否为空/sequenceNumber是否一致/是否已经有currentEditor，否则会直接返回null
* 分析3：把数据存储到lruEntries中去
* 分析4：写入日志文件，采用的是DIRTY，表示正在修改

##### 5.11.2 DiskLruCache#Editor#getFile

* 4.2的DiskLruCacheWrapper#put最终会调用此方法

```java
  public File getFile(int index) throws IOException {
      //分析1
      synchronized (DiskLruCache.this) {
        if (entry.currentEditor != this) {
            throw new IllegalStateException();
        }
        if (!entry.readable) {
            written[index] = true;
        }
          //分析2
        File dirtyFile = entry.getDirtyFile(index);
        if (!directory.exists()) {
            directory.mkdirs();
        }
        return dirtyFile;
      }
    }
```

* 分析1：可以看到采用的是同步方法
* 分析2：通过entry.getDirtyFile去获取文件，getDirtyFile拿到的是临时文件，返回的是临时文件

#### 5.12 DiskLruCache#Editor#commit

##### 5.12.1 DiskLruCache#Editor#commit

* 4.2的DiskLruCacheWrapper#put最终会调用此方法

```java
 public void commit() throws IOException {
      // The object using this Editor must catch and handle any errors
      // during the write. If there is an error and they call commit
      // anyway, we will assume whatever they managed to write was valid.
      // Normally they should call abort.
      completeEdit(this, true);
      committed = true;
    }
```

* 可以看到调用了completeEdit方法

##### 5.12.2 DiskLruCacher#completeEdit

* 在editor修改了缓存后调用，用来同步缓存内容和更新日志及相关状态
 * 要写缓存和日志，同步方法

```java
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    Entry entry = editor.entry;
    if (entry.currentEditor != editor) {
        throw new IllegalStateException();
    }

    // if this edit is creating the entry for the first time, every index must have a value
    if (success && !entry.readable) {
        for (int i = 0; i < valueCount; i++) {
            // 因为editor在edit的时候是将缓存内容写到dirty文件中的，因此这里要求dirty文件一定要存在
            if (!entry.getDirtyFile(i).exists()) {      
                editor.abort();
                throw new IllegalStateException("edit didn't create file " + i);
            }
        }
    }

    for (int i = 0; i < valueCount; i++) {
        File dirty = entry.getDirtyFile(i);
        if (success) {
            if (dirty.exists()) {
                File clean = entry.getCleanFile(i);
                // 写入成功，将dirty文件编程clean 文件
                dirty.renameTo(clean);      
                long oldLength = entry.lengths[i];
                long newLength = clean.length();
                entry.lengths[i] = newLength;
                // 更新缓存大小
                size = size - oldLength + newLength;    
            }
        } else {
            // dirty文件的使命完成
            deleteIfExists(dirty);      
        }
    }

    // 增加日志一条
    redundantOpCount++; 
    // entry clean了
    entry.currentEditor = null;     
    if (entry.readable | success) {
        entry.readable = true;
        journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        if (success) {
            // 更新sequenceNumber，这样之前的旧的snapshot就不会读错数据
            entry.sequenceNumber = nextSequenceNumber++;    
        }
    } else {
        lruEntries.remove(entry.key);
        journalWriter.write(REMOVE + ' ' + entry.key + '\n');
    }

    if (size > maxSize || journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }
}
```

* 主要步骤：首先把临时文件dirtyFiles写入到缓存文件cleanFiles中，并更新缓存大小；日志条数增加1；把currentEditor置空，表明当前使用完毕；由于修改数据了，所以sequenceNumber的值也需要改变，改变此值；然后判断是否需要重建。

#### 5.13 DiskLruCache#Editor#abortUnlessCommitted

* 4.2的DiskLruCacheWrapper#put最终会调用此方法

```java
    public void abort() throws IOException {
      completeEdit(this, false);
    }

    public void abortUnlessCommitted() {
      if (!committed) {
        try {
          abort();
        } catch (IOException ignored) {
        }
      }
    }
```

* 当committed为false的时候，说明异常了。则调用abort方法
* abort调用completeEdit方法，与上面不同的是参数为false，主要的目的就是，对其释放同步锁，让后面的可以拿到锁
* completeEdit是对象锁，所以同一个对象是可以拿到这个锁的，进入completeEdit方法。

### 六、总结

* Glide的磁盘缓存有不同的缓存策略，可以缓存源数据，也可以缓存解码数据，这个用户自己设置

* Glide的磁盘缓存主要是通过DiskLruCache来实现的
* DiskLruCache采用了journal日志技术，通过日志来辅助保证磁盘缓存的有效性。DiskLruCache内部通过LinkedHashMap数据结构进行存储的，并且按照访问顺序进行排序，保证了LRU机制
* LinkedHashMap采用String类型为key，Entry类型为value
* Entry包括临时文件和缓存文件，写入的时候是先写入到临时文件中，在从临时文件写入到缓存文件中去。du去的时候从缓存文件中获取
* 每次操作都会记录到journal日志中，进行记录，默认为200条记录。这样万一程序崩溃，下次初始化可以从日志文件中构建DiskLruCache#lruEntries



