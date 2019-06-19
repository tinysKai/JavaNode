# MySQL设计标准

## DDL设计标准

 

-2、**所有表的DDL，都不回退**
-1、**数据库命名规范，统一：vip_xxxx；表名不超过40个字符（即最大只能40个字符）**

0、**表一旦设计好，字段只允许增加，不允许减少（drop column），不允许改名称（change column）**

1、**统一使用INNODB存储引擎,UTF8编码**（整个数据库的编码统一为utf8_general_ci，为此不需要建立表的DDL加上特别CHARACTER SET utf8 COLLATE utf8_general_ci）；

2、**禁用**Stored procedure （包括存储过程，函数，触发器）；

3、**表必须有主键**，建议统一由Auto-Increment字段生成整型，不建议使用组合主键，**自增id只作为虚拟主键，不建议与业务数据处理有关联关系，如果把控不好，会有问题（案例：AUTO_INCREMENT主键字段不要与业务有关联关系）**

```
自增id作为pk的意义是啥？他们的表定义了pk，还需要自增id作为pk吗？
(1)方便dba运维，比如归档，批量更新数据，可以根据id来做范围归档，更方便，如 id >= $i and id < $i+3000 and id < $max and 其他条件，做到更精准的遍历数据
(2)如果使用varchar作为主键，索引比int大多了，bigint是8字节，索引大了，数据大了，IO压力也会加大
(3)按load到内存考虑，varchar 占用的内存空间，是它定义的字符长度的至少3倍
(4)容易产生索引分裂，int，bigint就很好的规避这个问题，InnoDB数据是按照主键聚簇的，数据在物理上按照主键大小顺序存储，使用其他列或者组合无法保证顺序插入，随机IO大，导致插入性能下降
```

4、字段的数据类型和长度，必须符合数据的实际, 不能滥用varchar来解决一切问题：

​                  **自增主键：bigint unsinged**

​                  状态/code字段：tinyint/int/varchar

​                  日期时间：timestamp

​                  日期：date

​                  带小数数值：decimal

​                  整数数值：tinyint/int/bigint (unsigned)

​                  字符：varchar

5、**尽量用单表查询，避免多表JOIN，禁止多于3表join，join的字段数据类型必须绝对一致**。

6、表名列名必须有注释（必须加上COMMENT '<字段扼要解说>'），表结构变更须由库表OWNER所在团队发起，

7、**不要使用TEXT、BLOB、char，请使用VARCHAR(N)**，N表示的是字符数不是字节数，比如VARCHAR(255)，可以最大可存储255个汉字，需要根据实际的宽度来选择N，请注意同一表中，**除blobs数据类型外，所有列允许最大长度之和 所占的 字节数** 不能超过 65535。

8、每张表数据量建议控制在千万级别行以下，为此设计阶段需考虑数据的归档。

9、如果字段只有true or false，请使用tinyint（数值范围-128~127），**禁止使用enum，不方便扩展，使用麻烦(insert/update 必须带''，否则是key非value)**，如模块分类：1订单 2商品；删除标志 0正常，1删除；状态 1为可选，2为不可选等等

10、存储时间（精确到秒）建议使用TIMESTAMP类型，因为TIMESTAMP使用4字节，DATETIME使用8个字节，同时TIMESTAMP具有自动赋值以及自动更新的特性。

11、**禁止default NULL**数字类型not null default 0，字符类型not null default ''，时间not null default '1970-01-01 00:00:00'；

```
已经明确不会成为索引列，为何不能default null
(1)NULL值会有额外的空间来存储，并不是Null标志位只固定占用1个字节==，而是以8为单位，满8个null字段就多1个字节，不满8个也占用1个字节，高位用0补齐。对于相同数据的表，字段中有NULL值的表比NOT NULL的大，为了空间容量和性能，参考：http://mysql.taobao.org/monthly/2016/08/07/
(2)NULL值是不相等的 ,对业务表述可能会有影响,不能=‘NULL’ 或者 = ‘null’，只能 is null，这个最终不利于开发的编写，随着代码迭代，人员的流动容易产生（实际也出现过），产生不必要的麻烦，如代码要写成 col =‘NULL’ or col is null 
(3)count(),max(),min()是忽略NULL的，而count(*)是包含NULL值，那么count(col)和count(*)结果是不一样的
```

