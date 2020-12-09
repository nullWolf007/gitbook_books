[TOC]

## Retrofit源码解析

#### 参考转载

* [Android：手把手带你 深入读懂 Retrofit 2.0 源码](https://www.jianshu.com/p/0c055ad46b6c)

### 一、基础知识

#### 1.1 前提阅读

* [Retrofit的使用](设计思想解读开源框架库\Retrofit\Retrofit的使用.md)

#### 1.2 Retrofit本质

* 其实Retrofit的本质是通过使用**大量的设计模式**进行**功能模块的解耦**，使得更加简单和流畅

![Retrofit本质](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\Retrofit本质.png)

* 通过解析 网络请求接口的注解 配置 网络请求参数
* 通过 动态代理 生成 网络请求对象
* 通过 网络请求适配器 将 网络请求对象 进行平台适配

* 通过 网络请求执行器 发送网络请求
* 通过 数据转换器 解析服务器返回的数据
* 通过 回调执行器 切换线程（子线程 ->>主线程）
* 用户在主线程处理返回结果

#### 1.3 角色说明

![Retrofit角色说明](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\Retrofit角色说明.png)

### 二、源码解析—创建Retrofit实例

#### 2.1 代码实例

```java
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fanyi.youdao.com/") // 设置网络请求的Url地址
                .addConverterFactory(GsonConverterFactory.create()) // 设置数据解析器
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava平台
                .build();
```

* 下面将对这段代码作详细解释

#### 2.2 Retrofit

##### 2.2.1 Retrofit#成员变量

* Retrofit.java

```java
public final class Retrofit {
    // 网络请求配置对象（对网络请求接口中方法注解进行解析后得到的对象）
  // 作用：存储网络请求相关的配置，如网络请求的方法、数据转换器、网络请求适配器、网络请求工厂、基地址等
  private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();

  // 网络请求器的工厂
  // 作用：生产网络请求器（Call）
  // Retrofit是默认使用okhttp
  private final okhttp3.Call.Factory callFactory;    
    
  private final HttpUrl baseUrl;// 网络请求的部分url地址
 
  // 数据转换器工厂的集合
  // 作用：放置数据转换器工厂
  // 数据转换器工厂作用：生产数据转换器（converter）
  private final List<Converter.Factory> converterFactories;
    
  // 网络请求适配器工厂的集合
  // 作用：放置网络请求适配器工厂
  // 网络请求适配器工厂作用：生产网络请求适配器（CallAdapter）
  // 下面会详细说明
  private final List<CallAdapter.Factory> adapterFactories;
  
    // 回调方法执行器
  private final Executor callbackExecutor;
    
  private final boolean validateEagerly;

```

* 其中CallAdapter会在接下俩详细介绍

##### 2.2.2 Retrofit#构造函数

```java
Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
      List<Converter.Factory> converterFactories, List<CallAdapter.Factory> adapterFactories,
      Executor callbackExecutor, boolean validateEagerly) {
    this.callFactory = callFactory;
    this.baseUrl = baseUrl;
    // unmodifiableList(list)近似于UnmodifiableList<E>(list)
    // 作用：创建的新对象能够对list数据进行访问，但不可通过该对象对list集合中的元素进行修改
    this.converterFactories = unmodifiableList(converterFactories); // Defensive copy at call site.
    this.adapterFactories = unmodifiableList(adapterFactories); // Defensive copy at call site.
    this.callbackExecutor = callbackExecutor;
    this.validateEagerly = validateEagerly;
  }
```

#### 2.3 CallAdapter

##### 2.3.1 概述

* 网络请求执行器（Call）的适配器
* Call在Retrofit里默认是`OkHttpCall`，在Retrofit中提供了四种CallAdapterFactory： ExecutorCallAdapterFactory（默认）、GuavaCallAdapterFactory、Java8CallAdapterFactory、RxJavaCallAdapterFactory

##### 2.3.2 作用

* 将默认的网络请求执行器（OkHttpCall）转换成适合被不同平台来调用的网络请求执行器形式
  * 如：一开始`Retrofit`只打算利用`OkHttpCall`通过`ExecutorCallbackCall`切换线程；但后来发现使用`Rxjava`更加方便（不需要Handler来切换线程）。想要实现`Rxjava`的情况，那就得使用`RxJavaCallAdapterFactoryCallAdapter`将`OkHttpCall`转换成`Rxjava(Scheduler)`：

```java
Retrofit.Builder().addCallAdapterFactory(RxJavaCallAdapterFactory.create())
```

##### 2.3.3 好处

* 用最小代价兼容更多平台，即能适配更多的使用场景

#### 2.4 Retrofit#Builder()

##### 2.4.1 Retrofit#Builder()

* 其实方法名字就能看出，这是建造者模式，那么我们看一下Builder类吧

```java
  public static final class Builder {
    private Platform platform;
    private okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private List<Converter.Factory> converterFactories = new ArrayList<>();
    private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
    private Executor callbackExecutor;
    private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
      converterFactories.add(new BuiltInConverters());
    }

    public Builder() {
      this(Platform.get());
    }
}
```

* 调用Builder方法，会调用Platform的get方法，去获取Platform实例对象
* 然后把Platform对象赋值给Builder的成员变量platform
* BuiltInConverters是一个内置的数据转换器工厂（继承Converter.Factory类），给converterFactories添加初始化的数据转换工厂

##### 2.4.2 Platform#get

```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

.....
```

* 可以看到调用Platform#get方法，最后会获得Platform的静态不可变成员变量PLATFORM，它的创建通过findPlatform去进行实例化的

##### 2.4.3 Platform#findPlatform

```java
  private static Platform findPlatform() {
    try {
        //Class.forName(xxx.xx.xx)的作用：要求JVM查找并加载指定的类（即JVM会执行该类的静态代码段）
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("org.robovm.apple.foundation.NSObject");
      return new IOS();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
```

* 可以看到Retrofit是支持多个平台的，上述又Android，Java和IOS。对于我们而言肯定是Android。调用了new Android()方法获取Platform对象

##### 2.4.4 Android

```java
  static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
        //创建默认的CallAdapter工厂类 ExecutorCallAdapterFactory
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
        //获取主线程的Handler
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```

* 可以看到Android继承自Platform，给设置了默认的CallAdapter工厂类，以及获取了主线程的Handler

##### 2.4.5 小结

* Retrofit#Builder()主要对其属性进行了初始化，赋值为默认的

#### 2.5 Retrofit#Builder#baseUrl()

##### 2.5.1 Retrofit#Builder#baseUrl()

```java
    public Builder baseUrl(String baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
        //分析1
      HttpUrl httpUrl = HttpUrl.parse(baseUrl);
      if (httpUrl == null) {
        throw new IllegalArgumentException("Illegal URL: " + baseUrl);
      }
        //分析2
      return baseUrl(httpUrl);
    }
```

* 分析1：把String类型的baseUrl转换成适合OkHttp的HttpUrl类型httpUrl
* 分析2：调用baseUrl方法，并把分析1处的httpUrl作为参数传递进去了

##### 2.5.2 Retrofit#Builder#baseUrl(httpUrl)

```java
public Builder baseUrl(HttpUrl baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      List<String> pathSegments = baseUrl.pathSegments();
      if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
      }
      this.baseUrl = baseUrl;
      return this;
}
```

* 把baseUrl赋值给Builder的成员变量baseUrl，然后返回此实例

#### 2.6 Retrofit#Builder#addConverterFactory()

##### 2.6.1 GsonConverterFactory

```java
.addConverterFactory(GsonConverterFactory.create())
```

* 可以看到先调用GsonConverterFactory.create()创建了一个实例，那么我们先看这个GsonConverterFactory

```java
public final class GsonConverterFactory extends Converter.Factory {
    //分析1
  public static GsonConverterFactory create() {
    return create(new Gson());
  }
    
	//分析2
  @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
  public static GsonConverterFactory create(Gson gson) {
    if (gson == null) throw new NullPointerException("gson == null");
    return new GsonConverterFactory(gson);
  }

  private final Gson gson;

  private GsonConverterFactory(Gson gson) {
    this.gson = gson;
  }
    ......
}
```

* GsonConverterFactory继承自Converter.Factory，所以可以放到addConverterFactory方法中
* 分析1：会调用new Gson来创建一个Gson对象，然后调用分析2的方法
* 分析2：会创建一个new GsonConverterFactory对象，然后把gson作为GsonConverterFactory的成员变量。
* 总结：GsonConverterFactory.creat()是创建了一个含有Gson对象实例的GsonConverterFactory，并返回给`addConverterFactory（）`

##### 2.6.2 Retrofit#Builder#addConverterFactory()

```java
public Builder addConverterFactory(Converter.Factory factory) {
      converterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
}
```

* 把上面创建的GsonConverterFactory实例对象，放入到Builder的成员变量converterFactories中。

#### 2.7 Retrofit#build()

* 这就是建造者模式的通用方法，其意义就是把Retrofit#Builder的成员变量赋值给Retrofit的成员变量

```java
public Retrofit build() {
 
 	  /*配置网络请求执行器（callFactory）*/
      okhttp3.Call.Factory callFactory = this.callFactory;
      // 如果没指定，则默认使用okhttp
      // 所以Retrofit默认使用okhttp进行网络请求
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

 	 /*配置回调方法执行器（callbackExecutor）*/
      Executor callbackExecutor = this.callbackExecutor;
      // 如果没指定，则默认使用Platform检测环境时的默认callbackExecutor
      // 即Android默认的callbackExecutor
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

 	  /*配置网络请求适配器工厂（CallAdapterFactory）*/
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      // 向该集合中添加创建的CallAdapter.Factory请求适配器（添加在集合器末尾）
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
    // 请求适配器工厂集合存储顺序：自定义1适配器工厂、自定义2适配器工厂...默认适配器工厂（ExecutorCallAdapterFactory）

 	  /*配置数据转换器工厂：converterFactory */
      // 在步骤2中已经添加了内置的数据转换器BuiltInConverters(）（添加到集合器的首位）
      // 在步骤4中又插入了一个Gson的转换器 - GsonConverterFactory（添加到集合器的首二位）
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
      // 数据转换器工厂集合存储的是：默认数据转换器工厂（ BuiltInConverters）、自定义1数据转换器工厂（GsonConverterFactory）、自定义2数据转换器工厂....

      // 最终返回一个Retrofit的对象，并传入上述已经配置好的成员变量
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```

* 获取合适的网络请求适配器和数据转换器都是从adapterFactories和converterFactories集合的首位开始遍历，因此集合中的工厂位置越靠前就拥有越高的使用权限
* 最后调用Retrofit的构造函数把其值付给Retrofit的成员变量。

#### 2.8 小结

Retrofit**使用建造者模式通过Builder类**建立了一个Retrofit实例，具体创建细节是配置了：

- 平台类型对象（Platform - Android）
- 网络请求的url地址（baseUrl）
- 网络请求工厂（callFactory）：默认使用OkHttpCall
- 网络请求适配器工厂的集合（adapterFactories）：本质是配置了网络请求适配器工厂- 默认是ExecutorCallAdapterFactory

- 数据转换器工厂的集合（converterFactories）：本质是配置了数据转换器工厂-默认是BuiltInConverters

- 回调方法执行器（callbackExecutor）：默认回调方法执行器作用是：切换线程（子线程 - 主线程）。对于Android而言默认的就是Platform#Androidr#MainThreadExecutor

### 三、源码分析-网络请求

#### 3.1 实例

* 接口：http://fy.iciba.com/ajax.php?a=fy&f=auto&t=auto&w=hello,world

* Api.java

```java
public interface Api {
    @GET("/ajax.php")
    Call<GetDataResultBean> getData(@QueryMap Map<String, String> map);
}
```

* GetDataResultBean.java:返回数据
* MainActivity.java

```java
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fy.iciba.com")
                .addConverterFactory(GsonConverterFactory.create())
                .build();

Api api = retrofit.create(Api.class);
//a=fy&f=auto&t=auto&w=hello%20world
HashMap<String, String> hashMap = new HashMap<>();
hashMap.put("a", "fy");
hashMap.put("f", "auto");
hashMap.put("t", "auto");
hashMap.put("w", "hello,World");

Call<GetDataResultBean> call = api.getData(hashMap);
```

* 接下来主要对MainActivity中的代码进行源码解析

#### 3.2 Retrofit#create

##### 3.2.1 Retrofit#create()

* 上面调用了retrofit.create(Api.class)方法，那么我们看一下源码

```java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
	//分析1
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    
	//分析2
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),  // 动态生成接口的实现类 
        new Class<?>[] { service },// 动态创建实例
        new InvocationHandler() {// 将代理类的实现交给 InvocationHandler类作为具体的实现（下面会解释）
          private final Platform platform = Platform.get();

          // 在 InvocationHandler类的invoke()实现中，除了执行真正的逻辑（如再次转发给真正的实现类对象），还可以进行一些有用的操作
         // 如统计执行时间、进行初始化和清理、对接口调用进行检查等。
          @Override public Object invoke(Object proxy, Method method, 
                                         @Nullable Object[] args)throws Throwable {
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 下面会详细介绍 invoke（）的实现
            // 即下面三行代码
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

* 分析1：判断是否需要提前验证，然后调用eagerlyValidateMethods方法。
* 分析2：创建了网络请求接口的动态代理对象，即通过动态代理创建网络请求接口的实例 （并最终返回）。该动态代理是为了拿到网络请求接口实例上所有注解

##### 3.2.2 Retrofit#eagerlyValidateMethods

```java
  private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
      //分析1
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);
      }
    }
  }
