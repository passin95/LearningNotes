# OkHttp 源码解析
<!-- TOC -->

- [OkHttp 源码解析](#okhttp-源码解析)
    - [说明](#说明)
    - [一、OkHttp 的基本使用](#一okhttp-的基本使用)
    - [二、Request 和 Response](#二request-和-response)
        - [Request](#request)
        - [Response](#response)
    - [OkHttpClient](#okhttpclient)
        - [OkHttpClient 的成员变量](#okhttpclient-的成员变量)
        - [OkHttpClient 对 Call.Factory 的实现](#okhttpclient-对-callfactory-的实现)
    - [三、RealCall](#三realcall)
        - [Dispatcher](#dispatcher)
        - [execute()](#execute)
        - [enqueue()](#enqueue)
        - [getResponseWithInterceptorChain()](#getresponsewithinterceptorchain)
    - [四、Interceptor](#四interceptor)
        - [RetryAndFollowUpInterceptor](#retryandfollowupinterceptor)
        - [BridgeInterceptor](#bridgeinterceptor)
        - [CacheInterceptor](#cacheinterceptor)
        - [ConnectInterceptor](#connectinterceptor)
        - [CallServerInterceptor](#callserverinterceptor)

<!-- /TOC -->


## 说明

本文对 Http 基础知识不会过多描述，若对 Http 基础不够了解请先移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 。

本文源码为 OkHttp 3.10.0 版本，该版本 OkHttp 底层已不再使用 HttpURLConnection，而是自己重写了 tcp/ip 层的实现。

## 一、OkHttp 的基本使用

```java 

// OkHttpClient 为网络请求器工厂接口 Call.Factory 的实现类
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

## 二、Request 和 Response

### Request

Request 类比较简单，主要是对构建 Http 请求报文的变量进行赋值。

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
  // 缓存控制器，以请求头的方式使用。
  private volatile CacheControl cacheControl; // Lazily initialized.

  public static class Builder {

    public Builder() {
      // 默认请求方法为 GET
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
}
```

### Response

```java
public final class Response implements Closeable {
  // 请求体
  final Request request;
  // 应用层协议
  final Protocol protocol;
  // 返回状态码
  final int code;
  // 附带消息
  final String message;
  // SSL/TLS 握手协议验证时的信息
  final @Nullable Handshake handshake;
  // 响应头
  final Headers headers;
  // 响应体
  final @Nullable ResponseBody body;
  // 网络请求响应结果
  final @Nullable Response networkResponse;
  // 缓存响应结果
  final @Nullable Response cacheResponse;
  // 重定向前
  final @Nullable Response priorResponse;
  // 发送请求报文时间戳
  final long sentRequestAtMillis;
  // 接口响应报文时间戳
  final long receivedResponseAtMillis;
  // 缓存控制器
  private volatile CacheControl cacheControl; // Lazily initialized.
}
```

## OkHttpClient

### OkHttpClient 的成员变量

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  // 网络请求调度器
  final Dispatcher dispatcher;
  // 代理方式，分包有 DIRECT (直连)、HTTP 、SOCKS 三种。
  final @Nullable Proxy proxy;
  // 具体使用的哪一版本的应用层协议，例如 http1.0、http1.1，http2.0。
  final List<Protocol> protocols;
  // Http 和 Https 的 TLS 版本和 密码套件的选择，okhttp 默认优先使用 MODERN_TLS。
  // MODERN_TLS 是连接到最新的 HTTPS 服务器的安全配置。
  // COMPATIBEL_TLS 是连接到过时的 HTTPS 服务器的安全配置。
  // CLEARTEXT 是用于 http://开头的 URL 的非安全配置。
  final List<ConnectionSpec> connectionSpecs;
  // 拦截器（这 2 个的区别下面会讲）
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  // 网络请求时间监听工厂
  final EventListener.Factory eventListenerFactory;
  // 针对存在使用多个代理时，选择下一次网络请求使用的代理方式。
  final ProxySelector proxySelector;
  // CookieJar 是一个接口方法，用于使用 Cookie 时存放 List<Cookie> 和根据 HttpUrl 去找到相应 Cookie。
  final CookieJar cookieJar;
  // OkHttp 提供的缓存类。
  final @Nullable Cache cache;
  // CacheInterceptor 为缓存操作应该具有的方法接口，构造函数的参数为 InternalCache，即面向接口编程。
  // 提供拓展给开发者实现自己的缓存算法。
  final @Nullable InternalCache internalCache;
  // Socket 工厂，网络层建立连接使用。
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
  // 是否在 https 收到 http 请求时（反之亦然），是否自动请求重定向指向的网站。
  final boolean followSslRedirects;
  // 是否自动请求重定向指向的网站。
  final boolean followRedirects;
  // 请求失败后是否需要重试。
  final boolean retryOnConnectionFailure;
  // 连接超时时间。
  final int connectTimeout;
  // 发出 request 报文结束至收到的 response 报文所允许的最大时间。
  final int readTimeout;
  // 发送 request 报文开始至发送结束所允许的最大时间。
  final int writeTimeout;
  // webSocket 使用 用于间隔向对方确认继续。
  final int pingInterval;
```

### OkHttpClient 对 Call.Factory 的实现

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory{

    // 使用工厂模式构建 Call 的实现类 RealCall
  @Override public Call newCall(Request request) {
    // 三个参数分别为 OkHttpClient 本身，Request，是否使用 WebSocket
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

}
```

## 三、RealCall

该类的核心方法有 2 个：execute() 和 enqueue() ，在看这两个方法前先简单看下 Dispatcher 类。

### Dispatcher

下面将先对 Dispatcher 类的构造、变量以及涉及到同步请求的方法进行解读。

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
      // 默认情况下，该默认线程数为 0，所能最多线程数为 Integer.MAX_VALUE，当空闲线程超过 60 秒没有新任务时销毁回收。
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

  synchronized void enqueue(AsyncCall call) {
  // 如果正在运行的网络请求数和不同域名总和满足条件
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      // 线程池执行网络请求（AsyncCall 的父类是 Runnable，在 run 中进行网络请求）
      executorService().execute(call);
    } else {
      // 不满足则加入等待队列
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

    // 当该线程池没有 Runnable 可执行时回调。
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

}
```

### execute()

execute() 为同步请求方式,即会在调用 Call.execute() 的当前线程进行网络请求。
execute() 和 enqueue() 的核心方法都在 getResponseWithInterceptorChain() 中。

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    // 网络请求过程监听，默认空实现
    // 网络请求开始事件回调
    eventListener.callStart(this);
    try {
      // dispatcher 为网络请求调度器。
      client.dispatcher().executed(this);
      //进行网络请求并得到响应结果
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      // 网络请求失败事件回调
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

```java
final class RealCall{

  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加用户通过 OkHttpClient 添加的拦截器
    interceptors.addAll(client.interceptors());

    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      // client.networkInterceptors() 和 client.interceptors() 的本质区别在于，networkInterceptor 可能会被上面（先添加的）的拦截器拦截
      // 例如当使用缓存并且缓存有效时，CacheInterceptor 会直接返回缓存的 Response，不将 Request 向下发送。
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    
    // chain 内部维护了所有要执行的拦截器列表，originalRequest 为原始（未经过拦截器处理的）的 request
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    // 开始发起网络请求，并得到最终的 response 并返回
    return chain.proceed(originalRequest);
  }
```

## 四、Interceptor

Interceptor 也叫拦截器，它像工厂流水线一样，传递用户发起的请求 Request，每一个拦截器完成相应的功能，目的也是对网络请求可能存在的需求和问题进行拆分。
Interceptor.Chain 的原理类似于 android 的触摸事件分发机制，即先执行的拦截器，会拿到最后的 response。在每一个拦截器 Interceptor 的 intercept() 方法中，会调用
chain.proceed() 方法中把它自己组装完的 Request 发给下一个拦截器，直至最后一个拦截器的 intercept() 方法 return 得到 response（除了最后一个拦截器外，调用完 chain.proceed() 得到的 response 并不一定马上返回，可能进行后续操作），再 return 返回给上一个拦截器，上一个拦截器得到 response 后继续向下执行至 return，返回该拦截器最终的 response，继续依次先上一个拦截器执行直至得到最终 response。同时，如果在特定条件下不想将 Request 向下传递，只需要在特定条件下构建一个新的 response 并在调用 chain.proceed()return 掉。

### RetryAndFollowUpInterceptor

```java
public final class RetryAndFollowUpInterceptor implements Interceptor {

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    // 用于协调 Connections（例如 Https、tpc 连接过程）、Streams（请求与响应的过程）、Calls（一个 Call 可能对应多个 Connection 和 Stream）三者之间的关系。
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    // 反复重试直到满足特定条件 return 结束循环。
    while (true) {
      // 取消请求后，释放资源 抛出异常给调用方处理
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      // 是否释放连接,用于判断在 finally 是否释放资源
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

      // priorResponse 是重定向前的 response。
      // 即 response 为重定向后请求后得到的 response 就不再需要原请求的 ResponseBody。
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }
      // 针对不同 responseCode 对 Request 进行包装后重新请求，当不需要包装的时候返回 null。
      Request followUp = followUpRequest(response, streamAllocation.route());

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        // 最终返回该拦截器包装后的 response。
        return response;
      }

      closeQuietly(response.body());

      // 最多调用 followUpRequest() 方法的次数为 20
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      // 请求体已被 UnrepeatableRequestBody 类标记
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      // followUp 中的 HttpUrl 和 response 中的 HttpUrl 是否请求的是同一个 URL（一般是针对重定向后 URL 变化）
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

   // 此方法返回 true 将重试，false 则不重试。
   private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // 用户配置当连接失败的时候是否进行重试，client.retryOnConnectionFailure() 默认为 ture
    if (!client.retryOnConnectionFailure()) return false;

    // 如果不是 ConnectionShutdownException 并且请求体已被 UnrepeatableRequestBody 类标记
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // 该异常 e 是否可以解决。例如抛出了 ProtocolException,说明协议出问题了，无法解决，则直接返回 false。
    if (!isRecoverable(e, requestSendStarted)) return false;

    // 是否还有未尝试连接的 ip 地址
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }
}
```

### BridgeInterceptor

BridgeInterceptor 把在 OkHttpClint 的部分配置拼接到实际的请求报文中，相关配置的含义请移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md
除此之外，对请求报文的 cookie 进行提取设置，对响应报文的 cookie 进行保存，以及响应报文的解码。

```java
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    // 拼接请求信息
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
      // OkHttp 默认支持 gzip 压缩方式。
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    // 从 cookieJar 拿到该 URL 所对应的保存的 cookies。
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // 用户代理，用于针对不同的代理返回不同的 response 报文。
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    // 交给下一个拦截器，并得到返回的 Response。
    Response networkResponse = chain.proceed(requestBuilder.build());

    // 将需要保存的 cookie 存入 cookieJar
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 如果是压缩方式是 gzip，则对 Response 进行解压
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

### CacheInterceptor


```java
public final class CacheInterceptor implements Interceptor {
  // 网络请求缓存接口规范，OkHttp 默认提供了实现类 Cache
  final InternalCache cache;

  public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    // OkHttp 默认不进行 Cache，若设置了 Cache 则传入 Request 提取该次 Request 的 cacheCandidate(缓存响应)
    // Cache 以 Request 的 HttpUrl 作为 Key 保存和提取
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    
    // 这里是整个缓存机制的关键，CacheStrategy 会根据请求头所使用的缓存策略和所处的手机状况对 networkRequest 和 cacheResponse 下面 2 个变量进行构建或置 null
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();

    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    // 用于记录 APP 运行期间，发起请求次数、网络请求次数（和服务器建立连接）、缓存使用次数
    if (cache != null) {
      cache.trackResponse(strategy);
    }
    
    if (cacheCandidate != null && cacheResponse == null) {
      // 缓存已失效，释放该次缓存的资源
      closeQuietly(cacheCandidate.body()); 
    }

    // 构建一个失败的 Response 
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
    // 不使用网络并拥有合适的缓存后构建新的 Response 返回给上一个拦截器
    if (networkRequest == null) {
      // 构建新的 Response，并将无用的 body 置于 null，减小内存开销，这里的细节如下：
      // 1.将新的 Response 的 Body 使用的 cacheResponse 的 ResponseBody，即缓存下来的 ResponseBody
      // 2.将 cacheResponse 的 ResponseBody 置 null，并赋值给新的 Response 的变量 cacheResponse。
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    
    // 执行到这里，说明没有使用缓存或本地便可校验的缓存失效，接下来将尝试进行网络连接通信。
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 该种缓存策略叫 ETag，用于判断自从上次请求后，请求的链接内容是否已被更改
    // 如果未更改，返回响应码 304 ，并且不会返回响应体内容。
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        // 符合 ETag 缓存策略，利用缓存的 response 构建 response 返回
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    // 拿到正常请求到的的 response
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    // 如果使用了 cache，则对 response 进行缓存。
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

### ConnectInterceptor

ConnectInterceptor 是 OkHttp 对 TCP 层的封装，此处不再深挖源码。

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
    // 用于向 TCP 层读写数据
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    // 获取到可用的连接并建立并加入连接池管理
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

### CallServerInterceptor

```java
public final class CallServerInterceptor implements Interceptor {
  private final boolean forWebSocket;

  public CallServerInterceptor(boolean forWebSocket) {
    this.forWebSocket = forWebSocket;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    // 向 BufferedSink(OutputStream) 中写请求头信息
    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    // 如果该方法可能包含请求体并且该方法的请求体不为 null
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      //请求头添加了"Expect:100-continue", 用于询问服务器是否愿意接受数据，只有等到服务器的应答后，再将数据发送给服务器
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // 服务器同意接受数据，开始写入请求正文。
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // 服务器拒绝接受，则关闭此次连接，防止HTTP/1 连接被重用
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    // 构建 Response, 写入 request，握手情况，请求时间，响应时间
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    // 服务器返回此响应码 code 表示已收到请求的第一部分,正在等待其余部分
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    // 用于 webSocket，请求者要求服务器切换协议，服务器已确认并准备切换。
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      // 读取 ResponseBody 数据并赋值
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    // 如果添加了请求头“Connection:close”，则关闭连接，OkHttp默认使用长连接
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    // 响应码code 为 204 或者 205，一般不包含响应体，若返回了响应体，则抛 ProtocolException
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```