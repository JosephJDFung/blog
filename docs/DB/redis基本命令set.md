## sets 类型及操作

- set 是集合，和我们数学中的集合概念相似，对集合的操作有添加删除元素，有对多个集合
求交并差等操作，操作中 key 理解为集合的名字。
- set 的是通过 hash table 实现的，所以添加、删除和查找的复杂度都是 O(1)。hash table 会随
着添加或者删除自动的调整大小。需要注意的是调整 hash table 大小时候需要同步（获取写
锁）会阻塞其他读写操作，可能不久后就会改用跳表（skip list）来实现，跳表已经在 sorted 
set 中使用了。关于 set 集合类型除了基本的添加删除操作，其他有用的操作还包含集合的
取并集(union)，交集(intersection)，差集(difference)。通过这些操作可以很容易的实现 sns
中的好友推荐和 blog 的 tag 功能。

### sadd
向名称为 key 的 set 中添加元素
```
redis 127.0.0.1:6379> sadd myset "hello"
(integer) 1
redis 127.0.0.1:6379> sadd myset "world"
(integer) 1
redis 127.0.0.1:6379> sadd myset "world"
(integer) 0
redis 127.0.0.1:6379> smembers myset
1) "world"
2) "hello"
```
用 smembers 来查看 myset 中的所有元素。

### srem
删除名称为 key 的 set 中的元素 member
```
redis 127.0.0.1:6379> sadd myset2 "one"
(integer) 1
redis 127.0.0.1:6379> sadd myset2 "two"
(integer) 1
redis 127.0.0.1:6379> sadd myset2 "three"
(integer) 1
redis 127.0.0.1:6379> srem myset2 "one"
(integer) 1
redis 127.0.0.1:6379> srem myset2 "four"
(integer) 0
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
```

### spop
随机返回并删除名称为 key 的 set 中一个元素
```
redis 127.0.0.1:6379> sadd myset3 "one"
(integer) 1
redis 127.0.0.1:6379> sadd myset3 "two"
(integer) 1
redis 127.0.0.1:6379> sadd myset3 "three"
(integer) 1
redis 127.0.0.1:6379> spop myset3
"three"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
```
向 myset3 中添加了三个元素后，再调用 spop 来随机删除一个元素，可以看到
three 元素被删除了。

### sdiff
返回所有给定 key 与第一个 key 的差集
```
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> sdiff myset2 myset3
1) "three"
```
可以将 myset2 和 myset3 换个顺序来看一下结果:

```
redis 127.0.0.1:6379> sdiff myset3 myset2
1) "one"
```

### sdiffstore
返回所有给定 key 与第一个 key 的差集，并将结果存为另一个 key
```
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> sdiffstore myset4 myset2 myset3
(integer) 1
redis 127.0.0.1:6379> smembers myset4
1) "three"
```
### sinter
返回所有给定 key 的交集
```
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> sinter myset2 myset3
1) "two"
```

### sinterstore
返回所有给定 key 的交集，并将结果存为另一个 key

### sunion

返回所有给定 key 的并集

```
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> sunion myset2 myset3
1) "three"
2) "one"
3) "two"
```
### sunionstore
返回所有给定 key 的并集，并将结果存为另一个 key

### smove
从第一个 key 对应的 set 中移除 member 并添加到第二个对应 set 中
```
redis 127.0.0.1:6379> smembers myset2
1) "three"
2) "two"
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> smove myset2 myset7 three
(integer) 1
redis 127.0.0.1:6379> smembers myset7
1) "three"
```
通过本例可以看到，myset2 的 three 被移到 myset7 中了
### scard
返回名称为 key 的 set 的元素个数
```
redis 127.0.0.1:6379> scard myset2
(integer) 1
```
通过本例可以看到，myset2 的成员数量为 1

### sismember
测试 member 是否是名称为 key 的 set 的元素
```
redis 127.0.0.1:6379> smembers myset2
1) "two"
redis 127.0.0.1:6379> sismember myset2 two
(integer) 1
redis 127.0.0.1:6379> sismember myset2 one
(integer) 0
```
通过本例可以看到，two 是 myset2 的成员，而 one 不是。

### srandmember

随机返回名称为 key 的 set 的一个元素，但是不删除元素

```
redis 127.0.0.1:6379> smembers myset3
1) "two"
2) "one"
redis 127.0.0.1:6379> srandmember myset3
"two"
redis 127.0.0.1:6379> srandmember myset3
"one"
```
