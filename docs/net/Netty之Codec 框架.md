# Netty之Codec 框架

编码和解码——数据从一种特定协议格式到另一种格式的转换。这种处理模式是由通常被称为“codecs(编解码器)”的组件来处理的。Netty提供了一些组件,利用它们可以很容易地为各种不同协议编写编解码器。

codec的组成部分有两个，分别是：decoder(解码器)和encoder(编码器)。

“消息”是一个结构化的字节序列，语义为一个特定的应用程序——它的“数据”。encoder 是组件,转换消息格式适合传输(就像字节流),而相应的 decoder 转换传输数据回到程序的消息格式。逻辑上,“从”消息转换来是当作操作 outbound（出站）数据,而转换“到”消息是处理 inbound（入站）数据。

解码器负责将消息从字节或其他序列形式转成指定的消息对象，编码器的功能则相反；解码器负责处理“入站”数据，编码器负责处理“出站”数据。编码器和解码器的结构很简单，消息被编码后解码后会自动通过ReferenceCountUtil.release(message)释放，如果不想释放消息可以使用ReferenceCountUtil.retain(message)，这将会使引用数量增加而没有消息发布，大多数时候不需要这么做。

## Netty提供的Decoder(解码器)

Netty 提供了丰富的解码器抽象基类，我们可以很容易的实现这些基类来自定义解码器。主要分两类：

- 解码字节到消息（ByteToMessageDecoder 和 ReplayingDecoder）
- 解码消息到消息（MessageToMessageDecoder）

decoder 负责将“入站”数据从一种格式转换到另一种格式，Netty的解码器是一种 ChannelInboundHandler 的抽象实现。实践中使用解码器很简单，就是将入站数据转换格式后传递到 ChannelPipeline 中的下一个ChannelInboundHandler 进行处理；这样的处理是很灵活的，我们可以将解码器放在 ChannelPipeline 中，重用逻辑。


>ByteToMessageDecoder

ByteToMessageDecoder 是用于将字节转为消息（或其他字节序列）。

你不能确定远端是否会一次发送完一个完整的“信息”,因此这个类会缓存入站的数据,直到准备好了用于处理。

方法名称|描述
-|-
Decode|这是唯一需要实现的抽象方法。 通过具有输入字节的ByteBuf和添加了解码消息的List来调用它。 反复调用encode（），直到列表返回时为空。 然后将List的内容传递到管道中的下一个处理程序。
decodeLast|所提供的默认实现只调用了decode（）。当Channel变为非活动状态时，此方法被调用。

假设我们接收一个包含简单整数的字节流,每个都单独处理。在本例中,我们将从入站 ByteBuf 读取每个整数并将其传递给 pipeline 中的下一个ChannelInboundHandler。“解码”字节流成整数我们将扩展ByteToMessageDecoder，实现类为“ToIntegerDecoder”

```java
public class ToIntegerDecoder extends ByteToMessageDecoder {  //1 实现继承了 ByteToMessageDecode 用于将字节解码为消息

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        if (in.readableBytes() >= 4) {  //2 检查可读的字节是否至少有4个 ( int 是4个字节长度)
            out.add(in.readInt());  //3 从入站 ByteBuf 读取 int ， 添加到解码消息的 List 中
        }
    }
}
```

尽管 ByteToMessageDecoder 简化了这个模式,你会发现它还是有点烦人,在实际的读操作(这里 readInt())之前，必须要验证输入的 ByteBuf 要有足够的数据。

一旦一个消息被编码或解码它自动被调用ReferenceCountUtil.release(message) 。如果稍后还需要用到这个引用而不是马上释放,可以调用 ReferenceCountUtil.retain(message)。这将增加引用计数,防止消息被释放。

>ReplayingDecoder

ReplayingDecoder 是 byte-to-message 解码的一种特殊的抽象基类，读取缓冲区的数据之前需要检查缓冲区是否有足够的字节，使用ReplayingDecoder就无需自己检查；若ByteBuf中有足够的字节，则会正常读取；若没有足够的字节则会停止解码。

ByteToMessageDecoder 和 ReplayingDecoder

`注意到 ReplayingDecoder 继承自 ByteToMessageDecoder` 

也正因为这样的包装使得 ReplayingDecoder 带有一定的局限性：

- 不是所有的标准 ByteBuf 操作都被支持，如果调用一个不支持的操作会抛出 UnreplayableOperationException
- ReplayingDecoder 略慢于 ByteToMessageDecoder

如果不引入过多的复杂性 使用 ByteToMessageDecoder 。否则,使用ReplayingDecoder。

```java
public class ToIntegerDecoder2 extends ReplayingDecoder<Void> {   //1 实现继承自 ReplayingDecoder 用于将字节解码为消息

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        out.add(in.readInt());  //2 从入站 ByteBuf 读取整型，并添加到解码消息的 List 中
    }
}
```

>MessageToMessageDecoder

用于从一种消息解码为另外一种消息（例如，POJO 到 POJO）

 Integer to String
