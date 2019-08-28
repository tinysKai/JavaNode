## Zookeeper的ZAB协议

## 简介

在做分布式系统时，我们常常需要维护管理集群的配置信息、服务的注册发现、共享锁等功能，而 ZooKeeper 正是解决这些问题的一把好手。ZAB(ZooKeeper Atomic Broadcast) 则是为 ZooKeeper 设计的一种支持崩溃恢复的原子广播协议。

在看 ZAB 之前我们先复习一下两阶段提交协议

## 两阶段提交



![img](https://user-gold-cdn.xitu.io/2018/8/28/1657ee237f6a347d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



两阶段提交顾名思义主要分为两个阶段

### 第一阶段（请求阶段）

协调者首先会发送某个事务的执行请求给其它所有的参与者，当参与者收到 perpare 请求时会检查自身并告诉协调者自己的决策是同意还是取消

### 第二阶段（提交阶段）

协调者将根据第一阶段的投票结果发送提交或回滚请求（一般是所有参与者都返回同意就发送提交请求，否则发送回滚请求）。

当然两阶段提交协议并不完美，而且存在数据不一致、同步阻塞、单点等问题，这里不在本文的讨论范围

## 协议介绍

好了，复习完两阶段提交协议，接下来我们继续来分析 ZAB 协议。

很多人会误以为 ZAB 协议是 Paxos 的一种特殊实现，事实上他们是两种不同的协议。ZAB 和 Paxos 最大的不同是，ZAB 主要是为分布式主备系统设计的，而 Paxos 的实现是一致性状态机 (state machine replication)

尽管 ZAB 不是 Paxos 的实现，但是 ZAB 也参考了一些 Paxos 的一些设计思想，比如：

- leader 向 follows 提出提案 (proposal)
- leader 需要在达到法定数量 (半数以上) 的 follows 确认之后才会进行 commit
- 每一个 proposal 都有一个纪元 (epoch) 号，类似于 Paxos 中的选票(ballot)

### ZAB 特性

1. 一致性保证

2. 1. 可靠提交 (Reliable delivery) - 如果一个事务 A 被一个 server 提交(committed) 了，那么它最终一定会被所有的 server 提交
   2. 全局有序 (Total order) - 假设有 A、B 两个事务，有一台 server 先执行 A 再执行 B，那么可以保证所有 server 上 A 始终都被在 B 之前执行
   3. 因果有序 (Causal order) - 如果发送者在事务 A 提交之后再发送 B, 那么 B 必将在 A 之前执行

3. 只要大多数（法定数量）节点启动，系统就行正常运行

4. 当节点下线后重启，它必须保证能恢复到当前正在执行的事务

### ZAB 的具体实现

- ZooKeeper 由 client、server 两部分构成
- client 可以在任何一个 server 节点上进行读操作
- client 可以在任何一个 server 节点上发起写请求，非 leader 节点会把此次写请求转发到 leader 节点上。由 leader 节点执行
- ZooKeeper 使用改编的两阶段提交协议来保证 server 节点的事务一致性

#### ZXID



![img](https://user-gold-cdn.xitu.io/2018/8/28/1657ee237fdbc982?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



ZooKeeper 会为每一个事务生成一个唯一且递增长度为 64 位的 ZXID,ZXID 由两部分组成：低 32 位表示计数器 (counter) 和高 32 位的纪元号(epoch)。epoch 为当前 leader 在成为 leader 的时候生成的，且保证会比前一个 leader 的 epoch 大

实际上当新的 leader 选举成功后，会拿到当前集群中最大的一个 ZXID，并去除这个 ZXID 的 epoch, 并将此 epoch 进行加 1 操作，作为自己的 epoch。

#### 历史队列 (history queue)

每一个 follower 节点都会有一个先进先出（FIFO) 的队列用来存放收到的事务请求，保证执行事务的顺序

可靠提交由 ZAB 的事务一致性协议保证 全局有序由 TCP 协议保证 因果有序由 follower 的历史队列 (history queue) 保证

## ZAB 工作模式

- 广播 (broadcast) 模式
- 恢复 (recovery) 模式

### 广播 (broadcast) 模式



![img](https://user-gold-cdn.xitu.io/2018/8/28/1657ee237f339efa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



1. leader 从客户端收到一个写请求
2. leader 生成一个新的事务并为这个事务生成一个唯一的 ZXID，
3. leader 将这个事务发送给所有的 follows 节点
4. follower 节点将收到的事务请求加入到历史队列 (history queue) 中, 并发送 ack 给 ack 给 leader
5. 当 leader 收到大多数 follower（超过法定数量）的 ack 消息，leader 会发送 commit 请求
6. 当 follower 收到 commit 请求时，会判断该事务的 ZXID 是不是比历史队列中的任何事务的 ZXID 都小，如果是则提交，如果不是则等待比它更小的事务的 commit



![img](https://user-gold-cdn.xitu.io/2018/8/28/1657ee2414525584?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 恢复模式

恢复模式大致可以分为四个阶段

- 选举
- 发现
- 同步
- 广播

恢复过程的步骤大致可分为

1. 当 leader 崩溃后，集群进入选举阶段，开始选举出潜在的新 leader(一般为集群中拥有最大 ZXID 的节点)
2. 进入发现阶段，follower 与潜在的新 leader 进行沟通，如果发现超过法定人数的 follower 同意，则潜在的新 leader 将 epoch 加 1，进入新的纪元。新的 leader 产生
3. 集群间进行数据同步，保证集群中各个节点的事务一致
4. 集群恢复到广播模式，开始接受客户端的写请求

当 leader 在 commit 之后但在发出 commit 消息之前宕机，即只有老 leader 自己 commit 了，而其它 follower 都没有收到 commit 消息 新的 leader 也必须保证这个 proposal 被提交.(新的 leader 会重新发送该 proprosal 的 commit 消息)

当 leader 产生某个 proprosal 之后但在发出消息之前宕机，即只有老 leader 自己有这个 proproal，当老的 leader 重启后 (此时左右 follower), 新的 leader 必须保证老的 leader 必须丢弃这个 proprosal.(新的 leader 会通知上线后的老 leader 截断其 epoch 对应的最后一个 commit 的位置)

#### 参考

[链接](https://juejin.im/entry/5b84d589e51d453885032159)

