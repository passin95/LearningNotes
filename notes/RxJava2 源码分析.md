
## 前言

本文源码分析基于 [RxJava](https://github.com/ReactiveX/RxJava) 2.2.1版本，主要从易到难一步步模拟实际操作的方式分析 RxJava 的链式调用结构以及线程的切换原理。

为了更清晰的分析源码，以下的代码示例中的被观察者将使用 Single，而不是 Observable。Single和Observable区别在于Single只会向观察者发送一个数据（例如在网络请求使用），而 Observable 则可以依次向观察者发送多个数据。




## 初探 RxJava

首先看一下一个最简单的链式订阅过程。

```java
// 被观察者的实例化
Single<Integer> single = Single.just(1);

// 观察者的实例化，SingleObserver为一个接口，定义了观察者的回调方法规范。
SingleObserver<Integer> observer = new SingleObserver<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        // 开始订阅
    }

    @Override
    public void onSuccess(Integer integer) {
        // 成功拿到数据
        Timber.w("2222222222");
    }

    @Override
    public void onError(Throwable e) {
        // 异常
    }
};

// 从语意上看被观察者 “订阅” 观察者，实际是观察者订阅被观察者（下面也将如此描述），
// 因此多次调用single.subscribe()是互不影响的。
single.subscribe(observer);

```

### Single.just()

被观察者的构建过程，基本上所有的被观察者的构建的Api源码都是相似的逻辑。

```java
public static <T> Single<T> just(final T item) {
    ObjectHelper.requireNonNull(item, "value is null");
    // 对全局Single被观察者进行转换。
    return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
}

// 对传入的 Single（被观察者）做统一转换
public static <T> Single<T> onAssembly(@NonNull Single<T> source) {
    Function<? super Single, ? extends Single> f = onSingleAssembly;

    // 将Single类或Single的父类或Single实现的接口类(SingleSource），转换成继承 Single 的被观察者（例如将所有转成SingleJust）
    if (f != null) {
        return apply(f, source);
    }
    // 默认情况下 f 为null，所以一般返回构建的被观察者。
    return source;
}
```

因此Single.just()的重点便是被观察者SingleJust的实例化。

```java
public final class SingleJust<T> extends Single<T> {

    final T value;

    // 很简单，将 value 保存下来，并生成一个被观察者。
    public SingleJust(T value) {
        this.value = value;
    }

    // 该方法为抽象类 Single 的抽象方法，该方法会在订阅时被调用，也是订阅的核心方法（下面会说明何时被调用）。
    // observer为订阅该被观察者的观察者，在订阅时，会传入observer，由相应的Single子类（这里是SingleJust）控制observer不同方法的执行时机。
    @Override
    protected void subscribeActual(SingleObserver<? super T> observer) {
        observer.onSubscribe(Disposables.disposed());
        observer.onSuccess(value);
    }

}
````

### single.subscribe(observer)

观察者订阅被观察者

```java
public final void subscribe(SingleObserver<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "subscriber is null");

    // 对全局观察者进行转换，默认不转换。
    observer = RxJavaPlugins.onSubscribe(this, observer);

    ObjectHelper.requireNonNull(observer, "subscriber returned by the RxJavaPlugins hook is null");

    try {
        // 我们可以看到订阅过程(执行Single.subscribe(observer)方法)的实质是传入observer执行subscribeActual(observer)方法。
        subscribeActual(observer);
    } catch (NullPointerException ex) {
        throw ex;
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        NullPointerException npe = new NullPointerException("subscribeActual failed");
        npe.initCause(ex);
        throw npe;
    }
}
```

### 小结

- 我们可以发现被观察者 single 的实例化以及观察者 observer 的实例化都处于**当前所在线程**(重点，有利于RxJava后续的理解)。
- 当被观察者 single **被订阅时（调用single.subscribe(observer)）**，执行的核心方法为被观察者 single 的 subscribeActual(observer) 方法，通过传入观察者 observer 的对象去控制观察者 observer 的方法执行。

<img src="../pictures//RxJavaPic1.png" /> 


## 再探 RxJava

我们在上一个Demo的基础上加了线程的切换以及Map操作符和FlatMap操作符的使用。

本小节，皆已以下代码作为Demo。

```java
// 实例化观察者
SingleObserver<String> observer = new SingleObserver<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        // 开始订阅
    }

    @Override
    public void onSuccess(String integer) {
        // 成功拿到数据
    }

    @Override
    public void onError(Throwable e) {
        // 异常
    }
};

