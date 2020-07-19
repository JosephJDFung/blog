# Netty已经提供的 ChannelHandler 和 Codec

Netty 提供了很多共同协议的编解码器和处理程序,您可以几乎“开箱即用”的使用他们,而无需花在相当乏味的基础设施问题。我们将探索这些工具和他们的好处。这包括支持 SSL/TLS,WebSocket 和 谷歌SPDY,通过数据压缩使 HTTP 有更好的性能。

- 使用 SSL/TLS 加密 Netty 程序
- 构建 Netty HTTP/HTTPS 程序
- 处理空闲连接和超时
- 解码分隔符和基于长度的协议
- 写大数据
- 序列化数据

## 使用 SSL/TLS 加密 Netty 程序

现在数据隐私问题受到越来越多人的关注，开发人员在进行开发的时候必须把这个问题考虑好。最基础的就是需要熟悉加密协议 SSL 和 TLS 等更上一层的其他协议来实现数据安全。作为一个 HTTPS 网站的用户，你是安全的。当然，这些协议是广泛不基于 http 的应用程序，例如安全SMTP（SMTPS）邮件服务，甚至于关系数据库系统。 

为了支持 SSL/TLS,Java 提供了 javax.net.ssl API 的类SslContext 和 SslEngine 使它相对简单的实现解密和加密。Netty 的利用该 API 命名 SslHandler 的 ChannelHandler 实现, 有一个内部 SslEngine 做实际的工作。

1. 加密的入站数据被 SslHandler 拦截，并被解密
2. 前面加密的数据被 SslHandler 解密
3. 平常数据传过 SslHandler
4. SslHandler 加密数据并它传递出站

SslHandler 使用 ChannelInitializer 添加到 ChannelPipeline。

```java
public class SslChannelInitializer extends ChannelInitializer<Channel> {

    private final SslContext context;
    private final boolean startTls;
    public SslChannelInitializer(SslContext context,
    boolean client, boolean startTls) {   //1 使用构造函数来传递 SSLContext 用于使用(startTls 是否启用)
        this.context = context;
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        SSLEngine engine = context.newEngine(ch.alloc());  //2 从 SslContext 获得一个新的 SslEngine 。给每个 SslHandler 实例使用一个新的 SslEngine
        engine.setUseClientMode(client); //3 设置 SslEngine 是 client 或者是 server 模式
        ch.pipeline().addFirst("ssl", new SslHandler(engine, startTls));  //4 添加 SslHandler 到 pipeline 作为第一个处理器
    }
}
```

在大多数情况下,SslHandler 将成为 ChannelPipeline 中的第一个 ChannelHandler 。这将确保所有其他 ChannelHandler 应用他们的逻辑到数据后加密后才发生,从而确保他们的变化是安全的。

SslHandler 有很多有用的方法。例如,在握手阶段两端相互验证,商定一个加密方法。您可以配置 SslHandler 修改其行为或提供 在SSL/TLS 握手完成后发送通知,这样所有数据都将被加密。 SSL/TLS 握手将自动执行

名称|描述
-|-
setHandshakeTimeout(...) setHandshakeTimeoutMillis(...) getHandshakeTimeoutMillis()|Set and get the timeout, after which the handshake ChannelFuture is notified of failure.
setCloseNotifyTimeout(...) setCloseNotifyTimeoutMillis(...) getCloseNotifyTimeoutMillis()|Set and get the timeout after which the close notify will time out and the connection will close. This also results in having the close notify ChannelFuture fail.
handshakeFuture()|Returns a ChannelFuture that will be notified once the handshake is complete. If the handshake was done before it will return a ChannelFuture that contains the result of the previous handshake.
close(...)|Send the close_notify to request close and destroy the underlying SslEngine.


## 构建 Netty HTTP/HTTPS 应用

最常见的一种协议是 HTTP/HTTPS ，尤其在智能手机上更是广泛使用。虽然我们可以通过HTTP或HTTPS访问一家公司的主页，但是它其实还有别的用处。就像许多组织会通过HTTP（S）来公开 WebService API ，这样做的目的是可以缓解独立平台带来的弊端。

