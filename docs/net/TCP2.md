# TCP的流迭、拥塞处理

## TCP的RTT算法
TCP重传机制我们知道Timeout的设置对于重传非常重要。

- 设长了，重发就慢，丢了老半天才重发，没有效率，性能差；
- 设短了，会导致可能并没有丢就重发。于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

而且，这个超时时间在不同的网络的情况下，根本没有办法设置一个死的值。只能动态地设置。 为了动态地设置，TCP引入了RTT——Round Trip Time，也就是一个数据包从发出去到回来的时间。这样发送端就大约知道需要多少的时间，从而可以方便地设置Timeout——RTO（Retransmission TimeOut），以让我们的重传机制更高效。

RFC793经典算法

Karn / Partridge 算法

Jacobson / Karels 算法

## TCP滑动窗口
TCP必需要解决的可靠传输以及包乱序（reordering）的问题，所以，TCP必需要知道网络实际的数据处理带宽或是数据处理速度，这样才不会引起网络拥塞，导致丢包。

TCP引入了一些技术和设计来做网络流控，Sliding Window是其中一个技术。 TCP头里有一个字段叫Window，又叫Advertised-Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。

过程：
- 接收端LastByteRead指向了TCP缓冲区中读到的位置，NextByteExpected指向的地方是收到的连续包的最后一个位置，LastByteRcved指向的是收到的包的最后一个位置，我们可以看到中间有些数据还没有到达，所以有数据空白区。
- 发送端的LastByteAcked指向了被接收端Ack过的位置（表示成功发送确认），LastByteSent表示发出去了，但还没有收到成功确认的Ack，LastByteWritten指向的是上层应用正在写的地方。

于是：
- 接收端在给发送端回ACK中会汇报自己的AdvertisedWindow = MaxRcvBuffer – LastByteRcvd – 1;
- 而发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。

