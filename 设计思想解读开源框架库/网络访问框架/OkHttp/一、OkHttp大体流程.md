[TOC]

## OkHttp大体流程

#### 转载

* [OkHttp大流程分析](https://juejin.cn/post/6844904137969106951)
* [彻底理解OkHttp - OkHttp 源码解析及OkHttp的设计思想](https://juejin.cn/post/6844903743951994887#heading-5)

### 一、前言

#### 1.1 前言

* 本文源码基于以下版本

  ```groovy
  implementation("com.squareup.okhttp3:okhttp:3.9.0")
  ```

#### 1.2 实例

* 用过OkHttp的小伙伴们，都知道先new一个OkHttpClient的builder，紧接着添加`Interceptor`、`NetWorkInterceptor`和连接超时等配置，然后通过build方法构建出OkHttpClient对象；
* 第二部分接着new了一个Request的builder对象，设置了url和header等相关参数，接着也是通过build方法生成了Request对象；
* 最后通过第一步OkHttpClient的newCall传入request，拿到了Call对象，接着通过Call的同步方法execute和异步方法enqueue拿到Response，这里文字计较多，我直接上一个大家熟悉的代码:

```java
OkHttpClient.Builder builder = new OkHttpClient.Builder();
builder.connectTimeout(10, TimeUnit.SECONDS);
builder.addInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Log.d(TAG, "Interceptor url:" + chain.request().url().toString());
        return chain.proceed(chain.request());
    }
});
builder.addNetworkInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Log.d(TAG, "NetworkInterceptor url:" + chain.request().url().toString());
        return chain.proceed(chain.request());
    }
});
OkHttpClient build = builder.build();
//创建请求
Request request = new Request.Builder()
        .url("https://www.baidu.com")
        .build();
Call call = build.newCall(request);
//异步方法
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        Log.d(TAG, "onFailure: " + e.getMessage());
    }
    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.d(TAG, "response:" + response.body().string());
    }
});
//同步方法
try {
	Response response = call.execute();
} catch (IOException e) {
	e.printStackTrace();
}
```

* 会在第二节对其进行详细分析

### 二、源码分析

#### 2.1 OkHttpClient

* 对于创建OkHttpClient而言，最好创建一个单例OkHttpClient实例，重复使用它。因为每个Client都有它自己的连接池connection pool和线程池thread pool，重用这些连接池和线程池可以有效的减少延迟和节约内存。

##### 2.1.1 OkHttpClient#成员变量

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
    ......
	//调度器 里面包含了线程池和三个队列（readyAsyncCalls：保存等待执行的异步请求)
  final Dispatcher dispatcher;
    
    //代理类，默认有三种代理模式DIRECT(直连),HTTP（http代理）,SOCKS（socks代理），这三种模式
  final @Nullable Proxy proxy;
    
    //协议集合，协议类，用来表示使用的协议版本，比如`http/1.0,`http/1.1,`spdy/3.1,`h2等
  final List<Protocol> protocols;
    
    //连接规范，用于配置Socket连接层。对于HTTPS，还能配置安全传输层协议（TLS）版本和密码套件
  final List<ConnectionSpec> connectionSpecs;
    
  final List<Interceptor> interceptors;//普通的拦截器
  final List<Interceptor> networkInterceptors;//netWork拦截器
  final EventListener.Factory eventListenerFactory;
    
    //代理选择类，默认不使用代理，即使用直连方式，当然，我们可以自定义配置，以指定URI使用某种代理，类似代理软件的PAC功能。
  final ProxySelector proxySelector;
    
  final CookieJar cookieJar;//Cookie的保存获取
    
    //缓存类，内部使用了DiskLruCache来进行管理缓存，匹配缓存的机制不仅仅是根据url，而且会根据请求方法和请求头来验证是否可以响应缓存。此外，仅支持GET请求的缓存。
  final @Nullable Cache cache;
    
  final @Nullable InternalCache internalCache;//内部缓存
    
    //Socket的抽象创建工厂，通过`createSocket来创建Socket
  final SocketFactory socketFactory;
    
    //安全套接层工厂，HTTPS相关，用于创建SSLSocket。一般配置HTTPS证书信任问题都需要从这里着手。对于不受信任的证书一般会提示javax.net.ssl.SSLHandshakeException异常。
  final @Nullable SSLSocketFactory sslSocketFactory;
    
    //证书链清洁器，HTTPS相关，用于从[Java]的TLS API构建的原始数组中统计有效的证书链，然后清除跟TLS握手不相关的证书，提取可信任的证书以便可以受益于证书锁机制。
  final @Nullable CertificateChainCleaner certificateChainCleaner;
    
    //主机名验证器，与HTTPS中的SSL相关，当握手时如果URL的主机名不是可识别的主机，就会要求进行主机名验证
  final HostnameVerifier hostnameVerifier;
    
    // 证书锁，HTTPS相关，用于约束哪些证书可以被信任，可以防止一些已知或未知的中间证书机构带来的攻击行为。如果所有证书都不被信任将抛出SSLPeerUnverifiedException异常。
  final CertificatePinner certificatePinner;
    
    //身份认证器，当连接提示未授权时，可以通过重新设置请求头来响应一个新的Request。状态码401表示远程服务器请求授权，407表示代理服务器请求授权。该认证器在需要时会被RetryAndFollowUpInterceptor触发。
  final Authenticator proxyAuthenticator;
    
  final Authenticator authenticator;//本地身份验证
  final ConnectionPool connectionPool;//连接池 复用连接
  final Dns dns;//域名
  final boolean followSslRedirects;//是否遵循SSL重定向
  final boolean followRedirects;//是否重定向
  final boolean retryOnConnectionFailure;//失败是否重新连接
  final int connectTimeout;//连接超时
  final int readTimeout;//读取超时
  final int writeTimeout;//写入超时
  final int pingInterval;
    ......
}
```

##### 2.1.2 常用的

* dispatcher：调度器 同步和异步处理类
* interceptors：普通的拦截器
* networkInterceptors：netWork拦截器
* connectTimeout：连接超时
* readTimeout：读取超时
* writeTimeout：写入超时

##### 2.1.3 概述

* 其中重要的是Dispatcher，他负责同步和异步去获取数据。
* 同时还有两种拦截器以及超时参数配置。两种拦截器会在后续详细介绍

#### 2.2 OkHttpClient#Builder

```java
OkHttpClient.Builder builder = new OkHttpClient.Builder();
builder.xxx进行设置
OkHttpClient build = builder.build();
```

* 一般创建的时候都这么使用，首先创建OkHttpClient.Builder，然后调用方法对其OkHttpClient.Builder的成员变量进行设置，最后调用OkHttpClient#build#Build方法进行设置

##### 2.2.1 OkHttpClient#Builder

```java
public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS; //默认支持的协议
      connectionSpecs = DEFAULT_CONNECTION_SPECS; //默认的连接规范
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault(); //默认的代理选择器，直连
      cookieJar = CookieJar.NO_COOKIES; //默认不进行管理Cookie
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE; //主机验证
      certificatePinner = CertificatePinner.DEFAULT; //证书锁，默认不开启
      proxyAuthenticator = Authenticator.NONE; //默认不进行授权
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool(); //连接池
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      //超时时间
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
}
```

* 对OkHttpClient#Builder的成员变量进行初始化，进行默认的配置

##### 2.2.2 OkHttpClient.Builder#xxx

* 这个主要是通过各种方法根据自己的需求修改OkHttpClient.Builder的成员变量
* 例如connectTimeout方法，修改OkHttpClient.Builder#connectTimeout的值

```java
  public Builder connectTimeout(long timeout, TimeUnit unit) {
      connectTimeout = checkDuration("timeout", timeout, unit);
      return this;
    }
