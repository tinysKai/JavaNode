## Mysql慢SQL排查

#### 查询语句

```sql
 -- 查询慢SQL登记的阈值,单位秒
  SHOW VARIABLES LIKE 'long%';
 
  
 -- 查询slow相关配置 
 SHOW VARIABLES LIKE 'slow%';
 
 -- 未使用索引的查询也被记录到慢查询日志中（可选项）
  SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
 
 -- 查询是否写到表格
 SHOW VARIABLES LIKE 'log_output%';
 
 -- 设置慢SQL到表格,TABLE/FILE
 -- log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中
 -- MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。
 -- 使用set 方式只对当前数据库生效,重启后失效,想一直生效需修改my.cnf
 SET GLOBAL log_output='TABLE';
 
  
  -- 查询记录到表的慢SQL
  SELECT * FROM mysql.slow_log WHERE start_time > '2019/12/11 15:00:00'; 
  
  
```

```sql
-- 如果未开启则可使用以下语句开启

set global slow_query_log='ON'; //开启慢SQL日志
set global slow_query_log_file='/var/lib/mysql/test-slow.log';//记录日志地址
set global long_query_time=1;//最大执行时间
```



#### 查询展示

![微信截图_20191212093337.png](http://ww1.sinaimg.cn/large/8bb38904gy1g9torv6nlcj20c803c0so.jpg)



#### 日志分析

  针对日志文件可使用`mysqldumpslow`来分析日志

```shell
[root@DB-Server ~]# mysqldumpslow --help
Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]
 
Parse and summarize the MySQL slow query log. Options are
 
  --verbose    verbose
  --debug      debug
  --help       write this text to standard output
 
  -v           verbose
  -d           debug
  -s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
                al: average lock time
                ar: average rows sent
                at: average query time
                 c: count
                 l: lock time
                 r: rows sent
                 t: query time  
  -r           reverse the sort order (largest last instead of first)
  -t NUM       just show the top n queries
  -a           don't abstract all numbers to N and strings to 'S'
  -n NUM       abstract numbers with at least n digits within names
  -g PATTERN   grep: only consider stmts that include this string
  -h HOSTNAME  hostname of db server for *-slow.log filename (can be wildcard),
               default is '*', i.e. match all
  -i NAME      name of server instance (if using mysql.server startup script)
  -l           don't subtract lock time from total time
```



```shell
# -s, 是表示按照何种方式排序
	#c: 访问计数
	#l: 锁定时间
	#r: 返回记录
	#t: 查询时间
	#al:平均锁定时间
	#ar:平均返回记录数
	#at:平均查询时间

#-t, 是top n的意思，即为返回前面多少条的数据；

# -g, 后边可以写一个正则匹配模式，大小写不敏感的；

# 得到返回记录集最多的10个SQL。
mysqldumpslow -s r -t 10 /tmp/mysql06_slow.log

# 得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /tmp/mysql06_slow.log

# 得到按照时间排序的前10条里面含有左连接的查询语句。
mysqldumpslow -s t -t 10 -g “left join” /tmp/mysql06_slow.log

# 另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现刷屏的情况。
mysqldumpslow -s r -t 20 /tmp/mysql06-slow.log | more
```



