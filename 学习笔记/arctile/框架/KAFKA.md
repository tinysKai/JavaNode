# KAFKA概述

##  定义/目标

​	一个分布式流式处理平台，它以高吞吐、可持久化、可水平扩展、支持流数据处理等多种特性而被广泛使用.

kafka现主要扮演三个角色 :

+ 消息系统
+ 存储系统(kafka有消息持久化功能以及多副本机制)
+ 流式处理平台

## kafka体系架构

![https://ws3.sinaimg.cn/large/005BYqpggy1g3g5g5nxz9j30y60hin3p.jpg](https://s2.ax1x.com/2019/05/27/VZT8KJ.png)



## kafka基本概念

+ 主题
+ 分区
+ 偏移量
+ 多副本
+ AR/ISR/OSR
+ 生产者
+ 消费者
+ 消费者组

### 主题&分区

Kafka 中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题，而消费者负责订阅主题并进行消费。

主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题，很多时候也会把分区称为主题分区（Topic-Partition）。同一主题下的不同分区包含的消息是不同的，分区在存储层面可以看作一个可追加的日志（Log）文件，消息在被追加到分区日志文件的时候都会分配一个特定的偏移量（offset）。

### 偏移量

offset 是消息在分区中的唯一标识，Kafka 通过它来保证消息在分区内的顺序性，不过 offset 并不跨越分区，也就是说，Kafka 保证的是分区有序而不是主题有序。

Kafka 中的分区可以分布在不同的服务器（broker）上，以此来提供比单个 broker 更强大的性能。**在创建主题的时候可以通过指定的参数来设置分区的个数，当然也可以在主题创建完成之后去修改分区的数量，通过增加分区的数量可以实现水平扩展。**



### 多副本

同一分区的不同副本中保存的是相同的消息，副本之间是“一主多从”的关系，其中 **leader 副本负责处理读写请求，follower 副本只负责与 leader 副本的消息同步**。副本处于不同的 broker 中，当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供服务。Kafka 通过多副本机制实现了故障的自动转移，当 Kafka 集群中某个 broker 失效时仍然能保证服务可用。

下图展示了一个4个kafka的集群下三个分区以及三个副本(包括一个leader副本)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3g6bui9ukj30z10jt0y4.jpg](https://s2.ax1x.com/2019/05/27/VZH37R.png)

### SR

分区中的所有副本统称为 AR（Assigned Replicas）。

所有与 leader 副本保持一定程度同步的副本（包括 leader 副本在内）组成ISR（In-Sync Replicas），ISR 集合是 AR 集合中的一个子集。默认情况下，当 leader 副本发生故障时，只有在 ISR 集合中的副本才有资格被选举为新的 leader

与 leader 副本同步滞后过多的副本（不包括 leader 副本）组成 OSR（Out-of-Sync Replicas）

**SR中有两个名词,HW与LEO.**

HW 是 High Watermark 的缩写，俗称高水位，它标识了一个特定的消息偏移量（offset），消费者只能拉取到这个 offset 之前的消息。

LEO 是 Log End Offset 的缩写，它标识当前日志文件中下一条待写入消息的 offset,分区 ISR 集合中的每个副本都会维护自身的 LEO，而 ISR 集合中最小的 LEO 即为分区的 HW，对消费者而言只能消费 HW 之前的消息。

![https://ws3.sinaimg.cn/large/005BYqpggy1g3g6v73mcoj30ne09ddgk.jpg](https://s2.ax1x.com/2019/05/27/VZbcZ9.png)

*如上图的同步过程中,leader 副本的 LEO 为5，follower1 的 LEO 为5，follower2 的 LEO 为4，那么当前分区的 HW 取最小值4，此时消费者可以消费到 offset 为0至3之间的消息。*

kafka使用的此ISR复制模式能有效的权衡了数据可靠性和性能之间的关系。

#### 消费者组

逻辑上的概念，是Kafka实现单播和广播两种消息模型的手段。同一个topic的数据，会广播给不同的group；同一个group中的worker，只有一个worker能拿到这个数据。