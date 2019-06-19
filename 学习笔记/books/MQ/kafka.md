《Kafka权威指南》读书笔记
=

Kafka概要
-
> Kafka是一个流平台,在这个平台上可以发布和订阅数据流,并把他们保存起来,进行处理


>`Kafka不同点`
+ 分布式系统,以集群方式运行,可自由伸缩
+ 定制化存储数据需求
+ 流式处理能力让我们可以用很少的代码就能够动态的处理派生流和数据集
    
>为了提高效率，消息被分批次写入Kafka . 批次就是一组消息，这些消息属于同一个主题和分区
    

>主题与分区

    Kafka的悄息通过主题进行分类.主题可以被分为若干个分区 ，一个分区就是一个提交日志.
    消息以追加的方式写入分区 ，然后以先入先出的顺序读取.
    由于一个主题一般包含几个分区，因此无法在整个主题范围内保证消息的顺序，但可以保证消息在单个分区内的顺序
    
>消费者是消费者群组的一部分，也就是说，会有一个或多个消费者共同读取一个主题.群组保证每个分区只能被一个消费者使用.
 
 
>每个集群都有一个broker同时充当了集群控制器的角色.负责管理工作包括将分区分配给broker和监控broker.


>`集群`

    在集群中，一个分区从属于一个broker, 该broker被称为分区的首领
    一个分区可以分配给多个broker 这个时候会发生分区复制
    这种复制机制为分区提供了消息冗余，如果有一个broker失效，其他broker可以接管领导权

>保留消息（在一 定期限内）是 Kafka 的 一个重要特性.  
 Kafka broker 默认的消息保留策略 : 要么保留一段时间（比如7天），要么保留到消息达到一定大小的字节数(比如1G)

                                                                              
>常见命令

    创建主题
        ./kafka-topics.sh --create --zookeeper localhost:2222  --replication-factor 1  --partitions 1   --topic test
    
    查询主题
        ./kafka-topics.sh  --zookeeper localhost:2222 --describe  --topic test
    
    
    往主题发送消息
        ./kafka-console-producer.sh --broker-list localhost:9092 --topic test
    
    从主题读取消息
        ./kafka-console-consumer.sh --zookeeper localhost:2222 --topic test --from-beginning


                                                                              
                                                                              
生产者
-

>`kafka创建客户端代码`

     Properties props = new Properties();
     props.put("metadata.broker.list", kafkaConnectString.trim());
     props.put("bootstrap.servers", kafkaConnectString.trim());
     props.put("producer.type", "async");//消息发送类型同步(sync)还是异步(async将本地buffer)
     props.put("compression.codec", "none");//消息的压缩格式，默认为none不压缩，gzip, snappy, lz4
     props.put("request.required.acks", "1");//0不应答,1表示集群的首领收到,1的情况还是有可能发生丢失数据的,比如某刻选举的新首领无收到刚刚的消息
     props.put("message.send.max.retries", 3);//失败重试次数
     props.put("retry.backoff.ms", 100);//重试间隔
     props.put("queue.buffering.max.ms", 10);//缓存数据的最大时间间隔
     props.put("batch.num.messages", 1000);//缓存数据的最大条数
     props.put("max.request.size", 1024 * 1024);
     props.put("key.serializer", "org.apache.kafka.common.serialization.ByteArraySerializer");
     props.put("value.serializer", "org.apache.kafka.common.serialization.ByteArraySerializer");
     this.producer = new KafkaProducer<String, String>(props);
        

>  

>`kafka发送消息`  
 
 同步调用  
    
    ProducerRecord record = new ProducerRecord<String,byte[]>(topic,value.getBytes());
    producer.send(record);
 
异步调用  

    ProducerRecord record = new ProducerRecord<String,byte[]>(topic,value.getBytes());
    producer.send(record,callback);//callback参数实现了Callback接口的onCompletion方法,发送消息会回调到这里
    

消费者
-

##### 消费者与消费群组 
一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息.  
假设一个主题有四个分区,而消费者群组有两个消费者,那么此时每个消费者会消费其中的各两个分区..当有四个消费者时,每个消费者会消费一个分区.  
`而当消费数超过了主题分区数时,多余的消费者会闲置`.
可以针对同一个主题创建多个消费者组来实现一对多的传送数据,多个消费者组的情况下,每个消费者组都会得到一样的消息,其中每个消费者组再根据其自身的组内消费者去分配


##### 分区再均衡
分区的所有权从一个消费者转移到另一个消费者称为再平衡.在再均衡期间，消费者无陆读取消息，造成整个群组一小段时间的不可用


