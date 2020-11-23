[TOC]

# WebView详细介绍

#### 参考

* [Android：这是一份全面 & 详细的Webview使用攻略](https://www.jianshu.com/p/3c94ae673e2a)

### 一、概述

#### 1.1 前言

* 现在越来越多的应用都使用WebView来内置web网页，比如淘宝啥的

#### 1.2 简介

* `WebView`是一个基于`webkit`引擎、展现`web`页面的控件。
  * Android的Webview在低版本和高版本采用了不同的webkit版本内核，4.4后直接使用了Chrome。

#### 1.2 WebView作用

* 显示和渲染Web网页
* 直接使用html文件(网络上或本地assets做布局
* 可和javascript作交互使用

### 二、WebView常用方法

#### 2.1 加载url

```java
//方式1. 加载一个网页：
webView.loadUrl("http://www.google.com/");

//方式2：加载apk包中的html页面
webView.loadUrl("file:///android_asset/test.html");

//方式3：加载手机本地的html页面
webView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");

// 方式4： 加载 HTML 页面的一小段内容
WebView.loadData(String data, String mimeType, String encoding)
// 参数说明：
// 参数1：需要截取展示的内容
// 内容里不能出现 ’#’, ‘%’, ‘\’ , ‘?’ 这四个字符，若出现了需用 %23, %25, %27, %3f 对应来替代，否则会出现异常
// 参数2：展示内容的类型
// 参数3：字节码
```

#### 2.2 WebView的状态

```java
//激活WebView为活跃状态，能正常执行网页的响应
webView.onResume() ；

//当页面被失去焦点被切换到后台不可见状态，需要执行onPause
//通过onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。
webView.onPause()；

//当应用程序(存在webview)被切换到后台时，这个方法不仅仅针对当前的webview而是全局的全应用程序的webview
//它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
webView.pauseTimers()
//恢复pauseTimers状态
webView.resumeTimers()；

//销毁Webview
//在关闭了Activity时，如果Webview的音乐或视频，还在播放。就必须销毁Webview
//但是注意：webview调用destory时,webview仍绑定在Activity上
//这是由于自定义webview构建时传入了该Activity的context对象
//因此需要先从父容器中移除webview,然后再销毁webview:
rootLayout.removeView(webView); 
webView.destroy();
```
#### 2.3 关于前进和后退网页

```java
//是否可以后退
Webview.canGoBack() 
//后退网页
Webview.goBack()

//是否可以前进                     
Webview.canGoForward()
//前进网页
Webview.goForward()

//以当前的index为起始点前进或者后退到历史记录中指定的steps
//如果steps为负数则为后退，正数则为前进
Webview.goBackOrForward(intsteps) 
```
**重点：**
* 问题：在不作任何处理的时候，点击Back键，整个Browser会调用finish()而结束自身(也就是点击back会退出当前浏览器)
* 目标：点击back，是网页的回退，而不是退出浏览器
* 解决方案：在Activity中监听Back事件，在方法中调用webview的回退方法
```java
public boolean onKeyDown(int keyCode, KeyEvent event) {
    if ((keyCode == KEYCODE_BACK) && mWebView.canGoBack()) { 
        mWebView.goBack();
        return true;
    }
    return super.onKeyDown(keyCode, event);
}
```

#### 2.3 清除缓存

```java
//清除网页访问留下的缓存
//由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
Webview.clearCache(true);

//清除当前webview访问的历史记录
//只会webview访问历史记录里的所有记录除了当前访问记录
Webview.clearHistory()；

//这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据
Webview.clearFormData()；
```

### 三、常用类

#### 3.1 WebSetting类

* 作用：对WebView进行配置和管理

* 配置步骤 & 常见方法：
##### 3.1.1 配置步骤1:对于webview需要访问网络的权限添加

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```
##### 3.1.2 配置步骤2：生成webview组件(两种方式)

```java
//方式1：直接在在Activity中生成
WebView webView = new WebView(this)

//方法2：在Activity的layout文件里添加webview控件：
WebView webview = (WebView) findViewById(R.id.webView1);
```

##### 3.1.3 配置步骤3：使用WebSetting进行配置

```java
//声明WebSettings子类
WebSettings webSettings = webView.getSettings();

//如果访问的页面中要与Javascript交互，则webview必须设置支持Javascript
webSettings.setJavaScriptEnabled(true);  

//支持插件
webSettings.setPluginsEnabled(true); 

//设置自适应屏幕，两者合用
webSettings.setUseWideViewPort(true); //将图片调整到适合webview的大小 
webSettings.setLoadWithOverviewMode(true); // 缩放至屏幕的大小

//缩放操作
webSettings.setSupportZoom(true); //支持缩放，默认为true。是下面那个的前提。
webSettings.setBuiltInZoomControls(true); //设置内置的缩放控件。若为false，则该WebView不可缩放
webSettings.setDisplayZoomControls(false); //隐藏原生的缩放控件

//其他细节操作
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); //关闭webview中缓存 
webSettings.setAllowFileAccess(true); //设置可以访问文件 
webSettings.setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口 
webSettings.setLoadsImagesAutomatically(true); //支持自动加载图片
webSettings.setDefaultTextEncodingName("utf-8");//设置编码格式
```
#### 3.2 设置webview缓存

* 当加载 html 页面时，WebView会在/data/data/包名目录下生成 database 与 cache 两个文件夹
* 请求的 URL记录保存在 WebViewCache.db，而 URL的内容是保存在 WebViewCache 文件夹下
* 是否启用缓存
```javascript
    //优先使用缓存: 
    WebView.getSettings().setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); 
        //缓存模式如下：
        //LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据
        //LOAD_DEFAULT: （默认）根据cache-control决定是否从网络上取数据。
        //LOAD_NO_CACHE: 不使用缓存，只从网络获取数据.
        //LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据。

    //不使用缓存: 
    WebView.getSettings().setCacheMode(WebSettings.LOAD_NO_CACHE);
