# Netty 架构模型的组件

Netty 的架构模型，核心组件包括：

- Bootstrap 和 ServerBootstrap
- Channel
- ChannelHandler
- ChannelPipeline
- EventLoop
- ChannelFuture

## BOOTSTRAP
Netty 应用程序通过设置 bootstrap（引导）类的开始，该类提供了一个 用于应用程序网络层配置的容器。

`作用`:Bootstrapping（引导） 是出现在Netty 配置程序的过程中，Bootstrapping在给服务器绑定指定窗口或者要连接客户端的时候会使用到。

Bootstrapping 有以下两种类型：

- 一种是用于客户端的Bootstrap
- 一种是用于服务端的ServerBootstrap



## CHANNEL
底层网络传输 API 必须提供给应用 I/O操作的接口，如读，写，连接，绑定等等。对于我们来说，这是结构几乎总是会成为一个“socket”。 Netty 中的接口 Channel 定义了与 socket 丰富交互的操作集：bind, close, config, connect, isActive, isOpen, isWritable, read, write 等等。 Netty 提供大量的 Channel 实现来专门使用。这些包括 AbstractChannel，AbstractNioByteChannel，AbstractNioChannel，EmbeddedChannel， LocalServerChannel，NioSocketChannel 等等。

## CHANNELHANDLER
ChannelHandler 支持很多协议，并且提供用于数据处理的容器。我们已经知道 ChannelHandler 由特定事件触发。 ChannelHandler 可专用于几乎所有的动作，包括将一个对象转为字节（或相反），执行过程中抛出的异常处理。

