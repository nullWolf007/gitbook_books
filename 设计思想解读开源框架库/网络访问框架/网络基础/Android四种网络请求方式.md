[TOC]

# Android四种网络请求方式

### 参考链接

* [**Android4种网络连接方式HttpClient、HttpURLConnection、OKHttp和Volley优缺点**](https://blog.csdn.net/xiechengfa/article/details/71535563)

## 一、前言

### 1.1 四种常用的网络请求方式

#### 1.1.1 HttpClient

**优点**

* Apache HttpClient高效稳定，有很多API

**缺点**

* 由于API太多，很难在不破坏兼容性的情况下对它进行升级和扩展，维护成本高，故Android 开发团队不愿意在维护该库而是转投更为轻便的HttpUrlConnection

* Android早就不推荐HttpClient，5.0之后干脆废弃，后续会删除。6.0删除了HttpClient。Java开发用HttpClient，官方推荐Android开发用HttpUrlConnection。

**不推荐使用**

#### 1.1.2 HttpUrlConnection

**优点**

* HttpUrlConnection是一种多用途、轻量极的HTTP客户端，使用它来进行HTTP操作可以适用于大多数的应用程序。虽然HttpUrlConnection的API提供的比较简单，但是同时这也使得我们可以更加容易地去使用和扩展它。比较轻便，灵活，易于扩展。

**缺点**

* 在android 2.2及以下版本中HttpUrlConnection存在着一些bug，所以建议在android 2.3以后使用HttpUrlConnection，2.3之前使用HttpClient。

**可以使用**

#### 1.1.3 Volley

**说明**

* Volley由谷歌开发，是一个简化网络任务的库。他负责处理请求，加载，缓存，线程，同步等问题。它可以处理JSON，图片，缓存，文本源，支持一定程度的自定义。Volley在Android 2.3及以上版本，使用的是HttpURLConnection，而在Android 2.2及以下版本，使用的是HttpClient。 

**缺点**

* 不过再怎么封装Volley在功能拓展性上始终无法与OkHttp相比。而且OkHttp得到了官方的认可

**可以使用**

#### 1.1.4 OkHttp

**优点**

* OkHttp扮演着传输层的角色。
* OkHttp使用Okio来大大简化数据的访问与存储，Okio是一个增强 java.io 和 java.nio的库。
* OkHttp 处理了很多网络疑难杂症：会从很多常用的连接问题中自动恢复。如果您的服务器配置了多个IP地址，当第一个IP连接失败的时候，OkHttp会自动尝试下一个IP。
* OkHttp还处理了代理服务器问题和SSL握手失败问题。
* OkHttp是Android版Http客户端(Android 2.3以上)。非常高效，支持SPDY(可以合并多个到同一个主机的请求)、连接池、GZIP和 HTTP 缓存。
* 默认情况下，OKHttp会自动处理常见的网络问题，像二次连接、SSL的握手问题。
* 如果你的应用程序中集成了OKHttp，Retrofit默认会使用OKHttp处理其他网络层请求。
* 从Android4.4开始HttpURLConnection的底层实现采用的是okHttp 
* 缓存响应避免重复的网络请求

**支持功能**

*  一般的get请求 
*   一般的post请求 
*  基于Http的文件上传 
*  文件下载 
*  上传下载的进度回调 
*  加载图片 
*  支持请求回调，直接返回对象、对象集合 
*  支持session的保持 
*  支持自签名网站https的访问，提供方法设置下证书就行 
*  支持取消某个请求

**推荐使用**

## 二、OkHttp的使用

### 2.1 示例代码

```java
// 创建OkHttpClient对象
OkHttpClient client = new OkHttpClient.Builder()
	.readTimeout(10, TimeUnit.SECONDS)//数据传输时间
    .connectTimeout(9, TimeUnit.SECONDS)//连接时间
    .connectionPool(new ConnectionPool(5, 30, TimeUnit.SECONDS))
    //连接池 每个地址的空闲连接数为 5个，每个空闲连接的存活时间为30秒. 存活时间如果存在高并发的情况的话 需要设置小一点
    .build();
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
//创建网络请求 get方法
HttpUrl.Builder urlBuilder = HttpUrl.parse("http://test").newBuilder();
urlBuilder.addQueryParameter("id", "1");
Request getRequest = new Request.Builder()
	.url(urlBuilder.build())
    .header("token", "111")
    .get()
    .build();
//创建Call对象
Call postCall = client.newCall(postRequest);
Call getCall = client.newCall(getRequest);
//同步请求
try {
	Response response = getCall.execute();
    String result = response.body().string();
} catch (IOException e) {
	e.printStackTrace();
}
//异步请求
postCall.enqueue(new Callback() {
	@Override
    public void onFailure(@NotNull Call call, @NotNull IOException e) {
    	System.out.println("失败");
	}

	@Override
    public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
    	System.out.println("成功");
        String result = response.body().string();
	}
});
```





