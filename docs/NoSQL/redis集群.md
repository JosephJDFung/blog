## redis集群

- 主从配置
- Sentinel模式
- Cluster模式

### 主从配置
在主从复制中，数据库分为两类：主数据库(master)和从数据库(slave)。

* 主数据库可以进行读写操作，当写操作导致数据变化时会将数据同步给从数据库。
* 从数据库一般都是只读，并且接收主数据库同步过来的数据

* 一个master可以拥有多个slave，一个slave只能对应一个master

* slave挂了不影响其他slave的读和master的读和写，重新启动后会将数据从master同步过来

* master挂了以后，不影响slave的读，但redis不再提供写服务，master重启后redis将重新对外提供写服务

* master挂了以后，不会在slave节点中重新选一个master

#### slaver

```
# vim /usr/local/redis/redis.conf
daemonize yes               #允许后台启动
logfile "/usr/local/redis/redis.log"   #日志路径
replicaof 127.0.0.1 6379
masterauth 123456   #slave连接master密码，master可省略
requirepass 123456  #设置master连接密码，slave可省略

```
- 5.0版本使用REPLICAOF代替了之前版本的SLAVEOF,如果使用5.0及之后版本,则建议新命令REPLICAOF。

启动redis
```
systemctl start redis
```
- 查看集群状态：
```
info replication
```

#### Sentinel模式
主从模式的弊端就是不具备高可用性，当master挂掉以后，Redis将不能再对外提供写入操作。
* sentinel模式是建立在主从模式的基础上，如果只有一个Redis节点，sentinel就没有任何意义

* 当master挂了以后，sentinel会在slave中选择一个做为master，并修改它们的配置文件，其他slave的配置文件也会被修改

* 当master重新启动后，它将不再是master而是做为slave接收新的master的同步数据

* sentinel因为也是一个进程有挂掉的可能，所以sentinel也会启动多个形成一个sentinel集群

* 多sentinel配置的时候，sentinel之间也会自动监控

* 当主从模式配置密码时，sentinel也会同步将配置信息修改到配置文件中

* 一个sentinel或sentinel集群可以管理多个主从Redis，多个sentinel也可以监控同一个redis

* sentinel和Redis部署需要在不同机器部署

#### 工作机制

* 每个sentinel以每秒钟一次的频率向它所知的master，slave以及其他sentinel实例发送一个 PING 命令 

* 如果一个实例距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被sentinel标记为主观下线。 

* 如果一个master被标记为主观下线，则正在监视这个master的所有sentinel要以每秒一次的频率确认master的确进入了主观下线状态

* 当有足够数量的sentinel（大于等于配置文件指定的值）在指定的时间范围内确认master的确进入了主观下线状态， 则master会被标记为客观下线 

* 在一般情况下， 每个sentinel会以每 10 秒一次的频率向它已知的所有master，slave发送 INFO 命令 

* 当master被sentinel标记为客观下线时，sentinel向下线的master的所有slave发送 INFO 命令的频率会从 10 秒一次改为 1 秒一次 
    * 若没有足够数量的sentinel同意master已经下线，master的客观下线状态就会被移除；
    * 若master重新向sentinel的 PING 命令返回有效回复，master的主观下线状态就会被移除

#### Sentinel模式搭建

- `sentinel.conf`配置文件
```
# vim /usr/local/redis/sentinel.conf

daemonize yes
logfile "/usr/local/redis/sentinel.log"
dir "/usr/local/redis/sentinel"                 #sentinel工作目录
sentinel monitor mymaster 192.168.30.128 6379 2      #判断master失效至少需要2个sentinel同意，建议设置为n/2+1，n为sentinel个数
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 30000       #判断master主观下线时间，默认30s

```

`注意`，sentinel auth-pass mymaster 123456需要配置在sentinel monitor mymaster 下面，否则启动报错


### Redis-cluster集群

