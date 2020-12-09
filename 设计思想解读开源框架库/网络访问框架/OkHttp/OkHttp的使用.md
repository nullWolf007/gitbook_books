[TOC]
# OkHttp详解

### 参考链接

* [**Okhttp 使用详解**](https://www.jianshu.com/p/cdab05b87a9d)
* [**OkHttp官方教程解析-彻底入门OkHttp使用**](https://blog.csdn.net/mynameishuangshuai/article/details/51303446)

## 一、前言

### 1.1 Android四种网络请求

* 详情请点击[Android四种网络请求方式](https://github.com/nullWolf007/Android/blob/master/%E8%BF%9B%E9%98%B6/%E7%BD%91%E7%BB%9C%E7%9B%B8%E5%85%B3/Android%E5%9B%9B%E7%A7%8D%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82%E6%96%B9%E5%BC%8F.md)

### 1.2 OkHttp简介

#### 1.2.1 简述

* 由上述四种为网络请求的比较，我们可以发现OkHttp已经成为Android端最流行的网络第三方类库。

#### 1.2.2 引入

* 直接添加依赖即可

  ```xml
  //这是文档书写时 最新版本
  implementation 'com.squareup.okhttp3:okhttp:4.3.1'
  ```

#### 1.2.3 基本步骤

* 创建OkHttpClient，并进行配置
* 创建请求(如：GET/POST/DELETE/PUT)
* 执行请求(同步/异步)

## 二、OkHttp的基本使用

* 以下代码基于`com.squareup.okhttp3:okhttp:4.3.1`

### 2.1 创建OkHttpClient

#### 2.1.1 创建简单的OkHttpClient

```java
OkHttpClient client = new OkHttpClient.Builder().build();
```

#### 2.1.2 创建稍复杂的OkHttpClient

```java
File cacheDir = new File(getCacheDir(), "okhttp_cache");
Cache cache = new Cache(cacheDir, 10 * 1024 * 1024);//缓存目录 缓存大小
OkHttpClient client = new OkHttpClient.Builder()
	.readTimeout(10, TimeUnit.SECONDS)//数据传输时间
    .writeTimeout(10, TimeUnit.SECONDS)//数据写入时间
    .connectTimeout(9, TimeUnit.SECONDS)//连接时间
    .addInterceptor(new MyInterceptor())//应用拦截器：添加消息头，打印日志之类
    .addNetworkInterceptor(new MyNetworkInterceptor())//添加网络拦截器
    .connectionPool(new ConnectionPool(5, 30, TimeUnit.SECONDS))
    //连接池 每个地址的空闲连接数为 5个，每个空闲连接的存活时间为30秒. 存活时间如果存在高并发的情况的话 需要设置小一点
    .cache(cache)//设置缓存
    .build();
```

#### 2.1.3 两种拦截器

**Interceptor(ApplicationInterceptor应用拦截器)**

* 不需要担心中间过程的响应,如重定向和重试.
* 总是只调用一次,即使HTTP响应是从缓存中获取.
* 观察应用程序的初衷. 不关心OkHttp注入的头信息如: If-None-Match.
* 允许短路而不调用Chain.proceed(),即中止调用.
* 允许重试,使Chain.proceed()调用多次.

**NetworkInterceptor(网络拦截器)**

* 能够操作中间过程的响应,如重定向和重试.

* 当网络短路而返回缓存响应时不被调用.

* 只观察在网络上传输的数据.

* 携带请求来访问连接.

**异同**

* 相同点
  * 都能对server返回的response进行拦截
  * 这两种拦截器本质上都是基于Interceptor接口，由开发者实现这个接口，然后将自定义的Interceptor类的对象设置到OkHttpClient对象中。所以，他们的对象，本质上没什么不同，都是Interceptor的实现类的对象。
  * 两者都会被add到OkHttpClient内的一个ArrayList中。当请求执行的时候，多个拦截器会依次执行（list本身就是有序的）
* 不同点
  * OkHttpClient添加两种拦截器的api不同。添加应用拦截器的接口是addInterceptor()，而添加网络拦截器的接口是addNetworkInterceptor().
  * 两者负责的区域不同，应用拦截器作用于OkHttpCore到Application之间，网络拦截器作用于network和OkHttpCore之间
  * 当访问的url发生重定向时，网络拦截器有可能被执行多次，但是不论任何情况，应用拦截器只会被执行一次。

**示例拦截器--自定义日志拦截器**

```java
// 自定义日志拦截器
class LoggingInterceptor implements Interceptor {
    private static final String TAG = "LoggingInterceptor";

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();

        long t1 = System.currentTimeMillis();
        Log.d(TAG, String.format("Sending request %s",
                request.url()));

        Response response = chain.proceed(request);

        long t2 = System.currentTimeMillis();
        Log.d(TAG, String.format("Received response for %s in %s",
                response.request().url(), (t2 - t1)));

        MediaType mediaType = response.body().contentType();
        String content = response.body().string();
        Log.d(TAG, content);

        response = response.newBuilder()
                .body(ResponseBody.create(mediaType, content))
                .build();
        return response;
    }
}
```

#### 2.1.4 注意

* **response.body().string() 只能调用一次**
* 由于OkHttpClient连接会keep-alive一定时间，并且会开辟一个线程的线程池。所以如果短时间大量的创建OkHttpClient实例，会导致线程数激增，然后导致问题。所以开发中应该**尽可能的减少OkHttpClient实例**，同时这样才可能尽可能地复用连接池、线程池等，减少开销。

### 2.2 创建请求

#### 2.2.1 GET请求

```java
//创建网络请求 get方法
HttpUrl.Builder urlBuilder = HttpUrl.parse("http://test").newBuilder();
urlBuilder.addQueryParameter("id", "1");//添加GET请求参数
Request getRequest = new Request.Builder()
	.url(urlBuilder.build())
    .header("token", "111")
    .get()
    .build();
Call getCall = client.newCall(getRequest);
```

#### 2.2.2 POST请求

##### 使用RequestBody传递json

```java
//使用json的形式构建一个RequestBody对象来存放待提交的数据
MediaType mediaType = MediaType.Companion.parse("application/json");
String jsonStr = "{\"username\":\"lisi\",\"nickname\":\"李四\"}";
RequestBody requestBody = RequestBody.Companion.create(jsonStr, mediaType);
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .header("token", "111")
    .post(requestBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

##### 使用RequestBody传递File

```java
MediaType mediaType = MediaType.Companion.parse("File/*");
File file = new File("path");
RequestBody requestBody = RequestBody.Companion.create(file, mediaType);
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .post(requestBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

##### 使用FormBody传递键值对参数

```java
// 构建一个RequestBody对象来存放待提交的数据
RequestBody requestBody = new FormBody.Builder()
	.add("username", "admin")
    .add("password", "123456")
    .build();
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .header("token", "111")
    .post(requestBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

##### 使用MultipartBody同时传递键值对参数和File对象

```java
//一般ForBody就是针对键值对 RequestBody就是多媒体 都有的话就是MultipartBody
File file = new File("path");
MediaType mediaType = MediaType.Companion.parse("File/*");
//构建RequestBody存放多媒体
RequestBody requestBody = RequestBody.create(file, mediaType);
// 构建一个MultipartBody对象来存放待提交的数据
MultipartBody multipartBody = new MultipartBody.Builder()
	.setType(MultipartBody.FORM)
    .addFormDataPart("id", "1")
    .addFormDataPart("file", file.getName(), requestBody)//三个参数的就是文件
    .build();
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .post(multipartBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

##### 使用MultipartBody提交分块请求

```java
//addPart和addFormDataPart类似，事实上addFormDataPart内部就是调用的addPart
File file = new File("path");
MediaType mediaType = MediaType.Companion.parse("File/*");
//构建RequestBody存放多媒体
RequestBody requestBody = RequestBody.create(file, mediaType);
// 构建一个MultipartBody对象来存放待提交的数据
MultipartBody multipartBody = new MultipartBody.Builder()
	.setType(MultipartBody.FORM)
    .addPart(
    	Headers.of("token", "1"),
        new FormBody.Builder().add("id", "1").build()
	)
    .addPart(
    	Headers.of("token", "2"),
        requestBody
	)
    .build();
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .post(multipartBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

##### 自定义RequestBody实现流的上传

```java
//requestBody对象重写了writeTo方法，里面有个BufferedSink对象，这个是okio包中的输出流，使用这个方法可以实现上传流的功能
//使用RequestBody上传文件时，并没有实现断点续传的功能。我可以使用这种方法结合RandomAccessFile类实现断点续传的功能。
File file = new File("path");
//构建RequestBody存放多媒体
RequestBody requestBody = new RequestBody() {
	@Nullable
    @Override
    public MediaType contentType() {
    	return null;
	}

    @Override
    public void writeTo(@NotNull BufferedSink bufferedSink) throws IOException {
    	FileInputStream fileInputStream = new FileInputStream(file);
        byte[] buffer = new byte[1024 * 8];
        if (fileInputStream.read(buffer) != -1) {
        	bufferedSink.write(buffer);
		}
	}
};
//创建网络请求 post方法
Request postRequest = new Request.Builder()
	.url("http://test")
    .post(requestBody)
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
```

### 2.3 执行请求

#### 2.3.1 同步请求

```java
//Android中网络请求不能在主线程中
Response response = postCall.execute();
```

#### 2.3.2 异步请求

```java
postCall.enqueue(new Callback() {
	@Override
    public void onFailure(@NotNull Call call, @NotNull IOException e) {
    	//失败回调
	}

    @Override
    public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
    	//成功回调，当前线程为子线程，如果需要更新UI，需要post到主线程中
        boolean successful = response.isSuccessful();
        //响应消息头
        Headers headers = response.headers();
        //响应消息体
        ResponseBody body = response.body();
        String content = response.body().string();//返回的数据内容
        //缓存控制
        CacheControl cacheControl = response.cacheControl();
	}
});
//String content=response.body().string(); //获取字符串
//InputStream inputStream = response.body().byteStream();//获取字节流(比如下载文件)
```

## 三、其他设置

### 3.1 设置请求头

```java
Request postRequest = new Request.Builder()
	.url("http://test")
    .header("User-Agent", "Test")
    .addHeader("token", "123")
    .build();
```

### 3.2 设置超时

```java
OkHttpClient client = new OkHttpClient.Builder()
	.connectTimeout(10, TimeUnit.SECONDS)//连接超时
    .readTimeout(10, TimeUnit.SECONDS)//读取超时(数据传输时间)
    .writeTimeout(10, TimeUnit.SECONDS)//写入超时
    .build();
```

### 3.3 设置缓存

```java
File cacheDir = new File(getCacheDir(), "okhttp_cache");
Cache cache = new Cache(cacheDir, 10 * 1024 * 1024);//缓存目录 缓存大小
OkHttpClient client = new OkHttpClient.Builder()
	.cache(cache)
    .build();
```

* 为了缓存响应，你需要一个你可以读写的缓存目录，和缓存大小的限制。这个缓存目录应该是私有的，不信任的程序应不能读取缓存内容。 
* 一个缓存目录同时拥有多个缓存访问是错误的。大多数程序只需要调用一次new OkHttpClient()，在第一次调用时配置好缓存，然后其他地方只需要调用这个实例就可以了。否则两个缓存示例互相干扰，破坏响应缓存，而且有可能会导致程序崩溃。 
* 响应缓存使用HTTP头作为配置。你可以在请求头中添加Cache-Control: max-stale=3600 ,OkHttp缓存会支持。你的服务通过响应头确定响应缓存多长时间，例如使用Cache-Control: max-age=9600。

## 四、实例核心代码

* 接口地址(金山词霸API)：http://fy.iciba.com/ajax.php?a=fy&f=auto&t=auto&w=hello,world
* 主要分给几部分：NetUrl存储路径的后半部分，Api存储接口方法，AppConfig存储了路径的前半部分(接口前半部分都是同一的)，NetUtils提供获取Retrofit的方法，GetDataResultBean为返回类

* 查看核心源代码请点击[RxJava2+Retrofit2+OkHttp实例核心代码](https://github.com/nullWolf007/ToolProject/tree/master/RxJava2%2BRetrofit2%2BOkHttp实例核心代码)



