[TOC]

## Retrofit的使用

#### 转载参考

* [Android Retrofit 2.0 的详细 使用攻略（含实例讲解）](https://www.jianshu.com/p/a3e162261ab6)

### 一、前言

#### 1.1 前提阅读

* [Android四种网络请求方式](设计思想解读开源框架库\网络访问框架\网络基础\Android四种网络请求方式md)

#### 1.2  简介

* 准确来说，**Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装。**
* 通俗来讲，Retrofit就是对OkHttp进行了封装，使其使用起来更加方便
* 原因：网络请求的工作本质上是 `OkHttp` 完成，而 Retrofit 仅负责 网络请求接口的封装

![Retrofit本质过程](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\Retrofit本质过程.png)

#### 1.3 本质过程

- App应用程序通过 Retrofit 请求网络，实际上是使用 Retrofit 接口层封装请求参数、Header、Url 等信息，之后由 OkHttp 完成后续的请求操作
- 在服务端返回数据之后，OkHttp 将原始的结果交给 Retrofit，Retrofit根据用户的需求对结果进行解析

#### 1.4 网络请求库对比

![网络请求库对比](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\网络请求库对比.png)

### 二、基本使用

#### 2.1 使用步骤

* **步骤1：**添加Retrofit库的依赖
* **步骤2：**创建 接收服务器返回数据 的类
* **步骤3：**创建 用于描述网络请求 的接口
* **步骤4：**创建 Retrofit 实例
* **步骤5：**创建 网络请求接口实例 并 配置网络请求参数
* **步骤6：**发送网络请求（采用最常用的异步方式）
* **步骤7：**处理服务器返回的数据

#### 2.2 步骤1:添加Retrofit库的依赖

##### 2.2.1 在build.gradle中添加retrofit库

```groovy
implementation 'com.squareup.retrofit2:retrofit:2.3.0'
```

##### 2.2.2 在AndroidMainfest中添加权限

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

#### 2.3 步骤2:创建 接收服务器返回数据 的类

* ReceptionBean.java

```java
public class ReceptionBean {
    ...
    // 根据返回数据的格式和数据解析方式（Json、XML等）定义
    // 下面会在实例进行说明
}
```

#### 2.4 步骤3:创建 用于描述网络请求 的接口

##### 2.4.1 概述

* Retrofit将 Http请求 抽象成 Java接口：采用 **注解** 描述网络请求参数 和 配置网络请求参数
  * 用 动态代理 动态 将该接口的注解“翻译”成一个 Http 请求，最后再执行 Http 请求
  * 注：接口中的每个方法的参数都需要使用注解标注，否则会报错

##### 2.4.2 实例

```java
public interface GetRequest_Interface {

    @GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
    Call<Translation>  getCall();
    // @GET注解的作用:采用Get方法发送网络请求
 
    // getCall() = 接收网络请求数据的方法
    // 其中返回类型为Call<*>，*是接收数据的类（即上面定义的Translation类）
    // 如果想直接获得Responsebody中的内容，可以定义网络请求返回值为Call<ResponseBody>
}
```

* 此处表示使用的是GET的请求方法，关于注解的含义以及使用会在第三节详细讲解

#### 2.5 步骤4:创建 Retrofit 实例

```java
 Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fanyi.youdao.com/") // 设置网络请求的Url地址
                .addConverterFactory(GsonConverterFactory.create()) // 设置数据解析器
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava平台
                .build();
```

##### 2.5.1 解析器

* addConverterFactory方法就是给Retrofit添加解析器
* Retrofit支持多种数据解析方式
* 使用时需要在Gradle添加依赖

| 数据解析器 |                    Gradle依赖                    |
| ---------- | :----------------------------------------------: |
| Gson       |   com.squareup.retrofit2:converter-gson:2.x.x    |
| Jackson    |  com.squareup.retrofit2:converter-jackson:2.x.x  |
| Simple XML | com.squareup.retrofit2:converter-simplexml:2.x.x |
| Protobuf   | com.squareup.retrofit2:converter-protobuf:2.x.x  |
| Moshi      |   com.squareup.retrofit2:converter-moshi:2.x.x   |
| Wire       |   com.squareup.retrofit2:converter-wire:2.x.x    |
| Scalars    |  com.squareup.retrofit2:converter-scalars:2.x.x  |

##### 2.5.2 网络请求适配器

* addCallAdapterFactory给Retrofit添加网络请求适配器

* Retrofit支持多种网络请求适配器方式：guava、Java8和rxjava
  * 使用时如使用的是 `Android` 默认的 `CallAdapter`，则不需要添加网络请求适配器的依赖，否则则需要按照需求进行添加，Retrofit 提供的 `CallAdapter`
* 使用时需要在Gradle添加依赖：

| 网络请求适配器 |                 Gradle依赖                  |
| -------------- | :-----------------------------------------: |
| guava          | com.squareup.retrofit2:adapter-guava:2.x.x  |
| Java8          | com.squareup.retrofit2:adapter-java8:2.x.x  |
| rxjava         | com.squareup.retrofit2:adapter-rxjava:2.x.x |

#### 2.6 步骤5:创建 网络请求接口实例

```java
// 创建 网络请求接口 的实例
GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

//对 发送请求 进行封装
Call<Reception> call = request.getCall();
```

#### 2.7 步骤6:发送网络请求（异步 / 同步）

```java
//发送网络请求(异步)
call.enqueue(new Callback<Translation>() {
	//请求成功时回调
    @Override
    public void onResponse(Call<Translation> call, Response<Translation> response) {
    	//请求处理,输出结果
        response.body().show();
	}

    //请求失败时候的回调
    @Override
    public void onFailure(Call<Translation> call, Throwable throwable) {
    	System.out.println("连接失败");
	}
});

// 发送网络请求（同步）
Response<Reception> response = call.execute();
```

#### 2.8 步骤7:处理返回数据

* 通过`response`类的 `body()`对返回的数据进行处理

### 三、注解类型

![常见注解类型](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\常见注解类型.png)

#### 3.1 网络请求方法

![注解网络请求方法](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\注解网络请求方法.png)

##### 3.1.1 URL组成

* 其中@GET、@POST、@PUT、@DELETE、@HEAD就对应HTTP网络请求方法
* Retrofit把网络请求的URL分成了两部分设置

```java
// 第1部分：在网络请求接口的注解设置
@GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
Call<Translation>  getCall();

// 第2部分：在创建Retrofit实例时通过.baseUrl()设置
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fanyi.youdao.com/") //设置网络请求的Url地址
                .addConverterFactory(GsonConverterFactory.create()) //设置数据解析器
                .build();
```

* 所以 网络请求的完整URL = 在创建Retrofit实例时通过.baseUrl()设置 + 网络请求接口的注解设置

##### 3.1.2 @HTTP

- 作用：替换**@GET、@POST、@PUT、@DELETE、@HEAD**注解的作用 及 更多功能拓展
- 具体使用：通过属性**method、path、hasBody**进行设置

```java
public interface GetRequest_Interface {
    /**
     * method：网络请求的方法（区分大小写）
     * path：网络请求地址路径
     * hasBody：是否有请求体
     */
    @HTTP(method = "GET", path = "blog/{id}", hasBody = false)
    Call<ResponseBody> getCall(@Path("id") int id);
    // {id} 表示是一个变量
}
```

#### 3.2 标记

![注解标记](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\注解标记.png)

##### 3.2.1 @FormUrlEncoded

* 作用：表示发送form-encoded的数据

  * 每个键值对需要用@Filed来注解键名，随后的对象需要提供值。

##### 3.2.2 @Multipart

  - 作用：表示发送form-encoded的数据（适用于 有文件 上传的场景）

    * 每个键值对需要用@Part来注解键名，随后的对象需要提供值。

##### 3.2.3 实例

* 接口定义

```java
public interface GetRequest_Interface {
        /**
         *表明是一个表单格式的请求（Content-Type:application/x-www-form-urlencoded）
         * <code>Field("username")</code> 表示将后面的 <code>String name</code> 中name的取值作为 username 的值
         */
        @POST("/form")
        @FormUrlEncoded
        Call<ResponseBody> testFormUrlEncoded1(@Field("username") String name, @Field("age") int age);
         
        /**
         * {@link Part} 后面支持三种类型，{@link RequestBody}、{@link okhttp3.MultipartBody.Part} 、任意类型
         * 除 {@link okhttp3.MultipartBody.Part} 以外，其它类型都必须带上表单字段({@link okhttp3.MultipartBody.Part} 中已经包含了表单字段的信息)，
         */
        @POST("/form")
        @Multipart
        Call<ResponseBody> testFileUpload1(@Part("name") RequestBody name, @Part("age") RequestBody age, @Part MultipartBody.Part file);

}
```

* 使用接口

```java
// 具体使用
GetRequest_Interface service = retrofit.create(GetRequest_Interface.class);
// @FormUrlEncoded 
Call<ResponseBody> call1 = service.testFormUrlEncoded1("Carson", 24);
        
//@Multipart
RequestBody name = RequestBody.create(textType, "Carson");
RequestBody age = RequestBody.create(textType, "24");

MultipartBody.Part filePart = MultipartBody.Part.createFormData("file", "test.txt", file);
Call<ResponseBody> call3 = service.testFileUpload1(name, age, filePart);
```

#### 3.3 网络请求参数

![注解网络请求参数](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\注解网络请求参数.png)

##### 3.3.1 @Header & @Headers

* 作用：添加请求头 &添加不固定的请求头

* 实例

```java
// @Header
@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)

// @Headers
@Headers("Authorization: authorization")
@GET("user")
Call<User> getUser()

// 以上的效果是一致的。
// 区别在于使用场景和使用方式
// 1. 使用场景：@Header用于添加不固定的请求头，@Headers用于添加固定的请求头
// 2. 使用方式：@Header作用于方法的参数；@Headers作用于方法
```

##### 3.3.2 @Body

- 作用：以 `Post`方式 传递 自定义数据类型 给服务器
- 特别注意：如果提交的是一个Map，那么作用相当于 `@Field`

* 不过Map要经过 `FormBody.Builder` 类处理成为符合 OkHttp 格式的表单，如：

```java
FormBody.Builder builder = new FormBody.Builder();
builder.add("key","value");
```

##### 3.3.3 @Field & @FieldMap

- 作用：发送 Post请求 时提交请求的表单字段
- 具体使用：与 `@FormUrlEncoded` 注解配合使用

```java
public interface GetRequest_Interface {
        /**
         *表明是一个表单格式的请求（Content-Type:application/x-www-form-urlencoded）
         * <code>Field("username")</code> 表示将后面的 <code>String name</code> 中name的取值作为 username 的值
         */
        @POST("/form")
        @FormUrlEncoded
        Call<ResponseBody> testFormUrlEncoded1(@Field("username") String name, @Field("age") int age);

		/**
         * Map的key作为表单的键
         */
        @POST("/form")
        @FormUrlEncoded
        Call<ResponseBody> testFormUrlEncoded2(@FieldMap Map<String, Object> map);

}
```

##### 3.3.4 @Part & @PartMap

- 作用：发送 Post请求 时提交请求的表单字段
  * 与@Field的区别：功能相同，但携带的参数类型更加丰富，包括数据流，所以适用于 有文件上传 的场景
- 具体使用：与 `@Multipart` 注解配合使用
- 接口

```java
public interface GetRequest_Interface {

          /**
         * {@link Part} 后面支持三种类型，{@link RequestBody}、{@link okhttp3.MultipartBody.Part} 、任意类型
         * 除 {@link okhttp3.MultipartBody.Part} 以外，其它类型都必须带上表单字段({@link okhttp3.MultipartBody.Part} 中已经包含了表单字段的信息)，
         */
        @POST("/form")
        @Multipart
        Call<ResponseBody> testFileUpload1(@Part("name") RequestBody name, @Part("age") RequestBody age, @Part MultipartBody.Part file);

        /**
         * PartMap 注解支持一个Map作为参数，支持 {@link RequestBody } 类型，
         * 如果有其它的类型，会被{@link retrofit2.Converter}转换，如后面会介绍的 使用{@link com.google.gson.Gson} 的 {@link retrofit2.converter.gson.GsonRequestBodyConverter}
         * 所以{@link MultipartBody.Part} 就不适用了,所以文件只能用<b> @Part MultipartBody.Part </b>
         */
        @POST("/form")
        @Multipart
        Call<ResponseBody> testFileUpload2(@PartMap Map<String, RequestBody> args, @Part MultipartBody.Part file);

        @POST("/form")
        @Multipart
        Call<ResponseBody> testFileUpload3(@PartMap Map<String, RequestBody> args);
}
```

* 使用接口

```java
// 具体使用
MediaType textType = MediaType.parse("text/plain");
RequestBody name = RequestBody.create(textType, "Carson");
RequestBody age = RequestBody.create(textType, "24");
RequestBody file = RequestBody.create(MediaType.parse("application/octet-stream"), "这里是模拟文件的内容");

// @Part
MultipartBody.Part filePart = MultipartBody.Part.createFormData("file", "test.txt", file);
Call<ResponseBody> call3 = service.testFileUpload1(name, age, filePart);
ResponseBodyPrinter.printResponseBody(call3);

// @PartMap
// 实现和上面同样的效果
Map<String, RequestBody> fileUpload2Args = new HashMap<>();
fileUpload2Args.put("name", name);
fileUpload2Args.put("age", age);
Call<ResponseBody> call4 = service.testFileUpload2(fileUpload2Args, filePart); //单独处理文件
ResponseBodyPrinter.printResponseBody(call4);
}
```

##### 3.3.5 @Query和@QueryMap

* 作用：用于 `@GET` 方法的查询参数（Query = Url 中 ‘?’ 后面的 key-value）
  * 如：url = [http://www.println.net/?cate=android](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.println.net%2F%3Fcate%3Dandroid)，其中，Query = cate
* 具体使用：配置时只需要在接口方法中增加一个参数即可：

```java
@GET("/")    
Call<String> cate(@Query("cate") String cate);
```

##### 3.3.6 @Path

- 作用：URL地址的缺省值(替换)
- 具体使用：

```java
public interface GetRequest_Interface {

        @GET("users/{user}/repos")
        Call<ResponseBody>  getBlog（@Path("user") String user ）;
        // 访问的API是：https://api.github.com/users/{user}/repos
        // 在发起请求时， {user} 会被替换为方法的第一个参数 user（被@Path注解作用）
}
```

##### 3.3.7 @Url

- 作用：直接传入一个请求的 URL变量 用于URL设置
- 具体使用：

```java
public interface GetRequest_Interface {

        @GET
        Call<ResponseBody> testUrlAndQuery(@Url String url, @Query("showAll") boolean showAll);
       // 当有URL注解时，@GET传入的URL就可以省略
       // 当GET、POST...HTTP等方法中没有设置Url时，则必须使用 {@link Url}提供

}
```

#### 3.4 总结

![注解汇总](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\注解汇总.png)

### 四、实例核心代码

#### 4.1 金山词霸

* 接口地址(金山词霸API)：http://fy.iciba.com/ajax.php?a=fy&f=auto&t=auto&w=hello,world
* 主要分给几部分：NetUrl存储路径的后半部分，Api存储接口方法，AppConfig存储了路径的前半部分(接口前半部分都是同一的)，NetUtils提供获取Retrofit的方法，GetDataResultBean为返回类

* 查看核心源代码请点击[RxJava2+Retrofit2+OkHttp实例核心代码](https://github.com/nullWolf007/ToolProject/tree/master/RxJava2%2BRetrofit2%2BOkHttp实例核心代码)

