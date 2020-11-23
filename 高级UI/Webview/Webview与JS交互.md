[TOC]

## Webview与JS交互

#### 转载

* [Android：你要的WebView与 JS 交互方式 都在这里了](http://blog.csdn.net/carson_ho/article/details/64904691)

### 一、交互方式总结

#### 1.1 两种方式

* Android与JS通过WebView互相调用方法，实际上是：
  * Android去调用JS的代码
  * JS去调用Android的代码

* 二者沟通的桥梁是WebView

#### 1.2 Android调用JS代码

* 通过`WebView`的`loadUrl（）`

* 通过`WebView`的`evaluateJavascript（）`

#### 1.3 JS调用Android代码

* 通过`WebView`的`addJavascriptInterface（）`进行对象映射

* 通过 `WebViewClient` 的`shouldOverrideUrlLoading ()`方法回调拦截 url

* 通过 `WebChromeClient` 的`onJsAlert()`、`onJsConfirm()`、`onJsPrompt（）`方法回调拦截JS对话框`alert()`、`confirm()`、`prompt（）` 消息

###  二、Android通过WebView调用 JS 代码

#### 2.1 通过`WebView`的`loadUrl()`

- 实例介绍：点击Android按钮，即调用WebView JS（文本名为`javascript`）中callJS（）
- 具体使用：

##### 2.1.1 步骤1：将需要调用的JS代码以`.html`格式放到src/main/assets文件夹里

* 为了方便展示，本文是采用Andorid调用本地JS代码说明；

* 实际情况时，Android更多的是调用远程JS代码，即将加载的JS代码路径改成url即可

* 需要加载JS代码：index.html

```javascript
<!--文本名：javascript-->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Carson_Ho</title>

    <!--JS代码-->
    <script>
        <!--Android需要调用的方法-->
        function callJS(){
            alert("Android调用了JS的callJS方法");
        }
    </script>
</head>
</html>
```

##### 2.1.2 步骤2：在Android里通过WebView设置调用JS代码

* activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="10dp"
    tools:context=".MainActivity">

    <WebView
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="400dp" />

    <Button
        android:id="@+id/button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="点击调用JS代码" />
</LinearLayout>
```

* Android代码：MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    WebView mWebView;
    Button button;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        button = (Button) findViewById(R.id.button);
        mWebView =(WebView) findViewById(R.id.webview);
        WebSettings webSettings = mWebView.getSettings();
        // 设置与Js交互的权限
        webSettings.setJavaScriptEnabled(true);
        // 设置允许JS弹窗
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);
        // 先载入JS代码
        // 格式规定为:file:///android_asset/文件名.html
        mWebView.loadUrl("file:///android_asset/index.html");

        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 通过Handler发送消息
                mWebView.post(new Runnable() {
                    @Override
                    public void run() {
                        // 注意调用的JS方法名要对应上
                        // 调用javascript的callJS()方法
                        mWebView.loadUrl("javascript:callJS()");
                    }
                });
            }
        });

        // 由于设置了弹窗检验调用结果,所以需要支持js对话框
        // webview只是载体，内容的渲染需要使用webviewChromClient类去实现
        // 通过设置WebChromeClient对象处理JavaScript的对话框
        //设置响应js 的Alert()函数
        mWebView.setWebChromeClient(new WebChromeClient() {
            @Override
            public boolean onJsAlert(WebView view, String url, String message, final JsResult result) {
                AlertDialog.Builder b = new AlertDialog.Builder(MainActivity.this);
                b.setTitle("Alert");
                b.setMessage(message);
                b.setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        result.confirm();
                    }
                });
                b.setCancelable(false);
                b.create().show();
                return true;
            }
        });
    }
}
```

![效果图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS04MjZkMGFhMDY1ZjcwY2IxLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)

* **特别注意：JS代码调用一定要在 `onPageFinished（）` 回调之后才能调用，否则不会调用。**

* `onPageFinished()`属于WebViewClient类的方法，主要在页面加载结束时调用

#### 2.2 通过`WebView`的`evaluateJavascript()`

##### 2.2.1 优点

- 该方法比第一种方法效率更高、使用更简洁。
  * 因为该方法的执行不会使页面刷新，而第一种方法（loadUrl ）的执行则会。
  * Android 4.4 后才可使用

##### 2.2.2 使用

* 只需要将第一种方法的loadUrl()换成下面该方法即可，点击事件代码修改如下

```java
button.setOnClickListener(new View.OnClickListener() {
	@Override
    public void onClick(View v) {
    	// 通过Handler发送消息
        mWebView.post(new Runnable() {
        	@Override
            public void run() {
            	// 注意调用的JS方法名要对应上
                // 调用javascript的callJS()方法
                mWebView.evaluateJavascript("javascript:callJS()", new ValueCallback<String>() {
                	@Override
                    public void onReceiveValue(String value) {
                    	//此处为 js 返回的结果
					}
				});
			}
		});
	}
});
```

#### 2.3 方法对比

![方式对比图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0zMGYwOTVkNGM5ZTYzOGZkLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)

#### 2.4 使用建议

* 两种方法混合使用，即Android 4.4以下使用方法1，Android 4.4以上方法2

```java
// Android版本变量
final int version = Build.VERSION.SDK_INT;
// 因为该方法在 Android 4.4 版本才可使用，所以使用时需进行版本判断
if (version < 18) {
    mWebView.loadUrl("javascript:callJS()");
} else {
    mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            //此处为 js 返回的结果
        }
    });
}
```

### 三、JS通过WebView调用 Android 代码

* 对于JS调用Android代码的方法有3种：

1. 通过`WebView`的`addJavascriptInterface（）`进行对象映射
2. 通过 `WebViewClient` 的`shouldOverrideUrlLoading ()`方法回调拦截 url
3. 通过 `WebChromeClient` 的`onJsAlert()`、`onJsConfirm()`、`onJsPrompt（）`方法回调拦截JS对话框`alert()`、`confirm()`、`prompt（）` 消息

#### 3.1 通过 `WebView`的`addJavascriptInterface()`进行对象映射

##### 3.1.1 步骤1：定义一个与JS对象映射关系的Android类：AndroidToJs

* AndroidToJs.java

```java
public class AndroidToJs {
    // 定义JS需要调用的方法
    // 被JS调用的方法必须加入@JavascriptInterface注解
    @JavascriptInterface
    public void hello(String msg) {
        Log.e("AndroidToJs", "JS调用了Android的hello方法");
    }
}
```

##### 3.1.2 步骤2：将需要调用的JS代码以`.html`格式放到src/main/assets文件夹里

* 需要加载JS代码：index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Carson</title>
    <script>
         function callAndroid(){
            <!--由于对象映射，所以调用test对象等于调用Android映射的对象即AndroidToJs实例-->
            test.hello("js调用了android中的hello方法");
         }
    </script>
</head>
<body>
    <!--点击按钮则调用callAndroid函数-->
    <button type="button" id="button1" onclick="callAndroid()">点击调用Android代码</button>
</body>
</html>
```

##### 3.1.3 步骤3：在Android里通过WebView设置Android类与JS代码的映射

```java
public class MainActivity extends AppCompatActivity {
    WebView mWebView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mWebView = (WebView) findViewById(R.id.webview);
        WebSettings webSettings = mWebView.getSettings();

        // 设置与Js交互的权限
        webSettings.setJavaScriptEnabled(true);

        // 通过addJavascriptInterface()将Java对象映射到JS对象
        //参数1：Javascript对象名
        //参数2：Java对象名
        mWebView.addJavascriptInterface(new AndroidToJs(), "test");//AndroidToJS类对象映射到js的test对象

        // 加载JS代码
        // 格式规定为:file:///android_asset/文件名.html
        mWebView.loadUrl("file:///android_asset/index.html");
    }
}
```

<img src="..\..\images\高级UI\Webview\addJavascriptInterface实例显示.PNG" alt="addJavascriptInterface实例显示" style="zoom: 50%;" />

* 点击输出结果

```java
com.example.study E/AndroidToJs: JS调用了Android的hello方法
```

##### 3.1.4 特点

- 优点：使用简单
  * 仅将Android对象和JS对象映射即可
- 缺点：存在严重的漏洞问题，具体请看文章：[你不知道的 Android WebView 使用漏洞](http://www.jianshu.com/p/3a345d27cd42)

#### 3.2 通过 `WebViewClient` 的方法`shouldOverrideUrlLoading()`回调拦截 url

##### 3.2.1 具体原理

* Android通过 `WebViewClient` 的回调方法`shouldOverrideUrlLoading()`拦截 url

* 解析该 url 的协议

* 如果检测到是预先约定好的协议，就调用相应方法
  * 即JS需要调用Android的方法

##### 3.2.2 步骤：在JS约定所需要的Url协议

* 
  JS代码：index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Carson_Ho</title>

    <script>
         function callAndroid(){
            <!--约定的url协议为：js://webview?arg1=111&arg2=222*/ -->
            document.location = "js://webview?arg1=111&arg2=222";
         }
    </script>
</head>

<!-- 点击按钮则调用callAndroid（）方法  -->
<body>
<button type="button" id="button1" onclick="callAndroid()">点击调用Android代码</button>
</body>
</html>
```

