[TOC]

## OkHttp连接池复用

#### 参考

* [面试官:Okhttp连接池是咋实现的?](https://juejin.cn/post/6898145227765186567#heading-0)
* [Okhttp连接池](https://www.jianshu.com/p/32de43ce0252)
* [OkHttp拦截器 (四) ConnectInterceptor拦截器分析](https://blog.csdn.net/finley_feng/article/details/104467506)

### 一、基础知识

#### 1.1 前提阅读

* [一、OkHttp大体流程](设计思想解读开源框架库\OkHttp\一、OkHttp大体流程.md)
* 本文是基于OkHttp3.9版本的

#### 1.2 连接池

##### 1.2.1 无连接池

**大概流程**

* 首先进行TCP三次握手建立连接

* 写入请求数据

* 读取相应的数据

* 最后进行TCP四次挥手断开连接

  ![img](https://upload-images.jianshu.io/upload_images/19956127-4ea0889b80991291.png?imageMogr2/auto-orient/strip|imageView2/2/w/741/format/webp)

**缺点**

* 每次都得进行TCP三次握手，TCP四次挥手，是非常消耗网络资源和浪费时间的。

##### 1.2.2 有连接池

**大概流程**

* 在上面的基础上使用keep-alive



![img](https://upload-images.jianshu.io/upload_images/19956127-9733ad1c06a77aa4.png?imageMogr2/auto-orient/strip|imageView2/2/w/746/format/webp)

**优点**

* 在timeout空闲时间内，连接不会关闭，相同重复的request将复用原先的connection，减少握手的次数，大幅提高效率。
* 并非keep-alive的timeout设置时间越长，就越能提升性能。长久不关闭会造成过多的僵尸连接和泄露连接出现。
* Okhttp支持5个并发KeepAlive，默认链路生命为5分钟(链路空闲后，保持存活的时间)，连接池有ConectionPool实现，对连接进行回收和管理。

### 二、源码分析—引言

#### 2.1 RealCall#getResponseWithInterceptorChain

* 由[一、OkHttp大体流程](设计思想解读开源框架库\OkHttp\一、OkHttp大体流程.md)可知，会进行到如下方法。

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

* 其中ConnectInterceptor拦截器是关键，这是真正去和服务器进行连接的部分。所以我们接下来重点将这个ConnectInterceptor

#### 2.2 ConnectInterceptor#构造方法

```java
  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }
```

* 主要就是传入了OkHttpClient对象

#### 2.3 ConnectInterceptor#intercept

* 由[一、OkHttp大体流程](设计思想解读开源框架库\OkHttp\一、OkHttp大体流程.md)可知，会调用拦截器的intercept方法

```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //获取Requset对象
    //分析1
    Request request = realChain.request();
    //分析2
    StreamAllocation streamAllocation = realChain.streamAllocation();
	//分析3 判断是不是GET请求方式
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //分析4
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    //分析5
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

* 分析1：从RealInterceptorChain中获取Request对象，Requset对象封装了请求（路径，请求方法，body等）
* 分析2：从RealInterceptorChain中获取StreamAllocation对象，在RetryAndFollowUpInterceptor中创建的StreamAllocation，此处从拦截器链中获取
* 分析3：判断是不是GET请求方式
* 分析4：生成HttpCodec对象；HttpCodec：用来编码HTTP requests和解码HTTP responses
* 分析5：调用StreamAllocation的connection方法，建立连接；RealConnection：连接对象，负责发起与服务器的连接。这个RealConnection对象是用来进行实际的网络IO传输的

#### 2.4 RealInterceptorChain#StreamAllocation

* 这个对象是在哪赋值的呢？对于用户没有配置拦截器而言，第一个拦截器是RetryAndFollowUpInterceptor。其实RealInterceptorChain的成员变量就是在RetryAndFollowUpInterceptor的intercept方法中赋值的。
* RetryAndFollowUpInterceptor#intercept

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();
   	......
    streamAllocation = new StreamAllocation(client.connectionPool(), 
                                            createAddress(request.url()),call,
                                            eventListener, callStackTrace);
	......
        response = realChain.proceed(request, streamAllocation, null, null);
    ......
}
```

* 为其赋值了ConnectionPool对象，Address对象，Call对象等

#### 2.5 小结

* 可以看到ConnectInterceptor#intercept中主要都是对StreamAllocation的使用，那么我们将在下一节讲解StreamAllocation

### 三、源码分析—StreamAllocation

#### 3.1 StreamAllocation#成员变量

```java
public final class StreamAllocation {
  public final Address address;//地址
  private RouteSelector.Selection routeSelection;//选中的路由集合
  private Route route;//路由
  private final ConnectionPool connectionPool;//连接池
  /**
   *Calls：Stream的逻辑顺序，通常是初始请求以及重定向请求。
   * 并且希望单个请求的所有流都被保存在同一个Connection上，以实现更好的行为
   */
  public final Call call;//请求对象
  public final EventListener eventListener;//事件回调
  private final Object callStackTrace;//日志
 
  // State guarded by connectionPool.
  private final RouteSelector routeSelector;//路由选择器
  private int refusedStreamCount;
  /**
   * Connection：连接到远程服务器的物理Socket连接，Connection建立起来可能会比较慢，并且能够取消已经建立的Connection
   */
  private RealConnection connection;//HTTP连接
  private boolean reportedAcquired;
  private boolean released;
  private boolean canceled;
  private HttpCodec codec;//HTTP请求的编码和响应解码
	......
}
```

#### 3.2 StreamAllocation#newStream

##### 2.5.1 StreamAllocation#newStream

```java
public HttpCodec newStream(OkHttpClient client, Interceptor.Chain chain, 
                           boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();//设置的连接超时时间
    int readTimeout = chain.readTimeoutMillis();//读取超时
    int writeTimeout = chain.writeTimeoutMillis();//写入超时
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();//连接失败是否重试标志

    try {
        //分析1 重点
      RealConnection resultConnection = findHealthyConnection(connectTimeout, 
											readTimeout,writeTimeout, 																connectionRetryEnabled, 
                                            doExtensiveHealthChecks);
        //分析2
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

* 分析1：调用findHealthyConnection方法，通过策略寻找一个真正的RealConnection连接，生成实际的网络连接类，RealConnection利用Socket建立连接。
* 分析2：调用RealConnection的newCodec方法，创建HttpCodec对象（通过网络连接的实际类生成网络请求和网络响应的编码类）

##### 3.2.2 RealConnection#newCodec

```java
  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```

* 若采用Protocol.HTTP_2协议，即Http2Connection存在，创建Http2Codec，否则，创建Http1Codec，封装BufferedSource和BufferedSink。

##### 3.2.3 HttpCodec

* 创建请求和解析相应

```java
public interface HttpCodec {
  /**
   * The timeout to use while discarding a stream of input data. Since this is used for connection
   * reuse, this timeout should be significantly less than the time it takes to establish a new
   * connection.
   */
  int DISCARD_STREAM_TIMEOUT_MILLIS = 100;

  //为发送请求而提供的，创建请求体，以用于发送请求体数据。
  Sink createRequestBody(Request request, long contentLength);
 
  //为发送请求而提供的，写入请求头部。
  void writeRequestHeaders(Request request) throws IOException;
 
  //将请求刷新到基础套接字
  void flushRequest() throws IOException;
 
  //将请求刷新到底层套接字，并发出不再传输字节的信号
  void finishRequest() throws IOException;
 
  //从HTTP传输解析响应头的字节。
  Response.Builder readResponseHeaders(boolean expectContinue) throws IOException;
 
  //为获得响应而提供的，打开请求体，以用于后续获取请求体数据。
  ResponseBody openResponseBody(Response response) throws IOException;
 
  //取消请求执行。
  void cancel();
}
```

#### 3.3 StreamAllocation#findHealthyConnection

* 由3.2.1可知，其内部调用了StreamAllocation#findHealthyConnection方法

```java
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    //分析1 开启while循环，通过调用findConnection方法获取RealConnection对象赋值给cadidat
    while (true) {
        //分析2 获取RealConnection对象
      RealConnection candidate = findConnection(connectTimeout, readTimeout,
                                                writeTimeout,connectionRetryEnabled);

      //如果candidate的successCount==0,直接返回candidate，while循环结束;
      synchronized (connectionPool) {
          //等于0的时候表示整个网络请求已经结束了
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      //调用candidate的isHealthy()方法，进行“健康检查”，如果candidate是一个不“健康”的对象，
      //其中不“健康”指的是Socket没有关闭、或者它的输入输出流没有关闭，
      //则对调用noNewStreams()方法进行销毁处理，接着继续循环。
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

* 分析1：开启while循环，通过调用findConnection方法获取RealConnection对象赋值给cadidat
* 分析2：调用findConnection方法去获取RealConnection对象（当前keep-alive有效时间内的连接）

#### 3.4 StreamAllocation#findConnection

* 步骤1：StreamAllocation的connection如果可以复用则复用;

* 步骤2：如果StreamAllocation的connection不能复用，则从连接池中获取RealConnection对象，获取成功则返回;

* 步骤3：如果连接池里没有，则new一个RealConnection对象；

* 步骤4：调用RealConnection的connect（）方法发起请求；connect-->connectSocket()进行socket连接-->Platform.get().connectSocket()-->socket.connect(address, connectTimeout);(此时进行了三次握手),握手完成后调用establishProtocol()。

* 步骤5：将RealConnection对象存进连接池中，以便下次复用
* 步骤6：返回RealConnection对象

```java
private RealConnection findConnection(int connectTimeout, int readTimeout, 
		int writeTimeout,boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    //定义最终返回的RealConnection
    RealConnection result = null;
    Route selectedRoute = null;
    //用来标识有没有分配的连接
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
     	......
     	//步骤1
		//尝试使用已分配的连接（StreamAllocation内部保存的RealConnection）
		//我们在这里需要小心，因为我们已经分配的连接可能已经被限制在创建新的流中
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
        // 如果connection 不为null，也就是有完好的连接(RealConnection),则复用，赋值给result
      if (this.connection != null) {
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        releasedConnection = null;
      }
		//步骤2
        //如果result==null,也就是说this.connection==null
        //说明上面找不到可用复用的  从连接池中获取可用的连接
      if (result == null) {
		//连接池中获取，调用其get方法
        Internal.instance.get(connectionPool, address, this, null);
          // 找到对应的RealConnection对象
        if (connection != null) {
            //更改标志位，赋值给result
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
        // 已经找到RealConnection对象，直接返回
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    //如果我们需要选择一条路线，就选一条。这是一个阻塞操作
    boolean newRouteSelection = false;
    // 线路的选择，多IP操作
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    //步骤3
    //如果没有可用连接，则自己创建一个
    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // 获取路由表中的信息
        // 现在我们有一组IP地址，再次尝试从池中获取连接。这可能由连接合并而匹配
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
            // 获取到连接
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

        //没有获取到
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

       //建一个连接并立即将其分配给该分配。 这使得异步cancel（）可以中断我们即将进行的握手
        route = selectedRoute;
        refusedStreamCount = 0;
          //如果从缓存中没有找到连接，则自己创建一个 new一个新的连接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // 如果我们第二次从连接池发现一个连接，我们就完成了.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    //连接具体方法 开始TCP以及TLS握手操作,这是阻塞操作  http的三次握手操作
    result.connect(
        connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    // 将新创建的连接，放入连接池中
    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

        // 将RealConnection 连接放入连接池
      // Pool the connection.
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
        // 如果同时创建了到同一地址的另一个多路复用连接，则释放此连接并获取该连接
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

* 代码的流程都在前面进行了说明，同时代码都进行了详细的注释了

* 有关连接池的部分主要是

```java
Internal.instance.get(connectionPool, address, this, null);
Internal.instance.get(connectionPool, address, this, route);
Internal.instance.put(connectionPool, result);
```

* 这部分将在下一节详细介绍

#### 3.5 StreamAllocation#acquire

```java
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();
	//赋值给全局变量 StreamAllocation的成员变量connection
    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
      //创建StreamAllocationReference对象并添加到allocations集合中
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

### 四、源码解析-ConnectionPool连接池

#### 4.1 Internal#instance

* 从3.2中我们可以知道，主要对连接池的操作是如下

```java
Internal.instance.get(connectionPool, address, this, null);
Internal.instance.get(connectionPool, address, this, route);
Internal.instance.put(connectionPool, result);
```

* 主要涉及Internal成员变量instance，以及它的get和put方法

##### 4.1.1 Internal#instance

```java
public abstract class Internal {
	......
  public static Internal instance;
```

* 可以看到这instance是Internal的静态成员变量，同时instance是Internal对象，同时Internal是抽象类。其实很容易看出来这就是一个共享的实例对象。
* 那么instance是在哪进行实例化的呢？其实是在OkHttpClient的静态代码块中执行的

##### 4.1.2 OkHttpClient#静态代码块实例instance

* 有关static可点击查看[static](必备Java知识/关键字/static.md)
* 静态代码块只会执行一次

```java
static {
    Internal.instance = new Internal() {
      @Override public void addLenient(Headers.Builder builder, String line) {
        builder.addLenient(line);
      }

      @Override public void addLenient(Headers.Builder builder, String name, String value) {
        builder.addLenient(name, value);
      }

      @Override public void setCache(OkHttpClient.Builder builder, InternalCache internalCache) {
        builder.setInternalCache(internalCache);
      }

      @Override public boolean connectionBecameIdle(
          ConnectionPool pool, RealConnection connection) {
        return pool.connectionBecameIdle(connection);
      }

      @Override public RealConnection get(ConnectionPool pool, Address address,
          StreamAllocation streamAllocation, Route route) {
        return pool.get(address, streamAllocation, route);
      }

      @Override public boolean equalsNonHost(Address a, Address b) {
        return a.equalsNonHost(b);
      }

      @Override public Socket deduplicate(
          ConnectionPool pool, Address address, StreamAllocation streamAllocation) {
        return pool.deduplicate(address, streamAllocation);
      }

      @Override public void put(ConnectionPool pool, RealConnection connection) {
        pool.put(connection);
      }

      @Override public RouteDatabase routeDatabase(ConnectionPool connectionPool) {
        return connectionPool.routeDatabase;
      }

      @Override public int code(Response.Builder responseBuilder) {
        return responseBuilder.code;
      }

      @Override
      public void apply(ConnectionSpec tlsConfiguration, SSLSocket sslSocket, boolean isFallback) {
        tlsConfiguration.apply(sslSocket, isFallback);
      }

      @Override public HttpUrl getHttpUrlChecked(String url)
          throws MalformedURLException, UnknownHostException {
        return HttpUrl.getChecked(url);
      }

      @Override public StreamAllocation streamAllocation(Call call) {
        return ((RealCall) call).streamAllocation();
      }

      @Override public Call newWebSocketCall(OkHttpClient client, Request originalRequest) {
        return RealCall.newRealCall(client, originalRequest, true);
      }
    };
  }
```

* 通过匿名的方式对Internal抽象类进行了实现，由于put和get都是抽象方法，所以Internal对象的get方法和put方法对应的就是上面的get和put方法

##### 4.1.3 Internal#get

* Internal#get对应的就是OkHttpClient#静态代码块中匿名实现的其get方法

```java
@Override public RealConnection get(ConnectionPool pool, Address address,
          StreamAllocation streamAllocation, Route route) {
        return pool.get(address, streamAllocation, route);
}
```

* 可以看到最终调用的是pool的get方法，其中pool是ConnectionPool对象

##### 4.1.4 Internal#put

* Internal#put对应的就是OkHttpClient#静态代码块中匿名实现的其put方法

```java
@Override public void put(ConnectionPool pool, RealConnection connection) {
        pool.put(connection);
}
```

* 可以看到最终调用的是pool的put方法，其中pool是ConnectionPool对象

#### 4.2 ConnectionPool#成员变量

```java
private final Deque<RealConnection> connections = new ArrayDeque<>();
```

* connections：用于存放RealConnection的ArrayDeque队列

#### 4.3 ConnectionPool#get

##### 4.3.1 ConnectionPool#get

```java
@Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
        //isEligible方法判断是否可用
      if (connection.isEligible(address, route)) {
          //调用acquire
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
```

**概述**

* 通过遍历connections(用于存放RealConnection的ArrayDeque队列)
* 调用RealConnection的isEligible()方法判断其是否可用，RealConnection会在下一节详细介绍
* 如果可用就会调用streamAllocation的acquire()方法，并返回connection。

##### 4.3.2 RealConnection#isEligible

* 判断connection是否可用

```java
  public boolean isEligible(Address address, @Nullable Route route) {
    // If this connection is not accepting new streams, we're done.
    if (allocations.size() >= allocationLimit || noNewStreams) return false;

    // If the non-host fields of the address don't overlap, we're done.
    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;

      //分析1
    // If the host exactly matches, we're done: this connection can carry the address.
    if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false;

    // 2. The routes must share an IP address. This requires us to have a DNS address for both
    // hosts, which only happens after route planning. We can't coalesce connections that use a
    // proxy, since proxies don't tell us the origin server's IP address.
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;

    // 3. This connection's server certificate's must cover the new host.
    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;

    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }

    return true; // The caller's address can be carried by this connection.
  }
```

* 分析1：如果两个host相同的话，就返回true
* host的值

```
* <ul>
*   <li>A regular host name, like {@code android.com}.
*   <li>An IPv4 address, like {@code 127.0.0.1}.
*   <li>An IPv6 address, like {@code ::1}. Note that there are no square braces.
*   <li>An encoded IDN, like {@code xn--n3h.net}.
* </ul>
*
* <p><table summary="">
*   <tr><th>URL</th><th>{@code host()}</th></tr>
*   <tr><td>{@code http://android.com/}</td><td>{@code "android.com"}</td></tr>
*   <tr><td>{@code http://127.0.0.1/}</td><td>{@code "127.0.0.1"}</td></tr>
*   <tr><td>{@code http://[::1]/}</td><td>{@code "::1"}</td></tr>
*   <tr><td>{@code http://xn--n3h.net/}</td><td>{@code "xn--n3h.net"}</td></tr>
* </table>
```

* 官方的注释已经很明白了，比如同一个ip，同一个域名等

##### 4.3.3 StreamAllocation#acquire

```java
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();
	//赋值给全局变量 StreamAllocation的成员变量connection
    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
      //创建StreamAllocationReference对象并添加到allocations集合中
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

- 先是从连接池中获取的RealConnection对象赋值给StreamAllocation的成员变量connection；
- 创建StreamAllocationReference对象(StreamAllocation对象的弱引用)，
  并添加到RealConnection的allocations集合中，到时可以通过allocations集合的大小来判断网络连接次数是否超过OkHttp指定的连接次数。

#### 4.4 ConnectionPool#put

##### 4.4.1 ConnectionPool#put

```java
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
      //判断是否执行清理任务
    if (!cleanupRunning) {
      cleanupRunning = true;
        //分析1
      executor.execute(cleanupRunnable);
    }
      //将连接添加到连接池
    connections.add(connection);
  }
```

*  put()方法在将连接添加到连接池之前，会先执行清理任务，通过判断cleanupRunning是否在执行，如果当前清理任务没有执行，则更改cleanupRunning标识，并执行清理任务cleanupRunnable。
* 分析1：可以看出来这是一个线程池executor，调用execute，通过名字我们知道cleanupRunnable继承自Runnable，所以将会执行cleanupRunnable的run方法。那么我们去代码中证明一下自己的猜想

```java
//可以看到executor是一个线程池对象
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
```

* 主要的功能还是将连接添加到连接池中

##### 4.4.2 ConnectionPool#cleanupRunnable#run

* 用来执行清理任务的

```java
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
          //cleanup方法对连接池进行清理，返回进行下次清理的间隔时间
        long waitNanos = cleanup(System.nanoTime());
          //如果返回的数据间隔为-1，则会接受循环
        if (waitNanos == -1) return;
          //如果大于1会调用wait进行等待
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
                //进行等待
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```

* cleanupRunnable方法主要调用cleanup()对连接进行清理，

##### 4.4.3 ConnectionPool#cleanup

* cleanup()方法通过for循环遍历connections队列，记录最大空闲时间和空闲时间最长的连接；如果存在超过空闲保活时间或空闲连接数超过最大空闲连接数限制的连接，则从connections中移除，最后执行关闭该连接的操作。

```java
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        //判断当前连接有没有正在使用
        if (pruneAndGetAllocationCount(connection, now) > 0) {
            //如果真在使用了inUseConnectionCount加1，并且直接跳出该次循环
          inUseConnectionCount++;
          continue;
        }

          //如果不是正在使用的连接，则把该变量加1
        idleConnectionCount++;

        //算出当前连接已经存活的时间
        long idleDurationNs = now - connection.idleAtNanos;
          //找出存活最久的那个连接
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      //如果最大存活的连接超过了keep-alive设置的时间
        //或者没有正在使用的连接数超过了maxIdleConnections
        //则从连接池中移除
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        //删除连接
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
          //如果还存在没有正在使用的连接，但是存活时间没超过keep-alive设置的时间并且个数没超过最大的个数，返回还要等多久的时间才能进行下次清除操作
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        //如果都是正在使用的连接，则直接返回keep-alive设置的时间，也就是5分钟
        return keepAliveDurationNs;
      } else {
        //如果连正在使用的连接都没有，则直接返回-1，说明不需要下次的清除操作
        cleanupRunning = false;
        return -1;
      }
    }

      //关闭Socket
    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```

* 遍历连接，pruneAndGetAllocationCount()方法查看连接是否在使用。
* inUseConnectionCount，正在使用数量
* idleConnectionCount，空闲数量
* idleDurationNs，空闲连接，计算已空闲时间，查找到空闲时间最长的。

**清理算法**

* longestIdleDurationNs(最长空闲时间)，大于5min或空闲数量超过5个(默认)，将对应longestIdleConnection连接删除，返回0，下一次立即清理。
* 不足5min，空闲连接数量<5，返回timeout(距离5min还有多久)，线程阻塞timeout后，再次清理。
* 空闲连接=0，且正使用连接>0，返回5min，最长等待时间。
* 空闲和正使用连接都0，返回-1，不清理，线程结束。设置cleanupRunning标志位，下次put()方法重新唤起cleanupRunnable任务。

**清理过程**

* 从连接池队列ArrayDeque中删除该项连接。
* closeQuietly()方法关闭Socket。

**using连接判断**

* 遍历RealConnection内部Reference<StreamAllocation>列表
* reference.get()不空，说明有StreamAllocation正引用RealConnection。
* reference.get()不存在，说明StreamAllocation已被Jvm清理，同时，从references列表中删除该项reference。
* 若references列表是空，说明references弱引用都是空，没有StreamAllocation使用该连接。

#### 4.5 流程

##### 4.5.1 StreamAllocation#findConnection

* 主要就是调用StreamAllocation#findConnection去获取连接

  * 步骤1：StreamAllocation的connection如果可以复用则复用;
  * 步骤2：如果StreamAllocation的connection不能复用，则从连接池中获取RealConnection对象，获取成功则返回;
  * 步骤3：如果连接池里没有，则new一个RealConnection对象；
  * 步骤4：调用RealConnection的connect（）方法发起请求；connect-->connectSocket()进行socket连接-->Platform.get().connectSocket()-->socket.connect(address, connectTimeout);(此时进行了三次握手),握手完成后调用establishProtocol()。
  * 步骤5：将RealConnection对象存进连接池中，以便下次复用
  * 步骤6：返回RealConnection对象

##### 4.5.2 连接池己见

* 首先从StreamAllocation中获取，能用就复用
* 不能有就从ConnectionPool中获取，通过调用ConnectionPool对象的get方法去获取。获取到了就复用，ConnectionPool使用如下的方式保存着连接，使用的是双向队列的数据结构。

```java
private final Deque<RealConnection> connections = new ArrayDeque<>();
```

* 如果没有获取到，那么就自己创建一个，然后调用ConnectionPool对象的put方法，把连接放入到connections对象中。put的时候会清理，清理超过空闲保活时间或空闲连接数超过最大空闲连接数限制的连接

##### 4.5.3 总结

* 对于ConnectionPool连接池而言，就是内部存在Deque双向队列，存储着RealConnection，优先进行复用connections中的连接，复用不了就创建一个并放入到connections中，然后放入的时候可能执行清理任务，清理掉不符合要求的连接。

* 一个OkHttpClient实例对象只有一个ConnectionPool，所以对于同一个OkHttpClient实例，复用的都是同一个connections

### 五、源码分析-RealConnection

#### 5.1 RealConnection#成员变量

#### 5.2 RealConnection#connect

* 在3.4 StreamAllocation#findConnection中会调用result.connect，其中result就是RealConnection对象
* RealConnection#connect这里是真正和服务器建立连接的方法

```java
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");
    // 路线选择
    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);
 
    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
      }
    }
    // 开始连接
    while (true) {
      try {
        //建立隧道连接
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            // 我们无法连接隧道，但正确关闭了我们的资源
            break;
          }
        } else {
          // 建立普通连接
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
        //建立协议
        // 不管是建立隧道连接，还是建立普通连接，都少不了建立协议这一步。
        // 这一步是在建立好了TCP连接之后，而在TCP能被拿来收发数据之前执行的。
        // 它主要为数据的加密传输做一些初始化，比如TLS握手，HTTP/2的协议协商等
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        //完成连接
        break;
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;
 
        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);
 
        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }
 
        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }
 
    if (route.requiresTunnel() && rawSocket == null) {
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }
 
    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }
