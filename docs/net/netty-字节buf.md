# Netty中的Buffer API

Buffer API主要包括

- ByteBuf
- ByteBufHolder

Netty 根据 reference-counting(引用计数)来确定何时可以释放 ByteBuf 或 ByteBufHolder 和其他相关资源，从而可以利用池和其他技巧来提高性能和降低内存的消耗。这一点上不需要开发人员做任何事情，但是在开发 Netty 应用程序时，尤其是使用 ByteBuf 和 ByteBufHolder 时，你应该尽可能早地释放池资源。 Netty 缓冲 API 提供了几个优势：

- 可以自定义缓冲类型
- 通过一个内置的复合缓冲类型实现零拷贝
- 扩展性好，比如 StringBuilder
- 不需要调用 flip() 来切换读/写模式
- 读取和写入索引分开
- 方法链
- 引用计数
- Pooling(池)

## 字节数据的容器ByteBuf

网络通信都是要基于底层的字节流来传输，那么传输所使用的数据接口就要求是效率高得、使用方便的而且容易使用的，Netty的ByteBuf更好能够达到这些要求。

ByteBuf 是一个已经经过优化的很好使用的数据容器，字节数据可以有效的被添加到 ByteBuf 中或者也可以从 ByteBuf 中直接获取数据。ByteBuf中有两个索引：一个用来读，一个用来写。这两个索引达到了便于操作的目的。我们可以按顺序的读取数据，也可以通过调整读取数据的索引或者直接将读取位置索引作为参数传递给get方法来重复读取数据。

### ByteBuf 的工作原理

- 写入数据到 ByteBuf 后，writerIndex（写入索引）增加写入的字节数。读取字节后，readerIndex（读取索引）也增加读取出的字节数。你可以读取字节，直到写入索引和读取索引处在相同的位置。此时ByteBuf不可读，所以下一次读操作将会抛出 IndexOutOfBoundsException，就像读取数组时越位一样。

- 调用 ByteBuf 的以 "read" 或 "write" 开头的任何方法都将自动增加相应的索引。另一方面，"set" 、 "get"操作字节将不会移动索引位置，它们只会在指定的相对位置上操作字节。

- 可以给ByteBuf指定一个最大容量值，这个值限制着ByteBuf的容量。任何尝试将写入超过这个值的数据的行为都将导致抛出异常。ByteBuf 的默认最大容量限制是 Integer.MAX_VALUE。

- ByteBuf 类似于一个字节数组，最大的区别是读和写的索引可以用来控制对缓冲区数据的访问。

### ByteBuf 使用模式

>HEAP BUFFER(堆缓冲区)

最常用的模式是 ByteBuf 将数据存储在 JVM 的堆空间，这是通过将数据存储在数组的实现。堆缓冲区可以快速分配，当不使用时也可以快速释放。它还提供了直接访问数组的方法，通过 ByteBuf.array() 来获取 byte[]数据。 

```java
ByteBuf heapBuf = ...;
if (heapBuf.hasArray()) {                //1 检查 ByteBuf 是否有支持数组。
    byte[] array = heapBuf.array();        //2 如果有的话，得到引用数组。
    int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();                //3 计算第一字节的偏移量。
    int length = heapBuf.readableBytes();//4 获取可读的字节数。
    handleArray(array, offset, length); //5 使用数组，偏移量和长度作为调用方法的参数。
}

```

- `注意：`

    - 访问非堆缓冲区 ByteBuf 的数组会导致UnsupportedOperationException， 可以使用 ByteBuf.hasArray()来检查是否支持访问数组。
    - 这个用法与 JDK 的 ByteBuffer 类似

> DIRECT BUFFER(直接缓冲区)

“直接缓冲区”是另一个 ByteBuf 模式。在 JDK1.4 中被引入 NIO 的ByteBuffer 类允许 JVM 通过本地方法调用分配内存

- 其目的是

    - 通过免去中间交换的内存拷贝, 提升IO处理速度; 直接缓冲区的内容可以驻留在垃圾回收扫描的堆区以外。
    - DirectBuffer 在 -XX:MaxDirectMemorySize=xxM大小限制下, 使用 Heap 之外的内存, GC对此”无能为力”,也就意味着规避了在高负载下频繁的GC过程对应用线程的中断影响

“直接缓冲区”对于那些通过 socket 实现数据传输的应用来说，是一种非常理想的方式。如果你的数据是存放在堆中分配的缓冲区，那么实际上，在通过 socket 发送数据之前，JVM 需要将先数据复制到直接缓冲区。

