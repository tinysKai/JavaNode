#### redis只使用db0
避免使用select <db> ，使用登录上去默认的db0，（如果使用其他的db，发出的select <db>是包含在qps里的，那么也会消耗redis的资源）。  

#### redis慎用命令
+ lrange 禁用（0，-1）//当list列表过长的时候，每次的lrange 0 -1 都会取回全部列表，造成大量的cpu消耗，并且，当lrange start end的start较大的时候，时间复杂度为0(start+[end -start]) .
+ hgetall //hgetall和lrange类似，当hgetall的hashmap 长度过大的时候，会取回全部的hashmap，造成大量的cpu消耗。
+ smember 当set元素很多的，redis的相应时间很长，可以考虑使用sscan迭代方式。

#### 建议使用多参数命令（multi-argument commands）
使用多参数命令主要的有点是可以减少redis的响应延迟。redis单进程响应所有连接的操作请求，队列中最近的命令需要等待之前所有的命令完成才能开始执行。   

![redis](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/redisM.png)  

例如，需要添加1000个元素到一个list中，使用lset需要1000次请求，使用lpush/rpush只需要一次请求。
需要注意的是，当操作的数据量过大的时候，容易阻塞其它连接的操作。


#### 建议使用pipeline [redis cluster主流客户端驱动都不支持]
在客户端，使用pipeline可以把多个命令一次行发送到redis server，这样可以减少很多client与server之间通信roundtrips，多次网络延迟开销减少为1次  

需注意 :  
+ 当pipeline中命令的个数太多，返回的结果数据过大，可能导致接收缓冲区的溢出
+ timeout现象(redis客户端（程序端）socket在一段时间内没得到响应，会话超时会断开redis的连接。)  

#### redis故障处理
正确处理回写  
+ 当redis出现故障的时候，要有回源的数据库或者底层系统，回源超时时间要设置合理；
+ 当redis出现故障的时候，要合理处理回写redis策略，特别是如果redis服务器持续过载或者服务器down机的时候，回写很可能会失败。所以不管回写是否成功，都要返回回源数据给调用方  
+ 当redis出现故障的时候，要合理评估底层系统或者数据库（往往是MySQL）的QPS能力和响应时间，要选择性降级前端业务，确保不进一步压死数据库或者底层系统  





