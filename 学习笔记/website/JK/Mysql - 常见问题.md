## Mysql - 常见问题

#### mysql偶尔的查询慢原因

- redo log写满了,需将redo log刷新到磁盘
- 内存不足时,需淘汰现内存中的页,对于脏页需将数据刷新到磁盘.刷新脏页时常态,但若一个查询需淘汰的脏页过多则会影响查询响应时间变长

#### 删除表数据为啥表文件大小不变

我们在删除整个表的时候，可以使用 drop table 命令回收表空间。但删除记录是不会回收页的,只会**重新复用**.而且删除的记录有可能只是占用到某页的其中几部分,页中的数据没全部被删除也不能回收.

可以使用` alter table A engine=InnoDB `命令来重建表来达到收缩表空间的作用,这个操作是支持`online ddl`的.

#### mysql表中字段常见的varchar(255)设计原因

小于255都需要一个字节记录长度，超过255就需要两个字节.



#### 两个表的关联查询无法应用到索引分析

```sql
mysql> select d.* from tradelog l, trade_detail d 
	where d.tradeid=l.tradeid and l.id=2; 
```

explain后,tradelog为驱动表,trade_detail为被驱动表,并且被驱动关联上用不到索引(两个表的traceid都建了索引的).其查询的过程是先到tradelog表查询一条条记录的traceid,再到trade_detail查询明细.

![eblbH1.png](http://ww1.sinaimg.cn/large/8bb38904ly1g65dz123fjj213804r3z5.jpg)

在到trade_detail表根据traceid去查询记录时,本应该是能用到索引的,但此时用不上,经常发生的场景是两个表的字符集不同,此案例的trade_detail表为`utf8`,而tradelog表为`utf8mb4`

#### 针对简单查询长时间不返回的排查

```sql
mysql> select * from t where id=1;
```

针对sql语句慢的问题,我们可以先使用`show processlist`来查看具体的执行语句,观察可得查询语句在等到MDL锁.所以我们可以排查下是谁持有MDL写锁而长时间不释放,然后将其kill掉.

![https://s2.ax1x.com/2019/08/09/ebBiQI.png](http://ww1.sinaimg.cn/large/8bb38904ly1g65dzyga2oj210g05xt9l.jpg)

可在MySQL 启动时设置 performance_schema=on来排查，相比于设置为 off 会有 10% 左右的性能损失.通过查询 `sys.schema_table_lock_waits` 这张表，我们就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。

![https://s2.ax1x.com/2019/08/09/ebBh6I.png](http://ww1.sinaimg.cn/large/8bb38904ly1g65e0ujpsfj20q406z74h.jpg)

#### 解决行锁冲突步骤

```sql
mysql> select * from t sys.innodb_lock_waits 
	where 
		locked_table=`'${database}'.'${table}'`\G
```

![https://s2.ax1x.com/2019/08/09/ebWQ4x.png](http://ww1.sinaimg.cn/large/8bb38904ly1g65e1huop2j20o00iln0e.jpg)

#### 一个有趣的查询

```sql
mysql> CREATE TABLE `table_a` (
  `id` int(11) NOT NULL,
  `b` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `b` (`b`)
) ENGINE=InnoDB;

--背景 假设现在表里面，有 100 万行数据，其中有 10 万行数据的 b 的值是’1234567890’

mysql> select * from table_a where b='1234567890abcd'; 
-- 注意这里的字符串值已经超过表格定义的长度10

-- 正常来讲,应该是木有记录符合,mysql应该能比较机制的判断并很快返回的,但这个查询却执行得很慢
```

mysql的执行过程

1. 在传给引擎执行的时候，做了字符截断。因为引擎里面这个行只定义了长度是 10，所以只截了前 10 个字节，就是’1234567890’进去做匹配；这样满足条件的数据有 10 万行；
2. 因为是 select *， 所以要做 10 万次回表；
3. 每次回表以后查出整行，到 server 层一判断，b 的值都不是’1234567890abcd’;
4. 返回结果是空。

#### 间隙锁引发的死锁

背景

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);

```

![https://s2.ax1x.com/2019/08/09/eqpi38.png](http://ww1.sinaimg.cn/large/8bb38904ly1g65e2c641fj20q80a0wfb.jpg)

由于木有`id = 9`的记录,A会加上间隙锁(5,10],而B同理也会加上(5,10]的间隙锁.

B中执行insert时会等待A中的间隙锁,而A中执行insert时也需等待B的间隙锁.因此造成了死锁.

#### 主备出现延迟的原因

1. 主备机器性能不一致
2. 大事务
3. 执行DDL操作
4. 备库在执行数据备份操作
5. 备库的查询量大
6. 主库并发量大能并发执行语句,而从库的解析语句可能是单线程模式的

#### Mysql大表查询流程

一行一行的查询数据发送往`net_buffer`,当`net_buffer`满时发往网络的`socket sendbuffer`,最终通过网络发送到客户端的`socket receive buffer`.在mysql数据是`边读边发的`,期间占用的最大内存基本是`net_buffer`的大小.可通过`net_buffer_length `参数来修改对应的大小.

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5vs7dz3flj20uc0fgdhu.jpg)

#### 长事务的影响

+ 长事务中的更新语句会占用着行锁,会导致别的语句更新被锁住
+ 长事务中的读语句会导致 undo log不能被回收，导致回滚段空间膨胀。



#### join语句驱动表的选择

**选择小表作为驱动表**

+ 在被驱动表能使用到索引的情况下,小表做驱动表的全表扫描数据量少
+ 在被驱动表用不到索引的情况下,小表做驱动表依旧有优势

一个被驱动表使用了索引的explain结果以及流程如下

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5vtg0cr7dj212q04j3z3.jpg)

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5vthklz7nj20vq0og76y.jpg)

若被驱动表用不上索引,mysql会使用`Block Nested-Loop Join`来关联查询,，算法的流程是这样的：

1. 把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 select *，因此是把整个表 t1 放入了内存；
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。

#### union与union all的区别

+ union会去重,而`union all`不去重
+ union由于要去重所以用到了内部临时表(explain下在extra会有`Using temporary`)

#### 自增主键不连续的原因

innodb的自增值在mysql 8.0前是保存在内存里的,直到mysql 8.0才将其持久化.因此mysql重启后读的自增值是计算表里面id的最大值加上步长的.innodb出于性能考虑,在唯一索引冲突以及事务回滚时不回退自增id,因此只保证了自增id是递增的,但不保证是连续的.以下是出现不连续的情况 : 

+ insert数据时唯一索引冲突,自增值是不回滚的
+ 事务回滚时,自增值也是不回滚的
+ mysql重启后手动删除最大id的记录是能重复创建两条一样主键的记录的
+ 批量插入数据时,mysql自增申请的策略是指数增长的,如1,2,4每次按照这个数量递增申请的,那最后浪费掉的主键id也会造成不连续



#### insert into … on duplicate key update

`insert into … on duplicate key update` 这个语义的逻辑是，插入一行数据，如果碰到唯一键约束，就执行后面的更新语句。

针对表 t (唯一索引在c列)里面已经有了 (1,1,1) 和 (2,2,2) 这两行执行以下更新 : 

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5yc436igjj20lb06e74t.jpg)

需要注意的是，执行这条语句的 affected rows 返回的是 2，很容易造成误解。实际上，真正更新的只有一行，只是在代码实现上，insert 和 update 都认为自己成功了，update 计数加了 1， insert 计数也加了 1。