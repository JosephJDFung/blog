# mysql优化

## 慢查询

>参数说明

- slow_query_log 慢查询开启状态
- slow_query_log_file 慢查询日志存放的位置（这个目录需要MySQL的运行帐号的可写权限，一般设置为MySQL的数据存放目录）
- long_query_time 查询超过多少秒才记录

```shell
mysql> show variables like 'slow_query%';
+---------------------------+----------------------------------+
| Variable_name             | Value                            |
+---------------------------+----------------------------------+
| slow_query_log            | OFF                              |
| slow_query_log_file       | /mysql/data/localhost-slow.log   |
+---------------------------+----------------------------------+

mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
```

>设置方法

方法一：全局变量设置
将 slow_query_log 全局变量设置为“ON”状态

`mysql> set global slow_query_log='ON'; `

设置慢查询日志存放的位置

`mysql> set global slow_query_log_file='/usr/local/mysql/data/slow.log';`

查询超过1秒就记录

`mysql> set global long_query_time=1;`


方法二：配置文件设置

修改配置文件my.cnf，在[mysqld]下的下方加入

```
[mysqld]
slow_query_log = ON
slow_query_log_file = /usr/local/mysql/data/slow.log
long_query_time = 1
```

## explain关键字

分析 SQL 执行效率是优化 SQL 的重要手段，通过上面讲的两种方法，定位到慢查询语句后，我们就要开始分析 SQL 执行效率了，子曾经曰过：“工欲善其事，必先利其器”，我们可以通过 explain、show profile 和 trace 等诊断工具来分析慢查询。

>Explain字段详解

