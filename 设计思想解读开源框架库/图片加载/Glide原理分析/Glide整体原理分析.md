[TOC]

## Glide整体原理分析

#### 转载

* [探索 Glide 原理](https://juejin.cn/post/6882536990400020494#heading-87)

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eef35148bc364510b979eee27abb1ef6~tplv-k3u1fbpfcp-watermark.webp)

## 前言

### 1. Glide 基本用法

接下来的讲解将基于 Glide 目前的最新版本 4.11。

Glide 的使用特别简单，首先添加依赖。

![添加依赖.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/930824fc4be945bda50d9682dd49e194~tplv-k3u1fbpfcp-zoom-1.image)

然后调用下面这三个方法。

![三个方法.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48649296742a433388fe6fb48cebb961~tplv-k3u1fbpfcp-zoom-1.image)

- with()

  可以传 Applicaiton、Activity 、Fragment 与 view 等类型的参数，加载图片的请求会与该参数的生命周期绑定在一起。

- load()

  可以传图片的网络地址、Drawable 等。

- into()

  一般传 ImageView 。

### 2. 内容概览

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7fdec498dca4ff9b23f42725e621542~tplv-k3u1fbpfcp-zoom-1.image)

Glide 加载图片大致可分为三个步骤。

1. 发起加载图片请求

   当我们用 into() 方法加载图片时，就是发起了一次图片加载请求；

2. 执行解码任务

   我们在 load() 方法中设置的图片来源会传到 DecodeJob 中，DecodeJob 就会被 Engine 提交到线程池中开始执行；

3. 加载图片

   当 DecodeJob 对图片解码完成后，就会把图片加载到 ImageView 中；

接下来会以这三个步骤为基础来展开 Glide 的图片加载流程，下面是每个大节讲解的内容。

##### 1. 四步启动解码任务

第一大节将会讲解从我们调用 into() 方法到启动 DecodeJob 的 run() 方法的过程中，Glide 做了哪些事情，包含了 Request、Target 、 Engine 和 DecodeJob 等内容。

##### 2. 六步加载图片

第二大节会讲解当 DecodeJob 获取到图片数据后，会怎么处理图片数据，也就是解码、加载图片和编码的过程，包括 ModelLoader、ResourceDecoder、Transformation、ResourceTranscoder 以及 ResourceEncoder 的实现。

##### 3. Glide 缓存原理

这里会讲 Glide 的图片缓存相关的实现，包括内存缓存、磁盘缓存、BitmapPool 以及 ArrayPool 等内容。

##### 4. Glide 初始化流程与配置

这一节会讲解 Glide 的初始化流程，包括 Glide 调用配置的方式、AppGlideModule 的两个方法中用到的 Registry 和 GlideBuilder 在 Glide 中的作用。

##### 5. Glide 图片加载选项

Glide 开放了非常多的图片加载选项，我们不一定全都用得上，但是了解这些选项，可以让我们在需要的时候能调用对应的选项，不用再自己实现一遍。

## 1. 四步启动解码任务

这一节我们先来看下从 into() 到启动 DecodeJob 的过程中涉及哪些对象。

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a39c4f04a714f91bfd9ebbcbc65801f~tplv-k3u1fbpfcp-watermark.image)

从我们调用 into() 方法加载图片到启动解码任务 DecodeJob，大致分为 4 个步骤，涉及下面 4 个对象。

1. 加载请求 Request
2. 加载目标 Target
3. 加载引擎 Engine
4. 解码任务 DecodeJob

下面我们围绕这 4 个对象看看 Glide 的解码任务的启动流程。

### 1.1 Request

#### 1.1.1 请求构建器 RequestBuilder

##### 1. into()

对于我们发起的图片加载请求，Glide 会把这个请求封装为 Request，而 RequestBuilder 就是 Request 的构建器。

当我们用 into() 方法加载图片时，调用的其实是 RequestBuilder 的 into() 方法，这个方法会创建一个 Request ，并把 Request 传给请求管理器。

##### 2. Model

Glide 会把我们在 load() 中传入的图片来源数据封装为 Model ，而这个 Model 具体就是 RequestBuilder 中的 model 字段，类型为 Object 。

##### 3. 加载选项

RequestBuilder 继承了 BaseRequestOptions 抽象类，我们平时用的 centerCrop() 等方法大部分都是 BaseRequestOptions 的方法，关于图片加载选项在后面会讲到。

#### 1.1.2 请求管理器 RequestManager

RequestManager 有下面几个特点。

- 绑定生命周期
- 监听网络状态
- 创建请求构建器
- 启动请求

##### 1. 绑定生命周期

在 Glide 中，一个 Context 对应一个 RequestManager，当我们调用 with() 方法时，RequestManager 会用对应的 Context 创建一个 RequestManagerFragment 。

RequestManagerFragment 是一个无布局的 Fragment，主要是用来做生命周期关联的，当这个 Fragment 感知到 Activity 的生命周期发生变化时，就会告诉请求管理器，让它去做暂停请求、继续请求和取消请求等操作。

如果我们用的是 ApplicationContext 加载某张图片，那就意味着这次图片加载操作的生命周期是与应用的生命周期绑定的。

##### 2. 监听网络状态

RequestManager 中有一个网络连接监听器 RequestManagerConnectivityListener ，它实现了ConnectivityListener 接口，每次网络状态切换时，RequestManager 就会重启所有的图片加载请求。

##### 3. 创建请求构建器

我们在加载图片时调用的 load() 方法是 RequestManager 的方法，调用这个方法其实是创建了一个请求构建器 RequestBuilder，RequestManager 中有很多创建 RequestBuilder 的方法，比如 asDrawable()、asBitmap() 、asFile() 等，这些方法对应着不同泛型参数的 RequestBuilder 。

load() 方法支持下面这些类型的参数。

- Bitmap
- Drawable
- String
- Uri
- URL
- File
- Integer（resourceId）
- byte[]
- Object

##### 4. 启动请求

RequestManager 的 track() 方法调用了目标跟踪器 TargetTracker 的 track() 方法，还调用了请求跟踪器 RequestTracker 的 runRequest() 方法 。

- TargetTracker

  TargetTracker 实现了 LifecycleListener ，它会根据页面生命周期播放和暂停动画，比如暂停 Gif 动画。

- RequestTracker

  RequestTracker 的 runRequest() 方法调用了 Request.begin() 方法。

  在 Request 的 begin() 方法中会获取 View 的尺寸，获取到了尺寸后就会调用 Engine 的 load() 方法启动图片加载请求。

#### 1.1.3 Request 的 6 种状态