>HTTP Decoder, Encoder 和 Codec

HTTP 是请求-响应模式，客户端发送一个 HTTP 请求，服务就响应此请求。Netty 提供了简单的编码、解码器来简化基于这个协议的开发工作。

![](https://atts.w3cschool.cn/attachments/image/20170808/1502160101936428.jpg)

1. HTTP Request 第一部分是包含的头信息
2. HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
3. LastHttpContent 标记是 HTTP request 的结束，同时可能包含头的尾部信息
4. 完整的 HTTP request

![](https://atts.w3cschool.cn/attachments/image/20170808/1502160084809431.jpg)

1. HTTP response 第一部分是包含的头信息
2. HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
3. LastHttpContent 标记是 HTTP response 的结束，同时可能包含头的尾部信息
4. 完整的 HTTP response

HTTP 请求/响应可能包含不止一个数据部分,它总是终止于 LastHttpContent 部分。FullHttpRequest 和FullHttpResponse 消息是特殊子类型,分别表示一个完整的请求和响应。所有类型的 HTTP 消息(FullHttpRequest ，LastHttpContent 以及那些如清单8.2所示)实现 HttpObject 接口。

将支持 HTTP 添加到应用程序。添加正确的 ChannelHandler 到 ChannelPipeline 中

```java
public class HttpPipelineInitializer extends ChannelInitializer<Channel> {

    private final boolean client;

    public HttpPipelineInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("decoder", new HttpResponseDecoder());  //1 client: 添加 HttpResponseDecoder 用于处理来自 server 响应
            pipeline.addLast("encoder", new HttpRequestEncoder());  //2 client: 添加 HttpRequestEncoder 用于发送请求到 server
        } else {
            pipeline.addLast("decoder", new HttpRequestDecoder());  //3 server: 添加 HttpRequestDecoder 用于接收来自 client 的请求
            pipeline.addLast("encoder", new HttpResponseEncoder());  //4 server: 添加 HttpResponseEncoder 用来发送响应给 client
        }
    }
}
```

>HTTP消息聚合

安装 ChannelPipeline 中的初始化之后,你能够对不同 HttpObject 消息进行操作。但由于 HTTP 请求和响应可以由许多部分组合而成，你需要聚合他们形成完整的消息。为了消除这种繁琐任务， Netty 提供了一个聚合器,合并消息部件到 FullHttpRequest 和 FullHttpResponse 消息。这样您总是能够看到完整的消息内容。

这个操作有一个轻微的成本,消息段需要缓冲,直到完全可以将消息转发到下一个 ChannelInboundHandler 管道。但好处是,你不必担心消息碎片。

实现自动聚合只需添加另一个 ChannelHandler 到 ChannelPipeline。

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {

    private final boolean client;

    public HttpAggregatorInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //1 client: 添加 HttpClientCodec
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //2 server: 添加 HttpServerCodec 作为我们是 server 模式时
        }
        pipeline.addLast("aggegator", new HttpObjectAggregator(512 * 1024));  //3 添加 HttpObjectAggregator 到 ChannelPipeline, 使用最大消息值是 512kb
    }
}
```

>HTTP 压缩

使用 HTTP 时建议压缩数据以减少传输流量，压缩数据会增加 CPU 负载，现在的硬件设施都很强大，大多数时候压缩数据时一个好主意。Netty 支持“gzip”和“deflate”，为此提供了两个 ChannelHandler 实现分别用于压缩和解压。

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {

    private final boolean isClient;
    public HttpAggregatorInitializer(boolean isClient) {
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient) {
            pipeline.addLast("codec", new HttpClientCodec()); //1 client: 添加 HttpClientCodec
            pipeline.addLast("decompressor",new HttpContentDecompressor()); //2 client: 添加 HttpContentDecompressor 用于处理来自服务器的压缩的内容
        } else { 
            pipeline.addLast("codec", new HttpServerCodec()); //3 server: HttpServerCodec
            pipeline.addLast("compressor",new HttpContentCompressor()); //4 server: HttpContentCompressor 用于压缩来自 client 支持的 HttpContentCompressor
        }
    }
}
```
Java 6或者更早版本，如果要压缩数据，需要添加 jzlib 到 classpath
```xml
<dependency>
    <groupId>com.jcraft</groupId>
        <artifactId>jzlib</artifactId>
    <version>1.1.3</version>
</dependency>
```