12、**所有表必须有create_time和last_update_time，方便后期数据分析与记录变化排查，哪怕只是配置表，只有10行记录;** 为进一步明确操作来源，统一加上created_by, last_updated_by两个字段，记录数据的创建者和修改者。

13、**所有业务实体表/关系表，禁止硬删除，必须软删除，加上is_deleted字段，标注这条记录的状态。**

14、**加字段禁止使用after，因为你不确定全局代码里面（如其他团队使用你的表）是否都insert into table（col，col，col。。。） value，如果你在中间插一个字段，就导致数据偏移的问题了，影响可大可小，同样select \* 的也可能会影响数值的偏移，所以才要求，禁止after，必须带default（第18点要求）。**

15、库名、表名、字段名、索引名必须使用小写字母，**线上MySQL是忽略大小写的（lower_case_table_names=1）**；**（MySQL存储是区分大小写的，****查询要区分大小写，如short url 对应的long url，则select xxx from short_map where binary url = ‘’ ）**

​        **（另外，禁止使用mysql的关键字及保留字作为名称  – mysql5.7 关键字保留字清单）**

 16、表大小控制在3千万行以内，控制行数：
（1）控制DDL变更时间；
（2）同时，缩短整库备份时间，降低恢复难度；
（3）从性能角度看，走索引，每次查询数据精准到就几百行，22亿和1亿性能是没有区别的，如果是比较特殊场景，如where后面不带时间范围条件，而你明知道n个月前的数据不需要，全表查某个类型（type）的查询就有性能问题了。

| 表行数           | 1千万行 | 2千万行 | 3千万行 | 4千万行 | 备注 |
| ---------------- | ------- | ------- | ------- | ------- | ---- |
| DDL变更需要n小时 | 1       | 2       | 3       | 4       |      |

Online DDL的速度：总行数/3000行每秒/3600秒= n 小时，**注意：一个库要变更10张表，只能是串行，不能并行，假设每一张表都要1小时的话，那么10张表变更完，就是10个小时**



## SQL语句标准

 

0、**禁止多于2表的join。**

1、使用prepared statement，可以提供性能并且避免SQL注入。

2、SELECT语句只获取需要的字段，禁止使用SELECT * FROM语句，这是有效防止新增字段对应用逻辑的影响，还能减少对性能的影响；

3、INSERT语句必须显式的指明字段名称，不使用INSERT INTO table value()。

4、禁止在where子句中对字段施加函数，如to_date（add_time)>xxxxx,应改为:add_time >= unix_timestamp(date_add(str_to_date('20130227','%Y%m%d'),interval - 29 day))

5、UPDATE、DELETE语句不使用LIMIT 。**以前我们使用的是MySQL 5.0，使用statment模式，所以有此规范，目前5.5，row和mixed模式不会出现，此规则去掉。**

6、写到应用程序里的SQL语句，禁止一切DDL操作，如对这些权限有要求，必需与DBA协商同意方可使用

7、WHERE条件中必须使用合适的类型，避免MySQL进行隐式类型转化，如ISENDED=1，字段类型是tinyint，那么不能是ISENDED=‘1’。

8、避免在SQL语句进行数学运算或者函数运算，容易将业务逻辑和DB耦合在一起。

9、INSERT语句使用batch提交。

10、避免使用存储过程、触发器、函数等，容易将业务逻辑和DB耦合在一起，并且MySQL的存储过程、触发器、函数中存在一定的bug。

11、使用合理的SQL语句减少与数据库的交互次数。

12、不使用ORDER BY RAND()，使用其他方法替换。

13、建议使用合理的分页方式以提高分页的效率。

14、InnoDB表避免使用COUNT(*)操作，计数统计实时要求较强可以使用memcache或者redis，非实时统计可以使用单独统计表，定时更新。

15、不建议使用%前缀模糊查询，例如LIKE “%weibo”。

16、避免多余的排序。使用GROUP BY 时，默认会进行排序，当你不需要排序时，可以使用order by null,例如Select a.OwnerUserID,count(*) cnt from DP_MessageList a group by a.OwnerUserID order by null;

17.**新增排序要求：不鼓励在DB里排序，特别是只有1000行一下的，请在app server上排序，app server有上百台，而db仅仅个位数的服务器数量，排序都在db，会把db压垮的，特别是禁止上千行的排序在db这边。**

