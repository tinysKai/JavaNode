## Kafka分区重平衡

#### 优先选举

分区使用多副本机制来提升可靠性，但只有 leader 副本对外提供读写服务，而 follower 副本只负责在内部进行消息的同步。而对同一个分区而言，同一个broker节点不可能出现它的多个副本。如果一个broker的leader副本相比其他broker过多，则会造成某broker负载过高而其他broker无压力的情况。

随着时间的更替，Kafka 集群的 broker 节点不可避免地会遇到宕机或崩溃的问题，当分区的 leader 节点发生故障时，其中一个 follower 节点就会成为新的 leader 节点，这样就会导致集群的负载不均衡，从而影响整体的健壮性和稳定性。 

**优先副本是指在AR集合列表中的第一个副本。所谓的优先副本的选举是指通过一定的方式促使优先副本选举为 leader 副本，以此来促进集群的负载均衡，这一行为也可以称为“分区平衡”。**

在 Kafka 中可以提供分区自动平衡的功能，与此对应的 broker 端参数是 `auto.leader. rebalance.enable`，此参数的默认值为 true，即默认情况下此功能是开启的。如果开启分区自动平衡的功能，则 Kafka的控制器会启动一个定时任务，这个定时任务会轮询所有的 broker 节点，计算每个 broker 节点的分区不平衡率（broker 中的不平衡率=非优先副本的 leader 个数/分区总数）是否超过·leader.imbalance.per.broker.percentage参数配置的比值，默认值为10%，如果超过设定的比值则会自动执行优先副本的选举动作以求分区平衡。执行周期由参数 `leader.imbalance.check.interval.seconds` 控制，默认值为300秒，即5分钟。不过在生产环境中不建议将 `auto.leader.rebalance.enable` 设置为默认的 true，因为这可能引起负面的性能问题，也有可能引起客户端一定时间的阻塞。

在服务器重启导致的分区不平衡，Kafka提供了对分区leader副本进行重新平衡的功能。可直接使用脚本触发整个kafka服务器集群内所有主题分区**基于优先副本**的重平衡。但由于重平衡是基于zk的来记录元数据信息的。所以如果这些数据超过zk节点数据允许的大小，则选举会失败。所以kafka还提供了文件的方式来操作需重新选举的分区。

```
//对所有主题基于优先选举来进行重平衡
[root@node1 kafka_2.11-2.0.0]# bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181/kafka

```

```json
{
        "partitions":[
                {
                        "partition":0,
                        "topic":"topic-partitions"
                },
                {
                        "partition":1,
                        "topic":"topic-partitions"
                },
                {
                        "partition":2,
                        "topic":"topic-partitions"
                }
        ]
}
```

通过 kafka-perferred-replica-election.sh 脚本配合 `path-to-json-file` 参数来对主题 topic-partitions 执行优先副本的选举操作

```
//使用上述json进行针对指定主题指定分区进行重平衡
[root@node1 kafka_2.11-2.0.0]# bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181/kafka --path-to-json-file election.json
Created preferred replica election path with topic-partitions-0,topic-partitions-1, topic-partitions-2
Successfully started preferred replica election for partitions Set(topic- partitions-0, topic-partitions-1, topic-partitions-2)
```





#### 分区转移

当kafka集群的某个broker故障或有硬件故障时，这些失效的分区副本并不会自动迁移到其它broker节点上，而这会影响整个集群的均衡负载和整个服务的可用性和可靠性。所以我们需对该broker进行下线并进行相应的分区转移。

当集群中新增 broker 节点时，只有新创建的主题分区才有可能被分配到这个节点上，而之前的主题分区并不会自动分配到新加入的节点中，因为在它们被创建时还没有这个新节点，这样新节点的负载和原先节点的负载之间严重不均衡。

Kafka 提供了 kafka-reassign-partitions.sh 脚本来执行分区重分配的工作，它可以在集群扩容、broker 节点失效的场景下对分区进行迁移。kafka-reassign-partitions.sh 脚本的使用分为3个步骤：

+ 创建需要一个包含主题清单的 JSON 文件
+ 根据主题清单和 broker 节点清单生成一份重分配方案，根据这份方案执行具体的重分配动作。