```

##### 2.2.3 OkHttpClient#build

```java
public OkHttpClient build() {
      return new OkHttpClient(this);
}
```

* 调用如下的方法OkHttpClient

```java
  OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
    this.connectionSpecs = builder.connectionSpecs;
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.eventListenerFactory = builder.eventListenerFactory;
    this.proxySelector = builder.proxySelector;
    this.cookieJar = builder.cookieJar;
    this.cache = builder.cache;
    this.internalCache = builder.internalCache;
    this.socketFactory = builder.socketFactory;

    boolean isTLS = false;
    for (ConnectionSpec spec : connectionSpecs) {
      isTLS = isTLS || spec.isTls();
    }

    if (builder.sslSocketFactory != null || !isTLS) {
      this.sslSocketFactory = builder.sslSocketFactory;
      this.certificateChainCleaner = builder.certificateChainCleaner;
    } else {
      X509TrustManager trustManager = systemDefaultTrustManager();
      this.sslSocketFactory = systemDefaultSslSocketFactory(trustManager);
      this.certificateChainCleaner = CertificateChainCleaner.get(trustManager);
    }

    this.hostnameVerifier = builder.hostnameVerifier;
    this.certificatePinner = builder.certificatePinner.withCertificateChainCleaner(
        certificateChainCleaner);
    this.proxyAuthenticator = builder.proxyAuthenticator;
    this.authenticator = builder.authenticator;
    this.connectionPool = builder.connectionPool;
    this.dns = builder.dns;
    this.followSslRedirects = builder.followSslRedirects;
    this.followRedirects = builder.followRedirects;
    this.retryOnConnectionFailure = builder.retryOnConnectionFailure;
    this.connectTimeout = builder.connectTimeout;
    this.readTimeout = builder.readTimeout;
    this.writeTimeout = builder.writeTimeout;
    this.pingInterval = builder.pingInterval;

    if (interceptors.contains(null)) {
      throw new IllegalStateException("Null interceptor: " + interceptors);
    }
    if (networkInterceptors.contains(null)) {
      throw new IllegalStateException("Null network interceptor: " + networkInterceptors);
    }
  }
