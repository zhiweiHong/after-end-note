# Netty 入门

​		我们举个例子来入门Netty，就以时间服务器为例。服务器接收客户端的请求，一旦请求正确，则返回当前的时间。客户端负责发送消息。由于是入门案例，本案例不考虑TCP粘包/拆包问题。同时，我会根据例子对其中设计的Netty类进行相关的说明。

## `TimeServerNetty`

```java
public class TimeServerNetty {

    public void bind(int port) {
        // (1)
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();
		// (2)
        ServerBootstrap serverBootstrap = new ServerBootstrap();

        serverBootstrap.group(bossGroup, workGroup)
                .channel(NioServerSocketChannel.class)	// (3)
                .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new StringDecoder());
                        socketChannel.pipeline().addLast(new TimeServerHanlder());
                    }
                })
                .childOption(ChannelOption.SO_KEEPALIVE, true) // (5)
            	.option(ChannelOption.SO_BACKLOG, 1024)	; // (6)
        System.out.println("Server is start");
        try {
            // 绑定端口，同步等待成功
            ChannelFuture channelFuture = serverBootstrap.bind(port).sync(); // (7)
            // 等待服务端端口关闭
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 优雅的退出
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

1. `NioEventLoopGroup`是一个用于处理I/O操作的多线程时间轮询器。Netty提供多种`EventLoopGroup`用于实现不同的场景。在这个例子中，我们用到了两个`EventLoopGroup`。**boss，一般用于接受传入的连接**。**worker，一旦boss接受了连接并将接受的连接注册给worker，就处理已接受连接的流量**。使用多少个线程以及如何将它们映射到创建的通道`EventLoopGroup`的实现，甚至可以通过构造函数进行配置。
2. `ServerBootstrap`是用于创建服务的帮助类。你可以直接使用一个Channel创建服务。无论如何，请记住这是一个枯燥的过程，并且在大多数情况下中你不需要这么做。
3. 我们指定`NioServerSocketChannel`的class，这个class用于实例化一个接受传入的连接的Channel。
4. `ChannelInitializer`是一个用于帮助用户配置一个新的Channel的特殊处理器。用户可以添加类似于`TimeServerHanlder`去实现他们自己定义的逻辑。同时，这个地方也可以配置其他的处理器，包括`StringDecoder`(将传入的消息转为String的处理器)
5. 我们可以通过`childOption`来配置连接的相关参数。比如我们正在写的`TCP/IP` 服务端，因此我们允许设置socket选项，类似于`tcpNoDelay` 和 `keeplive`。请根据`ChannelOption`的`api`文档以及指定的`ChannelConfig`实现，来获取支持`ChannelOption`的相关概述。
6. `option()`用于接受输入连接的`NioServerSocketChannel`。 `childOption()`用于父级`ServerChannel`接受的通道，在这种情况下为`NioServerSocketChannel`。
7. 我们使用`bind()`方法来绑定端口。

### `TimeServerHanlder`

```java
public class TimeServerHanlder extends ChannelHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception { // (1)
        String req = (String) msg;
        System.out.println("The time server receive order: " + req);
        String currentTime = Constants.SEND_MESSAGE.equalsIgnoreCase(req) ?
                new Date().toString() : "BAD ORDER";
        ByteBuf byteBuf = Unpooled.copiedBuffer(currentTime.getBytes());
        ctx.write(byteBuf); // (4)
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception { // (2)
        ctx.flush(); // (5)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception { // (3)
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. `channelRead`方法用于读取客户端传入的信息。
2. `channelReadComplete`用于客户端读取完毕的回调。
3. `exceptionCaught`用户异常信息时回调。
4. 调用write()方法向通道中写入数据。
5. 调用flush()方法刷新所有缓冲区的信息，保证信息的发送。

## `TimeClientNetty`

```java
public class TimeClientNetty {
    public void connect(int port, String host) {
        EventLoopGroup group = new NioEventLoopGroup(); // (1)
        Bootstrap bootstrap = new Bootstrap();	// (2)
        bootstrap.group(group)
                .channel(NioSocketChannel.class) // (3)
                .handler(new ChannelInitializer<SocketChannel>() {	// (4)
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new StringDecoder());
                        socketChannel.pipeline().addLast(new TimeClientHandler());
                    }
                })
            	.option(ChannelOption.SO_KEEPALIVE, true); // (5)
        ChannelFuture channelFuture = bootstrap.connect(host, port); // (6)
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

1. 这里只需要指定一个`EventLoopGroup`,因为这个`eventLoopGroup`既是boss也是work。或者说，对于Client端没有work和boss的说法。
2. `Bootstrap` 类似于 `ServerBootstrap`，这是客户端的专属。
3. 同样的，`NioSocketChannel`和`NioServerSocketChannel`类似，`NioSocketChannel`用于创建一个客户端的`Channel`
4. 对于客户端来说，在handler中配置`ChannelInitializer`即可。
5. 客户端没有`childOption`。
6. 我们使用`connect()`来连接服务端。

### `TimeClientHandler`

```java
public class TimeClientHandler extends ChannelHandlerAdapter {

    private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());
    private ByteBuf messageBuf;

    public TimeClientHandler() {

    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception { // (1)
        byte[] messageBytes = Constants.SEND_MESSAGE.getBytes();
        // 此处不能使用Unpooled.copiedBuffer(messageBytes)
        // 因为会copiedBuffer会初始化管道
        messageBuf = Unpooled.buffer(messageBytes.length);
        messageBuf.writeBytes(messageBytes);
        ctx.writeAndFlush(messageBuf);
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

1. 当建立连接时的回调方法。此处是连接建立后，直接向服务端发送信息。

# 总结

- 本案例只用于入门使用。
- 再次声明，本案例未考虑TCP粘包/拆包问题。

# 参考网站

https://netty.io/wiki/user-guide-for-5.x.html#wiki-h3-10