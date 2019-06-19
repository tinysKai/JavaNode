## REDIS-深度历险读书笔记
本笔记只是个人学习使用,系统学习请访问作者原小册连接 : *https://juejin.im/book/5afc2e5f6fb9a07a9b362527*

#### REDIS的5种基础数据结构
+ string 
+ list
+ hash
+ set
+ zset

#### string字符串结构
Redis的字符串是动态字符串，是可以修改的字符串，内部结构实现上类似于Java的ArrayList,采用预分配冗余空间的方式来减少内存的频繁分配，  
如图中所示，内部为当前字符串实际分配的空间capacity一般要高于实际字符串长度len。当字符串长度小于1M时，扩容都是加倍现有的空间，如果超过1M，扩容时一次只会多扩1M的空间。需要注意的是字符串最大长度为512M。  

![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0801.png)


#### Redis位图常见命令
+ setbit key index val 
    - val只能是0/1
+ getbit key index
+ bitcount key  [start end] 
   - bitcount 用来统计指定位置范围内 1 的个数
+ bitpos   key  value  [start end] 
    - 用来查找指定范围内出现的第一个 0 或 1,这里的位置值代表字符顺序,所以代表着只能查询位为8的位置
    - bitpos key 1 1 1  # 从第二个字符算起，第一个 1 位  
    
位数据设置完后可以直接使用字符串命令来操作位数据
+ get key
+ set key val


#### HyperLogLog
一种轻量级的有误差的set结构指令,适合用在需要去重的精确度不高的统计方案时,会有大概20%的统计误差

常见命令
+ pfadd
+ pfcount
+ pfmerge


#### 布隆过滤器 (Bloom Filter) 
一个不精确的类set结构,可以使用它判断某个对象是否存在  
当布隆过滤器说某个值存在时，这个值可能不存在；当它说不存在时，那就肯定不存在

>常见命令
+ bf.add key val
+ bf.exists key val
+ bf.madd key val1 val2 
+ bf.mexists key val1 val2

>布隆过滤器的原理  

布隆过滤器对应到Redis的数据结构里面就是一个大型的位数组和几个不一样的无偏hash函数。所谓无偏就是能够把元素的hash值算得比较均匀  

![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0802.png)

添加时  

向布隆过滤器中添加key时，会使用多个hash函数对key进行hash算得一个整数索引值然后对位数组长度进行取模运算得到一个位置，     
每个hash函数都会算得一个不同的位置。 再把位数组的这几个位置都置为1就完成了add操作。


判断时  
向布隆过滤器询问key是否存在时，跟add一样，也会把hash的几个位置都算出来，看看位数组中这几个位置是否都位1，  
只要有一个位为0，那么说明布隆过滤器中这个key不存在。如果都是1，这并不能说明这个key 就一定存在，只是极有可能存在，  
因为这些位被置为1可能是因为其它的key存在所致。如果这个位数组比较稀疏，判断正确的概率就会很大，如果这个位数组比较拥挤，判断正确的概率就会降低.


#### 简单限流
>原理
使用zset计算在滑动窗口内的元素个数是否超过最大限额数,同一个用户同一种行为对应同一个key,时间戳来做score,val值是一个唯一标志就行  
判断时计算当前时间的前一个时间窗口内的元素个数是否超过最大限额数

```java
public class SimpleRateLimiter {

  private Jedis jedis;

  public SimpleRateLimiter(Jedis jedis) {
    this.jedis = jedis;
  }
  
  /**
   * period 滑动窗口,在滑动窗口时间内最大的操作数为maxCount ,单位s
   */
  public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {
    String key = String.format("hist:%s:%s", userId, actionKey);
    long nowTs = System.currentTimeMillis();
    //使用管道
    Pipeline pipe = jedis.pipelined();
    pipe.multi();
    //添加此次操作,score的值为时间戳用来计算滑动窗口的,val的值唯一标志就行
    pipe.zadd(key, nowTs, "" + nowTs);
    //删除本次滑动窗口外的元素
    pipe.zremrangeByScore(key, 0, nowTs - period * 1000);
    //计算剩余元素个数
    Response<Long> count = pipe.zcard(key);
    //设置这个zset的超时时间为滑动窗口+1秒
    pipe.expire(key, period + 1);
    pipe.exec();
    pipe.close();
    return count.get() <= maxCount;
  }

  public static void main(String[] args) {
    Jedis jedis = new Jedis();
    SimpleRateLimiter limiter = new SimpleRateLimiter(jedis);
    for(int i=0;i<20;i++) {
      System.out.println(limiter.isActionAllowed("laoqian", "reply", 60, 5));
    }
  }

}


```