![SingleRequest 6 种状态.png](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

前面讲到的 Request 具体就是 SingleRequest ，SingleRequest 中有一个 Status 枚举类，包含了请求的 6 种状态。

##### 1. 待运行 PENDING

当我们通过 into() 创建了一个 SingleRequest 后，该 Request 就进入了待运行状态。

##### 2. 已清除 CLEARED

每次我们用 into() 方法加载图片时，RequestManager 都会先看下我们传入的 Target 是否有对应的 Request ，如果有的话就会调用该 Request 的 clear() 方法释放资源，这时 Request 就进入了已清除状态。

##### 3. 待测量 WAITING_FOR_SIZE

当 RequestManager 调用 RequestTracker 的 runRequest() 方法后，RequestTracker 就会调用 Request 的 begin() 方法，这时请求就进入了待测量状态。

##### 4. 运行中 RUNNING

在 SingleRequest 的 begin() 方法中，调用了 Target 的 getSize() 方法获取 ImageView 的尺寸，获取到尺寸后，SingleRequst 会调用 Engine 的 load() 方法启动图片加载请求，这时 Request 就进入了运行中状态。

##### 5. 已完成 COMPLETE

当 Engine 从内存中加载到资源，或者通过解码任务加载到资源后，就会调用 SingleRequest 的 onResourceReady() 方法，这时 Request 就进入了已完成状态。

##### 6. 失败 FAILED

当解码任务 DecodeJob 在处理图片的过程中遇到异常时，就会调用 EngineJob 的 onLoadFailed() 方法，然后 EngineJob 会调用 SingleRequest 的 onLoadFailed() 方法，这时 SingleRequest 就进入了失败状态。

#### 1.1.4 三种占位图

![占位图显示流程.png](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

我们在加载图片时，可以设置 placeholder、error 和 fallback 三种占位图。

- placeholder

  图片加载完成前显示的占位图；

- error

  图片加载失败时显示的占位图；

- fallback

  图片来源为空时显示的占位图；

使用占位图时，要注意占位图是不会使用 Transformation 进行变换的，如果你想弄个圆角或圆形的占位图，可以用 submit().get() 获取对应变换后的占位图的 Drawable 对象，然后传到对应的占位图设置方法中。

#### 1.1.5 Request 相关问题

下面是几个跟 Request 相关的问题，看看你能不能答得上来。

1. 我们平时用 Glide 加载图片调用的 into() 方法是哪个类的方法？
2. 当设备的网络状态发生变化时，是谁负责重启图片加载请求？
3. Request 有哪几种状态？这些状态是如何流转的？
4. Glide 有几种占位图？分别在什么时候显示？

### 1.2 Target

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

当我们调用 into() 方法，传入 ImageView 后，Glide 会把 ImageView 转化为 Target ，下面我们来看下不同 Target 的作用。

#### 1.2.1 ImageViewTarget

##### 1. SizeDeterminer

ImageViewTarget 继承了 ViewTarget ，在 ViewTarget 中有一个用来获取尺寸的 SizeDeterminer ，SizeDeterminer 的 getSize() 方法拿到的尺寸，是把 ImageView 的内边距 padding() 去掉后的尺寸。

在 Glide 中，宽高分为请求宽高和原始宽高 ，而 SizeDeterminer 拿到的尺寸就是请求宽高，Glide 会根据请求宽高对图片进行缩放操作，以减少不必要的内存消耗。

##### 2. OnPreDrawListener

当 Request 获取 View 的尺寸失败时，ViewTarget 会通过 ViewTreeObserver 的 OnPreDrawListener 的回调来获取 View 的尺寸，然后再传给 Request。

##### 3. setResource()

ImageViewTarget 主要有 BitmapImageViewTarget 和 DrawableImageViewTarget 两个子类，它们两个的区别就在于它们的 setResource() 方法。

- BitmapImageViewTarget

  setResource() 用的是 ImageView 的 setImageBitmap() 方法；

- DrawableImageViewTarget

  setResource() 用的是 ImageView 的 setImageDrawable() 方法；

#### 1.2.2 RequestFutureTarget

##### 1. submit()

FutureTarget 是一个实现了 Future 和 Target 接口的接口，它只有一个 RequestFutureTarget 子类 ，当我们用 submit() 方法获取 Glide 加载好的图片资源时，就是创建了一个 RequestFutureTarget 。

##### 2. Waiter

RequestFutureTarget 是用 wait/notify 的方式来实现等待和通知的，这两个是 Object 的方法，Request 中有一个 Waiter ，当 DecodeJob 加载到图片后，RequestFutureTarget 就会让 Waiter 发出通知，这时我们的 get() 方法就能获取到返回值了。

这就是为什么我们用 RequestFutureTarget 的 get() 方法获取图片时，要把这个操作放在子线程运行。

#### 1.2.3 CustomTarget

给不是 View 的 Target 加载图片时，Glide 都把它作为 CustomTarget 。

##### 1. PreloadTarget

预加载 Target 。

当我们调用 preload() 选项预加载图片时，Glide 会把图片交给 PreloadTarget 处理，当 PreloadTarget 接收到图片资源后，就会让 RequestManager 把该请求的资源释放掉。

因为不需要等待资源加载完成，所以我们在用 preload() 预加载图片时，不用像 submit() 一样在子线程中执行。

##### 2. AppWidgetTarget

桌面组件 Target 。

当 AppWidgetTarget 接收到处理好的图片资源后，会把它设置给 RemoteView ，然后通过桌面组件管理器 AppWidgetManager 更新桌面组件。

##### 3. DelayTarget

GifTarget。

这是加载 Gif 图片时要用到的 Target ，关于 Glide 加载 Gif 图片的流程在后面会讲到。

##### 4. NotificationTarget

通知栏 Target 。

这个 Target 有一个 setBitmap 方法，会把图片设置给通知栏的 RemoteView ，然后通过 NotificationManager 更新通知栏中的通知。

#### 1.2.4 Target 相关问题

1. ImageViewTarget 是用什么来获取请求宽高的？
2. 为什么在用 submit() 获取图片时，要放在子线程中执行？
3. 使用 preload() 预加载图片时，用的是哪个 Target ？该 Target 获取到资源会后做什么？

### 1.3 Engine

下面我们来看一些与 Engine 相关的实现。

- Engine 的作用
- Key 的作用
- Resource 的作用
- BitmapPool

#### 1.3.1 Engine 的作用

Engine 是 Glide 的图片加载引擎，是 Glide 中非常重要的一个类，下面我们来看下 Engine 的作用。

##### 1. load()

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/24537c4453364244abf93828d8c3d429~tplv-k3u1fbpfcp-watermark.webp)

前面讲到了当我们调用 into() 方法时，就是间接调用了 Request.begin() 方法，而 Request 的 begin() 方法又调用了 Engine 的 load() 方法。

在 load() 方法中，Engine 会先用 EngineKeyFactory 创建资源标识符 Key，然后用这个 Key 去内存缓存中加载资源。

如果从内存中找到了资源，Engine 就会直接把资源回传给 Resource，如果没有加载到资源，Engine 就会创建并启动新的 EngineJob 和解码任务 DecodeJob。

##### 2. EngineKeyFactory

EngineKeyFactory 是 Engine 中一个负责生产 EngineKey 的工厂，EngineKey 是引擎任务资源标识符，关于什么是 Key 后面进一步讲。

在 Engine 启动新的任务加载图片前，会先通过 EngineKeyFactory 创建一个 EngineKey，然后让 DecodeJob 把资源与 EngineKey 进行绑定，这里说的绑定，其实就是把 model 放到 EngineKey 中。

##### 3. 回收资源

Engine 中有一个资源回收器 ResourceRecycler ，Resource 接口中有一个 recycle() 方法，关于 Resource 我们后面再讲。

这里只要知道，当 SingleRequest 被清除，比如在 into() 方法中发现 Target 已经有对应的 Request 时，Request 就会让 Engine 释放资源，具体做释放资源操作的就是 ResourceRecycler。

##### 4. 磁盘缓存提供器

LazyDiskCacheProvider 是 Engine 中的一个静态内部类，是磁盘缓存 DiskCache 的提供器，DiskCache 是一个接口，关于 DiskCache 的实现我们后面再讲。

##### 5. 启动新的解码任务

当 Engine 从内存中找不到对应的 Key 的资源时，就会启动新的解码任务。

Engine 会用加载任务工厂 EngineJobFactory 构建一个加载任务 EngineJob，然后再构建一个解码任务 DecodeJob。

EngineJob 这个名字看起来很霸气，但是实际上它并没有做什么事情，它只是 Engine 与 DecodeJob 之间沟通的桥梁。

当构建了 EngineJob 和 DecodeJob 后，Engine 就会把 DecodeJob 提交到线程池 GlideExecutor 中。

#### 1.3.2 Key

前面讲到了 Engine 会通过 EngineKeyFactory 创建资源标识符 Key ，那什么是 Key ？

Key 是 Glide 中的一个接口，是图片资源的标识符。

##### 1. 避免比较有误

Glide 的内存缓存和磁盘缓存用的都是 Glide 自己实现的 LruCache，LruCache 也就是最近最少使用缓存算法（Least Recently Used），LruCache 中有一个 LinkedHashMap ，这个 HashMap 的 Key 就是 Key 接口，而 Value 则是 Resource 接口。

在用对象作为 HashMap 的 Key 时，要重写 equals() 和 hashCode() 方法。

如果不重写这两个方法，那么当两个 Key 的内存地址不同，但是实际代表的资源相同时，使用父类 Object的 hasCode() 直接用内存地址做比较，那么结果会是不相等。

此外 Object 的 equals() 方法也是拿内存地址作比较，所以也要重写。

比如下面就是 ResourceCacheKey 的 equals() 判断逻辑。

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

##### 2. Key 实现类

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

下面是几个实现了 Key 接口的类。

- DataCacheKey

  原始图片数据标识符。

- ResourceCacheKey

  处理后的图片资源标识符。

- AndroidResourceSignature

  Android 资源标识符。当我们传入 into() 方法的图片是 R.drawable.xxx 时，Glide 就会把它封装为 AndroidResourceSignature 。

- ObjectKey

  通用资源标识符。

  可以说除了 App 自带的 Android 资源以外的图片资源都会用 ObjectKey 作为标识符，比如本地图片文件。

- EngineKey

  引擎资源标识符。

  这个 Key 是 Engine 对其他 Key 的封装，这时传进来的 Key 是以签名（Signature）的身份存在 EngineKey 中的。

#### 1.3.3 Resource

前面讲到了 Engine 会通过 ResourceRecycler 来回收资源，而 ResourceRecycler 调用了 Resource 的 recycle() 方法。

可能你想起来 Bitmap 就有一个可以回收图片内存的 recycle() 方法，没错，Glide 回收 Bitmap 的方式就是用的 Bitmap 自带的 recycle() 方法，但是这个过程又比这复杂一些。

Resource 是一个接口，其中一个实现类是 BitmapResource ，也就是位图资源，比如网络图片就会转化为 BitmapResource。

在 BitmapResource 中有一个位图池 BitmapPool，这是 Glide 用来复用 Bitmap 的一个接口，具体的实现类是 LruBitmapPool 。

在 BitmapResource 的 recycle() 方法中，会把对应的 Bitmap 通过 put() 方法放到 BitmapPool 中，关于 BitmapPool 在讲 Glide 缓存原理时会进一步讲。

#### 1.3.4 Engine 相关问题

1. Engine 的 load() 方法首先会做什么？
2. 为什么 Key 要重写 hashCode() 和 equals() 方法？
3. 负责回收 Resource 的是哪个类？
4. 加载 Drawable 资源时，会转化为哪种 Key？

### 1.4 DecodeJob

前面讲到了 Engine 在缓存中找不到资源时，就会创建新的加载任务 EngineJob 和新的解码任务 DecodeJob ，然后让 EngineJob 启动 DecodeJob。

DecodeJob 实现了 Runnable 接口，EngineJob 启动 DecodeJob 的方式就是把它提交给 GlideExecutor，如果我们没有调整磁盘缓存策略的话，那默认用的就是 diskCacheExecutor ，关于 GlideExecutor 在第 4 大节会讲，下面我们先看下 DecodeJob 的实现。

#### 1.4.1 runWrapped()

DecodeJob 的 run() 方法只是对 runWrapped() 可能遇到的异常进行了捕获，而 runWrapped() 方法会根据不同的运行理由 RunReason 运行不同的数据生成器。

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

##### 1. 三种运行理由

runWrapped() 会根据下面三种运行理由来执行解码任务。

- INITAILIZE

  从缓存中获取数据并解码；

- SWITCH_TO_SOURCE_SERVICE

  从来源获取数据后再进行解码；

- DECODE_DATA

  当获取数据的线程与 DecodeJob 的线程不同时，比如使用了 OkHttp-Integration 时，DecodeJob 会直接对数据进行解码；

##### 2. 初始化

当运行理由为默认状态 INITIALIZE 时，DecodeJob 会从磁盘中获取图片数据并进行解码。

##### 3. 从来源获取数据

当 DecodeJob 从缓存中获取不到数据时，就会把运行理由改为 SWITCH_TO_SOURCE_SERVICE ，也就是从来源获取数据，然后运行来源数据生成器 SourceGenerator 。

##### 4. 对检索到的数据进行解码

DecodeJob 通过数据生成器获取到数据后，就会调用 decodeFromRetrievedData() 方法来对检索到的数据进行解码。

#### 1.4.2 DecodeJob 数据获取流程

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ac82310713e4a32aea9d3bd6ced7071~tplv-k3u1fbpfcp-watermark.image)