##### 创建消费者
    
     Properties props = new Properties();
     props.put("bootstrap.servers", connectString.trim()); //kafka服务器端口信息
     props.put("group.id","groupName");//定义消费者属于哪个群组
     props.put("enable.auto.commit", "false"); //是否自动提交offset
    //props.put("auto.commit.interval.ms", "1000"); //每隔多久自动提交一次
     props.put("session.timeout.ms", "10000");//session超时时间
     //props.put("max.poll.interval.ms", "5000");//隔多久至少要poll一次
     props.put("max.poll.records", "1000");//每次最多取多少条记录
     props.put("offsets.storage", "zookeeper");
     //props.put("dual.commit.enabled", "true");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer");
     KafkaConsumer<String,byte[]> consumerClient = new KafkaConsumer<String,byte[]>(props);


##### 订阅主题

+ 列表模式订阅    
      consumerClient.subscribe(topicNames);
+ 正则订阅  
      consumerClient.subscribe("test.*");
      
##### 轮询消息

    //拉取数据
     ConsumerRecords<String,byte[]> records = consumerClient.poll(100);
     //处理数据      
     if(records!=null  && !records.isEmpty()){
        //根据分区迭代
        for(Iterator<TopicPartition>iterator=records.partitions().iterator();iterator.hasNext();){
     		  TopicPartition topicPartition=  iterator.next();
     	      List<ConsumerRecord<String, byte[]>> partitionRecordLists=records.records(topicPartition);
     	      if(partitionRecordLists!=null && !partitionRecordLists.isEmpty()){
                  for (ConsumerRecord<String,byte[]> record : partitionRecordLists){
                          T info = serializer.deserialize(record.value());
                          String infoStr = JSON.toJSONString(info);
                          logger.info("partition ={}, offset ={}, key ={}, value ={}", 
                             record.partition(),record.offset(), record.key(), infoStr);                              
                    }  
     	       }
        }
     }
     //处理偏移量
     Map<TopicPartition, OffsetAndMetadata> offsetMap = new HashMap<TopicPartition, OffsetAndMetadata>();
     //TODO 处理偏移量
     //提交到kafka
     consumerClient.commitSync(offsetMap);
       
         
##### 偏移量
>提交偏移量有四种方式
+ 自动提交(默认每5s提交一次,所以有可能会有5s的记录会重复消费)
+ 手动同步提交(commitSync[不指定参数提交用poll返回的最新偏移量],只要木有发生不可恢复的错误,同步提交会有重提交)
+ 手动异步提交(commitAsync[不指定参数提交用poll返回的最新偏移量],失败不会重试,因为异步有可能重试后旧的偏移量覆盖新的偏移量)
+ 指定偏移量的提交(commitSync和commitAsync都支持,控制粒度可细致到每一次拉取消息中的每个分区)

>我们还可以细致到在再均衡时处理偏移量问题  
在订阅主题时通过传递一个实现了ConsumerRebalanceListener接口的实例就行,接口下有两个方法
+ onPartitionsRevoked(会在再均衡之前和消费者停止读取消息之后被调用)
+ onPartitionsAssigned(在重新分配分区之后和消费者开始读取消息之前被调用)

>如果还想更细致的话还可以使用数据库保存偏移量信息,利用事务的原子性,以及kafka的seed方法来重新拉取



