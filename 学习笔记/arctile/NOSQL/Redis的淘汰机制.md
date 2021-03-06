## Redis的淘汰机制

#### 背景

如果redis的内存超出了服务器的物理内存时,内存的数据会开始和磁盘产生频繁的交换 (swap)。交换会让 Redis 的性能急剧下降，对于访问量比较频繁的 Redis 来说，这样龟速的存取效率基本上等于不可用。



#### 解决方案

在生产环境中我们是不允许 Redis 出现交换行为的，为了限制最大使用内存，Redis 提供了配置参数 `maxmemory` 来限制内存超出期望大小。

当实际内存超出 `maxmemory` 时，Redis 提供了几种可选策略 (**maxmemory-policy**) 来让用户自己决定该如何腾出新的空间以继续提供读写服务。

`maxmemory-policy`策略

+ **noeviction**   不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行。这是**默认**的淘汰策略。

+ **volatile-lru** 尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失。

+ **volatile-ttl** 跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 的值，ttl 越小越优先被淘汰。

+ **volatile-random** 跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key。

+ **allkeys-lru** 区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰。

+ **allkeys-random** 跟上面一样，不过淘汰的策略是随机的 key。

LRU(least recently used)算法 - 最近最少使用.



#### Redis的`近似LRU算法`

由于实现 LRU 算法除了需要 key/value 字典外，还需要附加一个链表，链表中的元素按照一定的顺序进行排列。而这需要增加大量额外的内存以及对redis现有结构进行较大改造.所以redis作者使用了一种`近似LRU`算法来实现淘汰机制.

Redis 为实现近似 LRU 算法，它给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。然后在实现写操作时若发现内存超过最大限制内存,则随机采样5个key将其中最旧的淘汰,依次循环直至内存小于最大内存限制.



#### 参考链接

<https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5b4c5158f265da0fa50a0b17>