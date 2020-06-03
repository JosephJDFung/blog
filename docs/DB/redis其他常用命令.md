## redis其他常用命令
Redis 提供了丰富的命令（command）对数据库和各种数据类型进行操作，这些 command
可以在 Linux 终端使用。在编程时，比如各类语言包，这些命令都有对应的方法。下面将 Redis提供的命令做一总结。

### 键值相关命令
#### keys
返回满足给定 pattern 的所有 key
```
redis 127.0.0.1:6379> keys *
1) "myzset2"
2) "myzset3"
3) "mylist"
4) "myset2"
5) "myset3"
6) "myset4"
7) "k_zs_1"
8) "myset5"
```
- 用表达式*，代表取出所有的 key
- 用表达式 mylist*，代表取出所有以 mylist 开头的 key

#### exists
确认一个 key 是否存在
```
redis 127.0.0.1:6379> exists age
(integer) 0
```

#### expire
设置一个 key 的过期时间(单位:秒)
```
redis 127.0.0.1:6379> expire addr 10
(integer) 1
redis 127.0.0.1:6379> ttl addr
(integer) 8
redis 127.0.0.1:6379> ttl addr
(integer) 1
redis 127.0.0.1:6379> ttl addr
(integer) -1
```
在本例中，我们设置 addr 这个 key 的过期时间是 10 秒，然后我们不断的用 ttl 来获取这个
key 的有效时长，直至为-1 说明此值已过期

#### del
删除一个 key
```
redis 127.0.0.1:6379> del age
(integer) 1
```
#### move
将当前数据库中的 key 转移到其它数据库中

#### persist
移除给定 key 的过期时间
#### rename
重命名 key
```
redis 127.0.0.1:6379[1]> rename age age_new
OK
```

#### type

返回值的类型
```
redis 127.0.0.1:6379> type myzset2
zset
redis 127.0.0.1:6379> type mylist
list
```

### 服务器相关命令

#### ping
测试连接是否存活

```
redis 127.0.0.1:6379> ping
PONG
//执行下面命令之前，我们停止 redis 服务器
redis 127.0.0.1:6379> ping
Could not connect to Redis at 127.0.0.1:6379: Connection refused
//执行下面命令之前，我们启动 redis 服务器
not connected> ping
PONG
```
- 第一个 ping 时，说明此连接正常
- 第二个 ping 之前，我们将 redis 服务器停止，那么 ping 是失败的
- 第三个 ping 之前，我们将 redis 服务器启动，那么 ping 是成功的

#### select
选择数据库。Redis 数据库编号从 0~15，我们可以选择任意一个数据库来进行数据的存取。

#### quit
退出连接。

#### dbsize
返回当前数据库中 key 的数目。

#### config get
获取服务器配置信息。
```
redis 127.0.0.1:6379> config get dir
1) "dir"
2) "/root/4setup/redis-2.2.12"
```
本例中我们获取了 dir 这个参数配置的值，如果想获取全部参数据的配置值也很简单，只需
执行”config get *”即可将全部的值都显示出来。
#### flushdb
删除当前选择数据库中的所有 key。
```
redis 127.0.0.1:6379> dbsize
(integer) 18
redis 127.0.0.1:6379> flushdb
OK
redis 127.0.0.1:6379> dbsize
(integer) 0
```

#### flushall
删除所有数据库中的所有 key。