# mysql基础

MySQL 是最流行的关系型数据库管理系统，在 WEB 应用方面 MySQL 是最好的 RDBMS(Relational Database Management System：关系数据库管理系统)应用软件之一。


## MySQL组成模块划分

![](https://pic1.zhimg.com/80/v2-6d8706f922a62d83a18d06859b1367b3_720w.png)

>MySQL向外提供的交互接口（Connectors）

Connectors组件，是MySQL向外提供的交互组件，如java,.net,php等语言可以通过该组件来操作SQL语句，实现与SQL的交互

>管理服务组件和工具组件(Management Service & Utilities)

提供对MySQL的集成管理，如备份(Backup),恢复(Recovery),安全管理(Security)等

>连接池组件(Connection Pool)

负责监听对客户端向MySQL Server端的各种请求，接收请求，转发请求到目标模块。每个成功连接MySQL Server的客户请求都会被创建或分配一个线程，该线程负责客户端与MySQL Server端的通信，接收客户端发送的命令，传递服务端的结果信息等

>SQL接口组件(SQL Interface)

接收用户SQL命令，如DML,DDL和存储过程等，并将最终结果返回给用户

>查询分析器组件(Parser)

SQL命令传递到解析器的时候会被解析器验证和解析，首先分析SQL命令语法的合法性，并尝试将SQL命令分解成数据结构，若分解失败，则提示SQL语句不合理

>优化器组件（Optimizer）

SQL语句在查询之前会使用查询优化器对查询进行优化。

>缓存组件（Caches & Buffers）

如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。这个缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，key缓存，权限缓存等

>存储引擎（Engine）

MySQL属于关系型数据库，而关系型数据库的存储是以表的形式进行的，对于表的创建，数据的存储，检索，更新等都是由MySQL存储引擎完成的，这也是MySQL存储引擎在MySQL中扮演的重要角色。

MySQL之所以有多种存储引擎，是因为MySQL的开源性决定的。MySQL存储引擎，从种类上来说，大致可归结为官方存储引擎和第三方存储引起。MySQL的开源性，允许第三方基于MySQL骨架，开发适合自己业务需求的存储引擎。

MySQL存储引擎作用
MySQL存储引擎在MySQL中扮演重要角色，其作比较重要作用，大致归结为如下两方面：

- 作用一：管理表创建，数据检索，索引创建等

- 作用二：满足自定义存储引擎开发。

`MySQL引擎种类`

不同种类的存储引擎，在存储表时的存储引擎表机制也有所不同，从MySQL存储引擎种类上来说，可以分为官方存储引擎和第三方存储引擎。

当前，也存在多种MySQL存储引擎，如MyISAM存储引擎，InnoDB存储引擎，NDB存储引擎，Archive存储引擎，Federated存储引擎，Memory存储引擎，Merge存储引擎，Parter存储引擎，Community存储引擎，Custom存储引擎和其他存储引擎。

其中，比较常用的存储引擎包括InnoDB存储引擎，MyISAM存储引擎和Momery存储引擎。

>物理文件（File System）

实际存储MySQL 数据库文件和一些日志文件等的系统，如Linux，Unix,Windows等。

物理文件包括：redolog、undolog、binlog（复制时还有relaylog）、errorlog、querylog、slowlog、data、index

---

## Mysql安装常用

验证 MySQL 安装

```
[root@host]# mysqladmin --version
```

Mysql安装成功后，默认的root用户密码为空，你可以使用以下命令来创建root用户的密码：
```
[root@host]# mysqladmin -u root password "new_password";

[root@host]# mysql -u root -p
Enter password:*******
```

>MySQL 管理

- MySQL 用户设置

如果你需要添加 MySQL 用户，你只需要在 mysql 数据库中的 user 表添加新用户即可。

以下为添加用户的的实例，用户名为guest，密码为guest123，并授权用户可进行 SELECT, INSERT 和 UPDATE操作权限：

```
root@host# mysql -u root -p
Enter password:*******
mysql> use mysql;
Database changed

mysql> INSERT INTO user 
          (host, user, password, 
           select_priv, insert_priv, update_priv) 
           VALUES ('localhost', 'guest', 
           PASSWORD('guest123'), 'Y', 'Y', 'Y');
Query OK, 1 row affected (0.20 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 1 row affected (0.01 sec)

mysql> SELECT host, user, password FROM user WHERE user = 'guest';
+-----------+---------+------------------+
| host      | user    | password         |
+-----------+---------+------------------+
| localhost | guest | 6f8c114b58f2ce9e |
+-----------+---------+------------------+
1 row in set (0.00 sec)
```

`注意`：

- 在 MySQL5.7 中 user 表的 password 已换成了authentication_string。

- password() 加密函数已经在 8.0.11 中移除了，可以使用 MD5() 函数代替。

- 在注意需要执行 FLUSH PRIVILEGES 语句。 这个命令执行后会重新载入授权表。

在创建用户时，为用户指定权限，在对应的权限列中，在插入语句中设置为 'Y' 即可，用户权限列表如下：

- Select_priv
- Insert_priv
- Update_priv
- Delete_priv
- Create_priv
- Drop_priv
- Reload_priv
- Shutdown_priv
- Process_priv
- File_priv
- Grant_priv
- References_priv
- Index_priv
- Alter_priv

另外一种添加用户的方法为通过SQL的 GRANT 命令，以下命令会给指定数据库TUTORIALS添加用户 zara ，密码为 zara123 。

```
root@host# mysql -u root -p
Enter password:*******
mysql> use mysql;
Database changed

mysql> GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP
    -> ON TUTORIALS.*
    -> TO 'zara'@'localhost'
    -> IDENTIFIED BY 'zara123';
```

>MySQL 8.0.11 版本之后创建用户方法如下：

`CREATE USER 'laowang'@'localhost' IDENTIFIED BY '123456';`

授予账户权限的方法如下：

`GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER ON *.* TO 'laowang'@'localhost';`

授予所有权限：

`GRANT ALL PRIVILEGES ON *.* TO 'laowang'@'localhost'；`

查看用户权限：

`show grants for 'laowang'@'localhost';`

>查看 MySQL 用户权限

查看当前用户（自己）权限：

`show grants;`

查看其他 MySQL 用户权限：

`show grants for dba@localhost;`

## MySQL 数据类型

>数值类型

类型|大小|范围（有符号）|范围（无符号）|用途
-|-|-|-|-
TINYINT|1 byte|(-128，127)|(0，255)|小整数值
SMALLINT|2 bytes|(-32 768，32 767)|(0，65 535)|大整数值
MEDIUMINT|3 bytes|(-8 388 608，8 388 607)|(0，16 777 215)|大整数值
INT或INTEGER|4 bytes|(-2 147 483 648，2 147 483 647)|(0，4 294 967 295)|大整数值
BIGINT|8 bytes|(-9,223,372,036,854,775,808，9 223 372 036 854 775 807)|(0，18 446 744 073 709 551 615)|极大整数值
FLOAT|4 bytes|(-3.402 823 466 E+38，-1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38)|0，(1.175 494 351 E-38，3.402 823 466 E+38)|单精度,浮点数值
DOUBLE|8 bytes|(-1.797 693 134 862 315 7 E+308，-2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)|0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)|双精度,浮点数值
DECIMAL|对DECIMAL(M,D) ，如果M>D，为M+2否则为D+2|依赖于M和D的值|依赖于M和D的值|小数值


