




# IPC 简介

IPC 是 Inter-Process Communication 的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。

IPC 不是 Android 所独有的，任何一种操作系统都有相应的 IPC 机制。

## 多进程造成的问题

- 静态成员和单例模式完全失效。
- 线程同步机制完全失效。
- SharedPreferences 的可靠性下降。
- Application 会多次创建。

## 序列化

### Serializable


使用方式：实现 Serializable 接口，并设置 serialVersionUID 标识（通过 IDE 对实现 Serializable 的类按 Enter + Alt 生成）。

```java
private static final long serialVersionUID = 3879888110527174428L;

```

原理：Serializable 序列化和反序列化的本质就是将对象写入文件以及反之。

serialVersionUID 的作用就是辅助该过程的，序列化的时候会将当前类的 serialVersionUID 写入序列化的文件中，当反序列化的时候系统会去检测文件中的 serialVersionUID 是否和当前类一致，一致则说明类无变化，反之发生变化无法反序列化。

```java
// 序列化过程
User user = new User("123");
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"))
out.writeObject(user);
out.close();

// 反序列化过程
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt")
User newUser = (User) in.readObject();
in.close();
```

### Parcelable

Parcelable 序列化的使用方式如下所示，它的本质是将一个完整的对象进行分解，
而分解后的每一部分都是Intent所支持的数据类型。也可以说是内存级别的序列化。

```java
public class User implements Parcelable {

    String name;

    public User(String name) {
        this.name = name;
    }


    /**
     * 返回当前对象的内容描述，如果含有文件描述符，返回 1(CONTENTS_FILE_DESCRIPTOR).
     */
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     * 将当前对象写入序列化结构中，其中 Flag 标识有两种值：0 或者 1.
     * 为 1(PARCELABLE_WRITE_RETURN_VALUE) 时标识当前对象需要作为返回值返回，不能立即释放资源。大部分情况下为 0.
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
    }

    /**
     * 从序列化后的对象中创建原始对象.
     */
    protected User(Parcel in) {
        this.name = in.readString();
    }

    /**
     * 反序列化时的操作类，它标明了如何创建序列化对象和数组，并通过
     * Parcel 的一系列 read 方法来完成反序列化过程.
     */
    public static final Creator<User> CREATOR = new Creator<User>() {

        /**
         * 从序列化后的对象中创建原始对象.
         */
        @Override
        public Text createFromParcel(Parcel source) {
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

Serializable 使用简单但是开销较大，序列化和反序列化都需要大量的 I/O 操作。而 Parcelable 是 Android 中的序列化方式，使用稍微麻烦（可用插件解决），但它的效率很高，因此在能达到同等需求的情况下，更推荐使用 Parcelable。

transient 关键字可以使一些属性不会被序列化。

ArrayList 中存储数据的数组 elementData 是用 transient 修饰的，因为这个数组是动态扩展的，并不是所有的空间都被使用，因此就不需要所有的内容都被序列化。通过重写序列化和反序列化方法，使得可以只序列化数组中有内容的那部分数据。

private transient Object[] elementData;

### Binder


####

## Android 中的 ICP 方式


### Bunble

Bunble 由于实现了 Parcelable 接口，所以它可以方便地在不同的进程间传输。一般用于在一个进程中启动另一个进程的 Activity、Service和 Receiver时，传输能够被序列化的数据。这是最简单的进程间通信方式。

### 文件共享

文件共享类似 Serializable 的本质，并且使用这种方式对文件格式是没要求的，只要读/写双方约定数据格式即可。但由于如果存在并发读/写的可能，可以导致读取的内容并不是最新的，因此如果使用该种方式需要妥善处理该问题（并发读写）。

### Messenger





# 参考资料

- 任玉刚. Android 开发艺术探究. 电子工业出版社, 2015.