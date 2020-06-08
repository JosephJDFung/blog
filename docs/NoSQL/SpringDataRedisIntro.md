## spring data redis 补充

### ConnectionFactory
就像所有的数据库连接，XXXXConnectionFactory就是连接工厂，通过配置单台服务器或者连接池（pool）的方式获取redis服务器的连接。


### RedisTemplate和StringRedisTemplate
就像Spring提供的JDBC，hibernate和ibatis的template一样，spring-data-redis也提供了一个基础的泛型RedisTemplate供开发者可以快速的利用代码完成基础的crud工作。而StringRedisTemplate则提供了最常用的String类型的实现。在实践中可以考虑完全省去dao层的设计，直接在service层注入相应的template实例。

1. 两者的关系是StringRedisTemplate继承RedisTemplate。

2. 两者的数据是不共通的；也就是说StringRedisTemplate只能管理StringRedisTemplate里面的数据，RedisTemplate只能管理RedisTemplate中的数据。

3. SDR默认采用的序列化策略有两种，一种是String的序列化策略，一种是JDK的序列化策略。
StringRedisTemplate默认采用的是String的序列化策略
RedisTemplate默认采用的是JDK的序列化策略

### Operations

- opsForXXX和boundXXXOps的区别
    - XXX为value的类型，前者获取一个operator，但是没有指定操作的对象（key），可以在一个连接（事务）内操作多个key以及对应的value；后者获取了一个指定操作对象（key）的operator，在一个连接（事务）内只能操作这个key对应的value。

### NoSQL数据库设计

>不持久化业务实体

- 一般来讲业务实体随着业务的变更会有较频繁的变化，在传统的基于sql的数据库中，通过ORM有较完善和简便的解决方案。而在Redis中，业务实体是通过序列化成字节数组的方式保存，然后通过反序列化来获取该实体，如果实体发生了变化，将发生无法正确序列化的问题。
- 综上所述，我们建议只用Redis来持久化字符串或者Map形式的数据。当然如果有以下两种情况则可以特殊考虑：
    - 实体简单并且没有太重要的业务意义，或者在很长时间内不会变化
    - 实体已经在sql数据库中持久化，仅仅通过Redis进行缓存


>Key的设计

除了官方最佳实践中通过业务单元描述和id通过冒号连接以外，考虑到我们会实际使用中（尤其是测试环境）会有多个应用同时使用一个Redis数据库的情况，建议在Key的最前面加上一个项目名称的顶级命名空间。 
- 例如，"pandora:catagory:5:items"或者"mercury:totalads"

[NOSQL 数据建模技术](https://coolshell.cn/articles/7270.html)