* 当该JS通过Android的`mWebView.loadUrl("file:///android_asset/javascript.html")`加载后，就会回调`shouldOverrideUrlLoading()`，接下来继续看步骤2：

##### 3.2.3 步骤2：在Android通过WebViewClient复写shouldOverrideUrlLoading()

* MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    WebView mWebView;

    WebViewClient webViewClient = new WebViewClient() {
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            // 步骤2：根据协议的参数，判断是否是所需要的url
            // 一般根据scheme（协议格式） & authority（协议名）判断（前两个参数）
            //假定传入进来的 url = "js://webview?arg1=111&arg2=222"（同时也是约定好的需要拦截的）
            Uri uri = Uri.parse(url);
            // 如果url的协议 = 预先约定的 js 协议
            // 就解析往下解析参数
            if (uri.getScheme().equals("js")) {
                // 如果 authority  = 预先约定协议里的 webview，即代表都符合约定的协议
                // 所以拦截url,下面JS开始调用Android需要的方法
                if (uri.getAuthority().equals("webview")) {
                    //  步骤3：
                    // 执行JS所需要调用的逻辑
                    Log.e("MainActivity", "js调用了Android的方法");
                    // 可以在协议上带有参数并传递到Android上
                    HashMap<String, String> params = new HashMap<>();
                    Set<String> collection = uri.getQueryParameterNames();
                }
                return true;
            }
            return super.shouldOverrideUrlLoading(view, url);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mWebView = (WebView) findViewById(R.id.webview);
        WebSettings webSettings = mWebView.getSettings();
        // 设置与Js交互的权限
        webSettings.setJavaScriptEnabled(true);
        // 设置允许JS弹窗
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        // 步骤1：加载JS代码
        // 格式规定为:file:///android_asset/文件名.html
        mWebView.loadUrl("file:///android_asset/index.html");

        // 复写WebViewClient类的shouldOverrideUrlLoading方法
        mWebView.setWebViewClient(webViewClient);
    }
}
```

<img src="..\..\images\高级UI\Webview\shouldOverrideUrlLoading实例显示.PNG" alt="shouldOverrideUrlLoading实例显示" style="zoom: 50%;" />

##### 3.2.4 特点

- 优点：不存在方式1的漏洞；
- 缺点：JS获取Android方法的返回值复杂。
  * 如果JS想要得到Android方法的返回值，只能通过 WebView 的 `loadUrl()`去执行 JS 方法把返回值传递回去，相关的代码如下：

```java
// Android：MainActivity.java
mWebView.loadUrl("javascript:returnResult(" + result + ")");

