## kafka常见命令

#### 创建主题

```shell
# 创建一个主题为`transaction`分区数为1,副本数为1的主题
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
--create --topic transaction \
--partitions 1 --replication-factor 1 --config retention.ms=15552000000 \
--config max.message.bytes=5242880
```

#### 查询所有主题

```shell
bin/kafka-topics.sh --bootstrap-server broker_host:port --list
```

#### 查询具体主题的信息

```shell
bin/kafka-topics.sh --bootstrap-server broker_host:port --describe --topic <topic_name>
```

#### 修改主题参数

```shell
# 指定修改的类型为主题(topics) 主题名为transaction  修改的参数为最大消息大小
bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name transaction \
--alter --add-config max.message.bytes=10485760
```

#### 修改主题分区

```shell
bin/kafka-topics.sh --bootstrap-server broker_host:port \
--alter --topic <topic_name> --partitions <新分区数>
```

#### 删除主题

```shell
# 删除操作是异步的，执行完这条命令不代表主题立即就被删除了。它仅仅是被标记成“已删除”状态而已。Kafka 会在后台默默地开启主题删除操作。
# 因此，通常情况下，你都需要耐心地等待一段时间。
bin/kafka-topics.sh --bootstrap-server broker_host:port --delete  --topic <topic_name>
```

#### 调整消费者位移

```shell
#指定到最早位移处
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-earliest –execute

# 指定到最新位移处
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-latest --execute ​​

# 指定到当前位移最新提交处
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-current --execute

# 指定到指定位移处
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-offset <offset> --execute

# 指定到大于指定时间处开始
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --to-datetime 2019-06-20T20:00:00.000 --execute
```

#### 生产消息

```shell
$ bin/kafka-console-producer.sh --broker-list kafka-host:port \
--topic test-topic --request-required-acks -1 \
--producer-property compression.type=lz4
```

####  消费消息

```shell
# `from-beginning`不指定的话表示从最新位移读取消息
# `group`添加group是希望每次都指定一个消费者组来执行命令,不然脚本会自动生成很多`console-consumer` 开头的消费者组
$ bin/kafka-console-consumer.sh --bootstrap-server kafka-host:port --topic test-topic --group test-group \
--from-beginning --consumer-property enable.auto.commit=false 
```

#### 性能测试

```shell
# 生产者 --关注`99th`对应的时间,表示99%的耗时在这个内
$ bin/kafka-producer-perf-test.sh --topic test-topic --num-records 10000000 --throughput -1 --record-size 1024 --producer-props bootstrap.servers=kafka-host:port acks=-1 linger.ms=2000 compression.type=lz4
2175479 records sent, 435095.8 records/sec (424.90 MB/sec), 131.1 ms avg latency, 681.0 ms max latency.
4190124 records sent, 838024.8 records/sec (818.38 MB/sec), 4.4 ms avg latency, 73.0 ms max latency.
10000000 records sent, 737463.126844 records/sec (720.18 MB/sec), 31.81 ms avg latency, 681.00 ms max latency, 4 ms 50th, 126 ms 95th, 604 ms 99th, 672 ms 99.9th.


# 消费者
$ bin/kafka-consumer-perf-test.sh --broker-list kafka-host:port --messages 10000000 --topic test-topic
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2019-06-26 15:24:18:138, 2019-06-26 15:24:23:805, 9765.6202, 1723.2434, 10000000, 1764602.0822, 16, 5651, 1728.1225, 1769598.3012
```

#### 查看消息总数

```shell
#  通过查找最早以及当前的位移的差值来累加计算, `-time`参数解释 -1表示latest，-2表示earliest
$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list kafka-host:port --time -2 --topic test-topic
test-topic:0:0
test-topic:1:0
$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list kafka-host:port --time -1 --topic test-topic
test-topic:0:5500000
test-topic:1:5500000
```

#### 查看具体的消费者组位移

![daa9a9b9-024b-4785-aad4-78e8777d35b7-2575962.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1galyrpbtjfj21mc042jve.jpg)