#### 漏斗限流
顾名思义,通过限制桶的容量,限制访问数量.  
桶会定期的流水,而有行为就会添加水.当水超过桶容量时就会溢出

>jvm级别的漏斗算法
```java
public class FunnelRateLimiter {

  static class Funnel {
    //桶容量
    int capacity;
    //漏水速度
    float leakingRate;
    //剩余容量
    int leftQuota;
    //上次漏水时间
    long leakingTs;

    public Funnel(int capacity, float leakingRate) {
      this.capacity = capacity;
      this.leakingRate = leakingRate;
      this.leftQuota = capacity;
      this.leakingTs = System.currentTimeMillis();
    }

    void makeSpace() {
      long nowTs = System.currentTimeMillis();
      long deltaTs = nowTs - leakingTs;
      //计算此次需要流出的水的容量
      int deltaQuota = (int) (deltaTs * leakingRate);
      if (deltaQuota < 0) { // 间隔时间太长，整数数字过大溢出
        this.leftQuota = capacity;
        this.leakingTs = nowTs;
        return;
      }
      if (deltaQuota < 1) { // 腾出空间太小，最小单位是1
        return;
      }
      //更新剩余容量
      this.leftQuota += deltaQuota;
      //更新上次漏水时间
      this.leakingTs = nowTs;
      if (this.leftQuota > this.capacity) {
        this.leftQuota = this.capacity;
      }
    }

    boolean watering(int quota) {
      //先漏水  
      makeSpace();
      if (this.leftQuota >= quota) {
        this.leftQuota -= quota;
        return true;
       }
       return false;
    }
    
    private Map<String, Funnel> funnels = new HashMap<>();
    
      public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
        String key = String.format("%s:%s", userId, actionKey);
        Funnel funnel = funnels.get(key);
        if (funnel == null) {
          funnel = new Funnel(capacity, leakingRate);
          funnels.put(key, funnel);
        }
        return funnel.watering(1); // 需要1个quota
      }
    }
  }  

```

>REDIS-CELL
  
Redis4.0提供了一个限流Redis模块，它叫 redis-cell。该模块也使用了漏斗算法，并提供了原子的限流指令。  
该模块只有1条指令cl.throttle，它的参数和返回值都略显复杂

```
> cl.throttle laoqian:reply 15 30 60 1
                      ▲     ▲  ▲  ▲  ▲
                      |     |  |  |  └───── need 1 quota (可选参数，默认值也是1)
                      |     |  └──┴─────── 30 operations / 60 seconds 这是漏水速率
                      |     └───────────── 15 capacity 这是漏斗容量
                      └─────────────────── key laoqian...

  
  
> cl.throttle laoqian:reply 15 30 60
1) (integer) 0   # 0 表示允许，1表示拒绝
2) (integer) 15  # 漏斗容量capacity (15 + 1 = 16)
3) (integer) 14  # 漏斗剩余空间left_quota
4) (integer) -1  # 如果拒绝了，需要多长时间后再试(漏斗有空间了，单位秒)
5) (integer) 2   # 多长时间后，漏斗完全空出来(left_quota==capacity，单位秒)...


```


#### SCAN
SCAN cursor [MATCH pattern] [COUNT count]  

REDIS在2.8版本添加了scan命令.scan 相比 keys 具备有以下特点:
+ 复杂度虽然也是 O(n)，但是它是通过游标分步进行的，不会阻塞线程;
+ 提供 limit 参数，可以控制每次返回结果的最大条数，limit 只是一个 hint，返回的结果可多可少;
+ 同 keys 一样，它也提供模式匹配功能;
+ 服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数;
+ 返回的结果可能会有重复，需要客户端去重复，这点非常重要;
+ 遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的;
+ 单次返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零;...

```
 //从0游标开始筛选"key99"开头的key,只遍历1000个
 scan 0 match key99* count 1000
```

>字典结构

在 Redis 中所有的 key 都存储在一个很大的字典中，这个字典的结构和 Java 中的 HashMap 一样，是一维数组 + 二维链表结构，  
第一维数组的大小总是 2^n(n>=0)，扩容一次数组大小空间加倍  