// JS：javascript.html
function returnResult(result){
    alert("result is" + result);
}
```

#### 3.3 通过 `WebChromeClient` 的`onJsAlert()`、`onJsConfirm()`、`onJsPrompt()`方法回调拦截JS对话框`alert()`、`confirm()`、`prompt()` 消息

##### 3.3.1 常用方法

* 在JS中，有三个常用的对话框方法：

![Paste_Image.png](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0xMzg1Zjc0ODYxOGFmODg2LnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)

##### 3.3.2 原理

* Android通过 `WebChromeClient` 的`onJsAlert()`、`onJsConfirm()`、`onJsPrompt（）`方法回调分别拦截JS对话框（即上述三个方法），得到他们的消息内容，然后解析即可。

##### 3.3.3 说明

* 下面的例子将用**拦截 JS的输入框（即`prompt（）`方法）**说明 ：
  * 常用的拦截是：拦截 JS的输入框（即`prompt（）`方法）
  * 因为只有`prompt（）`可以返回任意类型的值，操作最全面方便、更加灵活；而alert（）对话框没有返回值；confirm（）对话框只能返回两种状态（确定 / 取消）两个值

##### 3.3.4 步骤1：加载JS代码，如下：

* index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Carson_Ho</title>

    <script>
        function clickprompt(){
            <!--  调用prompt（） -->
            var result=prompt("js://webview?arg1=111&arg2=222");
            alert("demo " + result);
        }
    </script>
</head>

<body>
<!-- 点击按钮则调用clickprompt()  -->
<button type="button" id="button1" onclick="clickprompt()">点击调用Android代码</button>
</body>
</html>
```