>使用 HTTPS

启用 HTTPS，只需添加 SslHandler

```java
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {

    private final SslContext context;
    private final boolean client;

    public HttpsCodecInitializer(SslContext context, boolean client) {
        this.context = context;
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        SSLEngine engine = context.newEngine(ch.alloc());
        pipeline.addFirst("ssl", new SslHandler(engine));  //1 添加 SslHandler 到 pipeline 来启用 HTTPS

        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //2 client: 添加 HttpClientCodec
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //3 server: 添加 HttpServerCodec ，如果是 server 模式的话
        }
    }
}
```

上面的代码就是一个很好的例子，解释了 Netty 的架构是如何让“重用”变成了“杠杆”。我们可以添加一个新的功能,甚至是一样重要的加密支持,几乎没有工作量,只需添加一个ChannelHandler 到 ChannelPipeline。

>WebSocket

HTTP 是不错的协议，但是如果需要实时发布信息怎么做？有个做法就是客户端一直轮询请求服务器，这种方式虽然可以达到目的，但是其缺点很多，也不是优秀的解决方案，为了解决这个问题，便出现了 WebSocket。

WebSocket 允许数据双向传输，而不需要请求-响应模式。早期的WebSocket 只能发送文本数据，然后现在不仅可以发送文本数据，也可以发送二进制数据，这使得可以使用 WebSocket 构建你想要的程序。下图是WebSocket 的通信示例图：

WebSocket 规范及其实现是为了一个更有效的解决方案。简单的说, 一个WebSocket 提供一个 TCP 连接两个方向的交通。结合 WebSocket API 它提供了一个替代 HTTP 轮询双向通信从页面到远程服务器。

也就是说,WebSocket 提供真正的双向客户机和服务器之间的数据交换。 我们不会对内部太多的细节,但我们应该提到,虽然最早实现仅限于文本数据，但现在不再是这样,WebSocket可以用于任意数据,就像一个正常的套接字。

- 在这种情况下的通信开始于普通 HTTP ，并“升级”为双向 WebSocket。

