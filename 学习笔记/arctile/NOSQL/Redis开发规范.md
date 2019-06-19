## Redis开发规范

#### 慎用的命令

由于redis是单线程来处理客户端的请求的,所以如果个别请求处理时间较长,则势必会影响下一个请求的访问

我们需注意复杂度为O(n)的指令操作,若其n不可控,则使用O(n)指令会有相应的风险

以下为常见的需注意的O(n)指令

+ lrange (禁用lrange(0,-1))
+ hgetall 
+ smember(当需遍历集合的时候,可使用sscan来迭代)



#### 建议多使用多命令参数

+ mset
+ mget
+ lpush,rpush(代替lset)
+ lrange(代替lindex,`LRANGE key start stop`时间复杂度: O(S+N)， S为偏移量 `start` ， N为指定区间内元素的数量)
+ hmset
+ hmget

需要注意的是，当操作的数据量过大的时候，容易阻塞其它连接的操作。



#### 内存使用建议

redis机器上物理内存应该保留一半以上的空闲内存作为redis bgsave或者aofbgrewrite使用。否则在做rdb（bgsave）或者aof bgrewrite时候出现大量的swap，导致redis性能严重下降。单个redis实例内存maxmemory不大于10G

bgsave和bgrewriteaof是redis进行持久化以及复制功能的解决方案，目前数据库平台使用master-slave方式保证高可用，master主写主读，slave仅用来做灾备用，主从都不做持久化。



#### Redis去重时的优化

比如`邮件去重`可以使用多个set,按照邮件的第一个字母来分散到不同的set中,减少比较的维度





#### Redis超时配置

+ connectionTimeout (链接超时,默认为2s,建议为200ms)
+ soTimeout (读超时,默认为2s,建议为200ms,读基本为0.02ms,但这里需注意公网的网络波动,若网络不太稳定200ms不一定合适)

由于超时和重试的机制对应用在缓存节点故障期间的可用性影响较大，如果请求的总的超时时间过长会大量BLOCK应用的线程导致服务不可用，所以清楚的了解应用在极限情况下的最大超时时间非常重要。这里对相关请求的超时时间的计算做重点说明：

- redis cluster: 请求最大响应时间`maxRT=max(attempts * connectionTimeout, maxWaitMillis+soTimeout)` 使用推荐配置则maxRT=600ms
- redis sharding: 请求最大响应时间maxRT=max(connectionTimeout, maxWaitMillis+soTimeout),使用推荐配置则maxRT=450ms

以上均为异常情况下的极限的缓存访问的时间，应用可以根据自己的情况相应的做些调整。

**需要注意的是，目前的Jedis一共有下列几个方法会无视用户设置的SoTimeout并且在执行前将SoTimeout设置为0：**

1. blpop
2. brpop
3. brpoplpush
4. eval
5. evalsha

**使用这些命令需要特别注意，线程会被长时间BLOCK。**

