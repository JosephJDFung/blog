# Netty ByteBufHolder的使用

我们时不时的会遇到这样的情况：即需要另外存储除有效的实际数据各种属性值。HTTP响应就是一个很好的例子；与内容一起的字节的还有状态码，cookies等

Netty 提供的 ByteBufHolder 可以对这种常见情况进行处理。 ByteBufHolder 还提供了对于 Netty 的高级功能，如缓冲池，其中保存实际数据的 ByteBuf 可以从池中借用，如果需要还可以自动释放。

ByteBufHolder 有那么几个方法。到底层的这些支持接入数据和引用计数。

>ByteBufHolder

名称|描述
-|-
data()|	返回 ByteBuf 保存的数据
copy()|	制作一个 ByteBufHolder 的拷贝，但不共享其数据(所以数据也是拷贝).

如果你想实现一个“消息对象”有效负载存储在 ByteBuf，使用ByteBufHolder 是一个好主意。

## Netty之ByteBuf 分配

### ByteBufAllocator

为了减少分配和释放内存的开销，Netty 通过支持池类 ByteBufAllocator，可用于分配的任何 ByteBuf 我们已经描述过的类型的实例。是否使用池是由应用程序决定的

>ByteBufAllocator methods

名称|描述
-|-
buffer() buffer(int) buffer(int, int)	|Return a ByteBuf with heap-based or direct data storage.
heapBuffer() heapBuffer(int) heapBuffer(int, int)|	Return a ByteBuf with heap-based storage.
directBuffer() directBuffer(int) directBuffer(int, int)	|Return a ByteBuf with direct storage.
compositeBuffer() compositeBuffer(int) heapCompositeBuffer() heapCompositeBuffer(int) directCompositeBuffer()directCompositeBuffer(int)	|Return a CompositeByteBuf that can be expanded by adding heapbased or direct buffers.
ioBuffer()	|Return a ByteBuf that will be used for I/O operations on a socket.


通过一些方法接受整型参数允许用户指定 ByteBuf 的初始和最大容量值。你可能还记得，ByteBuf 存储可以扩大到其最大容量。

得到一个 ByteBufAllocator 的引用很简单。你可以得到从 Channel （在理论上，每 Channel 可具有不同的 ByteBufAllocator ），或通过绑定到的 ChannelHandler 的 ChannelHandlerContext 得到它，用它实现了你数据处理逻辑。

>ByteBufAllocator 的两种方式。

```java
Channel channel = ...;
ByteBufAllocator allocator = channel.alloc(); //1 从 channel 获得 ByteBufAllocator
....
ChannelHandlerContext ctx = ...;
ByteBufAllocator allocator2 = ctx.alloc(); //2 从 ChannelHandlerContext 获得 ByteBufAllocator
...
```

Netty 提供了两种 ByteBufAllocator 的实现，一种是 PooledByteBufAllocator,用ByteBuf 实例池改进性能以及内存使用降到最低，此实现使用一个“jemalloc”内存分配。其他的实现不池化 ByteBuf 情况下，每次返回一个新的实例。

Netty 默认使用 PooledByteBufAllocator，我们可以通过 ChannelConfig 或通过引导设置一个不同的实现来改变。

### Unpooled （非池化）缓存

当未引用 ByteBufAllocator 时，上面的方法无法访问到 ByteBuf。对于这个用例 Netty 提供一个实用工具类称为 Unpooled,，它提供了静态辅助方法来创建非池化的 ByteBuf 实例。

名称|描述
-|-
buffer() buffer(int) buffer(int, int)|	Returns an unpooled ByteBuf with heap-based storage
directBuffer() directBuffer(int) directBuffer(int, int)	|Returns an unpooled ByteBuf with direct storage
wrappedBuffer()	|Returns a ByteBuf, which wraps the given data.
copiedBuffer()	|Returns a ByteBuf, which copies the given data

在 非联网项目，该 Unpooled 类也使得它更容易使用的 ByteBuf API，获得一个高性能的可扩展缓冲 API，而不需要 Netty 的其他部分的。

>ByteBufUtil

ByteBufUtil 静态辅助方法来操作 ByteBuf，因为这个 API 是通用的，与使用池无关，这些方法已经在外面的分配类实现。

也许最有价值的是 hexDump() 方法，这个方法返回指定 ByteBuf 中可读字节的十六进制字符串，可以用于调试程序时打印 ByteBuf 的内容。一个典型的用途是记录一个 ByteBuf 的内容进行调试。十六进制字符串相比字节而言对用户更友好。 而且十六进制版本可以很容易地转换回实际字节表示。

另一个有用方法是 使用 boolean equals(ByteBuf, ByteBuf),用来比较 ByteBuf 实例是否相等。在 实现自己 ByteBuf 的子类时经常用到。

## Netty引用计数器

>在Netty 4中为 ByteBuf 和 ByteBufHolder（两者都实现了 ReferenceCounted 接口）引入了引用计数器。

引用计数器本身并不复杂；它能够在特定的对象上跟踪引用的数目，实现了ReferenceCounted 的类的实例会通常开始于一个活动的引用计数器为 1。而如果对象活动的引用计数器大于0，就会被保证不被释放。当数量引用减少到0，将释放该实例。需要注意的是“释放”的语义是特定于具体的实现。最起码，一个对象，它已被释放应不再可用。

>这种技术就是诸如 PooledByteBufAllocator 这种减少内存分配开销的池化的精髓部分。

```java
Channel channel = ...;
ByteBufAllocator allocator = channel.alloc(); //1 从 channel 获取 ByteBufAllocator
....
ByteBuf buffer = allocator.directBuffer(); //2 从 ByteBufAllocator 分配一个 ByteBuf
assert buffer.refCnt() == 1; //3 检查引用计数器是否是 1
...
```

```java
ByteBuf buffer = ...;
boolean released = buffer.release(); 
...
```

release（）将会递减对象引用的数目。当这个引用计数达到0时，对象已被释放，并且该方法返回 true。

如果尝试访问已经释放的对象，将会抛出 IllegalReferenceCountException 异常。

需要注意的是一个特定的类可以定义自己独特的方式其释放计数的“规则”。 例如，release() 可以将引用计数器直接计为 0 而不管当前引用的对象数目。

>谁负责 release？

在一般情况下，最后访问的对象负责释放它。