但是直接缓冲区的缺点是在内存空间的分配和释放上比堆缓冲区更复杂，另外一个缺点是如果要将数据传递给遗留代码处理，因为数据不是在堆上，你可能不得不作出一个副本。

```java
ByteBuf directBuf = ...
if (!directBuf.hasArray()) {            //1 检查 ByteBuf 是不是由数组支持。如果不是，这是一个直接缓冲区。
    int length = directBuf.readableBytes();//2 获取可读的字节数
    byte[] array = new byte[length];    //3 分配一个新的数组来保存字节
    directBuf.getBytes(directBuf.readerIndex(), array);        //4  字节复制到数组   
    handleArray(array, 0, length);  //5 将数组，偏移量和长度作为参数调用某些处理方法
}
```

显然，这比使用数组要多做一些工作。因此，如果你事前就知道容器里的数据将作为一个数组被访问，你可能更愿意使用堆内存。

### COMPOSITE BUFFER(复合缓冲区)

最后一种模式是复合缓冲区，我们可以创建多个不同的 ByteBuf，然后提供一个这些 ByteBuf 组合的视图。复合缓冲区就像一个列表，我们可以动态的添加和删除其中的 ByteBuf，JDK 的 ByteBuffer 没有这样的功能。

Netty 提供了 ByteBuf 的子类 CompositeByteBuf 类来处理复合缓冲区，CompositeByteBuf 只是一个视图。

`警告`

- CompositeByteBuf.hasArray() 总是返回 false，因为它可能既包含堆缓冲区，也包含直接缓冲区

- 例如，一条消息由 header 和 body 两部分组成，将 header 和 body 组装成一条消息发送出去，可能 body 相同，只是 header 不同，使用CompositeByteBuf 就不用每次都重新分配一个新的缓冲区


JDK 的 ByteBuffer 的一个实现。两个 ByteBuffer 的数组创建保存消息的组件，第三个创建用于保存所有数据的副本。

```java
// 使用数组保存消息的各个部分
ByteBuffer[] message = { header, body };

// 使用副本来合并这两个部分
ByteBuffer message2 = ByteBuffer.allocate(
        header.remaining() + body.remaining());
message2.put(header);
message2.put(body);
message2.flip();
```

这种做法显然是低效的;分配和复制操作不是最优的方法，操纵数组使代码显得很笨拙。CompositeByteBuf 的改进版本:

```java
CompositeByteBuf messageBuf = ...;
ByteBuf headerBuf = ...; // 可以支持或直接
ByteBuf bodyBuf = ...; // 可以支持或直接
//1 追加 ByteBuf 实例的 CompositeByteBuf
messageBuf.addComponents(headerBuf, bodyBuf);
// ....
messageBuf.removeComponent(0); // 移除头    //2 删除 索引1的 ByteBuf

for (int i = 0; i < messageBuf.numComponents(); i++) {                        //3 遍历所有 ByteBuf 实例。
    System.out.println(messageBuf.component(i).toString());
}
```

可以简单地把 CompositeByteBuf 当作一个可迭代遍历的容器。 CompositeByteBuf 不允许访问其内部可能存在的支持数组，也不允许直接访问数据，这一点类似于直接缓冲区模式:

```java
CompositeByteBuf compBuf = ...;
int length = compBuf.readableBytes();    //1 得到的可读的字节数。
byte[] array = new byte[length];        //2 分配一个新的数组,数组长度为可读字节长度。
compBuf.getBytes(compBuf.readerIndex(), array);    //3 读取字节到数组
handleArray(array, 0, length);    //4 使用数组，把偏移量和长度作为参数
```

Netty 尝试使用 CompositeByteBuf 优化 socket I/O 操作，消除 原生 JDK 中可能存在的的性能低和内存消耗问题。虽然这是在Netty 的核心代码中进行的优化，并且是不对外暴露的，但是作为开发者还是应该意识到其影响。

## 字节级别的操作

除了基本的读写操作， ByteBuf 还提供了它所包含的数据的修改方法。

>随机访问索引

ByteBuf 使用zero-based 的 indexing(从0开始的索引)，第一个字节的索引是 0，最后一个字节的索引是 ByteBuf 的 capacity - 1，下面代码是遍历 ByteBuf 的所有字节：

```java
ByteBuf buffer = ...;
for (int i = 0; i < buffer.capacity(); i++) {
    byte b = buffer.getByte(i);
    System.out.println((char) b);
}
```

注意通过索引访问时不会推进 readerIndex （读索引）和 writerIndex（写索引），我们可以通过 ByteBuf 的 readerIndex(index) 或 writerIndex(index) 来分别推进读索引或写索引

>顺序访问索引