>日期和时间类型

类型|大小
( bytes)|范围|格式|用途
-|-|-|-|-
DATE|3|1000-01-01/9999-12-31|YYYY-MM-DD|日期值
TIME|3|'-838:59:59'/'838:59:59'|HH:MM:SS|时间值或持续时间
YEAR|1|1901/2155|YYYY|年份值
DATETIME|8|1000-01-01 00:00:00/9999-12-31 23:59:59|YYYY-MM-DD HH:MM:SS|混合日期和时间值
TIMESTAMP|4|1970-01-01 00:00:00/2038,结束时间是第 2147483647 秒，北京时间 2038-1-19 11:14:07，格林尼治时间 2038年1月19日 凌晨 03:14:07,YYYYMMDD HHMMSS|混合日期和时间值，时间戳

>字符串类型

类型|大小|用途
-|-|-
CHAR|0-255 bytes|定长字符串
VARCHAR|0-65535 bytes|变长字符串
TINYBLOB|0-255 bytes|不超过 255 个字符的二进制字符串
TINYTEXT|0-255 bytes|短文本字符串
BLOB|0-65 535 bytes|二进制形式的长文本数据
TEXT|0-65 535 bytes|长文本数据
MEDIUMBLOB|0-16 777 215 bytes|二进制形式的中等长度文本数据
MEDIUMTEXT|0-16 777 215 bytes|中等长度文本数据
LONGBLOB|0-4 294 967 295 bytes|二进制形式的极大文本数据
LONGTEXT|0-4 294 967 295 bytes|极大文本数据