// 实例化被观察者
Single<String> single = Single.just(1)
        .subscribeOn(Schedulers.io())
        .flatMap(new Function<Integer, SingleSource<Integer>>() {
            @Override
            public SingleSource<Integer> apply(Integer integer) throws Exception {
                return Single.just(integer + 1);
            }
        })
        .observeOn(AndroidSchedulers.mainThread())
        .map(new Function<Integer, String>() {
            @Override
            public String apply(Integer integer) throws Exception {
                return integer + "";
            }
        });
        
// 订阅
single.subscribe(observer);
````

### RxJava 链式调用结构的本质

首先我们看被观察者的实例化，这里拿single.subscribeOn()方法举例（所有的操作符都基本类似），我们会发现，链式被观察者的实例化过程的本质是**以当前被观察者对象作为新被观察者构造方法的参数生成一个新的观察者并返回**，也就是说被观察者的实例化过程是一个个新的观察者对象生成的过程，且实例化的过程所处线程为当前线程，没有涉及到任何操作符的接口实现或线程切换（在订阅的时候才执行）。

```java
public final Single<T> subscribeOn(final Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new SingleSubscribeOn<T>(this, scheduler));
}
```

从上面我们可以知道，此时的single为最后创建（调用map操作符后）的被观察者，即此时的single的对象类型为SingleMap,接着我们看一下 single.subscribe(observer) 实际做了什么。

```java
public final class SingleMap<T, R> extends Single<R> {
   final SingleSource<? extends T> source;

   public SingleMap(SingleSource<? extends T> source, Function<? super T, ? extends R> mapper) {
        this.source = source;
        this.mapper = mapper;
    }
   
   @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
        // 从构造函数我们可以知道 source 为上一个被观察者(实例化SingleMap的参数)，
        // 传入t（订阅single的观察者）作为参数并实例化一个新的观察者，
        // 最后用新的观察者订阅 source。
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }
```

从上面可以看到single.subscribe(observer)的本质为实例化一个新的观察者订阅上一个被观察者。
然后我们看一下MapSingleObserver拿到 observer（single的观察者）做了什么。

```java
static final class MapSingleObserver<T, R> implements SingleObserver<T> {

    final SingleObserver<? super R> t;

    final Function<? super T, ? extends R> mapper;

    MapSingleObserver(SingleObserver<? super R> t, Function<? super T, ? extends R> mapper) {
        this.t = t;
        this.mapper = mapper;
    }

    // 我们可以看到当MapSingleObserver.onXXX()被调用时，最终都会调用传进来的t（observer）的onXXX()方法。
    @Override
    public void onSubscribe(Disposable d) {
        t.onSubscribe(d);
    }

    @Override
    public void onSuccess(T value) {
        // 此处暂时省略其它代码

        t.onSuccess(v);
    }

    @Override
    public void onError(Throwable e) {
        t.onError(e);
    }
}
```

而其它的链式调用也是同样的过程，直至在某一个被观察者中subscribeActual()方法给观察者的onxxx()方法传入具体的数据。在本例中的数据来源为Single.just(1)，也就是SingleJust类。

```java
public final class SingleJust<T> extends Single<T> {

    final T value;

    public SingleJust(T value) {
        this.value = value;
    }

    @Override
    protected void subscribeActual(SingleObserver<? super T> s) {
        // s 为SingleJust的观察者，调用 s 的接口方法去通知观察者，
        // 之后 s 又会通知下一个观察者，并依次向下通知
        s.onSubscribe(Disposables.disposed());
        s.onSuccess(value);
    }

}
```

整个 RxJava 链式调用结构便如下图所示：

<img src="../pictures//RxJavaPic2.png" /> 

**下面的分析将继续沿用该图的变量名以方便解读**

### Map和FlapMap

在看 Map 操作符和FlapMap 操作符的核心源码之前，先看一下Function接口，该接口定义了一个方法，用于将类型T转成R。Map和FlapMap的本质也是让开发者自己实现该接口，并对observer的value进行数据转换再往下一个observer传递。

```java
public interface Function<T, R> {
    /**
     * Apply some calculation to the input value and return some other value.
     * @param t the input value
     * @return the output value
     * @throws Exception on error
     */
    R apply(@NonNull T t) throws Exception;
}
```

#### map

```java
public final class SingleMap<T, R> extends Single<R> {
    static final class MapSingleObserver<T, R> implements SingleObserver<T> {

        final SingleObserver<? super R> t;

        final Function<? super T, ? extends R> mapper;

        MapSingleObserver(SingleObserver<? super R> t, Function<? super T, ? extends R> mapper) {
            this.t = t;
            this.mapper = mapper;
        }

        @Override
        public void onSuccess(T value) {
            // T 为自身（observer1） 想要的数据类型，
            // R 为下一个 observer（observer）想要的数据类型
            R v;
            try {
                // 我们在实例化该被观察者时会实现 mapper 接口，此时调用该实现转换数据类型。
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(e);
                return;
            }

            t.onSuccess(v);
        }

    }
```

