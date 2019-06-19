Redis 深度历险：核心原理与应用实践
-

###  字符串
Redis 中的字符串是可以修改的字符串，在内存中它是以字节数组的形式存在的。
```
//伪代码
struct SDS<T> {
  T capacity; // 数组容量
  T len; // 数组长度
  byte flags; // 特殊标识位，不理睬它
  byte[] content; // 数组内容
}
```

Redis 的字符串有两种存储方式，在长度特别短时，使用 emb 形式存储 (embeded)，当长度超过 44 时，使用 raw 形式存储。  

##### Redis 对象头结构体
所有的 Redis 对象都有下面的这个结构头,这样一个 RedisObject 对象头需要占据 16 字节的存储空间。
```
struct RedisObject {
    int4 type; // 4bits
    int4 encoding; // 4bits
    //为了记录对象的 LRU 信息，使用了 24 个 bit 来记录 LRU 信息
    int24 lru; // 24bits
    //每个对象都有个引用计数，当引用计数为零时，对象就会被销毁，内存被回收
    int32 refcount; // 4bytes 
    //ptr 指针将指向对象内容 (body) 的具体存储位置
    void *ptr; // 8bytes，64-bit system
} robj;
```

接着我们再看 SDS 结构体的大小，在字符串比较小时，SDS 对象头的大小是capacity+3，至少是 3。  
意味着分配一个字符串的最小空间占用为 19 字节 (16+3)。
```
struct SDS {
    int8 capacity; // 1byte
    int8 len; // 1byte
    int8 flags; // 1byte
    byte[] content; // 内联数组，长度为 capacity
}
```

SDS 结构体中的 content 中的字符串是以字节\0结尾的字符串，  
之所以多出这样一个字节，是为了便于直接使用 glibc 的字符串处理函数，以及为了便于字符串的调试打印输出。  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082701.png) 
 
##### 扩容策略
 字符串在长度小于 1M 之前，扩容空间采用加倍策略，也就是保留 100% 的冗余空间。  
 当长度超过 1M 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只会多分配 1M 大小的冗余空间。
 
##### 注意
第一次创建字符串是木有冗余空间的,即上面的capacity是等于len的.  
执行append之后开始冗余，因为平时的应用中大多数字符串只有只读的需求，一旦遇到append指令意味着它是需要支持修改的，于是才给分配了冗余空间


### 字典Dict
dict 是 Redis 服务器中出现最为频繁的复合型数据结构    

运用举例
+ hash 结构的数据会用到字典
+ 整个 Redis 数据库的所有 key 和 value 也组成了一个全局字典  
+ 带过期时间的 key 集合也是一个字典  
+ zset 集合中存储 value 和 score 值的映射关系也是通过 dict 结构实现的

```
struct RedisDb {
    dict* dict; // all keys  key=>value
    dict* expires; // all expired keys key=>long(timestamp)
    ...
}

struct zset {
    dict *dict; // all values  value=>score
    zskiplist *zsl;
}...

```

dict 结构内部包含两个 hashtable，通常情况下只有一个 hashtable 是有值的
但是在 dict 扩容缩容时，需要分配新的 hashtable，然后进行渐进式搬迁，这时候两个 hashtable 存储的分别是旧的 hashtable 和新的 hashtable。  
待搬迁结束后，旧的 hashtable 被删除，新的 hashtable 取而代之.
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082702.png)  
 
所以，字典数据结构的精华就落在了 hashtable 结构上了。hashtable 的结构和 Java 的 HashMap 几乎是一样的，都是通过分桶的方式解决 hash 冲突。  
第一维是数组，第二维是链表。数组中存储的是第二维链表的第一个元素的指针。

```
struct dictEntry {
    void* key;
    void* val;
    dictEntry* next; // 链接下一个 entry
}
struct dictht {
    dictEntry** table; // 二维
    long size; // 第一维数组的长度
    long used; // hash 表中的元素个数
    ...
}

```

结构图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082703.png)

