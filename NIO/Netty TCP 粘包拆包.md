# Netty TCP 粘包/拆包

## TCP 粘包/拆包

### 概念

​		TCP底层并不了解上层业务数据，会根据TCP缓冲区的实际情况进行包的划分，所以在业务上认为一个完整的包可能会被TCP拆分成多个包进行发送，也可能把多个小的包封装成一个包发送。前者被称为拆包，后者被称为粘包。

## 发生原因

1. 应用程序write写入的字节大小大于socket缓冲区大小。

2. 进行MSS大小的TCP分段。

3. 以太网帧的payload大于MTU进行IP分片

   ![image-20200702172751068](.\TCP粘包原理.png)

## 解决方法

1. 消息定长，例如每个报文的大小为固定长度200字节，如果不够，空位补空格。
2. 在包的尾部增加回车换行符进行分割。
3. 将消息分为消息头和消息体，消息头中包含表示消息总长度（或者消息体长度）的字段。通常设计思路为消息头的第一个字段使用int32来表示消息的总长度
4. 更复杂的应用层协议

## 案例

​		我们可以快速的搭建一个TCP粘包案例，来对上述的概念进行验证。

### TimeServerNetty

```java
public class TimeServerNetty {

    public void bind(int port) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();

        serverBootstrap.group(bossGroup, workGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new StringDecoder());
                        socketChannel.pipeline().addLast(new TimeServerHanlder());
                    }
                })
                .childOption(ChannelOption.SO_KEEPALIVE, true);
        System.out.println("Server is start");
        try {
            ChannelFuture channelFuture = serverBootstrap.bind(port).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        TimeServerNetty serverNetty = new TimeServerNetty();
        int port = 8080;
        serverNetty.bind(port);
    }
}
```

### TimeServerHanlder

```java
public class TimeServerHanlder extends ChannelHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String req = (String) msg;
        System.out.println("The time server receive order: " + req);
        String currentTime = Constants.SEND_MESSAGE.equalsIgnoreCase(req) ?
                new Date().toString() : "BAD ORDER";
        currentTime += System.getProperty("line.separator");
        ByteBuf byteBuf = Unpooled.copiedBuffer(currentTime.getBytes());
        ctx.write(byteBuf);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

### TimeClientNetty

```java
public class TimeClientNetty {
    public void connect(int port, String host) {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new StringDecoder());
                        socketChannel.pipeline().addLast(new TimeClientHandler());
                    }
                });
        ChannelFuture channelFuture = bootstrap.connect(host, port);
        try {
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        TimeClientNetty client = new TimeClientNetty();
        client.connect(8080,"127.0.0.1");

    }
}
```

### TimeClientHandler

```java
public class TimeClientHandler extends ChannelHandlerAdapter {

    private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());
    private ByteBuf messageBuf;

    public TimeClientHandler() {

    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        byte[] messageBytes = (Constants.SEND_MESSAGE + System.getProperty("line.separator")).getBytes();
        for (int i = 0; i < 100; i++) {
            messageBuf = Unpooled.buffer(messageBytes.length);
            messageBuf.writeBytes(messageBytes);
            ctx.writeAndFlush(messageBuf);
        }
        logger.info("send message");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String response = (String) msg;
        System.out.println("get response " + response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        logger.warning("throw " + cause.getLocalizedMessage());
        ctx.close();
    }
}
```

### 分析

​		我们在客户端连接上服务端的时候，一次性发送100个信息给服务端。按照未出异常的情况，服务端应该接受100条信息，并且客户端返回100条消息。然而结果真的是这样子吗？

​		服务端接受的消息

```shell
Server is start
The time server receive order: QUERY TIME ORDER
QUERY TIME ORDER
... more
The time server receive order: 
QUERY TIME ORDER
... more
```

​		实际上，服务端只接受了2条信息。而客户端呢？客户端会接受两条信息吗？

​		客户端接受信息：

```shell
get response BAD ORDERBAD ORDER
```

​		这个案例很典型。客户端发送的消息占用字节数不多，TCP将这些消息粘起来，等到缓冲区满了之后再发送出去。同样的，服务端返回给客户端的消息占用的字节并不多，于是也是拼接到一起，所以只返回了一条数据。

### Netty 解决

​		由于代码过分冗长，我只贴出修改的位置以及对应的类。Netty主要是使用LineBasedFrameDecoder类。只需要在`ChannelInitializer`相关的位置进行添加这个处理类就可以了。

#### TimeServerNetty - bind()

```java
public void bind(int port) {
        ... more
        serverBootstrap.group(bossGroup, workGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new LineBasedFrameDecoder(1024));
                        socketChannel.pipeline().addLast(new StringDecoder());
                        socketChannel.pipeline().addLast(new TimeServerHanlder());
                    }
                })
                .childOption(ChannelOption.SO_KEEPALIVE, true);
       ... more
    }
```

#### TimeClientNetty - connect()

```java
public void connect(int port, String host) {
    ... more
    bootstrap.group(group)
            .channel(NioSocketChannel.class)
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel socketChannel) throws Exception {
                    socketChannel.pipeline().addLast(new LineBasedFrameDecoder(1024));
                    socketChannel.pipeline().addLast(new StringDecoder());
                    socketChannel.pipeline().addLast(new TimeClientHandler());
                }
            });
    ... more
}
```

## 总结

1. 发送的消息务必以回车符作为结尾，否则即便是Netty的LineBasedFrameDecoder也不会接受消息。对应的[解决方法](#解决方法)的`在包的尾部增加回车换行符进行分割`
2. LineBasedFrameDecoder(1024)传入的就是字节的最大长度，对应的[解决方法](#解决方法)的`消息定长`
3. 这是Netty的入门解决方法，后续Netty还提供特殊的分隔符解决方案。