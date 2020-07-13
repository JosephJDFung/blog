# Netty核心功能

## 不用 Netty 实现 I/O

阻塞 IO 实现
```java
public class PlainOioServer {

    public void serve(int port) throws IOException {
        final ServerSocket socket = new ServerSocket(port);     //1 绑定服务器到指定的端口。
        try {
            for (;;) {
                final Socket clientSocket = socket.accept();    //2 接受一个连接。
                System.out.println("Accepted connection from " + clientSocket);

                new Thread(new Runnable() {                        //3 创建一个新的线程来处理连接。
                    @Override
                    public void run() {
                        OutputStream out;
                        try {
                            out = clientSocket.getOutputStream();
                            out.write("Hi!\r\n".getBytes(Charset.forName("UTF-8")));                            //4 将消息发送到连接的客户端。
                            out.flush();
                            clientSocket.close();                //5 一旦消息被写入和刷新时就 关闭连接。

                        } catch (IOException e) {
                            e.printStackTrace();
                            try {
                                clientSocket.close();
                            } catch (IOException ex) {
                                // ignore on close
                            }
                        }
                    }
                }).start();                                        //6 启动线程。
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

##  NIO 实现的例子

```java
public class PlainNioServer {
    public void serve(int port) throws IOException {
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        ServerSocket ss = serverChannel.socket();
        InetSocketAddress address = new InetSocketAddress(port);
        ss.bind(address);                                            //1 绑定服务器到制定端口
        Selector selector = Selector.open();                        //2 打开 selector 处理 channel
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);    //3 注册 ServerSocket 到 ServerSocket ，并指定这是专门意接受 连接。
        final ByteBuffer msg = ByteBuffer.wrap("Hi!\r\n".getBytes());
        for (;;) {
            try {
                selector.select();                                    //4 等待新的事件来处理。这将阻塞，直到一个事件是传入。
            } catch (IOException ex) {
                ex.printStackTrace();
                // handle exception
                break;
            }
            Set<SelectionKey> readyKeys = selector.selectedKeys();    //5 从收到的所有事件中 获取 SelectionKey 实例。
            Iterator<SelectionKey> iterator = readyKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                try {
                    if (key.isAcceptable()) {                //6 检查该事件是一个新的连接准备好接受。
                        ServerSocketChannel server =
                                (ServerSocketChannel)key.channel();
                        SocketChannel client = server.accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_WRITE |
                                SelectionKey.OP_READ, msg.duplicate());    //7 接受客户端，并用 selector 进行注册。
                        System.out.println(
                                "Accepted connection from " + client);
                    }
                    if (key.isWritable()) {                //8 检查 socket 是否准备好写数据。
                        SocketChannel client =
                                (SocketChannel)key.channel();
                        ByteBuffer buffer =
                                (ByteBuffer)key.attachment();
                        while (buffer.hasRemaining()) {
                            if (client.write(buffer) == 0) {        //9 将数据写入到所连接的客户端。如果网络饱和，连接是可写的，那么这个循环将写入数据，直到该缓冲区是空的。
                                break;
                            }
                        }
                        client.close();                    //10 关闭连接。
                    }
                } catch (IOException ex) {
                    key.cancel();
                    try {
                        key.channel().close();
                    } catch (IOException cex) {
            
                    }
                }
            }
        }
    }
}
```

## 采用 Netty 实现 I/O 和 NIO

```java
public class NettyOioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
        EventLoopGroup group = new OioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();        //1 创建一个 ServerBootstrap
 
            b.group(group)                                    //2 使用 NioEventLoopGroup 允许非阻塞模式（NIO）
             .channel(OioServerSocketChannel.class)
             .localAddress(new InetSocketAddress(port))
             .childHandler(new ChannelInitializer<SocketChannel>() {//3 指定 ChannelInitializer 将给每个接受的连接调用
                 @Override
                 public void initChannel(SocketChannel ch) 
                     throws Exception {
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {            //4 添加的 ChannelHandler 拦截事件，并允许他们作出反应
                         @Override
                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
                             ctx.writeAndFlush(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);//5 写信息到客户端，并添加 ChannelFutureListener 当一旦消息写入就关闭连接
                         }
                     });
                 }
             });
            ChannelFuture f = b.bind().sync();  //6 绑定服务器来接受连接
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();        //7 释放所有资源
        }
    }
}

```

## Netty NIO 版本

Netty NIO 只改变一行代码，就从 OIO 传输 切换到了 NIO。

```java

public class NettyNioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();    //1 创建一个 ServerBootstrap
            b.group(new NioEventLoopGroup(), new NioEventLoopGroup())   //2 使用 NioEventLoopGroup 允许非阻塞模式（NIO）
             .channel(NioServerSocketChannel.class)
             .localAddress(new InetSocketAddress(port))
             .childHandler(new ChannelInitializer<SocketChannel>() {    //3 指定 ChannelInitializer 将给每个接受的连接调用
                 @Override
                 public void initChannel(SocketChannel ch) 
                     throws Exception {
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {    //4 添加的 ChannelInboundHandlerAdapter() 接收事件并进行处理
                         @Override
                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
                             ctx.writeAndFlush(buf.duplicate())                //5 写信息到客户端，并添加 ChannelFutureListener 当一旦消息写入就关闭连接
                                .addListener(ChannelFutureListener.CLOSE);
                         }
                     });
                 }
             });
            ChannelFuture f = b.bind().sync();                    //6 绑定服务器来接受连接
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();                    //7 释放所有资源
        }
    }
}

```

Netty使用的是统一的API，所以 Netty 中实现的每个传输都是用了同样的API，你使用什么来实现并不在它的关心范围内。Netty 通过操作接口Channel 、ChannelPipeline 和 ChannelHandler来实现。