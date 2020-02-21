
# OkHttp 源码分析

<!-- TOC -->

- [OkHttp 源码分析](#okhttp-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
  - [一、说明](#%E4%B8%80%E8%AF%B4%E6%98%8E)
  - [二、OkHttp 的基本使用](#%E4%BA%8Cokhttp-%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8)
  - [三、Request 和 Response](#%E4%B8%89request-%E5%92%8C-response)
    - [3.1 Request](#31-request)
    - [3.2 Response](#32-response)
  - [四、OkHttpClient](#%E5%9B%9Bokhttpclient)
    - [4.1 OkHttpClient 的成员变量](#41-okhttpclient-%E7%9A%84%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F)
    - [4.2 OkHttpClient 对 Call.Factory 的实现](#42-okhttpclient-%E5%AF%B9-callfactory-%E7%9A%84%E5%AE%9E%E7%8E%B0)
  - [五、RealCall](#%E4%BA%94realcall)
    - [5.1 Dispatcher](#51-dispatcher)
    - [5.2 execute() 和 enqueue()](#52-execute-%E5%92%8C-enqueue)
    - [5.3 getResponseWithInterceptorChain()](#53-getresponsewithinterceptorchain)
  - [六、Interceptor](#%E5%85%ADinterceptor)
    - [6.1 RetryAndFollowUpInterceptor](#61-retryandfollowupinterceptor)
    - [6.2 BridgeInterceptor](#62-bridgeinterceptor)
    - [6.3 CacheInterceptor](#63-cacheinterceptor)
    - [6.4 ConnectInterceptor](#64-connectinterceptor)
    - [6.5 CallServerInterceptor](#65-callserverinterceptor)

<!-- /TOC -->

## 一、说明

本文对 Http 基础知识不会过多描述，若对 Http 基础不够了解请先移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 。

本文源码为 OkHttp 3.10.0 版本，该版本 OkHttp 底层已不再使用 HttpURLConnection，而是自己重写了 TCP/IP 层的实现。

## 二、OkHttp 的基本使用

```java 

// OkHttpClient 为网络请求器工厂接口 Call.Factory 的实现类。
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

## 三、Request 和 Response

### 3.1 Request

Request 类比较简单，主要是对构建 Http 请求报文所需要的数据。

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
      // 默认请求方法为 GET。
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
}
```

### 3.2 Response

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
  // 重定向前响应结果
  final @Nullable Response priorResponse;
  // 发送请求报文时间戳
  final long sentRequestAtMillis;
  // 接口响应报文时间戳
  final long receivedResponseAtMillis;
  // 缓存控制器
  private volatile CacheControl cacheControl; // Lazily initialized.
  
}
```

## 四、OkHttpClient

### 4.1 OkHttpClient 的成员变量

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  
  // 网络请求调度器。
  final Dispatcher dispatcher;
  // 代理方式，分别有 DIRECT (直连)、HTTP、SOCKS 三种。
  final @Nullable Proxy proxy;
  // 所支持的应用层协议，例如 http1.0、http1.1，http2.0。
  final List<Protocol> protocols;
  // Https 的 TLS 版本和密码件的选择，OkHttp 默认使用 MODERN_TLS 和 CLEARTEXT 。
  // MODERN_TLS 是连接到最新的 HTTPS 服务器的安全配置；
  // COMPATIBEL_TLS 是连接到过时的 HTTPS 服务器的安全配置；
  // CLEARTEXT 是用于 http://开头的 URL 的非安全配置。
  final List<ConnectionSpec> connectionSpecs;
  // 网络请求拦截器（这 2 个的区别下面会讲）。
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  // 网络请求过程监听工厂。
  final EventListener.Factory eventListenerFactory;
  // 针对存在使用多个代理时，选择下一次网络请求使用的代理方式。
  final ProxySelector proxySelector;
  // CookieJar 是一个接口方法，得自己实现 Cookie 的存储和取出方式。OkHttp 会请求和响应的过程中自动在请求头中配置 Cookie。
  // 在接收到 Cookie 时存放于 List<Cookie> 中，以及在使用时根据 HttpUrl 去找到对应的 Cookie。
  final CookieJar cookieJar;
  // OkHttp 默认提供的 Http 响应缓存类，实现方式是 DiskLruCache。默认没有开启，需自行配置缓存存储位置和存储空间上限。
  // 缓存策略是 Http 协议的缓存规范，详情可看 CacheStrategy 类。
  final @Nullable Cache cache;
  // 提供拓展给开发者实现自己的缓存算法。
  // 如果不想使用 OkHttp 提供的缓存方式 Cache，可自定义缓存方式。InternalCache 和 Cache 的缓存方式只会取其一。
  final @Nullable InternalCache internalCache;
  // Socket 工厂，用于创建 Socket，也就是 TCP 端口。
  final SocketFactory socketFactory;
  // 建立 SSL 连接的工厂类。
  final @Nullable SSLSocketFactory sslSocketFactory;
  // 对服务器发送过来的证书进行层级梳理。
  final @Nullable CertificateChainCleaner certificateChainCleaner;
  // 用于验证 HTTPS 握手过程中对方发过来的证书中的所属者和用户请求服务器的 Hostname 是否一致。
  final HostnameVerifier hostnameVerifier;
  // Android 证书锁，即不全信任客户端证书库，还对服务器证书传过来的证书指纹和 CertificatePinner 写入的指纹进行匹配。
  final CertificatePinner certificatePinner;
  // 当返回错误码 401 时，自动调用该接口的实现类进行认证。区别在于第一个是向代理服务器进行认证，第二个是向目标服务器进行认证。
  final Authenticator proxyAuthenticator;
  final Authenticator authenticator;
  // 连接池，多个 OkHttpClient 可共用一个连接池。
  final ConnectionPool connectionPool;
  // 域名解析就是将人们惯用的域名转换成为机器可读的 IP 地址的过程，可使用腾讯云或阿里云的 DNS 提升解析速度和安全性。
  final Dns dns;
  // 是否在 https 收到 http 请求时（反之亦然），是否自动请求重定向指向的网站。默认为 true。
  final boolean followSslRedirects;
  // 是否自动请求重定向指向的网站。优先级比 followSslRedirects 高。
  final boolean followRedirects;
  // 请求失败后是否需要重试。
  final boolean retryOnConnectionFailure;
  // 连接超时时间（包含了 TCP 连接、TLS 验证等）。
  final int connectTimeout;
  // 发出 request 报文结束至收到的 response 报文所允许的最大时间
  final int readTimeout;
  // 发送 request 报文开始至发送 request 报文结束所允许的最大时间
  final int writeTimeout;
  // 用于 webSocket ，间隔向对方确认心跳。
  final int pingInterval;
  
}
```

### 4.2 OkHttpClient 对 Call.Factory 的实现

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory{

  // 使用工厂模式构建 Call 的实现类 RealCall。
  @Override public Call newCall(Request request) {
    // 三个参数分别为 OkHttpClient 本身，Request，是否使用 WebSocket。
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

}
```

## 五、RealCall

该类的核心方法有 2 个：execute() 和 enqueue()，其中 execute 是同步请求，enqueue 是异步请求。

在看这两个方法前先简单看下 Dispatcher 类。

### 5.1 Dispatcher

下面将先对 Dispatcher 类的构造、变量以及涉及到同步请求的方法进行解读。

```java
public final class Dispatcher {
  
  // 所允许的同时进行网络请求的最大数量。
  private int maxRequests = 64;
  // 所允许的同时进行网络请求不同域名总和的最大数量。
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  // 该线程池只在异步请求时使用，一般情况下使用 Retrofit 都是同步请求。
  // 若有异步请求的需求，最好手动传入项目统一使用的线程池，优化性能。
  private @Nullable ExecutorService executorService;

  // 准备执行的异步网络请求队列。
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 正在执行的异步网络请求队列。
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 正在执行的同步网络请求队列。
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
    // 如果正在运行的网络请求数和不同域名总和满足条件。
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      // 线程池执行网络请求（AsyncCall 的父类是 Runnable，在 run 中进行网络请求）。
      executorService().execute(call);
    } else {
      // 不满足则加入等待队列。
      readyAsyncCalls.add(call);
    }
  }

  // 同步请求，由于在当前线程堵塞等待请求，因此只用维护一个正在执行的同步网络请求队列。
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  // 异步请求完成。
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  // 同步请求完成。
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    // idleCallback 空闲时回调。
    Runnable idleCallback;
    synchronized (this) {
      // 移除队列 calls 中的 call。
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      // 是否推进队列，即是否将等待执行的队列添加进正在执行的队列。
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

### 5.2 execute() 和 enqueue()

execute() 和 enqueue() 的核心方法都在 getResponseWithInterceptorChain() 中。

execute()：

```java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  // 网络请求过程监听，默认空实现。
  // 网络请求开始事件回调。
  eventListener.callStart(this);
  try {
    // dispatcher 为网络请求调度器。
    client.dispatcher().executed(this);
    // 进行网络请求并得到响应结果。
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } catch (IOException e) {
    // 网络请求失败事件回调。
    eventListener.callFailed(this, e);
    throw e;
  } finally {
    // execute 请求完成。
    client.dispatcher().finished(this);
  }
}
```

enqueue()：

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
  
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 进行网络请求并得到响应结果。
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

### 5.3 getResponseWithInterceptorChain()

```java
final class RealCall{

  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加用户自定义的拦截器。最先拿到 request，最后拿到 response。
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
    
    // chain 内部维护了所有要执行的拦截器列表，originalRequest 为原始（未经过拦截器处理的）的 request。
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    // 开始发起网络请求，并得到最终的 response 并返回。
    return chain.proceed(originalRequest);
  }
```

```java
public final class RealInterceptorChain implements Interceptor.Chain {

 public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // 一个 Request 请求只能使用一次。
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    // 调用链中的下一个拦截器。
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
}
```

## 六、Interceptor

Interceptor 也叫拦截器，它像工厂流水线一样，传递用户发起的请求 Request，每一个拦截器完成相应的功能，目的也是对网络请求可能存在的需求和问题进行拆分处理。

Interceptor.Chain 的原理类似于 Android 的触摸事件分发机制，即在拦截器不拦截的情况下，先执行的拦截器，会拿到最后的 response。

在每一个拦截器 Interceptor 的 intercept() 中，会调用 chain.proceed() 中把它自己组装完的 Request 通过递归发给下一个拦截器，直至最后一个拦截器的 intercept() return 得到 response（调用完 chain.proceed() 得到的 response 并不一定马上返回，还可以对其进行后续操作），上一个拦截器得到 response 后继续向下执行至 return，并一直反复递归至返回得到最终的 response。

同时，如果在特定条件下不想将 Request 向下传递，只需要在特定条件下构建一个新的 response 并在调用 chain.proceed() 后 return 掉（拦截掉）。

### 6.1 RetryAndFollowUpInterceptor

RetryAndFollowUpInterceptor 顾名思义，用于在某些情况进行网络请求重试的作用。

```java
public final class RetryAndFollowUpInterceptor implements Interceptor {

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    // 用于协调 Connections（Https 和 TCP 的连接）、Streams（运输层）、Calls 三者之间的关系。
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    // 反复重试直到满足特定条件 return 结束循环。
    while (true) {
      // 取消请求后，释放资源，抛出异常给调用方处理。
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      // 是否释放连接,用于判断在 finally 是否释放资源。
      // 当遇到无法处理或连接出现异常时，释放 StreamAllocation 所有资源。
      // 若无法处理，则抛出异常中止请求，否则重新连接。
      boolean releaseConnection = true;
      try {
        // 获取响应结果（可能会多次获取，因为可能存在多次重试）。
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

      // 若 priorResponse 不为 null，则代表 priorResponse 是本次重定向前的 response。
      if (priorResponse != null) {
        response = response.newBuilder()
            // 对于新的 response，重定向前的 body 已毫无意义。
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      // 根据 response 和 Route 去判定是否需要再次请求。
      // 以 responseCode 去判定 超时、401 授权失败、代理、重定向等情况是否需要进行再次请求。
      Request followUp = followUpRequest(response, streamAllocation.route());

      // followUp 不为 null，表示不再需要再次请求。
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        // 返回最终的 response。
        return response;
      }

      closeQuietly(response.body());

      // 最多调用 followUpRequest() 方法的次数为 20。
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      // 实现 UnrepeatableRequestBody 接口的请求体不应该进行重试。提供给开发者自定义请求体。
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      // followUp 中的 HttpUrl 和 response 中的 HttpUrl 是否请求的是同一个 URL（一般是针对重定向后 URL 变化）。
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

    // 用户配置当连接失败的时候是否进行重试，client.retryOnConnectionFailure() 默认为 ture。
    if (!client.retryOnConnectionFailure()) return false;

    // 如果不是 ConnectionShutdownException 并且请求体已被 UnrepeatableRequestBody 类标记。
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // 该异常 e 是否可以解决。例如抛出了 ProtocolException，说明协议出问题了，无法解决，则直接返回 false。
    if (!isRecoverable(e, requestSendStarted)) return false;

    // 是否还有未尝试连接的路由地址。
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }

}
```

### 6.2 BridgeInterceptor

BridgeInterceptor 作用是将 OkHttpClint 的部分配置拼接到请求报文中，相关配置的含义请移步 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md
除此之外，对请求报文的 cookie 进行提取设置，对响应报文的 cookie 进行保存,以及提供默认的请求报文、响应报文的编解码方式。

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
      // 默认让连接存活一段时间。
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

    // 从 cookieJar 拿到该 URL 所对应的保存的 cookies。OkHttp 没有提供默认实现，因此默认情况下拿到的都是空列表。
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

    // 将需要保存的 cookie 存入 cookieJar。
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 如果是压缩方式是 gzip，则对 Response 进行解压。
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

### 6.3 CacheInterceptor

```java
public final class CacheInterceptor implements Interceptor {
  
  final InternalCache cache;

  public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    // OkHttp 默认不进行缓存，若设置了 Cache 则以 Request 作为 key 获取所对应的 cacheCandidate(缓存响应)。
    // 默认实现类 Cache 以 Request 的 HttpUrl 作为 Key 保存和提取。
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    
    // 这里是整个缓存机制的关键，CacheStrategy 是 OkHttp 遵循 Http 的缓存规范的实现类，
    // 会根据请求头所使用的缓存策略和所处的手机状况对 networkRequest 和 cacheResponse 2 个变量进行配置。
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    
    // networkRequest 仅在不使用网络（请求头 Cache-Control:only-if-cached）、缓存未过期、缓存有效且无变化（Cache-Control:immutable）为 null。
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    // 用于记录 APP 运行期间，发起请求次数、网络请求次数（和服务器建立连接）、缓存使用次数。
    if (cache != null) {
      cache.trackResponse(strategy);
    }
    
    if (cacheCandidate != null && cacheResponse == null) {
      // 缓存已失效，释放该次缓存所占的资源。
      closeQuietly(cacheCandidate.body()); 
    }

    // 如果不使用网络并且缓存不存在，则构建一个 code 为 504 的 Response。
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

    if (networkRequest == null) {
      // 执行到这里，说明不使用网络且缓存没过期。
      // 返回缓存的响应体，设置 cacheResponse 又没设置 networkResponse，从而其它拦截器可以得知该响应体的数据来源只有缓存。
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    
    // 执行到这里，说明没有缓存或缓存失效，接下来将尝试进行网络连接通信。
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    if (cacheResponse != null) {
      // 执行到这里，说明缓存存在，但是已过期。
      // 该种缓存策略叫 ETag，用于判断自从上次请求后，请求的链接内容是否已被更改（由服务器在请求头告知），
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        // 用缓存 response 为主和服务器响应的部分信息 拼接 response 返回。 
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
        // 更新缓存。
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    // 构建最后的 response，指定一下该次请求可能存在的无效缓存。
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    // 如果使用了 cache，则尝试该 response 进行缓存。
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

### 6.4 ConnectInterceptor

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
    // 查找可用的连接后，打开与目标服务器的连接。
    // 配置 Socket 的相关信息以及确定 Http 请求编码、响应解码的方式（例如 Http1 和 Http2 的协议不一致）。
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

### 6.5 CallServerInterceptor

负责与服务器连接过程中请求与响应的 I/O 操作，即向 Socket 里写入请求数据和读取响应数据。

作为最后一个拦截器，不会再调用 Chain.proceed()，而是直接从 I/O 流中读取数据构建 Response。

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

    // 向 BufferedSink(OutputStream) 中写入请求头信息。
    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    // 如果该方法可能包含请求体并且该方法的请求体不为 null。
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // 请求头添加了"Expect:100-continue", 用于询问服务器是否愿意接受数据，只有等到服务器的应答后，再将数据发送给服务器。
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        // 将 BufferedSink 中的数据刷到接收端（服务器）。
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        // 读取 BufferedSource 中的数据构建 Response.Builder。
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      // 若 responseBuilder == null，表示服务器有响应体返回，让客户端重新读取数据流。
      if (responseBuilder == null) {
        // 开始写入请求正文。
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        // 写入请求体。
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // 服务器拒绝，则关闭此次连接，防止 HTTP/1 连接被重用。
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    // 将请求报文的所有数据刷到服务器。
    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      // 再次读取响应体的数据。
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    // 构建 Response, 写入 request，握手情况，请求时间，响应时间。
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // 服务器返回此响应码 code 表示服务器只发送了一部分响应报文，仍有数据未发送。
      // 继续尝试读取响应数据。
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
      // 设置响应体。
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    // 如果添加了请求头“Connection:close”，则在请求完毕后关闭连接，OkHttp 默认使用长连接。
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    // 响应码 code 为 204 或者 205，一般不包含响应体，若返回了响应体，则抛出 ProtocolException。
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```