在 DecodeJob 的 getNextStage() 方法中，会根据当前的解码步骤 stage 来判断进行什么操作。

DecodeJob 把提取数据分为了 6 个阶段，这 6 个阶段是 Stage 枚举类中的值。

##### 1. INITIALIZE

初始化。

当解码处于这个阶段时，DecodeJob 会根据磁盘缓存策略，判断是否要从磁盘缓存中获取处理过的图片资源，是的话就用 ResourceCacheGenerator 获取图片资源，当用 ResourceCacheGenerator 获取到 Resource 后，就会开始对资源进行解码。

如果磁盘缓存策略设定了不从缓存中获取 Resource，那就会切换到 RESOURCE_CACHE 阶段。

##### 2. RESOURCE_CACHE

从缓存中获取处理过的图片资源。

当解码处于这个阶段时，DecodeJob 会根据磁盘缓存策略，判断是否要从磁盘缓存中获取未处理过的图片原始数据，是的话就用 DataCacheGenerator 获取图片数据。

##### 3. DATA_CACHE

从缓存中获取原始数据。

如果磁盘缓存策略设定了不获取缓存中的图片资源和原始数据 ，又或者是获取不到数据，DecodeJob 那就会切换到 DATA_CACHE 阶段。

如果我们在加载图片时调用了 onlyRetrieveFromCache(true) ，那么 DecodeJob 就会不会切换到 SOURCE 阶段从来源获取数据，而是会切换到 FINISH 阶段结束数据获取流程。

否则就会切换到 SOURCE 阶段。

##### 4. SOURCE

从图片来源获取原始数据。

如果 DecodeJob 在 RESOURCE_CACHE 和 DATA_CACHE 阶段都没有拿到图片数据，那就会用 SourceGenerator 从图片来源获取图片数据。

##### 5. ENCODE

编码。