```

* 此方法主要就是把OkHttpClient.Builder的成员变量赋值给OkHttpClient的成员变量

##### 2.2.4 小结

* 总体来说，就是通过设置修改Builder的成员变量，然后通过build方法把Builder的成员变量赋值给OkHttpClient的成员变量
* 为什么设计Builder？可点击查看详情[Builder模式](架构能力\设计模式\Builder模式.md)

#### 2.3 Request

##### 2.3.1 Request#成员变量

```java
public final class Request {
  final HttpUrl url;//请求路径
  final String method;//请求方式
  final Headers headers;//请求头的key value配置
  final @Nullable RequestBody body;//请求体的key value配置
  final Object tag;;//请求tag标识
    
  private volatile CacheControl cacheControl; // Lazily initialized.
}  
```

* 主要就是对请求的参数(路径/请求方式/body等)进行各种配置

##### 2.3.2 Request#Builder

* 这个和上面的机制是一样的，这里就不赘述了，最终设置的还是Request的成员变量

#### 2.4 OkHttpClient#newCall

```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

* 可以看到主要是调用了RealCall的newRealCall方法，同时把request传进去了

#### 2.5 RealCall#newRealCall

```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```

* 主要就是创建了一个RealCall对象，并返回了

### 三、源码解析-异步请求

#### 3.1 RealCall#enqueue

* RealCall请求分为同步和异步
* 因为一般都使用异步方式，所以着重讲解异步方式

```java
  @Override public void enqueue(Callback responseCallback) {
      //分析1：同步锁将executed置为true
    synchronized (this) {
        //如果是executed了，则直接抛异常
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
      //分析2：调用dispatcher的enqueue方法，并且将AsyncCall作为参数
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

* 分析1：确保每个RealCall只能被执行一次不能重复执行
* 分析2：重点是调用dispatcher的enqueue方法，并且将AsyncCall作为参数。这个是真正进行异步处理的地方。其中client.dispatcher()就是OkHttpClient的成员变量dispatcher，在OkHttpClient#Builder中进行初始化的

```java
 public Dispatcher dispatcher() {
    return dispatcher;
  }
```

#### 3.2 Dispatcher#enqueue(AsyncCall)

##### 3.2.1 Dispatcher#enqueue(AsyncCall)

```java
 //TODO 执行异步请求
    synchronized void enqueue(AsyncCall call) {
        //TODO 同时请求不能超过并发数(64,可配置调度器调整)
        //TODO okhttp会使用共享主机即 地址相同的会共享socket
        //TODO 同一个host最多允许5条线程通知执行请求
        if (runningAsyncCalls.size() < maxRequests &&
                runningCallsForHost(call) < maxRequestsPerHost) {
            //TODO 加入运行队列 并交给线程池执行
            runningAsyncCalls.add(call);
            //TODO AsyncCall 是一个runnable，放到线程池中去执行，查看其execute实现
            executorService().execute(call);
        } else {
            //TODO 加入等候队列
            readyAsyncCalls.add(call);
        }
    }

```

* 主要就是把AsyncCall对象加入到runningAsyncCalls中，然后使用线程池去执行

* 里面有很多是Dispatcher的成员变量

##### 3.2.2 Dispatcher#成员变量

```java
//同时能进行的最大请求数
private int maxRequests = 64;

//同时请求的相同HOST的最大个数 SCHEME :// HOST [ ":" PORT ] [ PATH [ "?" QUERY ]]
private int maxRequestsPerHost = 5;

