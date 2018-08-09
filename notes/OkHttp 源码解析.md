


## 说明

本文对 Http 基础知识不会过多描述，若对 Http 基础不够了解请先移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 。
本文源码为 OkHttp 3.10.0 版本。此外该版本 OkHttp 底层已不再使用HttpURLConnection，而是自己重写了 tcp/ip 层的实现。

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

### OkHttpClient的成员变量

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  // 网络请求调度器
  final Dispatcher dispatcher;
  // 代理方式，分包有 DIRECT (直连)、HTTP 、SOCKS 三种。
  final @Nullable Proxy proxy;
  // 具体使用的哪一版本的应用层协议，例如 http1.0、http1.1，http2.0。
  final List<Protocol> protocols;
  // Http和Https的 TLS 版本和 密码套件的选择，okhttp默认优先使用 MODERN_TLS。
  // MODERN_TLS是连接到最新的HTTPS服务器的安全配置。
  // COMPATIBEL_TLS是连接到过时的HTTPS服务器的安全配置。
  // CLEARTEXT是用于http://开头的URL的非安全配置。
  final List<ConnectionSpec> connectionSpecs;
  // 
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  // 网络请求时间监听工厂
  final EventListener.Factory eventListenerFactory;
  // 针对存在使用多个代理时，选择下一次网络请求使用的代理方式。
  final ProxySelector proxySelector;
  // CookieJar是一个接口方法，用于使用Cookie时存放List<Cookie>和根据HttpUrl去找到相应Cookie。
  final CookieJar cookieJar;
  // OkHttp 提供的缓存类。
  final @Nullable Cache cache;
  // CacheInterceptor为缓存操作应该具有的方法接口，构造函数的参数为InternalCache，即面向接口编程。
  // 提供拓展给开发者实现自己的缓存算法。
  final @Nullable InternalCache internalCache;
  // Socket工厂，网络层建立连接使用。
  final SocketFactory socketFactory;
  // 用于 https 连接。
  final @Nullable SSLSocketFactory sslSocketFactory;
  // 对发送过来的证书进行层级梳理。
  final @Nullable CertificateChainCleaner certificateChainCleaner;
  // 用于验证对方发过来的证书中的 Hostname 和用户请求服务器的 Hostname 是否一致。
  final HostnameVerifier hostnameVerifier;
  // Android 证书锁，即不全信任客户端证书库，还对服务器证书传过来的证书指纹和客户端应用中已写入的指纹进行匹配。
  final CertificatePinner certificatePinner;
  // 当返回错误码 407 时，自动进行认证。区别在于第一个是向代理服务器进行认证，第二个是向目标服务器进行认证。
  final Authenticator proxyAuthenticator;
  final Authenticator authenticator;
  // 连接池
  final ConnectionPool connectionPool;
  final Dns dns;
  // 是否在https收到http请求时（反之亦然），是否自动请求重定向指向的网站。
  final boolean followSslRedirects;
  // 是否自动请求重定向指向的网站。
  final boolean followRedirects;
  // 请求失败后是否需要重试。
  final boolean retryOnConnectionFailure;
  // 连接超时时间。
  final int connectTimeout;
  // 发出request报文结束至收到的response报文所允许的最大时间。
  final int readTimeout;
  // 发送request报文开始至发送结束所允许的最大时间。
  final int writeTimeout;
  // webSocket使用 用于间隔向对方确认继续。
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

该类的核心方法有2个：execute() 和 enqueue() ，在看这两个方法前先简单看下Dispatcher类。

### Dispatcher

下面将先对Dispatcher类的构造、变量以及涉及到同步请求的方法进行解读。

```java
public final class Dispatcher {
  // 所允许的同时进行的网络请求的最大数量
  private int maxRequests = 64;
  // 所允许的同时进行网络请求不同域名总和的最大数量
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  // 线程池
  private @Nullable ExecutorService executorService;

  // 准备执行的异步网络请求队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 正在执行的异步网络请求队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 正在执行的同步网络请求队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();


  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      // 默认情况下，该默认线程数为0，所能最多线程数为Integer.MAX_VALUE，当空闲线程超过60秒没有新任务时销毁回收。
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

  synchronized void enqueue(AsyncCall call) {
  // 如果正在运行的网络请求数和不同域名总和满足条件
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      //线程池执行网络请求（AsyncCall的父类是Runnable，在run中进行网络请求）
      executorService().execute(call);
    } else {
      //不满足则加入等待队列
      readyAsyncCalls.add(call);
    }
  }

  // 同步请求，由于在当前线程堵塞等待请求，因此只用维护一个正在执行的同步网络请求队列。
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  // 异步请求完成
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  // 同步请求完成
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  // 
  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      // 移除队列 calls 中的 call
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      // 是否推进队列，即是否将等待执行的队列添加进正在执行的队列
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    // 当该线程池没有Runnable可执行时回调。
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

}
```

### execute()

execute()为同步请求方式,即会在调用Call.execute()的当前线程进行网络请求。
execute()和enqueue()的核心方法都在getResponseWithInterceptorChain()中。

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
      // dispatcher为网络请求调度器。
      client.dispatcher().executed(this);
      //进行网络请求并得到响应结果
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
        // 进行网络请求并得到响应结果
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

### getResponseWithInterceptorChain()

Interceptor 也叫拦截器，它像工厂流水线一样，传递用户发起的请求 Request，每一个拦截器完成相应的功能，目的也是对网络请求可能存在的需求进行拆分。
拦截器的原理类似于android的触摸事件分发机制，即先执行的拦截器，会拿到最后的response。在每一个拦截器Interceptor的intercept方法中，会在
chain.proceed()方法中把组装完的Request发给下一个拦截器，直至最后一个拦截器的intercept方法 return 得到response，再依次返回给上一个拦截器，依次拿到ruturn
的 response 直至第一个拦截器也return 后得到最终response。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 用户
    interceptors.addAll(client.interceptors());

    // RetryAndFollowUpInterceptor 拦截器，主要针对 3XX 重定向和
    // 部分特殊情况（401 认证处理，408 请求超时，503 服务器暂时不可用）进行处理（请求重试）
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    
    // chain 内部维护了所有要执行的拦截器列表，在 proceed 内部会唤醒下一个 Interceptor ，调用 intercept 来进行下一步
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