当磁盘缓存策略设定了要对图片资源进行缓存时，那么在获取到数据后，DecodeJob 就会用 ResourceDecoder 对资源进行编码，也就是把图片放到磁盘缓存中。

##### 6. FINISH

结束。

#### 1.4.3 三种数据生成器

当 DecodeJob 切换阶段后，会调用 getNextGenerator() 切换不同阶段对应的生成器，这里说的生成器，指的是 DataFetcherGenerator 接口。

DataFetcherGenerator 不是像名字说的那样用来创建 DataFetcher 的，DataFetcherGenerator 与 DataFetcher 是通过 ModelLoader 来关联的。

DataFetcherGenerator 会通过 ModelLoader 构建数据封装对象 LoadData ，然后通过 LoadData 中的 DataFetcher 来加载数据。

LoadData 是 ModelLoader 的内部类，它有来源标识符 Key 和 DataFetcher 两个字段。

在 ModelLoader 中最重要的就是 buildLoadData() 方法，不同类型的 Model 对应的 ModelLoader 所创建出来的 LoadData() 也不同。

下面我们来看下 DataFetcherGenerator ，这个接口中最重要的方法是 startNext() ，具体实现了这个接口有下面三个类。

- SourceGenerator

  来源数据生成器。

- DataCacheGenerator

  原始缓存数据生成器。

- ResourceCacheGenerator

  缓存资源生成器。

以 SourceGenerator 为例，我们来看下 startNext() 方法的处理流程。

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

##### 1. 是否获取到了需要缓存的数据

当 SourceGenerator 加载完数据后，会再次进入 startNext() 方法，这时就获取到了需要缓存的数据。

##### 2. 是否保存原始数据

如果磁盘缓存策略设定了要保存图片的原始数据，就用数据提取器加载数据，否则就直接把图片加载给 Target 。

##### 3. 加载数据

当需要保存原始数据或数据有加载路径时，SourceGenerator 就会根据 Model 的类型，使用对应的 DataFetcher 来提取数据，比如从网络上下载图片。

##### 4. 是否保存原始数据

当 SourceGenerator 获取到数据后，会再次判断是否要保存原始数据，否则就直接把图片加载给 Target 。

##### 5. 编码

当 SourceGenerator 从 DataFetcher 中拿到数据后，会再走一遍 startNext() 方法，然后用编码器 Encoder 对数据进行编码，也就是把图片放到磁盘缓存中。

##### 6. 从磁盘中获取数据

当 SourceGenerator 把数据保存到磁盘后，不会直接加载图片，而是从磁盘中拿这张图片，然后再进行加载。

#### 1.4.4 onResourceDecoded()

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

当 DecodeJob 调用 ResourceDecoder 的 decode() 方法，并且获取到编码结果后，会调用 onResourceDecoded() 方法应用变换选项以及初始化编码管理器。

##### 1. 应用变换选项

对于处理过的 Resource，onResourceDecoded() 不会再次进行变换，否则就会对图片进行变换操作。

##### 2. 回收图片资源

当对资源应用了变换选项后，DecodeJob 会把原来的资源回收掉，因为这个资源接下来也用不上了。

##### 3. 缓存变换后图片资源

onResourceDecoded() 方法中，会根据磁盘缓存策略判断是否要对资源进行编码，如果要进行编码的话，会根据不同的编码策略创建不同的 Key 。

Glide 有 SOURCE 和 TRANSFORMED 两种编码策略，分别代表对原始数据进行编码和对变换后资源进行编码。

- SOURCE

  GIF 编码器 GifDrawableEncoder 中用的编码策略；

- TRANSFORMED

  位图编码器 BitmapEncoder 中用的编码策略；

##### 4. 初始化编码管理器

创建好 Key 后不会直接对图片进行编码，而是会修改编码管理器的 Key ，等到转码完成后再用 ResourceEncoder 进行编码。

#### 1.4.5 DecodeJob 相关问题

1. DecodeJob 会根据哪些理由来执行任务？
2. DecodeJob 提取数据的过程分为哪几个阶段？
3. DataFetcherGenerator 有哪些实现类？
4. Glide 有几种编码策略？

## 2. 六步加载图片

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/123e8c1372ab4f69be576dc66f218fa5~tplv-k3u1fbpfcp-watermark.webp)

看完了解码任务启动流程，下面我们来看下当 DecodeJob 获取到图片数据后是怎么处理这些数据的，在文章的开头已经讲过 Glide 解码大致的 5 步，这里再补充一个，就是在把图片加载到 Target 后，DecodeJob 会通过 ResourceEncoder 把图片保存到本地。

其中关于 Target 在 1.2 小节已经讲过，下面就不再多讲了，我们来看下其他的对象。

Glide 对图片解码的过程涉及下面 6 个概念。

1. 数据来源（Model）
2. 原始数据（Data）
3. 资源（Resource）
4. 变换后资源（TransformedResource）
5. 转码后资源（TranscodedResource）
6. 目标（Target）

##### 1. 数据来源（Model）

Glide 会以 Model 的形式封装图片来源 ，Model 可以是 URL、本地文件和网络图片等类型。

##### 2. 原始数据（Data）

Glide 把数据源转换为Model 后，会把它加工成原始数据 Data ，一般就是输入流 InputStream ，Glide 会把这些输入流封装为 Data ，而 ModelLoader 则负责从 Data 获取原始数据。

##### 3. 资源（Resource）

获取到原始数据后，Glide 会用资源解码器 ResourceDecoder 对原始数据进行解码，比如把输入流 InputStream 解码为 Bitmap，解码后的资源就是 Resource 。

##### 4. 变换后资源（TransformedResource）

Glide 会根据我们的变换选项处理 Resource ，比如用 centerCrop() 裁剪就是一种变换，变换后的 Resource 就叫 TransformedResource ，负责转换的就是 Transformation 。

##### 5. 转码后资源（TranscodedResource）

Glide 除了能加载静态图片，还能加载 Gif 动态图，解码后的 Bitmap 和 Gif 的类型不是统一的，为了统一处理静态和动态图片，Glide 会把 Bitmap 转换为 GlideBitmapDrawable ，而负责转码的角色则是 ResourceTranscoder 。

##### 6. 目标（Target）

Glide 最终会把图片显示到目标 Target 上，比如 ImageView 对应的就是 ImageViewTarget 。

### 2.1 ModelLoader

ModelLoader 是一个接口，负责创建 LoadData ，它有两个泛型参数 Model 和 Data。

- Model

  代表图片来源的类型，比如图片的网络地址的 Model 类型为 String ；

- Data

  代表图片的原始数据的类型，比如网络图片对应的类型为 InputStream ；

##### 1. Factory

在 DataFetcherGenerator 获取图片数据时，会调用 ModelLoaderRegistry 的 getModelLoaders() 方法，这个方法中会根据 model 的类型用 MultiModelLoaderFactory 生成对应的 ModelLoader，比如能够解析字符串的 ModelLoader 就有 7 个，关于 ModelLoaderRegistry 在后面讲 Glide 配置的时候会讲到。

此外每一个 ModelLoader 的实现类中都定义了一个实现了 ModelLoaderFactory 接口的静态内部类 。

##### 2. handles()

一个 Model 对应这么多 ModelLoader，每个 ModelLoader 加载数据的方式都不同，这时候就要用 handles() 方法了。

ModelLoader 接口有 handles() 和 buildLoadData() 两个方法，handles() 用于判断某个 Model 是否能被自己处理，比如 HttpUriLoader 的 handles() 会判断传进来的字符串是否以 http 或 https 开头，是的话则可以处理。

