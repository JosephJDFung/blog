## hash 类型及操作

Redis hash 是一个 string 类型的 field 和 value 的映射表.它的添加、删除操作都是 O(1)（平均）。
hash 特别适合用于存储对象。相较于将对象的每个字段存成单个 string 类型。将一个对象存
储在 hash 类型中会占用更少的内存，并且可以更方便的存取整个对象。省内存的原因是新
建一个 hash 对象时开始是用 zipmap（又称为 small hash）来存储的。这个 zipmap 其实并不
是 hash table，但是 zipmap 相比正常的 hash 实现可以节省不少 hash 本身需要的一些元数据
存储开销。尽管 zipmap 的添加，删除，查找都是 O(n)，但是由于一般对象的 field 数量都不太多。所以使用 zipmap 也是很快的,也就是说添加删除平均还是 O(1)。如果 field 或者 value
的大小超出一定限制后，Redis 会在内部自动将 zipmap 替换成正常的 hash 实现. 这个限制可
以在配置文件中指定
- hash-max-zipmap-entries 64 #配置字段最多 64 个
- hash-max-zipmap-value 512 #配置 value 最大为 512 字节

### hset
设置 hash field 为指定值，如果 key 不存在，则先创建。
```
redis 127.0.0.1:6379> hset myhash field1 Hello
(integer) 1
```
### hsetnx

设置 hash field 为指定值，如果 key 不存在，则先创建。如果 field 已经存在，返回 0，nx 是
not exist 的意思。

```
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 1
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 0

```
### hmset

同时设置 hash 的多个 field。

```
redis 127.0.0.1:6379> hmset myhash field1 Hello field2 World
OK
```

### hget
获取指定的 hash field。
```
redis 127.0.0.1:6379> hget myhash field1
"Hello"
redis 127.0.0.1:6379> hget myhash field2
"World"
redis 127.0.0.1:6379> hget myhash field3
(nil)
```
### hmget
获取全部指定的 hash filed。
```
redis 127.0.0.1:6379> hmget myhash field1 field2 field3
1) "Hello"
2) "World"
3) (nil)
```
### hincrby

指定的 hash filed 加上给定值。
```
redis 127.0.0.1:6379> hset myhash field3 20
(integer) 1
redis 127.0.0.1:6379> hget myhash field3
"20"
redis 127.0.0.1:6379> hincrby myhash field3 -8
(integer) 12
redis 127.0.0.1:6379> hget myhash field3
"12"
```
将 field3 的值从 20 降到了 12，即做了一个减 8 的操作。


### hexists

测试指定 field 是否存在。
```
redis 127.0.0.1:6379> hexists myhash field1
(integer) 1
redis 127.0.0.1:6379> hexists myhash field9
(integer) 0
```
通过上例可以说明 field1 存在，但 field9 是不存在的。


### hlen
返回指定 hash 的 field 数量。
```
redis 127.0.0.1:6379> hlen myhash
(integer) 4
```
### hdel
删除指定 hash 的 field 。
```
redis 127.0.0.1:6379> hlen myhash
(integer) 4
redis 127.0.0.1:6379> hdel myhash field1
(integer) 1
redis 127.0.0.1:6379> hlen myhash
(integer) 3
```

### hkeys
返回 hash 的所有 field。
```
redis 127.0.0.1:6379> hkeys myhash
1) "field2"
2) "field"
3) "field3"
```
### hvals
返回 hash 的所有 value。
```
redis 127.0.0.1:6379> hvals myhash
1) "World"
2) "Hello"
3) "12"
```
###  hgetall

获取某个 hash 中全部的 filed 及 value。
```
redis 127.0.0.1:6379> hgetall myhash
1) "field2"
2) "World"
3) "field"
4) "Hello"
5) "field3"
6) "12"
```








