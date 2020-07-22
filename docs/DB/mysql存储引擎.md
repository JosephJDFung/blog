# mysql存储引擎

使用MySQL插件式存储引擎体系结构，允许数据库用户为特定的应用需求选择专门的存储引擎，完全不需要管理任何特殊的应用编码要求。采用MySQL服务器体系结构，由于在存储级别上提供了一致和简单的应用模型和API，应用程序编程人员和DBA可不再考虑所有的底层实施细节。因此，尽管不同的存储引擎具有不同的能力，应用程序是与之分离的。

![](https://img2018.cnblogs.com/i-beta/1397037/202002/1397037-20200220161412192-910498182.png)

>InnoDB 存储引擎

InnoDB 是事务型数据库的首选引擎，支持事务安全表 (ACID)，支持行锁定和外键。`MySQL5.5.5 之后，InnoDB 作为默认存储引擎`，InnoDB 主要特性有：

- InnoDB 给 MySQL 提供了具有提交、回滚和崩溃恢复能力的事务安全(ACID 兼容)存储引擎。InnoDB 锁定在行级并且也在 SELECT 语句中提供一个类似 Oracle 的非锁定读。这些功能增加了多用户部署和性能。在 SQL 查询中，可以自由地将 InnoDB 类型的表与其他 MySQL 的表的类型混合起来，甚至在同一个查询中也可以混合。
- InnoDB 是为处理巨大数据量的最大性能设计。它的 CPU 的效率可能是任何其它基于磁盘的关系数据库引擎所不能匹敌的。
- InnoDB 存储引擎完全与 MySQL 服务器整合，InnoDB 存储引擎为在主内存中缓存数据和索引而维持它自己的缓冲池。InnoDB 将它的表和索引存在一个逻辑表空间中，表空间可以包含数个文件（或原始磁盘分区）。这与 MyISAM 表不同，比如在 MyISAM 表中每个表被存在分离的文件中。InnoDB 表可以是任何尺寸，即使在文件尺寸被艰制为 2GB 的操作系统上。
- InnoDB 支持外键完整性约束（FOREIGN KEY)。 存储表中的数据时，每张表的存储都按主键顺序存放，如果没有显示在表定义时指定主键，InnoDB 会被每一行生成一个 6B 的 ROWID，并以此作为主健。
- InnoDB 被用在众多需要高性能的大型数据库站点上。 InnoDB 不创建目录，使用 InnoDB 时，MySQL 将在 MYSQL 数据目录下创建一个名为 ibdata1 的 10MB 大小的自动扩展数据文件，以及两个名为ib_logfile() 和 ib_logfile1 的 5MB 大小的日志文件。

- hash索引的创建由InnoDB存储引擎引擎自动优化创建，根据表的使用创建，程序员干预不了

>innodb逻辑存储结构

innodb的逻辑存储单元由大到小分别是 tablespace,segment,extent,page(block)组成