scan 指令返回的游标就是第一维数组的位置索引，我们将这个位置索引称为槽 (slot)。  
如果不考虑字典的扩容缩容，直接按数组下标挨个遍历就行了。  
limit参数就表示需要遍历的槽位数，之所以返回的结果可能多可能少，是因为不是所有的槽位上都会挂接链表，有些槽位可能是空的，还有些槽位上挂接的链表上的元素可能会有多个。  
每一次遍历都会将 limit 数量的槽位上挂接的所有链表元素进行模式匹配过滤后，一次性返回给客户端。...

>字典扩容  

![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0804.png)    

假设当前的字典的数组长度由8位扩容到16位，那么3号槽位011将会被rehash到3号槽位和11号槽位，   
也就是说该槽位链表中大约有一半的元素还是3号槽位，其它的元素会放到11号槽位，  
11这个数字的二进制是1011，就是对3的二进制011增加了一个高位 1。


>scan遍历顺序  

`高位进位加法来遍历` : 高位进位法从左边加，进位往右边移动  (`还有这种操作!!!简直赞啊`)
之所以使用这样特殊的方式进行遍历，是考虑到字典的扩容和缩容时避免槽位的遍历重复和遗漏。   
 

![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0803.png)  

观察这张图，我们发现采用高位进位加法的遍历顺序，rehash 后的槽位在遍历顺序上是相邻的。

假设当前要即将遍历110这个位置(橙色)，那么扩容后，当前槽位上所有的元素对应的新槽位是 0110和1110(深绿色)，也就是在槽位的二进制数增加一个高位0或1。  
这时我们可以直接从 0110 这个槽位开始往后继续遍历，0110 槽位之前的所有槽位都是已经遍历过的，这样就可以避免扩容后对已经遍历过的槽位进行重复遍历。

再考虑缩容，假设当前即将遍历110这个位置(橙色)，那么缩容后，当前槽位所有的元素对应的新槽位是10(深绿色)，也就是去掉槽位二进制最高位。  
这时我们可以直接从 10 这个槽位继续往后遍历，10 槽位之前的所有槽位都是已经遍历过的，这样就可以避免缩容的重复遍历。  
不过缩容还是不太一样，它会对图中010 这个槽位上的元素进行重复遍历，因为缩融后 10 槽位的元素是 010 和 110 上挂接的元素的融合。...

>更多scan指令
+ zscan
+ hscan
+ sscan

>扫描大key  

redis-cli -h 127.0.0.1 -p 7001 --bigkeys -i 0.1  //-i表示每隔 100 条 scan 指令就会休眠 0.1s



#### 持久化

快照RDB原理  
由于redis是单线程机制,如果使用唯一的线程来进行备份写IO会严重影响性能.  
所以redis父进程fork一个子进程来对内存数据进行序列化.父子进程共享一份redis内存数据.  
而在fork完后的其它数据请求,redis会使用操作系统的"COPY ON WRITE"机制,父进程在复制分离出来的数据段进行操作,而子进程在原来的数据段进行操作(对子进程来讲数据从复制那一刻就不变的,所以才有快照的说法) 

AOF重写原理  
redis进程fork一个子进程来对内存中的数据进行遍历转换成一系列的redis指令,序列化到一个新的AOF文件..
然后再将重写期间的指令拼凑到这个新文件上,然后替换掉旧AOF文件,完成重写任务  

fsnyc--强制刷新内存缓存到磁盘  
程序对AOF日志文件进行写操作时，实际上是将内容写到了内核为文件描述符分配的一个内存缓存中，然后内核会异步将脏数据刷回到磁盘的  
为防止内存期间的指令因服务器宕机而丢失,redis提供了三种机制来减少这个丢失 :
+ appendfsync always
+ appendfsync no 这个永不 是根据系统刷新输出缓冲区的时间来决定的，一般来说是30s
+ appendfsync everysec 【默认的】


运维  

快照是通过开启子进程的方式进行的，遍历整个内存，大块写磁盘会加重系统负载.它是一个比较耗资源的操作。  
AOF的fsync是一个耗时的IO操作，它会降低Redis性能，同时也会增加系统IO负担  
所以通常 Redis 的主节点是不会进行持久化操作，持久化操作主要在从节点进行


