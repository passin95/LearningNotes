


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
Interceptor.Chain的原理类似于android的触摸事件分发机制，即先执行的拦截器，会拿到最后的response。在每一个拦截器Interceptor的intercept()方法中，会调用
chain.proceed()方法中把它自己组装完的Request发给下一个拦截器，直至最后一个拦截器的intercept()方法 return 得到 response（因为除了最后一个拦截器外，调用完chain.proceed()得到的response并不一定马上返回，可能进行后续操作），再依次返回给上一个拦截器，上一个拦截器得到response后继续向下执行直到返回该拦截器最终的response，继续依次先上一个拦截器执行直至得到最终response。同时，如果不想将response向下传递，只要当前拦截器包装完需求后，直接

```java
final class RealCall{

  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加用户通过OkHttpClient添加的拦截器
    interceptors.addAll(client.interceptors());


    // 添加
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    
    // chain 内部维护了所有要执行的拦截器列表，originalRequest 为最初（未经过拦截器处理的）的request。
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    // 开始发起网络请求，并得到最终的response并返回。
    return chain.proceed(originalRequest);
  }
```

#### RetryAndFollowUpInterceptor


```java

public final class RetryAndFollowUpInterceptor implements Interceptor {

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    // 用于协调Connections（例如Https连接过程）、Streams（请求与响应的过程）、Calls（一个Call可能对应多个Connection和Stream）三者之间的关系。
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    // 反复重试直到满足特定条件return结束循环。
    while (true) {
      // 取消请求后，释放资源 抛出异常给调用方处理
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      // 是否释放连接,用于判断在finally是否释放资源
      boolean releaseConnection = true;
      try {
        // 获取响应结果（可能会多次获取，因为可能存在多次重试）
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // priorResponse是重定向前的response。
      // 即response为重定向后请求后得到的response就不再需要添加请求体。
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }
      // 针对不同 responseCode 对Request进行包装后重新请求，当不需要包装的时候返回null。
      Request followUp = followUpRequest(response, streamAllocation.route());

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        //最终返回该拦截器包装后的response。
        return response;
      }

      closeQuietly(response.body());

      //最多调用followUpRequest()方法的次数为20
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      // 请求体已被UnrepeatableRequestBody类标记
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      // followUp中的HttpUrl 和 response中的HttpUrl 是否请求的是同一个URL（一般是针对重定向后URL变化）
      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }

   //此方法返回true将重试，false则不重试。
   private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // 用户配置当连接失败的时候是否进行重试，client.retryOnConnectionFailure()默认为ture
    if (!client.retryOnConnectionFailure()) return false;

    // 如果不是 ConnectionShutdownException 并且请求体已被UnrepeatableRequestBody类标记
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // 该异常e是否可以解决。例如抛出了 ProtocolException,说明协议出问题了，无法解决，则直接返回false。
    if (!isRecoverable(e, requestSendStarted)) return false;

    // 是否还有未尝试连接的ip地址
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }
}
```

#### BridgeInterceptor

BridgeInterceptor 把在 OkHttpClint 的部分配置拼接到实际的请求报文中，相关配置的含义请移步https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md
除此之外，对请求报文的cookie进行提取设置，对响应报文的cookie进行保存，以及响应报文的解码。

```java
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      // OkHttp默认支持gzip压缩方式。
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    // 从cookieJar拿到该URL所对应的保存的cookies。
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // 用户代理，用于针对不同的代理返回不同的response报文。
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    // 交给下一个拦截器，并得到返回的Response。
    Response networkResponse = chain.proceed(requestBuilder.build());

    // 将需要保存的cookie存入cookieJar
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 如果是压缩方式是gzip，则对Response进行解压
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

}
```

#### CacheInterceptor

#### ConnectInterceptor

用于建立Http连接。

```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //用于向TCP 读写数据
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```