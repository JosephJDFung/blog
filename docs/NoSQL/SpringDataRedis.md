### Spring Data Redis
Spring Data Redis（SDR）框架通过Spring出色的基础架构支持消除了与存储库交互所需的冗余任务和样板代码，从而简化了编写使用Redis键值存储库的Spring应用程序的过程。

[api](https://docs.spring.io/spring-data/redis/docs/2.3.0.RELEASE/api/)

#### 创建
```xml
<dependencies>

  <!-- other dependency elements omitted -->

  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>2.3.0.RELEASE</version>
  </dependency>

</dependencies>

<!-- 在pom.xml中将Spring的版本更改为5.2.6.RELEASE -->
<spring.framework.version>5.2.6.RELEASE</spring.framework.version>
```
- Redis需求
    - Spring Redis需要Redis 2.6或更高版本，Spring Data Redis与Lettuce和Jedis集成，这是两个流行的Redis开源Java库。

- 连接到Redis
    - 使用Redis和Spring时的首要任务之一是通过IoC容器连接到商店。为此，需要Java连接器（或绑定）。无论选择哪种库，您都只需要使用一组Spring Data Redis API（在所有连接器中行为均一）：org.springframework.data.redis.connection软件包及其RedisConnection与RedisConnectionFactory接口以及用于与Redis进行活动连接和检索的连接。


> 配置lettuce连接器

- 将以下内容添加到pom.xml文件dependencies元素中：
```xml
<dependencies>

  <!-- other dependency elements omitted -->

  <dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>5.3.0.RELEASE</version>
  </dependency>

</dependencies>
```

- 创建新的Lettuce连接工厂：
```java
@Configuration
class AppConfig {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    return new LettuceConnectionFactory(new RedisStandaloneConfiguration("server", 6379));
  }
}

```
还可以调整一些特定于Lettuce的连接参数。默认情况下，`LettuceConnection`由`LettuceConnectionFactory`共享创建的所有实例对于所有非阻塞和非事务性操作共享相同的线程安全本机连接。要每次使用专用连接，请将设置`shareNativeConnection`为`false`。`LettuceConnectionFactory`还可以配置为使用` LettucePool`来池化阻塞和事务连接。

### Redis Sentinel支持

为了处理高可用性Redis，Spring Data Redis 使用RedisSentinelConfiguration，如以下示例所示：

```java
/**
 * Jedis
 */
@Bean
public RedisConnectionFactory jedisConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new JedisConnectionFactory(sentinelConfig);
}

/**
 * Lettuce
 */
@Bean
public RedisConnectionFactory lettuceConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new LettuceConnectionFactory(sentinelConfig);
}

```
> RedisSentinelConfiguration也可以使用定义PropertySource，可以设置以下属性：

- 配置属性
    - spring.redis.sentinel.master：主节点名称。
    - spring.redis.sentinel.nodes：以逗号分隔的host：port对列表。
    - spring.redis.sentinel.password：使用Redis Sentinel进行身份验证时要应用的密码

- 有时，需要与其中一个哨兵进行直接互动。使用RedisConnectionFactory.getSentinelConnection()或RedisConnection.getSentinelCommands()可以访问已配置的第一个活动Sentinel。

> sentinel身份验证仅可使用Lettuce进行。


### 通过RedisTemplate处理对象

大多数用户可能会使用`RedisTemplate`及其相应的软件包`org.springframework.data.redis.core`。实际上，由于模板具有丰富的功能集，因此它是Redis模块的中心类。该模板为Redis交互提供了高级抽象。虽然`RedisConnection`提供了接受和返回二进制值（byte数组）的低级方法，但是模板负责序列化和连接管理，使用户无需处理此类细节。


| 接口|描述
-|-
GeoOperations | Redis的地理空间操作的，比如GEOADD，GEORADIUS.
HashOperations | Redis哈希操作
HyperLogLogOperations|Redis的HyperLogLog操作，例如PFADD，PFCOUNT，...
ListOperations|Redis列表操作
SetOperations|Redis SET操作
ValueOperations|Redis字符串（或值）操作
ZSetOperations|Redis zset（或排序集）操作
BoundGeoOperations|Redis键绑定地理空间操作
BoundHashOperations|Redis哈希键绑定操作
BoundKeyOperations|Redis键绑定操作
BoundListOperations|Redis列表键绑定操作
BoundSetOperations|Redis SET键绑定操作
BoundValueOperations|Redis字符串（或值）键绑定操作
BoundZSetOperations|Redis zset（或排序集）键绑定操作

- 配置后，该模板是线程安全的，并且可以在多个实例之间重用。

`RedisTemplate`大多数操作使用基于Java的序列化器。这意味着模板编写或读取的任何对象都将通过Java进行序列化和反序列化。您可以在模板上更改序列化机制，Redis模块提供了几种实现，可在`org.springframework.data.redis.serializer`软件包中找到。您还可以将任何序列化器设置为null，并通过将`enableDefaultSerializer`属性设置为false来将RedisTemplate与原始字节数组一起使用。请注意，模板要求所有键都不为空。但是，只要基础串行器接受这些值，它们就可以为空。

### String-focused 
由于java.lang.String存储在Redis中的键和值非常普遍，因此Redis模块提供了RedisConnection和的两个扩展RedisTemplate，分别是StringRedisConnection（及其DefaultStringRedisConnection实现），并且StringRedisTemplate是用于密集型String操作的便捷的一站式解决方案。除了绑定到String键外，模板和连接还使用StringRedisSerializer下面的键，这意味着存储的键和值是可读的（假设Redis和您的代码使用相同的编码）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:p="http://www.springframework.org/schema/p"
  xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" p:use-pool="true"/>

  <bean id="stringRedisTemplate" class="org.springframework.data.redis.core.StringRedisTemplate" p:connection-factory-ref="jedisConnectionFactory"/>
  ...
</beans>
```

```java
public class Example {

  @Autowired
  private StringRedisTemplate redisTemplate;

  public void addLink(String userId, URL url) {
    redisTemplate.opsForList().leftPush(userId, url.toExternalForm());
  }
}
```

### 序列化器

- 从框架的角度来看，Redis中存储的数据仅为字节。虽然Redis本身支持各种类型，但在大多数情况下，它们是指数据的存储方式，而不是其表示的内容。由用户决定是否将信息转换为字符串或任何其他对象。

> 在Spring Data中，用户（自定义）类型和原始数据之间的转换（反之亦然）在org.springframework.data.redis.serializer包中的Redis中进行处理。
- 该软件包包含两种类型的序列化器，顾名思义，它们负责序列化过程：

    - 基于的两路串行器RedisSerializer。

    - 使用RedisElementReader和的元素读者和作家RedisElementWriter。

- 这些变体之间的主要区别在于，它们RedisSerializer主要byte[]在读者和作家使用时进行序列化ByteBuffer。

- 有多种实现方式：
    - JdkSerializationRedisSerializer，默认用于RedisCache和RedisTemplate。

    - StringRedisSerializer。

> 默认情况下，RedisCache并且RedisTemplate配置为使用Java本机序列化。Java本机序列化以允许由有效载荷引起的远程代码执行而闻名，这些有效载荷利用易受攻击的库和类注入未验证的字节码。在反序列化步骤中，操纵输入可能导致应用程序中不需要的代码执行。因此，请勿在不受信任的环境中使用序列化。通常，我们强烈建议您使用其他任何消息格式（例如JSON）。

> 如果您担心由Java序列化引起的安全漏洞，请考虑在核心JVM级别上使用通用序列化过滤器机制

### 哈希映射

>可以通过在Redis中使用各种数据结构来存储数据。Jackson2JsonRedisSerializer可以转换为JSON格式的对象。理想情况下，可以使用纯键将JSON存储为值。您可以使用Redis哈希来实现结构化对象的更复杂映射。Spring Data Redis提供了各种将数据映射到哈希的策略（取决于用例）：

- 通过使用HashOperations和序列化器直接映射

- 使用Redis仓库

- 使用HashMapper和HashOperations

#### 哈希映射器

哈希映射器是映射对象到`Map<K, V>`转换器。HashMapper适用于Redis哈希。

- 有多种实现方式：

    - BeanUtilsHashMapper使用Spring的BeanUtils。

    - ObjectHashMapper使用对象到哈希映射。

    - Jackson2HashMapper使用FasterXML Jackson。

```java
public class Person {
  String firstname;
  String lastname;

  // …
}

public class HashMapping {

  @Autowired
  HashOperations<String, byte[], byte[]> hashOperations;

  HashMapper<Object, byte[], byte[]> mapper = new ObjectHashMapper();

  public void writeHash(String key, Person person) {

    Map<byte[], byte[]> mappedHash = mapper.toHash(person);
    hashOperations.putAll(key, mappedHash);
  }

  public Person loadHash(String key) {

    Map<byte[], byte[]> loadedHash = hashOperations.entries("key");
    return (Person) mapper.fromHash(loadedHash);
  }
}
```

#### Jackson2HashMapper

Jackson2HashMapper使用FasterXML Jackson提供域对象的Redis哈希映射。 Jackson2HashMapper可以将顶级属性映射为哈希字段名称，并且可以选择将结构展平。简单类型映射为简单值。复杂类型（嵌套对象，集合，MAP等）表示为嵌套JSON。

展平为所有嵌套属性创建单个哈希条目，并尽可能将复杂类型解析为简单类型。

- 拼合要求所有属性名称都不得干扰JSON路径。使用展平时，不支持在地图键中使用点或括号或将其用作属性名称。产生的哈希不能映射回一个对象。

- java.util.Date与java.util.Calendar以毫秒表示。toString如果jackson-datatype-jsr310类路径上的JSR-310日期/时间类型序列化为它们的形式。

### Redis消息传递（Pub / Sub）

Spring Data为Redis提供了专用的消息传递集成，其功能和命名与Spring Framework中的JMS集成相似。

- Redis消息传递可以大致分为两个功能区域：
    - 消息的发布或产生
    - 订阅或消费消息

- 这是通常称为“发布/订阅”（简称“发布/订阅”）的模式的示例。所述RedisTemplate类用于消息生成。对于类似于Java EE的消息驱动bean样式的异步接收，Spring Data提供了专用的消息侦听器容器，该容器用于创建消息驱动的POJO（MDP），并为同步接收提供RedisConnection合同。

在org.springframework.data.redis.connection和org.springframework.data.redis.listener软件包提供了对Redis的消息的核心功能。

#### 发布（发送消息）
要发布消息，可以与其他操作一起使用低级RedisConnection或高级RedisTemplate。这两个实体都提供了该publish方法，该方法接受消息和目标通道作为参数。虽然RedisConnection需要原始数据（字节数组），但是RedisTemplate让任意对象作为消息传递，如下例所示：
```java
// send message through connection RedisConnection con = ...
byte[] msg = ...
byte[] channel = ...
con.publish(msg, channel); // send message through RedisTemplate
RedisTemplate template = ...
template.convertAndSend("hello!", "world");

```

#### 订阅（接收消息）

>在接收方，可以通过直接命名一个频道或多个频道或使用模式匹配来订阅一个或多个频道。后一种方法非常有用，因为它不仅允许使用一个命令创建多个订阅，而且还可以侦听在订阅时尚未创建的频道（只要它们与模式匹配）。

>在底层，RedisConnection提供subscribe和pSubscribe映射Redis命令以分别按通道或按模式进行预订的方法。请注意，可以将多个通道或模式用作参数。要更改连接的预订或查询连接是否在侦听，请RedisConnection提供getSubscription和isSubscribed方法。

- `注意`: Spring Data Redis中的订阅命令被阻止。也就是说，在连接上调用订阅会导致当前线程在开始等待消息时阻塞。仅当取消订阅时才释放线程，这是在另一个线程调用unsubscribe或pUnsubscribe在同一连接上发生的。

> 一旦订阅，连接即开始等待消息。仅允许添加新订阅，修改现有订阅以及取消现有订阅的命令。调用比其他任何东西subscribe，pSubscribe，unsubscribe，或pUnsubscribe抛出异常。

>为了订阅消息，需要实现MessageListener回调。每次收到新消息时，该方法都会调用回调并运行用户代码onMessage。该接口不仅可以访问实际消息，还可以访问通过其接收到的消息的通道，以及订阅使用的与该通道匹配的模式（如果有）。此信息使被呼叫者不仅可以按内容来区分各种消息，还可以检查其他细节。

#### 消息侦听器容器
- 由于其阻塞性质，低级别订阅并不吸引人，因为它需要每个侦听器都进行连接和线程管理。为了减轻这个问题，Spring Data提供了RedisMessageListenerContainer，可以完成所有繁重的工作。如果您熟悉EJB和JMS，则应该熟悉这些概念，因为它被设计为尽可能接近Spring Framework及其消息驱动的POJO（MDP）的支持。

- RedisMessageListenerContainer充当消息侦听器容器。它用于从Redis通道接收消息并驱动MessageListener注入到该通道中的实例。侦听器容器负责消息接收的所有线程，并分派到侦听器中进行处理。消息侦听器容器是MDP与消息传递提供程序之间的中介，并负责注册接收消息，资源获取和释放，异常转换等。这使您作为应用程序开发人员，可以编写与接收消息（并对其进行响应）相关的（可能很复杂的）业务逻辑，并将样板Redis基础结构问题委托给框架。

- 此外，为了最大程度地减少应用程序占用空间，RedisMessageListenerContainer即使多个侦听器不共享预订，也可以让多个侦听器共享一个连接和一个线程。因此，无论应用程序跟踪多少侦听器或通道，运行时间成本在整个生命周期中都保持不变。此外，该容器允许更改运行时配置，以便您可以在应用程序运行时添加或删除侦听器，而无需重新启动。此外，容器使用惰性订阅方法，RedisConnection仅在需要时才使用。如果所有侦听器都已取消订阅，则将自动执行清除，然后释放线程。

- 为了帮助解决消息的异步特性，容器需要使用java.util.concurrent.Executor（或Spring的TaskExecutor）调度消息。根据负载，侦听器的数量或运行时环境，应更改或调整执行程序，以更好地满足您的需求。特别是在托管环境（例如应用程序服务器）中，强烈建议选择一种适当的TaskExecutor方式来利用其运行时。

#### MessageListenerAdapter
`MessageListenerAdapter`类是Spring的异步支持消息的最后一个组件。简而言之，它使您几乎可以将任何类公开为MDP（尽管存在一些约束）。

```java
public interface MessageDelegate {
  void handleMessage(String message);
  void handleMessage(Map message); void handleMessage(byte[] message);
  void handleMessage(Serializable message);
  // pass the channel/pattern as well
  void handleMessage(Serializable message, String channel);
 }
```

请注意，尽管该接口未扩展该MessageListener接口，但仍可以通过使用MessageListenerAdapter该类将其用作MDP 。还要注意如何使用各种消息处理方法是根据强类型的内容不同的Message类型，他们可以接收和处理。另外，可以将发送消息的通道或模式作为type的第二个参数传递给方法String：

```java
public class DefaultMessageDelegate implements MessageDelegate {
  // implementation elided for clarity...
}
```

MessageDelegate接口的上述实现（上述DefaultMessageDelegate类）如何完全没有 Redis依赖项。这确实是我们使用以下配置将其制作为MDP的POJO：

```xml
<?xml version="1.0" encoding="UTF-8"?>
 <beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:redis="http://www.springframework.org/schema/redis"
   xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
   http://www.springframework.org/schema/redis https://www.springframework.org/schema/redis/spring-redis.xsd">

<!-- the default ConnectionFactory -->
<redis:listener-container>
  <!-- the method attribute can be skipped as the default method name is "handleMessage" -->
  <redis:listener ref="listener" method="handleMessage" topic="chatroom" />
</redis:listener-container>

<bean id="listener" class="redisexample.DefaultMessageDelegate"/>
 ...
<beans>
```

前面的示例使用Redis名称空间声明消息侦听器容器，并自动将POJO注册为侦听器。完整的定义如下：

```xml
<bean id="messageListener" class="org.springframework.data.redis.listener.adapter.MessageListenerAdapter">
  <constructor-arg>
    <bean class="redisexample.DefaultMessageDelegate"/>
  </constructor-arg>
</bean>

<bean id="redisContainer" class="org.springframework.data.redis.listener.RedisMessageListenerContainer">
  <property name="connectionFactory" ref="connectionFactory"/>
  <property name="messageListeners">
    <map>
      <entry key-ref="messageListener">
        <bean class="org.springframework.data.redis.listener.ChannelTopic">
          <constructor-arg value="chatroom"/>
        </bean>
      </entry>
    </map>
  </property>
</bean>
```

每次接收到消息时，适配器都会自动且透明地在RedisSerializer底层格式和所需对象类型之间执行转换（使用configure ）。容器捕获并处理由方法调用引起的任何异常（默认情况下，异常会被记录）。


