


<!-- TOC -->

- [IO](#io)
  - [一、Java I/O](#%E4%B8%80java-io)
  - [二、NIO](#%E4%BA%8Cnio)
    - [2.1 通道](#21-%E9%80%9A%E9%81%93)
    - [2.2 缓冲区](#22-%E7%BC%93%E5%86%B2%E5%8C%BA)
  - [三、Okio](#%E4%B8%89okio)

<!-- /TOC -->
# IO

IO 的本质是程序内存与外界（文件或网络）进行数据交互的过程。

## 一、Java I/O

Java 的 I/O 大概可以分成以下几类：

- 磁盘操作：File
- 字节操作：InputStream 和 OutputStream
- 字符操作：Reader 和 Writer
- 对象操作：Serializable
- 网络操作：Socket
- 新的输入/输出：NIO

以一个 Demo 介绍他们之间的使用关系。读、写都有之相对应的写、读方法。

```java
public class IO {

    public static void main(String[] args) {
        writeJavaIO();
        System.out.println(readJavaIO());
    }

    private static String readJavaIO() {
        try (InputStream inputStream = new FileInputStream("./app/text.txt");
                // 将字节流转换成字符流。
                Reader reader = new InputStreamReader(inputStream);
                // 在每次读取数据中间加入缓冲区，减小每次读取都需要与文件直接交互的次数，从而提高性能。
                BufferedReader bufferedReader = new BufferedReader(reader)) {

            return bufferedReader.readLine();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private static void writeJavaIO() {
        try (BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(new FileOutputStream("./app/text.txt")))
        {
            // 字符
            bufferedOutputStream.write(111);
            // 手动将缓冲区数据（未满）写入文件中，
            // 大于缓冲区大小时，会在调用 write() 时自动刷入。
            bufferedOutputStream.flush();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

```
o
```

## 二、NIO

新的输入/输出 (NIO) 库是在 JDK 1.4 中引入的，弥补了原来的 I/O 的不足，提供了高速的、面向块的 I/O。

I/O 与 NIO 最重要的区别是数据打包和传输的方式，I/O 以流的方式处理数据，而 NIO 以块的方式处理数据。

面向流的 I/O 一次处理一个字节数据：一个输入流产生一个字节数据，一个输出流消费一个字节数据。为流式数据创建过滤器非常容易，链接几个过滤器，以便每个过滤器只负责复杂处理机制的一部分。不利的一面是，面向流的 I/O 通常相当慢。

面向块的 I/O 一次处理一个数据块，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

I/O 包和 NIO 已经很好地集成了，java.io.* 已经以 NIO 为基础重新实现了，所以现在它可以利用 NIO 的一些特性。例如，java.io.* 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。

### 2.1 通道

通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动(一个流必须是 InputStream 或者 OutputStream 的子类)，而通道是双向的，可以用于读、写或者同时用于读写。

通道包括以下类型：

- FileChannel：从文件中读写数据。
- DatagramChannel：通过 UDP 读写网络中数据。
- SocketChannel：通过 TCP 读写网络中数据。
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。

### 2.2 缓冲区

发送给一个通道的所有数据都必须首先放到缓冲区中，同样地，从通道中读取的任何数据都要先读到缓冲区中。也就是说，不会直接对通道进行读写数据，而是要先经过缓冲区。

缓冲区本质上是一个数组，它提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer

## 三、Okio

```java
public class IO {

    public static void main(String[] args) throws FileNotFoundException {
        whiteOkio();
        readOkio();
        whiteOkioOnlyBuffer();
    }

     private static void readOkio() {
        try (BufferedSource source = Okio.buffer(Okio.source(new File("./app/text.txt")))) {
            System.out.println(source.readUtf8());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void writeOkio() {
        try (BufferedSink sink = Okio.buffer(Okio.sink(new File("./app/text.txt")))){
            sink.writeUtf8("abab");
            sink.writeUtf8("333");
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void writeOkioOnlyBuffer() {
        try (Buffer buffer = new Buffer();
                DataOutputStream dataOutputStream = new DataOutputStream(buffer.outputStream());
                DataInputStream inputStream = new DataInputStream(buffer.inputStream())) {
            // 写入buffer，并直接从 buffer 读出，并没有经过文件。
            dataOutputStream.writeUTF("abab");
            dataOutputStream.writeBoolean(false);
            System.out.println("whiteOkioOnlyBuffer" + inputStream.readUTF());
            System.out.println("whiteOkioOnlyBuffer" + inputStream.readBoolean());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

```
abab333
whiteOkioOnlyBuffer     abab
whiteOkioOnlyBuffer     false
```
