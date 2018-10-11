


## WebView

WebView 是一个用来显示 Web 网页的控件，包含一个浏览器该有的基本功能，例如：滚动、缩放、前进、后退下一页、搜索、执行 Js等功能。

在 Android 4.4 之前使用 WebKit 作为渲染内核，4.4 之后采用 chrome 内核。

### WebView API

#### 加载

```java

// 加载网页 url，也可以执行js函数。
webView.loadUrl("http://www.jianshu.com/u/fa272f63280a");
// 加载apk包中的html页面。
webView.loadUrl("file:///android_asset/test.html");
// 加载手机本地的html页面。
webView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");
// 停止当前正在加载的页面。
webView.stopLoading();

// 重新加载当前请求。
webView.reload():
```

#### 状态

```java
// 激活WebView为活跃状态，能正常执行网页的响应。
webView.onResume();
// 当页面被失去焦点被切换到后台不可见状态调用（不会暂停JavaScript）。
webView.onPause();
// 该方法面向全局整个应用程序的webview，它会暂停所有webview的layout，parsing，JavaScript Timer。当程序进入后台时，该方法的调用可以降低CPU功耗。
webView.pauseTimers();
// 恢复pauseTimers状态。
webView.resumeTimers();
// 在关闭了Activity时，如果 Webview 的音乐或视频，还在播放，就必须销毁先 Webview。
rootLayout.removeView(webView); 
webView.destroy();
```

### 操作

```java
// 是否可以后退。
webView.canGoBack();
// 是否可以前进                     
webView.canGoForward();
// 回退到上一页。
webView.goBack();
// 前进到下一页。
webView.goForward()
// 以当前的index为起始点前进或者后退到历史记录中指定的steps，
// 如果steps为负数则为后退，正数则为前进。
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
// 清空网页访问留下的缓存数据。需要注意的时，由于缓存是全局的，所以只要是WebView用到的缓存都会被清空，即便其他地方也会使用到。若设为false，则只清空内存里的资源缓存，而不清空磁盘里的。
Webview.clearCache(true);
// 清除当前 WebView 对象访问的历史记录。
webView.clearHistory():
// 清除ssl信息。
webView.clearSslPreferences():
// 清除网页查找的高亮匹配字符。
webView.clearMatches():
```

### WebSettings

```java
websettings.setJavaScriptEnabled()
```
