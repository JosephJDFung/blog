# Netty核心之Transport（传输）

网络应用程序让人与系统之间可以进行通信，当然网络应用程序也可以将大量的数据从一个地方转移到另一个地方。如何做到这一点取决于具体的网络传输，但转移始终是相同的：字节通过线路。传输的概念帮助我们抽象掉的底层数据转移的机制。

当你做过Java中的网络编程的时候，你应该会发现要支持的并发连接会比预期中要多得多，当然这只是在某些时候会出现的情况。如果你再尝试从阻塞切换到非阻塞传输，则可能遇会到的问题，因为 Java 的公开的网络 API 来处理这两种情况有很大的不同。

Netty 在传输层的API是统一的，这使得比你用 JDK 实现更简单。你无需重构整个代码库，然后将时间花到其他更值得去做的事情上。


transport
- `NIO` NIO-在高连接数时使用
- `OIO` OIO-在低连接数、需要低延迟时、阻塞时使用
- `Local(本地)` Local-在同一个JVM内通信时使用
- `Embedded(内嵌)` Embedded-测试ChannelHandler时使用






## 基于Netty传输的API

![](https://atts.w3cschool.cn/attachments/image/20170808/1502159388173816.jpg)

每个 Channel 都会分配一个 ChannelPipeline 和ChannelConfig。ChannelConfig 负责设置并存储 Channel 的配置，并允许在运行期间更新它们。传输一般有特定的配置设置，可能实现了 ChannelConfig. 的子类型。

ChannelPipeline 容纳了使用的 ChannelHandler 实例，这些ChannelHandler 将处理通道传递的“入站”和“出站”数据以及事件。ChannelHandler 的实现允许你改变数据状态和传输数据。

我们可以使用 ChannelHandler 做下面一些事情：

- 传输数据时，将数据从一种格式转换到另一种格式
- 异常通知
- Channel 变为 active（活动） 或 inactive（非活动） 时获得通知
- Channel 被注册或注销时从 EventLoop 中获得通知
- 通知用户特定事件

### Intercepting Filter（拦截过滤器）

ChannelPipeline 实现了常用的 Intercepting Filter（拦截过滤器）设计模式。UNIX管道是另一例子：命令链接在一起，一个命令的输出连接到 的下一行中的输入。

你还可以在运行时根据需要添加 ChannelHandler 实例到ChannelPipeline 或从 ChannelPipeline 中删除，这能帮助我们构建高度灵活的 Netty 程序。例如，你可以支持 [STARTTLS](http://en.wikipedia.org/wiki/STARTTLS) 协议，只需通过加入适当的 ChannelHandler（这里是 SslHandler）到的ChannelPipeline 中，当被请求这个协议时。

### Channel 提供方法

方法名称|描述
-|-
eventLoop()|返回分配给Channel的EventLoop
pipeline()|返回分配给Channel的ChannelPipeline
isActive()|返回Channel是否激活，已激活说明与远程连接对等
localAddress()|返回已绑定的本地SocketAddress
remoteAddress()|返回已绑定的远程SocketAddress
write()|写数据到远程客户端，数据通过ChannelPipeline传输过去
flush()|刷新先前的数据
writeAndFlush(...)|一个方便的方法用户调用write(...)而后调用flush()

#### 写数据到远程已连接客户端可以调用Channel.write()方法

```java
Channel channel = ...; // 获取channel的引用
ByteBuf buf = Unpooled.copiedBuffer("your data", CharsetUtil.UTF_8);            //1 创建 ByteBuf 保存写的数据
ChannelFuture cf = channel.writeAndFlush(buf); //2 写数据，并刷新

cf.addListener(new ChannelFutureListener() {    //3 添加 ChannelFutureListener 即可写操作完成后收到通知
    @Override
    public void operationComplete(ChannelFuture future) {
        if (future.isSuccess()) {                //4 写操作没有错误完成
            System.out.println("Write successful");
        } else {
            System.err.println("Write error");    //5 写操作完成时出现错误
            future.cause().printStackTrace();
        }
    }
});
```

Channel 是线程安全(thread-safe)的，它可以被多个不同的线程安全的操作，在多线程环境下，所有的方法都是安全的。正因为 Channel 是安全的，我们存储对Channel的引用，并在学习的时候使用它写入数据到远程已连接的客户端，使用多线程也是如此。

多线程例子:

```java
final Channel channel = ...; // 获取channel的引用
final ByteBuf buf = Unpooled.copiedBuffer("your data",
        CharsetUtil.UTF_8).retain();    //1 创建一个 ByteBuf 保存写的数据
Runnable writer = new Runnable() {        //2 创建 Runnable 用于写数据到 channel
    @Override
    public void run() {
        channel.writeAndFlush(buf.duplicate());
    }
};
Executor executor = Executors.newCachedThreadPool();//3 获取 Executor 的引用使用线程来执行任务

//写进一个线程
executor.execute(writer);        //4 手写一个任务，在一个线程中执行

//写进另外一个线程
executor.execute(writer);        //5 在另一个线程中执行


```

## Netty中包含的 Transport

虽然Netty不能支持所有的传输协议，但是Netty自身是携带了一些传输协议的，这些Netty自带的传输协议已经能够满足我们的使用。Netty应用程序的传输协议依赖的是底层协议，接下来我们学习的内容就是Netty中包含的传输协议。

Netty中的传输方式有如下几种：

方法名称|包|描述
-|-|-
NIO|io.netty.channel.socket.nio|基于java.nio.channels的工具包，使用选择器作为基础的方法。
OIO|io.netty.channel.socket.oio|基于java.net的工具包，使用阻塞流。
Local|io.netty.channel.local|用来在虚拟机之间本地通信。
Embedded|io.netty.channel.embedded|嵌入传输，它允许在没有真正网络的传输中使用 ChannelHandler，可以非常有用的来测试ChannelHandler的实现。

### NIO-Nonblocking I/O
NIO传输是目前最常用的方式，它通过使用选择器提供了完全异步的方式操作所有的 I/O，NIO 从Java 1.4才被提供。

NIO 中，我们可以注册一个通道或获得某个通道的改变的状态，通道状态有下面几种改变：

- 一个新的 Channel 被接受并已准备好
- Channel 连接完成
- Channel 中有数据并已准备好读取
- Channel 发送数据出去

处理完改变的状态后需重新设置他们的状态，用一个线程来检查是否有已准备好的 Channel，如果有则执行相关事件。

在这里可能只同时一个注册的事件而忽略其他的。选择器所支持的操作在 SelectionKey 中定义，具体如下：

方法名称|描述
-|-
OP_ACCEPT|有新连接时得到通知
OP_CONNECT|连接完成后得到通知
OP_REA|准备好读取数据时得到通知
OP_WRITE|写入更多数据到通道时得到通知，大部分时间

## 同个 JVM 内的本地 Transport 通信
Netty 提供了“本地”传输，为运行在同一个 Java 虚拟机上的服务器和客户之间提供异步通信。此传输支持所有的 Netty 常见的传输实现的 API。

在此传输中，与服务器 Channel 关联的 SocketAddress 不是“绑定”到一个物理网络地址中，而是在服务器是运行时它被存储在注册表中，当 Channel 关闭时它会注销。由于该传输不是“真正的”网络通信，它不能与其他传输实现互操作。因此，客户端是希望连接到使用本地传输的的服务器时，要注意正确的用法。除此限制之外，它的使用是与其他的传输是相同的。

## 内嵌 Transport
Netty中 还提供了可以嵌入 ChannelHandler 实例到其他的 ChannelHandler 的传输，使用它们就像辅助类，增加了灵活性的方法，使您可以与你的 ChannelHandler 互动。

该嵌入技术通常用于测试 ChannelHandler 的实现，但它也可用于将功能添加到现有的 ChannelHandler 而无需更改代码。嵌入传输的关键是Channel 的实现，称为“EmbeddedChannel”。
