


<!-- TOC -->

- [IO](#io)
    - [Java I/O](#java-io)
    - [Okio](#okio)

<!-- /TOC -->
# IO

IO 的本质是程序内存与外界（文件或网络）进行数据交互的过程。

## Java I/O

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

## Okio

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