```

* 主要就是调用了connectTunnel()方法和connectSocket()，这是两种不同的模式。
* connectSocket()是普通连接
* connectTunnel()是隧道连接

#### 5.3 RealConnection#connectSocket普通连接

##### 5.3.1 RealConnection#connectSocket普通连接

```java
private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();
    //根据代理类型的不同处理Socket
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);
 
    eventListener.connectStart(call, route.socketAddress(), proxy);
 
    //设置超时时间
    rawSocket.setSoTimeout(readTimeout);
    try {
        //分析1
      //获取指定的平台去进行连接
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
 
    // The following try/catch block is a pseudo hacky way to get around a crash on Android 7.0
    // More details:
    // https://github.com/square/okhttp/issues/3245
    // https://android-review.googlesource.com/#/c/271775/
    try {
      //得到连接器的输入流对象
      source = Okio.buffer(Okio.source(rawSocket));
      //得到连接器的输出流对象
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
```

* 完成在原始套接字上构建完整的HTTP或HTTPS连接所需的所有工作。

* 分析1：调用Platform.get().connectSocket方法去连接

##### 5.3.2 Platform#get

```java
  public static Platform get() {
    return PLATFORM;
  }
private static final Platform PLATFORM = findPlatform();
```

* Platform.get()方法获取了一个Platform对象

##### 5.3.3 Platform#connectSocket

```java
public void connectSocket(Socket socket, InetSocketAddress address,
      int connectTimeout) throws IOException {
    socket.connect(address, connectTimeout);
}
```

* 调用Socket的connect方法建立连接，所以OkHttp底层用的也是Socket。
* 有关Socket详细可点击查看[Socket解析](设计思想解读开源框架库\网络访问框架\网络基础\Socket解析.md)

#### 5.4 RealConnection#connectTunnel隧道连接

##### 5.4.1 RealConnection#connectTunnel

```java
private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout, 
                           Call call,EventListener eventListener) throws IOException {
    //分析1 构造一个 建立隧道连接请求
    Request tunnelRequest = createTunnelRequest();
    HttpUrl url = tunnelRequest.url();
    for (int i = 0; i < MAX_TUNNEL_ATTEMPTS; i++) {
        // 与HTTP代理服务器建立TCP连接
      connectSocket(connectTimeout, readTimeout, call, eventListener);
        //分析3
        // 创建隧道。这主要是将 建立隧道连接 请求发送给HTTP代理服务器，并处理它的响应
      tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);

      if (tunnelRequest == null) break; // Tunnel successfully created.

      // The proxy decided to close the connection after an auth challenge. We need to create a new
      // connection, but this time with the auth credentials.
      closeQuietly(rawSocket);
      rawSocket = null;
      sink = null;
      source = null;
      eventListener.connectEnd(call, route.socketAddress(), route.proxy(), null);
    }
  }