* 如果是拦截警告框（即`alert()`），则触发回调`onJsAlert（）`；

* 如果是拦截确认框（即`confirm()`），则触发回调`onJsConfirm（）`；

##### 3.3.5 步骤2：在Android通过`WebChromeClient`复写`onJsPrompt()`

```java
public class MainActivity extends AppCompatActivity {

    WebView mWebView;
    //    Button button;
    WebChromeClient webChromeClient = new WebChromeClient() {
        // 拦截输入框(原理同方式2)
        // 参数message:代表promt（）的内容（不是url）
        // 参数result:代表输入框的返回值
        @Override
        public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
            // 根据协议的参数，判断是否是所需要的url(原理同方式2)
            // 一般根据scheme（协议格式） & authority（协议名）判断（前两个参数）
            //假定传入进来的 url = "js://webview?arg1=111&arg2=222"（同时也是约定好的需要拦截的）

            Uri uri = Uri.parse(message);
            // 如果url的协议 = 预先约定的 js 协议
            // 就解析往下解析参数
            if (uri.getScheme().equals("js")) {
                // 如果 authority  = 预先约定协议里的 webview，即代表都符合约定的协议
                // 所以拦截url,下面JS开始调用Android需要的方法
                if (uri.getAuthority().equals("webview")) {
                    // 执行JS所需要调用的逻辑
                    System.out.println("js调用了Android的方法");
                    // 可以在协议上带有参数并传递到Android上
                    HashMap<String, String> params = new HashMap<>();
                    Set<String> collection = uri.getQueryParameterNames();
                    //参数result:代表消息框的返回值(输入值)
                    result.confirm("js调用了Android的方法成功啦");
                }
                return true;
            }
            return super.onJsPrompt(view, url, message, defaultValue, result);
        }

        // 拦截JS的警告框
        @Override
        public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
            return super.onJsAlert(view, url, message, result);
        }

        // 拦截JS的确认框
        @Override
        public boolean onJsConfirm(WebView view, String url, String message, JsResult result) {
            return super.onJsConfirm(view, url, message, result);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mWebView = (WebView) findViewById(R.id.webview);
        WebSettings webSettings = mWebView.getSettings();
        // 设置与Js交互的权限
        webSettings.setJavaScriptEnabled(true);
        // 设置允许JS弹窗
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        // 格式规定为:file:///android_asset/文件名.html
        mWebView.loadUrl("file:///android_asset/index.html");

        mWebView.setWebChromeClient(webChromeClient);
    }
}
```

![效果图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS0wMThkZTNmMmJhMTgyOGMwLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)

#####  3.3.6 三种方式的对比 & 使用场景

![示意图](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS04YzkxNDgxMzI1YTUyNTNlLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA)

### 四、总结

- 本文主要对**Android通过WebView与JS的交互方式进行了全面介绍**

![示意图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NDQzNjUtNjEzYjU3YzkzZGZmMmViOC5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcCU3Q2ltYWdlVmlldzIvMi93LzEyNDA)