#### 管道
管道的本质是客户端批量发送多次指令,而不是服务端一次处理多条指令.
管道只是把本来应该 `写 -> 读 -> 写 -> 读` 的多次网络调用顺序转换为 `写 -> 写 -> 读 -> 读`的一次网络调用.  
服务器根本没有任何区别对待，还是收到一条消息，执行一条消息，回复一条消息的正常的流程  
客户端通过对管道中的指令列表改变读写顺序就可以大幅节省IO时间

redis的自带压测工具 --> `redis-benchmark`
首先我们对一个普通的 set 指令进行压测，QPS 大约 5w/s。
```
> redis-benchmark -t set -q
SET: 51975.05 requests per second
```
我们加入管道选项-P参数，它表示单个管道内并行的请求数量，看下面P=2，QPS 达到了 9w/s。
```
> redis-benchmark -t set -P 2 -q
SET: 91240.88 requests per second
```

再看看P=3，QPS 达到了 10w/s。
```
> redis-benchmark -t set -P 3 -q
SET: 102354.15 requests per second...
```

>redis请求交互流程图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0805.png)  

流程解释  
 I.发送指令 : redis在客户端是先写内核缓冲,操作系统将缓冲内容经网卡经网络发送到服务端网卡,服务端网卡将数据发送到内核缓存,redis服务进程将数据从缓冲读出 ..
 II.接收指令 : redis服务端将响应数据发送至内核缓冲,操作系统将内容经网卡经网络发送至redis客户端网卡再转到内核缓冲中,redis客户端再从内核缓冲读取响应内容.  

总结  
 可以看出客户端和服务端是顺序相反但相同的流程.而且可以分析得出两边的写操作都是很快就到内核然后完成,而读取操作需经网络,所以基本耗时都是在读操作.  
 所以管道的本质就是将能快速完成的写命令先完成,而后将这些指令打包经由一次网络传送到服务端,服务端返回也是这个道理.  
   
   
#### 事务
指令
+ multi 事务的开始
+ exec  事务的执行
+ discard  事务的抛弃  

jedis代码实例
![redis](http://group.store.qq.com/qun/V10dOswR3Unu71/V3tNZ1gH5N941mkvtkm/800?w5=1162&h5=640&rf=viewer_421)

redis事务的交互图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0806.png)  

redis事务的串行化  
redis的事务不保证原子性,只保证隔离性(当前执行的事务有着不被其它事务打断的权利)

优化  
事务若单独使用会产生多次网络IO消耗,故事务一般结合管道使用
只使用管道不使用事务,无法保证事务执行过程的隔离性.  
只使用事务不使用管道,没充分减少网络IO消耗.  

Watch命令 -- 一个使用Watch使用乐观锁的例子  
```java
import java.util.List;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;

public class TransactionDemo {

  public static void main(String[] args) {
    Jedis jedis = new Jedis();
    String userId = "abc";
    String key = keyFor(userId);
    jedis.setnx(key, String.valueOf(5));  //setnx 做初始化
    System.out.println(doubleAccount(jedis, userId));
    jedis.close();
  }

  public static int doubleAccount(Jedis jedis, String userId) {
    String key = keyFor(userId);
    while (true) {
      jedis.watch(key);
      int value = Integer.parseInt(jedis.get(key));
      value *= 2; // 加倍
      //watch必须配合事务
      Transaction tx = jedis.multi();
      tx.set(key, String.valueOf(value));
      List<Object> res = tx.exec();
      if (res != null) {
        break; // 成功了
      }
    }
    return Integer.parseInt(jedis.get(key)); // 重新获取余额
  }

  public static String keyFor(String userId) {
    return String.format("account_%s", userId);
  }

}
```


#### 压缩
redis对内部小的集合数据结构都使用ziplist来压缩存储.  
Redis 的 intset 是一个紧凑的整数数组结构，它用于存放元素都是整数的并且元素个数较少的set集合。  

ziplist的数据结构  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis0807.png)  
+ 如果它存储的是 hash 结构，那么 key 和 value 会作为两个 entry 相邻存在一起  
+ 如果它存储的是 zset，那么 value 和 score 会作为两个 entry 相邻存在一起

