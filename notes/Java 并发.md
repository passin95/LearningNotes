
# 一、基础概念

## 线程和进程

- 线程是操作系统能够进行运算调度的最小单位，它依赖于进程创建，是进程中的实际运作单位。

- 每个进程都拥有自己独立的内存空间，且该进程下所有线程都共享该内存空间。

- 进程和线程都是独立的执行路径，但一个进程可以有多个线程。 

- 每个进程都有自己的内存空间、可执行代码和唯一的进程标识符（PID），而每个线程在Java中都有自己的私有栈，堆一般使用进程共享内存并与其他线程共享。

## 线程的状态（生命周期）

[<img src="../pictures//线程生命周期.png" />](https://www.cnblogs.com/huangzejun/p/7908898.html)
<center>点击图片跳转图片出处</center>
 
###  新建(New）

创建Thread对象，在start启动之前，该线程不存在。

### 可运行（Runnable）

该状态可细分为可运行(Runnable)和运行中(Running)两个状态。

由于线程和进程的运行都听令于CPU的调度，在CPU没有通过轮询或其他方式从任务可执行队列选中该线程前，处于Runnable状态，选中之后处于Running状态。

Running状态的线程也是属于Runnable状态，反之不成立。

### 阻塞（Blocked）

Blocked状态一般为：
- 调用了sleep()或wait()或在run方法中其它线程的.join()方法。
- 为了获取某个锁资源，从而加入到该锁的阻塞队列时。
- 正在进行某个阻塞的IO操作，例如网络数据的读写。

### 无限期等待（Watting）

等待其它线程显式地唤醒，否则不会被分配 CPU 时间片。

### 限期等待（Timed Waiting）

无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒。

### 死亡（Terminated）

线程结束任务之后自己结束，或者产生了异常而结束。

# Thread

## Thread的构造函数

```java
private void init(ThreadGroup group, Runnable target, String name, long stackSize, AccessControlContext acc) {
    // 线程组 ThreadGroup 默认为null
    // 线程名name默认为"Thread-" + 数字(递增)，在进程销毁后重新数字从0重新开始。
    // 一般情况下，stackSize越大，方法可递归调用的深度越深，但和具体的软硬件有关，
    // 一般通过-xms设置栈大小，因此 stackSize 一般使用默认值 0。
    // AccessControlContext默认为null

    if (name == null) {
        throw new NullPointerException("name cannot be null");
    } else {
        this.name = name;
        // 父线程为实例化Thread对象时的线程。
        Thread parent = currentThread(); 
        SecurityManager securityManager = System.getSecurityManager();

        // 如果没有指定线程组
        if (group == null) {
                if (securityManager != null) {
                    group = securityManager.getThreadGroup();
                }

                // 默认使用父线程的线程组。
                if (group == null) {
                    group = parent.getThreadGroup();
                }
            }

            // 省略部分代码

            // 默认线程下，新线程的线程组和父线程是同一个
            // 优先级和是否是守护线程和父线程一致。
            this.group = group;
            this.daemon = parent.isDaemon();
            this.priority = parent.getPriority();

            // 省略部分代码
    }
}
```

- currentThread()方法为获取当前线程，在线程调用start()前并没有创建线程，因此可以发现一个线程的创建一个线程的创建由另一个线程完成，并且被创建线程的父线程为创建它的线程。


## 守护线程

守护线程是一种比较特殊的线程，一般用于处理一些后台的工作，比如垃圾回收（GC）线程。当JVM 中没有一个非守护线程时，则 JVM 的进程会推出。

调用Thread.setDaemon()方法便可设置线程类型，true代表守护线程，false 代表正常线程。该方法只在线程启动之前有效。

## Thread API

### sleep

Thread.sleep(millis)会休眠当前线程一定的毫秒，该方法不会放弃 monitor 锁的所有权。

该方法可能抛出InterruptedException,开发者可在T 中调用isInterrupted()

#### TimeUnit 代替 sleep

TimeUnit 对 sleep提供了很好的封装，且可读性更强，推荐用TimeUnit 代替 sleep。

```java
Thread.sleep(12257088L);
TimeUnit.HOURS.sleep(3);
TimeUnit.MINUTES.sleep(24);
TimeUnit.SECONDS.sleep(17);
TimeUnit.MILLISECONDS.sleep(88);
```

### yield

Thread.yield()方法用于告知CPU调度器当前线程愿意放弃所占用CPU资源，如果CPU资源不紧张，则可能会忽略提醒。

Thread.yield()会使当前线程从Running状态转换为Runnable状态。

### 中断

一个线程执行完毕之后会自动结束，如果在运行过程中发生异常也会提前结束。

#### interrupt()和InterruptedException

通过调用一个线程的 interrupt() 来中断该线程，如果该线程处于阻塞、限期等待或者无限期等待状态，那么就会抛出 InterruptedException，从而提前结束该线程。但是不能中断 I/O 阻塞和 synchronized 锁阻塞。

除了上述情况之外，仅仅主动调用interrupt()并不会对实际线程的执行代码造成任何影响,它仅仅是将 interrupt 标识为已打断状态。也就是说，当外部调用interrupt()时，线程是否中断执行，取决于线程自身是否愿意结束（结合isInterrupted()使用），标识仅仅作为参考作用。

```java
public class InterruptExample {
    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new MyThread1();
        thread1.start();
        thread1.interrupt();
        System.out.println("Main run");
    }

    private static class MyThread1 extends Thread {
        @Override
        public void run() {
            try {
                TimeUnit.MINUTES.sleep(2);
                System.out.println("Thread run");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
Main run
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at me.passin.demo.InterruptExample$MyThread1.run(InterruptExample.java:24)
```

值得注意的是线程执行的run方法中若捕捉了InterruptedException 异常之后会擦除该线程的 interrupt 标识。

```java
public class InterruptExample {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(){
            @Override
            public void run() {
                try {
                    System.out.println("Thread is interrupted ? "+ isInterrupted());
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    System.out.println("Thread InterruptedException");
                    e.printStackTrace();
                }
                System.out.println("Thread run");
            }
        };
        thread.start();
        thread.interrupt();
        TimeUnit.MILLISECONDS.sleep(100);
        System.out.println("Thread is interrupted ? "+ thread.isInterrupted());
    }
}
```

```
Thread is interrupted ? true
Thread InterruptedException
Thread run
Thread is interrupted ? false
```

#### isInterrupted()和Thread.interrupted()

这2个方法的返回值都用于当前线程是否被打断，不同在于，isInterrupted()不会影响 interrupt 标识的改变，而Thread.interrupted()在调用结束后会擦除掉当前线程的 interrupt 标识（标识为未打断状态），即如果当前被打断了（调用了interrupt()方法）,第一次调用Thread.interrupted() 会返回true，然后interrupt 标识会被擦除，第二次再调用Thread.interrupted()则会返回 false，除非该线程在两次调用Thread.interrupted()期间线程又一次被打断。

isInterrupted()和interrupt()结合使用

```java
public class InterruptExample {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(){
            @Override
            public void run() {
                while (true) {
                    System.out.println("Thread run");
                    if (isInterrupted()) {
                        System.out.println("Thread active acceptance of interruption");
                        return;
                    }
                }
            }
        };
        thread.start();
        thread.interrupt();
        TimeUnit.MILLISECONDS.sleep(100);
        System.out.println("Thread is interrupted ? "+ thread.isInterrupted());
    }
}
```

```
Thread run
Thread active acceptance of interruption
Thread is interrupted ? false
```

### join

在线程中调用另一个线程的 join() 方法，会将当前线程挂起(处于Blocked状态)，直到目标线程执行结束或到达给定的时间或被打断。

对于以下代码，虽然 b 线程先启动，但是因为在 b 线程中调用了 a 线程的 join() 方法，b 线程会等待 a 线程结束才继续执行，因此最后能够保证 a 线程的输出先于 b 线程的输出。

```java
public class JoinExample {

    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        B b = new B(a);
        b.start();
        a.start();
//      b.interrupt();
    }

    private static class A extends Thread {
        @Override
        public void run() {
            System.out.println("A");
        }
    }

    private static class B extends Thread {
        private A a;

        B(A a) {
            this.a = a;
        }

        @Override
        public void run() {
            try {
                a.join();
            } catch (InterruptedException e) {
                System.out.println("ThreadB InterruptedException");
            }
            System.out.println("B");
        }
    }
}
```

```
A
B
```

取消注释 b.interrupt() 后的结果：
```
ThreadB InterruptedException
B
A
```

# 线程安全与数据同步

## synchronized

synchronized 关键字提供了一种锁的机制，确保共享变量的线程间互斥访问，从而防止数据不一致的问题。

synchronized 包括两个monitor enter 和 monitor exit 两个指令，它能够保证在任何时候任何线程执行到monitor enter成功之前都必须从主内存中获取数据，而不是从CPU的缓存中取数据，在 monitor exit 运行成功之后，会将更新后的值刷入主内存中。

## 死锁

死锁是指两个或两个以上的进程（线程）在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。

### 死锁产生条件

死锁的发生必须具备以下四个必要条件：

#### 互斥条件

指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放。

#### 请求和保持条件

指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。

#### 不剥夺条件

指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。

#### 环路等待条件

指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源。

### 简单的死锁Demo

多线程操作时，交叉锁可能导致死锁的Demo

```java
private final Object MONITOR_READ = new Object();
private final Object MONITOR_WRITE = new Object();

public void read() {
    synchronized (MONITOR_READ) {
        synchronized (MONITOR_WRITE) {
            // ……
        }
    }
}

public void write() {
    synchronized (MONITOR_WRITE) {
        synchronized (MONITOR_READ) {
            // ……
        }
    }
}   
```

# 线程间通信



# 多线程开发规范

- 为线程赋予一个有意义的名字有助于问题的排查和线程的追踪。
- 缩小同步范围，从而减少锁争用。例如对于 synchronized，应该尽量使用同步块而不是同步方法。
- 


# 面试题解

### 什么是线程


### Thread类run()和start()的区别

Thread.run()运行在当前线程，start()启动新创建的线程,并在新线程执行run()方法。

##

# 参考资料
- 汪文君. Java高并发编程详解 [M]. 机械工业出版社, 2018.
- 周志明. 深入理解 Java 虚拟机 [M]. 机械工业出版社, 2011.
- [CS-Notes. Java 并发](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20并发.md)