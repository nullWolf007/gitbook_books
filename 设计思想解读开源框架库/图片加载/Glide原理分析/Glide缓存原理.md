[TOC]

## Glide缓存原理

#### 参考

* [Glide 源码分析解读-缓存模块-基于最新版Glide 4.9.0](https://zhuanlan.zhihu.com/p/60426316)
* [glide缓存之ActiveResources](https://www.jianshu.com/p/43656b43cb1b)

### 一、基础知识

#### 1.1 Glide启动解码任务流程

* 详情请查看[二、启动解码任务流程](..\..\..\设计思想解读开源框架库\图片加载\Glide原理分析\二、启动解码任务流程.md)

#### 1.2 Engine

* Engine 是 Glide 的图片加载引擎。
* 其中重中之中的功能就是从缓存的获取数据

#### 1.3 说明

* 本文源码分析是基于4.11.0版本

```groovy
implementation 'com.github.bumptech.glide:glide:4.11.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0'
```

### 二、Engine

#### 2.1 缓存流程

* 缓存获取的相关逻辑集中在Engine#load方法中

* Engine#load逻辑整体流程图如下所示：
![Glide#Engine#load获取缓存流程](..\..\..\images\设计思想解读开源框架库\图片加载\Glide原理分析\Glide#Engine#load获取缓存流程.png)

#### 2.2 Engine#构造函数

* 调用构造函数，最终会调用此方法

```java
public class Engine
        implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
  	private static final String TAG = "Engine";
  	private static final int JOB_POOL_SIZE = 150;
  	private static final boolean VERBOSE_IS_LOGGABLE = Log.isLoggable(TAG, Log.VERBOSE);
  	private final Jobs jobs;
  	private final EngineKeyFactory keyFactory;
  	private final MemoryCache cache;
  	private final EngineJobFactory engineJobFactory;
  	private final ResourceRecycler resourceRecycler;
  	private final LazyDiskCacheProvider diskCacheProvider;
  	private final DecodeJobFactory decodeJobFactory;
  	private final ActiveResources activeResources;
	.......
    @VisibleForTesting
    Engine(
            MemoryCache cache,
            DiskCache.Factory diskCacheFactory,
            GlideExecutor diskCacheExecutor,
            GlideExecutor sourceExecutor,
            GlideExecutor sourceUnlimitedExecutor,
            GlideExecutor animationExecutor,
            Jobs jobs,
            EngineKeyFactory keyFactory,
            ActiveResources activeResources,
            EngineJobFactory engineJobFactory,
            DecodeJobFactory decodeJobFactory,
            ResourceRecycler resourceRecycler,
            boolean isActiveResourceRetentionAllowed) {
        this.cache = cache;
        this.diskCacheProvider = new LazyDiskCacheProvider(diskCacheFactory);

        if (activeResources == null) {
            activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
        }
        this.activeResources = activeResources;
        activeResources.setListener(this);

        if (keyFactory == null) {
            keyFactory = new EngineKeyFactory();
        }
        this.keyFactory = keyFactory;

        if (jobs == null) {
            jobs = new Jobs();
        }
        this.jobs = jobs;

        if (engineJobFactory == null) {
            engineJobFactory =
                    new EngineJobFactory(
                            diskCacheExecutor,
                            sourceExecutor,
                            sourceUnlimitedExecutor,
                            animationExecutor,
                            /*engineJobListener=*/ this,
                            /*resourceListener=*/ this);
        }
        this.engineJobFactory = engineJobFactory;

        if (decodeJobFactory == null) {
            decodeJobFactory = new DecodeJobFactory(diskCacheProvider);
        }
        this.decodeJobFactory = decodeJobFactory;

        if (resourceRecycler == null) {
            resourceRecycler = new ResourceRecycler();
        }
        this.resourceRecycler = resourceRecycler;

        cache.setResourceRemovedListener(this);
    }
	......
}
```

* 对于该文章而言，需要关注的是缓存相关的成员变量，所以精简为如下代码

```java
public class Engine
        implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
  	private final MemoryCache cache;
  	private final LazyDiskCacheProvider diskCacheProvider;
  	private final ActiveResources activeResources;
	.......
    @VisibleForTesting
    Engine(
            MemoryCache cache,
            DiskCache.Factory diskCacheFactory,
            GlideExecutor diskCacheExecutor,
            GlideExecutor sourceExecutor,
            GlideExecutor sourceUnlimitedExecutor,
            GlideExecutor animationExecutor,
            Jobs jobs,
            EngineKeyFactory keyFactory,
            ActiveResources activeResources,
            EngineJobFactory engineJobFactory,
            DecodeJobFactory decodeJobFactory,
            ResourceRecycler resourceRecycler,
            boolean isActiveResourceRetentionAllowed) {
        this.cache = cache;
        this.diskCacheProvider = new LazyDiskCacheProvider(diskCacheFactory);
        if (activeResources == null) {
            activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
        }
        this.activeResources = activeResources;
        activeResources.setListener(this);
        ......
        cache.setResourceRemovedListener(this);
    }
	......
}
```

* 主要就是MemoryCache/LazyDiskCacheProvider/ActiveResources这三个对象的实例

#### 2.3 Engine#load

* 跟上面一样，下面只会列出和本文章相关部分的源码

```java
public <R> LoadStatus load(
      ......) {
    //分析1
    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> memoryResource;
    synchronized (this) {
		//分析2
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
          //分析3
        	return waitForExistingOrStartNewJob(
                glideContext,
                model,
                signature,
                width,
                height,
                resourceClass,
                transcodeClass,
                priority,
                diskCacheStrategy,
                transformations,
                isTransformationRequired,
                isScaleOnlyOrNoTransform,
                options,
                isMemoryCacheable,
                useUnlimitedSourceExecutorPool,
                useAnimationPool,
                onlyRetrieveFromCache,
                cb,
                callbackExecutor,
                key,
                startTime);
      }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
  }
```

* 分析2：这里是我们讨论的重点，从方法名我们也不难看出这是一个从内存中获取数据的方法

#### 2.4 Engine#loadFromMemory

```java
  @Nullable
  private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }
	
      //分析1
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return active;
    }
      
      //分析2
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return cached;
    }

    return null;
  }
```

* 分析1：调用loadFromActiveResources方法获取EngineResource，如果获取到了，会直接返回
* 分析2：如果上面的方法没有获取到，则调用loadFromCache方法获取EngineResource，如果不为null，返回此对象，不然返回null
* 其实loadFromActiveResources就是从ActiveResources中获取数据，loadFromCache就是从MemoryCache中获取数据，这两个会在后续进行详细介绍

#### 2.5 从DiskCache获取数据

| 由于从DiskCache中获取比较复杂，只说一下方法的流程，具体自己查看源码取看

* 构造函数中创建LazyDiskCacheProvider遍历diskCacheProvider，并作为参数传递给DecodeJobFactory对象decodeJobFactory作为其内部的成员变量

* Engine#load方法是起始
* Engine#load中调用Engine#waitForExistingOrStartNewJob方法
* Engine#waitForExistingOrStartNewJob方法中调用engineJob.start(decodeJob)方法，其中decodeJob是DecodeJobFactory对象decodeJobFactorybuild出来的
* EngineJob#start方法中运行线程池，最终调用DecodeJob对象decodeJob的run方法
* DecodeJob#run调用DecodeJob#runWrapped方法
* DecodeJob#runWrapped方法依次调用DecodeJob#getNextGenerator方法和DecodeJob#runGenerators方法
* DecodeJob#getNextGenerator获取到ResourceCacheGenerator或者DataCacheGenerator或者SourceGenerator对象
* DecodeJob#runGenerators方法调用DataFetcherGenerator对象的startNext方法
* ResourceCacheGenerator/DataCacheGenerator/SourceGenerator都是继承自DataFetcherGenerator，调用其startNext方法
* 举例DataCacheGenerator#startNext方法中调用helper.getDiskCache().get(originalKey);即DecodeHelper#getDiskCache
* DecodeHelper#getDiskCache方法调用DecodeJob.DiskCacheProvider#getDiskCache
* DecodeJob.DiskCacheProvider是一个接口类，它的实现类就是Engine#LazyDiskCacheProvider，所以等同于Engine#LazyDiskCacheProvider#getDiskCache
* Engine#LazyDiskCacheProvider#getDiskCache获取的是DiskCache的diskCache对象
* 所以helper.getDiskCache().get(originalKey)等同于DiskCache#get方法
* 其中ResourceCacheGenerator也调用了类似的方法，从DiskCache中获取数据

#### 2.6 总结

* 先从ActiveResources中获取数据
* 从MemoryCache中获取数据
* 从DiskCache中获取数据

### 三、GlideBuilder

#### 3.1 Glide#with

```java
Glide.with(this).load(R.drawable.ic_launcher_background).into(imageView);
```

* 一般我们使用如上的代码调用Glide

```java
@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}
```

* 会调用getRetriever方法

#### 3.2 Glide#getRetriever

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
	Preconditions.checkNotNull(
    	context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}
```

* 紧接着会调用Glide的get方法

#### 3.3 Glide#get

```java
@NonNull
public static Glide get(@NonNull Context context) {
	if (glide == null) {
    	GeneratedAppGlideModule annotationGeneratedModule =
        	getAnnotationGeneratedGlideModules(context.getApplicationContext());
        synchronized (Glide.class) {
            if (glide == null) {
                checkAndInitializeGlide(context, annotationGeneratedModule);
            }
        }
	}

    return glide;
}
```

* 如果glide为null，会调用checkAndInitializeGlide方法

#### 3.4 Glide#checkAndInitializeGlide

```java
@GuardedBy("Glide.class")
private static void checkAndInitializeGlide(@NonNull Context context,
	@Nullable GeneratedAppGlideModule generatedAppGlideModule) {
	if (isInitializing) {
    	throw new IllegalStateException(
        	"You cannot call Glide.get() in registerComponents(),"
            + " use the provided Glide instance instead");
	}
    isInitializing = true;
    initializeGlide(context, generatedAppGlideModule);
    isInitializing = false;
}
```

* 调用initializeGlide方法

#### 3.5 Glide#initializeGlide

```java
@GuardedBy("Glide.class")
private static void initializeGlide(@NonNull Context context, 
	@Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    initializeGlide(context, new GlideBuilder(), generatedAppGlideModule);
}
```

* 这里面会创建GlideBuilder对象，new GlideBuilder()

#### 3.6 GlideBuilder#构造函数

* 这是默认的空的构造函数

#### 3.7 Glide#initializeGlide

```java
private static void initializeGlide(@NonNull Context context,
	@NonNull GlideBuilder builder,
	@Nullable GeneratedAppGlideModule annotationGeneratedModule) {
    Context applicationContext = context.getApplicationContext();
    ......
	Glide glide = builder.build(applicationContext);
    ......
}
```

* 其中会调用GlideBuilder的build方法

#### 3.8 GlideBuilder#build

```java
  @NonNull
  Glide build(@NonNull Context context) {
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }

    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }

    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }

    if (bitmapPool == null) {
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              animationExecutor,
              isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }

    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled,
        isImageDecoderEnabledForBitmaps);
  }
