# Retrofit 源码解析

<!-- TOC -->

- [Retrofit 源码解析](#retrofit-源码解析)
    - [一、Retrofit 的基本使用](#一retrofit-的基本使用)
    - [二、Retrofit 的构建](#二retrofit-的构建)
        - [Retrofit 的成员变量](#retrofit-的成员变量)
        - [Retrofit.Builder](#retrofitbuilder)
    - [三、Retrofit.create()](#三retrofitcreate)
        - [Retrofit.loadServiceMethod()](#retrofitloadservicemethod)
    - [四、ServiceMethod 的构建](#四servicemethod-的构建)
        - [ServiceMethod 的成员变量](#servicemethod-的成员变量)
        - [ServiceMethod.Builder](#servicemethodbuilder)
            - [createCallAdapter()](#createcalladapter)
            - [createResponseConverter()](#createresponseconverter)
            - [parseParameter()](#parseparameter)
    - [五、CallAdapter](#五calladapter)
        - [RxJava2CallAdapterFactory](#rxjava2calladapterfactory)
    - [六、Converter](#六converter)
        - [BuiltInConverters](#builtinconverters)
    - [七、OkHttpCall](#七okhttpcall)
        - [createRawCall()](#createrawcall)
            - [serviceMethod.toCall()](#servicemethodtocall)
        - [parseResponse()](#parseresponse)
    - [八、serviceMethod.adapt(okHttpCall)](#八servicemethodadaptokhttpcall)
        - [BodyObservable](#bodyobservable)

<!-- /TOC -->

## 一、Retrofit 的基本使用

本文将直接结合 Gson 以及 RxJava 的使用以代码+注解的方式按运行流程解读源码，非粗读源码，而是对主流程进行深度剖析。
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

### Retrofit 的成员变量

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

###  Retrofit.Builder

```java
public Retrofit build() {
    if (baseUrl == null) {
      throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    // 默认使用 okhttp 提供的 OkHttpClient。
    if (callFactory == null) {
      callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    // 默认根据所在的编译环境去取决于使用哪种具体 platform 的实现类
    if (callbackExecutor == null) {
      callbackExecutor = platform.defaultCallbackExecutor();
    }

   
    // 网络请求适配器工厂集合存储顺序：自定义 1 适配器工厂、自定义 2 适配器工厂...android 环境下默认适配器工厂（ExecutorCallAdapterFactory）
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    
    callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // 数据转换器工厂
    List<Converter.Factory> converterFactories =
        new ArrayList<>(1 + this.converterFactories.size());
        
    // 数据转换器工厂集合存储的是：默认数据转换器工厂（ BuiltInConverters）、自定义 1 数据转换器工厂（GsonConverterFactory）、自定义 2 数据转换器工厂....
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);

   // 构建 Retrofit
    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
  }
```

## 三、Retrofit.create()

```java
public <T> T create(final Class<T> service) {
    // 检查该类是否是接口以及该类是否有 extends 其它接口
    Utils.validateServiceInterface(service);
    // 是否在调用 create(Class) 时检测接口中所有方法是否符合规范，而不是在调用方法才检测
    // 适合在开发、测试时使用 (ServiceMethod 对象构建的时候会进行一系列的检查)
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    // Java 动态代理，Proxy.newProxyInstance 会把参数 service 类中的所有方法都实现 InvocationHandler 接口中的 invoke() 方法，
    // 在调用该类的方法时，会回调到 invoke() 方法
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // 如果该方法是 Object.class 原本就存在的方法，则直接调用原本的方法
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 针对 java8 做的兼容，针对非 android 环境的情况，具体可看 Platform 类
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 通过解析网络请求接口方法的参数、返回值和注解类型获取网络请求所需的数据。
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            // 提供 OkHttp 进行网络请求所需的数据并将 Retrofit 和 OkHttp 连接到一起
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 返回想要的数据适配结果
            return serviceMethod.adapt(okHttpCall);
          }
        });
}
```

### Retrofit.loadServiceMethod()

该方法比较简单，使用了懒加载的思想，并对 ServiceMethod 进行了缓存。

```java
// Retrofit 的全局变量
private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?, ?> loadServiceMethod(Method method) {

  ServiceMethod<?, ?> result = serviceMethodCache.get(method);
  // 如果存在直接返回
  if (result != null) return result;
  // 对 serviceMethodCache 进行双重校验锁，防止多线程操作时可能重复构建
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

### ServiceMethod 的成员变量

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
  // 余下的描述请到 https://github.com/passin95/LearningNotes/blob/master/notes/HTTP.md 进行深入了解
  private final Headers headers;
  private final MediaType contentType;
  private final boolean hasBody;
  private final boolean isFormEncoded;
  private final boolean isMultipart;

  // ParameterHandler 是一个抽象类，定义了一个抽象方法，统一规范向 RequestBuilder 参数进行赋值。
  为不同注解对 RequestBuilder
  //ParameterHandler[] 保存着方法所有参数中不同注解是向 RequestBuilder 赋值的方式
  private final ParameterHandler<?>[] parameterHandlers;

}
```

### ServiceMethod.Builder

```java
public ServiceMethod build() {
  // 根据网络请求接口返回类型和方法注解（不一定用到，例如 RxjavaCallAdapter 便没用到），遍历获取到合适的网络请求适配器
  callAdapter = createCallAdapter();
  // responseType 为最终我们想要的结果，例如使用的 RxJava2CallAdapterFactory
  // 假设接口方法返回值为 Observable<User>,则 responseType 为 User
  responseType = callAdapter.responseType();
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  // 根据 responseType 和方法上注解，从 Retrofit 对象中获取可用的数据转换器 
  responseConverter = createResponseConverter();

  // 解析网络请求接口方法的注解和注解值,获取该次请求的 httpMethod、hasBody、contentType 等
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }


  if (httpMethod == null) {
    throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
  }

  // 规范检查 不含有请求体时，不应该为@Multipart 注解或@FormEncoded 注解，
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

  // 获取该方法的参数数量
  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0; p < parameterCount; p++) {
    // 获取第 p 个参数的参数类型
    Type parameterType = parameterTypes[p];
    if (Utils.hasUnresolvableType(parameterType)) {
      throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
          parameterType);
    }

    // 获取第 p 个参数的所有注解
    Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    if (parameterAnnotations == null) {
      throw parameterError(p, "No Retrofit annotation found.");
    }
    // 通过解析参数的类型和注解构建不同的 ParameterHandler<?> 对象实现类
    // 后面会调用 parseParameter.apply() 方法赋值给 RequestBuilder 构建网络请求
    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }

  return new ServiceMethod<>(this);
}
```

#### createCallAdapter()

```java
final class ServiceMethod<R, T> {
    private CallAdapter<T, R> createCallAdapter() {
      // 获取该方法的返回类型对应的 Type 对象
      Type returnType = method.getGenericReturnType();
      // 如果有无法辨别的 Type 抛出异常
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      // 如果返回类型为 Void，抛出异常
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      // 获取该方法上的所有注解
      Annotation[] annotations = method.getAnnotations();
      try {
        // 通过返回类型 Type 和方法所有注解 annotations 获取一个可用的的 CallAdapter
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) {
        // 没有一个可用的 CallAdapter 抛出异常
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

    // 从 skipPast 所在 callAdapterFactories 位置的下一个开始遍历，即跳过该 CallAdapter.Factory
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      // 遍历 CallAdapter.Factory 是否有可用的 CallAdapter
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    
    // 没有可用的 callAdapter,便并抛出异常说明
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


#### createResponseConverter()

```java
final class ServiceMethod<R, T> {

  private Converter<ResponseBody, T> createResponseConverter() {
    Annotation[] annotations = method.getAnnotations();
    try {
      // responseType 为非方法接口返回类型，而是响应报文中的 ResponseBody 想转化为的类型。
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

#### parseParameter()

```java
private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      // 遍历一个参数上所有注解
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        // 取最后一个有效的注解结果
        result = annotationAction;
      }

      // 没有一个有效的注解解析，则抛异常
      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }
```

这里抽取 parseParameterAnnotation() 方法中的一个注解解析过程解读。

```java
private ParameterHandler<?> parseParameterAnnotation(
     int p, Type type, Annotation[] annotations, Annotation annotation) {
       // p：方法第 p 个参数；type 为变量类型；annotations 为该参数上所有注解; annotation 为 annotations[p]
       // 这里假定 type 为一个 Map<String,String>
       if (annotation instanceof FieldMap) {
         // 没有添加@FormEncoded 抛异常
        if (!isFormEncoded) {
          throw parameterError(p, "@FieldMap parameters can only be used with form encoding.");
        }
        // 此时 rawParameterType 为 Map
        Class<?> rawParameterType = Utils.getRawType(type);
        // 如果 rawParameterType 不是 Map 的实现类
        if (!Map.class.isAssignableFrom(rawParameterType)) {
          throw parameterError(p, "@FieldMap parameter type must be Map.");
        }
        Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
        // 如果 Map 没有声明具体的泛型类型则抛异常
        if (!(mapType instanceof ParameterizedType)) {
          throw parameterError(p,
              "Map must include generic types (e.g., Map<String, String>)");
        }
        
        ParameterizedType parameterizedType = (ParameterizedType) mapType;
        // 获取 Map<String,String> 第一个泛型，此处为 String
        Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
        // 如果第一个泛型（key）不为 String 抛异常
        if (String.class != keyType) {
          throw parameterError(p, "@FieldMap keys must be of type String: " + keyType);
        }
        // 获取第二个泛型（value），此处为 String
        Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
        // 将 valueType 转换为 String，一般默认使用的是 ToStringConverter 转换器
        Converter<?, String> valueConverter =
            retrofit.stringConverter(valueType, annotations);
        
        // 标记使用了普通表单
        gotField = true;
        // 构建 ParameterHandler.FieldMap
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

    // 遍历 value 解析 T 并赋值给 RequestBuilder
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

  // 网络请求接口方法返回类型
  Type responseType();

  // 将 Call<R> 适配成 T
  T adapt(Call<R> call);


  // 工厂模式，一是对外隐藏 CallAdapter 的实例化，二是制定构建 CallAdapter 的规范
  abstract class Factory {
 
    // 提供 Type 和 Annotation 让 CallAdapter 去确定自己能够处理的 Type 或 Annotation，若返回 null 则表示无法处理
    public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    /**
     * Extract the upper bound of the generic parameter at {@code index} from {@code type}. For
     * example, index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
     * 获取该类型中第 index 个泛型的实际类型
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * Extract the raw class type from {@code type}. For example, the type representing
     * {@code List<? extends Runnable>} returns {@code List.class}.
     * 获取声明泛型的类或者接口
     */
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

### RxJava2CallAdapterFactory

接下来以 RxJava2CallAdapterFactory 为例对看下 CallAdapter.Factory 的实现。
```java

public final class RxJava2CallAdapterFactory extends CallAdapter.Factory {

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    // 此处假设 returnType 由接口方法 Method.getGenericReturnType() 得到 Observable<User>
    // 此处 rawType 即为 Observable.class
    Class<?> rawType = getRawType(returnType);

    // Completable 是 RxJava 的一种被观察者形式。
    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }

    // 如果 rawType 不为 RxJava 以下四种被观察者 返回 null 表示无法适配为当前 rawType
    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

    boolean isResult = false;
    boolean isBody = false;
    Type responseType;
    // 泛型参数化的对象，自动实现 ParameterizedType 接口（即申明泛型为具体类）
    // 如果返回类型 returnType 没有使用泛型，抛异常
    if (!(returnType instanceof ParameterizedType)) {
      String name = isFlowable ? "Flowable"
          : isSingle ? "Single"
          : isMaybe ? "Maybe" : "Observable";
      throw new IllegalStateException(name + " return type must be parameterized"
          + " as " + name + "<Foo> or " + name + "<? extends Foo>");
    }

    // 获取第一个泛型类型，以 Observable<User> 为例，则此处 observableType 为 User。
    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    // 此时 rawObservableType 也为 User
    // 若假设 observableType 为 Observable<Response<User>> 则 rawObservableType 为 Response<User>
    Class<?> rawObservableType = getRawType(observableType);

    if (rawObservableType == Response.class) {
      //如果没有申明泛型类型，抛异常
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Response must be parameterized"
            + " as Response<Foo> or Response<? extends Foo>");
      }
      // 拿到 Response<T> 第一个泛型类型，也就是 T
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Result must be parameterized"
            + " as Result<Foo> or Result<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      // responseType 则也为 User
      responseType = observableType;
      isBody = true;
    }
    // 构建 RxJava2CallAdapter 并返回
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
     * 将 ResponseBody 转换为其它类型对象，如果该工厂无法处理该转换则返回 null
     * 例如将 ResponseBody 转化为一个 Bean 对象
     */
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(Type type,
        Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    /**
     * 将一种类型对象转换为 RequestBody, 如果该工厂无法处理该转换则返回 null
     * 例如将 bean 对象转化为一个 ResponseBody
     */
    public @Nullable Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    /**
     * 将一个类型对象转换为 String
     */
    public @Nullable Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }

  }
}
```

### BuiltInConverters

BuiltInConverters 是 Retrofit 默认提供的 Converter.Factory。

```java
final class BuiltInConverters extends Converter.Factory {
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    if (type == ResponseBody.class) {
      // 方法注解 annotations 是否包含@Streaming 注解
      // @Streaming 的一般用于大文件的传输
      // 不用 Streaming 的时候，把整个网络响应的数据全部加载完，然后返回加载后的数据，并生成了新的 ResponseBody，避免服务器再次传输数据
      // 使用 Streaming 则是用于边接收数据边处理（不断更新Response）
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
  

  // 该转换器在 Retrofit 主要用于将网络接口方法的变量转换为 String 类型
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
      // 不能多次调用
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
          // 创建 okhttp3.Call
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
    // 解析 OkHttp 的 Response
    return parseResponse(call.execute());
  }
}
```
### createRawCall()

```java
  private okhttp3.Call createRawCall() throws IOException {
    // args 为方法参数实际的传参 此处的作用就是为 RequestBuilder 成员变量赋值并构造 okhttp3.Call
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
#### serviceMethod.toCall()

```java
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
  // 构建 RequestBuilder
  RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
      contentType, hasBody, isFormEncoded, isMultipart);

  @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
  ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

  int argumentCount = args != null ? args.length : 0;
  if (argumentCount != handlers.length) {
    throw new IllegalArgumentException("Argument count (" + argumentCount
        + ") doesn't match expected count (" + handlers.length + ")");
  }
  // 调用 ParameterHandler.apply() 为 RequestBuilder 赋值
  for (int p = 0; p < argumentCount; p++) {
    handlers[p].apply(requestBuilder, args[p]);
  }

  // 构建 OkHttp.Call
  return callFactory.newCall(requestBuilder.build());
}
```

### parseResponse()
```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // 构建一个一样的新的 ResponseBody(如果持有引用，可能出现服务器再次发送导致修改 ResponseBody)
  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();

  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    try {
      // Buffer the entire body to avoid future I/O.
      ResponseBody bufferedBody = Utils.buffer(rawBody);
      return Response.error(bufferedBody, rawResponse);
    } finally {
      rawBody.close();
    }
  }

  if (code == 204 || code == 205) {
    rawBody.close();
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    // 通过数据转换器转换成我们想要的 responseType(例如 Call<User> 则 T 为 User)
    T body = serviceMethod.toResponse(catchingBody);
    // 构建 Retrofit 的 response
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

这里直接以 RxJava2CallAdapter 看一下如何将 Call<R> 转换为 RxJava 的形式。

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
    // 如果接口方法返回类型为 Observable<Result<T>>
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } 
    // 如果接口方法返回类型即不是 Observable<Result<T>>，也不是 Observable<Response<T>>，而是 Observable<Bean> 类型（最常使用）。
    else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } 
    // 如果接口方法返回类型为 Observable<Response<T>>
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

我们简单看一下如何将 RxJava 插入同步的网络请求

```java
final class CallExecuteObservable<T> extends Observable<Response<T>> {
  private final Call<T> originalCall;

  CallExecuteObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  // 该方法在调用 subscribe() 方法时被调用
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallDisposable disposable = new CallDisposable(call);
    observer.onSubscribe(disposable);

    boolean terminated = false;
    try {
      Response<T> response = call.execute();
      if (!disposable.isDisposed()) {
        //返回response给observer的实现类调用。
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

### BodyObservable

```java
final class BodyObservable<T> extends Observable<T> {
  private final Observable<Response<T>> upstream;

  // upstream 为被观察者 CallExecuteObservable
  BodyObservable(Observable<Response<T>> upstream) {
    this.upstream = upstream;
  }

  // 该方法在调用 subscribe() 方法时被调用
  @Override protected void subscribeActual(Observer<? super T> observer) {
    // 传入外部调用的观察者 observer 作为变量实例化 BodyObservable 并产生订阅关系
    upstream.subscribe(new BodyObserver<T>(observer));
  }

  private static class BodyObserver<R> implements Observer<Response<R>> {
    private final Observer<? super R> observer;
    private boolean terminated;

    BodyObserver(Observer<? super R> observer) {
      this.observer = observer;
    }

    @Override public void onNext(Response<R> response) {
      // 主要代码是这一行，对于 BodyObserver 来说，观察到的是 response
      // 对于外部观察者 observer 来说，如果网络请求成功，得到的是 response.body()
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

