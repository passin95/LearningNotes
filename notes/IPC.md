



- [Android IPC](./notes/IPC.md)

  IPC、Android IPC 方式、Binder。


# IPC

IPC 是 Inter-Process Communication 的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。

## 序列化方式

为什么需要对对象进行序列化呢？本质需求是由于进程间的限制，无法直接读取其它进程的内存对象，从而将进程内的内存对象转换为可在多进程环境中直接读写的“对象”。

### Serializable

Serializable 是 Java 所提供的序列化接口，本质就是将对象写入文件以及反之。

```java
// 序列化过程。
User user = new User("123");
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"))
out.writeObject(user);
out.close();

// 反序列化过程。
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt")
User newUser = (User) in.readObject();
in.close();
```

使用方式：实现 Serializable 接口，并设置 serialVersionUID 标识（通过 IDE 对实现 Serializable 的类按 Enter + Alt 生成）；若类中没有显式指定serialVersionUID，则程序会根据类名、接口名、属性和方法自动生成一个。

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1087027745012376875L;
}

```

serialVersionUID 的作用就是辅助序列化过程的，序列化的时候会将当前类的 serialVersionUID 写入序列化的文件中，当反序列化的时候系统会去检测文件中的 serialVersionUID 是否和当前类一致，一致则说明类无变化，反之发生变化无法反序列化，并抛出异常。

### Parcelable

Parcelable 也是一个接口，它的本质是将一个对象进行分解，并将需要序列化的字段写入 Parcel，Parcel 可以在 Binder 中传输，是内存级别的序列化。

序列化的使用方式如下所示：

```java
public class User implements Parcelable {

    String name;

    public User(String name) {
        this.name = name;
    }