```

* 创建了一个Glide对象
* 其中把涉及到本章内容的相关部分提取出来

```java
@NonNull
Glide build(@NonNull Context context) {
    //根据当前机器参数计算需要设置的缓存大小
	if (memorySizeCalculator == null) {
    	memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
	//创建 Bitmap 池
	if (bitmapPool == null) {
    	int size = memorySizeCalculator.getBitmapPoolSize();
      	if (size > 0) {
        	bitmapPool = new LruBitmapPool(size);
	 	} else {
        	bitmapPool = new BitmapPoolAdapter();
      	}
    }
	//创建内存缓存
    if (memoryCache == null) {
      	memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
	//创建磁盘缓存
    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
}
```

* 主要就是MemorySizeCalculator/BitmapPool/MemoryCache/DiskCache.Factory对象实例
* 这部分会在接下来作详细的解释

### 四、MemorySizeCalculator

* 详情请查看[三、Glide缓存原理之BitmapPool](..\..\..\设计思想解读开源框架库\图片加载\Glide原理分析\三、Glide缓存原理之BitmapPool.md)第六部分

### 五、BitmapPool

* 详情请查看[三、Glide缓存原理之BitmapPool](..\..\..\设计思想解读开源框架库\图片加载\Glide原理分析\三、Glide缓存原理之BitmapPool.md)

### 六、ActiveResources

* 详情请点击查看[Glide缓存原理之ActiveResources](设计思想解读开源框架库\图片加载\Glide原理分析\Glide缓存原理之ActiveResources)

### 七、MemoryCache

* 详情请点击查看[Glide缓存原理之LruCache](设计思想解读开源框架库\图片加载\Glide原理分析\Glide缓存原理之LruCache)

### 八、DiskCache

* 详情请点击查看[Glide缓存原理之LruCache](设计思想解读开源框架库\图片加载\Glide原理分析\Glide缓存原理之LruCache)