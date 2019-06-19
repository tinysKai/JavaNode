## Redis客户端的使用方式

#### redis客户端的使用模式

+ 交互模式
+ 直接模式

交互模式就是使用`./redis-cli`或`./redis-cli -h localhost -p port`.

```
[root@wujy-bir2n bin]# ./redis-cli
127.0.0.1:6379> set key1 1
OK
127.0.0.1:6379> get key1
"1"
127.0.0.1:6379> 
```

直接模式就是直接将命令参数传递给`redis-cli`脚本来单词执行

```
[root@wujy-bir2n bin]# 
[root@wujy-bir2n bin]# ./redis-cli set key2 2
OK
[root@wujy-bir2n bin]# ./redis-cli get key2
"2"
[root@wujy-bir2n bin]# 
```



#### 批量执行命令

+ 管道
+ 重定向

**管道模式** - 在文本文件中存储执行命令,然后使用`cat file |redis-cli`命令来批量操作

**重定向模式** - 将文件的输出流指向`/redis-cli < tmp.txt`



#### 重复执行指令

redis-cli 还支持重复执行指令多次，每条指令执行之间设置一个间隔时间，如此便可以观察某条指令的输出内容随时间变化。

```
//一共运行三次,间隔一秒,观察ops的变化
[root@wujy-bir2n bin]# ./redis-cli -r 3 -i 1 info | grep ops
instantaneous_ops_per_sec:0
instantaneous_ops_per_sec:0
instantaneous_ops_per_sec:0
```



#### 监控服务器状态

```
//-i可控制输出时间间隔
[root@wujy-bir2n bin]# ./redis-cli --stat [-i 1]
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections          
4          790.62K  1       0       23 (+0)             14          
4          790.62K  1       0       24 (+1)             14          
4          790.62K  1       0       25 (+1)             14          
4          790.62K  1       0       26 (+1)             14          
4          790.62K  1       0       27 (+1)             14          
4          790.62K  1       0       28 (+1)             14          
4          790.62K  1       0       29 (+1)             14          
4          790.62K  1       0       30 (+1)             14         
```

#### 运行lua脚本

```
//2表示有两个key,即KEY[1],KEY[2],在key输入完毕后就是输入key对应的AGRV的参数值
127.0.0.1:6379> eval "return redis.pcall('mset', KEYS[1], ARGV[1], KEYS[2], ARGV[2])" 2 key3 key4 3 4
OK

```



#### 扫描大KEY

redis-cli 对于每一种对象类型都会记录长度最大的 KEY，对于每一种对象类型，刷新一次最高记录就会立即输出一次。它能保证输出长度为 Top1 的 KEY，但是 Top2、Top3等 KEY 是无法保证可以扫描出来的。一般的处理方法是多扫描几次，或者是消灭了 Top1 的 KEY 之后再扫描确认还有没有次大的 KEY。

```
$ ./redis-cli --bigkeys -i 0.01
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest zset   found so far 'hist:aht:main:async_finish:20180425:17' with 1440 members
[00.00%] Biggest zset   found so far 'hist:qps:async:authorize:20170311:27' with 2465 members
[00.00%] Biggest hash   found so far 'job:counters:6ya9ypu6ckcl' with 3 fields
[00.01%] Biggest string found so far 'rt:aht:main:device_online:68:{-4}' with 4 bytes
[00.01%] Biggest zset   found so far 'machine:load:20180709' with 2879 members
[00.02%] Biggest string found so far '6y6fze8kj7cy:{-7}' with 90 bytes
```



#### 指令监控

使用monitor指令能得出redis服务器接收到的指令

```
[root@wujy-bir2n bin]# ./redis-cli
127.0.0.1:6379> monitor
OK
1557322952.142627 [0 127.0.0.1:54354] "get" "key1"
1557322956.850602 [0 127.0.0.1:54354] "get" "key2"
1557322960.863751 [0 127.0.0.1:54354] "set" "key5" "5"
```



#### 参考链接

<https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5bcfd27051882577e962f064>