![](https://pics7.baidu.com/feed/8694a4c27d1ed21b07e77eea5eb5afc150da3ff7.png?token=6c4930d7a741abe9640293af70b47a5d&s=5EAA3463119FC1CE4AD5E1DB0300C0B1)

- table 显示这一行的数据是关于哪张表的
- type 这是重要的列，显示连接使用了何种类型。从最好到最差的连接类型为const、eq_reg、ref、range、indexhe和ALL 
- rows 显示需要扫描行数
- key 使用的索引

![](https://pics3.baidu.com/feed/3801213fb80e7bec1016e0e9dcf5cb3d99506bcf.png?token=958579a2316a0db049a186a8adb2e035&s=5AA8346319D9C4CA4ED5F1CF0300C0B1)

![](https://pics1.baidu.com/feed/902397dda144ad344d105e002b797ef133ad85ef.png?token=4b9989dc6fa7bd9d3aecf2811de754bc&s=1EAA74231DDEC4CA1C7D81DE0300C0B1)

![](https://pics3.baidu.com/feed/77c6a7efce1b9d16188894500f05c68a8d546491.png?token=377602db9ad4b28f7ebc09d452309567&s=3AAA742319BEE4CE4AF441DA0300C0B1)

## show profile

show profile 命令用于跟踪执行过的sql语句的资源消耗信息，可以帮助查看sql语句的执行情况，可以在做性能分析或者问题诊断的时候作为参考。

>确定是否支持 profile

`mysql> select @@have_profiling;`

>查看 profiling 状态

`mysql> select @@profiling;`
0表示关闭，1表示开启，默认是关闭的。

使用如下命令开启：`mysql> set profiling=1;`

>show profiles 

显示最近发送到服务器上执行的语句的资源使用情况.显示的记录数由变量:profiling_history_size 控制,默认15条。

```sql
mysql> show profiles;
+----------+------------+-------------------------------+
| Query_ID | Duration   | Query                         |
+----------+------------+-------------------------------+
|        1 | 0.00371475 | show variables like '%profi%' |
|        2 | 0.00005700 | show prifiles                 |
|        3 | 0.00011775 | SELECT DATABASE()             |
|        4 | 0.00034875 | select * from student         |
+----------+------------+-------------------------------+
```

>根据profile id查询指定SQL执行详情

通过 show profile for query id 可看到执行过的 SQL 每个状态和消耗时间：

```sql 
mysql> SHOW PROFILE;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| checking permissions | 0.000040 |
| creating table       | 0.000056 |
| After create         | 0.011363 |
| query end            | 0.000375 |
| freeing items        | 0.000089 |
| logging slow query   | 0.000019 |
| cleaning up          | 0.000005 |
+----------------------+----------+
7 rows in set (0.00 sec)

mysql> SHOW PROFILE FOR QUERY 1;
+--------------------+----------+
| Status             | Duration |
+--------------------+----------+
| query end          | 0.000107 |
| freeing items      | 0.000008 |
| logging slow query | 0.000015 |
| cleaning up        | 0.000006 |
+--------------------+----------+
4 rows in set (0.00 sec)

mysql> SHOW PROFILE CPU FOR QUERY 2;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| checking permissions | 0.000040 | 0.000038 |   0.000002 |
| creating table       | 0.000056 | 0.000028 |   0.000028 |
| After create         | 0.011363 | 0.000217 |   0.001571 |
| query end            | 0.000375 | 0.000013 |   0.000028 |
| freeing items        | 0.000089 | 0.000010 |   0.000014 |
| logging slow query   | 0.000019 | 0.000009 |   0.000010 |
| cleaning up          | 0.000005 | 0.000003 |   0.000002 |
+----------------------+----------+----------+------------+
7 rows in set (0.00 sec)

```

`注意`：“show profiles”已弃用，SHOW PROFILES将来会被Performance Schema替换掉，但是现在还是非常非常实用的，8.0目前还在支持中。

- 开启该功能，会对 MySQL 性能有所影响，因此只建议分析问题时临时开启。


## trace分析优化器

MySQL5.6提供了对SQL的跟踪trace，通过trace文件能够进一步了解为什么优化器选择A执行计划而不是选择B执行计划，帮助我们更好地理解优化器行为。

使用方式：首先打开trace，设置格式为JSON，设置trace最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能够完整显示。

```sql

mysql> SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
Query OK, 0 rows affected (0.01 sec)
 
mysql> SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
Query OK, 0 rows affected (0.02 sec)
```
>optimizer_trace变量

1. 通过SHOW VARIABLES LIKE 'optimizer_trace'，可以看到变量的Value是enabled=off,one_line=off，即默认是禁止的。
2. enabled代表功能是否开启，one_line的值是控制输出格式的，如果为on那么所有输出都将在一行中展示。
3. 通过语句SET optimizer_trace="enabled=on";来开启
4. OPTIMIZER_TRACE是表（表和变量，名字一样但是大小不一样），其是在information_schema的数据库里面

>OPTIMIZER_TRACE表
1. 包含四列具体如下：
2. QUERY：表示我们的查询语句。
3. TRACE：表示优化过程的JSON格式文本
4. MISSING_BYTES_BEYOND_MAX_MEM_SIZE：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
5. INSUFFICIENT_PRIVILEGES：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是1，我们暂时不关心这个字段的值。

>otpimzer trace功能的作用和优化的大致阶段

1. 这个功能可以让我们方便的查看优化器生成执行计划的整个过程
2. prepare阶段
3. optimize阶段
4. execute阶段
5. 基于成本的优化主要集中在optimize阶段
6. 单表查询来说，我们主要关注optimize阶段的"rows_estimation"这个过程，这个过程深入分析了对单表查询的各种执行方案的成本
7. 对于多表连接查询来说，我们更多需要关注"considered_execution_plans"这个过程，这个过程里会写明各种不同的连接方式所对应的成本


>具体跟踪的json格式的信息：

```sql
select * from test1,test2 where test1.id=test2.id and test1.id>4999900 | {
  "steps": [
    {
      "join_preparation": {           --连接准备
        "select#": 1,                 --join准备的第一步
        "steps": [                    --解析成编号，解析数据库和表
          {
            "expanded_query": "/* select#1 */ select `test1`.`id` AS `id`,`test1`.`k` AS `k`,`test1`.`c` AS `c`,`test1`.`pad` AS `pad`,`test2`.`id` AS `id`,`test2`.`k` AS `k`,`test2`.`c` AS `c`,`test2`.`pad` AS `pad` from `test1` join `test2` where ((`test1`.`id` = `test2`.`id`) and (`test1`.`id` > 4999900))"
          }
        ]
      }
    },
    {
      "join_optimization": {         --join优化
        "select#": 1,
        "steps": [
          {
            "condition_processing": {       --where条件
              "condition": "WHERE",
              "original_condition": "((`test1`.`id` = `test2`.`id`) and (`test1`.`id` > 4999900))",
              "steps": [       --优化的步骤
                {
                  "transformation": "equality_propagation",         --等值优化
                  "resulting_condition": "((`test1`.`id` > 4999900) and multiple equal(`test1`.`id`, `test2`.`id`))"   --把test.id>4999900放到前面，test1.id=test2.id使用多等值连接
                },
                {
                  "transformation": "constant_propagation",        --常量优化
                  "resulting_condition": "((`test1`.`id` > 4999900) and multiple equal(`test1`.`id`, `test2`.`id`))"
                },
                {
                  "transformation": "trivial_condition_removal",   --琐碎的条件排除
                  "resulting_condition": "((`test1`.`id` > 4999900) and multiple equal(`test1`.`id`, `test2`.`id`))"
                }
              ]
            }
          },
          {
            "table_dependencies": [     --表依赖
              {
                "table": "`test1`",        --表名
                "row_may_be_null": false,  --是否有null值，flase是没有
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              },
              {
                "table": "`test2`",
                "row_may_be_null": false,
                "map_bit": 1,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [    --相关优化索引使用
              {
                "table": "`test1`",  
                "field": "id",             --索引字段
                "equals": "`test2`.`id`",  --连接的等值字段
                "null_rejecting": false
              },
              {
                "table": "`test2`",
                "field": "id",
                "equals": "`test1`.`id`",
                "null_rejecting": false
              }
            ]
          },
          {
            "rows_estimation": [          --行评估
              {
                "table": "`test1`",
                "range_analysis": {          --范围分析
                  "table_scan": {
                    "rows": 4804854,          --4804854行数据
                    "cost": 1.03e6            --花费1.03e6
                  },
                  "potential_range_indices": [   --可能的范围指数
                    {
                      "index": "PRIMARY",
                      "usable": true,           --可使用的索引
                      "key_parts": [
                        "id"
                      ]                         --可使用的索引字段
                    },
                    {
                      "index": "k_1",
                      "usable": false,         --不能使用的索引
                      "cause": "not_applicable"  --不被应用
                    }
                  ],
                  "setup_range_conditions": [     --设置范围条件
                  ],
                  "group_index_range": {          --组范围索引
                    "chosen": false,              --不选择的
                    "cause": "not_single_table"   --goup by不是一个表的，所以不选择
                  },
                  "analyzing_range_alternatives": {     --分析每个索引做范围扫描的花费
                    "range_scan_alternatives": [       --范围扫描花费
                      {
                        "index": "PRIMARY",
                        "ranges": [
                          "4999900 < id"
                        ],
                        "index_dives_for_eq_ranges": true,    --索引驱动等值范围扫描
                        "rowid_ordered": true,                --rowid是顺序的
                        "using_mrr": false,                   --不能使用mrr，因为是主键
                        "index_only": false,                  
                        "rows": 99,                           --过滤出来99行
                        "cost": 21.434,                       --花费21.434
                        "chosen": true                        --这个索引被选择选择
                      }
                    ],
                    "analyzing_roworder_intersect": {         --分析执行顺序阶段
                      "usable": false,                        --不可使用
                      "cause": "too_few_roworder_scans"       --少数的执行顺序扫描
                    }
                  },
                  "chosen_range_access_summary": {           --选择范围访问概述
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "PRIMARY",
                      "rows": 99,
                      "ranges": [
                        "4999900 < id"
                      ]
                    },
                    "rows_for_plan": 99,
                    "cost_for_plan": 21.434,
                    "chosen": true
                  }
                }
              },


              ......
```

## explain,show profile 和 trace

- explain：获取 MySQL 中 SQL 语句的执行计划，比如语句是否使用了关联查询、是否使用了索引、扫描行数等；

- profile：可以清楚了解到SQL到底慢在哪个环节；

- trace：查看优化器如何选择执行计划，获取每个可能的索引选择的代价。

## SQL优化

1. 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

2. 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：
```sql
select id from t where num is null
```

可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：
```sql
select id from t where num=0
```
3. 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

4. 应尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

```sql
select id from t where num=10 or num=20
```

可以这样查询：

```sql
select id from t where num=10
union all
select id from t where num=20
```
5. in 和 not in 也要慎用，否则会导致全表扫描，如：

```sql
select id from t where num in(1,2,3)
```
对于连续的数值，能用 between 就不要用 in 了：

```sql
select id from t where num between 1 and 3
```

6. 下面的查询也将导致全表扫描：

```sql
select id from t where name like '%abc%'
```
7. 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```sql
select id from t where num/2=100

#应改为:
select id from t where num=100*2
```

8.应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```sql
select id from t where substring(name,1,3)='abc'--name以abc开头的id
#应改为:
select id from t where name like 'abc%'
```
9. 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

10. 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。

11. 不要写一些没有意义的查询，如需要生成一个空表结构：

```sql
select col1,col2 into #t from t where 1=0
这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：
create table #t(...)
```


12. 很多时候用 exists 代替 in 是一个好的选择：

```sql
select num from a where num in(select num from b)
用下面的语句替换：
select num from a where exists(select 1 from b where num=a.num)
```

13. 并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。

14. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，
因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。
一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。

15. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。
这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

16. 尽可能的使用 varchar 代替 char ，因为首先变长字段存储空间小，可以节省存储空间，
其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

17. 任何地方都不要使用 select * from t ，用具体的字段列表代替“*”，不要返回用不到的任何字段。

18. 避免频繁创建和删除临时表，以减少系统表资源的消耗。

19. 临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件，最好使用导出表。

20. 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，避免造成大量 log ，
以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

21. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。

22. 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

23. 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。

24. 与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，尤其是在必须引用几个表才能获得所需的数据时。
在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

25. 尽量避免大事务操作，提高系统并发能力。

26. 尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

27. 假如现在有组合索引
第一个索引列dname会使用索引，单独使用第二个索引列loc不会使用索引
组合索引列都使用情况下都会执行索引


28. 模糊查询like
如果like 前加入%通配符会扫描全表，所以不会使用索引

```sql
EXPLAIN SELECT * FROM dept WHERE dname LIKE 'AfZIrJvZNO%'
```

29. 使用or关键字，会不会使用索引
如果要让他使用索引的情况下，保证列都要有索引

```sql
EXPLAIN SELECT * FROM dept WHERE deptno='10070' OR dname='AfZIrJvZNO'
```

30. 判断是否为null 不要使用=null 要用is null

`EXPLAIN SELECT * FROM dept WHERE dname IS NULL`

31. 分组group by，扫描全表，进行排序,不使用索引
不排序 order by null

32. 查询时尽量少用>= <= 等等
33. 多表联查尽量使用连接查询 join
34. 查询数据量较大时，可以采用缓存，分表，分页等等操作减轻压力