//线程池
private @Nullable ExecutorService executorService;

//异步等待队列
//双端队列，支持首尾两端 双向开口可进可出，方便移除
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

//正在进行的异步队列
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
```

* 其中线程池通过ThreadPoolExecutor方法创建，参数表示
  * 核心线程数：0
  * 最带线程数：Integer.MAX_VALUE
  * 空闲线程存活时间：60
  * 单位：TimeUnit.SECONDS秒
  * 阻塞队列：SynchronousQueue
  * 线程工厂类：Util.threadFactory("OkHttp Dispatcher", false)

```java
public synchronized ExecutorService executorService() {
	if (executorService == null) {
    	executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
}
```

#### 3.3 RealCall#AsyncCall

* 由于把AsyncCall放入了ExecutorService对象的execute方法中，所以研究一下AsyncCall

```java
 final class AsyncCall extends NamedRunnable {
```

* AsyncCall继承自NamedRunnable，那么看一下NamedRunnable

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}

```

* 可以看到NamedRunnable继承自Runnable
* 有[线程池基本解析](必备Java知识/并发编程/线程池/线程池基本解析.md)一章可知，线程池的execute传入的是Runnable对象，肯定会执行Runnable的run方法，由于AsyncCall没有重写run方法，所以调用的是NamedRunnable的run方法，可以看到NamedRunnable#run方法中主要的就是执行的抽象方法execute，那么它的实现类就是AsyncCall，所以实际上最后执行的是AsyncCall实现的execute方法

#### 3.4 RealCall#AsyncCall#execute

```java
final class AsyncCall extends NamedRunnable {
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        //责任链模式
        //拦截器链  执行请求
          //分析1
        Response response = getResponseWithInterceptorChain();
        //回调结果
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        //移除队列
          //分析2
        client.dispatcher().finished(this);
      }
    }
  }

```

* 分析1：可以看到真正去获取数据的是通过getResponseWithInterceptorChain()方法拿到返回结果Response的
* 分析2：执行client.dispatcher().finished(this)去移除队列，即Dispatcher的finish方法

#### 2.10 Dispatcher#finish

```java
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }
```

* 调用finished方法

```java
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
        int runningCallsCount;
        Runnable idleCallback;
        synchronized (this) {
            //calls 移除队列
            //分析1
            if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
            //检查是否为异步请求，检查等候的队列 readyAsyncCalls，如果存在等候队列，则将等候队列加入执行队列
            //分析2
            if (promoteCalls) promoteCalls();
            // 运行队列的数量
            runningCallsCount = runningCallsCount();
            idleCallback = this.idleCallback;
        }
        //闲置调用
        if (runningCallsCount == 0 && idleCallback != null) {
            idleCallback.run();
        }
    }

```

* 分析1：calls就是传入的runningAsyncCalls，首先从runningAsyncCalls去移除队列
* 分析2：判断是否有等待队列，如果有把等待队列加入到执行队列中

```java
private void promoteCalls() {
        //检查 运行队列 与 等待队列
        if (runningAsyncCalls.size() >= maxRequests) return; 
        if (readyAsyncCalls.isEmpty()) return; 

        //将等待队列加入到运行队列中
        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
            AsyncCall call = i.next();
            //相同host的请求没有达到最大，加入运行队列
            if (runningCallsForHost(call) < maxRequestsPerHost) {
                i.remove();
                runningAsyncCalls.add(call);
                executorService().execute(call);
            }

            if (runningAsyncCalls.size() >= maxRequests) return; 
        }
    }
```

* 通过各种判断，如果符合条件，就把等待队列中的任务移到运行队列中去

#### 2.11 getResponseWithInterceptorChain

* 这是真正去获取数据的方法：执行网络请求和返回响应结果

```java
// 核心代码 开始真正的执行网络请求
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    //责任链
    List<Interceptor> interceptors = new ArrayList<>();
    //在配置okhttpClient 时设置的intercept 由用户自己设置
    interceptors.addAll(client.interceptors());
    //负责处理失败后的重试与重定向
    interceptors.add(retryAndFollowUpInterceptor);
    //负责把用户构造的请求转换为发送到服务器的请求 、把服务器返回的响应转换为用户友好的响应 处理 配置请求头等信息
    //从应用程序代码到网络代码的桥梁。首先，它根据用户请求构建网络请求。然后它继续呼叫网络。最后，它根据网络响应构建用户响应。
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //处理 缓存配置 根据条件(存在响应缓存并被设置为不变的或者响应在有效期内)返回缓存响应
    //设置请求头(If-None-Match、If-Modified-Since等) 服务器可能返回304(未修改)
    //可配置用户自己设置的缓存拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //连接服务器 负责和服务器建立连接 这里才是真正的请求网络
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      //配置okhttpClient 时设置的networkInterceptors
      //返回观察单个网络请求和响应的不可变拦截器列表。
      interceptors.addAll(client.networkInterceptors());
    }
    //执行流操作(写出请求体、获得响应数据) 负责向服务器发送请求数据、从服务器读取响应数据
    //进行http请求报文的封装与请求报文的解析
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //创建责任链
      //分析1
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    //执行责任链
      //分析2
    return chain.proceed(originalRequest);
  }
```

* 从上述代码中，可以看出都实现了Interceptor接口，这是okhttp最核心的部分，采用责任链的模式来使每个功能分开，每个Interceptor自行完成自己的任务，并且将不属于自己的任务交给下一个，简化了各自的责任和逻辑。

* 这其实是一种设计模式：责任链模式。请点击查看详细[责任链模式](架构能力\设计模式\责任链模式.md)
* 大体流程：把各种Interceptor的实现类对象都放入到List中了，然后再分析1把list作为参数创建了Interceptor.Chain对象chain，然后再分析2执行了chain的proceed方法。

#### 3.7 RealInterceptorChain#构造方法

```java
public RealInterceptorChain(List<Interceptor> interceptors, 
                            StreamAllocation streamAllocation,
                            HttpCodec httpCodec, RealConnection connection, 
                            int index, Request request, Call call,
                            EventListener eventListener, int connectTimeout, 
                            int readTimeout, int writeTimeout) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
    this.call = call;
    this.eventListener = eventListener;
    this.connectTimeout = connectTimeout;
    this.readTimeout = readTimeout;
    this.writeTimeout = writeTimeout;
}
```

* 就是对成员变量进行赋值，重点是interceptors，它包含了各种的Interceptor的实现类对象

```java
private final List<Interceptor> interceptors;
```

#### 2.13 RealInterceptorChain#proceed

* 调用了Interceptor.Chain的proceed方法，由于Interceptor.Chain是一个接口，所以调用的是其实现类RealInterceptorChain的proceed方法

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    @Nullable Connection connection();

    Call call();
      
	......
  }
}
```

* RealInterceptorChain#proceed

```java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
}
```

* 调用proceed方法

```java
public Response proceed(Request request, StreamAllocation streamAllocation, 
		HttpCodec httpCodec,RealConnection connection) throws IOException {
    //分析2
    if (index >= interceptors.size()) throw new AssertionError();
    calls++;
	......
	//分析1
    //创建新的拦截链，链中的拦截器集合index+1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation,
										httpCodec,connection, index + 1, request, call, 
										eventListener, connectTimeout, readTimeout,
        								writeTimeout);
    //执行当前的拦截器
    //如果在配置okhttpClient时没有设置intercept默认是先执行RetryAndFollowUpInterceptor拦截器
    Interceptor interceptor = interceptors.get(index);
    //执行拦截器
    Response response = interceptor.intercept(next);
	......
    return response;
  }