#### flatMap

```java
public final class SingleFlatMap<T, R> extends Single<R> {
  static final class SingleFlatMapCallback<T, R>
    extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable {

        @Override
        public void onSuccess(T value) {
            SingleSource<? extends R> o;

            try {
                // 此处和Map操作符基本类似，只是限制了mapper的泛型上下限
                // 该接口实现中得到了一个新的被观察者（链），在 Demo 为Single.just(integer + 1)
                o = ObjectHelper.requireNonNull(mapper.apply(value), "The single returned by the mapper is null");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                actual.onError(e);
                return;
            }

            if (!isDisposed()) {
                // 订阅后，将是一个新的链式调用过程，该过程走完才接上原来的调用链
                // 此时FlatMapSingleObserver类似结构图中的observer
                // 在 FlatMapSingleObserver 的 onxxx() 方法中续上 actual（observer2）继续向下传递结果。
                o.subscribe(new FlatMapSingleObserver<R>(this, actual));
            }
        }

        static final class FlatMapSingleObserver<R> implements SingleObserver<R> {

            final SingleObserver<? super R> actual;

            FlatMapSingleObserver(AtomicReference<Disposable> parent, SingleObserver<? super R> actual) {
                this.parent = parent;
                this.actual = actual;
            }

            @Override
            public void onSuccess(final R value) {
                actual.onSuccess(value);
            }

        }
    }
}
```

### 线程切换

#### subscribeOn

```java
public final class SingleSubscribeOn<T> extends Single<T> {
    final SingleSource<? extends T> source;

    final Scheduler scheduler;

    @Override
    protected void subscribeActual(final SingleObserver<? super T> s) {
        // SubscribeOnObserver既是一个观察者，也是一个 Runnable。
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s, source);
        // 此处注意，SingleSubscribeOn被订阅时，便已经调用该方法。
        s.onSubscribe(parent);

        // scheduler对不同线程池的使用做了一层接口封装,作为线程调度器使用。
        // 调用scheduler.scheduleDirect()方法将在scheduler所对应的线程池中执行SubscribeOnObserver的run()方法，
        // run()方法执行的是source（single4）的订阅过程，
        // 也就是说，若之后不再切换线程，从source（single4）的subscribeActual()开始之后的所有代码都在该线程池执行。
        Disposable f = scheduler.scheduleDirect(parent);

        parent.task.replace(f);

    }

    static final class SubscribeOnObserver<T>
    extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {

        final SingleObserver<? super T> actual;

        final SequentialDisposable task;

        final SingleSource<? extends T> source;

        SubscribeOnObserver(SingleObserver<? super T> actual, SingleSource<? extends T> source) {
            this.actual = actual;
            this.source = source;
            this.task = new SequentialDisposable();
        }

        @Override
        public void onSuccess(T value) {
            actual.onSuccess(value);
        }

        @Override
        public void run() {
            source.subscribe(this);
        }
    }

}
```

#### observeOn

```java
public final class SingleObserveOn<T> extends Single<T> {

    final SingleSource<T> source;

    final Scheduler scheduler;

    @Override
    protected void subscribeActual(final SingleObserver<? super T> s) {
        // 该方法中只有正常的订阅过程，不涉及到线程切换
        source.subscribe(new ObserveOnSingleObserver<T>(s, scheduler));
    }

    static final class ObserveOnSingleObserver<T> extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {

        final SingleObserver<? super T> actual;

        final Scheduler scheduler;


        @Override
        public void onSuccess(T value) {
            this.value = value;
            // 我们可以看到在调用 onxxx()之后才执行线程切换并执行run()方法
            Disposable d = scheduler.scheduleDirect(this);
        }

        @Override
        public void run() {
            Throwable ex = error;
            if (ex != null) {
                actual.onError(ex);
            } else {
                actual.onSuccess(value);
            }
        }
    }
}
```

#### 小结

以Demo的代码为例，从single.subcriber(observer)（观察者订阅被观察者）开始实际的线程切换如下图所示：

- 灰色 - 当前默认线程
- 绿色 - subscribeOn(Scheduler)，Demo为IO线程
- 蓝色 - observeOn(Scheduler)，Demo为main线程

<img src="../pictures//RxJavaPic3.png" /> 

### 终看 RxJava




