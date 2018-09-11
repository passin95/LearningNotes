
# 一、基础概念

## 线程和进程

- 线程是操作系统能够进行运算调度的最小单位，它依赖于进程创建，是进程中的实际运作单位。

- 每个进程都拥有自己独立的内存空间，且该进程下所有线程都共享该内存空间。

- 进程和线程都是独立的执行路径，但一个进程可以有多个线程。 

- 每个进程都有自己的内存空间、可执行代码和唯一的进程标识符（PID），而每个线程在Java中都有自己的私有栈，堆一般使用进程共享内存并与其他线程共享。

## 线程的状态（生命周期）

 <img src="../pictures//线程生命周期.png" />
 <br></br>

 
###  新建(New）

创建Thread对象，在start启动之前，该线程不存在。

### 可运行（Runnable）

该状态可细分为可运行(Runnable)和运行中(Running)两个状态。

由于线程和进程的运行都听令于CPU的调度，在CPU没有通过轮询或其他方式从任务可执行队列选中该线程前，处于Runnable状态，选中之后处于Running状态。

Running状态的线程也是属于Runnable状态，反之不成立。

### 阻塞（Blocking）

Blocking状态一般为：
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





### 小结




## 多线程开发规范

- 为线程赋予一个有意义的名字有助于问题的排查和线程的追踪。

##

# 面试题解

### 什么是线程


### Thread类run()和start()的区别

Thread.run()运行在当前线程，start()启动新创建的线程,并在新线程执行run()方法。

##

# 参考资料
- 汪文君. Java高并发编程详解 [M]. 机械工业出版社, 2018.
- 周志明. 深入理解 Java 虚拟机 [M]. 机械工业出版社, 2011.
- [CS-Notes. Java 并发](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20并发.md)