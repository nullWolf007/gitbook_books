[TOC]

## ModelLoader与Register机制

#### 参考

* [Glide开发之旅2-ModelLoader以及注册机制](https://www.jianshu.com/p/108309c2065a)

### 一、ModelLoader

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



# 一、ModelLoader

Glide通过ModelLoader完成图片的加载过程封装。

```java
/**
 *
 * @param <Mode> 转换前的类型，比如说file、url等等
 * @param <Data> 转换后的类型， 比如说Bitmap、outStream等等
 */
public interface ModelLoader<Mode, Data> {

    /**
     * 返回此loader是否能够处理对应Mode的数据
     * @param mode
     * @return
     */
    boolean handles(Mode mode);

    /**
     * 创建加载数据
     * @param mode
     * @return
     */
    LoadData<Data> buildData(Mode mode);

    class LoadData<Data> {

        //缓存的key，为了标注唯一性
        public final Key key;
        //加载数据
        public final DataFetcher<Data> dataFetcher;

        public LoadData(Key key, DataFetcher<Data> fetcher) {
            this.key = key;
            this.dataFetcher = fetcher;
        }
    }

    interface ModelLoaderFactory<Mode, Data> {

        ModelLoader<Mode, Data> build(ModelLoaderRegistry registry);
    }
}
```

使用Glide图片可能存在于文件、网络等地方。其中的Model则代表了加载来源模型：Uri、File等；Data则代表加载模型后的数据：InputStream、byte[]等。
 通过buildData函数创建LoadData。LoadData中的DataFetcher如下：



```java
/**
 * 负责加载数据的接口，至于怎么加载，看具体实现类
 * @param <Data>
 */
public interface DataFetcher<Data> {

    interface DataFetcherCallback<Data> {

        /**
         * 数据加载完成
         */
        void onFetcherReady(Data data);

        /**
         * 加载失败
         */
        void onLoadFailed(Exception e);
    }

    void loadData(DataFetcherCallback<Data> callback);

    void cancel();
}
```

来看一个ModelLoader实现：



```dart
/**
* 处理http请求的ModelLoader
* 将URI转换为InputStream
*/
public class HttpUrlLoader implements ModelLoader<Uri, InputStream> {

   /**
    * http类型的uri
    * @param uri
    * @return
    */
   @Override
   public boolean handles(Uri uri) {
       String scheme = uri.getScheme();
       return scheme.equalsIgnoreCase("http")
               || scheme.equalsIgnoreCase("https");
   }

   @Override
   public LoadData<InputStream> buildData(Uri uri) {

       /**
        * eg:以下这两个图片虽然一样，但是对象不一样，key的作用就是为了区分
        * Object 1 =  Uri.fromFile(new File("sdcard/a.png));
        * Object 2 =  Uri.fromFile(new File("sdcard/a.png));
        */
       return new LoadData<InputStream>(new ObjectKey(uri), new HttpUriFetcher(uri));
   }

   public static class Factory implements ModelLoaderFactory<Uri, InputStream> {

       @Override
       public ModelLoader<Uri, InputStream> build(ModelLoaderRegistry registry) {
           return new HttpUrlLoader();
       }
   }
}
```

对应的DataFetcher实现类：HttpUriFetcher



```java
public class HttpUriFetcher implements DataFetcher<InputStream> {

    private final Uri uri;
    private boolean isCanceled;

    public HttpUriFetcher(Uri uri) {
        this.uri = uri;
    }

    @Override
    public void loadData(DataFetcherCallback<InputStream> callback) {
        HttpURLConnection conn = null;
        InputStream is = null;
        try {
            URL url = new URL(uri.toString());
            conn = (HttpURLConnection) url.openConnection();
            conn.connect();
            is = conn.getInputStream();
            int responseCode = conn.getResponseCode();
            if (isCanceled) {
                return;
            }
            if (responseCode == HttpURLConnection.HTTP_OK) {
                callback.onFetcherReady(is);
            } else {
                callback.onLoadFailed(new RuntimeException(conn.getResponseMessage()));
            }
        }  catch (MalformedURLException e) {
            callback.onLoadFailed(e);
        } catch (IOException e) {
            callback.onLoadFailed(e);
        } finally {
            if (null != conn) {
                conn.disconnect();
            }
            if (null != is) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @Override
    public void cancel() {
        isCanceled = true;
    }
}
```

HttpUriLoader与之对应的HttpUriFetcher负责从网络获取图片，当使用Uri并且Uri满足：



![img](https:////upload-images.jianshu.io/upload_images/11690108-21602375c394cfe7.png?imageMogr2/auto-orient/strip|imageView2/2/w/487/format/webp)





HttpUriLoader则可以加载图片，如何加载已经在HttpUriFetcher中给出具体实现。

看起来还不错，但是我们需要支持其他来源的图片（如：文件），而不仅仅只能从网络获得，所以可能会有多个Loader与Fetcher实现，并且为了更加灵活的插拔，定义一个注册机（ModelLoaderRegistry）让我们对所有支持的来源进行注册、组装。

# 二、ModelLoaderRegistry

ModelLoaderRegistry负责注册所有支持的ModelLoader：



```dart
*/
public class ModelLoaderRegistry<Model, Data> {

    private List<Entry<Model, Data>> entries = new ArrayList<>();

    /**
     * 负责注册Loader
     * @param modelClass 数据来源类型 比如说url
     * @param dataClass 转换数据之后的类型 比如说inputStream
     * @param factory 创建ModelLoader的工厂
     */
    public synchronized void add(Class<Model> modelClass, Class<Data> dataClass,
                                 ModelLoader.ModelLoaderFactory<Model, Data> factory) {

        entries.add(new Entry<Model, Data>(modelClass, dataClass, factory));

    }

    public <Model, Data>ModelLoader<Model, Data> build(Class<Model> modelClass, Class<Data> dataClass) {

        List<ModelLoader<Model, Data>> loaders = new ArrayList<>();
        for (Entry<?, ?> entry : entries) {

            //是否我们需要的Model与Data类型的Loader
            if (entry.handles(modelClass, dataClass)) {
                loaders.add((ModelLoader<Model, Data>) entry.factory.build(this));
            }

        }
        if (loaders.size() > 1) {
            //表示我们找到了多个Loader
            return new MultiModelLoader<>(loaders);

        } else if (loaders.size() == 1) {
            return loaders.get(0);
        } else {
            throw new RuntimeException("Not Match :" + modelClass.getName() + ",Data :" + dataClass.getName());
        }
    }

    /**
     * 查找匹配的model类型的ModleLoader
     * @param modelClass
     * @param <Model>
     * @return
     */
    public <Model> List<ModelLoader<Model, ?>> getModelLoaders(Class<Model> modelClass) {
        List<ModelLoader<Model, ?>> loaders = new ArrayList<>();
        for (Entry<?, ?> entry : entries) {
            if (entry.handles(modelClass)) {
                loaders.add((ModelLoader<Model, ?>) entry.factory.build(this));
            }
        }
        return loaders;
    }

    private static class Entry<Model, Data> {
        Class<Model> modelClass;
        Class<Data> dataClass;
        ModelLoader.ModelLoaderFactory<Model, Data> factory;

        public Entry(Class<Model> modelClass, Class<Data> dataClass, ModelLoader.ModelLoaderFactory<Model, Data> factory) {
            this.modelClass = modelClass;
            this.dataClass = dataClass;
            this.factory = factory;
        }

        boolean handles(Class<?> modelClass, Class<?> dataClass) {
            //A.isAssignableFrom(B) 判断B和A是同一个类型，或者B是A的子类
            return this.modelClass.isAssignableFrom(modelClass)
                    && this.dataClass.isAssignableFrom(dataClass);
        }

        boolean handles(Class<?> modelClass) {
            // A.isAssignableFrom(B) B和A是同一个类型 或者 B是A的子类
            return this.modelClass.isAssignableFrom(modelClass);
        }
    }
}
```

使用之前必须调用add函数对需要组装的Loader进行注册：



![img](https:////upload-images.jianshu.io/upload_images/11690108-477091e2e80ac669.png?imageMogr2/auto-orient/strip|imageView2/2/w/845/format/webp)





当需要加载一个String类型的来源则会查找到StringLoader。但是一个String它可能属于文件地址也可能属于一个网络地址，所以StringLoader.StreamFactory在创建StringLoader的时候是：



```java
public class StringModelLoader implements ModelLoader<String, InputStream> {

    private final ModelLoader<Uri, InputStream> loader;

    public StringModelLoader(ModelLoader<Uri, InputStream> loader) {
        this.loader = loader;
    }

    @Override
    public boolean handles(String s) {
        return true;
    }

    @Override
    public LoadData<InputStream> buildData(String model) {
        Uri uri = null;
        if (model.startsWith("/")) {
            uri = Uri.fromFile(new File(model));
        } else {
            uri = Uri.parse(model);
        }
        return loader.buildData(uri);
    }

    public static class Factory implements ModelLoaderFactory<String, InputStream> {

        @Override
        public ModelLoader<String, InputStream> build(ModelLoaderRegistry registry) {
            return new StringModelLoader(registry.build(Uri.class, InputStream.class));
        }
    }
}
```

首先StringLoader装饰了一个ModelLoader<Uri,Data>类型的Loader，这个Loader来自StreamFactory中的modelRegistry.build(Uri.class,InputStream.class),回头查看到ModelLoaderRegistry的build函数：



![img](https:////upload-images.jianshu.io/upload_images/11690108-0669daded0e00c8b.png?imageMogr2/auto-orient/strip|imageView2/2/w/796/format/webp)

image.png



这个MultiModelLoader中存在一个集合，只要集合中存在一个Loader能够处理对应的Model，那么这个MultiModelLoader就可以处理对应的Model。
 所以当需要处理String类型的来源的时候，会创建一个MultiModelLoader，这个MultiModelLoader中就包含了一个HttpUriLoader与一个UriFileLoader。当字符串是以Http或者https开头则能由HttpUriLoader处理，否则交给UriFileLoader来加载：



![img](https:////upload-images.jianshu.io/upload_images/11690108-be386d6bb194ea55.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)




 定位到Loader，通过buildData获得一个LoadData，使用其中的Fetcher就可以加载到一个泛型Data类型的数据，比如InputStream。然后通过注册的解码器解码InputStream获得Bitmap。











