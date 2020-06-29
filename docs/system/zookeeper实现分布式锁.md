# zookeeper实现分布式锁

zookeeper中关于节点的性质：

- 有序节点：假如当前有一个父节点为/lock，我们可以在这个父节点下面创建子节点；zookeeper提供了一个可选的有序特性，例如我们可以创建子节点“/lock/node-”并且指明有序，那么zookeeper在生成子节点时会根据当前的子节点数量自动添加整数序号，也就是说如果是第一个创建的子节点，那么生成的子节点为/lock/node-0000000000，下一个节点则为/lock/node-0000000001，依次类推。

- 临时节点：客户端可以建立一个临时节点，在会话结束或者会话超时后，zookeeper会自动删除该节点。

- 事件监听：在读取数据时，我们可以同时对节点设置事件监听，当节点数据或结构变化时，zookeeper会通知客户端。

- 当前zookeeper有如下四种事件：
    - 节点创建；
    - 节点删除；
    - 节点数据修改；
    - 子节点变更。

## zookeeper实现分布式锁的算法流程

假设锁空间的根节点为/lock：

1. 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推。

2. 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听/lock的子节点变更消息，获得子节点变更通知后重复此步骤直至获得锁；

3. 执行业务代码；

4. 完成业务流程后，删除对应的子节点释放锁。

>步骤1中创建的临时节点能够保证在故障的情况下锁也能被释放，考虑这么个场景：假如客户端a当前创建的子节点为序号最小的节点，获得锁之后客户端所在机器宕机了，客户端没有主动删除子节点；如果创建的是永久的节点，那么这个锁永远不会释放，导致死锁；由于创建的是临时节点，客户端宕机后，过了一定时间zookeeper没有收到客户端的心跳包判断会话失效，将临时节点删除从而释放锁。

>在步骤2中获取子节点列表与设置监听这两步操作的原子性问题，考虑这么个场景：客户端a对应子节点为/lock/lock-0000000000，客户端b对应子节点为/lock/lock-0000000001，客户端b获取子节点列表时发现自己不是序号最小的，但是在设置监听器前客户端a完成业务流程删除了子节点/lock/lock-0000000000，客户端b设置的监听器岂不是丢失了这个事件从而导致永远等待了？这个问题不存在的。因为zookeeper提供的API中设置监听器的操作与读操作是原子执行的，也就是说在读子节点列表时同时设置监听器，保证不会丢失事件。

>这个算法有个极大的优化点：假如当前有1000个节点在等待锁，如果获得锁的客户端释放锁时，这1000个客户端都会被唤醒，这种情况称为“羊群效应”；在这种羊群效应中，zookeeper需要通知1000个客户端，这会阻塞其他的操作，最好的情况应该只唤醒新的最小节点对应的客户端。应该怎么做呢？在设置事件监听时，每个客户端应该对刚好在它之前的子节点设置事件监听，例如子节点列表为/lock/lock-0000000000、/lock/lock-0000000001、/lock/lock-0000000002，序号为1的客户端监听序号为0的子节点删除消息，序号为2的监听序号为1的子节点删除消息。

## 调整后的分布式锁算法流程如下：

1. 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推；

2. 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听刚好在自己之前一位的子节点删除消息，获得子节点变更通知后重复此步骤直至获得锁；

3. 执行业务代码；

4. 完成业务流程后，删除对应的子节点释放锁。

## Curator实现

我们可以直接使用curator这个开源项目提供的zookeeper分布式锁实现。

```xml
<dependency>

    <groupId>org.apache.curator</groupId>

    <artifactId>curator-recipes</artifactId>

    <version>4.0.0</version>

</dependency>
```

```java
public static void main(String[] args) throws Exception {

    //创建zookeeper的客户端

    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

    CuratorFramework client = CuratorFrameworkFactory.newClient("10.21.41.181:2181,10.21.42.47:2181,10.21.49.252:2181", retryPolicy);
    client.start();

    //创建分布式锁, 锁空间的根节点路径为/curator/lock

    InterProcessMutex mutex = new InterProcessMutex(client, "/curator/lock");

    mutex.acquire();

    //获得了锁, 进行业务流程

    System.out.println("Enter mutex");

    //完成业务流程, 释放锁

    mutex.release();

    //关闭客户端

    client.close();

}

```

>关键的核心操作就只有mutex.acquire()和mutex.release()

- zookeper的实现主要有下面四类类:
    - InterProcessMutex：分布式可重入排它锁
    - InterProcessSemaphoreMutex：分布式排它锁
    - InterProcessReadWriteLock：分布式读写锁
    - InterProcessMultiLock：将多个锁作为单个实体管理的容器

## menagerie实现

menagerie基于Zookeeper实现了java.util.concurrent包的一个分布式版本。这个封装是更大粒度上对各种分布式一致性使用场景的抽象。其中最基础和常用的是一个分布式锁的实现： 

- org.menagerie.locks.ReentrantZkLock，通过ZooKeeper的全局有序的特性和EPHEMERAL_SEQUENTIAL类型znode的支持，实现了分布式锁。

- 具体做法是：不同的client上每个试图获得锁的线程，都在相同的basepath下面创建一个EPHEMERAL_SEQUENTIAL的node。
    - EPHEMERAL表示要创建的是临时znode，创建连接断开时会自动删除；
    - SEQUENTIAL表示要自动在传入的path后面缀上一个自增的全局唯一后缀,作为最终的path。

因此对不同的请求ZK会生成不同的后缀，并分别返回带了各自后缀的path给各个请求。因为ZK全局有序的特性，不管client请求怎样先后到达，在ZKServer端都会最终排好一个顺序，因此自增后缀最小的那个子节点，就对应第一个到达ZK的有效请求。然后client读取basepath下的所有子节点和ZK返回给自己的path进行比较，当发现自己创建的sequential node的后缀序号排在第一个时，就认为自己获得了锁；否则的话，就认为自己没有获得锁。这时肯定是有其他并发的并且是没有断开的client/线程先创建了node。
