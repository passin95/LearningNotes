<!-- TOC -->

- [Handler 消息机制](#handler-消息机制)
  - [一、Java 篇](#一java-篇)
    - [1.1 架构图](#11-架构图)
    - [1.2 Looper](#12-looper)
      - [1.2.1 Looper.prepare()](#121-looperprepare)
      - [1.2.2 Looper.loop()](#122-looperloop)
      - [1.2.3 looper.quit()](#123-looperquit)
    - [1.3 Handler](#13-handler)
      - [1.3.1 构造函数和成员变量](#131-构造函数和成员变量)
      - [1.3.2 消息分发和处理](#132-消息分发和处理)
      - [1.3.3 消息发送](#133-消息发送)
      - [1.3.4 消息移除](#134-消息移除)
    - [1.4 MessageQueue](#14-messagequeue)
      - [1.4.1 构造函数](#141-构造函数)
      - [1.4.2 messageQueue.next()](#142-messagequeuenext)
      - [1.4.3 messageQueue.enqueueMessage()](#143-messagequeueenqueuemessage)
      - [1.4.4 messageQueue.removeMessages()](#144-messagequeueremovemessages)
      - [1.4.5 messageQueue.postSyncBarrier()](#145-messagequeuepostsyncbarrier)
      - [1.4.6 messageQueue.removeSyncBarrier()](#146-messagequeueremovesyncbarrier)
    - [1.5 Message](#15-message)
      - [1.5.1 成员变量](#151-成员变量)
      - [1.5.2 消息池](#152-消息池)
  - [二、Native 篇](#二native-篇)
    - [2.1 epoll 机制](#21-epoll-机制)
      - [2.1.1 多路复用](#211-多路复用)
      - [2.1.2 API](#212-api)
      - [2.1.3 eventfd](#213-eventfd)
    - [2.2 简介](#22-简介)
    - [2.3 MessageQueue.nativeInit()](#23-messagequeuenativeinit)
    - [2.4 MessageQueue.nativePollOnce()](#24-messagequeuenativepollonce)
    - [2.5 MessageQueue.nativeWake()](#25-messagequeuenativewake)

<!-- /TOC -->

# Handler 消息机制

在说 Handler 之前，先说一个本质概念：我们所有跑的代码都是在线程中执行的，并且一旦线程中的代码执行完毕，也就代表线程即将死亡，而各种系统是如何设计，能够让主线程一直运行呢（进程未退出的情况下）？答案就是死循环，而 Handler 机制就是围绕这个死循环做文章。

## 一、Java 篇

### 1.1 架构图

<img src="../pictures//Handler%20架构图.webp" width="700"/>

- Message：可以理解为数据类，包含了 Runable 接口。
- MessageQueue：向消息队列中加入消息(MessageQueue.enqueueMessage)和取走消息队列的消息(MessageQueue.next)；
- Handler：向消息队列发送、移除事件的拓展支持(Handler.sendMessage)和处理相应消息(Handler.dispatchMessage)，从设计来看，Handler 用于拓展功能以及提供开发者 API，而 MessageQueue 只管消息相关的操作并隐藏实现细节；
- Looper：不断循环执行(Looper.loop)，从 MessageQueue 中拿到新的消息并分发分发给目标处理者（Handler）。

### 1.2 Looper

Looper 的作用：不断循环(Looper.loop)，从 MessageQueue 中拿到新的消息并分发分发给目标处理者（Handler）。

本文以 Android 进程主线程为例看 Handler 机制：

```java
public class ActivityThread {

    static volatile Handler sMainThreadHandler;  // set once in main()

    /**
     * 进程启动后的入口
     */
    public static void main(String[] args) {
        ......
        // 1. 做一些 Handler 消息机制的初始化。
        Looper.prepareMainLooper();
        
        ......
        
        // 2. 启动循环。
        Looper.loop();
    }
    
}
```

#### 1.2.1 Looper.prepare()

Looper.prepare() 主要做的事情其实就 2 件：

- 实例化当前线程所对应的 Looper，并将其存储到线程中（Thread 对象的 ThreadLocalMap 变量）。
- 实例化 MessageQueue。

```java
public final class Looper {

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static Looper sMainLooper;  // guarded by Looper.class

    final MessageQueue mQueue;
    final Thread mThread;

    public static void prepare() {
        // 默认调用 prepare(true)，表示这个 Looper 允许退出。
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        // 一个线程只能有一个 Looper，为什么这么做呢？
        // 假设没有这行代码，那么多次调用可能的结果是：
        // 1.只多调 Looper.prepare()，那么新加入的消息都在新的 Looper 对象中，并且由于没有紧跟 Looper.loop()，加入的消息永远不会执行。
        // 2.多调 Looper.prepare()、Looper.loop()，那么相当于在一个死循环中再加一层死循环，且新增的死循环没退出之前，外层 Looper 的其它消息永远得不到执行.
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        // 可以看到在创建 Looper 对象的时候, 同时会创建一个 MessageQueue 对象, 将它保存到 Looper 对象的成员变量 mQueue 中, 因此每一个 Looper 对象都对应一个 MessageQueue 对象。
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    public static @Nullable Looper myLooper() {
        // 通过 ThreadLocal 获取当前线程中的 Looper 对象。
        return sThreadLocal.get();
    }
}
```


#### 1.2.2 Looper.loop()

Looper.loop() 的核心部分：

- 从 MessageQueue 中获取到下一条 Message；
- 将 Message 分发给相应的 target 处理；
- 分发后的 Message 处理完后，回收到消息池，以便重复利用。

```java
public static void loop() {
    // 拿到当前线程中的 Looper 对象。
    final Looper me = myLooper();

    final MessageQueue queue = me.mQueue;

    Binder.clearCallingIdentity();
    // 确保在权限检查时基于本地进程，而不是调用进程。
    final long ident = Binder.clearCallingIdentity();

    // （1）进入线程的死循环。
    for (;;) {
        // （2）可能会阻塞，只有退出 Looper（looper.quit()）后，msg 才会返回 null，也通过此 return 终止掉死循环。
        Message msg = queue.next();
        if (msg == null) {
            return;
        }

        final Printer logging = me.mLogging;
        if (logging != null) {
            // 消息处理前的回调。
            // 在消息执行结束后同样有一个回调，因此可通过这 2 个回调去做 Looper 消息的耗时判断，例如主线程消息时差大于 5 秒则认为卡顿，BlockCanary 的原理也是基于此。
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        // 分发消息（处理消息）。
        msg.target.dispatchMessage(msg);

        // 消息执行结束后的回调。
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // 将消息放到消息池中，以便重复利用。
        msg.recycleUnchecked();
    }
}
```

#### 1.2.3 looper.quit()

```java
public void quit() {
    mQueue.quit(false);
}

public void quitSafely() {
    mQueue.quit(true);
}
```

最终调用的是 messageQueue.quit()。

```java
void quit(boolean safe) {
    // 当 mQuitAllowed 为 false，表示不运行退出，强行调用 quit() 会抛出异常。
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    // 在取存消息时，也会加上该锁，主要是为了多线程下的线程安全。
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            // 只移除尚未触发的所有消息，对于准备处理的消息并不移除。
            // 这个准备处理的定义是指若当前时间的时间戳大于 message.when，则表明准备要执行，该值在消息加入消息队列时赋值。
            removeAllFutureMessagesLocked();
        } else {
            // 移除所有的消息，并回收到消息池中。
            removeAllMessagesLocked();
        }

        // mPtr 是用于和 native 交互的句柄，当前线程可能处于休眠状态，需要唤醒退出。
        nativeWake(mPtr);
    }
}
```

### 1.3 Handler

Handler 的作用：向消息队列发送事件的拓展支持(Handler.sendMessage)和处理相应消息(Handler.dispatchMessage)。

#### 1.3.1 构造函数和成员变量

```java
public class Handler {

    final Looper mLooper;
    final Callback mCallback;
    final boolean mAsynchronous;

    public Handler() {
        this(null, false);
    }

    public Handler(@Nullable Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        // 必须先在当前线程执行 Looper.prepare()，才能获取 Looper 对象，否则为 null。
        // 同时可以发现，对于该构造函数来说，「Handler 在哪个线程创建，就归属于哪个线程的 Looper」。
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        // 该消息是否是否是异步的，Handler 有一个同步屏障的概念，在开启的情况下，异步的消息会先执行（优先级高），下文做详细讲解。
        mAsynchronous = async;
    }

    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        // 主动申明该 Handler 属于哪个 Looper。
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
}
```

#### 1.3.2 消息分发和处理

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // 优先执行 Message 自己内部的 Runnable。
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            // 当 Handler 存在 Callback 成员变量时，回调方法 handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // Handler 自身的回调方法 handleMessage()。
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}

public void handleMessage(@NonNull Message msg) {
    // 用于重写，也是一般情况下最常用的方式。
}
```

#### 1.3.3 消息发送

```java
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}

public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}

public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageAtTime(msg, uptimeMillis);
}

public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    // 该方法通过设置消息的触发时间为 0，从而使 Message 加入到消息队列的队头。
    return enqueueMessage(queue, msg, 0);
}
```

可以发现所有的发消息方式，最终都是调用 messageQueue.enqueueMessage()。

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        // 归属于该 Handler 的消息都是异步消息。
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

#### 1.3.4 消息移除

都是调用 MessageQueue 的方法，下文再细看。

```java
public final void removeCallbacks(@NonNull Runnable r) {
    mQueue.removeMessages(this, r, null);
}

public final void removeCallbacks(@NonNull Runnable r, @Nullable Object token) {
    mQueue.removeMessages(this, r, token);
}

public final void removeCallbacksAndMessages(@Nullable Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}

public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}

public final void removeMessages(int what, @Nullable Object object) {
    mQueue.removeMessages(this, what, object);
}
```

### 1.4 MessageQueue

MessageQueue 是消息机制的 Java 层和 native 层的连接纽带，大部分核心方法都交给 native 层来处理，关于这些 native 方法的介绍，见 [二、Native 篇](#二native-篇)。

#### 1.4.1 构造函数

```java
public final class MessageQueue {
    
    private final boolean mQuitAllowed; // true 表示这个消息队列是可停止的。
    private long mPtr;                  // 描述一个 Native 的句柄
    
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        // 获取一个 native 句柄。
        mPtr = nativeInit();
    }
}
```

#### 1.4.2 messageQueue.next()

```java
Message next() {
    final long ptr = mPtr;
    // ptr==0，代表调用了 quit()，表示退出消息循环，此时会返回 null。
    if (ptr == 0) {
        return null;
    }


    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        // 阻塞操作（释放线程的时间片），直到等待 nextPollTimeoutMillis 时长，或者消息队列被唤醒。
        // 当 nextPollTimeoutMillis == 0，不阻塞。
        // 当 nextPollTimeoutMillis == -1，会一直等待下去。
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            // 正常情况下，message 的 target 都不会为 null，若为 null，则说明开启了同步屏障（postSyncBarrier()）。
            // 简单点说就是开启的情况下，只执行异步消息(msg.isAsynchronous() == true)，可以把该类消息理解成优先级高的消息，
            // 也因此常常用于视图的更新（ViewRootImp#scheduleTraversals()），以尽可能保证 UI 的流畅。
            if (msg != null && msg.target == null) {
                // 不断尝试寻找异步消息，直到找到或者最终 msg == null。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 当前的消息还未到要执行的时间节点，则在下一轮循环中阻塞休眠一定时间。
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 成功获取 MessageQueue 中的即将要执行的消息。
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    // 设置消息的使用状态，即 flags |= FLAG_IN_USE。
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 消息队列中没有消息或者开启了同步屏障，却没有异步消息。
                nextPollTimeoutMillis = -1;
            }

            // 退出 Handler 机制，通过返回 null 中止 Looper 的循环。
            if (mQuitting) {
                dispose();
                return null;
            }

            // 从概念上来看，mIdleHandlers 是在 Looper 空闲时执行的任务。
            // 从细节来看：
            // 1. 空闲的标准在于消息队列为空，或者是当前时间小于下一条消息的时间。
            // 2. pendingIdleHandlerCount 字段只在第一次循环时为 -1，在下面的代码中会置为 0，
            // 也就是说，每取一次消息（messageQueue.next()），只会尝试一次去处理空闲任务。
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // 没有需要 Looper 空闲时执行的任务，跳到下一次循环，准备阻塞休眠。
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null;

            boolean keep = false;
            try {
                // 执行空闲任务，返回值为执行完毕后，是否保留。
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // pendingIdleHandlerCount 赋值为 0，以便不再运行它们。
        pendingIdleHandlerCount = 0;

        // 在执行空闲任务的过程中，可能有新的 message 加入消息队列，因此赋值为 0，不要阻塞等待。
        nextPollTimeoutMillis = 0;
    }
}
```

#### 1.4.3 messageQueue.enqueueMessage()

```java
boolean enqueueMessage(Message msg, long when) {
    // 每一个正常的 Message 必须有对应的 target。
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            // 处于正在退出，直接回收 msg。
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 将 msg 作为消息队列的队头。
            // p == null，代表消息队列中没有消息。
            msg.next = p;
            mMessages = msg;
            // 若处于阻塞状态又有新消息加入消息队列，则需要唤醒。
            needWake = mBlocked;
        } else {
            // p.target == null 说明处于同步屏障状态，
            // 在此条件下，mBlocked == true，说明没有异步消息，处于阻塞休眠中，
            // 新加入的消息又为异步消息，则需要唤醒执行异步消息。
            // 其它情况下从逻辑上不存在需要唤醒的可能。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                // 将消息按时间顺序插入到消息队列中。
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }

        if (needWake) {
            // 唤醒 Looper 对应的线程（因为当前线程可以是任意线程）。
            nativeWake(mPtr);
        }
    }
    return true;
}
```

#### 1.4.4 messageQueue.removeMessages()

移除消息有 2 种方式，一个是 what，一个是 Runnable 对象，两者的逻辑几乎一致，区别在于 Runnable 判断的是引用的地址是否一致。

```java
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }
    synchronized (this) {
        Message p = mMessages;
        // 第一个循环：由于队头遍历的特殊性，先移除从队头开始「所有符合条件且连续」的消息。
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }
        // 移除剩余的符合要求的消息。
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

#### 1.4.5 messageQueue.postSyncBarrier()

开启同步屏障，让异步消息先执行。

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 将同步屏障的消息 msg 根据时间排序插入到消息队列中。
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        // 返回值作为该次同步屏障的标识，用于关闭删除该次同步屏障的消息。
        return token;
    }
}
```

#### 1.4.6 messageQueue.removeSyncBarrier()

```java
public void removeSyncBarrier(int token) {
     synchronized (this) {
         Message prev = null;
         Message p = mMessages;
         //从消息队列找到 target 为空,并且 token 相等的消息。
         while (p != null && (p.target != null || p.arg1 != token)) {
             prev = p;
             p = p.next;
         }
         final boolean needWake;
         // 该消息不在队头，说明还未处于同步屏障的状态中，直接移除该消息即可。
         if (prev != null) {
             prev.next = p.next;
             needWake = false;
         } else {
             // 处于开启同步屏障状态，可能处于阻塞等待异步消息状态，需要唤醒。
             mMessages = p.next;
             needWake = mMessages == null || mMessages.target != null;
         }
         p.recycleUnchecked();

         if (needWake && !mQuitting) {
             nativeWake(mPtr);
         }
     }
 }
```

### 1.5 Message

#### 1.5.1 成员变量

```java
public final class Message implements Parcelable {
    
    public int what; // 消息类别

    public int arg1; // 参数 1

    public int arg2; // 参数 2

    public Object obj; // 消息可携带的对象。

    Handler target; // 该消息所归属的 Handler。

    Message next; // 消息队列的数据结构是链表。
}
```

#### 1.5.2 消息池

池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率，还能对资源的管理有一层很好的封装。而 Handler 机制作为 Android 进程内最重要的交互方式，伴随着大量使用 Message 的需求。

```java
public final class Message {
    public static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;

    private static final int MAX_POOL_SIZE = 50;
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                // m 作为即将复用的消息，从消息池链表中分离出来使用。
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        // 当消息池为空时，直接创建 Message 对象。
        return new Message();
    }
    
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    void recycleUnchecked() {
        // 回收该消息，重置相应参数，并加入消息池队列的队头。
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```

## 二、Native 篇

### 2.1 epoll 机制

#### 2.1.1 多路复用

IO 多路复用是一种同步 IO 模型，实现一个线程可以监视多个文件句柄。一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作，没有文件句柄就绪时会阻塞应用程序，交出 CPU。

与多进程和多线程技术相比，IO 多路复用技术的最大优势是系统开销小，系统不必为每个 IO 操作都创建进程或线程，也不必维护这些进程或线程，从而大大减小了系统的开销。

select、poll、epoll 就是 IO 多路复用三种实现方式。

- select 最大连接数为进程文件描述符上限，一般为 1024；每次调用 select 拷贝 fd；轮询方式工作时间复杂度为 O(n)；
- poll 最大连接数无上限；每次调用 poll 拷贝 fd；轮询方式工作时间复杂度为 O(n)；
- epoll 最大连接数无上限；首次调用 epoll_ctl 拷贝 fd，调用 epoll_wait 时不拷贝；回调方式工作时间复杂度为 O(1)。

#### 2.1.2 API

```java
// 创建 eventpoll 对象，并将 eventpoll 对象放到 epfd 对应的 file->private_data 上，返回一个 epfd，即 eventpoll 文件描述符。
int epoll_create(int size);

/**
 * 对一个 epfd 进行操作。
 * @param op 表示要执行的操作，包括 EPOLL_CTL_ADD (添加)、EPOLL_CTL_DEL (删除)、EPOLL_CTL_MOD (修改)。
 * @param fd 表示被监听的文件描述符。
 * @param event 表示要被监听的事件，包括：
 *        - EPOLLIN（表示被监听的 fd 有可以读的数据）
 *        - EPOLLOUT（表示被监听的 fd 有可以写的数据）
 *        - EPOLLPRI（表示有可读的紧急数据）
 *        - EPOLLERR（对应的 fd 发生异常）
 *        - EPOLLHUP（对应的 fd 被挂断）
 *        - EPOLLET（设置 EPOLL 为边缘触发）
 *        - EPOLLONESHOT（只监听一次）
 * @return 成功 0；失败 -1。
 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)

/**
 * 等待 epfd 监听的 fd 所产生对应的事件。
 * @param epfd 表示 epoll 文件描述符。
 * @param events 表示回传处理事件的数组。
 * @param maxevents 表示每次能处理的最大事件数
 * @param timeout 等待 IO 的超时时间，等于 0 表示不阻塞，-1 表示一直阻塞直到 IO 被唤醒，大于 0 表示阻塞指定的时间后自动被唤醒。
 * @return 产生的监听事件数。
 */
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

#### 2.1.3 eventfd

eventfd 是 Linux 系统中一个用来通知事件的文件描述符，基于内核空间向用户空间应用发送通知的机制，可以有效地被用来实现用户空间事件驱动的应用程序，它只有一个系统调用接口：

```java
/**
 * 打开一个 eventfd 文件并返回文件描述符，支持 epoll/poll/select 操作。
 */
int eventfd(unsigned int initval, int flags);
```

### 2.2 简介

除了 MessageQueue 的 native 方法，native 层本身也有一套完整的消息机制，用于处理 native 的消息。

在整个消息机制中，而 MessageQueue 是连接 Java 层和 Native 层的纽带，换言之，Java 层可以向 MessageQueue 消息队列中添加消息，Native 层也可以向 MessageQueue 消息队列中添加消息。

MessageQueue 中涉及的 native 方法如下，这里主要看几个关键的 native 方法：

```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

### 2.3 MessageQueue.nativeInit()

（1）new MessageQueue()

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    // 作为与 native 交互的句柄。
    mPtr = nativeInit();
}
```
（2）android_os_MessageQueue_nativeInit()

```cpp
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    // 创建了一个 NativeMessageQueue 对象。
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    // 增加引用计数。
    nativeMessageQueue->incStrong(env); 
    // 将这个 NativeMessageQueue 对象强转成了一个句柄返回 Java 层。
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
（3）new NativeMessageQueue()

```cpp
class NativeMessageQueue : public MessageQueue, public LooperCallback {
public:
    NativeMessageQueue();
    ......
private:
    JNIEnv* mPollEnv;
    jobject mPollObj;
    jthrowable mExceptionObj;
};

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    // 尝试调用 Looper.getForThread 获取当前线程的 Looper 对象（C++）。
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        // 若当前线程没有创建过 Looper，则创建 Looper 对象。
        mLooper = new Looper(false);
        // 给当前线程绑定这个 Looper 对象。
        Looper::setForThread(mLooper);
    }
}
```

（4）new Looper()

```cpp
// system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    // 通过 eventfd 系统调用返回一个文件描述符，用于内核空间向用户空间（APP）进行事件通知。
    // Android 6.0 之前为 pipe, 6.0 之后为 eventfd。
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```

（5）rebuildEpollLocked()

```cpp
void Looper::rebuildEpollLocked() {
    ......
    // 创建一个 epoll 对象, 将其文件描述符保存在成员变量 mEpollFd 中。
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    ......
    // 使用 epoll 监听 mWakeEventFd, 用于唤醒线程。
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
    ......
}
```

### 2.4 MessageQueue.nativePollOnce()

（1）messageQueue.next()

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis);
        ...
    }
```

（2）android_os_MessageQueue_nativePollOnce()
```cpp
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    // 将 Java 层传递下来的 mPtr 转换为 nativeMessageQueue。
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    // 调用了 NativeMessageQueue 的 pollOnce。
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}

```

（3）NativeMessageQueue::pollOnce()

```cpp
// frameworks/base/core/jni/android_os_MessageQueue.cpp
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    ......
    // 调用了 Looper 的 pollOne。
    mLooper->pollOnce(timeoutMillis);
    ......
}
```

（4）Looper::pollOnce()

```cpp
// system/core/libutils/Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ......
        // result 不为 0 说明读到了消息（Java 层唤醒）。
        if (result != 0) {
            ......
            return result;
        }
        // 若未读到消息, 则调用 pollInner（可能会多次执行）。
        result = pollInner(timeoutMillis);
    }
}
```

（5）Looper::pollInner()

```cpp
// system/core/libutils/Looper.cpp
int Looper::pollInner(int timeoutMillis) {
    ......
    int result = POLL_WAKE;
    ......
    // 即将处于 idle 状态。
    mPolling = true; 
    // fd 最大个数为 16。
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    // 关键点：调用 epoll_wait 阻塞直到监听到 mEpollFd 中的 IO 事件（nativeWake()）, 或超过一定时间 timeoutMillis 后。
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    // 不再处于 idle 状态。
    mPolling = false; 

    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR; // epoll 监听的事件个数小于 0，发生错误，直接跳转 Done;
        goto Done;
    }

    if (eventCount == 0) {  //epoll 监听的事件个数等于 0，发生超时，直接跳转 Done;
        result = POLL_TIMEOUT;
        goto Done;
    }

    // 走到这里，说明被唤醒，开始处理事件。
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;             
        uint32_t epollEvents = eventItems[i].events;
        // 重点：mWakeEventFd 用于向应用层 Handler 发送事件通知。
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                // 若已经唤醒了，则读取并清空管道数据。
                awoken();
            }
            // 
        } else {
           ......
        }
    }
    ......
    return result;
}

```

### 2.5 MessageQueue.nativeWake()

（1）MessageQueue.enqueueMessage()

```java
boolean enqueueMessage(Message msg, long when) {
    // 将 Message 按时间顺序插入 MessageQueue。
    if (needWake) {
        nativeWake(mPtr);
    }
}
```

（2）android_os_MessageQueue_nativeWake()

```cpp
// frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
```

（3）nativeMessageQueue::wake()

```cpp
// frameworks/base/core/jni/android_os_MessageQueue.cpp
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

（4）Looper::wake()

```cpp
// system/core/libutils/Looper.cpp
void Looper::wake() {
    // 向文件描述符 mWakeEventFd 写入一个新的数据，从而唤醒通过 epoll_wait 进入休眠的线程。
    // 其中 TEMP_FAILURE_RETRY 是一个宏定义，当执行 write 失败后，会不断重复执行，直到执行成功为止。
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    ....
}
```