![滑动窗口](https://coolshell.cn/wp-content/uploads/2014/05/tcpswwindows.png)

黑模型就是滑动窗口

1. 已收到ack确认的数据。
2. 发还没收到ack的。
3. 在窗口中还没有发出的（接收方还有空间）。
4. 窗口以外的数据（接收方没空间）

### Silly Window Syndrome

Silly Window Syndrome翻译成中文就是“糊涂窗口综合症”。正如你上面看到的一样，如果我们的接收方太忙了，来不及取走Receive Windows里的数据，那么，就会导致发送方越来越小。到最后，如果接收方腾出几个字节并告诉发送方现在有几个字节的window，而我们的发送方会义无反顾地发送这几个字节。

>TCP+IP头有40个字节，为了几个字节，要达上这么大的开销，这太不经济了。

网络上有个MTU，对于以太网来说，MTU是1500字节，除去TCP+IP头的40个字节，真正的数据传输可以有1460，这就是所谓的MSS（Max Segment Size）注意，TCP的RFC定义这个MSS的默认值是536，这是因为 RFC 791里说了任何一个IP设备都得最少接收576尺寸的大小（实际上来说576是拨号的网络的MTU，而576减去IP头的20个字节就是536）。

Silly Windows Syndrome这个现像就像是你本来可以坐200人的飞机里只做了一两个人。 要解决这个问题也不难，就是避免对小的window size做出响应，直到有足够大的window size再响应，这个思路可以同时实现在sender和receiver两端。

- 如果这个问题是由Receiver端引起的，那么就会使用 David D Clark’s 方案。在receiver端，如果收到的数据导致window size小于某个值，可以直接ack(0)回sender，这样就把window给关闭了，也阻止了sender再发数据过来，等到receiver端处理了一些数据后windows size 大于等于了MSS，或者，receiver buffer有一半为空，就可以把window打开让send 发送数据过来。
- 如果这个问题是由Sender端引起的，那么就会使用著名的 Nagle’s algorithm。这个算法的思路也是延时处理，他有两个主要的条件：1）要等到 Window Size>=MSS 或是 Data Size >=MSS，2）收到之前发送数据的ack回包，他才会发数据，否则就是在攒数据。

另外，Nagle算法默认是打开的，所以，对于一些需要小包场景的程序——比如像telnet或ssh这样的交互性比较强的程序，你需要关闭这个算法。你可以在Socket设置TCP_NODELAY选项来关闭这个算法（关闭Nagle算法没有全局参数，需要根据每个应用自己的特点来关闭）

`TCP_CORK其实是更新激进的Nagle算汉，完全禁止小包发送，而Nagle算法没有禁止小包发送`，只是禁止了大量的小包发送。最好不要两个选项都设置。

## TCP的拥塞处理

TCP通过一个timer采样了RTT并计算RTO，但是，如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络。这是一个灾难。

>TCP不是一个自私的协议，当拥塞发生的时候，要做自我牺牲。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了。

- 拥塞控制主要是四个算法：
    - 1）慢启动
    - 2）拥塞避免
    - 3）拥塞发生
    - 4）快速恢复。

### 慢热启动算法 – Slow Start

慢启动的算法如下(cwnd全称Congestion Window)：

- 1）连接建好的开始先初始化cwnd = 1，表明可以传一个MSS大小的数据。

- 2）每当收到一个ACK，cwnd++; 呈线性上升

- 3）每当过了一个RTT，cwnd = cwnd*2; 呈指数让升

- 4）还有一个ssthresh（slow start threshold），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”（后面会说这个算法）

### 拥塞避免算法 – Congestion Avoidance

算法如下：

- 1）收到一个ACK时，cwnd = cwnd + 1/cwnd

- 2）当每过一个RTT时，cwnd = cwnd + 1

这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

### 拥塞状态时的算法

当丢包的时候，会有两种情况：

- 1）等到RTO超时，重传数据包。TCP认为这种情况太糟糕，反应也很强烈。

    - sshthresh =  cwnd /2
    - cwnd 重置为 1
    - 进入慢启动过程
- 2）Fast Retransmit算法，也就是在收到3个duplicate ACK时就开启重传，而不用等到RTO超时。

    - TCP Tahoe的实现和RTO超时一样。
    - TCP Reno的实现是：
        - cwnd = cwnd /2
        - sshthresh = cwnd
        - 进入快速恢复算法——Fast Recovery

RTO超时后，sshthresh会变成cwnd的一半，这意味着，如果cwnd<=sshthresh时出现的丢包，那么TCP的sshthresh就会减了一半，然后等cwnd又很快地以指数级增涨爬到这个地方时，就会成慢慢的线性增涨。我们可以看到，TCP是怎么通过这种强烈地震荡快速而小心得找到网站流量的平衡点的。

### 快速恢复算法 – Fast Recovery

TCP New Reno

在没有SACK的支持下改进Fast Recovery算法

- 当sender这边收到了3个Duplicated Acks，进入Fast Retransimit模式，开发重传重复Acks指示的那个包。如果只有这一个包丢了，那么，重传这个包后回来的Ack会把整个已经被sender传输出去的数据ack回来。如果没有的话，说明有多个包丢了。我们叫这个ACK为Partial ACK。
- 一旦Sender这边发现了Partial ACK出现，那么，sender就可以推理出来有多个包被丢了，于是乎继续重传sliding window里未被ack的第一个包。直到再也收不到了Partial Ack，才真正结束Fast Recovery这个过程

>算法示意图

![算法示意图](https://coolshell.cn/wp-content/uploads/2014/05/tcp.fr_-900x315.jpg)


## TCP常见面试题

>为啥三次握手

三次握手能保证数据可靠传输又能提高传输效率。

若握手是两次：如果只是两次握手， 至多只有连接发起方的起始序列号能被确认， 另一方选择的序列号则得不到确认。

若客户端没有收到server的对SYN的ACK确认报文，会重发SYN报文，服务器和回复ACK，连接建立。数据发送完毕，这条连接被正常关闭。这时，延迟的SYN报文发到了server，server误以为这是client重新发送的同步报文，又回复了一个ACK，和client建立了连接，但是其实此时client已经没有数据要发送了，所以server的时间就会被浪费。

>为啥四次挥手

要保证双方都关闭了连接。因为TCP是全双工的。

在建立连接时，服务器可以把SYN和ACK放在一个包中发送。

- 但是在断开连接时，如果一端收到FIN包，但此时仍有数据未发送完，此时就需要先向对端回复FIN包的ACK。等到将剩下的数据都发送完之后，再向对端发送FIN，断开这个方向的连接。
- 因此很多时候断开连接时FIN和ACK需要在两个数据包中发送（当server也没有数据要发送了，FIN和ACK是可以放在一起发送的），因此需要四次握手

>为啥最后挥手要等2MSL
- 为了保证可靠的断开TCP的双向连接，确保足够的时间让对方收到ACK包。若客户端回复的ACK丢失，server会在超时时间到来时，重传最后一个fin包，处于TIME_WAIT状态的client可以继续回复Fin包，发送ACK。
- 保证让迟来的TCP报文段有足够的时间被识别和丢弃，避免新旧连接混淆。有些路由器会缓存没有收到的数据包，如果新的连接开启，这些数据包可能就会和新的连接中的数据包混在一起。连接结束了，网络中的延迟报文也应该被丢弃掉，以免影响立刻建立的新连接。

在Client发送出最后的ACK回复，但该ACK可能丢失。Server如果没有收到ACK，将不断重复发送FIN片段。所以Client不能立即关闭，它必须确认Server接收到了该ACK。Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。

所谓的2MSL是两倍的MSL(Maximum Segment Lifetime最大 报文段存活时间（往返时间）)。MSL指一个片段在网络中最大的存活时间，2MSL就是一个发送和一个回复所需的最大时间。如果直到2MSL，Client都没有再次收到FIN，那么Client推断ACK已经被成功接收，则结束TCP连接。

>accept发生在三次握手的哪一步

accept会监听已完成队列是否非空，当队列为空时，accept就会阻塞。当队列非空时，就从已完成队列中取出一项并返回。

客户端 connect 成功返回是在第二次握手，服务端 accept 成功返回是在三次握手成功之后。

>三次握手过程中有哪些不安全性

SYN洪泛攻击

伪装的IP向服务器发送一个SYN请求建立连接，然后服务器向该IP回复SYN和ACK，但是找不到该IP对应的主机，当超时时服务器收不到ACK会重复发送。当大量的攻击者请求建立连接时，服务器就会存在大量未完成三次握手的连接，服务器主机backlog被耗尽而不能响应其它连接。

- 当你在服务器上看到大量的半连接状态时，特别是源IP地址是随机的，基本上可以断定这是一次SYN攻击.在Linux下可以如下命令检测是否被Syn攻击
```
netstat -n -p TCP | grep SYN_RECV
```

- 防范措施：
    - 1、降低SYN timeout时间，使得主机尽快释放半连接的占用
    - 2、采用SYN cookie设置，如果短时间内连续收到某个IP的重复SYN请求，则认为受到了该IP的攻击，丢弃来自该IP的后续请求报文
    - 3、在网关处设置过滤，拒绝将一个源IP地址不属于其来源子网的包进行更远的路由et/mulinsen77/java/article/details/88925672

>超时重传和快速重传

- 超时重传：当超时时间到达时，发送方还未收到对端的ACK确认，就重传该数据包
- 快速重传：当后面的序号先到达，如接收方接收到了1、 3、 4，而2没有收到，就会立即向发送方重复发送三次ACK=2的确认请求重传。如果发送方连续收到3个相同序号的ACK，就重传该数据包。而不用等待超时

>TCP首部长度有哪些字段

TCP首部有20个字节的固定数据，用来存放报文传输过程所需的信息。有源端口，目的端口，序号，确认号，数据偏移，保留，控制位，紧急URG，控制ACK，推送PSH，复位RST，同步SYN,终止FIN，窗口，检验和，紧急指针，选项，MSS，窗口扩大，时间戳，SACK。

>TCP和UDP有什么区别

- TCP是传输控制协议，提供的是面向连接的，可靠地字节流服务。实际数据传输之前服务器和客户端要进行三次握手，会话结束后结束连接。UDP是用户数据报协议，是无连接的。因为无连接，而且没有超时重发机制，所以UDP传输速度很快。主要用于视频传输（但其实现在各大视频商都是用HTTP协议，而HTTP是基于TCP），实时视频。
- TCP保证数据按序到达，提供流量控制和拥塞控制，在网络拥堵的时候会减慢发送字节数，而UDP不管网络是否拥堵。
- TCP是连接的，所以服务是一对一服务，而UDP可以1对1，也可以1对多（多播），也可以多对多。

>TIME_WAIT状态的作用

主动发起关闭连接的一方，才会有 TIME-WAIT 状态。

需要 TIME-WAIT 状态，主要是两个原因：

- 防止具有相同「四元组」的「旧」数据包被收到；
- 保证「被动关闭连接」的一方能被正确的关闭，即保证最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭；

>TCP 和 UDP 应用场景：

由于 TCP 是面向连接，能保证数据的可靠性交付，因此经常用于：

- FTP 文件传输
- HTTP / HTTPS

由于 UDP 面向无连接，它可以随时发送数据，再加上UDP本身的处理既简单又高效，因此经常用于：

- 包总量较少的通信，如 DNS 、SNMP 等
- 视频、音频等多媒体通信
- 广播通信

>为什么 UDP 头部没有「首部长度」字段，而 TCP 头部有「首部长度」字段呢？

原因是 TCP 有可变长的「选项」字段，而 UDP 头部长度则是不会变化的，无需多一个字段去记录 UDP 的首部长度。

>为什么 TIME_WAIT 等待的时间是 2MSL

MSL 是 Maximum Segment Lifetime，报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为 TCP 报文基于是 IP 协议的，而 IP 头中有一个 TTL 字段，是 IP 数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减 1，当此值为 0 则数据报将被丢弃，同时发送 ICMP 报文通知源主机。

MSL 与 TTL 的区别： MSL 的单位是时间，而 TTL 是经过路由跳数。所以 MSL 应该要大于等于 TTL 消耗为 0 的时间，以确保报文已被自然消亡。

TIME_WAIT 等待 2 倍的 MSL，比较合理的解释是： 网络中可能存在来自发送方的数据包，当这些发送方的数据包被接收方处理后又会向对方发送响应，所以一来一回需要等待 2 倍的时间。

>TIME_WAIT 过多有什么危害？

如果服务器有处于 TIME-WAIT 状态的 TCP，则说明是由服务器方主动发起的断开请求。

过多的 TIME-WAIT 状态主要的危害有两种：

- 第一是内存资源占用；
- 第二是对端口资源的占用，一个 TCP 连接至少消耗一个本地端口；

如果服务端 TIME_WAIT 状态过多，占满了所有端口资源，则会导致无法创建新连接。

>大量TIME_WAIT产生的原因及解决办法
原因：对于基于TCP的HTTP协议，关闭TCP连接的是Server端，这样，Server端回进入TIME_WAIT状态，可想而知，对于访问量大的Web Server，会存在大量的TIME_WAIT状态。

- 解决办法：
    - （1）开启socket重用，允许TIME_WAIT的socket重新用于TCP连接
    - （2）开启快速回收
    
>如何优化 TIME_WAIT？

- 打开 net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_timestamps 选项；
    - 引入了时间戳，我们在前面提到的 2MSL问题就不复存在了，因为重复的数据包会因为时间戳过期被自然丢弃。
    - net.ipv4.tcp_tw_reuse要慎用，因为使用了它就必然要打开时间戳的支持 net.ipv4.tcp_timestamps，当客户端与服务端主机时间不同步时，客户端的发送的消息会被直接拒绝掉。小林在工作中就遇到过。。。排查了非常的久
- net.ipv4.tcp_max_tw_buckets
    - 这个值默认为 18000，当系统中处于 TIME_WAIT 的连接一旦超过这个值时，系统就会将所有的 TIME_WAIT 连接状态重置。
    - 这个方法过于暴力，而且治标不治本，带来的问题远比解决的问题多，不推荐使用。
- 程序中使用 SO_LINGER ，应用强制使用 RST 关闭。
    - 通过设置 socket 选项，来设置调用 close 关闭连接行为。
    ``` 
    structlingerso_linger;so_linger.l_onoff = 1;so_linger.l_linger = 0;setsockopt(s, SOL_SOCKET, SO_LINGER, &so_linger,sizeof(so_linger));
    ```
    - 如果l_onoff为非 0， 且l_linger值为 0，那么调用close后，会立该发送一个RST标志给对端，该 TCP 连接将跳过四次挥手，也就跳过了TIME_WAIT状态，直接关闭。
    - 这为跨越TIME_WAIT状态提供了一个可能，不过是一个非常危险的行为，不值得提倡

>针对 TCP 应该如何 Socket 编程？

![Socket 编程](https://pics0.baidu.com/feed/30adcbef76094b3698aacd7744b80bdf8c109def.jpeg?token=8c67151008359d168163b144e051f674)

基于 TCP 协议的客户端和服务器工作

- 服务端和客户端初始化 socket，得到文件描述符；
- 服务端调用 bind，将绑定在 IP 地址和端口;
- 服务端调用 listen，进行监听；
- 服务端调用 accept，等待客户端连接；
- 客户端调用 connect，向服务器端的地址和端口发起连接请求；
- 服务端 accept 返回用于传输的 socket 的文件描述符；
- 客户端调用 write 写入数据；服务端调用 read 读取数据；
- 客户端断开连接时，会调用 close，那么服务端 read 读取数据的时候，就会读取到了 EOF，待处理完数据后，服务端调用 close，表示连接关闭。

>TCP在listen时的参数backlog的意义

- linux内核中会维护两个队列：
    - （1）未完成队列：接收到一个SYN建立连接请求，处于SYN_RCVD状态
    - （2）已完成队列：已完成TCP三次握手过程，处于ESTABLISHED状态
    - 当有一个SYN到来请求建立连接时，就在未完成队列中新建一项。当三次握手过程完成后，就将套接口从未完成队列移动到已完成队列。
    - backlog曾被定义为两个队列的总和的最大值，也曾将backlog的1.5倍作为未完成队列的最大长度
- 一般将backlog指定为5
