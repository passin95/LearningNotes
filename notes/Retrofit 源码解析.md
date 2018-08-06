## 一、Retrofit的基本使用

本文将直接结合Gson以及RxJava的使用以代码+注解的方式按运行流程解读源码，非粗读源码，而是尽量对主流程进行深度剖析。
```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com")
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build();
Observable<List<User>> userList = retrofit.create(UserService.class).getUserList(1, 15);
```

## 二、Retrofit的成员变量

通过 Builder 设计模式初始化变量。
```java
  public static final class Builder {
    // 该类对于android系统来说，
    private final Platform platform;
    // 一般都是 OkHttpClient 
    private @Nullable okhttp3.Call.Factory callFactory;
    // 域名拼接头
    private HttpUrl baseUrl;
    // 数据解析器工厂 例如将body.string()转化为 Json
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    // Call的实现类可以理解为对一次网络请求的封装之后的一个对象，它会执行网络请求以获得响应（回调）
    // 而CallAdapter则是将Call接口适配成我们想要的返回结果。
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
    // 线程调度器，网络请求结束后使用什么线程执行响应。
    private @Nullable Executor callbackExecutor;
    // 是否快速检测类和方法是否规范
    private boolean validateEagerly;
```

```java
public <T> T create(final Class<T> service) {
    // 检查该类是否符合Retrofit使用规范 
    Utils.validateServiceInterface(service);
    // 是否在调用create(Class)时检测接口中所有方法定义是否正确，而不是在调用方法才检测。适合在开发、测试时使用(ServiceMethod 对象构建的时候会进行一系列规范检测)
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    // Java动态代理，Proxy.newProxyInstance会把参数 service 类中的所有方法都实现 InvocationHandler接口中的invoke()方法,在调用该类的方法时，会回调进InvocationHandler接口中的invoke()方法，并
    // 
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
  ```
  adssad