```

* 结合使用(离线加载)
```java
if (NetStatusUtil.isConnected(getApplicationContext())) {
    webSettings.setCacheMode(WebSettings.LOAD_DEFAULT);//根据cache-control决定是否从网络上取数据。
} else {
    webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);//没网，则从本地获取，即离线加载
}

webSettings.setDomStorageEnabled(true); // 开启 DOM storage API 功能
webSettings.setDatabaseEnabled(true);   //开启 database storage API 功能
webSettings.setAppCacheEnabled(true);//开启 Application Caches 功能

String cacheDirPath = getFilesDir().getAbsolutePath() + APP_CACAHE_DIRNAME;
webSettings.setAppCachePath(cacheDirPath); //设置  Application Caches 缓存目录
```

#### 3.3 WebViewClient类

* 作用：处理各种通知和请求事件
* 常见方法

##### 3.3.1 shouldOverrideUrlLoading()

* 作用：打开网页时不调用系统浏览器， 而是在本WebView中显示；在网页上的所有加载都经过这个方法,这个函数我们可以做很多操作。

```java
//步骤1. 定义Webview组件
Webview webview = (WebView) findViewById(R.id.webView1);

//步骤2. 选择加载方式
//方式1. 加载一个网页：
webView.loadUrl("http://www.google.com/");

//方式2：加载apk包中的html页面
webView.loadUrl("file:///android_asset/test.html");

//方式3：加载手机本地的html页面
webView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");

//步骤3. 复写shouldOverrideUrlLoading()方法，使得打开网页时不调用系统浏览器， 而是在本WebView中显示
webView.setWebViewClient(new WebViewClient(){
	@Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
    	view.loadUrl(url);
	return true;
    }
});
```
##### 3.3.2 onPageStarted()

* 作用：开始载入页面时候使用，我们可以设定一个loading页面，告诉用户正在等待网络响应
```java
 webView.setWebViewClient(new WebViewClient(){
      @Override
      public void  onPageStarted(WebView view, String url, Bitmap favicon) {
         //设定加载开始的操作
      }
  });
```
##### 3.3.3 onPageFinished()

* 作用：在页面结束是使用，我们可以关闭loading页面，切换程序动作
```java
webView.setWebViewClient(new WebViewClient(){
	@Override
    public void onPageFinished(WebView view, String url) {
    	//设定加载结束的操作
	}
});
```
##### 3.3.4 onLoadingResource()

* 作用：在加载页面资源的时候使用，每一个资源的加载都会调用一次
```java
webView.setWebViewClient(new WebViewClient(){
	@Override
    public boolean onLoadResource(WebView view, String url) {
    	//设定加载资源的操作
	}
});
```
##### 3.3.5 onReceiveError()

* 作用：加载页面服务器出现错误时使用(比如404)(如果我们显示网页自带的错误页面会很丑，所以我们会选择加载一个本地的html页面)
```java
//步骤1：写一个html文件（error_handle.html），用于出错时展示给用户看的提示页面
//步骤2：将该html文件放置到代码根目录的assets文件夹下

//步骤3：复写WebViewClient的onRecievedError方法
//该方法传回了错误码，根据错误类型可以进行不同的错误分类处理
webView.setWebViewClient(new WebViewClient(){
	@Override
    public void onReceivedError(WebView view, int errorCode, String description, String failingUrl){
		switch(errorCode){
            case HttpStatus.SC_NOT_FOUND:
            	view.loadUrl("file:///android_assets/error_handle.html");
                break;
		}
	}
});
```
##### 3.3.6 onReceiveSslError()

* 作用：处理http请求(webView默认是不处理https请求的，页面显示空白，需要进行如下设置： )
```java
webView.setWebViewClient(new WebViewClient() {    
	@Override    
    public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) 
    {
		handler.proceed();    //表示等待证书响应
        // handler.cancel();      //表示挂起连接，为默认方式
        // handler.handleMessage(null);    //可做其他处理
	}    
});  
```

#### 3.4 WebChromeClient类

* 作用：辅助WebView处理JavaScript的对话框，网站图标，网站标题等


##### 3.4.1 onProgressChanged()

* 获取网页加载进度并显示
```java
webview.setWebChromeClient(new WebChromeClient(){
      @Override
      public void onProgressChanged(WebView view, int newProgress) {
          if (newProgress < 100) {
              String progress = newProgress + "%";
              progress.setText(progress);
            } else {
        }
    });
}
```
##### 3.4.2 onReceiveTitle()

* 作用：获取web页中的标题
```java
webview.setWebChromeClient(new WebChromeClient(){
    @Override
    public void onReceivedTitle(WebView view, String title) {
       titleview.setText(title)；
    }
}
```
### 四、WebView与JS的交互

* 详情请点击[Webview与JS交互](高级UI/Webview/Webview与JS交互.md)

### 五、如何避免WebView在内存中的泄露

* 不在XML中创建WebView，而是在Activity中创建，context使用getApplicationContext()
```java
LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
        mWebView = new WebView(getApplicationContext());
        mWebView.setLayoutParams(params);
        mLayout.addView(mWebView）
```
* 在Activity销毁Webview的时候，先让webview加载null内容，然后在移除webview，在销毁webview，最后置空
```java
@Override
protected void onDestroy() {
	if (mWebView != null) {
    	mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
        mWebView.clearHistory();

        ((ViewGroup) mWebView.getParent()).removeView(mWebView);
        mWebView.destroy();
        mWebView = null;
	}
    super.onDestroy();
}
```
