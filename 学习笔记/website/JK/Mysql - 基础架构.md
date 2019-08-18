## Mysql - 基础架构

### 逻辑架构

MySQL架构主体分为服务层以及存储引擎层.

Server 层涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。而存储引擎层负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。

![https://s2.ax1x.com/2019/08/05/e2Sc38.png](http://ww1.sinaimg.cn/large/8bb38904gy1g5zhrd8tlej21hc140tpc.jpg)

#### 查询缓存

查询缓存由于对表中记录更新即会清空这张表的缓存,因此不建议在MySQL中使用查询缓存.

 MySQL 也提供了这种“按需使用”的方式。可以将参数 `query_cache_type` 设置成 DEMAND，这样对于默认的 SQL 语句都不使用查询缓存。而对于你确定要使用查询缓存的语句，可以用` SQL_CACHE `显式指定，像下面这个语句一样：

```sql
mysql> select SQL_CACHE * from T where ID=10；
```

### 日志模块

redo log（重做日志）和 binlog（归档日志）。

#### WAL

WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。

具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面。

WAL能提高写速度是收益于

1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。

#### redo log

InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![<https://s2.ax1x.com/2019/08/05/e2V9PJ.png>](http://ww1.sinaimg.cn/large/8bb38904gy1g5zhqf2ovej20vq0nsdj5.jpg))

`write pos` 是当前记录的位置,`checkpoint`是当前要擦除的位置.`write pos`和 `checkpoint `之间的是还空着的部分，可以用来记录新的操作。

有了 redo log，InnoDB(redo-log是引擎层特有的日志文件) 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为`crash-safe`.	`innodb_flush_log_at_trx_commit`这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数建议设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。

#### redo log的两阶段提交

`redo log prepare阶段`  --> `写进binlog` --> `redo log commit`

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5vfr4yg1aj20vq16adkz.jpg)

通常我们说 MySQL 的“双 1”配置，指的就是 `sync_binlog` 和 `innodb_flush_log_at_trx_commit` 都设置成 1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。

**两阶段提交的目的是为了让两份日志之间的数据一致.**

分析需要两阶段的原因 ,假设不使用两阶段提交,则假设以下场景 : 

1.写redo log成功后binlog写入失败,则MySQL恢复后由于redo log的`crash-safe`能恢复执行的命令结果,而binlog无对应的记录数据,因此会在主从同步或其它需要binlog的场景出现数据不一致的情况

2.先写binlog后写redo log也会同理出现数据不一致的情况.

#### binlog

binlog 的写入逻辑比较简单：事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。

`sync_binlog` 这个参数设置成 1 的时候，表示每次事务的 binlog都持久化到磁盘。这个参数建议设置成 1，这样可以保证 MySQL异常重启之后 binlog不丢失。

用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。

```sql
-- 将 master.000001 文件里面从第 2738 字节到第 2973 字节中间这段内容解析出来，放到 MySQL 去执行
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```



#### undo log

语句更新时候还会生成undo log(回滚日志),回滚日志在事务回滚的时候可以将修改的值恢复到事务前的状态.

#### 临时表特点

+ 一个临时表只能被创建它的 session 访问，对其他线程不可见。
+ 临时表可以与普通表同名。
+ 同一会话内有同名的临时表和普通表的时候，show create 语句，以及增删改查语句访问的是临时表。
+ show tables 命令不显示临时表。
+ 临时表只能被创建它的 session 访问，所以在这个 session 结束的时候，会自动删除临时表。

#### 分区表

分区表是针对引擎的,因此对server层来说多个分区也只有一张表,而对引擎来说是像多张表一样.

缺点

+ 第一次访问分区表的某个表时需访问所有的分区表
+ 所有分区共享一个MDL锁,如针对某分区执行DDL时,会因其它分区的查询语句而需等待MDL锁