##### 消费者退出
使用consume.wakeup()以及Runtime的关闭钩子来关闭消费者
    
    //关闭消费者的伪代码
    Runtime.getRuntime().addShutdownHook(new Thread(){
        public void run(){
            consume.wakeup();
        }
    )
    
    //消费者拉取消息的代码
    try{
        while(true){
            //拉取消息的代码
        }
    }catch(WakeupException e){
        //关闭异常忽略
    }finally{
        consumer.close();
    }
    
    

深入kafka
-

>kafka使用zk的临时节点来选举控制器,并在节点加入集群或退出集群时通知控制器.  
控制器负责在节点加入或离开集群时进行分区首领选举.控制器使用epoch 来避免“脑裂” .
  
  
>kafka定义 : “一个分布式的 、可分区的、可复制的提交日志服务”

>Kafka对可靠性保证的定义，消息只有在被写入到所有同步副本之后才被认为是已提交的.(这个跟生产者的ack参数有关,为1时只需首领副本写入,为all时才需所有副本写入)

>Kafka使用主题来组织数据，每个主题被分为若干个分区，每个分区有多个副本.  
副本有两种类型
+ 首领副本    每个分区都有一个首领 副本 . 为了保证一 致性，所有生产者请求和消费者请求都会经过这个副本.
+ 跟随者副本  跟随者副本不处理来自客户端的请求，它们唯一的任务是从首领那里复制消息，保持与首领一致的状态


>`物理存储`  
Kafka 的基本存储单元是分区.  
在创建主题时， Kafka首先会决定如何在broker间分配分区.  
假设你有6个broker,打算创建一个包含10个分区的主题，并且复制系数为3,Kafka就会有30个分区副本.


可靠的数据传递
-

##### 复制
Kafka的主题被分为多个分区,每个分区可以有多个副本，其中一个副本是首领.  
所有的事件都直接发送给首领副本，或者直接从首领副本读取事件.其他副本只需要与首领保持同步，并及时复制最新的事件.  

副本又分为以下两种
+ 同步副本(ISR) 分区首领是同步副本
+ 非同步副本 


一个滞后的同步副本会导致生产者和消费者变慢，因为在消息被认为已提交之前，客户端会等待所有同步副本接收消息.  
而如果一个副本不再同步了，我们就不再关心 它是否已经收到消息.虽然非同步副本同样滞后，但它并不会对性能产生任何影响.  
但是，更少的同步副本意味着更低的有效复制系数，在发生岩机时丢失数据的风险更大.

##### broker配置

>`复制系数`  
 主题级别的配置为replication.factor,而broker级别的配置为default.replication.factor.假设主题复制系数为3,即每个分区总共会被三个不同的broker复制3次
 
>`不完全的首领选举`  
  unclean.leader.election,默认配置为true.即当首领副本不可用时其它副本都是不同步的情况下,允许非同步副本成为新首领的配置  
  简而言之，如果我们允许不同步的副本成为首领，那么就要承担丢失数据和出现数据不一致的风险.  
  如果不允许它们成为首领，那么就要接受较低的可用性，因为我们必须等待原先的首领恢复到可用状态 .

>`最少同步副本`  
 min.insync.replicas 当同步副本小于最少同步副本时,生产者无法写入消息,但消费者仍能读取消息.
 
 >`生产者重试机制`  
 重试可解决像首领选举或网络连接类问题.但重试机制有风险,有可能在网络问题时发生重复推送消息的情况,.
 重试和恰当的错误处理可以保证每个消息"至少被保存一次".所以需在消息中加消息唯一标志,以及对消息做重复校验
 
 
##### 消费者的可靠性配置
 使用显式提交偏移量主要有两个目的 :
 + 减少重复处理消息
 + 因为消息处理逻辑在轮询[不在poll拉取消息的那个循环里]之外(比如异步话另起一个后台线程去处理,自动提交机制可能在消息还没处理完前就提交偏移量了)
 
 >注意消费者在轮询中的长时间处理  
 暂停轮询的时间一般不得超过几秒钟.即使不想获取更多的数据也得保持轮询.这样客户端才能往broker发送心跳.
 
 
 
 kafka连接器
 -
 ##### 在kafka connect 与 kafka 客户端之间的抉择  
 >Kafka 客户端是要被内嵌到应用程序里的，应用程序使用它们向 Kafka 写入数据或从 Kafka 读取数据.   
 如果你是开发人员，你会使用 Kafka 客户端将应用程序连接到Kafka ，井修改应用程序的代码，将数据推送到 Kafka 或者从 Kafka 读取数据.  
 如果要将 Kafka 连接到数据存储系统，可以使用 Connect,因为这些系统不是你开发的，你无能或者也不想修改它们的代码.  
 Connect 可以用于从外部数据存储系统读取数据， 或者将数据推送到外部存储系统.  
 如果数据存储系统提供了相应的连接器，那么非开发人员就可以通过配置连接器的方式来使用Connect.




管理kafka
 -
 
启动kafka
 ./kafka-server-start.sh -daemon ../config/server.properties
 
 
##### 主题操作

>`创建主题`  
kafka-topic.sh -zookeeper <zk address> -- create --topic <topic name> --replication-factor <num> --partitions <num>


>`为主题增加分区`  
kafka-topic.sh -zookeeper <zk address> --alter --topic <topic name> --partitions <num>  
对于基于键的主题添加分区是很困难的.如果改变了分区的数量，键到分区之间的映射也会发生变化 

>`删除主题`  
kafka-topic.sh -zookeeper <zk address> --delete --topic <topic name> 

>`列出全部主题`  
kafka-topic.sh -zookeeper <zk address> --list


>`列出主题详细信息`  
./kafka-topics.sh  --zookeeper localhost:2222  --describe  
结果如下:  

    topic:test	 PartitionCount:2	 ReplicationFactor:1	 Configs:  
	Topic: test	 Partition: 0	     Leader: 0	           Replicas: 0	 Isr: 0
	Topic: test	 Partition: 1	     Leader: 0	           Replicas: 0	 Isr: 0  
使用--under-replicated-partitions参数可以列出所有包含不同步副本的分区  
使用--unavailable-partitions参数可以列出所有木有首领的分区
		
		
		
##### 消费者群组

>`列出并描述群组`  
kafka-consume-groups.sh --new-consumer --bootstrap-server <kafka broker address> --list  
对于列出的任意群组来说，使用--describe代替--list ，井通过--group指定特定的群组，就可以获取该群组的详细信息.它会列出群组里所有主题的信息和每个分区的偏移量.  
kafka-consume-groups.sh --new-consumer --bootstrap-server <kafka broker address> --describe --group <group name>  





流式处理
-
略
		
		
	
