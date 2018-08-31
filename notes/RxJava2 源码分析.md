
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

我们在上一个Demo的基础上加了线程的切换以及Map操作符和flatMap操作符的使用。

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

首先我们看被观察者的实例化，这里拿single.subscribeOn()方法举例（所有的操作符都一样），我们会发现，链式被观察者的实例化过程的本质是**以当前被观察者对象作为新被观察者构造方法的参数生成一个新的观察者并返回**，也就是说被观察者的实例化过程是一个个新的观察者对象生成的过程，且实例化的过程所处线程为当前线程，没有涉及到任何操作符的接口实现或线程切换（在订阅的时候才执行）。

```java
public final Single<T> subscribeOn(final Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new SingleSubscribeOn<T>(this, scheduler));
}
```

从上面我们可以知道，此时的single为最后创建（调用map操作符后）的被观察者，即此时的single的对象类型为SingleMap,接着我们




### 






