

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
            // 手动将缓冲区数据（未满）写入文件中。
            bufferedOutputStream.flush();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```


## Okio