##### 渐进式rehash
大字典的扩容是比较耗时间的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面   
这是一个O(n)级别的操作，作为单线程的Redis表示很难承受这样耗时的过程,所以Redis使用渐进式rehash小步搬迁  
  
  
###  压缩列表ziplist
Redis 为了节约内存空间使用，zset 和 hash 容器对象在元素个数较少的时候，采用压缩列表 (ziplist) 进行存储。  
压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙。    
压缩列表为了支持双向遍历，所以才会有 ztail_offset 这个字段，用来快速定位到最后一个元素，然后倒着遍历。

```
struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}

struct entry {
    int<var> prevlen; // 前一个 entry 的字节长度
    int<var> encoding; // 元素类型编码
    optional byte[] content; // 元素内容
}
```

结构图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082801.png)  

##### 增加元素
因为 ziplist 都是紧凑存储，没有冗余空间 (对比一下 Redis 的字符串结构)。  
意味着插入一个新的元素就需要调用 realloc 扩展内存。  
取决于内存分配器算法和当前的ziplist内存大小，realloc可能会重新分配新的内存空间，并将之前的内容一次性拷贝到新的地址，  
也可能在原有的地址上进行扩展，这时就不需要进行旧内容的内存拷贝。

如果 ziplist 占据内存太大，重新分配内存和拷贝内存就会有很大的消耗。  
所以 ziplist 不适合存储大型字符串，存储的元素也不宜过多。

##### IntSet 
当 set 集合容纳的元素都是整数并且元素个数较小时，Redis 会使用 intset 来存储结合元素。  
intset 是紧凑的数组结构，同时支持 16 位、32 位和 64 位整数。
```
struct intset<T> {
    int32 encoding; // 决定整数位宽是 16 位、32 位还是 64 位
    int32 length; // 元素个数
    int<T> contents; // 整数数组，可以是 16 位、32 位和 64 位
}
```
结构图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082802.png)  


### 快速列表
Redis 早期版本存储 list 列表数据结构使用的是压缩列表 ziplist 和普通的双向链表 linkedlist，  
也就是元素少时用 ziplist，元素多时用 linkedlist。  

考虑到链表的附加空间相对太高，prev 和 next 指针就要占去 16 个字节 (64bit 系统的指针是 8 个字节)，  
另外每个节点的内存都是单独分配，会加剧内存的碎片化，影响内存管理效率。  
后续版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist。...

quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。

结构图  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis082803.png)    

```
struct ziplist {
    ...
}
struct ziplist_compressed {
    int32 size;
    byte[] compressed_data;
}
struct quicklistNode {
    quicklistNode* prev;
    quicklistNode* next;
    ziplist* zl; // 指向压缩列表
    int32 size; // ziplist 的字节总数
    int16 count; // ziplist 中的元素数量
    int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
    ...
}
struct quicklist {
    quicklistNode* head;
    quicklistNode* tail;
    long count; // 元素总数
    int nodes; // ziplist 节点的个数
    int compressDepth; // LZF 算法压缩深度
    ...
}
```

为了进一步节约空间，Redis 还会对 ziplist 进行压缩存储，使用 LZF 算法压缩，可以选择压缩深度。  
quicklist 默认的压缩深度是 0，也就是不压缩。压缩的实际深度由配置参数list-compress-depth决定。  
为了支持快速的 push/pop 操作，quicklist 的首尾两个 ziplist 不压缩，此时深度就是 1。  
如果深度为 2，就表示 quicklist 的首尾第一个 ziplist 以及首尾第二个 ziplist 都不压缩。

### zset的跳跃列表
Redis 的 zset 是一个复合结构，一方面它需要一个 hash 结构来存储 value 和 score 的对应关系，  
另一方面需要提供按照 score 来排序的功能，还需要能够指定 score 的范围来获取 value 列表的功能，这就需要另外一个结构「跳跃列表」。    
zset 的内部实现是一个 hash 字典加一个跳跃列表 (skiplist)。


### 紧凑列表-listpack
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis090101.png)   


结构
```
struct listpack<T> {
    int32 total_bytes; // 占用的总字节数
    int16 size; // 元素个数
    T[] entries; // 紧凑排列的元素列表
    int8 end; // 同 zlend 一样，恒为 0xFF
}

struct lpentry {
    int<var> encoding;
    optional byte[] content;
    int<var> length;
}
```