![](https://upload-images.jianshu.io/upload_images/6807865-f4eb8cca559045f3.png?imageMogr2/auto-orient/strip|imageView2/2/w/705/format/webp)

- 表空间(tablespace)

所有数据都是存放在表空间中的，启用了参数innodb_file_per_table，则每张表内的数据可以单独放到一个表空间中，每张表空间内存放的只是数据，索引和插入缓冲，其他类的数据，如undo信息，系统事务信息，二次写缓冲等还是存放在原来你的共享表空间。

- 段(segment)

常见的segment有数据段、索引段、回滚段。innodb是索引聚集表，所以数据就是索引，索引就是数据，那么数据段即是B+树的页节点(leaf node segment)，索引段即为B+树的非索引节点(non-leaf node segment)。而且段的管理是由引擎本身完成的。

- 区(extend)

　    区是由64个连续的页主成，每个页大小为16K，即每个区的大小为(64*16K)=1MB,对于大的数据段，mysql每次最多可以申请4个区，以此保证数据的顺序性能。

- 页(page)

页是innodb磁盘管理最小的单位，innodb每个页的大小是16K，且不可更改。常见的类型有：数据页 B-tree Node；undo页 Undo Log Page；系统页 System Page；事务数据页 Transaction system Page；插入缓冲位图页 Insert Buffer Bitmap；插入缓冲空闲列表页 Insert Buffer freeBitmap；未压缩的二进制大对象页Uncompressed BLOB Page；压缩的二进制大对象页 Compressed BLOB Page。

4.2.5、行

innodb存储引擎是面向行的(row-oriented),也就是说数据的存放按行进行存放。每个页最多可以存放16K/2～200行,也就是7992个行。

>MyISAM 存储引擎

MyISAM 基于 ISAM 的存储引擎，并对其进行扩展。它是在 Web、数据存储和其他应用环境下最常用的存储引擎之一。`MyISAM 拥有较高的插入、查询速度，但不支持事务。在 MySQL5.5.5 之前的版本中`，MyISAM 是默认存储引擎。MyISAM 主要特性有：

- 大文件（达 63 位文件长度）在支持大文件 系统和操作系统上被支持。
- 当把删除、更新及插入操作混合使用的时候，动态尺寸的行产生更少的碎片。这要通过合并相邻被删除的块，以及若下一个块被删除，就扩展到下一块来自动完成。
- 每个 MyISAM 表最大索引数是 64，这也可以通过编译来改变。对于键长度超过 250B 的情况，一个超过 1024B 的键将被用上。
- BLOB 和 TEXT 列可以被索引。
- NULL 值被允许在索引的列中。这个值占每个键的 0~1 个字节。
- 所有数字键值以高字节优先被存储以允许一个更高的索引压缩。
- 每表一个 AUTO_INCREMENT 列的内部处理。MyISAM 为 INSERT 和 UPDATE 操作自动更新这一列。使得 AUTO_INCREMENT 列更快（至少 10%）。在序列顶的值被删除除之后就不能再利用。
- 可以把数据文件和索引文件放在不同目录。
- 每个字符列可以有不同的字符集。
- 有 VARCHAR 的表可以固定或动态记录长度。
- VARCHAR 和 CHAR 列可以多达 64KB

使用 MyISAM 引擎创建数据库，将生产 3 个文件。文件的名字以表的名字开始，扩展名指出文件类型：`frm 文件存储表定义，数据文件的扩展名为 .MYD(MYData)，索引文件的扩展名是 .MYI（MYIndex)。`

>MEMORY 存储引擎

MEMORY 存储引擎将表中的数据存储到内存中，为查询和引用其他表数据提供快速访问。MEMORY 主要特性有：

- MEMORY 表的每个表可以多达 32 个索引，每个索引 16 列，以及 500B 的最大键长度。
- MEMORY 存储引擎引擎执行 HASH 和 BTREE 索引。
- 可以在一个 MEMORY 表中有非唯一键。
- MEMORY 不支持 BLOB 或 TEXT 列。
- MEMORY 表使用一个固定的记录长度格式。
- MEMORY 支持 AUTO_INCREMENT 列和对包含 NULL 值的列索引。
- MEMORY 表内容被存在内存中，内存是 MEMORY 表和服务器在查询处理时的空闲中创建的内部表共享。
- MEMORY 表在所有客户端之间共享（就像其他任何非 TEMPORARY 表）。
- 当不再需要 MEMORY 表的内容时，要释放被 MEMORY 表使用的内存，应该执行 DELETE FROM 或 TRUNCATE TABLE，或者删除整个表（使用 DROP TABLE)。

功能|MyISAM|Memory|InnoDB
-|-|-|-
存储限制|265TB|RAM|65TB
支持事务|No|No|Yes
支持全文索引|Yes|No|No
支持树索引|Yes|Yes|Yes
支持哈希索引|No|Yes|No
支持数据缓存|No|N/A|Yes
支持外键|No|No|Yes