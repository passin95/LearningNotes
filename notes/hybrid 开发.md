
- [WebView](#webview)
    - [WebView API](#webview-api)
        - [加载](#加载)
        - [状态](#状态)
        - [操作](#操作)
        - [清理](#清理)
    - [WebSettings](#websettings)
        - [缓存设置](#缓存设置)
    - [WebViewClient](#webviewclient)
    - [WebChromeClient](#webchromeclient)
- [Android 和 Js 的交互](#android-和-js-的交互)
    - [Android 调用 Js](#android-调用-js)
        - [WebView.loadUrl()](#webviewloadurl)
        - [WebView.evaluateJavascript()](#webviewevaluatejavascript)
    - [Js 调用 Android](#js-调用-android)
        - [WebView.addJavascriptInterface() 及漏洞处理](#webviewaddjavascriptinterface-及漏洞处理)
        - [WebViewClient 的 shouldOverrideUrlLoading()](#webviewclient-的-shouldoverrideurlloading)
        - [WebChromeClient 的 onJsAlert()、onJsConfirm()、onJsPrompt()](#webchromeclient-的-onjsalertonjsconfirmonjsprompt)
    - [VasSonic](#vassonic)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

## WebView

WebView 是一个用来显示 Web 网页的控件，包含一个浏览器该有的基本功能，例如：滚动、缩放、前进、后退下一页、搜索、执行 Js 等功能。

在 Android 4.4 之前使用 WebKit 作为渲染内核，4.4 之后采用 chrome 内核。


### WebView API

#### 加载

```java
// 加载网页 url，也可以执行 js 函数。
webView.loadUrl("http://www.jianshu.com/u/fa272f63280a");
// 加载 apk 包中的 html 页面。
webView.loadUrl("file:///android_asset/test.html");
// 加载手机本地的 html 页面。
webView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");
// 停止当前正在加载的页面。
webView.stopLoading();
// 重新加载当前请求。
webView.reload():
```

#### 状态

```java
// 激活 WebView 为活跃状态，能正常执行网页的响应。
webView.onResume();
// 当页面被失去焦点被切换到后台不可见状态调用（不会暂停 JavaScript）。
webView.onPause();
// 该方法面向全局整个应用程序的 webview，它会暂停所有 webview 的 layout，parsing，JavaScript Timer。当程序进入后台时，该方法的调用可以降低 CPU 功耗。
webView.pauseTimers();
// 恢复 pauseTimers 状态。
webView.resumeTimers();
// 在关闭了 Activity 时，如果 Webview 的音乐或视频，还在播放，就必须销毁先 Webview，防止内存泄漏。
((ViewGroup) webView.getParent()).removeView(webView);
// rootLayout.removeView(webView); 
webView.destroy();
```

#### 操作

```java
// 是否可以后退。
webView.canGoBack();
// 是否可以前进                     
webView.canGoForward();
// 回退到上一页。
webView.goBack();
// 前进到下一页。
webView.goForward()
// 以当前的 index 为起始点前进或者后退到历史记录中指定的 steps，
// 如果 steps 为负数则为后退，正数则为前进。
webView.goBackOrForward(intsteps) 
```

后退处理。

```java
public boolean onKeyDown(int keyCode, KeyEvent event) {
    if ((keyCode == KEYCODE_BACK) && webView.canGoBack()) {
        webView.goBack();
        return true;
    }
    return super.onKeyDown(keyCode, event);
}
```

#### 清理

```java
// 清空网页访问留下的缓存数据。需要注意的时，由于缓存是全局的，所以只要是 WebView 用到的缓存都会被清空，即便其他地方也会使用到。若设为 false，则只清空内存里的资源缓存，而不清空磁盘里的。
webview.clearCache(true);
// 清除当前 WebView 对象访问的历史记录。
webView.clearHistory();
// 清除 ssl 信息。
webView.clearSslPreferences();
// 清除网页查找的高亮匹配字符。
webView.clearMatches();
```

### WebSettings

```java
// 支持 Javascript。
// 若加载的 html 里有 JS 在执行动画等操作，会造成资源浪费（CPU、电量）,
// 可在 onStart() 和 onStop() 设置成 true 和 false。
websettings.setJavaScriptEnabled();
// 支持插件。
webSettings.setPluginsEnabled(true); 
// 可以访问文件。
webSettings.setAllowFileAccess(true); 
// 支持通过 JS 打开新窗口。
webSettings.setJavaScriptCanOpenWindowsAutomatically(true)。
// 支持自动加载图片。
webSettings.setLoadsImagesAutomatically(true);
// 设置默认编码格式。
webSettings.setDefaultTextEncodingName("utf-8");
// 关闭密码保存提醒 (默认为 true，最好设为 false)。
webSettings.setSavePassword(false);
// 设置允许 JS 弹窗
settings.setJavaScriptCanOpenWindowsAutomatically(true);



// 将图片调整到适合 webview 的大小。
websettings.setUseWideViewPort(true);
// 将页面缩放至屏幕的大小
webSettings.setLoadWithOverviewMode(true);
// 设置支持缩放，默认为 true，是 setBuiltInZoomControls() 的前提。
webSettings.setSupportZoom(true); 
// 设置内置的缩放控件可进行缩放。
webSettings.setBuiltInZoomControls(true);
// 隐藏缩放控件的按钮。
webSettings.setDisplayZoomControls(false);
```

#### 缓存设置

```java
// 设置缓存方式。
// LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据。
// LOAD_DEFAULT: 根据 cache-control 决定是否从网络上取数据。
// LOAD_NO_CACHE: 不使用缓存，只从网络获取数据。
// LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者 no-cache，都使用缓存中的数据。
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);


if (NetStatusUtil.isConnected(getApplicationContext())) {
    // 根据 cache-control 决定是否从网络上取数据。
    webSettings.setCacheMode(WebSettings.LOAD_DEFAULT);
} else {
    //没网络，则优先加载缓存。
    webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
}

// 开启 DOM storage API 功能
webSettings.setDomStorageEnabled(true); 
// 开启 database storage API 功能
webSettings.setDatabaseEnabled(true);
// 开启 Application Caches 功能
webSettings.setAppCacheEnabled(true);
// 设置 Android 私有缓存存储。
webSettings.setAppCachePath(WebCacheDirPath);
```

### WebViewClient

```java
WebViewClient webViewClient = new WebViewClient(){

        @Override
        public void onPageStarted(WebView view, String url, Bitmap favicon) {
            // 开始加载页面触发。
        }

        @Override
        public void onPageFinished(WebView view, String url) {
            // 结束加载页面触发。
        }

        @Override
        public void onLoadResource(WebView view, String url) {
            // 在加载页面资源时会触发，每一个资源（比如图片）的加载都会触发一次。
        }

        @RequiresApi(api = VERSION_CODES.M)
        @Override
        public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
            // 加载页面出现错误码时触发（4XX、5XX）。
            // 可以根据错误展示不同的错误页面（以 Android 端作为处理）
        }

        @Override
        public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) {
            // https 连接过程出错时触发，例如证书验证失败。
            // 处理方式：
            // handler.cancel() 取消加载。
            // handler.proceed() 继续加载，忽略 SSL 错误。
            handler.proceed();
            Log.d("test", error.getPrimaryError()+" "+error.getUrl());
        }

        @Override
        public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
            // 返回 true 表示自己处理此次请求。返回 false 表示由 webview 自行处理。
            return false;
        }

        @Override
        public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
            // 对网络请求进行拦截处理。
            return super.shouldInterceptRequest(view, request);
        }
    }
};
webView.setWebViewClient(webViewClient);
```

### WebChromeClient

用于辅助 WebView 处理 Javascript 的对话框,网站图标,网站标题等。

```java
WebChromeClient webChromeClient = new WebChromeClient() {

    /**
     * 加载网页进度变化。
     *
     * @param newProgress 进度值（0-100）
     */
    @Override
    public void onProgressChanged(WebView view, int newProgress) {
        Log.d("log", newProgress + "");
        super.onProgressChanged(view, newProgress);
    }

    /**
     * 获取网页标题，例如 www.baidu.com 的标题为“百度一下，你就知道”。
     *
     * @param title 标题名
     */
    public void onReceivedTitle(WebView view, String title) {
    }

    /**
     * 接受网页的 Icon。
     */
    public void onReceivedIcon(WebView view, Bitmap icon) {
    }


    /**
     * 在 Js 中调用 alert() 函数触发。
     *
     * @param url 请求对话的页面的 URL
     * @param message 内容
     * @return 客户端是否处理了 Alert。
     */
    @Override
    public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
        new AlertDialog.Builder(MainActivity.this)
                .setTitle("JsAlert")
                .setMessage(message)
                .setPositiveButton("OK", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        result.confirm();
                    }
                })
                .setCancelable(false)
                .show();
        return true;
    }

    /**
     *  支持 Js 确认框。
     */
    @Override
    public boolean onJsConfirm(WebView view, String url, String message, JsResult result) {
        new AlertDialog.Builder(MainActivity.this)
                .setTitle("JsConfirm")
                .setMessage(message)
                .setPositiveButton("OK", (dialog, which) -> result.confirm())
                .setNegativeButton("Cancel", (dialog, which) -> result.cancel())
                .setCancelable(false)
                .show();
        // 已处理
        return true;
    }

    /**
     * 支持 Js 输入框。
     */
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue,
            JsPromptResult result) {
        final EditText et = new EditText(MainActivity.this);
        et.setText(defaultValue);
        new AlertDialog.Builder(MainActivity.this)
                .setTitle(message).setView(et)
                .setPositiveButton("OK", (dialog, which) -> result.confirm(et.getText().toString()))
                .setNegativeButton("Cancel", (dialog, which) -> result.cancel())
                .setCancelable(false)
                .show();

        return true;
    }
};
webView.setWebChromeClient(webChromeClient);
```

## Android 和 Js 的交互

### Android 调用 Js

#### WebView.loadUrl()

将需要调用的 JS 代码以.html 格式放到 src/main/assets 文件夹里（更多的是调用远程 JS 代码）。

```java
// 设置能够与 Js 交互。
webSettings.setJavaScriptEnabled(true);
// 设置允许 Js 弹窗。
webSettings.setJavaScriptCanOpenWindowsAutomatically(true);
// 加载本地 html。
webView.loadUrl("file:///android_asset/javascript.html");
// 调用 Js 中的 callJS() 方法，调用 Js 方法必须在 onPageFinished() 回调之后生效。
webView.loadUrl("javascript:callJS()");
```

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Passin</title>
  <script>
    function callJS(){
      alert("Android 调用了 JS 的 callJS 方法");
   }
  </script>
</head>
</html> 
```

#### WebView.evaluateJavascript()

该方法比第一种方法效率更高、使用更简洁，因为因为该方法的执行不会使页面刷新。但需要 Android4.4 版本以上。

```java
if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
    webView.evaluateJavascript("javascript:callJS()", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            // value 为 Js 的返回值。
        }
    });
} else {
    webView.loadUrl("javascript:callJS()");
}
```

### Js 调用 Android

#### WebView.addJavascriptInterface() 及漏洞处理

```java
// Android 类对象映射到 Js 的 test 对象
webView.addJavascriptInterface(new AndroidtoJs(), "test");

public class AndroidtoJs extends Object {

    // 定义 JS 需要调用的方法
    // 被 JS 调用的方法必须加入@JavascriptInterface 注解 
    @JavascriptInterface
    public void hello(String msg) {
        System.out.println("Js 调用了 Android 的 hello 方法");
    }
}
```

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Passin</title>
  <script>
     function callAndroid(){
     // 由于对象映射，所以调用 test 对象等于调用 Android 映射的对象。
     test.hello("js 调用了 android 中的 hello 方法");
    }
  </script>
</head>
<body>
<button type="button" id="button1" onclick="callAndroid()"> 调用 android hello() 方法 </button>
</body>
</html>
```

该方式使用简单，仅将 Android 对象和 JS 对象映射即可，但在 Android 4.2 及以下存在漏洞。
漏洞产生原因：当 Js 拿到 Android 这个对象后，就可以调用这个 Android 对象中所有的方法，包括系统类（java.lang.Runtime 类），从而进行任意代码执行。
推荐使用 [SafeWebView](https://github.com/seven456/SafeWebView/tree/0f11155d38b765c3d832c4dab733f771610c73a3) 解决了 Android WebView 中 Js 注入漏洞问题，另外还包含了一些异常处理。除此之外还需注意（Android 4.0 以上）：

- 设置 webView.setSavePassword(false)，否则密码会被明文保到 /data/data/com.package.name/databases/webview.db 中。

- 如果不使用 file 协议（加载本地 html）禁用 file 协议。

```java
webSettings.setAllowFileAccess(false); 
webSettings.setAllowFileAccessFromFileURLs(false);
webSettings.setAllowUniversalAccessFromFileURLs(false);
```

- 对于需要使用 file 协议的应用，禁止 file 协议加载 JavaScript。
  
```java
if (url.startsWith("file://") {
    webSettings.setJavaScriptEnabled(false);
} else {
    webSettings.setJavaScriptEnabled(true);
}
```

#### WebViewClient 的 shouldOverrideUrlLoading()

Android通过 WebViewClient 的回调方法shouldOverrideUrlLoading ()拦截 url，再解析该 url 的协议，如果检测到是预先约定好的协议，就调用相应方法。

```java
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    Uri uri = Uri.parse(url);
    if (uri.getScheme().equals("js")) {
        if (uri.getAuthority().equals("webview")) {
            System.out.println("js调用了Android的方法");
            Set<String> collection = uri.getQueryParameterNames();
            for (String s : collection) {
                System.out.println(s);
                System.out.println(uri.getQueryParameter(s));
            }
        }
        return true;
    }
    return super.shouldOverrideUrlLoading(view, url);
}
```

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Passin</title>
  <script>
     function callAndroid(){
     document.location = "js://webview?username=passin&pwd=123"
      }
  </script>
</head>
<body>
<button type="button" id="button1" onclick="callAndroid()">调用 android 方法</button>
</body>
</html>
```

#### WebChromeClient 的 onJsAlert()、onJsConfirm()、onJsPrompt()

通过 WebChromeClient 的 onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调分别拦截 Js 对话框。该方式不存在漏洞问题。

```java
@Override
public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
    new AlertDialog.Builder(MainActivity.this)
            .setTitle("JsAlert")
            .setMessage(message)
            .setPositiveButton("OK", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    result.confirm();
                }
            })
            .show();
    return true;
}

@Override
public boolean onJsConfirm(WebView view, String url, String message, JsResult result) {
    new AlertDialog.Builder(MainActivity.this)
            .setTitle("JsConfirm")
            .setMessage(message)
            .setPositiveButton("OK", (dialog, which) -> result.confirm())
            .setNegativeButton("Cancel", (dialog, which) -> result.cancel())
            .show();
    // 已处理
    return true;
}

@Override
public boolean onJsPrompt(WebView view, String url, String message, String defaultValue,
        JsPromptResult result) {
    final EditText et = new EditText(MainActivity.this);
    new AlertDialog.Builder(MainActivity.this)
            .setTitle(message)
            .setView(et)
            .setPositiveButton("OK", (dialog, which) -> result.confirm(et.getText().toString()))
            .setNegativeButton("Cancel", (dialog, which) -> result.cancel())
            .show();
    return true;
```

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Passin</title>
  <script>
     function callAndroid(){
     var result = prompt("标题")
     alert(result);
    }
  </script>
</head>
<body>
<button type="button" id="button1" onclick="callAndroid()"> 调用 android 的方法 </button>
</body>
</html>
```

### VasSonic

[VasSonic](https://github.com/Tencent/VasSonic) 是腾讯开源的一个轻量级的高性能的Hybrid框架，专注于提升页面首屏加载速度，完美支持静态直出页面和动态直出页面，兼容离线包等方案。

具体使用参考 [WiKi](https://github.com/Tencent/VasSonic/blob/master/sonic-android/README.md)。

# 参考资料

- [最全面总结 Android WebView 与 JS 的交互方式](https://www.jianshu.com/p/345f4d8a5cfa)
- [Android WebView 全面干货指南](https://www.jianshu.com/p/fd61e8f4049e)