首先这个 listpack 跟 ziplist 的结构几乎一摸一样，只是少了一个zltail_offset字段。   
ziplist 通过这个字段来定位出最后一个元素的位置，用于逆序遍历。  
不过 listpack 可以通过其它方式来定位出最后一个元素的位置，所以zltail_offset字段就省掉了。  

元素的结构和 ziplist 的元素结构也很类似，都是包含三个字段。   
不同的是长度字段放在了元素的尾部，而且存储的不是上一个元素的长度，是当前元素的长度。   
正是因为长度放在了尾部，所以可以省去了zltail_offset字段来标记最后一个元素的位置，这个位置可以通过total_bytes字段和最后一个元素的长度字段计算出来。


##### 级联更新问题
listpack的设计彻底消灭了ziplist存在的级联更新行为，元素与元素之间完全独立，不会因为一个元素的长度变长就导致后续的元素内容会受到影响



### LFU vs LRU
redis4.0之前使用LRU模式淘汰冷数据,在4.0之后使用LFU淘汰冷数据    


淘汰算法
+ LRU(Least Recently Used) 最近最少使用.表示按照上一次的访问时间进行淘汰
+ LFU(least frequently used) 最不经常使用,表示按最近的访问频率进行淘汰，它比 LRU 更加精准地表示了一个 key 被访问的热度。   

redis对象结构头储存了lru字段记录对象的访问热度  
```
// redis 的对象头
typedef struct redisObject {
    unsigned type:4; // 对象类型如 zset/set/hash 等等
    unsigned encoding:4; // 对象编码如 ziplist/intset/skiplist 等等
    unsigned lru:24; // 对象的「热度」
    int refcount; // 引用计数
    void *ptr; // 对象的 body
} robj;
```  

##### LRU
在 LRU模式下，lru字段存储的是Redis时钟server.lruclock，Redis时钟是一个24bit的整数，默认是Unix时间戳对 2^24 取模的结果，大约97天清零一次。  
当某个 key 被访问一次，它的对象头的 lru 字段值就会被更新为server.lruclock。  

如果server.lruclock没有折返 (对 2^24 取模)，它就是一直递增的，这意味着对象的 LRU 字段不会超过server.lruclock的值。  
如果超过了，说明server.lruclock折返了。通过这个逻辑就可以精准计算出对象多长时间没有被访问——对象的空闲时间。  

计算规则  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis090102.png)   
  
  
##### LFU
在 LFU 模式下，lru 字段 24 个 bit 用来存储两个值，分别是ldt(last decrement time)和logc(logistic counter)。  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis090103.png)  

     
logc 是 8 个 bit，用来存储访问频次，因为 8 个 bit 能表示的最大整数值为 255，存储频次肯定远远不够，  
所以这 8 个 bit 存储的是频次的对数值，并且这个值还会随时间衰减。如果它的值比较小，那么就很容易被回收。  
为了确保新创建的对象不被回收，新对象的这 8 个 bit 会初始化为一个大于零的值，默认是LFU_INIT_VAL=5.  
     
计算空闲时间(单位分钟)  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis090104.png)  
     
更新ldt的时机  
+ LRU : 在每次访问时更新更新lru
+ LFU : 在redis进行淘汰逻辑时进行

如何打开LFU  

Redis 4.0 给淘汰策略配置参数maxmemory-policy增加了 2 个选项，分别是 volatile-lfu 和 allkeys-lfu，  
分别是对带过期时间的 key 进行 lfu 淘汰以及对所有的 key 执行 lfu 淘汰算法。



###  总览

Redis 的对象很多，但是对象的种类却是有限的，目前一共只有7种对象.  
HyperLogLog 和 Bitmap 一样，使用的是一个普通的动态字符串，而 Geo 使用的是 zset.
```
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
```

Type和Encoding的对应关系如下  
```
1. string  ==> raw|embstr|int
2. list ==> quicklist
3. hash ==> ziplist|hashtable
4. set ==> intset|hashtable
5. zset ==> ziplist|skiplist
6. stream => stream
```

