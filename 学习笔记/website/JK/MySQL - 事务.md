## MySQL - 事务

#### 定义

事务就是保证一组数据库操作要么都成功,要么全部失败.在MySQL中事务是在引擎层实现的.

#### 事务特性

ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性)

#### 隔离级别

| 隔离级别                  | 定义                                                         | 解决的问题                               |
| ------------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| 未提交读(read uncommited) | 一个事务能读到其他事务未提交的数据                           | /                                        |
| 读提交(read commited)     | 一个事务提交之后它的数据才能被其它事务看到                   | 脏读                                     |
| 可重复读(repeatable read) | 同一个事务多次执行同一条指令看到的结果是一样的(默认的隔离级别) | 脏读,不可重复读,幻读(innodb下可解决幻读) |
| 串行化读(serializable)    | 一个事务需等待前一个事务执行完成后才能执行                   | 脏读,不可重复读,幻读                     |

#### 隔离级别的实现

在隔离级别的实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。在“读提交”隔离级别下，这个视图是在每个 SQL 语句开始执行的时候创建的。这里需要注意的是，“读未提交”隔离级别下直接返回记录上的最新值，没有视图概念；而“串行化”隔离级别下直接用加锁的方式来避免并行访问。

#### 查看事务隔离级别

mysql> show variables like 'transaction_isolation';

#### 事务的启动方式

+ 显式启动事务语句， begin 或 start transaction。配套的提交语句是 commit，回滚语句是 rollback。
+ set autocommit=0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个 select 语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行 commit 或 rollback 语句，或者断开连接。

因此，建议总是使用 set autocommit=1, 通过显式语句的方式来启动事务。

#### 查询长事务

```sql
select * from information_schema.innodb_trx 
	where 
	TIME_TO_SEC(timediff(now(),trx_started))>60
```

#### 开启事务

`begin/start transaction` 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用` start transaction with consistent snapshot` 这个命令。

#### 事务隔离性

对于一个事务视图来说，除了自己的更新总是可见以外，有三种情况：

+ 版本未提交，不可见；
+ 版本已提交，但是是在视图创建后提交的，不可见；
+ 版本已提交，而且是在视图创建前提交的，可见。

更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”（current read）,当前读需要加锁.

可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

+ 对于可重复读，查询只承认在事务启动前就已经提交完成的数据；
+ 对于读提交，查询只承认在语句启动前就已经提交完成的数据；
+ 而当前读，总是读取已经提交完成的最新版本。

#### GTID

GTID 的全称是 Global Transaction Identifier，也就是全局事务 ID，是一个事务在提交的时候生成的，是这个事务的唯一标识。它由两部分组成，官方文档的格式是：`GTID=source_id:transaction_id`,其中：

- source_id是一个实例第一次启动时自动生成的，是一个全局唯一的值；
- transaction_id是一个整数，初始值是 1，每次提交事务的时候分配给这个事务，并加 1。

但建议理解为`GTID=server_uuid:gno`

其中：

- server_uuid 是一个实例第一次启动时自动生成的，是一个全局唯一的值；
- gno 是一个整数，初始值是 1，每次提交事务的时候分配给这个事务，并加 1。

官方文档后面的这个 transaction_id，我觉得容易造成误导，所以我改成了 gno。为什么说使用 transaction_id 容易造成误解呢？

因为，在 MySQL 里面我们说 transaction_id 就是指事务 id，事务 id 是在事务执行过程中分配的，如果这个事务回滚了，事务 id 也会递增，而 gno 是在事务提交的时候才会分配。

从效果上看，GTID 往往是连续的，因此我们用 gno 来表示更容易理解。

GTID 模式的启动也很简单，我们只需要在启动一个 MySQL 实例的时候，加上参数 gtid_mode=on 和 enforce_gtid_consistency=on 就可以了。



#### 查看命令

```sql
-- 搜数据库隔离级别
select @@global.tx_isolation;
-- 搜数据库隔离级别
select @@tx_isolation;

--查看你的mysql当前默认的存储引擎:
show variables like '%storage_engine%';

-- 当前运行的所有事务
select * from information_schema.innodb_trx;
-- 当前出现的锁
select * from information_schema.innodb_locks;
-- 锁等待的对应关系
select * from information_schema.innodb_lock_waits;

-- show processlist结果筛选
SELECT ID,USER,HOST,DB,command,TIME,state,info FROM information_schema.processlist
	WHERE DB LIKE '%${databaseName}%';

--查询指定数据库以及状态
SELECT ID,USER,HOST,DB,command,TIME,state,info FROM information_schema.processlist 
	WHERE DB LIKE '%${databaseName}%' AND state LIKE "%Lock%";
```

