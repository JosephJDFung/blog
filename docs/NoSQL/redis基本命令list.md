### lists 类型及操作
- list 是一个链表结构，主要功能是 push、pop、获取一个范围的所有值等等，操作中 key 理
解为链表的名字。
- Redis 的 list 类型其实就是一个每个子元素都是 string 类型的双向链表。链表的最大长度是(2
的 32 次方)。我们可以通过 push,pop 操作从链表的头部或者尾部添加删除元素。这使得 list
既可以用作栈，也可以用作队列。

- 有意思的是 list 的 pop 操作还有阻塞版本的，当我们`[lr]pop` 一个 list 对象时，如果 list 是空，
或者不存在，会立即返回 nil。但是阻塞版本的 `b[lr]pop` 可以则可以阻塞，当然可以加超时时
间，超时后也会返回 nil。为什么要阻塞版本的 pop 呢，主要是为了避免轮询。举个简单的
例子如果我们用 list 来实现一个工作队列。执行任务的 thread 可以调用阻塞版本的 pop 去获
取任务这样就可以避免轮询去检查是否有任务存在。当任务来时候工作线程可以立即返回，
也可以避免轮询带来的延迟。

###  lpush
在 key 对应 list 的头部添加字符串元素
- 我们先插入了一个 world，然后在 world 的头部插入了一个 hello。其中 lrange 是用于取 mylist 的内容。
```
redis 127.0.0.1:6379> lpush mylist "world"
(integer) 1
redis 127.0.0.1:6379> lpush mylist "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "world"

```

### rpush

在 key 对应 list 的尾部添加字符串元素

### linsert
在 key 对应 list 的特定位置之前或之后添加字符串元素

```
redis 127.0.0.1:6379> rpush mylist3 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist3 "world"
(integer) 2
redis 127.0.0.1:6379> linsert mylist3 before "world" "there"
(integer) 3
redis 127.0.0.1:6379> lrange mylist3 0 -1
1) "hello"
2) "there"
3) "world"
```
在此处我们先插入了一个 hello，然后在 hello 的尾部插入了一个 world，然后又在 world 的
前面插入了 there。

### lset

设置 list 中指定下标的元素值(下标从 0 开始)

```
redis 127.0.0.1:6379> rpush mylist4 "one"
(integer) 1
redis 127.0.0.1:6379> rpush mylist4 "two"
(integer) 2
redis 127.0.0.1:6379> rpush mylist4 "three"
(integer) 3
redis 127.0.0.1:6379> lset mylist4 0 "four"
OK
redis 127.0.0.1:6379> lset mylist4 -2 "five"
OK
redis 127.0.0.1:6379> lrange mylist4 0 -1
1) "four"
2) "five"
3) "three"
```

### lrem
从 key 对应 list 中删除 count 个和 value 相同的元素。
count>0 时，按从头到尾的顺序删除
```
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist5 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist5 2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "foo"
2) "hello"
```

count<0 时，按从尾到头的顺序删除

```
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist6 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist6 -2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "foo"
```

count=0 时，删除全部

```
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist7 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist7 0 "hello"
(integer) 3
redis 127.0.0.1:6379> lrange mylist7 0 -1
1) "foo"
```

###  ltrim

保留指定 key 的值范围内的数据

```
redis 127.0.0.1:6379> rpush mylist8 "one"
(integer) 1
redis 127.0.0.1:6379> rpush mylist8 "two"
(integer) 2
redis 127.0.0.1:6379> rpush mylist8 "three"
(integer) 3
redis 127.0.0.1:6379> rpush mylist8 "four"
(integer) 4
redis 127.0.0.1:6379> ltrim mylist8 1 -1
OK
redis 127.0.0.1:6379> lrange mylist8 0 -1
1) "two"
2) "three"
3) "four"
```

###  lpop
从 list 的头部删除元素，并返回删除元素

###  rpop

从 list 的尾部删除元素，并返回删除元素

### rpoplpush
从第一个 list 的尾部移除元素并添加到第二个 list 的头部,最后返回被移除的元素值，整个操
作是原子的.如果第一个 list 是空或者不存在返回 nil

```
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
3) "hello"
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "foo"
redis 127.0.0.1:6379> rpoplpush mylist5 mylist6
"hello"
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "hello"
3) "foo"
```

###  lindex
返回名称为 key 的 list 中 index 位置的元素
```
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
redis 127.0.0.1:6379> lindex mylist5 0
"three"
redis 127.0.0.1:6379> lindex mylist5 1
"foo"
```

### llen

返回 key 对应 list 的长度