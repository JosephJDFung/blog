# mysql 日志与备份

## binlog日志

Binlog日志，即binary log，是二进制日志文件，有两个作用，一个是增量备份，另一个是主从复制，即主节点维护一个binlog日志文件，从节点从binlog中同步数据，也可以通过binlog日志来恢复数据

Binlog日志包括两类文件；

- 第一个是二进制索引文件(后缀名为.index)
- 第二个为日志文件(后缀名为.00000*),记录数据库所有的DDL和DML(除了查询语句select)语句事件

>日志恢复数据库

1. 要想通过日志恢复数据库，在你的my.cnf文件里应该有如下的定义，log-bin=mysql-bin，这个是必须的.binlog-do-db=db_test,这个是指定哪些数据库需要日志，如果有多个数据库就每行一个，如果不指定的话默认就是所有数据库

```
[mysqld] 
log-bin=mysql-bin 
binlog-do-db=db_test 
binlog-do-db=db_test2 
```

2. 删除二进制日志:

```sql
a. mysql> system ls -ltr /var/lib/mysql/bintest*; 
mysql>reset master(清空所有的二进制日志文件) 
b. purge master logs to 'bintest.000006';(删除bintest.000006之前的二进制日志文件) 
c. purge master logs before '2007-08-10 04:07:00'(删除该日期之前的日志) 
d. 在my.cnf 配置文件中[mysqld]中添加: 
expire_logs_day=3设置日志的过期天数,过了指定的天数,会自动删除 
```

3. 恢复操作


1. 指定恢复时间

对于MySQL
4.1.4，可以在mysqlbinlog语句中通过–start-date和–stop-date选项指定DATETIME格式的起止时间。举例说明，假设在今天上午10:00(今天是2005年4月20日)，执行SQL语句来删除一个大表。要想恢复表和数据，你可以恢复前晚上的备份，并输入：

mysqlbinlog --stop-date="2005-04-20 9:59:59"
/var/log/mysql/mysql-bin.000001 | mysql -u root -pmypwd

该命令将恢复截止到在–stop-date选项中以DATETIME格式给出的日期和时间的所有数据。如果你没有检测到几个小时后输入的错误的SQL语句，可能你想要恢复后面发生的活动。根据这些，你可以用起使日期和时间再次运行mysqlbinlog：
mysqlbinlog --start-date="2005-04-20 10:01:00"
/var/log/mysql/mysql-bin.000001 | mysql -u root -pmypwd
在该行中，从上午10:01登录的SQL语句将运行。组合执行前夜的转储文件和mysqlbinlog的两行可以将所有数据恢复到上午10:00前一秒钟。你应检查日志以确保时间确切。下一节介绍如何实现。

2. 指定恢复位置

也可以不指定日期和时间，而使用mysqlbinlog的选项–start-position和–stop-position来指定日志位置。它们的作用与起止日选项相同，不同的是给出了从日志起的位置号。使用日志位置是更准确的恢复方法，特别是当由于破坏性SQL语句同时发生许多事务的时候。要想确定位置号，可以运行mysqlbinlog寻找执行了不期望的事务的时间范围，但应将结果重新指向文本文件以便进行检查。操作方法为：
```sql
mysqlbinlog --start-date="2005-04-20 9:55:00" --stop-date="2005-04-20
10:05:00" /var/log/mysql/mysql-bin.000001 > /tmp/mysql_restore.sql

```
该命令将在/tmp目录创建小的文本文件，将显示执行了错误的SQL语句时的SQL语句。你可以用文本编辑器打开该文件，寻找你不要想重复的语句。如果二进制日志中的位置号用于停止和继续恢复操作，应进行注释。用log_pos加一个数字来标记位置。使用位置号恢复了以前的备份文件后，你应从命令行输入下面内容：
```sql
mysqlbinlog --stop-position="368312" /var/log/mysql/mysql-bin.000001 |
mysql -u root -pmypwd mysqlbinlog --start-position="368315"
/var/log/mysql/mysql-bin.000001 | mysql -u root -pmypwd

```
上面的第1行将恢复到停止位置为止的所有事务。下一行将恢复从给定的起始位置直到二进制日志结束的所有事务。因为mysqlbinlog的输出包括每个SQL语句记录之前的SET
TIMESTAMP语句，恢复的数据和相关MySQL日志将反应事务执行的原时间。



```sql
mysqlbinlog --stop-date="2005-04-20 9:59:59" /var/log/mysql/mysql-bin.000001 | mysql -u root -pmypwd 

# 类似的语句，但是它一次只能操作一个日志文件，如果你变通一下变成这样

mysqlbinlog --stop-date="2005-04-20 9:59:59" /var/log/mysql/mysql-bin.0* | mysql -u root -pmypwd 


mysqlbinlog --stop-date="2005-04-20 9:59:59" /var/log/mysql/mysql-bin.[0-9]* | mysql -u root -pmypwd ，
# [0-9]*表示以数字开头的任何字符
```

>你可以通过–one-database 参数选择性的恢复单个数据库

```sql
mysqlbinlog --stop-date="2005-04-20 9:59:59" /var/log/mysql/mysql-bin.000001 | mysql -u root -pmypwd --one-database db_test
```

## mysql的冷热备份

在主流的关系数据库里，备份方式按照备份时数据库的状态还有备份的方式可以化分为热备 冷备 逻辑备份 物理备份

- 逻辑备份：一定是热备（也叫半热备，因为会锁库）的，也就是数据库灾备份时是开着的，使用mysqldump的命令

- 物理备份：分为热备和冷备，这两个其实本质上没有差别，都是通过对数据文件的拷贝实现
    - 物—热：采用xtrabackup这个第三方的软件实现
    - 物-冷：关闭数据库，然后使用cp命令


>逻辑备份

`mysqldump -uroot -predhat-S /var/tmp/mysql.sock --all-databases> /var/tmp/all.sql`

恢复

`mysql -uroot -predhat < /var/tmp/all.sql`

备份多个库

```sql

mysqldump -uroot -predhat-S /var/tmp/mysql.sock --databases db01 db02 db03 > /var/tmp/db01-03.sql

mysql -uroot -predhat </var/tmp/db01-03.sql

mysqldump --databases discuz >/var/tmp/discuz.sql

mysqldump --databases ucenter >/var/tmp/ucenter.sql

```

备份多张表
```sql
mysqldump -uroot -predhat -S/var/tmp/mysql.sock db01 t1 t2 t3 >/var/tmp/db01.t1-2.sql

mysql -uroot -predhat db01 </var/tmp/db01.t1-2.sql
```
–完全备份(mysqldump)+增量备份(binlog)

- 数据总量不大，一般在几百M的数据可以使用这种方法

- 如果数据量大太，每次备份锁表的时间会比较长，这样就可能影响上层应用正常使用