    /**
     * 返回当前对象的内容描述，如果含有文件描述符，返回 1(CONTENTS_FILE_DESCRIPTOR)。
     */
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     * 将当前对象写入序列化结构中，其中 Flag 标识有两种值：0 或者 1。
     * 为 1(PARCELABLE_WRITE_RETURN_VALUE) 时标识当前对象需要作为返回值返回，不能立即释放资源。大部分情况下为 0。
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
    }

    protected User(Parcel in) {
        this.name = in.readString();
    }

    /**
     * 反序列化时的操作类，它标明了如何创建序列化对象和数组，并通过
     * Parcel 的一系列 read 方法来完成反序列化过程.
     */
    public static final Creator<User> CREATOR = new Creator<User>() {

        /**
         * 反序列化，从 Parcel 中读取字段，注意序列化和反序列的顺序必须一致。
         */
        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        /**
         * 创建指定长度的原始对象数组.
         */
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
}
```

### Serializable 和 Parcelable 的选取。

Serializable 使用简单但是开销较大，序列化和反序列化都需要进行 I/O 操作。而 Parcelable 是 Android 中的序列化方式，使用稍微麻烦（可用插件解决），但它的效率相对较高，因此在能达到同等需求的情况下，更推荐使用 Parcelable。同时也可以考虑使用 twitter 开源的序列化框架 [Serial](https://github.com/twitter/Serial/blob/master/README-CHINESE.rst/)，它具备了前面两者的优点。

一般序列化框架都会对 transient 关键字进行兼容，因此 transient 修饰的字段不会被序列化。

# Android IPC

## Android 多进程模式

### 开启多进程模式

在 Android 中使用多进程的常规方法是：为四大组件指定 android:process 属性。指定的方式有两种（包名：me.passin.process）：

1. “:remote”：在当前的进程前附上当前的包名后，即进程名为 me.passin.process:remote 。属于当前应用的私有进程。
2. “me.passin.process.remote”：进程名为 me.passin.process.remote。属于全局进程。

### 运行机制和多进程造成的问题

1. Android 系统会为每个进程分配一个独立的虚拟机，每个虚拟机的启动过程其就是启动一个应用的过程。
2. Android 系统的进程和其它系统的进程一样，每个进程都拥有自己独立的内存空间，因此不同的虚拟机访问同一个类的对象会产生多份副本。

基于进程的特性和运行机制，在使用多进程时会造成以下问题：

- 静态成员和单例模式在多进程中失效；
- 线程同步机制在多进程中失效；
- SharedPreferences 的可靠性下降（不支持多进程并发读写文件）；
- Application 会多次创建。

## Android 应用层 IPC 方式

### Bundle

Bundle 由于实现了 Parcelable 接口，所以它可以方便地在不同的进程间传输。一般用于在一个进程中启动另一个进程的 Activity、Service和 Receiver时，传输能够被序列化的数据。这是最简单的进程间通信方式。

### 文件共享

文件共享类似 Serializable 的本质，并且使用这种方式对文件格式是没要求的，只要读/写双方约定数据格式即可。但由于如果存在并发读/写的可能，可以导致读取的内容并不是最新的，因此如果使用该种方式需要妥善处理该问题（并发读写）。

### Messenger

<style>
table th:first-of-type {
    width: 50pt;
}
table th:nth-of-type(2) {
    width: 90pt;
}
table th:nth-of-type(3) {
    width: 90pt;
} 
table th:nth-of-type(4) {
    width: 90pt;
} 
table th:nth-of-type(5) {
    width: 100pt;
} 
</style>


| 方式 | 优点 | 缺点 | 适用场景 | 本质 |
| :-- | :--- | :--- | :--- | :--- |
| Bundle | 简单易用 | 只能传输 Bundle 支持的数据类型 | 四大组件间的进程间通信 | Bunble 实现了 Parcelable 接口，通过 Parcelable 序列化方式传输 |
| 文件共享 | 简单易用 | 不适合高并发场景 | 无并发访问情形；Serializable；非多进程下的 SharedPreferences | 内存写入文件，文件读入内存 |
| AIDL | 功能强大，支持一对多并发通信；支持实时通信 | 使用较复杂，需要处理好线程同步 | 一对多且有 RPC（（Remote Procedure Call））需求 | 本质是提供了一种快速实现 Binder 的工具 |
| Messaenger | 支持一对多串行通信，支持实时通信 | 不支持 RPC；数据通过 Message 进行传输，因此只能传输 Bundle 支持的数据类型，不支持 object 字段 | 四大组件间的进程间通信 | AIDL，对 ALDL 进行了一层封装 |
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享 | 主要提供数据源的 CRUD | 一对多的进程间数据共享 | Binder |
| Socket | 可自定义传输协议、拓展性强，可用于网络传输，支持一对多并发实时通信 | 需要制定应用层传输协议，实现比较麻烦；传输效率相对较低 | 网络上不同进程通信 | 网络传输 |

## Binder

Binder 是 Android 中的一种跨进程通信方式，从 Android Framework 角度来说，Binder 是 ServiceManger 连接各种 Manager（ActivityManager、WindowManger）和相应 ManagerService 的桥梁。

### 为什么选 Binder

Android 系统是基于 Linux 的操作系统，为何不直接采用Linux现有的进程IPC方案呢，而采用 Binder 作为 IPC 机制呢，我们可以先看一下 Linux 现有的进程间 IPC 方式的区别：

1. 管道：在创建时分配一个page大小的内存，缓存区大小比较有限；
2. 消息队列：信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信；
3. 共享内存：无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；
4. 套接字：作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通信；
5. 信号量：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。
6. 信号: 不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；

**（1）性能**

Binder数据拷贝只需要一次，而管道、消息队列、Socket都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder性能仅次于共享内存。

**（2）稳定性**

Binder是基于C/S架构的，C/S架构是指客户端(Client)和服务端(Server)组成的架构，Client端有什么需求，直接发送给Server端去完成，架构清晰明朗，Server端与Client端相对独立，稳定性较好；而共享内存实现方式复杂，没有客户端与服务端之别，需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从稳定性角度看，Binder优于共享内存。

**（3）安全性**

传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Android作为一个开放的开源体系，拥有非常多的开发平台，App来源甚广，因此手机的安全显得额外重要；对于普通用户，绝不希望 APP 上传隐私数据、后台偷跑流量、消耗电量等问题。传统Linux IPC无任何保护措施，完全由上层协议来确保。

Android 系统则为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志。**通过使用 Binder，Android 系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限，例如部分权限是通过弹出权限询问对话框，让用户选择是否运行。**

### Binder 架构

Binder在整个Android系统中有这举足轻重的地位。在 zygote 进程孵化出system_server进程后，在system_server进程中出初始化支持整个Android framework的各种各样的Service，而这些Service从大的方向来划分，分为Java层Framework和Native Framework层(C++)的Service，几乎都是基于BInder IPC机制。Java framework：作为Server端继承(或间接继承)于Binder类，Client端继承(或间接继承)于BinderProxy类。例如 ActivityManagerService(用于控制Activity、Service、进程等) 这个服务作为Server端，间接继承Binder类，而相应的ActivityManager作为Client端，间接继承于BinderProxy类。 当然还有PackageManagerService、WindowManagerService等等很多系统服务都是采用C/S架构；Native Framework层：这是C++层，作为Server端继承(或间接继承)于BBinder类，Client端继承(或间接继承)于BpBinder。例如MediaPlayService(用于多媒体相关)作为Server端，继承于BBinder类，而相应的MediaPlay作为Client端，间接继承于BpBinder类。

## Android IPC 方式

## Bundle

Bundle 由于实现了 Parcelable 接口，因此支持在不同的进程间传输。

## 文件共享

SharedPreferences、Serializable 都是文件共享的实际运用，由于 Android 系统没有对读写文件进行限制，所以在并发读写时可能会出问题，因此需要开发者尽量避免并发的情况发生，或者设计出支持多进程读写文件的框架。

 4. Binder机制，从java到framework再到kenral层，面试官问的都很详细，遇到不会的也都会跟我解释。

### Binder


指定 android:process 属性

性能
首先说说性能上的优势。Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。Binder 只需要一次数据拷贝，性能上仅次于共享内存。

1.ActivityManagerServices，简称AMS，服务端对象，负责系统中所有Activity的生命周期
2.ActivityThread，App的真正入口。当开启App之后，会调用main()开始运行，开启消息循环队列，这就是传说中的UI线程或者叫主线程。与ActivityManagerServices配合，一起完成Activity的管理工作
3.ApplicationThread，用来实现ActivityManagerService与ActivityThread之间的交互。在ActivityManagerService需要管理相关Application中的Activity的生命周期时，通过ApplicationThread的代理对象与ActivityThread通讯。
4.ApplicationThreadProxy，是ApplicationThread在服务器端的代理，负责和客户端的ApplicationThread通讯。AMS就是通过该代理与ActivityThread进行通信的。
5.Instrumentation，每一个应用程序只有一个Instrumentation对象，每个Activity内都有一个对该对象的引用。Instrumentation可以理解为应用进程的管家，ActivityThread要创建或暂停某个Activity时，都需要通过Instrumentation来进行具体的操作。
6.ActivityStack，Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。
7.ActivityRecord，ActivityStack的管理对象，每个Activity在AMS对应一个ActivityRecord，来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像。
8.TaskRecord，AMS抽象出来的一个“任务”的概念，是记录ActivityRecord的栈，一个“Task”包含若干个ActivityRecord。AMS用TaskRecord确保Activity启动和退出的顺序。如果你清楚Activity的4种launchMode，那么对这个概念应该不陌生。

1.Window用于显示View和接收各种事件，Window有三种类型：应用Window(每个Activity对应一个Window)、子Window(不能单独存在，附属于特定Window)、系统window(Toast和状态栏)
2.Window分层级，应用Window在1-99、子Window在1000-1999、系统Window在2000-2999.WindowManager提供了增删改View三个功能。
3.Window是个抽象概念：每一个Window对应着一个View和ViewRootImpl，Window通过ViewRootImpl来和View建立联系，View是Window存在的实体，只能通过WindowManager来访问Window。
4.WindowManager的实现是WindowManagerImpl其再委托给WindowManagerGlobal来对Window进行操作，其中有四个List分别储存对应的View、ViewRootImpl、WindowManger.LayoutParams和正在被删除的View
5.Window的实体是存在于远端的WindowMangerService中，所以增删改Window在本端是修改上面的几个List然后通过ViewRootImpl重绘View，通过WindowSession(每个应用一个)在远端修改Window。
6.Activity创建Window：Activity会在attach()中创建Window并设置其回调(onAttachedToWindow()、dispatchTouchEvent()),Activity的Window是由Policy类创建PhoneWindow实现的。然后通过Activity#setContentView()调用PhoneWindow的setContentView。

# 参考资料

- 任玉刚. Android 开发艺术探究. 电子工业出版社, 2015.
- [CS-Notes. Java 基础](https://github.com/CyC2018/CS-Notes/blob/master/docs/notes/Java%20%E5%9F%BA%E7%A1%80.md)
- [Gityuan. 为什么Android要采用Binder作为IPC机制？](https://www.zhihu.com/question/39440766/answer/89210950)