## Kafka常见配置参数汇总

####  服务端

- unclean.leader.election.enable：是否允许 Unclean Leader 选举。 -- 推荐配置为false,即不允许

- message.max.bytes：控制 Broker 能够接收的最大消息大小。

- replica.fetch.max.bytes : 副本复制时的最大消息大小

- min.insync.replicas : 设置多少个副本写入成功才能成功写入消息 -- 一般将此值设置为大于1

- num.replica.fetchers -- 表示的是 Follower 副本用多少个线程来拉取消息，默认使用 1 个线程。

-  bootstrap.servers : 指定了这个 Producer 启动时要连接的 Broker 地址

  > 通常你指定 3～4 台就足以了。因为 Producer 一旦连接到集群中的任一台 Broker，就能拿到整个集群的 Broker 信息

  



#### 生产者

- ack -- 用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的。

  - acks = 1 - 默认值即为1。生产者发送消息之后，只要分区的 leader 副本成功写入消息，那么它就会收到来自服务端的成功响应。
  - acks = 0 - 生产者发送消息之后不需要等待任何服务端的响应。
  - acks = -1 或 acks = all。- 生产者在消息发送之后，需要等待 ISR 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。

- max.in.flight.requests.per.connection配置为1 - 在需保证消息有序的情况下需设置为1

  > 由于sender线程是会缓存未应答的消息的,如果此值大于1,则如果第一批次消息写入失败，而第二批次消息写入成功，那么生产者会重试发送第一批次的消息，此时如果第一批次的消息写入成功，那么这两个批次的消息就出现了错序。

- 调优吞吐量

  - batch.size -- 消息批次的大小,默认的 16KB 的消息批次大小一般都不适用于生产环境
    可调大到256K或512K,基于你的系统的每秒发送量来评估,
  - [linger.ms](http://linger.ms) -- 批次缓存时间,这个设置默认为0，即没有延迟。
    在希望调大吞吐量的情况下,可调大此值,[如设定linger.ms](http://如设定linger.ms)=5，当消息量比较小时会增加5ms的延迟。

-  compression.type : gzip -- 开启消息压缩算法 --一般可先不必进行消息压缩

  > 具体来说就是用 CPU 时间去换磁盘空间或网络 I/O 传输量，希望以较小的 CPU 开销带来更少的磁盘占用或更少的网络 I/O 传输。每个压缩过的消息集合在 Broker 端写入时都要发生解压缩操作，目的就是为了对消息执行各种验证。对kafka服务端有一定影响

#### 消费者

- fetch.message.max.bytes : 消费消息时的最大消息大小

- max.poll.records : 用来配置 Consumer 在一次拉取请求中拉取的最大消息数，默认值为500（条）

- [heartbeat.interval.ms](http://heartbeat.interval.ms) : 心跳到消费者协调器之间的预计时间,默认为3000

- [session.timeout.ms](http://session.timeout.ms) : 检测消费者是否失效的超时时间,默认为10000

- [max.poll.interval.ms](http://max.poll.interval.ms) : 两次poll操作的最大间隔时间,默认为300000

- auto.offset.reset : 从分区的什么时候开始拉取数据,默认为latest,其它可选值为“earliest”,“none”

- enable.auto.commit : 是否开启自动提交消费位移的功能,默认为true

- fetch.min.bytes : 默认是 1 字节，表示只要 Kafka Broker 端积攒了 1 字节的数据，就可以返回给 Consumer 端

  > 在需增加吞吐量的情况下,可适当调大让kafka多返回一些数据

- [fetch.max.wait.ms](http://fetch.max.wait.ms) : 用于指定 Kafka 的等待时间，默认值为500（ms）

  > 如果 Kafka 中没有足够多的消息而满足不了 fetch.min.bytes 参数的要求，那么最终会等待500ms。