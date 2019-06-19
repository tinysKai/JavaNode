## 副本因子

#### 修改副本因子

获取project.json的方式

```
# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --generate --topics-to-move-json-file reassign.json --broker-list 0,2

Current partition replica assignment
{"version":1,"partitions":[{"topic":"topic-reassign","partition":2,"replicas":[2,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":1,"replicas":[1,0],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":3,"replicas":[0,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":0,"replicas":[0,2],"log_dirs":["any","any"]}]}

Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"topic-reassign","partition":2,"replicas":[2,0],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":1,"replicas":[0,2],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":3,"replicas":[0,2],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":0,"replicas":[2,0],"log_dirs":["any","any"]}]}
```

```json
//获取到的project.json文件
{
    "version": 1,
    "partitions": [
        {
            "topic": "topic-throttle",
            "partition": 1,
            "replicas": [
                2,
                0
            ],
            "log_dirs": [
                "any",
                "any"
            ]
        },
        {
            "topic": "topic-throttle",
            "partition": 0,
            "replicas": [
                0,
                2
            ],
            "log_dirs": [
                "any",
                "any"
            ]
        },
        {
            "topic": "topic-throttle",
            "partition": 2,
            "replicas": [
                0,
                2
            ],
            "log_dirs": [
                "any",
                "any"
            ]
        }
    ]
}

//增加副本的方式是在副本数组中添加相应的brokerid以及在日志数组中添加any
{
    "topic": "topic-throttle",
    "partition": 1,
    "replicas": [
        2,
        1,
        0
    ],
    "log_dirs": [
        "any",
        "any",
        "any"
    ]
}
```



```c
//通过json文件来修改副本数
[root@node1 kafka_2.11-2.0.0]# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --execute --reassignment-json-file add.json
Current partition replica assignment

{"version":1,"partitions":[{"topic":"topic-throttle","partition":2,"replicas":[2,0],"log_dirs":["any","any"]},{"topic":"topic-throttle","partition":1,"replicas":[1,2],"log_dirs":["any","any"]},{"topic":"topic-throttle","partition":0,"replicas":[0,1],"log_dirs":["any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.
```



#### 压测分区数的性能工具

+  `kafka-producer- perf-test.sh`
+ `kafka-consumer-perf-test.sh`

```c
//主题topic-1中发送100万条消息，并且每条消息大小为1024B，生产者对应的acks参数为1
//throughput 用来进行限流控制，当设定的值小于0时不限流,当设定的值大于0时，当发送的吞吐量大于该值时就会被阻塞一段时间
//producer-props 参数用来指定生产者的配置，可同时指定多组配置，各组配置之间以空格分隔
[root@node1 kafka_2.11-2.0.0]# bin/kafka-producer-perf-test.sh --topic topic-1 --num-records 1000000 --record-size 1024 --throughput -1 --producer-props bootstrap. servers=localhost:9092 acks=1
273616 records sent, 54723.2 records/sec (53.44 MB/sec), 468.6 ms avg latency, 544.0 max latency.
337410 records sent, 67482.0 records/sec (65.90 MB/sec), 454.4 ms avg latency, 521.0 max latency.
341910 records sent, 68382.0 records/sec (66.78 MB/sec), 449.4 ms avg latency, 478.0 max latency.
1000000 records sent, 63690.210815 records/sec (62.20 MB/sec), 456.17 ms avg latency, 544.00 ms max latency, 458 ms 50th, 517 ms 95th, 525 ms 99th, 543 ms 99.9th.
```

```c
//消费主题 topic-1 中的100万条消息
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-perf-test.sh --topic topic-1 --messages 1000000 --broker-list localhost:9092
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2018-09-22 12:27:49:827, 2018-09-22 12:27:57:068, 976.5625, 134.8657, 1000000, 138102.4720, 105, 7136, 136.8501, 140134.5291
```

