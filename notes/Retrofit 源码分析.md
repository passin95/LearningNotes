
# Retrofit 源码分析

<!-- TOC -->

- [Retrofit 源码分析](#retrofit-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
  - [一、Retrofit 的基本使用](#%E4%B8%80retrofit-%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8)
  - [二、Retrofit 的构建](#%E4%BA%8Cretrofit-%E7%9A%84%E6%9E%84%E5%BB%BA)
    - [2.1 Retrofit 的成员变量](#21-retrofit-%E7%9A%84%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F)
    - [2.2 Retrofit.Builder](#22-retrofitbuilder)
  - [三、Retrofit.create()](#%E4%B8%89retrofitcreate)
    - [3.1 Retrofit.loadServiceMethod()](#31-retrofitloadservicemethod)
  - [四、ServiceMethod 的构建](#%E5%9B%9Bservicemethod-%E7%9A%84%E6%9E%84%E5%BB%BA)
    - [4.1 ServiceMethod 的成员变量](#41-servicemethod-%E7%9A%84%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F)
    - [4.2 ServiceMethod.Builder](#42-servicemethodbuilder)
      - [4.2.1 createCallAdapter()](#421-createcalladapter)
      - [4.2.2 createResponseConverter()](#422-createresponseconverter)
      - [4.2.3 parseParameter()](#423-parseparameter)
  - [五、CallAdapter](#%E4%BA%94calladapter)
    - [5.1 RxJava2CallAdapterFactory](#51-rxjava2calladapterfactory)
  - [六、Converter](#%E5%85%ADconverter)
    - [6.1 BuiltInConverters](#61-builtinconverters)
  - [七、OkHttpCall](#%E4%B8%83okhttpcall)
    - [7.1 createRawCall()](#71-createrawcall)
    - [7.2 parseResponse()](#72-parseresponse)
  - [八、serviceMethod.adapt(okHttpCall)](#%E5%85%ABservicemethodadaptokhttpcall)
    - [8.1 BodyObservable](#81-bodyobservable)

<!-- /TOC -->

## 一、Retrofit 的基本使用

本文将直接结合 Gson 以及 RxJava 的使用以代码+注解的方式按运行流程解读源码，主要对主流程进行源码剖析。

本文基于 Retrofit 2.4.0。

```java

public interface UserService {
    String HEADER_API_VERSION = "Accept: application/vnd.github.v3+json";

    @Headers({HEADER_API_VERSION})
    @GET("/users")
    Observable<List<User>> getUsers( @Query("since") int lastIdQueried,
            @Query("per_page") int perPage);
}
```

```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com")
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build();
Observable<List<User>> userList = retrofit.create(UserService.class).getUserList(1, 15);
```

## 二、Retrofit 的构建

### 2.1 Retrofit 的成员变量

通过 Builder 设计模式初始化变量。
```java
public final class Retrofit {
// 以 Method 为单位作为每个网络请求中的请求体的缓存使用
private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
// okhttp3.Call 为网络请求器，利用工程模式对外提供拓展并解耦，一般使用 okhttp 提供的默认实现 OkHttpClient
private @Nullable okhttp3.Call.Factory callFactory;
// 网络请求域名拼接头
private HttpUrl baseUrl;
// 数据转换器工厂列表
private final List<Converter.Factory> converterFactories = new ArrayList<>();
// 网络请求适配器
private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
// 线程调度器，网络请求结束后使用什么线程执行响应
private @Nullable Executor callbackExecutor;
// 是否快速检测类、方法、注解是否规范
private boolean validateEagerly;
```

### 2.2 Retrofit.Builder

```java
public Retrofit build() {
    if (baseUrl == null) {
      throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    
    if (callFactory == null) {
      // 默认使用 OkHttp 提供的 OkHttpClient。
      callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    // 默认根据所在的编译环境去取决于使用哪种具体 platform 的实现类。
    if (callbackExecutor == null) {
      callbackExecutor =  .defaultCallbackExecutor();
    }

   
    // 网络请求适配器工厂集合存储顺序：自定义 1 适配器工厂、自定义 2 适配器工厂...Android 环境下默认适配器工厂（ExecutorCallAdapterFactory）
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    
    callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // 数据转换器工厂。
    List<Converter.Factory> converterFactories =
        new ArrayList<>(1 + this.converterFactories.size());
        
    // 数据转换器工厂集合存储的是：默认数据转换器工厂（BuiltInConverters）、自定义 1 数据转换器工厂（GsonConverterFactory）、自定义 2 数据转换器工厂....
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);

   // 构建 Retrofit 对象。
    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
  }
```

## 三、Retrofit.create()

```java
public <T> T create(final Class<T> service) {
    // 检查该类是否是接口以及该类是否有 extends 其它接口。
    Utils.validateServiceInterface(service);
    // 是否在调用 create(Class) 时检测接口中所有方法是否符合规范，而不是在调用方法才检测，
    // 适合在开发、测试时使用 (ServiceMethod 对象构建的时候会进行一系列的检查)。
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    // Java 动态代理，Proxy.newProxyInstance 会把 service 类中的所有方法都实现 InvocationHandler 接口中的 invoke() 方法，
    // 在调用该类的任何方法，都会回调 invoke()。
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // 如果该方法是在 Object.class 中申明的，则直接调用原本 Object 中的方法。
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 针对 Java 环境做的兼容，当接口方法有默认实现时返回 true（Java8 特性）。
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 通过解析 service 接口方法的参数、返回值和注解类型，得到网络请求所需的数据、网络请求适配器工厂、数据转换器工厂。
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            // OkHttpCall 是 Retrofit 中抽象网络接口的 OkHttp 实现。
            // 本质上是 Retrofit 提供网络请求数据的来源和网络响应数据的转换，OkHttp 提供网络请求的支持。
            // 传入网络请求所需的数据和方法参数实例化 OkHttpCall。
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 返回网络请求适配器转换的结果，也就是 method 的返回值。
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}
```

### 3.1 Retrofit.loadServiceMethod()

该方法比较简单，使用了懒加载的思想，由于 ServiceMethod 在实例化的过程需要解析注解且比较耗时，因此进行了缓存。

```java
// Retrofit 的全局变量。
private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?, ?> loadServiceMethod(Method method) {

  ServiceMethod<?, ?> result = serviceMethodCache.get(method);
  // 如果 result 存在直接返回。
  if (result != null) return result;
  // 对 serviceMethodCache 进行双重校验锁，防止多线程操作时可能重复构建。
  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = new ServiceMethod.Builder<>(this, method).build();
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

## 四、ServiceMethod 的构建

### 4.1 ServiceMethod 的成员变量

```java
final class ServiceMethod<R, T> {
  
  private final okhttp3.Call.Factory callFactory;
  // 网络请求适配器
  private final CallAdapter<R, T> callAdapter;
  // 请求地址拼接头
  private final HttpUrl baseUrl;
  // 数据转换器
  private final Converter<ResponseBody, R> responseConverter;
  // Http 网络请求方法,例如"GET"
  private final String httpMethod;
  // 网络请求的相对地址,与 baseUrl 进行拼接
  private final String relativeUrl;
  // 余下的描述请到 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 进行深入了解。
  private final Headers headers;
  private final MediaType contentType;
  private final boolean hasBody;
  private final boolean isFormEncoded;
  private final boolean isMultipart;

  // ParameterHandler 是一个抽象类，用于解析方法中参数的注解向 RequestBuilder 提供数据。RequestBuilder 包含了一个请求报文所需的所有数据。
  private final ParameterHandler<?>[] parameterHandlers;

}
```

### 4.2 ServiceMethod.Builder

```java
public ServiceMethod build() {
  // 根据方法返回类型和方法注解（方法注解不一定会用到，例如 RxJavaCallAdapter 便没用到），遍历获取到合适的网络请求适配器。
  callAdapter = createCallAdapter();
  // responseType 为最终想要的响应体类型。
  // 例如对于 RxJava2CallAdapter 中，假设接口方法返回值为 Observable<User>，则 responseType 为 User。
  responseType = callAdapter.responseType();
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  // 根据 responseType 和方法上注解，遍历获取可用的数据转换器。
  // 例如默认情况下的响应体为 ResponseBody，可通过转换器转为 User。
  responseConverter = createResponseConverter();

  // 解析接口方法的注解和注解值，获取请求报文中的请求方法、附加请求头、请求地址、contentType。
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }


  if (httpMethod == null) {
    throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
  }

  // 规范检查，不含有请求体时，不应该为 @Multipart 注解或 @FormEncoded 注解。
  if (!hasBody) {
    if (isMultipart) {
      throw methodError(
          "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
    }
    if (isFormEncoded) {
      throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
          + "request body (e.g., @POST).");
    }
  }

  // 获取方法上的所有有注解的参数数量。
  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0; p < parameterCount; p++) {
    // 获取第 p 个参数的参数类型。
    Type parameterType = parameterTypes[p];
    // 参数类型校验。
    if (Utils.hasUnresolvableType(parameterType)) {
      throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
          parameterType);
    }

    // 获取第 p 个参数上的所有注解。
    Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    if (parameterAnnotations == null) {
      throw parameterError(p, "No Retrofit annotation found.");
    }
    // 通过解析参数类型和注解构建适合的 ParameterHandler<?> 对象实现类。
    // 后面会调用 parseParameter.apply() 方法提供给 RequestBuilder 数据从而构建 Okhttp 的 Request。
    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }

  return new ServiceMethod<>(this);
}
```

#### 4.2.1 createCallAdapter()

```java
final class ServiceMethod<R, T> {
    private CallAdapter<T, R> createCallAdapter() {
      // 获取该方法的返回类型对应的 Type 对象。
      Type returnType = method.getGenericReturnType();
      // 如果有无法辨别的 Type 抛出异常。
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      // 不支持无意义的返回类型 Void。
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      // 获取该方法上的所有注解。
      Annotation[] annotations = method.getAnnotations();
      try {
        // 通过返回类型 Type 和方法上所有注解 annotations 遍历获取一个可用的的 CallAdapter。
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) {
        // 没有一个可用的 CallAdapter 则抛出异常。
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
}
```

```java
public final class Retrofit {

  public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    // 从 skipPast 所在 callAdapterFactories 位置的下一个开始遍历，即跳过该 CallAdapter.Factory。
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      // 遍历 CallAdapter.Factory 是否有可用的 CallAdapter。
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    
    // 没有可用的 callAdapter，则抛出异常说明。
    StringBuilder builder = new StringBuilder("Could not locate call adapter for ")
        .append(returnType)
        .append(".\n");
    if (skipPast != null) {
      builder.append("  Skipped:");
      for (int i = 0; i < start; i++) {
        builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
      }
      builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
  }
}
```

#### 4.2.2 createResponseConverter()

```java
final class ServiceMethod<R, T> {

  private Converter<ResponseBody, T> createResponseConverter() {
    Annotation[] annotations = method.getAnnotations();
    try {
      // responseType 并不一定为方法返回类型，而是响应报文中的 ResponseBody 想转化的类型。
      return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw methodError(e, "Unable to create converter for %s", responseType);
    }
  }
}
```

```java
public final class Retrofit {
  public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
  }

  /**
   * Returns a {@link Converter} for {@link ResponseBody} to {@code type} from the available
   * {@linkplain #converterFactories() factories} except {@code skipPast}.
   *
   * @throws IllegalArgumentException if no converter available for {@code type}.
   */
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }

    // 如果没有合适的转换器则抛异常（没贴代码）。
  }
}
```

#### 4.2.3 parseParameter()

```java
private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      // 遍历参数上所有注解
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        // 取最后一个有效的注解结果。
        result = annotationAction;
      }

      // 没有一个有效的注解解析，则抛异常。
      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }
```

这里抽取 parseParameterAnnotation() 中的 @FieldMap 解析过程解读。

```java
private ParameterHandler<?> parseParameterAnnotation(
     int p, Type type, Annotation[] annotations, Annotation annotation) {
       // p 为方法第 p 个参数；type 为变量类型；annotations 为该参数上所有注解; annotation 为 annotations[p]。
       // 这里假定 type 为 Map<String,String>。
       if (annotation instanceof FieldMap) {
         // 没有添加 @FormEncoded 抛异常。
        if (!isFormEncoded) {
          throw parameterError(p, "@FieldMap parameters can only be used with form encoding.");
        }
        // 在上面假定的前提下，此时 rawParameterType 为 Map。
        Class<?> rawParameterType = Utils.getRawType(type);
        // 如果 rawParameterType 不是 Map 的实现类，则抛出异常。
        if (!Map.class.isAssignableFrom(rawParameterType)) {
          throw parameterError(p, "@FieldMap parameter type must be Map.");
        }
        Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
        // 如果 Map 没有声明具体的泛型类型，则抛出异常。
        if (!(mapType instanceof ParameterizedType)) {
          throw parameterError(p,
              "Map must include generic types (e.g., Map<String, String>)");
        }
        
        ParameterizedType parameterizedType = (ParameterizedType) mapType;
        // 获取 Map<String,String> 第一个泛型，此处为 String。
        Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
        // 如果第一个泛型（key）不为 String 抛异常。
        if (String.class != keyType) {
          throw parameterError(p, "@FieldMap keys must be of type String: " + keyType);
        }
        // 获取第二个泛型（value），此处为 String。
        Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
        // 将 valueType 转换为 String，一般默认使用的是 ToStringConverter 转换器。
        Converter<?, String> valueConverter =
            retrofit.stringConverter(valueType, annotations);
        
        // 标记使用了普通表单。
        gotField = true;
        
        return new ParameterHandler.FieldMap<>(valueConverter, ((FieldMap) annotation).encoded());
      }
}
```

```java
static final class FieldMap<T> extends ParameterHandler<Map<String, T>> {
  private final Converter<T, String> valueConverter;
  private final boolean encoded;

  FieldMap(Converter<T, String> valueConverter, boolean encoded) {
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }

  @Override void apply(RequestBuilder builder, @Nullable Map<String, T> value)
      throws IOException {
    if (value == null) {
      throw new IllegalArgumentException("Field map was null.");
    }

    // 遍历 value 通过 valueConverter 解析得到 T，并新增 Field 添加给 RequestBuilder。
    for (Map.Entry<String, T> entry : value.entrySet()) {
      String entryKey = entry.getKey();
      if (entryKey == null) {
        throw new IllegalArgumentException("Field map contained null key.");
      }
      T entryValue = entry.getValue();
      if (entryValue == null) {
        throw new IllegalArgumentException(
            "Field map contained null value for key '" + entryKey + "'.");
      }

      String fieldEntry = valueConverter.convert(entryValue);
      if (fieldEntry == null) {
        throw new IllegalArgumentException("Field map value '"
            + entryValue
            + "' converted to null by "
            + valueConverter.getClass().getName()
            + " for key '"
            + entryKey
            + "'.");
      }
      
      builder.addFormField(entryKey, fieldEntry, encoded);
    }
  }
}
```

## 五、CallAdapter

```java
public interface CallAdapter<R, T> {

  // responseType 为最终想要的响应体类型。
  // 例如对于 RxJava2CallAdapter 中，假设接口方法返回值为 Observable<User>，则 responseType 为 User。
  Type responseType();

  // 将 Call<R> 适配成 T。
  T adapt(Call<R> call);

  abstract class Factory {
 
   /**
    * 提供方法返回值 Type 和 Annotation 让 CallAdapter 去确定自己能够处理的 Type 或 Annotation，返回 null 则表示无法处理。
    */
    public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    /**
     * 获取该类型中第 index 个泛型的实际类型。
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * 获取声明泛型的类或者接口。
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

### 5.1 RxJava2CallAdapterFactory

以 RxJava2CallAdapterFactory 为例看下对 CallAdapter.Factory 的实现。

```java
public final class RxJava2CallAdapterFactory extends CallAdapter.Factory {

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    // 此处假设 returnType 由接口方法 Method.getGenericReturnType() 得到类型为 Observable<User>。
    // 则 rawType 即为 Observable.class。
    Class<?> rawType = getRawType(returnType);

    // Completable 是 RxJava 的一种被观察者形式。
    if (rawType == Completable.class) {
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }

    // 如果 rawType 不为 RxJava 以下四种被观察者，返回 null 表示无法适配为类型 rawType。
    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

    boolean isResult = false;
    boolean isBody = false;
    Type responseType;
    // 泛型参数化的对象，自动实现 ParameterizedType 接口（即申明泛型为具体类）。
    // 如果返回类型 returnType 没有使用泛型，抛异常。
    if (!(returnType instanceof ParameterizedType)) {
      String name = isFlowable ? "Flowable"
          : isSingle ? "Single"
          : isMaybe ? "Maybe" : "Observable";
      throw new IllegalStateException(name + " return type must be parameterized"
          + " as " + name + "<Foo> or " + name + "<? extends Foo>");
    }

    // 获取第一个泛型类型，以 Observable<User> 为例，则此处 observableType 为 User。
    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    // 此时 rawObservableType 也为 User。
    // 若假设 observableType 为 Observable<Response<User>> 则 rawObservableType 为 Response<User>。
    Class<?> rawObservableType = getRawType(observableType);

    if (rawObservableType == Response.class) {
      //如果没有申明泛型类型，抛异常。
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Response must be parameterized"
            + " as Response<Foo> or Response<? extends Foo>");
      }
      // 拿到 Response<T> 第一个泛型类型，也就是 T。
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Result must be parameterized"
            + " as Result<Foo> or Result<? extends Foo>");
      }
      // 拿到 Result<T> 第一个泛型类型，也就是 T。
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      // responseType 则也为 User。
      responseType = observableType;
      isBody = true;
    }
    // 构建 RxJava2CallAdapter 并返回。
    return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
        isSingle, isMaybe, false);
  }
}
```

## 六、Converter

```java
public interface Converter<F, T> {
  
  T convert(F value) throws IOException;

  abstract class Factory {
    /**
     * 将 ResponseBody 转换为其它类型对象，如果该工厂无法处理该转换则返回 null。
     * 例如将 ResponseBody 转化为一个 Bean 对象。
     */
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(Type type,
        Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    /**
     * 将一种类型对象转换为 RequestBody, 如果该工厂无法处理该转换则返回 null。
     * 例如将 bean 对象转化为一个 ResponseBody。
     */
    public @Nullable Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    /**
     * 将一个类型对象转换为 String。
     */
    public @Nullable Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }

  }
}
```

### 6.1 BuiltInConverters

BuiltInConverters 是 Retrofit 默认提供的 Converter.Factory（数据转换工厂）。

```java
final class BuiltInConverters extends Converter.Factory {
  
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    if (type == ResponseBody.class) {
      // 方法注解 annotations 是否包含@Streaming 注解，
      // @Streaming 的一般用于大文件的传输，
      // 不用 Streaming 的时候，把整个网络响应的数据全部加载完，然后返回加载后的数据，
      // 并生成了新的 ResponseBody，避免服务器再次传输数据，但会大量占用客户端的内存。
      // 使用 Streaming 则是用于边接收数据边处理（不断更新 Response）。
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    if (RequestBody.class.isAssignableFrom(Utils.getRawType(type))) {
      return RequestBodyConverter.INSTANCE;
    }
    return null;
  }

  static final class VoidResponseBodyConverter implements Converter<ResponseBody, Void> {
    static final VoidResponseBodyConverter INSTANCE = new VoidResponseBodyConverter();

    @Override public Void convert(ResponseBody value) {
      value.close();
      return null;
    }
  }

  static final class RequestBodyConverter implements Converter<RequestBody, RequestBody> {
    static final RequestBodyConverter INSTANCE = new RequestBodyConverter();

    @Override public RequestBody convert(RequestBody value) {
      return value;
    }
  }

  static final class StreamingResponseBodyConverter
      implements Converter<ResponseBody, ResponseBody> {
    static final StreamingResponseBodyConverter INSTANCE = new StreamingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) {
      return value;
    }
  }

  static final class BufferingResponseBodyConverter
      implements Converter<ResponseBody, ResponseBody> {
    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) throws IOException {
      try {
        // Buffer the entire body to avoid future I/O.
        return Utils.buffer(value);
      } finally {
        value.close();
      }
    }
  }
  
  // 该转换器在 Retrofit 主要用于将方法的参数转换为 String 类型。
  static final class ToStringConverter implements Converter<Object, String> {
    static final ToStringConverter INSTANCE = new ToStringConverter();

    @Override public String convert(Object value) {
      return value.toString();
    }
  }
}
```

## 七、OkHttpCall

对于 OkHttpCall 的 enqueue()、execute() 不进行逐步分析，这两个方法最重要的是 createRawCall() 的调用，即 okhttp3.Call 的创建。

```java
final class OkHttpCall<T> implements Call<T> {
  
  private final ServiceMethod<T, ?> serviceMethod;
  private final @Nullable Object[] args;
 
  @GuardedBy("this")
  private @Nullable okhttp3.Call rawCall;

  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      // 同一个请求不能多次使用，否则抛异常。
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else if (creationFailure instanceof RuntimeException) {
          throw (RuntimeException) creationFailure;
        } else {
          throw (Error) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          // 构建 okhttp3.Call。
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); 
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }
    // 解析 OkHttp 的 Response。
    return parseResponse(call.execute());
  }
}
```
### 7.1 createRawCall()

```java
private okhttp3.Call createRawCall() throws IOException {
  // args 为方法参数实际的传参，此处的作用就是为 RequestBuilder 成员变量赋值并构造 okhttp3.Call。
  okhttp3.Call call = serviceMethod.toCall(args);
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```
serviceMethod.toCall()：

```java
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
  
  // 构建 RequestBuilder 对象，用于构建请求报文。
  RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
      contentType, hasBody, isFormEncoded, isMultipart);

  @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
  ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

  int argumentCount = args != null ? args.length : 0;
  if (argumentCount != handlers.length) {
    throw new IllegalArgumentException("Argument count (" + argumentCount
        + ") doesn't match expected count (" + handlers.length + ")");
  }
  // 调用 ParameterHandler.apply() 从而提供给 RequestBuilder 请求报文的数据。
  for (int p = 0; p < argumentCount; p++) {
    handlers[p].apply(requestBuilder, args[p]);
  }

  // 构建 OkHttp 的 Call 对象，之后便是调用 OkHttp 的 API 了。
  return callFactory.newCall(requestBuilder.build());
}
```

### 7.2 parseResponse()

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // 构建一个新的 ResponseBody 用于返回给调用者，
  // 移除了请求体的数据源，调用方如果尝试拿便会抛异常。
  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();

  int code = rawResponse.code();
  // 2XX 表示成功响应。
  if (code < 200 || code >= 300) {
    try {
      // Buffer the entire body to avoid future I/O.
      ResponseBody bufferedBody = Utils.buffer(rawBody);
      return Response.error(bufferedBody, rawResponse);
    } finally {
      rawBody.close();
    }
  }

  // 成功响应，但不包含响应体。
  if (code == 204 || code == 205) {
    rawBody.close();
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    // 通过数据转换器将响应体转换成我们想要的 responseType(例如 Call<User> 则 T 为 User)。
    T body = serviceMethod.toResponse(catchingBody);
    // 构建 Retrofit 的 response。
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```

## 八、serviceMethod.adapt(okHttpCall)

```java
  T adapt(Call<R> call) {
    return callAdapter.adapt(call);
  }
```

这里直接以 RxJava2CallAdapter 为例看一下如何将 Call<R> 转换为 RxJava 的形式。

```java
final class RxJava2CallAdapter<R> implements CallAdapter<R, Object> {
  private final Type responseType;
  private final @Nullable Scheduler scheduler;
  private final boolean isAsync;
  private final boolean isResult;
  private final boolean isBody;
  private final boolean isFlowable;
  private final boolean isSingle;
  private final boolean isMaybe;
  private final boolean isCompletable;


  @Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

   
    Observable<?> observable;
    // 如果接口方法返回类型为 Observable<Result<T>>。
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } 
    // 如果接口方法返回类型即不是 Observable<Result<T>>，也不是 Observable<Response<T>>，而是 Observable<Bean> 类型（最常使用）。
    else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } 
    // 如果接口方法返回类型为 Observable<Response<T>>。
    else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return observable;
  }
}
```

我们简单看一下如何将 RxJava 插入同步的网络请求。

```java
final class CallExecuteObservable<T> extends Observable<Response<T>> {
  
  private final Call<T> originalCall;

  CallExecuteObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  // 该方法在调用 subscribe() 方法时被调用。
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallDisposable disposable = new CallDisposable(call);
    observer.onSubscribe(disposable);

    boolean terminated = false;
    try {
      // 同步执行 OkHttp 的网络请求，并拿到响应 Response。
      Response<T> response = call.execute();
      if (!disposable.isDisposed()) {
        //返回 response 给 observer 的实现类调用。
        observer.onNext(response);
      }
      if (!disposable.isDisposed()) {
        terminated = true;
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!disposable.isDisposed()) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
}
```

### 8.1 BodyObservable

```java
final class BodyObservable<T> extends Observable<T> {
  
  private final Observable<Response<T>> upstream;

  // upstream 为被观察者 CallExecuteObservable。
  BodyObservable(Observable<Response<T>> upstream) {
    this.upstream = upstream;
  }

  // 该方法在调用 subscribe() 方法时被调用。
  @Override protected void subscribeActual(Observer<? super T> observer) {
    // 传入外部调用的观察者 observer 作为变量实例化 BodyObservable 并产生订阅关系。
    upstream.subscribe(new BodyObserver<T>(observer));
  }

  private static class BodyObserver<R> implements Observer<Response<R>> {
    private final Observer<? super R> observer;
    private boolean terminated;

    BodyObserver(Observer<? super R> observer) {
      this.observer = observer;
    }

    @Override public void onNext(Response<R> response) {
      // 主要代码是这一行，对于 BodyObserver 来说，观察到的是 response，
      // 对于传入的观察者 observer 来说，如果网络请求成功，观察到的是 response.body()。
      if (response.isSuccessful()) {
        observer.onNext(response.body());
      } else {
        terminated = true;
        Throwable t = new HttpException(response);
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
}
```

