# Redis的Info指令

## Info获取的信息分类

- Server 服务器运行的环境参数
- Clients 客户端相关信息
- Memory 服务器运行内存统计数据
- Persistence 持久化信息
- Stats 通用统计数据
- Replication 主从复制相关信息
- CPU 使用情况
- Cluster 集群信息
- KeySpace 键值对统计数量信息

Info 可以一次性获取所有的信息，也可以按块取信息。

```
# 获取所有信息
> info
# 获取内存相关信息
> info memory
# 获取复制相关信息
> info replication
```



## Info指令的使用

####  查询指令次数

```
//查看OPS,每秒操作数
> redis-cli info stats |grep ops
instantaneous_ops_per_sec:789

//如果OPS过高,可使用monitor命令来持续观察期间操作的指令
[root@wujy-bir2n bin]# ./redis-cli
127.0.0.1:6379> monitor
OK
1556367502.597070 [0 127.0.0.1:37149] "keys" "*"
1556367512.639816 [0 127.0.0.1:37149] "keys" "*"

//观察完毕后可以按Ctrl + C 结束观察
[root@wujy-bir2n bin]# 

```



#### 查询客户端连接

```
127.0.0.1:6379> info clients
# Clients
connected_clients:1		//连接的客户端数量
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

//当链接数过多时,可使用client list来查询具体的链接,注意age属性表示链接的时长,单位为秒
127.0.0.1:6379> client list
addr=127.0.0.1:37149 fd=8 name= age=1535 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
127.0.0.1:6379> 
```



#### 查看内存

```
127.0.0.1:6379> info memory
# Memory
used_memory:808432
used_memory_human:789.48K # 内存分配器 (jemalloc) 从操作系统分配的内存总量
used_memory_rss:2154496 # 操作系统看到的内存占用 ,top 命令看到的内存
used_memory_peak:829256
used_memory_peak_human:809.82K   # Redis 内存消耗的峰值
used_memory_lua:33792
mem_fragmentation_ratio:2.67
mem_allocator:jemalloc-3.2.0
```



#### 查看主从复制效率

```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576  # 这个就是积压缓冲区大小
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
127.0.0.1:6379> 

```



#### 查看半同步失败次数

主从同步的指令缓存区是唤醒的,当主从延迟较多时,后面的指令会覆盖掉前面的指令.这时从只能通过快照来进行跟主同步.而快照是很耗费资源的一个操作,所以有时候需关注半同步状态

```
127.0.0.1:6379> info stats
# Stats
total_connections_received:2
total_commands_processed:9
instantaneous_ops_per_sec:0
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0		#半同步失败次数
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
127.0.0.1:6379> 

```