```

* 分析1：默认的index是0，然后index+1等于0，所以获取的是第一个加入list的拦截器。从中我们大概可以猜想处理，调用intercept方法之后会继续调用proceed方法，这样index就变成1+1=2，就会执行下一个拦截器的intercept方法，然后一直这样重复调用proceed，直到所有的拦截器都执行完毕，那样就会在分析2出抛出异常

* 那么我们可以任意查看一个拦截器的intercept方法去看一下，那么我们带着猜想查看一下CacheInterceptor 的intercept

#### 3.9 CacheInterceptor#intercept

* CacheInterceptor代码比较长，我们一步一步的来进行分析。

##### 3.9.1 无网络部分

* 首先我们先分析上部分代码当没有网络的情况下是如何处理获取缓存的。

```java
@Override public Response intercept(Chain chain) throws IOException
  {
	//分析1 获取request对应缓存的Response 如果用户没有配置缓存拦截器 cacheCandidate == null
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    //执行响应缓存策略
    long now = System.currentTimeMillis();
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //果networkRequest == null 则说明不使用网络请求
    Request networkRequest = strategy.networkRequest;
    //获取缓存中（CacheStrategy）的Response
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }
    //分析2 缓存无效 关闭资源
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    //分析3 networkRequest == null 不实用网路请求 且没有缓存 cacheResponse == null  返回失败
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    //分析4 不使用网络请求 且存在缓存 直接返回响应
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    
}
```

* 分析1：如果用户自己配置了缓存拦截器，cacheCandidate = cache.Response 获取用户自己存储的Response,否则 cacheCandidate = null;同时从CacheStrategy 获取cacheResponse 和 networkRequest

* 分析2：如果cacheCandidate ！= null 而 cacheResponse == null 说明缓存无效清楚cacheCandidate缓存。

* 分析3：如果networkRequest == null 说明没有网络，cacheResponse == null 没有缓存，返回失败的信息，责任链此时也就终止，不会在往下继续执行。

* 分析4：如果networkRequest == null 说明没有网络，cacheResponse != null 有缓存，返回缓存的信息，责任链此时也就终止，不会在往下继续执行。

##### 3.9.2 有网络部分

```java
    //执行下一个拦截器
    Response networkResponse = null;
    try {
        //分析1
        //此处调用了 chain.proceed方法，用来执行下一个拦截器
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    //网络请求 回来 更新缓存
    //如果存在缓存 更新
    if (cacheResponse != null) {
      //TODO 304响应码 自从上次请求后，请求需要响应的内容未发生改变
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    //缓存Response
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
        }
      }
    }

    return response;
  }