ByteBuf 提供两个指针变量支付读和写操作，读操作是使用 readerIndex()，写操作时使用 writerIndex()。这和JDK的ByteBuffer不同，ByteBuffer只有一个方法来设置索引，所以需要使用 flip() 方法来切换读和写模式。

ByteBuf 一定符合：0 <= readerIndex <= writerIndex <= capacity。

![p](https://atts.w3cschool.cn/attachments/image/20170808/1502159492537989.jpg)


1. 字节，可以被丢弃，因为它们已经被读

2. 还没有被读的字节是：“readable bytes（可读字节）”

3. 空间可加入多个字节的是：“writeable bytes（写字节）”

>可丢弃字节的字节

标有“可丢弃字节”的段包含已经被读取的字节。他们可以被丢弃，通过调用discardReadBytes() 来回收空间。这个段的初始大小存储在readerIndex，为 0，当“read”操作被执行时递增（“get”操作不会移动 readerIndex）。

ByteBuf.discardReadBytes() 可以用来清空 ByteBuf 中已读取的数据，从而使 ByteBuf 有多余的空间容纳新的数据，但是discardReadBytes() 可能会涉及内存复制，因为它需要移动 ByteBuf 中可读的字节到开始位置，这样的操作会影响性能，一般在需要马上释放内存的时候使用收益会比较大。

>可读字节

ByteBuf 的“可读字节”分段存储的是实际数据。新分配，包装，或复制的缓冲区的 readerIndex 的默认值为 0 。任何操作，其名称以 "read" 或 "skip" 开头的都将检索或跳过该数据在当前 readerIndex ，并且通过读取的字节数来递增。

如果所谓的读操作是一个指定 ByteBuf 参数作为写入的对象，并且没有一个目标索引参数，目标缓冲区的 writerIndex 也会增加了。

>索引管理

在 JDK 的 InputStream 定义了 mark(int readlimit) 和 reset()方法。这些是分别用来标记流中的当前位置和复位流到该位置。

同样，您可以设置和重新定位ByteBuf readerIndex 和 writerIndex 通过调用 markReaderIndex(), markWriterIndex(), resetReaderIndex() 和 resetWriterIndex()。这些类似于InputStream 的调用，所不同的是，没有 readlimit 参数来指定当标志变为无效。

您也可以通过调用 readerIndex(int) 或 writerIndex(int) 将指标移动到指定的位置。在尝试任何无效位置上设置一个索引将导致 IndexOutOfBoundsException 异常。

调用 clear() 可以同时设置 readerIndex 和 writerIndex 为 0。注意，这不会清除内存中的内容。

>查询操作

有几种方法，以确定在所述缓冲器中的指定值的索引。最简单的是使用 indexOf() 方法。更复杂的搜索执行以 ByteBufProcessor 为参数的方法。这个接口定义了一个方法，boolean process(byte value)，它用来报告输入值是否是一个正在寻求的值。

ByteBufProcessor 定义了很多方便实现共同目标值。

>衍生的缓冲区

“衍生的缓冲区”是代表一个专门的展示 ByteBuf 内容的“视图”。这种视图是由 duplicate(), slice(), slice(int, int),readOnly(), 和 order(ByteOrder) 方法创建的。所有这些都返回一个新的 ByteBuf 实例包括它自己的 reader, writer 和标记索引。然而，内部数据存储共享就像在一个 NIO 的 ByteBuffer。这使得衍生的缓冲区创建、修改其 内容，以及修改其“源”实例更廉价。

>ByteBuf 拷贝

如果需要已有的缓冲区的全新副本，使用 copy() 或者 copy(int, int)。不同于派生缓冲区，这个调用返回的 ByteBuf 有数据的独立副本。

若需要操作某段数据，使用 slice(int, int)

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8); //1 创建一个 ByteBuf 保存特定字节串。

ByteBuf sliced = buf.slice(0, 14);          //2 创建从索引 0 开始，并在 14 结束的 ByteBuf 的新 slice。
System.out.println(sliced.toString(utf8));  //3 打印 Netty in Action

buf.setByte(0, (byte) 'J');                 //4 更新索引 0 的字节。
// 5 断言成功，因为数据是共享的，并以一个地方所做的修改将在其他地方可见。
assert buf.getByte(0) == sliced.getByte(0);
```

下面看下如何将一个 ByteBuf 段的副本不同于 slice。

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);     //1 创建一个 ByteBuf 保存特定字节串。

ByteBuf copy = buf.copy(0, 14);               //2 创建从索引0开始和 14 结束 的 ByteBuf 的段的拷贝。
System.out.println(copy.toString(utf8));      //3 打印 Netty in Action

buf.setByte(0, (byte) 'J');                   //4 更新索引 0 的字节。
// 5 因为数据不是共享的，并以一个地方所做的修改将不影响其他
assert buf.getByte(0) != copy.getByte(0);
```