![](https://atts.w3cschool.cn/attachments/image/20170808/1502160124474125.jpg)

1. Client (HTTP) 与 Server 通讯
2. Server (HTTP) 与 Client 通讯
3. Client 通过 HTTP(s) 来进行 WebSocket 握手,并等待确认
3. 连接协议升级至 WebSocket

添加应用程序支持 WebSocket 只需要添加适当的客户端或服务器端WebSocket ChannelHandler 到管道。这个类将处理特殊 WebSocket 定义的消息类型,称为 帧。

名称|描述
-|-
BinaryWebSocketFrame|Data frame: binary data
TextWebSocketFrame|Data frame: text data
ContinuationWebSocketFrame|Data frame: text or binary data that belongs to a previous BinaryWebSocketFrame or TextWebSocketFrame
CloseWebSocketFrame|Control frame: a CLOSE request, close status code and a phrase
PingWebSocketFrame|Control frame: requests the send of a PongWebSocketFrame
PongWebSocketFrame|Control frame: sent as response to a PingWebSocketFrame

由于 Netty 的主要是一个服务器端技术重点在这里创建一个 WebSocket server 。使用WebSocketServerProtocolHandler 该类处理协议升级握手以及三个“控制”帧 Close, Ping 和 Pong。Text 和 Binary 数据帧将被传递到下一个处理程序(由你实现)进行处理。

```java
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(
                new HttpServerCodec(),
                new HttpObjectAggregator(65536),  //1 添加 HttpObjectAggregator 用于提供在握手时聚合 HttpRequest
                new WebSocketServerProtocolHandler("/websocket"),  //2 添加 WebSocketServerProtocolHandler 用于处理色好给你寄握手如果请求是发送到"/websocket." 端点，当升级完成后，它将会处理Ping, Pong 和 Close 帧
                new TextFrameHandler(),  //3 TextFrameHandler 将会处理 TextWebSocketFrames
                new BinaryFrameHandler(),  //4 BinaryFrameHandler 将会处理 BinaryWebSocketFrames
                new ContinuationFrameHandler());  //5 ContinuationFrameHandler 将会处理ContinuationWebSocketFrames
    }

    public static final class TextFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
            // Handle text frame
        }
    }

    public static final class BinaryFrameHandler extends SimpleChannelInboundHandler<BinaryWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, BinaryWebSocketFrame msg) throws Exception {
            // Handle binary frame
        }
    }

    public static final class ContinuationFrameHandler extends SimpleChannelInboundHandler<ContinuationWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, ContinuationWebSocketFrame msg) throws Exception {
            // Handle continuation frame
        }
    }
}
```

加密 WebSocket 只需插入 SslHandler 到作为 pipline 第一个 ChannelHandler

>SPDY

SPDY（读作“SPeeDY”）是Google 开发的基于 TCP 的应用层协议，用以最小化网络延迟，提升网络速度，优化用户的网络使用体验。SPDY 并不是一种用于替代 HTTP 的协议，而是对 HTTP 协议的增强。SPDY 实现技术：

- 压缩报头
- 加密所有
- 多路复用连接
- 提供支持不同的传输优先级

SPDY 主要有5个版本：

- 1 - 初始化版本，但没有使用
- 2 - 新特性，包含服务器推送
- 3 - 新特性包含流控制和更新压缩
- 3.1 - 会话层流程控制
- 4.0 - 流量控制，并与 HTTP 2.0 更加集成

SPDY 被很多浏览器支持，包括 Google Chrome, Firefox, 和 Opera

Netty 支持 版本 2 和 3 （包含3.1）的支持。这些版本被广泛应用，可以支持更多的用户。

## Netty检测空闲连接以及超时

为了能够及时的将资源释放出来，我们会检测空闲连接和超时。常见的方法是通过发送信息来测试一个不活跃的链接，通常被称为“心跳”，然后在远端确认它是否还活着。（还有一个方法是比较激进的，简单地断开那些指定的时间间隔的不活跃的链接）。

处理空闲连接是一项常见的任务,Netty 提供了几个 ChannelHandler 实现此目的。

名称|描述
-|-
IdleStateHandler|如果连接闲置时间过长，则会触发 IdleStateEvent 事件。在 ChannelInboundHandler 中可以覆盖 userEventTriggered(...) 方法来处理 IdleStateEvent。
ReadTimeoutHandler|在指定的时间间隔内没有接收到入站数据则会抛出 ReadTimeoutException 并关闭 Channel。ReadTimeoutException 可以通过覆盖 ChannelHandler 的 exceptionCaught(…) 方法检测到。
WriteTimeoutHandler|WriteTimeoutException 可以通过覆盖 ChannelHandler 的 exceptionCaught(…) 方法检测到。

当超过60秒没有数据收到时，就会得到通知，此时就发送心跳到远端，如果没有回应，连接就关闭。

```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));  //1 IdleStateHandler 将通过 IdleStateEvent 调用 userEventTriggered ，如果连接没有接收或发送数据超过60秒钟
        pipeline.addLast(new HeartbeatHandler());
    }

    public static final class HeartbeatHandler extends ChannelInboundHandlerAdapter {
        private static final ByteBuf HEARTBEAT_SEQUENCE = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("HEARTBEAT", CharsetUtil.ISO_8859_1));  //2 心跳发送到远端

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            if (evt instanceof IdleStateEvent) {
                 ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
                         .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);  //3 发送的心跳并添加一个侦听器，如果发送操作失败将关闭连接
            } else {
                super.userEventTriggered(ctx, evt);  //4 事件不是一个 IdleStateEvent 的话，就将它传递给下一个处理程序
            }
        }
    }
}
```

## Netty如何解码分隔符和基于长度的协议

>分隔符协议

经常需要处理分隔符协议或创建基于它们的协议，例如SMTP、POP3、IMAP、Telnet等等。Netty 附带的解码器可以很容易的提取一些序列分隔：

名称|描述
-|-
DelimiterBasedFrameDecoder|接收ByteBuf由一个或多个分隔符拆分，如NUL或换行符
LineBasedFrameDecoder|接收ByteBuf以分割线结束，如"\n"和"\r\n"

用 LineBasedFrameDecoder 处理

```java
public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(65 * 1024));   //1 添加一个 LineBasedFrameDecoder 用于提取帧并把数据包转发到下一个管道中的处理程序,在这种情况下就是 FrameHandler
        pipeline.addLast(new FrameHandler());  //2 添加 FrameHandler 用于接收帧
    }

    public static final class FrameHandler extends SimpleChannelInboundHandler<ByteBuf> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {  //3 每次调用都需要传递一个单帧的内容
            // Do something with the frame
        }
    }
}
```

使用 DelimiterBasedFrameDecoder 可以方便处理特定分隔符作为数据结构体的这类情况。

- 传入的数据流是一系列的帧，每个由换行（“\n”）分隔
- 每帧包括一系列项目，每个由单个空格字符分隔
- 一帧的内容代表一个“命令”：一个名字后跟一些变量参数

>Decoder for the command and the handler

```java
public class CmdHandlerInitializer extends ChannelInitializer<Channel> {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new CmdDecoder(65 * 1024));//1 添加一个 CmdDecoder 到管道；将提取 Cmd 对象和转发到在管道中的下一个处理器
        pipeline.addLast(new CmdHandler()); //2 添加 CmdHandler 将接收和处理 Cmd 对象
    }

    public static final class Cmd { //3 命令也是 POJO
        private final ByteBuf name;
        private final ByteBuf args;

        public Cmd(ByteBuf name, ByteBuf args) {
            this.name = name;
            this.args = args;
        }

        public ByteBuf name() {
            return name;
        }

        public ByteBuf args() {
            return args;
        }
    }

    public static final class CmdDecoder extends LineBasedFrameDecoder {
        public CmdDecoder(int maxLength) {
            super(maxLength);
        }

        @Override
        protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
            ByteBuf frame =  (ByteBuf) super.decode(ctx, buffer); //4 super.decode() 通过结束分隔从 ByteBuf 提取帧
            if (frame == null) {
                return null; //5 frame 是空时，则返回 null
            }
            int index = frame.indexOf(frame.readerIndex(), frame.writerIndex(), (byte) ' ');  //6 找到第一个空字符的索引。首先是它的命令名；接下来是参数的顺序
            return new Cmd(frame.slice(frame.readerIndex(), index), frame.slice(index +1, frame.writerIndex())); //7 从帧先于索引以及它之后的片段中实例化一个新的 Cmd 对象
        }
    }

    public static final class CmdHandler extends SimpleChannelInboundHandler<Cmd> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, Cmd msg) throws Exception {
            // Do something with the command  //8 处理通过管道的 Cmd 对象
        }
    }
}
```

- 类 Cmd - 存储帧的内容，其中一个 ByteBuf 用于存名字，另外一个存参数
- 类 CmdDecoder - 从重写方法 decode() 中检索一行，并从其内容中构建一个 Cmd 的实例
- 类 CmdHandler - 从 CmdDecoder 接收解码 Cmd 对象和对它的一些处理。

>基于长度的协议

基于长度的协议协议在帧头文件里定义了一个帧编码的长度,而不是结束位置用一个特殊的分隔符来标记。表8.6列出了 Netty 提供的两个解码器，用于处理这种类型的协议。

名称|描述
-|-
FixedLengthFrameDecoder|提取固定长度
LengthFieldBasedFrameDecoder|读取头部长度并提取帧的长度

LengthFieldBasedFrameDecoder 提供了几个构造函数覆盖各种各样的头长字段配置情况。清单8.10显示了使用三个参数的构造函数是maxFrameLength,lengthFieldOffset lengthFieldLength。在这 情况下,帧的长度被编码在帧的前8个字节。

```java
public class LengthBasedInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(
        new LengthFieldBasedFrameDecoder(65 * 1024, 0, 8)); //1 添加一个 LengthFieldBasedFrameDecoder ,用于提取基于帧编码长度8个字节的帧。
        pipeline.addLast(new FrameHandler()); //2 添加一个 FrameHandler 用来处理每帧
    }

    public static final class FrameHandler
        extends SimpleChannelInboundHandler<ByteBuf> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
        ByteBuf msg) throws Exception {
        // Do something with the frame 
        }
    }
}
```

## Netty如何编写大型数据

出于网络的原因，有一个特殊的问题需要我们思考，就是如何能够有效的在异步框架写大数据。因为写操作是非阻塞的，即使不能写出数据，也只是通知 ChannelFuture 完成了。每当发生这种情况，就必须停止写操作或面临内存耗尽的风险。所以在进行写操作的时候，会产生的大量的数据，在这种情况下我们要准备好处理因为连接远端缓慢而导致的延迟释放内存的问题。作为一个例子让我们考虑写一个文件的内容到网络。

我们讨论传输的时候，有提到 NIO 的“zero-copy（零拷贝）”功能，消除移动一个文件的内容从文件系统到网络堆栈的复制步骤。所有这一切发生在 Netty 的核心,因此所有所需的应用程序代码是使用 interface FileRegion 的实现,在 Netty 的API 文档中定义如下为一个通过 Channel 支持 zero-copy 文件传输的文件区域。

>通过 zero-copy 将文件内容从 FileInputStream 创建 DefaultFileRegion 并写入 使用 Channel

```java
FileInputStream in = new FileInputStream(file); //1 获取 FileInputStream
FileRegion region = new DefaultFileRegion(in.getChannel(), 0, file.length()); //2 创建一个新的 DefaultFileRegion 用于文件的完整长度

channel.writeAndFlush(region).addListener(new ChannelFutureListener() { //3 发送 DefaultFileRegion 并且注册一个 ChannelFutureListener
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        if (!future.isSuccess()) {
            Throwable cause = future.cause();
            // Do something
        }
    }
});
```

只是看到的例子只适用于直接传输一个文件的内容,没有执行的数据应用程序的处理。在相反的情况下,将数据从文件系统复制到用户内存是必需的,您可以使用 ChunkedWriteHandler。这个类提供了支持异步写大数据流不引起高内存消耗。

- 这个关键是 interface ChunkedInput

名称|描述
-|-
ChunkedFile|当你使用平台不支持 zero-copy 或者你需要转换数据，从文件中一块一块的获取数据
ChunkedNioFile|与 ChunkedFile 类似，处理使用了NIOFileChannel
ChunkedStream|从 InputStream 中一块一块的转移内容
ChunkedNioStream|从 ReadableByteChannel 中一块一块的转移内容

---

ChunkedStream,实现在实践中最常用。 所示的类被实例化一个 File 和一个 SslContext。当 initChannel() 被调用来初始化显示的处理程序链的通道。

当通道激活时，WriteStreamHandler 从文件一块一块的写入数据作为ChunkedStream。最后将数据通过 SslHandler 加密后传播。

```java
public class ChunkedWriteHandlerInitializer extends ChannelInitializer<Channel> {
    private final File file;
    private final SslContext sslCtx;

    public ChunkedWriteHandlerInitializer(File file, SslContext sslCtx) {
        this.file = file;
        this.sslCtx = sslCtx;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new SslHandler(sslCtx.createEngine()); //1 添加 SslHandler 到 ChannelPipeline.
        pipeline.addLast(new ChunkedWriteHandler());//2 添加 ChunkedWriteHandler 用来处理作为 ChunkedInput 传进的数据
        pipeline.addLast(new WriteStreamHandler());//3 当连接建立时，WriteStreamHandler 开始写文件的内容
    }

    public final class WriteStreamHandler extends ChannelInboundHandlerAdapter {  //4 当连接建立时，channelActive() 触发使用 ChunkedInput 来写文件的内容 (插图显示了 FileInputStream;也可以使用任何 InputStream )


        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            super.channelActive(ctx);
            ctx.writeAndFlush(new ChunkedStream(new FileInputStream(file)));
        }
    }
}
```
ChunkedInput 所有被要求使用自己的 ChunkedInput 实现，是安装ChunkedWriteHandler 在管道中

在本节中,我们讨论了

- 如何采用zero-copy（零拷贝）功能高效地传输文件
- 如何使用 ChunkedWriteHandler 编写大型数据而避免 OutOfMemoryErrors 错误。

## Netty序列化数据

在JDK中是使用了 ObjectOutputStream 和 ObjectInputStream 来通过网络将原始数据类型和 POJO 进行序列化和反序列化，API并不复杂，支持 java.io.Serializable 接口，但是它也不算是高效的。本节内容中，我们来看看 Netty 所提供的。  

>JDK 序列化

如果程序与端对端间的交互是使用 ObjectOutputStream 和 ObjectInputStream，并且主要面临的问题是兼容性，那么， JDK 序列化 是不错的选择。

名称|描述
-|-
CompatibleObjectDecoder|该解码器使用 JDK 序列化，用于与非 Netty 进行互操作。
CompatibleObjectEncoder|该编码器使用 JDK 序列化，用于与非 Netty 进行互操作。
ObjectDecoder|基于 JDK 序列化来使用自定义序列化解码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现
ObjectEncoder|基于 JDK 序列化来使用自定义序列化编码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现

>JBoss Marshalling 序列化

如果可以使用外部依赖 JBoss Marshalling 是个明智的选择。比 JDK 序列化快3倍且更加简练。

JBoss Marshalling 是另一个序列化 API,修复的许多 JDK序列化 API 中发现的问题,它与 java.io.Serializable 完全兼容。并添加了一些新的可调参数和附加功能,所有这些都可插入通过工厂配置外部化,类/实例查找表,类决议,对象替换,等等)

