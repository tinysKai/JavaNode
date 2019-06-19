## 主题配置管理

#### 主题,分区,副本与日志的关系图

![](https://s2.ax1x.com/2019/06/04/VNSnUO.png)



#### 查询主题

若不指定分区数与副本数,默认情况下会创建一个分区数为1,副本数为1的主题(即没复制的备用副本)

**kafka脚本创建主题的脚本**

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-create --partitions 4 --replication-factor 2
Created topic "topic-create". //此为控制台执行的输出结果

#使用config参数来创建主题
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --create --topic topic-config --replication-factor 1 --partitions 1 --config cleanup.policy=compact --config max.message.bytes=10000
Created topic "topic-config".
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-config
Topic:topic-config	PartitionCount:1	ReplicationFactor:1	Configs:cleanup.policy=compact,max.message.bytes=10000
	Topic: topic-config	Partition: 0	Leader: 0	Replicas: 0	Isr: 0    
```

**查看分区副本的分配细节**

Leader 表示分区的 leader 副本所对应的 brokerId

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topic topic-create
Topic:topic-create	PartitionCount:4	ReplicationFactor:2	Configs:
    Topic: topic-create	Partition: 0	Leader: 2	Replicas: 2,0	Isr: 2,0
    Topic: topic-create	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0,1
    Topic: topic-create	Partition: 2	Leader: 1	Replicas: 1,2	Isr: 1,2
    Topic: topic-create	Partition: 3	Leader: 2	Replicas: 2,1	Isr: 2,1
```

在使用 describe 指令查看主题信息时还可以额外指定 topics-with-overrides、under-replicated-partitions 和 unavailable-partitions 这三个参数来增加一些附加功能。

```c
//增加topics-with-overrides 参数可以找出所有包含覆盖配置的主题，它只会列出包含了与集群不一样配置的主题。
#[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topics-with-overrides
Topic:__consumer_offsets	PartitionCount:50	ReplicationFactor:1	Configs:segment.bytes=104857600,cleanup.policy=compact,compression.type=producer
Topic:topic-config	PartitionCount:1	ReplicationFactor:1	Configs:cleanup.policy=compact,max.message.bytes=10000
```

under-replicated-partitions 和 unavailable-partitions 参数都可以找出有问题的分区。通过 under-replicated-partitions 参数可以找出所有包含失效副本的分区。包含失效副本的分区可能正在进行同步操作，也有可能同步发生异常，此时分区的 ISR 集合小于AR集合。对于通过该参数查询到的分区要重点监控，因为这很可能意味着集群中的某个 broker 已经失效或同步效率降低等。

通过 unavailable-partitions 参数可以查看主题中没有 leader 副本的分区，这些分区已经处于离线状态，对于外界的生产者和消费者来说处于不可用的状态。

```c
//3台机器,4个分区,1份复制副本,现在模拟关闭节点2,节点3的场景
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-create --unavailable-partitions
	Topic: topic-create	Partition: 2	Leader: -1	Replicas: 1,2	Isr: 1
	Topic: topic-create	Partition: 3	Leader: -1	Replicas: 2,1	Isr: 1

[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-create 
Topic:topic-create	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-create	Partition: 0	Leader: 0	Replicas: 2,0	Isr: 0
	Topic: topic-create	Partition: 1	Leader: 0	Replicas: 0,1	Isr: 0
	Topic: topic-create	Partition: 2	Leader: -1	Replicas: 1,2	Isr: 1
	Topic: topic-create	Partition: 3	Leader: -1	Replicas: 2,1	Isr: 1
```



**通过 list 指令可以查看当前所有可用的主题**

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka –list
__consumer_offsets
topic-create
topic-demo
topic-config
```



#### 修改主题

当一个主题被创建之后，依然允许我们对其做一定的修改，比如修改分区个数、修改配置等，这个修改的功能就是由 kafka-topics.sh 脚本中的 alter 指令提供的。

```c
//修改分区数
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --alter --topic topic-config --partitions 3
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected
Adding partitions succeeded!

[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topic topic-config
Topic:topic-config	PartitionCount:3	ReplicationFactor:1	Configs:
    Topic: topic-config	Partition: 0	Leader: 2	Replicas: 2	Isr: 2
    Topic: topic-config	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
    Topic: topic-config	Partition: 2	Leader: 1	Replicas: 1	Isr: 1
```



#### 删除主题

必须将 delete.topic.enable 参数配置为 true 才能够删除主题，这个参数的默认值就是 true，如果配置为 false，那么删除主题的操作将会被忽略。在实际生产环境中，建议将这个参数的值设置为 true。

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --delete --topic topic-delete
Topic topic-delete is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```



#### 其它参数查询

通过执行无任何参数的 kafka-topics.sh 脚本，或者执行 kafka-topics.sh –help 来查看帮助信息

下面列出常用的参数项

<table>
<thead>
<tr>
<th>参 数 名 称</th>
<th>释 义</th>
</tr>
</thead>
<tbody>
<tr>
<td>alter</td>
<td>用于修改主题，包括分区数及主题的配置</td>
</tr>
<tr>
<td>config 	&lt;键值对&gt;</td>
<td>创建或修改主题时，用于设置主题级别的参数</td>
</tr>
<tr>
<td>create</td>
<td>创建主题</td>
</tr>
<tr>
<td>delete</td>
<td>删除主题</td>
</tr>
<tr>
<td>delete-config &lt;配置名称&gt;</td>
<td>删除主题级别被覆盖的配置</td>
</tr>
<tr>
<td>describe</td>
<td>查看主题的详细信息</td>
</tr>
<tr>
<td>disable-rack-aware</td>
<td>创建主题时不考虑机架信息</td>
</tr>
<tr>
<td>help</td>
<td>打印帮助信息</td>
</tr>
<tr>
<td>if-exists</td>
<td>修改或删除主题时使用，只有当主题存在时才会执行动作</td>
</tr>
<tr>
<td>if-not-exists</td>
<td>创建主题时使用，只有主题不存在时才会执行动作</td>
</tr>
<tr>
<td>list</td>
<td>列出所有可用的主题</td>
</tr>
<tr>
<td>partitions &lt;分区数&gt;</td>
<td>创建主题或增加分区时指定分区数</td>
</tr>
<tr>
<td>replica-assignment &lt;分配方案&gt;</td>
<td>手工指定分区副本分配方案</td>
</tr>
<tr>
<td>replication-factor &lt;副本数&gt;</td>
<td>创建主题时指定副本因子</td>
</tr>
<tr>
<td>topic &lt;主题名称&gt;</td>
<td>指定主题名称</td>
</tr>
<tr>
<td>topics-with-overrides</td>
<td>使用 describe 查看主题信息时，只展示包含覆盖配置的主题</td>
</tr>
<tr>
<td>unavailable-partitions</td>
<td>使用 describe 查看主题信息时，只展示包含没有 leader 副本的分区</td>
</tr>
<tr>
<td>under-replicated-partitions</td>
<td>使用 describe 查看主题信息时，只展示包含失效副本的分区</td>
</tr>
<tr>
<td>zookeeper</td>
<td>指定连接的 ZooKeeper 地址信息（必填项）</td>
</tr>
</tbody>
</table>



#### 配置管理

```
bin/kafka-configs.sh --zookeeper localhost:2181/kafka 
		--describe --entity-type topics --entity-name topic-config
```

--describe 指定了查看配置的指令动作，--entity-type 指定了查看配置的实体类型，--entity-name 指定了查看配置的实体名称。entity-type 只可以配置4个值：topics、brokers 、clients 和 users，entity-type 与 entity-name 的对应关系如下表所示。



使用alter指令来变更配置

```c
//使用add-config 参数来增加覆盖参数
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost:2181/ kafka --alter --entity-type topics --entity-name topic-config --add-config  cleanup.policy=compact,max.message.bytes=10000
Completed Updating config for entity: topic 'topic-config'.

//查询参数    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost:2181/ kafka --describe --entity-type topics --entity-name topic-config
Configs for topic 'topic-config' are max.message.bytes=10000,cleanup.policy= compact

//使用kafka-topics.sh脚本来查询主题
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-config --topics-with-overrides
Topic:topic-config	PartitionCount:3	ReplicationFactor:1	Configs:max.message.bytes=10000,cleanup.policy=compact

//使用delete-config删除参数
[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost:2181/ kafka --alter --entity-type topics --entity-name topic-config --delete-config  cleanup.policy,max.message.bytes
Completed Updating config for entity: topic 'topic-config'.

[root@node1 kafka_2.11-2.0.0]# bin/kafka-configs.sh --zookeeper localhost:2181/ kafka --describe --entity-type topics --entity-name topic-config
Configs for topic 'topic-config' are
```

**主题端参数**

与主题相关的所有配置参数在 broker 层面都有对应参数，比如主题端参数 cleanup. policy 对应 broker 层面的 log.cleanup.policy。如果没有修改过主题的任何配置参数，那么就会使用 broker 端的对应参数作为其默认值。

<table>
<thead>
<tr>
<th>主题端参数</th>
<th>释    义</th>
<th>对应的 broker 端参数</th>
</tr>
</thead>
<tbody>
<tr>
<td>cleanup.policy</td>
<td>日志压缩策略。默认值为 delete，还可以配置为 compact</td>
<td>log.cleanup.policy</td>
</tr>
<tr>
<td>compression.type</td>
<td>消息的压缩类型。默认值为 producer，表示保留生产者中所使用的原始压缩类型。还可以配置为 uncompressed、snappy、lz4、gzip</td>
<td>compression.type</td>
</tr>
<tr>
<td>delete.retention.ms</td>
<td>被标识为删除的数据能够保留多久。默认值为86400000，即1天</td>
<td>log.cleaner.delete.retention.ms</td>
</tr>
<tr>
<td>file.delete.delay.ms</td>
<td>清理文件之前可以等待多长时间，默认值为60000，即1分钟</td>
<td>log.segment.delete.delay.ms</td>
</tr>
<tr>
<td>flush.messages</td>
<td>需要收集多少消息才会将它们强制刷新到磁盘，默认值为 Long.MAX_VALUE，即让操作系统来决定。建议不要修改此参数的默认值</td>
<td>log.flush.interval.messages</td>
</tr>
<tr>
<td>flush.ms</td>
<td>需要等待多久才会将消息强制刷新到磁盘，默认值为 Long.MAX_VALUE，即让操作系统来决定。建议不要修改此参数的默认值</td>
<td>log.flush.interval.ms</td>
</tr>
<tr>
<td>follower.replication.throttled.replicas</td>
<td>用来配置被限制速率的主题所对应的 follower 副本列表</td>
<td>follower.replication.throttled.replicas</td>
</tr>
<tr>
<td>index.interval.bytes</td>
<td>用来控制添加索引项的频率。每超过这个参数所设置的消息字节数时就可以添加一个新的索引项，默认值为4096</td>
<td>log.index.interval.bytes</td>
</tr>
<tr>
<td>leader.replication.throttled.replicas</td>
<td>用来配置被限制速率的主题所对应的 leader 副本列表</td>
<td>leader.replication.throttled.replicas</td>
</tr>
<tr>
<td>max.message.bytes</td>
<td>消息的最大字节数，默认值为1000012</td>
<td>message.max.bytes</td>
</tr>
<tr>
<td>message.format.version</td>
<td>消息格式的版本，默认值为 2.0-IV1</td>
<td>log.message.format.version</td>
</tr>
<tr>
<td>message.timestamp.difference. max.ms</td>
<td>消息中自带的时间戳与 broker 收到消息时的时间戳之间最大的差值，默认值为 Long.MAX_VALUE。此参数只有在 meesage. timestamp.type 参数设置为 CreateTime 时才有效</td>
<td>log.message.timestamp. difference.max.ms</td>
</tr>
<tr>
<td>message.timestamp.type</td>
<td>消息的时间戳类型。默认值为 CreateTime，还可以设置为 LogAppendTime</td>
<td>log.message.timestamp. type</td>
</tr>
<tr>
<td>min.cleanable.dirty.ratio</td>
<td>日志清理时的最小污浊率，默认值为0.5</td>
<td>log.cleaner.min.cleanable. ratio</td>
</tr>
<tr>
<td>min.compaction.lag.ms</td>
<td>日志再被清理前的最小保留时间，默认值为0</td>
<td>log.cleaner.min.compaction. lag.ms</td>
</tr>
<tr>
<td>min.insync.replicas</td>
<td>分区ISR集合中至少要有多少个副本，默认值为1</td>
<td>min.insync.replicas</td>
</tr>
<tr>
<td>preallocate</td>
<td>在创建日志分段的时候是否要预分配空间，默认值为 false</td>
<td>log.preallocate</td>
</tr>
<tr>
<td>retention.bytes</td>
<td>分区中所能保留的消息总量，默认值为-1，即没有限制</td>
<td>log.retention.bytes</td>
</tr>
<tr>
<td>retention.ms</td>
<td>使用 delete 的日志清理策略时消息能够保留多长时间，默认值为604800000，即7天。如果设置为-1，则表示没有限制</td>
<td>log.retention.ms</td>
</tr>
<tr>
<td>segment.bytes</td>
<td>日志分段的最大值，默认值为1073741824，即1GB</td>
<td>log.segment.bytes</td>
</tr>
<tr>
<td>segment.index.bytes</td>
<td>日志分段索引的最大值，默认值为10485760，即10MB</td>
<td>log.index.size.max.bytes</td>
</tr>
<tr>
<td>segment.jitter.ms</td>
<td>滚动日志分段时，在 segment.ms 的基础之上增加的随机数，默认为0</td>
<td>log.roll.jitter.ms</td>
</tr>
<tr>
<td>segment.ms</td>
<td>最长多久滚动一次日志分段，默认值为604800000，即7天</td>
<td>log.roll.ms</td>
</tr>
<tr>
<td>unclean.leader.election.enable</td>
<td>是否可以从非 ISR 集合中选举 leader 副本，默认值为 false，如果设置为 true，则可能造成数据丢失</td>
<td>unclean.leader.election. enable</td>
</tr>
</tbody>
</table>