```

* 通过传入的Class对象，也就是我们接口类的Class对象，即Api.Class。
* 分析1：通过反射技术getDeclaredMethods，拿到Api类中所有的方法，然后对其进行遍历，然后调用loadServiceMethod方法，放入到serviceMethodCache中，由于下面也涉及此方法，将在后面一起说明

#### 3.3  InvocationHandler#invoke

##### 3.3.1 InvocationHandler#invoke

* 有关代理模式请点击[代理模式](Android必备技能/Hook/代理模式.md)
* 由于传入的是Api的ClassLoader，所以调用Api的方法，就会触发作为参数传入的InvocationHandler的invoke方法

```java
new InvocationHandler() {
	private final Platform platform = Platform.get();
       
    @Override public Object invoke(Object proxy, Method method, 
                                   @Nullable Object[] args)throws Throwable {
		if (method.getDeclaringClass() == Object.class) {
        	return method.invoke(this, args);
		}
        if (platform.isDefaultMethod(method)) {
        	return platform.invokeDefaultMethod(method, service, proxy, args);
		}
        //分析1
        ServiceMethod<Object, Object> serviceMethod =
        	(ServiceMethod<Object, Object>) loadServiceMethod(method);
        //分析2
		OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
        //分析3
        return serviceMethod.callAdapter.adapt(okHttpCall);
	}
}
```

* 分析1：调用loadServiceMethod，读取Api中的方法，并根据Method生成ServiceMethod对象，见3.4
* 分析2：根据第一步配置好的`ServiceMethod`对象和输入的请求参数创建`okHttpCall`对象，见3.5
* 分析3：将第二步创建的`OkHttpCall`对象传给第一步创建的`serviceMethod`对象中对应的网络请求适配器工厂的`adapt（）`，见3.6

##### 3.3.2 Retrofit#loadServiceMethod

* serviceMethodCache是Retrofit成员变量，用来存储的

```java
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
```

```java
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    //采用同步方法
    synchronized (serviceMethodCache) {
        //分析1
      result = serviceMethodCache.get(method);
        //分析2
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

* 分析1：如果serviceMethodCache中已经包含了此方法，说明不需要往里加了，serviceMethodCache中已经有了。
* 分析2：如果没有，则调用new ServiceMethod.Builder(this, method).build()根据Method创建一个ServiceMethod，然后放入到缓存serviceMethodCache中。具体见3.4

#### 3.4 ServiceMethod

##### 3.4.1 ServiceMethod#成员变量

```java
public final class ServiceMethod {
    final okhttp3.Call.Factory callFactory;   // 网络请求工厂  

    // 网络请求适配器工厂
    // 具体创建是在new ServiceMethod.Builder(this, method).build()最后的build()中
    // 下面会详细说明
    final CallAdapter<?> callAdapter;  

    // Response内容转换器  
    // 作用：负责把服务器返回的数据（JSON或者其他格式，由 ResponseBody 封装）转化为 T 类型的对象；
    private final Converter<ResponseBody, T> responseConverter; 


    private final HttpUrl baseUrl; // 网络请求地址  
    private final String relativeUrl; // 网络请求的相对地址  
    private final String httpMethod;   // 网络请求的Http方法  
    private final Headers headers;  // 网络请求的http请求头 键值对  
    private final MediaType contentType; // 网络请求的http报文body的类型  

    // 方法参数处理器
    // 作用：负责解析 API 定义时每个方法的参数，并在构造 HTTP 请求时设置参数；
    // 下面会详细说明
    private final ParameterHandler<?>[] parameterHandlers;  
```

* 从上面的成员变量可以看出，ServiceMethod对象包含了访问网络的所有基本信息

##### 3.4.2 ServiceMethod#构造方法

```java
// 作用：传入各种网络请求参数
ServiceMethod(Builder<T> builder) {
    this.callFactory = builder.retrofit.callFactory();  
    this.callAdapter = builder.callAdapter;   
    this.responseConverter = builder.responseConverter;   
  
    this.baseUrl = builder.retrofit.baseUrl();   
    this.relativeUrl = builder.relativeUrl;   
    this.httpMethod = builder.httpMethod;  
    this.headers = builder.headers;  
    this.contentType = builder.contentType; .  
    this.hasBody = builder.hasBody; y  
    this.isFormEncoded = builder.isFormEncoded;   
    this.isMultipart = builder.isMultipart;  
    this.parameterHandlers = builder.parameterHandlers;  
}
```

* 把ServiceMethod#Builder的值赋值到ServiceMethod成员变量，建造者模式

##### 3.4.3 ServiceMethod#Builder()

```java
public Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
    //分析1
      this.methodAnnotations = method.getAnnotations();
    //分析2
      this.parameterTypes = method.getGenericParameterTypes();
    //分析3
      this.parameterAnnotationsArray = method.getParameterAnnotations();
}
```

* 分析1：通过反射getAnnotations方法，获取此接口方法的注解。
* 分析2：通过反射getGenericParameterTypes方法，获取此接口方法里的参数类型。
* 分析3：通过反射getParameterAnnotations方法，获取此接口方法里的参数注解

```java
@GET("/ajax.php")
Call<GetDataResultBean> getData(@QueryMap Map<String, String> map);
```

* 分析1：拿到的@GET
* 分析2：拿到的Map
* 分析3：拿到的@QueryMap

##### 3.4.4 ServiceMethod#build()

```java
// 作用：控制ServiceMethod对象的生成流程
public ServiceMethod build() {

     // 根据网络请求接口方法的返回值和注解类型，从Retrofit对象中获取对应的网络请求适配器  -->关注点1
      callAdapter = createCallAdapter();    
     
    // 根据网络请求接口方法的返回值和注解类型，从Retrofit对象中获取该网络适配器返回的数据类型
      responseType = callAdapter.responseType();    
     
      // 根据网络请求接口方法的返回值和注解类型，从Retrofit对象中获取对应的数据转换器  -->关注点3
      // 构造 HTTP 请求时，我们传递的参数都是String
      // Retrofit 类提供 converter把传递的参数都转化为 String 
      // 其余类型的参数都利用 Converter.Factory 的stringConverter 进行转换
      // @Body 和 @Part 类型的参数利用Converter.Factory 提供的 requestBodyConverter 进行转换
      // 这三种 converter 都是通过“询问”工厂列表进行提供，而工厂列表我们可以在构造 Retrofit 对象时进行添加。
      responseConverter = createResponseConverter();    
   
      
       
      // 解析网络请求接口中方法的注解
      // 主要是解析获取Http请求的方法
     // 注解包括：DELETE、GET、POST、HEAD、PATCH、PUT、OPTIONS、HTTP、retrofit2.http.Headers、Multipart、FormUrlEncoded
     // 处理主要是调用方法 parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) ServiceMethod中的httpMethod、hasBody、relativeUrl、relativeUrlParamNames域进行赋值
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
      
      // 获取当前方法的参数数量
     int parameterCount = parameterAnnotationsArray.length;
     
      
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        // 为方法中的每个参数创建一个ParameterHandler<?>对象并解析每个参数使用的注解类型
        // 该对象的创建过程就是对方法参数中注解进行解析
        // 这里的注解包括：Body、PartMap、Part、FieldMap、Field、Header、QueryMap、Query、Path、Url 
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      } 
      return new ServiceMethod<>(this);

    }
```

* 根据返回值类型和方法标注从Retrofit对象的的网络请求适配器工厂集合和内容转换器工厂集合中分别获取到该方法对应的网络请求适配器和Response内容转换器；
* 根据方法的标注对ServiceMethod的域进行赋值
* 最后为每个方法的参数的标注进行解析，获得一个ParameterHandler<?>对象
* 该对象保存有一个Request内容转换器——根据参数的类型从Retrofit的内容转换器工厂集合中获取一个Request内容转换器或者一个String内容转换器。

###### 3.4.4.1 createCallAdapter

```java
 private CallAdapter<?> createCallAdapter() {

      // 获取网络请求接口里方法的返回值类型
      Type returnType = method.getGenericReturnType();      

      // 获取网络请求接口接口里的注解
      // 此处使用的是@Get
      Annotation[] annotations = method.getAnnotations();       
      try {

      	return retrofit.callAdapter(returnType, annotations); 
      // 根据网络请求接口方法的返回值和注解类型，从Retrofit对象中获取对应的网络请求适配器
      // 下面会详细说明retrofit.callAdapter（） -- >关注点2
      }
...

```

* 调用callAdapter方法

###### 3.4.4.2 callAdapter

```java
 public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

 public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {

    // 创建 CallAdapter 如下
    // 遍历 CallAdapter.Factory 集合寻找合适的工厂（该工厂集合在第一步构造 Retrofit 对象时进行添加（第一步时已经说明））
    // 如果最终没有工厂提供需要的 CallAdapter，将抛出异常
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);      
      if (adapter != null) {
        return adapter;
      }
    }
```

###### 3.4.4.3 createResponseConverter

```java
 private Converter<ResponseBody, T> createResponseConverter() {
      Annotation[] annotations = method.getAnnotations();
      try {
    
        // responseConverter 还是由 Retrofit 类提供  -->关注点4
        return retrofit.responseBodyConverter(responseType, annotations);
      } catch (RuntimeException e) { 
        throw methodError(e, "Unable to create converter for %s", responseType);
      }
    }
```

###### 3.4.4.4 responseBodyConverter

```java
  public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
  }

 public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {

       // 获取Converter 过程：（和获取 callAdapter 基本一致）
         Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this); 
       // 遍历 Converter.Factory 集合并寻找合适的工厂（该工厂集合在构造 Retrofit 对象时进行添加（第一步时已经说明））
       // 由于构造Retroifit采用的是Gson解析方式，所以取出的是GsonResponseBodyConverter
       // Retrofit - Converters 还提供了 JSON，XML，ProtoBuf 等类型数据的转换功能。
       // 继续看responseBodyConverter（） -->关注点5    
    }
```

###### 3.4.4.5 responseBodyConverter

```java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(Type type, 
    Annotation[] annotations, Retrofit retrofit) {

  
  TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
  // 根据目标类型，利用 Gson#getAdapter 获取相应的 adapter
  return new GsonResponseBodyConverter<>(gson, adapter);
}
```

###### 3.4.4.6 GsonResponseBodyConverter

```java
// 做数据转换时调用 Gson 的 API 即可。
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override 
   public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
}
```

#### 3.5 OkHttpCall

##### 3.5.1 OkHttpCall#成员变量

```java
public class OkHttpCall {
    private final ServiceMethod<T> serviceMethod; // 含有所有网络请求参数信息的对象  
    private final Object[] args; // 网络请求接口的参数 
    private okhttp3.Call rawCall; //实际进行网络访问的类  
    private Throwable creationFailure; //几个状态标志位  
    private boolean executed;  
    private volatile boolean canceled;  
```

##### 3.5.2 OkHttpCall#构造函数

* 3.3.1InvocationHandler#invoke最终会调用此构造函数

```java
OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
    // 传入了配置好的ServiceMethod对象和输入的请求参数
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
```

#### 3.6 适配器#adapter

* 如果没有设置的话，默认是ExecutorCallAdapterFactory，这里就以默认的作说明

##### 3.6.1 返回对象类型

* Android默认的是`Call<>`；
* 若设置了RxJavaCallAdapterFactory，返回的则是`Observable<>`

##### 3.6.2 ExecutorCallAdapterFactory#adapter

```java
public <R> Call<R> adapt(Call<R> call) {
                    return new ExecutorCallAdapterFactory.ExecutorCallbackCall(ExecutorCallAdapterFactory.this.callbackExecutor, call);
                }
```

* 调用ExecutorCallAdapterFactory的ExecutorCallbackCall方法

##### 3.6.3 ExecutorCallAdapterFactory#ExecutorCallbackCall

```java
ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
    // 传入上面定义的回调方法执行器
      // 用于进行线程切换  
	this.callbackExecutor = callbackExecutor;
    
    // 把上面创建并配置好参数的OkhttpCall对象交给静态代理delegate
      // 静态代理和动态代理都属于代理模式
     // 静态代理作用：代理执行被代理者的方法，且可在要执行的方法前后加入自己的动作，进行对系统功能的拓展
	this.delegate = delegate;
}
```

#### 3.7 获取数据

* 上面的实例通过如下代码去获取数据

```java
Call<GetDataResultBean> call = api.getData(hashMap);
```

##### 3.7.1 流程

* 由于动态代理的关系，当api调用getData方法时，会进入InvocationHandler # invoke方法
* `invoke(Object proxy, Method method, Object... args)`会传入3个参数
  * `Object proxy:`（代理对象）
  *  `Method method`（调用的`getData()`）
  *  `Object... args`（方法的参数，即`getData（*）`中的*）
* 在InvocationHandler # invoke方法中，利用反射技术获取getData()的注解信息，然后配合args，生成ServiceMethod对象
* 然后根据此ServiceMethod对象获取OkHttpCall类型对象

#### 3.8 总结

* 动态创建网络请求接口的实例**（代理模式 - 动态代理）**
* 创建 `serviceMethod` 对象**（建造者模式 & 单例模式（缓存机制））**
* 对 `serviceMethod` 对象进行网络请求参数配置：通过解析网络请求接口方法的参数、返回值和注解类型，从Retrofit对象中获取对应的网络请求的url地址、网络请求执行器、网络请求适配器 & 数据转换器。**（策略模式）**
* 对 `serviceMethod` 对象加入线程切换的操作，便于接收数据后通过Handler从子线程切换到主线程从而对返回数据结果进行处理**（装饰模式）**
* 最终创建并返回一个`OkHttpCall`类型的网络请求对象

### 四、执行网络请求

* Retrofit默认使用的时OkHttp，所以对应网络请求为OkHttpCall
* 可阅读[一、OkHttp大体流程](设计思想解读开源框架库\网络访问框架\OkHttp\一、OkHttp大体流程.md)和[二、OkHttp连接池复用](设计思想解读开源框架库\网络访问框架\OkHttp\二、OkHttp连接池复用.md)了解OkHttp原理

#### 4.1 同步请求 OkHttpCall.execute

##### 4.1.1 流程

* **步骤1：**对网络请求接口的方法中的每个参数利用对应`ParameterHandler`进行解析，再根据`ServiceMethod`对象创建一个`OkHttp`的`Request`对象
* **步骤2：**使用`OkHttp`的`Request`发送网络请求；
* **步骤3：**对返回的数据使用之前设置的数据转换器（GsonConverterFactory）解析返回的数据，最终得到一个`Response`对象

##### 4.1.2 OkHttpCall#execute

```java
@Override 
public Response<T> execute() throws IOException {
  okhttp3.Call call;

 // 设置同步锁
  synchronized (this) {
    call = rawCall;
    if (call == null) {
      try {
        // 分析1：创建一个OkHttp的Request对象请求
        call = rawCall = createRawCall();
      } catch (IOException | RuntimeException e) {
        creationFailure = e;
        throw e;
      }
    }
  }

    //分析2
	// 调用OkHttpCall的execute()发送网络请求（同步）
  // 解析网络请求返回的数据parseResponse（） -->关注2
  return parseResponse(call.execute());
}
```

* 分析1：调用createRawCall创建一个okhttp3.Call请求
* 分析2：调用okhttp3.Call的execute（其实时OkHttp的RealCall），然后调用parseResponse去解析返回的数据

##### 4.1.3 createRawCall

```java
private okhttp3.Call createRawCall() throws IOException {
  // 从ServiceMethod的toRequest（）返回一个Request对象
  Request request = serviceMethod.toRequest(args);
  // 根据serviceMethod和request对象创建 一个okhttp3.Request
  okhttp3.Call call = serviceMethod.callFactory.newCall(request);

  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

##### 4.1.4 RealCall#execute

* 这个可阅读[一、OkHttp大体流程](设计思想解读开源框架库\网络访问框架\OkHttp\一、OkHttp大体流程.md)

##### 4.1.5 parseResponse

```java
<--  关注2：parseResponse（）-->
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();
  // 收到返回数据后进行状态码检查
  // 具体关于状态码说明下面会详细介绍
  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
  }

  if (code == 204 || code == 205) {
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
      // 等Http请求返回后 & 通过状态码检查后，将response body传入ServiceMethod中，ServiceMethod通过调用Converter接口（之前设置的GsonConverterFactory）将response body转成一个Java对象，即解析返回的数据
    T body = serviceMethod.toResponse(catchingBody);

	// 生成Response类
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    ... // 异常处理
  }
}
```

##### 4.1.6 注意

* `ServiceMethod`几乎保存了一个网络请求所需要的数据

* 发送网络请求时，`OkHttpCall`需要从`ServiceMethod`中获得一个Request对象

* 解析数据时，还需要通过`ServiceMethod`使用`Converter`（数据转换器）转换成Java对象进行数据解析

##### 4.1.7 状态码

![状态码](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\状态码.png)

#### 4.2 异步请求 OkHttpCall.enqueue

##### 4.2.1 流程

* **步骤1：**对网络请求接口的方法中的每个参数利用对应`ParameterHandler`进行解析，再根据`ServiceMethod`对象创建一个`OkHttp`的`Request`对象
* **步骤2：**使用`OkHttp`的`Request`发送网络请求；
* **步骤3：**对返回的数据使用之前设置的数据转换器（GsonConverterFactory）解析返回的数据，最终得到一个`Response`对象
* **步骤4：**进行线程切换从而在主线程处理返回的数据结果

##### 4.2.2 实例

```java
call.enqueue(new Callback<JavaBean>() {
            @Override
            public void onResponse(Call<JavaBean> call, Response<JavaBean> response) {
                System.out.println(response.isSuccessful());
                if (response.isSuccessful()) {
                    response.body().show();
                }
                else {
                    try {
                        System.out.println(response.errorBody().string());
                    } catch (IOException e) {
                        e.printStackTrace();
                    } ;
                }
            }
```

- 从上面分析有：`call`是一个静态代理
- 使用静态代理的作用是：在okhttpCall发送网络请求的前后进行额外操作
  * 这里的额外操作是：线程切换，即将子线程切换到主线程，从而在主线程对返回的数据结果进行处理

#####  4.2.3 ExecutorCallAdapterFactory#enqueue

```java
@Override 
public void enqueue(final Callback<T> callback) {

      delegate.enqueue(new Callback<T>() {
     // 使用静态代理 delegate进行异步请求 ->>分析1
     // 等下记得回来
        @Override 
        public void onResponse(Call<T> call, final Response<T> response) {
          // 步骤4：线程切换，从而在主线程显示结果
          callbackExecutor.execute(new Runnable() {
          // 最后Okhttp的异步请求结果返回到callbackExecutor
          // callbackExecutor.execute()通过Handler异步回调将结果传回到主线程进行处理（如显示在Activity等等），即进行了线程切换
          // 具体是如何做线程切换 ->>分析2
              @Override 
               public void run() {
              if (delegate.isCanceled()) {
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override 
        public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```

* delegate就是静态代理类，实际上调用的还是OkHttpCall的enqueue方法

##### 4.2.4 OkHttpCall#enqueue

```java
@Override 
public void enqueue(final Callback<T> callback) {

    okhttp3.Call call;
    Throwable failure;

    //分析1
    synchronized (this) {
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    ......

        //分析2
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```

* 分析1：通过createRawCall方法获取OkHttp.Call对象，这里和上面是一样的，就不说明了
* 分析2：调用OkHttp的Call的enqueue去异步执行，其实时RealCall的enqueue方法去执行。

##### 4.2.5 线程切换

* 线程切换是通过一开始创建Retrofit对象时Platform在检测到运行环境是Android时进行创建的

```java
// 采用适配器模式
static class Android extends Platform {

    // 创建默认的回调执行器工厂
    // 如果不将RxJava和Retrofit一起使用，一般都是使用该默认的CallAdapter.Factory
    // 后面会对RxJava和Retrofit一起使用的情况进行分析
    @Override
      CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    @Override 
      public Executor defaultCallbackExecutor() {
      // 返回一个默认的回调方法执行器
      // 该执行器负责在主线程（UI线程）中执行回调方法
      return new MainThreadExecutor();
    }

    // 获取主线程Handler
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());


      @Override 
      public void execute(Runnable r) {
        // Retrofit获取了主线程的handler
        // 然后在UI线程执行网络请求回调后的数据显示等操作。
        handler.post(r);
      }
    }
  }
```

**切换线程的流程**

* 首先Platform会找到主线程的Handler，生成一个MainThreadExecutor实例
* Retrofit调用build()方法时，最终就会把MainThreadExecutor实例callbackExecutor传递到Retrofit中，同时使用Platform的defaultCallAdapterFactory使用callbackExecutor创建了CallAdapter

```java
callbackExecutor = platform.defaultCallbackExecutor();
List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
......
return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
```

* 这样ExecutorCallAdapterFactory中的callbackExecutor其实就是MainThreadExecutor实例

* ExecutorCallAdapterFactory的enqueue调用了callbackExecutor的execute方法，其实就是Platform中Android的MainThreadExecutor的execute方法，由于其是主线程的Handler，进而切换到了主线程

### 五、总结

#### 5.1 过程

* `Retrofit` 将 `Http`请求 抽象 成 `Java`接口
* 在接口里用 注解 描述和配置 网络请求参数
* 用动态代理 的方式，动态将网络请求接口的注解 解析 成`HTTP`请求
* 最后执行`HTTP`请求

#### 5.2 源码分析图

![Retrofit源码分析图](..\..\images\设计思想解读开源框架库\网络访问框架\Retrofit\Retrofit源码分析图.png)

