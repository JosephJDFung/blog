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

- master节点
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


