# NIO 入门笔记

​		NIO(Non-blocking I/O)，在Java领域也被称为New I/O，是一种同步非阻塞的I/O模型，也是I/O多路复用的基础。

## 缓冲区 Buffer

​		Buffer是一个对象，包含一些要写入或者要读出的数据。

​		在原先的IO中，所有的数据是面向流的，是单向的。

​		而在NIO中，所有的数据都是通过缓冲区处理的。在读取数据的时候，是直接读取到缓冲区中，在写入数据时，写入缓冲区中。

​		最常用的缓冲区类型是ByteBuffer。一个ByteBuffer可以在其底层字节数组上进行get/ser操作

## 通道 Channel

​		什么是通道？简而言之，是用于运输数据的管道。和流不同，通道是双向的，可以读也可以写。通道把数据从缓冲区中读取，也可以向缓冲区写入数据。

## 栗子

```java
package nio;

import java.io.*;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.util.Objects;

public class NIOFileDemo {
    private static final String PATH = "E:\\after-end-note\\NIO\\test.txt";
    public static void main(String[] args) throws Exception {
        writeFile(PATH,"\nI Love NaNa",true);
        readFile(PATH);
    }

    public static void readFile(String path) {
        FileInputStream inputStream = null;
        FileChannel channel = null;
        try {
            inputStream = new FileInputStream(new File("E:\\after-end-note\\NIO\\test.txt"));
            channel = inputStream.getChannel();

            // create ByteBuffer
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            buffer.clear();
            channel.read(buffer);

            buffer.flip();
            byte[] result = new byte[buffer.remaining()];
            buffer.get(result);
            System.out.println(new String(result, "UTF-8"));
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (Objects.nonNull(inputStream)) {
                    inputStream.close();
                }
                if (Objects.nonNull(channel)) {
                    channel.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void writeFile(String path, String msg, Boolean append) {
        FileChannel dest = null;
        try {
            dest = new FileOutputStream(new File(path), append).getChannel();
            ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
            writeBuffer.put(msg.getBytes());
            writeBuffer.flip();
            dest.write(writeBuffer);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (Objects.nonNull(dest)) {
                    dest.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```