名称|描述
-|-
CompatibleMarshallingDecoder|为了与使用 JDK 序列化的端对端间兼容。
CompatibleMarshallingEncoder|为了与使用 JDK 序列化的端对端间兼容。
MarshallingDecoder|使用自定义序列化用于解码，必须使用MarshallingEncoder 
MarshallingEncoder | 使用自定义序列化用于编码，必须使用 MarshallingDecoder

>ProtoBuf 序列化

ProtoBuf 来自谷歌，并且开源了。它使编解码数据更加紧凑和高效。它已经绑定各种编程语言,使它适合跨语言项目。

名称|描述
-|-
ProtobufDecoder|使用 ProtoBuf 来解码消息
ProtobufEncoder|使用 ProtoBuf 来编码消息
ProtobufVarint32FrameDecoder|在消息的整型长度域中，通过 "Base 128 Varints"将接收到的 ByteBuf 动态的分割

```java
public class ProtoBufInitializer extends ChannelInitializer<Channel> {

    private final MessageLite lite;

    public ProtoBufInitializer(MessageLite lite) {
        this.lite = lite;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new ProtobufVarint32FrameDecoder());//添加 ProtobufVarint32FrameDecoder 用来分割帧
        pipeline.addLast(new ProtobufEncoder());//添加 ProtobufEncoder 用来处理消息的编码
        pipeline.addLast(new ProtobufDecoder(lite));//添加 ProtobufDecoder 用来处理消息的解码
        pipeline.addLast(new ObjectHandler());//添加 ObjectHandler 用来处理解码了的消息
    }

    public static final class ObjectHandler extends SimpleChannelInboundHandler<Object> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
            // Do something with the object
        }
    }
}
```