- redis.conf文件中Redis集群配置参数
    - `cluster-enabled<yes/no>`：如果是，则在特定的Redis实例中启用Redis Cluster支持。否则，该实例将像往常一样作为独立实例启动。
    - `cluster-config-file<filename>`：请注意，尽管有此选项的名称，但它不是用户可编辑的配置文件，而是Redis Cluster节点每次发生更改时都会自动持久保存集群配置的文件（状态，基本上是状态），为了能够在启动时重新阅读它。该文件列出了诸如群集中其他节点之类的内容，它们的状态，持久变量等等。通常，由于收到某些消息，此文件将被重写并刷新到磁盘上。
    - `cluster-node-timeout<milliseconds>`：Redis群集节点不可用的最长时间（不将其视为失败）。如果主节点无法访问的时间超过指定的时间长度，则它的从节点将对其进行故障转移。此参数控制Redis Cluster中的其他重要事项。值得注意的是，在指定的时间内无法到达大多数主节点的每个节点都将停止接受查询。
    - `cluster-slave-validity-factor<factor>`：如果设置为零，则从服务器将始终尝试对主服务器进行故障转移，而不管主服务器和从服务器之间的链接保持断开状态的时间长短。如果该值为正，则将最大断开时间计算为节点超时值乘以此选项提供的系数，如果节点是从节点，则如果断开主链接的时间超过指定的时间，它将不会尝试启动故障转移。例如，如果节点超时设置为5秒，而有效性因子设置为10，则从服务器与主服务器断开连接超过50秒将不会尝试对其主服务器进行故障转移。请注意，如果没有从属能够对其进行故障转移，则任何非零的值都可能导致Redis群集在主服务器发生故障后不可用。在这种情况下，只有当原始主服务器重新加入集群后，集群才会返回可用状态。
    - `cluster-migration-barrier<count>`：一个主机将保持连接的最小数量的从机，以便另一个从机迁移到不再被任何从机覆盖的主机。有关更多信息，请参见本教程中有关副本迁移的相应部分。
    - `cluster-require-full-coverage<yes/no>`：如果设置为yes，默认情况下，如果某个节点未覆盖一定比例的密钥空间，集群将停止接受写入。如果该选项设置为no，即使仅可以处理有关密钥子集的请求，群集仍将提供查询。
    - `cluster-allow-reads-when-down<yes/no>`：如果将其设置为no（默认情况下为默认值），则当Redis群集被标记为失败时，或者当节点无法到达时，Redis群集中的节点将停止为所有流量提供服务达不到法定人数或完全覆盖。这样可以防止从不了解群集更改的节点读取可能不一致的数据。可以将此选项设置为yes，以允许在失败状态期间从节点进行读取，这对于希望优先考虑读取可用性但仍希望防止写入不一致的应用程序很有用。当仅使用一个或两个分片的Redis Cluster时，也可以使用它，因为它允许节点在主服务器发生故障但无法进行自动故障转移时继续为写入提供服务

#### 最小的Redis群集配置文件
```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

> 启用集群模式的只是cluster-enabled 指令。每个实例还包含该节点的配置存储位置的文件路径，默认情况下为nodes.conf。该文件永远不会被人类触及。它只是由Redis Cluster实例在启动时生成，并在需要时进行更新。

> 请注意，按预期工作的最小群集要求至少包含三个主节点。对于您的第一个测试，强烈建议启动一个包含三个主节点和三个从节点的六个节点群集。

> 为此，输入一个新目录，并创建以下目录，该目录以我们将在任何给定目录中运行的实例的端口号命名。

```
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```

启动每个实例
```
../redis-server ./redis.conf
```

从每个实例的日志中可以看到，由于不nodes.conf存在文件，因此每个节点都会为其分配一个新的ID。

```
[82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1
```

> 创建集群

- 如果使用的是Redis 5，这很容易实现，因为嵌入到中的Redis Cluster命令行实用程序redis-cli将为我们提供帮助，该实用程序可用于创建新集群，检查或重新分片现有集群等。

- 对于Redis版本3或4，有一个称为的旧工具redis-trib.rb，它非常相似。可以src在Redis源代码分发的目录中找到它。需要安装redisgem才能运行redis-trib。

Redis 5创建集群：
```
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1
```

Redis的4或3型：

```
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```
此处使用的命令是create，因为我们要创建一个新集群。该选项--cluster-replicas 1意味着我们要为每个创建的主机都希望有一个从机。其他参数是我要用于创建新集群的实例的地址列表。
