## 消费者管理

#### 查看集群的消费者组信息

```c
//查看消费者组
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
console-consumer-98513
groupIdMonitor
console-consumer-49560
    
//使用describe 来查询指定消费者组的消息信息
//LAG表示消息滞后的数量，是LOG-END-OFFSET(HW,高水位)与CURRENT-OFFSET 的数值之差
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
topic-monitor   0          668             668             0               consumer-1-063cdec2-b525-4ba3-bbfe-db9a92e3b21d /192.168.0.2  consumer-1
topic-monitor   1          666             666             0               consumer-1-063cdec2-b525-4ba3-bbfe-db9a92e3b21d /192.168.0.2  consumer-1
topic-monitor   2          666             666             0               consumer-1-273faaf0-c950-44a8-8a11-41a116f79fd4 /192.168.0.2  consumer-1    
```

消费组一共有 Dead、Empty、PreparingRebalance、CompletingRebalance、Stable 这几种状态，正常情况下，一个具有消费者成员的消费组的状态为 Stable。

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor --state

COORDINATOR (ID)         ASSIGNMENT-STRATEGY    STATE                #MEMBERS
192.168.0.4:9092 (2)    range                     Stable               2
```

查看消费者成员信息

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor --members

CONSUMER-ID         HOST            CLIENT-ID    #PARTITIONS
consumer-1-273	   /192.168.0.2  consumer-1      1
consumer-1-063	   /192.168.0.2  consumer-1      2
```



#### 删除消费者组

通过delete指令可删除一个指定的消费者组,但前提是消费者组中无运行着的消费者成员,否则会删除失败.

```c
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --delete --group groupIdMonitor
Deletion of requested consumer groups ('groupIdMonitor') was successful.

[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor 
Error: Consumer group 'groupIdMonitor' does not exist.
```



#### 消费位移管理

kafka-consumer-groups.sh 脚本还提供了重置消费组内消费位移的功能，具体是通过 reset-offsets 这个指令类型的参数来实施的，不过实现这一功能的前提是消费组内没有正在运行的消费者成员。

```c
//将消费组中的所有主题的所有分区的消费位移都置为0
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --all-topics --reset-offsets --to-earliest --execute

TOPIC                          PARTITION  NEW-OFFSET     
topic-monitor                  1          0              
topic-monitor                  0          0              
topic-monitor                  2          0              

[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor
Consumer group 'groupIdMonitor' has no active members.

TOPIC        PARTITION   CURRENT-OFFSET   LOG-END-OFFSET    LAG      CONSUMER-ID       HOST           CLIENT-ID
topic-monitor    1           0                   999             999             -               -               -
topic-monitor    0           0                  1001            1001            -               -               -
topic-monitor    2           0                  1000            1000            -               -               -
    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --topic topic-monitor:2 --reset-offsets --to-latest --execute

TOPIC                          PARTITION  NEW-OFFSET     
topic-monitor                  2          1000           

//将消费者组中主题 topic-monitor 分区2的消费位移置为分区的末尾    
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor
Consumer group 'groupIdMonitor' has no active members.

TOPIC         PARTITION    CURRENT-OFFSET   LOG-END-OFFSET   LAG          CONSUMER-ID     HOST         CLIENT-ID
topic-monitor    1               0               999             999               -               -               -
topic-monitor    0               0              1001            1001               -               -               -
topic-monitor    2           1000              1000                0  
```

`kafka-consumer-groups.sh`提供的消费位移方式

- to-earliest 
- to-earliest : 将消费位移调整到分区的开头
- to-latest : 将消费位移调整到分区的末尾
- by-duration <String: duration>：将消费位移调整到距离当前时间指定间隔的最早位移处。duration 的格式为“PnDTnHnMnS”。
- from-file <String: path to CSV file>：将消费位移重置到CSV文件中定义的位置。
- shift-by <Long: number-of-offsets>：把消费位移调整到当前位移 + number-of-offsets 处，number-of-offsets 的值可以为负数。
- to-current：将消费位移调整到当前位置处。
- to-datetime <String: datatime>：将消费位移调整到大于给定时间的最早位移处。datatime 的格式为“YYYY-MM-DDTHH:mm:SS.sss”。
- to-offset <Long: offset>：将消费位移调整到指定的位置。

`kafka-consumer-groups.sh` 脚本中还有两个参数` dry-run `和 `export`，`dry-run`` 是只打印具体的调整方案而不执行，export `是将位移调整方案以 CSV 的格式输出到控制台，而 `execute `才会执行真正的消费位移重置。

调整消费位移的例子

```c
# 查看当前消费组的消费位移
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor
Consumer group 'groupIdMonitor' has no active members.

TOPIC           PARTITION   CURRENT-OFFSET   LOG-END-OFFSET   LAG       CONSUMER-ID      HOST        CLIENT-ID
topic-monitor      1           999                999              0               -               -               -
topic-monitor      0          1001               1001             0               -               -               -
topic-monitor      2          1000               1000             0               -               -               -
# 将消费位移往前调整10，但是不执行
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --topic topic-monitor --reset-offsets --shift-by -10 --dry-run

TOPIC                          PARTITION  NEW-OFFSET     
topic-monitor                  2           990            
topic-monitor                  1           989            
topic-monitor                  0           991    
# 将消费位移调整为当前位移并将结果输出到控制台，但是也不执行
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --topic topic-monitor --reset-offsets --to-current --export -dry-run
topic-monitor,2,1000
topic-monitor,1,999
topic-monitor,0,1001
# 将消费位移再次往前调整20并输出结果，但是不执行
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --topic topic-monitor --reset-offsets --shift-by -20 --export --dry-run
topic-monitor,2,980
topic-monitor,1,979
topic-monitor,0,981
# 中间步骤：将上面的输出结果保存到offsets.csv文件中
# 通过from-file参数从offsets.csv文件中获取位移重置策略，并且执行
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092  --group groupIdMonitor --topic topic-monitor --reset-offsets --from-file offsets.csv --execute

TOPIC                          PARTITION  NEW-OFFSET     
topic-monitor                  2          980            
topic-monitor                  1          979            
topic-monitor                  0          981            
# 最终消费位移都往前重置了20
[root@node1 kafka_2.11-2.0.0]# bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupIdMonitor
Consumer group 'groupIdMonitor' has no active members.

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID     HOST            CLIENT-ID
topic-monitor   1           979                999              20              -               -               -
topic-monitor   0           981               1001              20              -               -               -
topic-monitor   2           980               1000              20              -      
```

#### 手动删除消息

kafka-delete-records.sh 这个脚本可以用来删除指定位置前的消息。但其实该指令只是将指定分区的 logStartOffset 置为相应的请求值（比如分区0的偏移量10），最终的删除消息的动作还是交由日志删除任务来完成的。

```json
//新建一个json用来指定删除的分区的位移,取名为 delete.json
{
    "partitions": [
        {
            "topic": "topic-monitor",
            "partition": 0,
            "offset": 10
        },
                {
            "topic": "topic-monitor",
            "partition": 1,
            "offset": 11
        },
                {
            "topic": "topic-monitor",
            "partition": 2,
            "offset": 12
        }
    ],
    "version": 1
}
```

```c
//删除分区位移前的消息
[root@node1 kafka_2.11-2.0.0]# bin/kafka-delete-records.sh --bootstrap-server localhost:9092 --offset-json-file delete.json
Executing records delete operation
Records delete operation completed:
partition: topic-monitor-0	low_watermark: 10
partition: topic-monitor-1	low_watermark: 11
partition: topic-monitor-2	low_watermark: 12
```