##### 3. buildLoadData()

ModelLoader 之间是存在嵌套关系的，比如 HttpUriLoader 的 buildLoadData() 方法就是调用的 HttpGlideUrlLoader 的 buildLoadData() 方法，HttpGlideUrlLoader 会创建一个 HttpUrlFetcher ，然后把它放到 LoadData() 中。

LoadData 是 ModelLoader 中定义的一个类，它只是放置了图片来源的 Key 和要用来提取数据的 DataFetcher ，没有其他方法。

### 2.2 ResourceDecoder

DataFetcherGenerator 使用 ModelLoader 构建完数据后，就会用 DataRewinder 对数据进行重绕，也就是重置数据，比如 InputStreamRewinder 就会调用 RecyclableBufferedInputStream 的 reset() 方法重置输入流对应的字节数组的位置。

ResourceDecoder 是一个接口，有非常多的实现类，比如网络图片对应的解码器为 StreamBitmapDecoder ，StreamBitmapDecoder 的 decode() 方法调用了降采样器 Downsampler 的 decode() 方法，下图是 Downsampler 的解码逻辑。

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3d597be068e49ec9743a135d311ccb8~tplv-k3u1fbpfcp-zoom-1.image)

##### 1. 设置目标宽高

除非我们通过 override() 方法把尺寸改为 Target.SIZE_ROGINAL ，否则 Glide 默认会把 ImageView 的大小作为加载图片的目标宽高。

##### 2. 计算缩放后宽高

根据不同的变换选项计算缩放后宽高。

##### 3. 创建空 Bitmap

根据计算后的目标宽高创建一个空的 Bitmap 。

##### 4. 使用 BitmapFactory 解码

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1b3caa2243d46ce9cd5073356f5bd25~tplv-k3u1fbpfcp-zoom-1.image)

Downsampler 的解码方式用的是 ImageReader 的 decodeBitmap() 方法，而 ImageReader 又调用了 BitmapFactory 的 decodeStream() 方法，BitmapFactory 最终调用的是 SkImageDecoder 的 decode() 方法。

##### 5. 把 Bitmap 放入 BitmapPool 中

在前面讲 Resource 的时候讲到了 BitmapResource 中有一个 BitmapPool，这个 BitmapPool 是由 Downsampler 传过去的，而 Downsampler 的 BitmapPool 是由 Glide 创建并传进来的。

### 2.3 Transformation

Transformation 是一个接口，它有一个 transform() 方法，这个方法是在 DecodeJob 中调用的，当 DecodeJob 发现数据源不是缓存中的 Resource 时，就会调用变换选项的 transform() 方法。

Transformation 的其中一个实现类是 BitmapTransformation，我们平时调用的 centerCrop() 就是 BitmapTransformation 的子类，centerCrop() 选项对应的是 CenterCrop 类，它实现了 Transformation 接口，具体的变换实现在 TransformationUtils 中。

##### 1. Matrix

以 centerCrop() 为例，TransformationUtils 的 centerCrop() 方法会先创建一个 Matrix 矩阵，然后根据传进来的 Bitmap 计算 Matrix 的缩放比例和平移坐标。

##### 2. drawBitmap()

配置好 Matrix 后，就会根据目标宽高创建一个空的目标 Bitmap ，然后把原始 Bitmap、目标 Bitmap 和 Matrix 传给 Canvas 的 drawBitmap() 方法，然后返回 Canvas 处理好的图片。

### 2.4 ResouceTranscoder

ResourceTranscoder 是一个接口，是 Glide 中的资源转码器，它有两个泛型参数 Z 和 R ，分别代表需要进行原始类型和转码目标类型。

比如 BitmapDrawableTranscoder 的原始类型是 Bitmap，转码目标类型是 BitmapDrawable，在BitmapDrawableTranscoder 的 transcode() 方法中，会把 Bitmap 转换为 BitmapDrawable ，以便 Target 进行处理。

### 2.5 ResourceEncoder

ResourceEncoder 是一个接口，是 Glide 中的资源编码器，ResourceEncoder 有好几个实现类，比如网络图片对应的编码器为 StreamEncoder。

在转码完成后，DecodeJob 会先把图片加载到 Target 中，然后用 ResourceEncoder 对图片进行编码，比如 StreamEncoder 的编码操作就是把输入流 InputStream 转化为图片文件，然后保存到本地。

### 2.6 图片加载相关问题

1. 图片加载分为哪几步？
2. ModelLoader 有哪些泛型参数？分别代表什么？
3. ResourceDecoder 会用哪个类进行解码？最终进行解码的哪个类？
4. 真正进行变换操作的是哪个类？变换操作使用了哪些类？
5. Glide 怎么对图片数据进行编码？

## 3. Glide 缓存原理

Glide 使用了三级缓存机制，图片的缓存分为内存、磁盘和来源，也就是从内存获取不到图片时，再去磁盘获取图片，从磁盘获取不到图片时，再从图片来源获取图片。

**三级缓存的优势在于节省流量和内存**，如果不用三级缓存，每次都从服务端获取图片的话，图片消耗的流量就会非常多，如果把所有图片都放在内存的话，那就有可能发生 OOM 。

下面我们来看下 Glide 的内存缓存原理、磁盘缓存原理和磁盘缓存策略。

### 3.1 Glide 内存缓存原理

前面提到 Engine 的 load() 方法会先在内存缓存中查找 Key 对应的资源，没有的话再启动新的解码任务。

这里说的内存缓存就是 MemoryCache，MemoryCache 是一个接口，它的实现类是 LruResourceCache。

LruResourceCache 不仅实现了 MemoryCache 接口，而且还是 LruCache 的子类，具体的内存缓存实现是在 LruCache 中。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59d455e074c04b7680e12471522521c4~tplv-k3u1fbpfcp-zoom-1.image)

在 LruCache 的 put() 方法中，首先会判断要保存的元素大小是否大于缓存最大值，如果是的话，则不进行保存，如果不是的话，则把当前容量加上元素的大小，并把该元素放入缓存。

LruCache 比较特别的就是它的 trimToSize() 方法和 LinkedHashMap 的 accessOrder 属性。

##### 1. trimToSize()

LruCache 在用 put() 方法保存新的元素时，它会通过 trimToSize() 方法移除最近最少使用的元素。

##### 2. accessOrder

LruCache 中是用 LinkedHashMap 保存数据的，并且这个 LinkedHashMap 的 accessOrder 的值为 true，也就是每一次获取 LinkedHashMap 中的元素时，这个元素都会被移到链表的尾端。

### 3.2 Glide 磁盘缓存原理

Glide 是用 DiskCache 保存图片文件的，DiskCache 是一个接口，这个接口中还定义了 Factory 和 Writer 两个接口，Writer 只是对 ResourceEncoder 的封装。

下面我们就来看看 DiskCache 和 DiskCache.Factory 的具体实现。

#### 3.2.1 DiskLruCache

DiskCache 有两个实现类， DiskCacheAdapter 和 DiskLruCacheWrapper，DiskCacheAdapter 只是一个空实现。

从名字可以看得出来 DiskLruCacheWrapper 是对 DiskLruCache 的封装，具体的实现是在 DiskLruCache 中，DataCacheGenerator 和 ResourceCacheGenerator 都是用的 DiskLruCache 来获取磁盘缓存数据的。

