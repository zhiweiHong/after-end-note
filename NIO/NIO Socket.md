# NIO Socket

对于NIO Socket而言，同样也有Buffer和Channel。与此同时，NIO Socket还引入了一个新的概念——Selector

## Selector

这个是Java NIO编程的基础。多路复用器提供选择已经就绪的任务的能力。简单的来说，Selector会不断的轮询其上的Channel，如果某个Channel上面有新的TCP连接接入、读和写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。

## NIO 服务端

### 通讯序列图

![image-20200701190337710](.\NIO服务端通信序列图.png)

我将根据序列图进行相关的步骤的说明：

1. 创建一个ServerSocketChannel。`ServerSocketChannel.open()`
2. 绑定一个端口，并且设置为非阻塞模式
3. 创建一个Selector
4. 在selector上注册channel。`serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT)`
5. 轮询就绪的Channel，得到对应的Key
6. 根据得到的SelectorKey进行相应的处理

### 服务端的栗子

以时间服务器为例。当服务器接受到`QUERY TIME ORDER`指令时，返回当前时间，否则返回`BAD ORDER`

#### TimeServer

```java
public class TimeServer {
    public static void main(String[] args) {
        MutliTimerServerHandler mutliTimerServerHandler = new MutliTimerServerHandler(8080);
        new Thread(mutliTimerServerHandler).start();
    }
}
```

#### MutliTimerServerHandler

```java
public class MutliTimerServerHandler implements Runnable {
    private Selector selector;

    private ServerSocketChannel serverChannel;

    public MutliTimerServerHandler(int port) {
        try {
            // 建立Selector
            selector = Selector.open();
            // 创建服务端Socket通道
            serverChannel = ServerSocketChannel.open();
            // 设置为非阻塞模式
            serverChannel.configureBlocking(false);
            // 绑定端口
            serverChannel.socket().bind(new InetSocketAddress(port), 1024);
            // 建立联系
            serverChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("The time server is start in port :" + port);
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }
    
    @SneakyThrows
    @Override
    public void run() {
        while (true) {
            // 会阻塞, 直到注册在 Selector 中的 Channel 发送可读写事件
            selector.select(100);
            // 获取连接
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> itor = selectionKeys.iterator();
            SelectionKey key = null;
            while (itor.hasNext()) {
                key = itor.next();
                try {
                    handleInput(key);
                } catch (IOException e) {
                    if (key != null) {
                        key.cancel();
                        if (key.channel() != null) {
                            key.channel().close();
                        }
                    }
                }
                itor.remove();
            }

        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            if (key.isAcceptable()) {
                ServerSocketChannel serverSocketChannel1 = (ServerSocketChannel) key.channel();
                // 获取客户端 channel
                SocketChannel socketChannel = serverSocketChannel1.accept();
                socketChannel.configureBlocking(false);
                // 选择器注册监听的事件，同时制定关联的附件
                socketChannel.register(key.selector(), SelectionKey.OP_READ | SelectionKey.OP_WRITE,
                        ByteBuffer.allocate(1024));
            }
            if (key.isReadable()) {
                // 读取代码
                SocketChannel sc = (SocketChannel) key.channel();
                // 获取对应的附件信息
                ByteBuffer buffer = (ByteBuffer) key.attachment();
                long readBytes = sc.read(buffer);

                //客户端关闭的链接。可以安全关闭
                if (readBytes == -1) {
                    sc.close();
                } else {
                    // 缓冲区读取到了数据，将其感兴趣的操作设置为可读可写。
                    key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                    // 打印读取的内容
                    buffer.flip();
                    byte[] bytes = new byte[buffer.remaining()];
                    buffer.get(bytes);
                    String body = new String(bytes, "UTF-8");
                    System.out.println("The time server receive order : " + body);
                    String currentTime = Constants.SEND_MESSAGE.equalsIgnoreCase(body) ? 
                        new Date().toString() :	"BAD ORDER";
                    doWrite(sc, currentTime);
                }
            }

        }
    }
	// 返回客户端消息
    private void doWrite(SocketChannel channel, String response) throws IOException {
        if (Objects.nonNull(response)) {
            byte[] bytes = response.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer);
        }
    }
}
```

## NIO 客户端

### 通讯序列图

![image-20200701191957115](.\NIO客户端通信序列图.png)

我将根据序列图进行相关的步骤的说明：

1. 打开SocketChannel，设置参数为非阻塞
2. 根据host，port连接服务器
3. 创建Selector
4. 在selector上注册SocketChannel，绑定OP_CONNECT事件
5. 轮询就绪的Channel
6. 向服务器发送信息
7. 等待服务器返回消息

#### 客户端的栗子

同样是以时间服务器为例。客户端创建完毕之后就向服务器发送消息，打印服务器返回的消息。

#### TImeClient

```java
public class TimeClient {
    public static void main(String[] args) {
        new Thread(new TimeClientHandle("127.0.0.1", 8080)).start();
    }
}
```

#### TimeClientHandle

```java
public class TimeClientHandle implements Runnable {

    private volatile boolean stop = false;
    private Selector selector;
    private SocketChannel socketChannel;
    private String host;
    private Integer port;

    public TimeClientHandle(String host, Integer port) {
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
            this.host = host;
            this.port = port;
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        try {
            doConnect();
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }

        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> itor = keys.iterator();
                SelectionKey key = null;
                while (itor.hasNext()) {
                    key = itor.next();
                    itor.remove();
                    handleInput(key);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            SocketChannel sc = (SocketChannel) key.channel();
            sc.configureBlocking(false);
            // 选择器注册监听的事件，同时制定关联的附件
            sc.register(key.selector(), SelectionKey.OP_READ ,
                    ByteBuffer.allocate(1024));
            if (key.isConnectable()) {
                if (sc.finishConnect()) {
                    doWrite(sc);
                } else {
                    System.exit(1);
                }
            }
            if (key.isReadable()) {
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, "UTF-8");
                    System.out.println("Now is :" + body);
                    this.stop = true;
                } else if (readBytes < 0) {
                    key.cancel();
                    sc.close();
                }
            }

        }
    }

    private void doWrite(SocketChannel sc) throws IOException {
        sc.register(selector, SelectionKey.OP_READ);
        byte[] req = Constants.SEND_MESSAGE.getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        sc.write(writeBuffer);
        if (!writeBuffer.hasRemaining()) {
            System.out.println("Send order 2 server succeed");
        }
    }

    private void doConnect() throws IOException, InterruptedException {
        if (socketChannel.connect(new InetSocketAddress(host, port))) {
            doWrite(socketChannel);
        } else {
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
    }
}
```

## 总结

- NIO解决了Java IO阻塞的问题。
- NIO整体的代码极为繁琐，对于开发而言很不友好。
- Selector是整个NIO的核心。