18**、**禁止使用 REPLACE INTO ；  

 **19、禁止使用子查询，select col、col from table where id in （select col from table）这是禁止的；**

 20、batch size 大小不能超过1000，同时请根据业务QPS和记录长度来评估1000以内什么值合适，如where col in （）的值不能超过1000。

 21、**禁止使用** UUID()，USER()这样的MYSQL INSIDE函数对于复制来说是很危险的，会导致主备数据不一致，重要的是会严重影响mysql性能。

 22、 如果应用使用的是长连接，应用必须具有自动重连的机制。但请避免每执行一个SQL去检查一次DB可用性； 



每个整数类型的存储和范围。

 

| **类型**           | **字节** | **最小值**              | **最大值**              |
| ------------------ | -------- | ----------------------- | ----------------------- |
|                    |          | **(带符号的/无符号的)** | **(带符号的/无符号的)** |
| TINYINT            | 1        | -128                    | 127                     |
| TINYINT unsigned   |          | 0                       | 255                     |
| SMALLINT           | 2        | -32768                  | 32767                   |
| SMALLINT unsigned  |          | 0                       | 65535                   |
| MEDIUMINT          | 3        | -8388608                | 8388607                 |
| MEDIUMINT unsigned |          | 0                       | 16777215                |
| INT                | 4        | -2147483648             | 2147483647              |
| INT unsigned       |          | 0                       | 4294967295              |
| BIGINT             | 8        | -9223372036854775808    | 9223372036854775807     |
| BIGINT unsigned    |          | 0                       | 18446744073709551615    |



 `DECIMAL(*M*,*D*) 最大 DECIMAL(*65*,*30*)，其中对应下表的占用字节数`

 

| Leftover Digits | Number of Bytes |
| --------------- | --------------- |
| 0               | 0               |
| 1–2             | 1               |
| 3–4             | 2               |
| 5–6             | 3               |
| 7–9             | 4               |



## 分区表使用规范

**原则上：禁止使用分区表！禁止使用分区表！禁止使用分区表！**

+ 分区表也是一个db特性，少一个特性，少一个功能bug的风险
+ 其实分区表解决的是，单表大数据量，然后这些数据不太重要，需要定期drop partition清理，方便清理而已，真正带来查询效率的，是索引和数据访问方式
+ **DBA无法做Online DDL，这个才是重点中的重点**

如果一定要用遵循
1、单表大数据量且有一定的字段冗余以后都不会做DDL了
2、然后这些数据生命周期很短，不太重要，**不需要归档，可以直接清理的**，定期drop partition可以方便清理，如监控数据，告警数据，一些日志数据等

# 读写分离使用标准

1、需在设计阶段考虑如果访问量非常大，且不做scale out表拆分的话，需读写分离，但读写分离注意主从复制有延迟的可能性； 

 

# 事务的处理标准

0、一个事务，处理的行数不能超过1000 rows/s 

1、禁止一些框架或定制化的底层类等使用set autocommit=0；set autocommit=1；这样控制事务，应该由程序把控，需要时begin；操作完后及时commit；(其实做不到,spring本身类似就是这么做的)

2、要合理使用事务，例子：购物车如下的 事务处理，可以更好的优化。



# 索引使用标准

 

1、非唯一索引建议使用“idx_表缩写名称_字段缩写名称”进行命名。

2、唯一索引建议使用“uniq_表缩写名称_字段缩写名称”进行命名。

3、索引名称必须使用小写。

4、唯一键不和主键重复。每个业务实体表和关系表都应该至少有一个业务主键对应的唯一索引。

5、索引字段的顺序需要考虑字段值去重之后的个数，个数多的放在前面，就是数据分布。

6、使用EXPLAIN判断SQL语句是否合理使用索引，尽量避免extra列出现：Using File Sort，Using Temporary。

7、UPDATE、DELETE语句需要根据WHERE条件添加索引。

8、合理创建联合索引（避免冗余），(a,b,c) 相当于 (a) 、(a,b) 、(a,b,c)。

9、合理利用覆盖索引。比如SELECT email,uid FROM user_email WHERE uid=xx，如果uid不是主键，适当时候可以将索引添加为index(uid,email)，以获得性能提升。

 

 

# 约束设计

a) 主键的内容不能被修改。

b) 外键约束一般不在数据库上创建，只表达一个逻辑的概念，由程序控制。

d) 禁用数据库外键

命名

a) 主键约束：默认PRIMARY；

b) unique约束：UK_<column_name>

c) check约束： CK_<column_name>

d) 外键约束： 业务禁用