```java
public class IntegerToStringDecoder extends
        MessageToMessageDecoder<Integer> { //1 实现继承自 MessageToMessageDecoder

    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out)
            throws Exception {
        out.add(String.valueOf(msg)); //2 通过 String.valueOf() 转换 Integer 消息字符串
    }
}
```
>在解码时处理大的帧

Netty 是异步框架需要缓冲区字节在内存中,直到你能够解码它们。因此,你不能让你的解码器缓存太多的数据以免耗尽可用内存。为了解决这个共同关心的问题， Netty 提供了一个 TooLongFrameException ,通常由解码器在帧太长时抛出。

为了避免这个问题,你可以在你的解码器里设置一个最大字节数阈值,如果超出,将导致 TooLongFrameException 抛出(并由 ChannelHandler.exceptionCaught() 捕获)。然后由译码器的用户决定如何处理它。虽然一些协议,比如 HTTP、允许这种情况下有一个特殊的响应,有些可能没有,事件唯一的选择可能就是关闭连接。

```java
public class SafeByteToMessageDecoder extends ByteToMessageDecoder {  //1 实现继承 ByteToMessageDecoder 来将字节解码为消息
    private static final int MAX_FRAME_SIZE = 1024;

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
                       List<Object> out) throws Exception {
        int readable = in.readableBytes();
        if (readable > MAX_FRAME_SIZE) { //2 检测缓冲区数据是否大于 MAX_FRAME_SIZE
            in.skipBytes(readable);        //3 忽略所有可读的字节，并抛出 TooLongFrameException 来通知 ChannelPipeline 中的 ChannelHandler 这个帧数据超长
            throw new TooLongFrameException("Frame too big!");
        }
        // do something
    }
}
```

>更多 Decoder

io.netty.handler.codec.LineBasedFrameDecoder 通过结束控制符("\n" 或 "\r\n").解析入站数据。 

io.netty.handler.codec.http.HttpObjectDecoder 用于 HTTP 数据解码

## Netty的Encoder(编码器)

我们对encoder有了定义，它是用来把出站数据从一种格式转换到另外一种格式，因此它实现了 ChanneOutboundHandler 。就像decoder一样，Netty 也为你提供了一组类来写 encoder ，当然这些类提供的是与 decoder 完全相反的方法，如下所示：

- 编码从消息到字节
- 编码从消息到消息

>MessageToByteEncoder

之前我们学习了如何使用 ByteToMessageDecoder 来将字节转换成消息，现在我们使用 MessageToByteEncoder 实现相反的效果。

方法名称|描述
-|-
encode| 它与出站消息一起调用，该类将编码为ByteBuf。 然后，将ByteBuf转发到ChannelPipeline中的下一个ChannelOutboundHandler。

这个类只有一个方法，而 decoder 却是有两个，原因就是 decoder 经常需要在 Channel 关闭时产生一个“最后的消息”。出于这个原因，提供了decodeLast()，而 encoder 没有这个需求。

-  encode shorts into a ByteBuf

```java
public class ShortToByteEncoder extends
        MessageToByteEncoder<Short> {  //1 实现继承自 MessageToByteEncoder
    @Override
    public void encode(ChannelHandlerContext ctx, Short msg, ByteBuf out)
            throws Exception {
        out.writeShort(msg);  //2 写 Short 到 ByteBuf
    }
}
```
Netty 提供很多 MessageToByteEncoder 类来帮助你的实现自己的 encoder 。其中 WebSocket08FrameEncoder 就是个不错的范例。可以在 io.netty.handler.codec.http.websocketx 包找到。

>MessageToMessageEncoder

将入站数据从一个消息格式解码成另一个格式。现在我们需要一种方法来将出站数据从一种消息编码成另一种消息。

- integer to string

```java
public class IntegerToStringEncoder extends
        MessageToMessageEncoder<Integer> { //1 实现继承自 MessageToMessageEncoder

    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, List<Object> out)
            throws Exception {
        out.add(String.valueOf(msg));  //2 转 Integer 为 String，并添加到 MessageBuf
    }
}
```

>Netty抽象 Codec(编解码器)类

我们在讨论解码器和编码器的时候，都是把它们当成不同的实体的，但是有时候如果在同一个类中同时放入入站和出站的数据和信息转换的话，发现会更加实用。而Netty中的抽象Codec（编解码器）类就能达到这个目的，它们成对地组合解码器和编码器，以此提供对于字节和消息都相同的操作（这些类实现了 ChannelInboundHandler 和 ChannelOutboundHandler ）。

您可能想知道是否有时候使用单独的解码器和编码器会比使用这些组合类要好，最简单的答案是，紧密耦合的两个函数减少了他们的可重用性，但是把他们分开实现就会更容易扩展。当我们研究抽象编解码器类时，我们也会拿它和对应的独立的解码器和编码器做对比。

>ByteToMessageCodec

ByteToMessageCodec 将为我们处理这个问题,因为它结合了ByteToMessageDecoder 和 MessageToByteEncoder。