```json
//reassign.json
{
        "topics":[
                {
                     "topic":"topic-reassign"
                }
        ],
        "version":1
}
```

```c
//指定json文件来生成一份候选的重分配方案
[root@node1 kafka_2.11-2.0.0]# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --generate --topics-to-move-json-file reassign.json --broker-list 0,2
Current partition replica assignment
{"version":1,"partitions":[{"topic":"topic-reassign","partition":2,"replicas":[2,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":1,"replicas":[1,0],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":3,"replicas":[0,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":0,"replicas":[0,2],"log_dirs":["any","any"]}]}

Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"topic-reassign","partition":2,"replicas":[2,0],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":1,"replicas":[0,2],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":3,"replicas":[0,2],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":0,"replicas":[2,0],"log_dirs":["any","any"]}]}

//将上面执行结果得到的分配方案结果保存到project.json中，执行重分配动作
[root@node1 kafka_2.11-2.0.0]# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --execute --reassignment-json-file project.json 
Current partition replica assignment

{"version":1,"partitions":[{"topic":"topic-reassign","partition":2,"replicas":[2,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":1,"replicas":[1,0],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":3,"replicas":[0,1],"log_dirs":["any","any"]},{"topic":"topic-reassign","partition":0,"replicas":[0,2],"log_dirs":["any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.
```



#### 复制限流

副本间的重分配的本质在于数据复制。数据复制会占用额外的资源，如果重分配的量太大必然会严重影响整体的性能，尤其是处于业务高峰期的时候。减小重分配的粒度，以小批次的方式来操作是一种可行的解决思路。但如果某个主题的流量特别大，则数据复制还是会影响到kafka集群的整体性能，因此kafka提供了复制限流机制。

副本间的复制限流有两种实现方式

+ kafka-config.sh 脚本
+ kafka-reassign-partitions.sh 脚本

kafka-config.sh 脚本主要以动态配置的方式来达到限流的目的，在 broker 级别有两个与复制限流相关的配置参数：follower.replication.throttled.rate 和 leader.replication. throttled.rate，前者用于设置 follower 副本复制的速度，后者用于设置 leader副本传输的速度，它们的单位都是 B/s。

```c
//broker 1中的 leader 副本和 follower 副本的复制速度限制在1024B/s之内，即1KB/s
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost: 2181/kafka --entity-type brokers --entity-name 1 --alter --add-config follower.replication. throttled.rate=1024,leader.replication.throttled.rate=1024
Completed Updating config for entity: brokers '1'.

//查看配置    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost: 2181/kafka --entity-type brokers --entity-name 1 --describe
Configs for brokers '1' are leader.replication.throttled.rate=1024,follower. replication.throttled.rate=1024  

//删除复制限流配置    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost: 2181/kafka --entity-type brokers --entity-name 1 --alter --delete-config follower. replication.throttled.rate,leader.replication.throttled.rate
Completed Updating config for entity: brokers '1'.    
```

在主题级别也有两个相关的参数来限制复制的速度：leader.replication.throttled. replicas 和 follower.replication.throttled.replicas，它们分别用来配置被限制速度的主题所对应的 leader 副本列表和 follower 副本列表。

kafka-reassign-partitions.sh 脚本本身也提供了限流的功能，只需一个 throttle 参数即可。推荐使用此方式限流.

```c
//设置此次重平衡的复制限流
[root@node1 kafka_2.11-2.0.0]# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --execute --reassignment-json-file project.json  --throttle 10
Current partition replica assignment

{"version":1,"partitions":[{"topic":"topic-throttle","partition":2,"replicas":[2,0],"log_dirs":["any","any"]},{"topic":"topic-throttle","partition":1,"replicas":[1,2],"log_dirs":["any","any"]},{"topic":"topic-throttle","partition":0,"replicas":[0,1],"log_dirs":["any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Warning: You must run Verify periodically, until the reassignment completes, to ensure the throttle is removed. You can also alter the throttle by rerunning the Execute command passing a new value.
The inter-broker throttle limit was set to 10 B/s
Successfully started reassignment of partitions.

//在重分配期间,修改限流速度    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-reassign-partitions.sh --zookeeper localhost:2181/kafka --execute --reassignment-json-file project.json  --throttle 1024
There is an existing assignment running.    
```