##### 1. Entry

和 LruCache 一样，DiskLruCache 中也有一个 LinkedHashMap ，这个 HashMap 的 Key 的类型为 String，Value 的类型为 Entry，从缓存中获取到的图片文件会放在 Entry的 cleanFiles 字段中。

##### 2. Editor

当图片加载进入编码阶段时，DecodeJob 会通过编码管理器调用 DiskLruCacheWrapper 的 put() 方法保存图片文件。

在 DiskLruCacheWrapper 的 put() 方法中，会通过 DiskCache 的缓存编辑器 Editor 获取图片文件，获取到图片文件后，就会用 Writer 把文件写入本地，写完后再调用 Editor 的 commit() 方法，把清理缓存的回调提交到清理线程池中。

##### 3. 清理资源

DiskLruCache 中有一个执行清理资源任务的线程池，线程池的线程数最多为 1，

这个线程池要执行的任务为 cleanupCallback 回调，这个回调会执行 trimToSize() 方法，为的就是把最近最少使用的文件清除掉。

#### 3.2.2 DiskCache.Factory

在 DiskCache 中有一个 Factory 工厂接口，这个接口用在了 Engine 的 LazyDiskCacheProvider 中。

在 Factory 接口中，定义了默认的磁盘缓存大小为 250M，默认的缓存目录名称为 "image_manager_disk_cache" 。

Factory 主要有下面 2 个实现类。

- ExternalPreferredCacheDiskCacheFactory

  用的是 getExternalCacheDir() 。

  对应的目录是 /data/user/0/包名/cache/image_manager_disk_cache。

- InternalCacheDiskCacheFactory

  用的是 context.getCacheDir() 。

  对应的目录是 /data/user/0/包名/cache 。

默认情况下 Glide 用的是 InternalCacheDiskCacheFactory ，如果想把图片放在外部缓存目录的话，可以在自定义的 GlideModule 设置 DiskCache 。

![GlideConfig.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e878ce47bfee4182b24c9b3586c98ad8~tplv-k3u1fbpfcp-zoom-1.image)

### 3.3 Glide 磁盘缓存策略

在加载图片时，我们可以用 diskCacheStratgy() 方法设置图片在磁盘的缓存策略，这个选项传入的参数类型为抽象类 DiskCacheStrategy。

磁盘缓存策略涉及到 Glide 的数据源类型 DataSource 和编码策略 EncodeStratefy，编码策略前面讲过了，下面我们先来看看数据源 DataSource。

#### 3.3.1 五种数据源

Glide 中定义了下面 5 种数据源 DataSource。

- LOCAL

  设备上有的数据，比如 App 内置的 Drawable 也属于 LOCAL ；

- REMOTE

  从服务端拿到的数据；

- DATA_DISK_CACHE

  从缓存中取出来的原始数据；

- RESOURCE_DISK_CACHE

  从缓存中取出来的图片资源；

- MEMORY_CACHE

  从内存缓存中取出来的数据；

#### 3.3.2 四个抽象方法

DiskCacheStrategy 有下面 4 个抽象方法，这个 4 个方法的返回值都是布尔值。

- isDataCacheable()
- isResourceCacheable()
- decodeCachedResource()
- decodeCacheData()

##### 1. isDataCacheable()

是否保存图片的原始数据。

DecodeJob 中用到的 SourceGenerator 在从图片来源获取到数据后，会根据这个方法判断是否保存图片的原始数据。

##### 2. isResourceCacheable()

是否保存解码后的图片数据。

当资源解码器对图片数据进行解码后，DecodeJob 就会根据这个方法的返回值决定是否保存该 Resource 。

##### 3. decodeCachedResource()

是否对缓存的解码后的图片数据进行解码。

在 DecodeJob 的 getNextStage() 中，会根据这个方法的返回值判断，如果返回值为 false，意味着跳过 RESOURCE_CACHE 步骤，也就是不对缓存中处理过的图片资源进行处理。

##### 4. decodeCachedData()

是否对缓存的原始数据进行解码。

在 DecodeJob 的 getNextStage() 方法中，会根据这个方法的返回值判断，如果该值为 false，意味着跳过 DATA_CACHE 步骤，也就是不对缓存中的原始图片数据进行处理。

#### 4.3.2 五种缓存策略

Glide 定义好的磁盘缓存策略有下面 5 种，默认为 AUTOMATIC。

- AUTOMATIC
- ALL
- NONE
- RESOURCE
- DATA

##### 1. AUTOMATIC

- isDataCacheable()

  只保存网络图片的原始数据；

- isResourceCacheable()

  只保存数据源为 DATA_DISK_CACHE 或 LOCAL ，并且编码策略为 TRANSFORMED 的图片资源；

- decodeCachedResource()

  true；

- decodeCachedData()

  true；

##### 2. ALL

- isDataCacheable()

  只保存网络图片的原始数据；

- isResourceCacheable()

  不保存数据源为 RESOURCE_DISK_CACHE 和 MEMORY_CACHE 的图片资源；

- decodeCachedResource()

  true；

- decodeCachedData()

  true；

##### 3. DATA

- isDataCacheable()

  不保存数据源为 DATA_DISK_CACHE 或 MEMORY_CACHE 的图片资源；

- isResourceCacheable()

  false；

- decodeCachedResource()

  false；

- decodeCachedData()

  true；

##### 4. RESOURCE

- isDataCacheable()

  false；

- isResourceCacheable()

  不保存数据源为 RESOURCE_DISK_CACHE 和 MEMORY_CACHE 的图片资源；

- decodeCachedResource()

  true；

- decodeCachedData()

  false；

##### 5. NONE

所有方法的返回值都为 false。

### 3.4 BitmapPool

##### 1. 减少 Bitmap 占用的内存

BitmapResource 的 BitmapPool 用的就是 GlideBuilder 中的 BitmapPool，Downsampler 在解码后，会把图片放入 BitmapPool 中，当 BitmapResource 被回收时，也会把 Bitmap 放到 BitmapPool 中。

具体需要用到 BitmapPool 中的 Bitmap 的地方在 TransformationUtils 中，TransformationUtils 在进行变换前会从 BitmapPool 中获取之前保存的 Bitmap。

之所以要这么做，是因为每一次变换都需要创建一个 Bitmap ，BitmapPool 就是为了复用这个 Bitmap 占用的内存，这样下次要做变换操作时，可以用同一个 Bitmap 就进行复用，以减少内存使用。

比如对于 RecyclerView 中的图片，它们的大小是一样的，没必要在变换时为每张图片都创建一个新的 Bitmap。

##### 2. LruPoolStrategy

BitmapPool 是一个接口，实现类为 LruBitmapPool ，具体的逻辑在 LruPoolStrategy 中。

LruPoolStrategy 也是一个接口，它的实现类为 SizeConfigStrategy，从 LruPoolStrategy 的名字可以看得出来，BitmapPool 用的是 LruCache 来保存 Bitmap 的。

在 LruPoolStrategy 中，会根据 Bitmap 的大小和编码选项，把 Bitmap 放到 GroupedLinkedHamp 中。

### 3.5 ArrayPool

和 BitmapPool 一样，ArrayPool 用的也是 LruCache，也是为了减少不必要的内存浪费。

比如在输入流编码器 StreamEncoder 中，当把输入流转化为文件时，需要创建一个新的字节数组，如果不用 ArrayPool，而图片是在列表中加载的，那就会创建很多不必要的的字节数组。

