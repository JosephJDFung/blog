## redis基本命令

### strings 类型及操作

- string 是最简单的类型，你可以理解成与 Memcached 是一模一样的类型，一个 key 对应一个
value，其上支持的操作与 Memcached 的操作类似。但它的功能更丰富。
- string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据，比如 jpg 图片或者序
列化的对象。从内部实现来看其实 string 可以看作 byte 数组，最大上限是 1G 字节，下面是
string 类型的定义:
```c
struct sdshdr {
 long len;
 long free;
 char buf[];
};
```

- len 是 buf 数组的长度。
- free 是数组中剩余可用字节数，由此可以理解为什么 string 类型是二进制安全的了，因为它
本质上就是个 byte 数组，当然可以包含任何数据了
- buf 是个 char 数组用于存贮实际的字符串内容，其实 char 和 c#中的 byte 是等价的，都是一
个字节。
- 另外 string 类型可以被部分命令按 int 处理.比如 incr 等命令，如果只用 string 类型，redis 就
可以被看作加上持久化特性的 memcached。

#### set
设置 key 对应的值为 string 类型的 value。
例如我们添加一个 name= Jack 的键值对，可以这样做:

```
redis 127.0.0.1:6379> set name Jack
```
#### setnx

设置 key 对应的值为 string 类型的 value。如果 key 已经存在，返回 0，nx 是 not exist 的意思。
例如我们添加一个 name= Jack_new 的键值对，可以这样做:

```
redis 127.0.0.1:6379> get name
"Jack"
redis 127.0.0.1:6379> setnx name Jack_new
(integer) 0
redis 127.0.0.1:6379> get name
"Jack"
```
由于原来 name 有一个对应的值，所以本次的修改不生效，且返回码是 0。

#### setex

设置 key 对应的值为 string 类型的 value，并指定此键值对应的有效期。
例如我们添加一个 haircolor= red 的键值对，并指定它的有效期是 10 秒，可以这样做:

```
redis 127.0.0.1:6379> setex haircolor 10 red
OK
redis 127.0.0.1:6379> get haircolor
"red"
redis 127.0.0.1:6379> get haircolor
(nil)
```

#### setrange

设置指定 key 的 value 值的子字符串。
```
redis 127.0.0.1:6379> setrange name 8 gmail.com
```

#### mset
一次设置多个 key 的值，成功返回 ok 表示所有的值都设置了，失败返回 0 表示没有任何值
被设置。
```
redis 127.0.0.1:6379> mset key1 v1 key2 v2
```

#### msetnx
一次设置多个 key 的值，成功返回 ok 表示所有的值都设置了，失败返回 0 表示没有任何值
被设置，但是不会覆盖已经存在的 key。

#### get
获取 key 对应的 string 值,如果 key 不存在返回 nil。
例如我们获取一个库中存在的键 name，可以很快得到它对应的 value

#### getset
设置 key 的值，并返回 key 的旧值，如果 key 不存在，那么将返回 nil

```
redis 127.0.0.1:6379> get name
"Jack"
redis 127.0.0.1:6379> getset name Jack_new
"Jack"
redis 127.0.0.1:6379> get name
"Jack_new"
```

#### getrange

获取指定 key 的 value 值的子字符串。
- 字符串左面下标是从 0 开始的
- 字符串右面下标是从-1 开始的
- 当下标超出字符串长度时，将默认为是同方向的最大下标

#### mget
一次获取多个 key 的值，如果对应 key 不存在，则对应返回 nil。

#### incr

对 key 的值做加加操作,并返回新的值。注意 incr 一个不是 int 的 value 会返回错误，incr 一个不存在的 key，则设置 key 为 1

```
redis 127.0.0.1:6379> set age 20
OK
redis 127.0.0.1:6379> incr age
(integer) 21
redis 127.0.0.1:6379> get age
"21"
```

#### incrby
同 incr 类似，加指定值 ，key 不存在时候会设置 key，并认为原来的 value 是 0

```
redis 127.0.0.1:6379> get age
"21"
redis 127.0.0.1:6379> incrby age 5
(integer) 26
redis 127.0.0.1:6379> get age
"26"
```

#### decr
对 key 的值做的是减减操作，decr 一个不存在 key，则设置 key 为-1

#### decrby
- 同 decr，减指定值。
- decrby 完全是为了可读性，我们完全可以通过 incrby 一个负值来实现同样效果，反之一样。

#### append

给指定 key 的字符串值追加 value,返回新字符串值的长度。
例如我们向 name 的值追加一个@126.com 字符串，那么可以这样做:

```
redis 127.0.0.1:6379> append name @126.com
```


#### strlen
取指定 key 的 value 值的长度。

```
redis 127.0.0.1:6379> strlen name
```