Redis 规定在小对象存储结构的限制条件如下：  
```
hash-max-ziplist-entries 512  # hash 的元素个数超过 512 就必须用标准结构存储
hash-max-ziplist-value 64  # hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储
list-max-ziplist-entries 512  # list 的元素个数超过 512 就必须用标准结构存储
list-max-ziplist-value 64  # list 的任意元素的长度超过 64 就必须用标准结构存储
zset-max-ziplist-entries 128  # zset 的元素个数超过 128 就必须用标准结构存储
zset-max-ziplist-value 64  # zset 的任意元素的长度超过 64 就必须用标准结构存储
set-max-intset-entries 512  # set 的整数元素个数超过 512 就必须用标准结构存储
```

#### 主从
复制  
Redis 的主从数据是异步同步的，所以分布式的 Redis 系统并不满足「一致性」要求。  
当客户端在 Redis 的主节点修改了数据后，立即返回，即使在主从网络断开的情况下，主节点依旧可以正常对外提供修改服务，所以 Redis 满足「可用性」


主从版本升级注意  
要求从库至少和主库一样新，否则主库的新指令同步过去从库不能识别，同步就会出错，所以升级版本时应该先升级从库，再升级主库

#### 集群的异同
+ 不支持事务
+ mget方法比单redis实例要慢的多,因为要拆分为多个mget
+ rename不再是原子方法(需将数据从原节点转移到目标节点)


#### Stream结构
redis5.0的最大的新特性就是多了一个Stream数据结构    
它是一个新的强大的支持多播的可持久化的消息队列，作者坦言Stream狠狠地借鉴了Kafka的设计

![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis081601.png)    
Redis Stream 的结构如上图所示，它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的ID和对应的内容。消息是持久化的，Redis重启后，内容还在

结构  
消息 ID -- timestampInMillis-sequence  
内容    -- 消息内容就是键值对，形如 hash 结构的键值对

指令  
+ xadd
+ xdel
+ xrange
+ xlen
+ del

消费者组  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis081602.png)    
  
  
#### INFO指令
```
# 获取所有信息
> info 

 
# 获取内存相关信息
> info memory 
 
# 获取复制相关信息
> info replication
 
#Redis 每秒执行多少次指令
> redis-cli info stats |grep ops
instantaneous_ops_per_sec:789
 
#Redis 连接了多少客户端
> redis-cli info clients
# Clients
connected_clients:124  # 这个就是正在连接的客户端数量
  
  
#列出所有的客户端链接地址来确定源头
> redis-cli client list
 
#Redis 内存占用多大
> redis-cli info memory | grep used | grep human
used_memory_human:827.46K # 内存分配器 (jemalloc) 从操作系统分配的内存总量
used_memory_rss_human:3.61M  # 操作系统看到的内存占用 ,top 命令看到的内存
used_memory_peak_human:829.41K  # Redis 内存消耗的峰值
used_memory_lua_human:37.00K # lua 脚本引擎占用的内存大小



```

```
#monitor 指令快速观察一下究竟是哪些 key 访问比较频繁，从而在相应的业务上进行优化，以减少 IO 次数#
> redis-cli monitor
```


复制积压缓冲区大小非常重要，它严重影响到主从复制的效率。  
当从库因为网络原因临时断开了主库的复制，然后网络恢复了，又重新连上的时候，  
这段断开的时间内发生在 master 上的修改操作指令都会放在积压缓冲区中，这样从库可以通过积压缓冲区恢复中断的主从同步过程。...


积压缓冲区是环形的，后来的指令会覆盖掉前面的内容。  
如果从库断开的时间过长，或者缓冲区的大小设置的太小，都会导致从库无法快速恢复中断的主从同步过程，因为中间的修改指令被覆盖掉了。  
这时候从库就会进行全量同步模式，非常耗费 CPU 和网络资源。

```
#复制积压缓冲区多大
> redis-cli info replication |grep backlog
repl_backlog_active:0
repl_backlog_size:1048576  # 这个就是积压缓冲区大小
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```


#### 过期策略
+ 定时清理(集中处理过期key)
+ 惰性删除(访问时判断是否过期)

定时扫描策略    

Redis 默认会每秒进行十次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略。
+ 从过期字典中随机 20 个 key；
+ 删除这 20 个 key 中已经过期的 key；
+ 如果过期的 key 比率超过 1/4，那就重复步骤 1；

所以常见做法是在固定过期时间基础上加上一个随机范围时间值.确保key不在同一时间失效


从库的过期策略  
从库不会进行过期扫描，从库对过期的处理是被动的。  
主库在 key 到期时，会在 AOF 文件里增加一条del指令，同步到所有的从库，从库通过执行这条del指令来删除过期的key