代码几乎是相同的，但所 衍生的 ByteBuf 效果是不同的。因此，使用一个 slice 可以尽可能避免复制内存。

### 读/写操作

读/写操作主要由2类：

- gget()/set() 操作从给定的索引开始，保持不变
- read()/write() 操作从给定的索引开始，与字节访问的数量来适用，递增当前的写索引或读索引

>下面是常见的 get() 操作

方法名称|描述
-|-
getBoolean(int)|返回当前索引的 Boolean 值
getByte(int) getUnsignedByte(int)|返回当前索引的(无符号)字节
getMedium(int) getUnsignedMedium(int)|返回当前索引的 (无符号) 24-bit 中间值
getInt(int) getUnsignedInt(int)|返回当前索引的(无符号) 整型
getLong(int) getUnsignedLong(int)|返回当前索引的 (无符号) Long 型
getShort(int) getUnsignedShort(int)|返回当前索引的 (无符号) Short 型
getBytes(int, ...)|字节

>et() operations

方法名称|描述
-|-
setBoolean(int, boolean)|在指定的索引位置设置 Boolean 值
setByte(int, int)|在指定的索引位置设置 byte 值
setMedium(int, int)|在指定的索引位置设置 24-bit 中间 值
setInt(int, int)|在指定的索引位置设置 int 值
setLong(int, long)|在指定的索引位置设置 long 值
setShort(int, int)|在指定的索引位置设置 short 值

>get() and set()用法

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);    //1 创建一个新的 ByteBuf 给指定 String 保存字节
System.out.println((char)buf.getByte(0));                    //2 打印的第一个字符，N

int readerIndex = buf.readerIndex();                        //3 存储当前 readerIndex 和 writerIndex
int writerIndex = buf.writerIndex();

buf.setByte(0, (byte)'B');                            //4 更新索引 0 的字符B

System.out.println((char)buf.getByte(0));                    //5 打印出的第一个字符，现在B
assert readerIndex == buf.readerIndex();                    //6 断言成功，因为这些操作永远不会改变索引
assert writerIndex ==  buf.writerIndex();

```

> read() 方法

方法名称|描述
-|-
readBoolean()　|　Reads the Boolean value at the current readerIndex and increases the readerIndex by 1.
readByte()　readUnsignedByte()　|Reads the (unsigned) byte value at the current readerIndex and increases　the readerIndex by 1.
readMedium()　readUnsignedMedium()　|Reads the (unsigned) 24-bit medium value at the current readerIndex and　increases the readerIndex by 3.
readInt()　readUnsignedInt()|　Reads the (unsigned) int value at the current readerIndex and increases　the readerIndex by 4.
readLong()　readUnsignedLong()　|　Reads the (unsigned) int value at the current readerIndex and increases　the readerIndex by 8.
readShort()　readUnsignedShort()|	Reads the (unsigned) int value at the current readerIndex and increases　the readerIndex by 2.
readBytes(int,int, ...)|Reads the value on the current readerIndex for the given length into the　given object. Also increases the readerIndex by the length.

>read() 方法都对应一个　write()

方法名称|描述
-|-
writeBoolean(boolean)|Writes the Boolean value on the current writerIndex and increases the　writerIndex by 1.
writeByte(int)	|　Writes the byte value on the current writerIndex and increases the　writerIndex by 1.
writeMedium(int)	|　Writes the medium value on the current writerIndex and increases the　writerIndex by 3.
writeInt(int)	|　Writes the int value on the current writerIndex and increases the　writerIndex by 4.
writeLong(long)	|　Writes the long value on the current writerIndex and increases the　writerIndex by 8.
writeShort(int)	|　Writes the short value on the current writerIndex and increases thewriterIndex by 2.
writeBytes(int，...）	|　Transfers the bytes on the current writerIndex from given resources.

>更多操作

方法名称|	描述
-|-
isReadable()|	Returns true if at least one byte can be read.
isWritable()|	Returns true if at least one byte can be written.
readableBytes()|	Returns the number of bytes that can be read.
writablesBytes()|	Returns the number of bytes that can be written.
capacity()|	Returns the number of bytes that the ByteBuf can hold. After this it will try to expand again until maxCapacity() is reached.
maxCapacity()|	Returns the maximum number of bytes the ByteBuf can hold.
hasArray()|	Returns true if the ByteBuf is backed by a byte array.
array()|	Returns the byte array if the ByteBuf is backed by a byte array