![](https://atts.w3cschool.cn/attachments/image/20170808/1502159311316226.jpg)
常用的一个接口是 ChannelInboundHandler，这个类型接收到入站事件（包括接收到的数据）可以处理应用程序逻辑。当你需要提供响应时，你也可以从 ChannelInboundHandler 冲刷数据。一句话，业务逻辑经常存活于一个或者多个 ChannelInboundHandler。

每个ChannelHandler需要做什么都取决与它的超类。为了能够让开发处理逻辑变得简单，Netty提供了一些默认的处理程序来实现形式的“adapter（适配器）”类。pipeline 中每个的 ChannelHandler 主要负责转发事件到链中的下一个处理器。这些适配器类（及其子类）会自动帮你实现，所以你只需要实现该特定的方法和事件。

>为什么用适配器？

有几个适配器类，可以减少编写自定义 ChannelHandlers ，因为他们提供对应接口的所有方法的默认实现。（也有类似的适配器，用于创建编码器和解码器，这我们将在稍后讨论。）这些都是创建自定义处理器时，会经常调用的适配器：ChannelHandlerAdapter、ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter、ChannelDuplexHandlerAdapter

### 编码器、解码器

发送或接收消息时，Netty 数据转换就发生了。入站消息将从字节转为一个Java对象;也就是说，“解码”。如果该消息是出站相反会发生：“编码”，从一个Java对象转为字节。其原因是简单的：网络数据是一系列字节，因此需要从那类型进行转换。

不同类型的抽象类用于提供编码器和解码器的，这取决于手头的任务。例如，应用程序可能并不需要马上将消息转为字节。相反，该​​消息将被转换 一些其他格式。一个编码器将仍然可以使用，但它也将衍生自不同的超类，

在一般情况下，基类将有一个名字类似 ByteToMessageDecoder 或 MessageToByteEncoder。在一种特殊类型的情况下，你可能会发现类似 ProtobufEncoder 和 ProtobufDecoder，用于支持谷歌的 protocol buffer。

严格地说，其他处理器可以做编码器和解码器能做的事。但正如适配器类简化创建通道处理器，所有的编码器/解码器适配器类 都实现自 ChannelInboundHandler 或 ChannelOutboundHandler。

对于入站数据，channelRead 方法/事件被覆盖。这种方法在每个消息从入站 Channel 读入时调用。该方法将调用特定解码器的“解码”方法，并将解码后的消息转发到管道中下个的 ChannelInboundHandler。

出站消息是类似的。编码器将消息转为字节，转发到下个的 ChannelOutboundHandler。

### SimpleChannelHandler

最常见的处理器是接收到解码后的消息并应用一些业务逻辑到这些数据。要创建这样一个 ChannelHandler，你只需要扩展基类SimpleChannelInboundHandler 其中 T 是想要进行处理的类型。这样的处理器，你将覆盖基类的一个或多个方法，将获得被作为输入参数传递所有方法的 ChannelHandlerContext 的引用。

在这种类型的处理器方法中的最重要是 channelRead0(ChannelHandlerContext，T)。在这个调用中，T 是将要处理的消息。 你怎么做，完全取决于你，但无论如何你不能阻塞 I/O线程，因为这可能是不利于高性能。

- 阻塞操作

I/O 线程一定不能完全阻塞，因此禁止任何直接阻塞操作在你的 ChannelHandler， 有一种方法来实现这一要求。你可以指定一个 EventExecutorGroup。当添加 ChannelHandler 到ChannelPipeline。此 EventExecutorGroup 将用于获得EventExecutor，将执行所有的 ChannelHandler 的方法。这EventExecutor 将从 I/O 线程使用不同的线程，从而释放EventLoop。

## CHANNELPIPELINE
ChannelPipeline 提供了一个容器给 ChannelHandler 链并提供了一个API 用于管理沿着链入站和出站事件的流动。每个 Channel 都有自己的ChannelPipeline，当 Channel 创建时自动创建的。 ChannelHandler 是如何安装在 ChannelPipeline？ 主要是实现了ChannelHandler 的抽象 ChannelInitializer。ChannelInitializer子类 通过 ServerBootstrap 进行注册。当它的方法 initChannel() 被调用时，这个对象将安装自定义的 ChannelHandler 集到 pipeline。当这个操作完成时，ChannelInitializer 子类则 从 ChannelPipeline 自动删除自身。

## ChannelHandler 和 ChannelPipeline

ChannelPipeline 就是 ChannelHandler 链的容器。

如果消息或任何其他入站事件被读到，将从 pipeline 头部开始，传递到第一个 ChannelInboundHandler。该处理器可能会或可能不会实际修改数据，取决于其特定的功能，在这之后 该数据将被传递到链中的下一个 ChannelInboundHandler。最后，将数据 到达 pipeline 的尾部，此时所有处理结束。

数据的出站运动（即，数据被“写入”）在概念上是相同的。在这种情况下的数据从尾部流过 ChannelOutboundHandlers 的链，直到它到达头部。超过这点，出站数据将到达的网络传输，在这里显示为一个 socket。通常，这将触发一个写入操作

在当前的链（chain）中，事件可以通过 ChanneHandlerContext 传递给下一个 handler。Netty 为此提供了抽象基类ChannelInboundHandlerAdapter 和 hannelOutboundHandlerAdapter，用来处理你想要的事件。 这些类提供的方法的实现，可以简单地通过调用 ChannelHandlerContext 上的相应方法将事件传递给下一个 handler。在实际应用中，您可以按需覆盖相应的方法即可。

所以，如果出站和入站操作是不同的，当 ChannelPipeline 中有混合处理器时将发生什么？虽然入站和出站处理器都扩展了 ChannelHandler，Netty 的 ChannelInboundHandler 的实现 和 ChannelOutboundHandler 之间的是有区别的，从而保证数据传递只从一个处理器到下一个处理器保证正确的类型。

当 ChannelHandler 被添加到的 ChannelPipeline 它得到一个 ChannelHandlerContext，它代表一个 ChannelHandler 和 ChannelPipeline 之间的“绑定”。它通常是安全保存对此对象的引用，除了当协议中的使用的是不面向连接（例如，UDP）。而该对象可以被用来获得 底层 Channel,它主要是用来写出站数据。

>实际上，在 Netty 发送消息可以采用两种方式：直接写消息给 Channel 或者写入 ChannelHandlerContext 对象。这两者主要的区别是， 前一种方法会导致消息从 ChannelPipeline的尾部开始，而后者导致消息从 ChannelPipeline 下一个处理器开始。

## EVENTLOOP
EventLoop 用于处理 Channel 的 I/O 操作。一个单一的 EventLoop通常会处理多个 Channel 事件。一个 EventLoopGroup 可以含有多于一个的 EventLoop 和 提供了一种迭代用于检索清单中的下一个。

## CHANNELFUTURE
Netty 所有的 I/O 操作都是异步。因为一个操作可能无法立即返回，我们需要有一种方法在以后确定它的结果。出于这个目的，Netty 提供了接口 ChannelFuture,它的 addListener 方法注册了一个 ChannelFutureListener ，当操作完成时，可以被通知（不管成功与否）。
