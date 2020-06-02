## sorted sets 类型及操作

- sorted set 是 set 的一个升级版本，它在 set 的基础上增加了一个顺序属性，这一属性在添加
修改元素的时候可以指定，每次指定后，zset 会自动重新按新的值调整顺序。可以理解为有
两列的 mysql 表，一列存 value，一列存顺序。操作中 key 理解为 zset 的名字。
- 和 set 一样 sorted set 也是 string 类型元素的集合，不同的是每个元素都会关联一个 double
类型的 score。sorted set 的实现是 skip list 和 hash table 的混合体。
- 当元素被添加到集合中时，一个元素到 score 的映射被添加到 hash table 中，所以给定一个
元素获取 score 的开销是 O(1),另一个 score 到元素的映射被添加到 skip list，并按照 score 排序，所以就可以有序的获取集合中的元素。添加，删除操作开销都是 O(log(N))和 skip list 的
开销一致,redis 的 skip list 实现用的是双向链表,这样就可以逆序从尾部取元素。sorted set 最
经常的使用方式应该是作为索引来使用.我们可以把要排序的字段作为 score 存储，对象的 id
当元素存储。

### zadd
向名称为 key 的 zset 中添加元素 member，score 用于排序。如果该元素已经存在，则根据score 更新该元素的顺序
```
redis 127.0.0.1:6379> zadd myzset 1 "one"
(integer) 1
redis 127.0.0.1:6379> zadd myzset 2 "two"
(integer) 1
redis 127.0.0.1:6379> zadd myzset 3 "two"
(integer) 0
redis 127.0.0.1:6379> zrange myzset 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "3"
```

本例中我们向 myzset 中添加了 one 和 two，并且 two 被设置了 2 次，那么将以最后一次的
设置为准，最后我们将所有元素都显示出来并显示出了元素的 score。

### zrem

删除名称为 key 的 zset 中的元素 member

```
redis 127.0.0.1:6379> zrange myzset 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "3"
redis 127.0.0.1:6379> zrem myzset two
(integer) 1
redis 127.0.0.1:6379> zrange myzset 0 -1 withscores
1) "one"
2) "1"
```

### zincrby
如果在名称为 key 的 zset 中已经存在元素 member，则该元素的 score 增加 increment；否则
向集合中添加该元素，其 score 的值为 increment

```
redis 127.0.0.1:6379> zadd myzset2 1 "one"
(integer) 1
redis 127.0.0.1:6379> zadd myzset2 2 "two"
(integer) 1
redis 127.0.0.1:6379> zincrby myzset2 2 "one"
"3"
redis 127.0.0.1:6379> zrange myzset2 0 -1 withscores
1) "two"
2) "2"
3) "one"
4) "3"
```

### zrank

返回名称为 key 的 zset 中 member 元素的排名(按 score 从小到大排序)即下标

```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zrank myzset3 two
(integer) 1
```

### zrevrank
返回名称为 key 的 zset 中 member 元素的排名(按 score 从大到小排序)即下标
```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zrevrank myzset3 two
(integer) 2
```

### zrevrange
返回名称为 key 的 zset（按 score 从大到小排序）中的 index 从 start 到 end 的所有元素
```
redis 127.0.0.1:6379> zrevrange myzset3 0 -1 withscores
1) "five"
2) "5"
3) "three"
4) "3"
5) "two"
6) "2"
7) "one"
8) "1"
```
首先按 score 从大到小排序，再取出全部元素

### zrangebyscore

返回集合中 score 在给定区间的元素

```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zrangebyscore myzset3 2 3 withscores
1) "two"
2) "2"
3) "three"
4) "3"
```
本例中，返回了 score 在 2~3 区间的元素

### zcount
返回集合中 score 在给定区间的数量

```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zcount myzset3 2 3
(integer) 2
```

本例中，计算了 score 在 2~3 之间的元素数目

### zcard
返回集合中元素个数
```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zcard myzset3
(integer) 4
```
从本例看出 myzset3 这个集全的元素数量是 4

### zscore
返回给定元素对应的 score
```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zscore myzset3 two
"2"
```
此例中我们成功的将 two 的 score 取出来了。

### zremrangebyrank
删除集合中排名在给定区间的元素
```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
7) "five"
8) "5"
redis 127.0.0.1:6379> zremrangebyrank myzset3 3 3
(integer) 1
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"

```
在本例中我们将 myzset3 中按从小到大排序结果的下标为 3 的元素删除了。

### zremrangebyscore
删除集合中 score 在给定区间的元素
```
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
redis 127.0.0.1:6379> zremrangebyscore myzset3 1 2
(integer) 2
redis 127.0.0.1:6379> zrange myzset3 0 -1 withscores
1) "three"
2) "3"
```
在本例中我们将 myzset3 中按从小到大排序结果的 score 在 1~2 之间的元素删除了。