>MySQL NULL 值处理

MySQL 使用 SQL SELECT 命令及 WHERE 子句来读取数据表中的数据,但是当提供的查询条件字段为 NULL 时，该命令可能就无法正常工作。

为了处理这种情况，MySQL提供了三大运算符:

- IS NULL: 当列的值是 NULL,此运算符返回 true。
- IS NOT NULL: 当列的值不为 NULL, 运算符返回 true。
- <=>: 比较操作符（不同于 = 运算符），当比较的的两个值相等或者都为 NULL 时返回 true。

关于 NULL 的条件比较运算是比较特殊的。你不能使用 = NULL 或 != NULL 在列中查找 NULL 值 。

在 MySQL 中，NULL 值与任何其它值的比较（即使是 NULL）永远返回 NULL，即 NULL = NULL 返回 NULL 。

`MySQL 中处理 NULL 使用 IS NULL 和 IS NOT NULL 运算符。`

## MySQL 正则表达式

模式|描述
-|-
^|匹配输入字符串的开始位置。如果设置了 RegExp 对象的 Multiline 属性，^ 也匹配 '\n' 或 '\r' 之后的位置。
$|匹配输入字符串的结束位置。如果设置了RegExp 对象的 Multiline 属性，$ 也匹配 '\n' 或 '\r' 之前的位置。
.|匹配除 "\n" 之外的任何单个字符。要匹配包括 '\n' 在内的任何字符，请使用像 '[.\n]' 的模式。
[...]|字符集合。匹配所包含的任意一个字符。例如， '[abc]' 可以匹配 "plain" 中的 'a'。
[^...]|负值字符集合。匹配未包含的任意字符。例如， '[^abc]' 可以匹配 "plain" 中的'p'。
p1|p2|p3|匹配 p1 或 p2 或 p3。例如，'z|food' 能匹配 "z" 或 "food"。'(z|f)ood' 则匹配 "zood" 或 "food"。
*|匹配前面的子表达式零次或多次。例如，zo* 能匹配 "z" 以及 "zoo"。* 等价于{0,}。
+|匹配前面的子表达式一次或多次。例如，'zo+' 能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。+ 等价于 {1,}。
{n}|n 是一个非负整数。匹配确定的 n 次。例如，'o{2}' 不能匹配 "Bob" 中的 'o'，但是能匹配 "food" 中的两个 o。
{n,m}|m 和 n 均为非负整数，其中n <= m。最少匹配 n 次且最多匹配 m 次。

查找name字段中以元音字符开头或以'ok'字符串结尾的所有数据：

`mysql> SELECT name FROM person_tbl WHERE name REGEXP '^[aeiou]|ok$';`

## MySQL 导出数据

>使用 SELECT ... INTO OUTFILE 语句导出数据

SELECT ... INTO OUTFILE 语句有以下属性:

