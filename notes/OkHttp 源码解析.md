


## 说明
本文针对 Http 基础知识不会过多于描述，若对Http基础不够了解请先移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 。

## 一、OkHttp 的基本使用

```java 

//OkHttpClient为 网络请求器工厂接口Call.Factory的实现类
OkHttpClient client = new OkHttpClient().newBuilder().build();

Request request = new Request.Builder()
        .url("https://api.github.com/users")
        .header("Accept", "application/vnd.github.v3+json")
        .build();

client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        e.printStackTrace();
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.d("okhttp response", response.body().string());
    }
});
```

## 二、Request的成员变量和 build 模式实例化

Request类比较简单，主要是对构建 Http 请求报文的变量进行赋值。

```java
public final class Request {

  // 网络请求地址
  final HttpUrl url;
  // 网络请求方法
  final String method;
  // 请求头
  final Headers headers;
  // 请求体
  final @Nullable RequestBody body;
  // 标签
  final Object tag;
  //缓存控制器，以请求头的方式使用。
  private volatile CacheControl cacheControl; // Lazily initialized.

  public static class Builder {

    public Builder() {
    // 默认请求方法为GET
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
}
```



## OkHttpClient

### OkHttpClient的成员变量和 build 模式实例化
```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  final Dispatcher dispatcher;
  final @Nullable Proxy proxy;
  final List<Protocol> protocols;
  final List<ConnectionSpec> connectionSpecs;
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  final EventListener.Factory eventListenerFactory;
  final ProxySelector proxySelector;
  final CookieJar cookieJar;
  final @Nullable Cache cache;
  final @Nullable InternalCache internalCache;
  final SocketFactory socketFactory;
  final @Nullable SSLSocketFactory sslSocketFactory;
  final @Nullable CertificateChainCleaner certificateChainCleaner;
  final HostnameVerifier hostnameVerifier;
  final CertificatePinner certificatePinner;
  final Authenticator proxyAuthenticator;
  final Authenticator authenticator;
  final ConnectionPool connectionPool;
  final Dns dns;
  final boolean followSslRedirects;
  final boolean followRedirects;
  final boolean retryOnConnectionFailure;
  final int connectTimeout;
  final int readTimeout;
  final int writeTimeout;
  final int pingInterval;
```

### OkHttpClient对Call.Factory的实现

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory{

    //使用工厂模式构建Call的实现类RealCall
  @Override public Call newCall(Request request) {
    //三个参数分别为OkHttpClient本身，Request，是否使用WebSocket
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

}
```
## RealCall

我们先以RealCall中的execute()为例进行解读。

### execute

execute()为同步请求方式,即会在调用Call.execute()的当前线程直接进行网络请求。

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    // 网络请求过程监听，默认空实现。
    // 网络请求开始。
    eventListener.callStart(this);
    try {
      //dispatcher为网络请求调度器，维护着一个线程池和三个队列（异步请求等待队列、异步请求进行时队列、同步请求队列）
      
      client.dispatcher().executed(this);
      //进行网络请求并得到响应
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      // 网络请求失败。
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      // execute 请求完成
      client.dispatcher().finished(this);
    }
  }
```
下面将先对Dispatcher类的构造、变量以及涉及到同步请求的方法进行解读。

### Dispatcher

```java
public final class Dispatcher {
  // 所允许的同时进行的网络请求的最大数量
  private int maxRequests = 64;
  // 所允许的同时进行网络请求的最多域名数
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      // 初始化线程池，该默认线程数为0，所能最多线程数为Integer.MAX_VALUE，当空闲线程超过60秒没有新任务时销毁回收。
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

### enqueue()

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

```java
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```


### Dispatcher