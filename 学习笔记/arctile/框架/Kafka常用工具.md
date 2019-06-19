## Kafka常用工具

#### Kafka Connect

`Kafka Connect`是一个工具，它为在Kafka和外部数据存储系统之间移动数据提供了一种可靠的且可伸缩的实现方式。`Kafka Connect`可以简单快捷地将数据从Kafka中导入或导出，数据范围涵盖关系型数据库、日志和度量数据、Hadoop 和数据仓库、NoSQL数据存储、搜索索引等。

原理图如下,

![](https://s2.ax1x.com/2019/06/08/VBx9C6.png)



#### Kafka Mirror Maker

Kafka Mirror Maker 是用于在两个集群之间同步数据的一个工具，其实现原理是通过从源集群中消费消息，然后将消息生产到目标集群中，也就是普通的生产和消费消息。

原理图如下,

![](https://s2.ax1x.com/2019/06/08/VBzCon.png)

#### Kafka Streams

Kafka Streams 是一个用于处理和分析数据的客户端库。它先把存储在 Kafka 中的数据进行处理和分析，然后将最终所得的数据结果回写到 Kafka 或发送到外部系统。

Kafka Streams 直接解决了流式处理中的很多问题：

- 毫秒级延迟的逐个事件处理
- 有状态的处理，包括连接（join）和聚合类操作
- 提供了必要的流处理原语，包括高级流处理 DSL 和低级处理器 API。高级流处理 DSL 提供了常用流处理变换操作，低级处理器 API 支持客户端自定义处理器并与状态仓库交互
- 使用类似 DataFlow 的模型对无序数据进行窗口化处理
- 具有快速故障切换的分布式处理和容错能力
- 无停机滚动部署

演示下Kafka自带的单词统计处理流`WordCountDemo`

`WordCountDemo`以硬编码的形式用到了两个主题：`streams-plaintext-input` 和 `streams-wordcount-output`。为了能够使示例程序正常运行，我们需要预先准备好这两个主题.

`WordCountDemo`的作用是从主题 `streams-plaintext-input` 中读取消息，然后对读取的消息执行单词统计，并将结果持续写入主题 `streams-wordcount-output`。

```c
//启动WordCountDemo
[root@node1 kafka_2.11-2.0.0]# bin/kafka-run-class.sh 
     org.apache.kafka.streams.examples.wordcount.WordCountDemo
     
```

在另外两个shell窗口中进行数据的写入与读取

```c
//写入单词数据
[root@node1 kafka_2.11-2.0.0]# bin/kafka-console-producer.sh --broker-list localhost:9092 --topic streams-plaintext-input
>hello kafka streams

//消费者端的单词统计
[root@node1 kafka_2.11-2.0.0]# bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --property print.key=true --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
hello 1
kafka 1
streams 1
```

```c
//写入第二行数据
[root@node1 kafka_2.11-2.0.0]# bin/kafka-console-producer.sh --broker-list localhost:9092 --topic streams-plaintext-input
>hello kafka streams
>I love kafka streams


//消费者端的单词统计
[root@node1 kafka_2.11-2.0.0]# bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --property print.key=true --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
I 1
love 1
kafka 2
streams 2
```