- LOAD DATA INFILE是SELECT ... INTO OUTFILE的逆操作，SELECT句法。为了将一个数据库的数据写入一个文件，使用SELECT ... INTO OUTFILE，为了将文件读回数据库，使用LOAD DATA INFILE。
- SELECT...INTO OUTFILE 'file_name'形式的SELECT可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有FILE权限，才能使用此语法。
- 输出不能是一个已存在的文件。防止文件数据被篡改。
- 你需要有一个登陆服务器的账号来检索文件。否则 SELECT ... INTO OUTFILE 不会起任何作用。
- 在UNIX中，该文件被创建后是可读的，权限由MySQL服务器所拥有。这意味着，虽然你就可以读取该文件，但可能无法将其删除。

>导出表作为原始数据

mysqldump 是 mysql 用于转存储数据库的实用程序。它主要产生一个 SQL 脚本，其中包含从头重新创建数据库所必需的命令 CREATE TABLE INSERT 等。

使用 mysqldump 导出数据需要使用 --tab 选项来指定导出文件指定的目录，该目标必须是可写的。

>将数据表及数据库拷贝至其他主机

如果你需要将数据拷贝至其他的 MySQL 服务器上, 你可以在 mysqldump 命令中指定数据库名及数据表。

[菜鸟教程MySQL 导出数据](https://www.runoob.com/mysql/mysql-database-export.html)

## MySQL 导入数据

>mysql 命令导入

```
mysql -u用户名    -p密码    <  要导入的数据库数据(abc.sql)
```

>source 命令导入

```
mysql> create database abc;      # 创建数据库
mysql> use abc;                  # 使用已创建的数据库 
mysql> set names utf8;           # 设置编码
mysql> source /home/abc/abc.sql  # 导入备份数据库
```

>使用 LOAD DATA 导入数据

MySQL 中提供了LOAD DATA INFILE语句来插入数据。 以下实例中将从当前目录中读取文件 dump.txt ，将该文件中的数据插入到当前数据库的 mytbl 表中。
```
mysql> LOAD DATA LOCAL INFILE 'dump.txt' INTO TABLE mytbl;
```

>使用 mysqlimport 导入数据

mysqlimport 客户端提供了 LOAD DATA INFILEQL 语句的一个命令行接口。mysqlimport 的大多数选项直接对应 LOAD DATA INFILE 子句。

从文件 dump.txt 中将数据导入到 mytbl 数据表中, 可以使用以下命令：

```
$ mysqlimport -u root -p --local mytbl dump.txt
password *****
```

mysqlimport 命令可以指定选项来设置指定格式,命令语句格式如下：

```
$ mysqlimport -u root -p --local --fields-terminated-by=":" \
   --lines-terminated-by="\r\n"  mytbl dump.txt
password *****

```

mysqlimport 语句中使用 --columns 选项来设置列的顺序：

```
$ mysqlimport -u root -p --local --columns=b,c,a \
    mytbl dump.txt
password *****
```

>mysqlimport的常用选项

选项|功能
-|-
-d or --delete|新数据导入数据表中之前删除数据数据表中的所有信息
-f or --force|不管是否遇到错误，mysqlimport将强制继续插入数据
-i or --ignore|mysqlimport跳过或者忽略那些有相同唯一 关键字的行， 导入文件中的数据将被忽略。
-l or -lock-tables|数据被插入之前锁住表，这样就防止了， 你在更新数据库时，用户的查询和更新受到影响。
-r or -replace|这个选项与－i选项的作用相反；此选项将替代 表中有相同唯一关键字的记录。
--fields-enclosed- by= char|指定文本文件中数据的记录时以什么括起的， 很多情况下 数据以双引号括起。 默认的情况下数据是没有被字符括起的。
--fields-terminated- by=char|指定各个数据的值之间的分隔符，在句号分隔的文件中， 分隔符是句号。您可以用此选项指定数据之间的分隔符。 默认的分隔符是跳格符（Tab）
--lines-terminated- by=str|此选项指定文本文件中行与行之间数据的分隔字符串 或者字符。 默认的情况下mysqlimport以newline为行分隔符。 您可以选择用一个字符串来替代一个单个的字符： 一个新行或者一个回车。

## [MySQL 内置函数](https://www.runoob.com/mysql/mysql-functions.html)

- 字符串函数
- 数字函数
- 日期函数
- 高级函数(查看mysql运行信息)

## 配置文件my.ini / my.cnf

```yml
[mysqld]
# 设置3306端口
port=3306
# 设置mysql的安装目录
basedir=D:\mysql-8.0.12-winx64\mysql-8.0.12-winx64\
# 设置mysql数据库的数据的存放目录
datadir=D:\database\MySQL\Data
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 默认使用“mysql_native_password”插件认证
default_authentication_plugin=mysql_native_password

[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8
[client]
# 设置mysql客户端连接服务端时默认使用的端口
port=3306
default-character-set=utf8
```

>配置文件的内容查找

```shell
# 查看并发数
mysql> show status like 'Threads%';
# 查看mysql服务器的最大连接数值
mysql> show variables like '%max_connections%';
# 查看mysql服务器响应的最大连接数:
mysql> show global status like 'Max_used_connections';
```

>配置文件my.cnf的内容修改

```shell

# 设置mysql服务器的最大连接数值
mysql> set GLOBAL max_connections=256
```

>myisam相关配置

```yml
#一般用来缓存MyISAM表的主键，也用于临时的磁盘表缓存主键，上面多次出现临时磁盘表，所以就算不用MyISAM也最好为其设置一个不小的值，默认32M
key_buffer_size = 64M
#全表扫描MyISAM表时的缓存，每个线程拥有下行的大小。
read_buffer_size = 2M
#排序操作时与磁盘之间的缓存，分到每个线程，默认16M
read_rnd_buffer_size = 24M
#MyISAM使用特殊树形进行批量插入时的缓存，如insert ... values(..)(..)(..)
bulk_insert_buffer_size = 64M
#MyISAM索引文件的最大限定，
myisam_max_sort_file_size = 10G
#如果一个myisam表有一个以上的索引， MyISAM可以使用一个以上线程来排序并行它们。较耗硬件资源，如果你的环境不错，可以增加此值。
myisam_repair_threads = 2
#自动检查和修复无法正确关闭MyISAM表。
myisam_recover
```

>innodb相关配置

```yml
#开启下条将会禁用 INNODB
#skip-innodb
#一般不用设置或者说设了也没多大用，InnoDB会自己与操作系统交互管理其附加内存池所使用InnoDB的存储数据的大小
innodb_additional_mem_pool_size = 16M
#innodb整体缓冲池大小，不宜过大，设为本地内存的 50%-75% 比较合适,在本机开发过程中可以设得较小一点如 64M,256M
innodb_buffer_pool_size = 256M
#InnoDB的数据存储在一个或多个数据文件组成的表空间
innodb_data_file_path = ibdata1:10M:autoextend
#用于异步IO操作的线程数量，默认为 4 ，可适当提高
innodb_file_io_threads = 8
#线程数内允许的InnoDB内核,不宜太高
innodb_thread_concurrency = 16
#InnoDB的事务日志快存行为,默认为 1，为0可减轻磁盘I/0操作，还有以为2
innodb_flush_log_at_trx_commit = 1
#InnoDB的用于的缓冲日志数据的大小
innodb_log_buffer_size = 8M
#日志文件，可设置为25%-90%的总体缓存大小，默认 256M. 修改此项要先删除datadir\ib_logfileXXX
innodb_log_file_size = 256M
#日志组数量，默认为3
innodb_log_files_in_group = 3
#InnoDB的日志文件位置。默认是MySQL的datadir
#innodb_log_group_home_dir
#InnoDB最大允许的脏页缓冲池的百分比，默认90
innodb_max_dirty_pages_pct = 80
#事务死锁超时设定
innodb_lock_wait_timeout = 120
```