### 3.6 Glide 缓存相关问题

1. LruCache 的缓存实现比较特别的是哪两点？
2. DiskLruCache 有哪 3 个特点？
3. DiskCacheStrategy 定义了几种磁盘缓存策略？

## 4. Glide 初始化流程与配置

### 4.1 Glide 初始化流程

在看 Glide 的配置前，我们先来看下 Glide 的初始化流程，因为读取配置就是在初始化的过程中读取的。

#### 4.1.1 with()

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebb6d7e6884b4d8789210a8397a8208d~tplv-k3u1fbpfcp-zoom-1.image)

当我们调用 Glide.with() 方法时，Glide 会先用 getRetriever() 方法获取请求管理器检索器，在这个方法中还会用 get() 方法获取 Glide 实例，获取不到的话就会初始化 Glide 。

#### 4.1.2 initializeGlide()

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3782a8787ae849ad8d481868965639bf~tplv-k3u1fbpfcp-zoom-1.image)

我们可以在 AndroidManifest 中声明 GlideModule，也可以用 @GlideModule 注解声明 GlideModule，走的都是上面这个流程。

##### 1. 应用选项

Glide 有一个 ApppModuleGenerator，它会把读取我们设定的 AppGlideModule 中的配置，然后生成一个 GeneratedAppGlideModuleImpl 配置。

然后用反射读取这个配置，读取到配置后，就会应用我们在 applyOptions() 中给 GlideBuilder 设置的选项。

如果不用生成加反射的话读取配置的话，Glide 并不知道我们会把配置叫什么，放哪里。

##### 2. 创建实例

应用选项后，就会创建一个 Glide 实例。

##### 3. 注册组件

创建完实例后，就会把实例的 registry 传到 registerComponents() 中，也就是我们修改编解码逻辑的地方。

##### 4. 注册回调

Glide 实现了 ComponentCallback 用于监听内存状态，这里的注册回调就是调用 ApplicationContext 的 registerComponentCallbacks() 方法。

### 4.2 Registry

Glide 有一个登记处 Registry ，它包含了下面这些 Registry 。

- 数据加载器登记处 ModelLoaderRegistry
- 编码器登记处 EncoderRegistry
- 资源解码器登记处 ResourceDecoderRegistry
- 资源编码器登记处 ResourceEncoderRegistry
- 数据重绕器登记处 DataRewinderRegistry
- 转码器登记处 DataRewinderRegistry
- 图片头部信息解析器登记处 ImageHeaderRegistry

在 Glide 的构造方法中，会把所有的编码、解码和数据加载等逻辑通过 Registry 的 append() 方法登记到 Registry 中，我们可以在 AppGlideModule 的 registerComponents() 方法中获取到 registry 实例，通过这个实例就可以替换掉对应的实现。

##### 1. registerComponents()

比如获取网络图片默认用的是 HttpUrlFetcher ，HttpUrlFetcher 是用的 HttpURLConnection 来获取图片数据的，我们可以在 registerComponents() 方法中，把 HttpUrlFetcher 替换为 OkHttp 。

![registerComponents().png](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

##### 2. Entry

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

之所以要用 Class 来替换 ModelLoader ，是因为 ModelLoaderRegistry 的 append() 方法会用来源类型（Model）、原始数据类型（Data）和 ModelLoaderFactory 来创建不同类型的 Entry ，这些 Entry 会保存在 MultiModelLoaderFactory 工厂中。

当 DataFetcherGenerator 通过 ModelLoader 获取数据时，则会通过 model 的 Class 信息来获取 ModelLoader，除了 ModelLoaderRegistry，其他的 Registry 中也有 Entry。

### 4.3 GlideBuilder

GlideBuilder 就是 Glide 的构建器，它包含了下面这些数据。

- 线程池
  - 图片来源线程池 sourceExecutor
  - 磁盘缓存线程池 diskCacheExecutor
  - 动画线程池 animationExecutor
- 内存大小计算器 memorySizeCalculator
- 网络状态监听器工厂 connectivityMonitorFactory
- 请求选项工厂 defaultRequestOptionsFactory
- 请求管理器工厂 requestManagerFactory
- 请求监听器列表 defaultRequestListeners
- 位图池 bitmapPool
- 数组池 arrayPool
- 内存缓存 MemoryCache
- 磁盘缓存工厂 diskCacheFactory
- 引擎 Engine

上面这些字段大多数都是可以在 AppGlideModule 的 applyOptions() 方法中，调用 GlideBuilder 的 setXXX() 方法来替换实现的，下面我们主要看下 Glide 线程池和内存大小计算器。

#### 4.3.1 Glide 线程池

GlideExecutor 是 Glide 的线程池实现，Glide 中有下面 4 种线程池。

##### 1. SourceExecutor

对图片来源解码的任务的线程池，线程数为最多为 4 ，最小为设备的 CPU 核数。

##### 2. unlimitedSourceExecutor

如果我们在加载图片时调用了 useUnlimitedSourceGeneratorsPool() 选项，那 Glide 就会用这个无线程数限制的线程池来获取图片。

##### 3. DiskCacheExecutor

对磁盘缓存数据解码的任务的线程池，线程数为 1 。

##### 4. AnimationExecutor

这也是用来从图片来源获取数据的线程池，而不是用来播放 GIF 动画的线程池，当我们加载图片时调用了 useAnimationPool(true) ，那在获取图片数据时 EngineJob 就会把 DecodeJob 放到 AnimationExecutor 中。

如果设备 CPU 核数大于等于 4 ，那 AnimationPool 线程数就是 2 ，否则就是 1 。

##### 5. 自定义线程池

Glide 对于线程池只允许用 GlideBuilder 中的 Builder 来设置参数，GlideExecutor.Builder 支持下面几个参数。

- setThreadTimeoutMillis()

  设置线程存活时间；

- setThreadCount()

  设置线程数；

- setUncaughtThrowableStrategy()

  设置异常处理策略，有三种可以选，也可以自己自定义，默认为 LOG。

  - IGNORE

    忽略异常；

  - LOG

    打印异常日志；

  - THROW

    抛出异常；

- setName()

  设置线程名称；

![applyOptions().png](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

#### 4.3.2 内存大小计算器

MemorySizeCalculator 负责计算 BitmapPool 、ArrayPool 和 MemoryCache 的大小。

##### 1. BitmapPool 大小

BitmapPool 大小为一屏可容纳的最高图片质量的大小，比如 1080 * 1920 * 4 ≈ 7.9M 。

##### 2. ArrayPool 大小

默认为 4M，如果系统版本低于 19 则为 2M。

##### 3. 内存缓存大小

内存缓存大小为两屏可容纳的最高图片质量的大小，比如 1080 * 1920 * 2 * 4 ≈ 15.8M 。

##### 4. maxSizeMultiplier

MemorySizeCalculator 在计算 BitmapPool 和 MemoryCache 大小时，会通过 getMaxSize() 方法，用 ActivityManager 获取 memoryClasss，然后用 memoryClass 的值乘以 maxSizeMultiplier，maxSizeMultiplier 默认为 0.4。

memoryClass 就是获取应用可用内存大小，比如我的 VIVO 手机给应用分配的可用内存为 256M，以我的手机为例，256 * 0.4 = 102.4 ，也就是默认情况下， BitmapPool 、ArrayPool 和 MemoryCache 的大小最多不会超过 102.4M。

当 BitmapPool 、ArrayPool 和 MemoryCache 的大小加起来大于最大值时，会按这个最大值重新计算 BitmapPool 和 MemoryCache 的大小。

##### 5. 自定义内存大小计算方式

和 GlideExecutor 一样，MemorySizeCalculator 也有一个 Builder，支持下面这些参数的设置。

- setMemoryCacheSizeScreens(screens)

  设置内存缓存大小为几屏的大小；

- setBitmapPoolScreens(screens)

  设置 BitmapPool 大小为几屏的大小；

- setArrayPoolSize(sizeBytes)

  设置 ArrayPool 大小；

- setMaxSizeMultiplier()

  设置最大值乘数大小；

### 4.4 初始化流程与配置相关问题

1. 什么是 Registry ？
2. 在哪里可以替换 Glide 的数据加载逻辑？
3. 在哪里可以修改 Glide 的内存使用计算方式？
4. Glide 有几种线程池？
5. BitmapPool 和 MemoryCache 的大小是怎么计算的？

## 5. Glide 图片加载选项

##### 1. placeholder(drawable)

如果用户打开 App 的时候，本来应该显示图片的控件，由于网络原因等了好几秒都没加载出来，这样用户体验就不好，所以我们可以加上一张占位图，这样用户就知道图片等下就出来了。

这里要注意的是，占位图只能是 App 内置图片，不能是网络图片，否则无网络的时候它就没作用了。

##### 2. error(drawable)

当用户的网络出现错误，图片加载失败时，一直显示占位图，用户就会一直等待，如果等了半天都没加载出来，用户就会觉得我们的 App 有问题。

这时候我们可以用一张错误占位图，这样用户就知道有可能是网络出问题了，切换一下网络，又或者是主动联系开发者。

如果没有设置这个参数的话，出错时会显示 placeholder 中传的占位图。

##### 3. fallback()

设置当数据模型为空，也就是我们传入 load() 中的值为空时要显示的图片，没有设置 fallback 会显示错误占位图，连错误占位图也没设置就会显示 placeholder 占位图。

##### 4. override(width, height)

如果我们不想让 Glide 把图片按 ImageView 的大小进行缩放，我们可以用这个方法来设置加载的目标宽高。

##### 5. fitCenter()

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d69458672c24b4789bd619b22d70ea2~tplv-k3u1fbpfcp-watermark.image)

fitCenter 是一个图片裁剪选项，用于把图片尺寸限定在 ImageView 内并居中，这样图片就能完全显示。

选择这个选项后，当图片的宽高比和 ImageView 的宽高比不同时，ImageView 就不会被填满。

##### 6. centerCrop()

当我们使用了 centerCrop ()，并且图片宽高比与 ImageView 不同时，Glide 会裁剪中间的部分，以填满 ImageView ，这时图像就不是完全显示的了。

##### 7. centerInside()

与 fitCenter 类似，不同的是，当 ImageView 的尺寸为 wrap_content 时，fitCenter() 会把图片放大，而 centerInside() 则会保持原图大小。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d29145fb093f4259812bd49743aa345f~tplv-k3u1fbpfcp-watermark.image)