什么会是一个好的 ByteToMessageCodec 用例?任何一个请求/响应协议都可能是,例如 SMTP。编解码器将读取入站字节并解码到一个自定义的消息类型 SmtpRequest 。当接收到一个 SmtpResponse 会产生,用于编码为字节进行传输。

>MessageToMessageCodec

MessageToMessageEncoder 从一个消息格式转换到另一个地方。

MessageToMessageCodec 是一个参数化的类，定义如下：

```java
public abstract class MessageToMessageCodec<INBOUND,OUTBOUND>
```

```java
protected abstract void encode(ChannelHandlerContext ctx,
OUTBOUND msg, List<Object> out)
protected abstract void decode(ChannelHandlerContext ctx,
INBOUND msg, List<Object> out)

```

encode() 处理出站消息类型 OUTBOUND 到 INBOUND，而 decode() 则相反。我们在哪里可能使用这样的编解码器?

在现实中,这是一个相当常见的用例,往往涉及两个来回转换的数据消息传递API 。这是常有的事,当我们不得不与遗留或专有的消息格式进行互操作。

- 例：WebSocketConvertHandler 是一个静态嵌套类，继承了参数为 WebSocketFrame（类型为 INBOUND）和 WebSocketFrame（类型为 OUTBOUND）的 MessageToMessageCode

```java
public class WebSocketConvertHandler extends MessageToMessageCodec<WebSocketFrame, WebSocketConvertHandler.WebSocketFrame> {  //1

    public static final WebSocketConvertHandler INSTANCE = new WebSocketConvertHandler();

    @Override
    protected void encode(ChannelHandlerContext ctx, WebSocketFrame msg, List<Object> out) throws Exception {   
        ByteBuf payload = msg.getData().duplicate().retain();
        switch (msg.getType()) {   //2
            case BINARY:
                out.add(new BinaryWebSocketFrame(payload));
                break;
            case TEXT:
                out.add(new TextWebSocketFrame(payload));
                break;
            case CLOSE:
                out.add(new CloseWebSocketFrame(true, 0, payload));
                break;
            case CONTINUATION:
                out.add(new ContinuationWebSocketFrame(payload));
                break;
            case PONG:
                out.add(new PongWebSocketFrame(payload));
                break;
            case PING:
                out.add(new PingWebSocketFrame(payload));
                break;
            default:
                throw new IllegalStateException("Unsupported websocket msg " + msg);
        }
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, io.netty.handler.codec.http.websocketx.WebSocketFrame msg, List<Object> out) throws Exception {
        if (msg instanceof BinaryWebSocketFrame) {  //3
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.BINARY, msg.content().copy()));
        } else if (msg instanceof CloseWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CLOSE, msg.content().copy()));
        } else if (msg instanceof PingWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PING, msg.content().copy()));
        } else if (msg instanceof PongWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PONG, msg.content().copy()));
        } else if (msg instanceof TextWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.TEXT, msg.content().copy()));
        } else if (msg instanceof ContinuationWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CONTINUATION, msg.content().copy()));
        } else {
            throw new IllegalStateException("Unsupported websocket msg " + msg);
        }
    }

    public static final class WebSocketFrame {  //4
        public enum FrameType {        //5
            BINARY,
            CLOSE,
            PING,
            PONG,
            TEXT,
            CONTINUATION
        }

        private final FrameType type;
        private final ByteBuf data;
        public WebSocketFrame(FrameType type, ByteBuf data) {
            this.type = type;
            this.data = data;
        }

        public FrameType getType() {
            return type;
        }

        public ByteBuf getData() {
            return data;
        }
    }
}

```

1. 编码 WebSocketFrame 消息转为 WebSocketFrame 消息
2. 检测 WebSocketFrame 的 FrameType 类型，并且创建一个新的响应的 FrameType 类型的 WebSocketFrame
3. 通过 instanceof 来检测正确的 FrameType
4. 自定义消息类型 WebSocketFrame
5. 枚举类明确了 WebSocketFrame 的类型

>CombinedChannelDuplexHandler

结合解码器和编码器在一起可能会牺牲可重用性。为了避免这种方式，并且部署一个解码器和编码器到 ChannelPipeline 作为逻辑单元而不失便利性。

```java
public class CombinedChannelDuplexHandler<I extends ChannelInboundHandler,O extends ChannelOutboundHandler>
```

```java
public class CombinedByteCharCodec extends CombinedChannelDuplexHandler<ByteToCharDecoder, CharToByteEncoder> {
    public CombinedByteCharCodec() {
        super(new ByteToCharDecoder(), new CharToByteEncoder());
    }
}
```

- CombinedByteCharCodec 的参数是解码器和编码器的实现用于处理进站字节和出站消息
- 传递 ByteToCharDecoder 和 CharToByteEncoder 实例到 super 构造函数来委托调用使他们结合起来。

不同的抽象编解码器类可以支持处理一个类中同时实现解码和编码；另一方面，如果我们需要更大的灵活性,希望结合现有实现我们也可以选择结合他们无需扩展抽象编解码器的任何类。