```

* 是否通过代理隧道建立HTTPS连接的所有工作。这里的问题是代理服务器可以发出一个验证质询，然后关闭连接
* 分析1：调用createTunnelRequest创建一个隧道连接的Request
* 分析2：就是5.3分析的
* 分析3：通过createTunnel方法建立隧道连接，这是重点。
* 如果要了解隧道代理可点击查看[SOCKS代理和HTTP普通代理隧道代理](设计思想解读开源框架库\网络访问框架\网络基础\SOCKS代理和HTTP普通代理隧道代理.md)

##### 5.4.2 RealConnection#createTunnel

```java
private Request createTunnel(int readTimeout, int writeTimeout, Request tunnelRequest,
      HttpUrl url) throws IOException {
    // Make an SSL Tunnel on the first message pair of each SSL + proxy connection.
    // 在每个SSL + 代理连接的第一个消息对上创建一个SSL隧道。
    String requestLine = "CONNECT " + Util.hostHeader(url, true) + " HTTP/1.1";
    while (true) {
      Http1Codec tunnelConnection = new Http1Codec(null, null, source, sink);
      source.timeout().timeout(readTimeout, MILLISECONDS);
      sink.timeout().timeout(writeTimeout, MILLISECONDS);
      tunnelConnection.writeRequest(tunnelRequest.headers(), requestLine);
      tunnelConnection.finishRequest();
      Response response = tunnelConnection.readResponseHeaders(false)
          .request(tunnelRequest)
          .build();
      // The response body from a CONNECT should be empty, but if it is not then we should consume
      // it before proceeding.
      long contentLength = HttpHeaders.contentLength(response);
      if (contentLength == -1L) {
        contentLength = 0L;
      }
      Source body = tunnelConnection.newFixedLengthSource(contentLength);
      Util.skipAll(body, Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
      body.close();
 
      switch (response.code()) {
        case HTTP_OK:
          // Assume the server won't send a TLS ServerHello until we send a TLS ClientHello. If
          // that happens, then we will have buffered bytes that are needed by the SSLSocket!
          // This check is imperfect: it doesn't tell us whether a handshake will succeed, just
          // that it will almost certainly fail because the proxy has sent unexpected data.
          if (!source.buffer().exhausted() || !sink.buffer().exhausted()) {
            throw new IOException("TLS tunnel buffered too many bytes!");
          }
          return null;
 
        case HTTP_PROXY_AUTH:
          tunnelRequest = route.address().proxyAuthenticator().authenticate(route, response);
          if (tunnelRequest == null) throw new IOException("Failed to authenticate with proxy");
 
          if ("close".equalsIgnoreCase(response.header("Connection"))) {
            return tunnelRequest;
          }
          break;
 
        default:
          throw new IOException(
              "Unexpected response code for CONNECT: " + response.code());
      }
    }
  }
```

* 要通过HTTP代理建立HTTPS连接，请发送未加密的CONNECT请求以创建代理连接。 如果代理需要授权，则可能需要重试

### 六、思考的问题

#### 6.1 对于同一个域名的接口一直调用连接池会有多少个连接

* 一般采用的时异步的方式调用的，所以会同时好多个调用，假设是k个，此时由于并发问题，由于从连接池connections获取的时候并没有put连接进去（put是在connect之后调用的，是建立连接之后调用的，所以一般时间较长），所以至多k个都会从connections取不到，然后各自又put自己的连接进去。这样就造成了连接池中存在好多相同的连接。
* 当有一个接口通过了connect，然后把自己的连接放进去了，这样就存在缓存了，那样后面的接口从connections就可以取到了，后面的接口就可以复用了。

#### 6.2 连接池规则

* 对于同一个host的都会复用

* 对于同一个域名的会复用同一个，比如下面两个，由于host都是zhidao.baidu.com，所以会复用

  * “https://zhidao.baidu.com/question/536751608.html”

  * "https://zhidao.baidu.com/question/1604078665855807427.html?fr=iks&word=123456789&ie=gbk"