##### 8. transform(Transformation)

设置变换选项，比如旋转选项 Rotate 和圆角选项 RoundedCorners 没有对应的方法可以直接设置，就可以用这个方法传进去。

如果想要同时多种变换选项，也要从这个参数传进去，比如 transform(CenterCrop(), RoundedCorners()) ，这样创建的就是多重变换 MultiTransformation ，否则只有后面设置的变换选项会起效。

##### 9. dontTransform()

禁止变换，调用这个方法后会删除之前设定的变换选项。

##### 10. skipMemoryCache(boolean)

跳过内存缓存。

默认情况下 Glide 会把图片放在内存缓存中，如果我们不想要让某张图片保留在内存缓存中，比如加载高清原图时，可以把这个值改为 true 。

##### 11. diskCacheStrategy(strategy)

磁盘缓存策略，Glide 自带了 5 种磁盘缓存策略，默认为 AUTOMATIC。

- ALL
- NONE
- DATA
- RESOURCE
- AUTOMATIC

##### 12. onlyRetrieveFromCache(boolean)

如果传 true ，表示不从图片来源获取数据，只从缓存中读取数据。

##### 13.priority

设置加载图片的优先级，比如运营图的优先级就比其他图片高，Glide 的加载优先级有下面四种。

- IMMEDIATE

  立即加载；

- HIGH

  高优先级；

- NORMAL

  正常优先级；

- LOW

  低优先级；

##### 14. sizeMultiplier(float)

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

按比例缩放图片。

##### 15. encodeFormat(CompressFormat)

设置下载或缓存的图片的编码类型，比如 JPEG、PNG、WEBP。

如果没有设置 encodeFormat，并且图像有透明通道，那 Glide 默认会把以 PNG 的方式保存图片，否则就是 JPEG。

##### 16. encodeQuality(int)

设置下载或缓存的图片的编码质量，这个参数会传到 Bitmap.compress() 方法中。

##### 17. frame(long)

取视频中某一帧的作为图片。

##### 18. format(DecodeFormat)

设置解码格式，比如 ARGB_8888、RGB_565 。

##### 19. timeout(milliseconds)

设置网络图片的请求超时，默认为 2500 毫秒。

##### 20. transition(options)

设置过渡动画，这里的 options 传的是 TransitionOptions ，这是一个抽象类，它有三个子类。

- GenericTransitionOptions.with()

  自定义动画。

- DrawableTransitionOptions.withCrossFade()

  设置图片淡入动画。

- BitmapTransitionOptions.withCrossFade()

  如果调用了 asBitmap() 方法，就要用这个过渡动画。

##### 21. dontAnimate()

不播放 GIF 动画 。

##### 22. apply(options)

应用选项。

![apply().png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df4d01858a4d443f95c1029566d320e0~tplv-k3u1fbpfcp-zoom-1.image)

##### 23. listener(RequestListener)

设置请求监听器 ，这个接口有下面两个回调。

- onResourceReady()

  图片加载完成回调。

- onLoadFailed()

  图片加载失败回调。

##### 24. asBitmap()

把解码类型改为 Bitmap 。

这个方法并不是 BaseRequestOptions 提供的，而是 RequestManager 中的，但是对于外部使用者来说，这就是一个加载选项，不同的是，这个选项需要在 load() 方法前调用。

##### 25.asFile()

把解码类型改为 File，也就是用 submit().get() 获取到的类型为 File 。

##### 26. submit()

我们可以用 submit() 方法来获取我们想要的图片类型，比如文件、Bitmap 和 Drawable 等，但是要注意的是这个方法要在子线程中执行。

![submit().png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f043dd78717043e9abd103b60d8d29f4~tplv-k3u1fbpfcp-zoom-1.image)

##### 27. downloadOnly()

设定只下载图片，不加载图片。

这个选项就是 asFile() + diskCacheStrategy(DATA) + priority(LOW) + skiptMmoeryCache(true) 。

##### 28. download()

这个方法相当于是用 download(image) 替换 downloadOnly().load(image) 。

##### 29. preload(width, height)

![preload().png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05299eacd34b474c87b5b2ac3549777e~tplv-k3u1fbpfcp-zoom-1.image)

预加载图片，可以用来提前下载一些下一次启动应用的时候会用到的图片，比如闪屏页广告。

和 downloadOnly() 的区别在于不需要在子线程调用。