## Redis的特性

+ 管道
+ 事务
+ 小对象压缩

### 管道

**定义**

​	Redis 管道技术可以在服务端未响应时，客户端可以继续向服务端发送请求，并最终一次性读取所有服务端的响应。

**特点**

	+ 管道不保证原子性
	+ 服务端依旧正常处理指令,只是客户端通过对管道中的指令列表改变读写顺序就可以大幅节省 IO 时间

**原理**

- 管道通过减少客户端与redis的通信次数来实现降低往返延时时间

- 改变原本一发一收的消息模式,优化为可以批量发送消息,一次读取服务端所有响应消息的模式

  - ![https://ws3.sinaimg.cn/large/005BYqpggy1g2ygx2p559j30h00feta1.jpg](https://s2.ax1x.com/2019/05/12/Eh1YNj.png)

- 内核读写原理分析

  - ![https://ws3.sinaimg.cn/large/005BYqpggy1g2ygxyhfwjj30v00fh78d.jpg](https://s2.ax1x.com/2019/05/12/Eh1a3q.png)
  - 客户端的写耗时主要在于缓冲区满时清空发送缓冲的等待,若发送缓冲未满则直接发送到缓存速度快
  - 客户端的读超时主要是等待redis的服务端数据发送到接收缓冲,包括客户端的读缓冲区的等待数据,网络传输等

  **所以对于管道来说，连续的`write`操作根本就没有耗时，之后第一个`read`操作会等待一个网络的来回开销，然后所有的响应消息就都已经回送到内核的读缓冲了，后续的 `read` 操作直接就可以从缓冲拿到结果，瞬间就返回了**

  ------

  

  ### 事务

  **指令**

  - multi
  - exec
  - discard

  

  **特点**

  + 隔离性 -- 隔离性中的串行化——当前执行的事务有着不被其它事务打断的权利

  + 假原子性-- 事务中连续多个的命令.其中一两个的失败不影响其他命令的执行.因此无法满足原子性的要么全部成功,要么全部失败特点

  **原理**

  - 所有的指令在 exec 之前不执行，而是缓存在服务器的一个事务队列中，服务器一旦收到 exec 指令，才开执行整个事务队列，执行完毕后一次性返回所有指令的运行结果
  - 因为 Redis 的单线程特性，它不用担心自己在执行队列的时候被其它指令打搅，可以保证他们能得到的「原子性」执行

  **优化**

     一般事务都伴随着多个指令,此时使用管道能减少客户端与redis服务端的网络交互次数,提升效率

  **交互流程**

  ![https://ws3.sinaimg.cn/large/005BYqpggy1g2yhb84sddj30mi0aqgmh.jpg](https://s2.ax1x.com/2019/05/12/Eh3Dzt.png)

------

### 小对象压缩

#### 结构类型

**ziplist**

- 使用场景 - Redis 内部管理的集合数据结构很小，它会使用紧凑存储形式压缩存储

- 数据结构图

  ![https://ws3.sinaimg.cn/large/005BYqpggy1g2yhgw3t2lj30ng08lt9t.jpg](https://s2.ax1x.com/2019/05/12/Eh3TyV.png)

   hash结构 -- key 和 value 会作为两个 entry 相邻存在一起

   zset结构 -- value 和 score 会作为两个 entry 相邻存在一起

- 判断数据结构的命令

  ​	object encoding key

**intset**

定义

​	Redis 的 intset 是一个紧凑的整数数组结构，它用于存放元素都是整数的并且元素个数较少的 set 集合

数据结构图

![https://ws3.sinaimg.cn/large/005BYqpggy1g2yhi0efo4j30km05rq3h.jpg](https://s2.ax1x.com/2019/05/12/Eh3jY9.png)



#### 使用小对象的限制条件

- 总结
  - 任意key/val的长度必须在64内才能使用小对象压缩
  - 任意非排序元素的元素个数在512内可使用小对象
  - 需排序的zset元素在128内才能使用小对象
- 配置
  - hash-max-ziplist-entries 512 # hash 的元素个数超过 512 就必须用标准结构存储
  - hash-max-ziplist-value 64 # hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储
  - list-max-ziplist-entries 512 # list 的元素个数超过 512 就必须用标准结构存储
  - list-max-ziplist-value 64 # list 的任意元素的长度超过 64 就必须用标准结构存储
  - zset-max-ziplist-entries 128 # zset 的元素个数超过 128 就必须用标准结构存储
  - zset-max-ziplist-value 64 # zset 的任意元素的长度超过 64 就必须用标准结构存储
  - set-max-intset-entries 512 # set 的整数元素个数超过 512 就必须用标准结构存储