```

* 分析1：主要就是执行下一个拦截器，递归往下执行

* 责任链执行完毕后，会返回最终响应数据，如果缓存存在更新缓存，如果缓存不存在加入到缓存中去。

#### 3.10 责任链的好处

* 这样就体现出了，责任链这样实现的好处了，当责任链执行完毕，如果拦截器想要拿到最终的数据做其他的逻辑处理等，这样就不用在做其他的调用方法逻辑了，直接在当前的拦截器就可以拿到最终的数据。

#### 3.11 流程图

* 大体上就是拦截器一层层往里调用，然后返回是一层层往外返回。类似于DFS
* 第一个拦截器最先调用，然后逐步往下调用，最后一个拦截器最后被调用
* 最后一个拦截器最先返回数据，然后逐步往上返回数据，第一个拦截器最后返回数据。
* 每个拦截器都能拿到返回数据，然后自己对其数据进行各种处理，这就是它的好处。

<img src="..\..\..\images\设计思想解读开源框架库\网络访问框架\OkHttp\OkHttp责任链流程图.png" alt="OkHttp责任链流程图" style="zoom:80%;" />

* **更正：应该是先ConnectInterceptor后是NetworkInterceptors。对于3.9版本而言**

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

###  四、源码分析-同步请求

* 同步请求使用的比较少，这里就简单带过

#### 4.1 RealCall#execute()

* 调用RealCall的execute去进行同步请求

```java
//同步执行请求 直接返回一个请求的结果
  @Override public Response execute() throws IOException {
      //分析1
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    //调用监听的开始方法
    eventListener.callStart(this);
    try {
      //交给调度器去执行
        //分析2
      client.dispatcher().executed(this);
      //获取请求的返回数据
        //分析3
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      //执行调度器的完成方法 移除队列
      client.dispatcher().finished(this);
    }
  }
```

* 分析1：synchronized (this) 避免重复执行，上面的文章部分有讲。
* 分析2：client.dispatcher().executed(this); 实际上调度器只是将call 加入到了同步执行队列中。

```java
 synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

* 分析3：调用getResponseWithInterceptorChain方法去获取数据，这个方法上面做了详细的说明了
* 分析4：去移除队列，这个方法上面也做了详细的说明了

### 五、实例

### 六、总结

#### 6.1 OkHttp总体流程图

<img src="..\..\..\images\设计思想解读开源框架库\网络访问框架\OkHttp\OkHttp总体流程图.png" alt="OkHttp总体流程图" style="zoom: 50%;" />

* OkhttpClient 实现了Call.Fctory,负责为Request 创建 Call；

* RealCall 为Call的具体实现，其enqueue() 异步请求接口通过Dispatcher()调度器利用ExcutorService实现，而最终进行网络请求时和同步的execute()接口一致，都是通过 getResponseWithInterceptorChain() 函数实现

* getResponseWithInterceptorChain() 中利用 Interceptor 链条，责任链模式 分层实现缓存、透明压缩、网络 IO 等功能；最终将响应数据返回给用户。