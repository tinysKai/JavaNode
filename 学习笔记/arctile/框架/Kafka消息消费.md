## Kafka消息消费

#### 消费者与消费者组

每个消费者都有一个对应的消费组。当消息发布到主题后，只会被投递给订阅它的**每个消费组中的一个消费者**。

Kafka的消费者默认是按照分区来分配消费的,每个分区只能隶属于一个消费者组的一个消费者.

消费者数量的提升会提高消费消息的速度,但随着消费者数跟分区数一致时,也就是分区数跟消费者一一对应时,此时再增加的消费者只能闲置,而不能再提高整体的消费速度了.

![https://ws3.sinaimg.cn/large/005BYqpggy1g3izo1dhmmj30pr0do768.jpg](https://s2.ax1x.com/2019/05/30/VKYxG6.png)

按照 Kafka 默认的规则，上图最后的分配结果是消费组A中的每一个消费者分配到1个分区，消费组B中的每一个消费者分配到2个分区，两个消费组之间互不影响。每个消费者只能消费所分配到的分区中的消息。



#### 消费者消费消息逻辑步骤

1. 配置消费者客户端参数及创建相应的消费者实例
2. 订阅主题
3. 拉取消息并消费
4. 提交消费位移
5. 关闭消费者实例

#### 消息订阅

集合订阅的方式 `subscribe(Collection)`、`正则表达式订阅的方式 subscribe(Pattern)` 和`指定分区的订阅方式 assign(Collection) `分表代表了三种不同的订阅状态：`AUTO_TOPICS`、`AUTO_PATTERN` 和 `USER_ASSIGNED`（如果没有订阅，那么订阅状态为 NONE）。然而这三种状态是互斥的，在一个消费者中只能使用其中的一种，否则会报出 IllegalStateException 异常.注意通过 subscribe() 方法订阅主题具有消费者自动再均衡的功能,而指定分区的 assign() 方法是木有的.

一个消费者可以订阅一个或多个主题，客户端为我们提供了 subscribe() 方法订阅了主题，对于这个方法而言，既可以以集合的形式订阅多个主题，也可以以正则表达式的形式订阅特定模式的主题。subscribe 的几个重载方法如下：

```java
//ConsumerRebalanceListener是是用来设置相应的再均衡监听器的
public void subscribe(Collection<String> topics, ConsumerRebalanceListener listener)
public void subscribe(Collection<String> topics)
public void subscribe(Pattern pattern, ConsumerRebalanceListener listener)
public void subscribe(Pattern pattern)
```

对于消费者使用集合的方式（subscribe(Collection)）来订阅主题而言，如果前后两次订阅了不同的主题，那么消费者以最后一次的为准(不会进行两个订阅的主题并集进行订阅的).

如果消费者采用的是正则表达式的方式（subscribe(Pattern)）订阅，在之后的过程中，如果有人又创建了新的主题，并且主题的名字与正则表达式相匹配，那么这个消费者就可以消费到新添加的主题中的消息。

```java
consumer.subscribe(Pattern.compile("topic-.*"));
```



#### 偏移量

对于消息在分区中的位置，我们将 offset 称为“偏移量”；对于消费者消费到的位置，将 offset 称为“位移”，有时候也会更明确地称之为“消费位移”。

在旧消费者客户端中，消费位移是存储在 ZooKeeper 中的。而在新消费者客户端中，消费位移存储在 Kafka 内部的主题__consumer_offsets 中。这里把将消费位移存储起来（持久化）的动作称为“提交”，消费者在消费完消息之后需要执行消费位移的提交。

**偏移量提交方式**

+ 自动提交

+ 同步提交(commitSync)

  + 无参数方法,使用当前poll方法拉取的最新位移量

  + 有参数方法,用来提交指定分区的位移

    ```java
    public void commitSync(final Map<TopicPartition, OffsetAndMetadata> offsets);
    ```

+ 异步提交



同步提交可使用批量处理,批量提交的方式来提高效率

```java
final int batchSize = 500;
List<ConsumerRecord> buffer = new ArrayList<>();
while (isRunning.get()) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records) {
        buffer.add(record);
    }
    if (buffer.size() >= batchSize) {
        doSomething();
        consumer.commitSync();
        buffer.clear();
    }
}
```

使用按照分区的方式来批量提交偏移量

```java
try {
    while (isRunning.get()) {
        ConsumerRecords<String, String> records = consumer.poll(1000);
        for (TopicPartition partition : records.partitions()) {
            List<ConsumerRecord<String, String>> partitionRecords =
                    records.records(partition);
            for (ConsumerRecord<String, String> record : partitionRecords) {
                doSomeThing();
            }
            //读取相同分区的最后一条位移
            long lastConsumedOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
            consumer.commitSync(Collections.singletonMap(partition,
                    new OffsetAndMetadata(lastConsumedOffset + 1)));
        }
    }
} finally {
    consumer.close();
}
```



#### 控制或关闭消费

KafkaConsumer 提供了对消费速度进行控制的方法，在有些应用场景下我们可能需要暂停某些分区的消费而先消费其他分区，当达到一定条件时再恢复这些分区的消费。KafkaConsumer 中使用 pause() 和 resume() 方法来分别实现暂停某些分区在拉取操作时返回数据给客户端和恢复某些分区向客户端返回数据的操作。

```java
public void pause(Collection<TopicPartition> partitions);
public void resume(Collection<TopicPartition> partitions);
```



#### 指定位移消费

KafkaConsumer 中的 seek() 方法提供了从特定的位移处开始拉取消息的机制,让我们得以追前消费或回溯消费。

```java
public void seek(TopicPartition partition, long offset);
```

获取偏移量的方法

```java
//用来获取指定分区的末尾的消息位置
public Map<TopicPartition, Long> endOffsets(Collection<TopicPartition> partitions);
public Map<TopicPartition, Long> endOffsets(Collection<TopicPartition> partitions,Duration timeout);

//用来获取指定分区的起始的消息位置,不一定每次都是0,因为日志清理会清除旧数据
public Map<TopicPartition, Long> beginningOffsets(Collection<TopicPartition> partitions);
public Map<TopicPartition, Long> beginningOffsets(Collection<TopicPartition> partitions,Duration timeout);

//直接指定起始/末尾消费的seek方法
public void seekToBeginning(Collection<TopicPartition> partitions);
public void seekToEnd(Collection<TopicPartition> partitions);

//根据时间来查询对应分区的偏移量
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(Map<TopicPartition, Long> timestampsToSearch)
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(Map<TopicPartition, Long> timestampsToSearch, Duration timeout)
    
```







#### 消费者代码实现

初始化消费者实例

```java
//创建消费者的工厂
public class KafkaConsumerFactory {
	
	private static KafkaConsumerFactory kafkaConsumerFactory;
	
	private static String lock="lock";
	
    //初始化自定义的工厂类
	public static KafkaConsumerFactory build(){
		if(kafkaConsumerFactory==null){
			synchronized(lock){
				if(kafkaConsumerFactory==null){
					kafkaConsumerFactory = new KafkaConsumerFactory();
				}
			}
		}
		return kafkaConsumerFactory;
	}
    
    //创建消费者实例
	public  KafkaConsumer<String,byte[]> generatorKafkaConsumer(boolean isAutoTask,String connectString,String groupId){
		Properties props = generationProperties(isAutoTask,connectString,groupId);
		KafkaConsumer<String,byte[]> consumerClient = new KafkaConsumer<String,byte[]>(props);
		return consumerClient;
	}
    
    
	private Properties generationProperties(boolean isAutoTask,String connectString,String groupId){
		Properties props= commonProperties(connectString);
         //消费者组设置
	     if(isAutoTask){
	    	 props.put("group.id", groupId.trim());//消费者组
	     }else{
	    	 props.put("group.id", UUID.randomUUID().toString());//消费者组
	     }
	     return props;
	}
    
	private Properties commonProperties(String connectString){
		 Properties props = new Properties();
          //注意这里并非需要设置集群中全部的broker地址，消费者会从现有的配置中查找到全部的Kafka集群成员。
         //这里设置两个以上的 broker 地址信息，当其中任意一个宕机时，消费者仍然可以连接到 Kafka 集群上。
		 props.put("bootstrap.servers", connectString.trim()); //kafka服务器信息
		 props.put("enable.auto.commit", "false"); //是否自动提交offset
	    //props.put("auto.commit.interval.ms", "1000"); //每隔多久自动提交一次
		 props.put("session.timeout.ms", "10000");//session超时时间
		//props.put("max.poll.interval.ms", "5000");//隔多久至少要poll一次
	     props.put("max.poll.records", "1000");//每次最多取多少条记录
	     props.put("offsets.storage", "zookeeper");
		//props.put("dual.commit.enabled", "true");
	     props.put("key.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer");
	     props.put("value.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer");
	     return props;
	}
}
```

订阅主题 && 拉取数据

```java
@Service
public class KafkaAutoFetchDataThread extends Thread{
	
	private Logger logger = LoggerFactory.getLogger(KafkaAutoFetchDataThread.class);
	@Autowired
	private SysParamService sysParamService;
	@Autowired
	private IndexConfigRepository repository;
	private  KafkaConsumer<String,byte[]> consumerClient =null;
	private UdpSerializer serializer = new UdpSerializerImpl();
	private int commitIntervalms = 1000 * 10; //提交偏移量间隔
	private long lastCommitTimestampe = 0; //最近提交offset时间
	private long oldOffsetMapCRC32Value=0L;//旧偏移量的crc32值
	public static volatile boolean isChangeTopic = false; //是否有topic改变
	@Override
	public void run() {
		try{
			initClient();
			fetchData();
		}catch(Exception e){
			logger.error("KafkaHighLevelConsumer-->run,e={}",e);
		}
	}
	/**
	 * 初始化客户端
	 * @throws InterruptedException
	 */
	private void initClient() throws InterruptedException{
		String kafkaConnectString = sysParamService.getParamValue(ETLConstants.KAFKA_CONNECT_STRING,"");
		String groupId = sysParamService.getParamValue(ETLConstants.KAFKA_GROUP_ID,"");
		consumerClient = KafkaConsumerFactory.build().generatorKafkaConsumer(true, kafkaConnectString, groupId);
    	subscribeTopic();
	}
	/**
	 *监听队列
	 * @throws InterruptedException
	 */
	private void subscribeTopic() throws InterruptedException{
		Set<String> topicNames =  KafkaBuffer.getTopicNames();
	   	 while(true){
	   		 if(topicNames !=null && !topicNames.isEmpty()){
                  //一次性订阅全部主题
	   			 consumerClient.subscribe(topicNames);
	   			 return;
	   		 }else{
	   			 logger.info("KafkaAutoFetchDataThread-->subscribeTopic 暂时没有要监听的队列信息");
	   			Thread.sleep(15000);
	   			topicNames =  KafkaBuffer.getTopicNames();
	   		 }
	   	 }
	}
	/**
	 * 重新监听队列，先提交现在的偏移量然后再重新监听
	 * @throws InterruptedException
	 */
	private void resetTopic() throws InterruptedException{
		logger.info("KafkaAutoFetchDataThread-->resetTopic 发现有队列变更，重新监听所有队列");
		commitOffset(KafkaBuffer.getTopicPartitionMap());
		consumerClient.close();
		initClient();
		KafkaAutoFetchDataThread.isChangeTopic=false;
	}
	/**
	 * 获取kafka数据
	 */
	private void fetchData(){
		this.lastCommitTimestampe = System.currentTimeMillis();
		while (true) {
			try{
				if(isChangeTopic){//当定时器检查到数据库中有新增主题时
                      //触发重新初始化一个新的消费者来监听新的全部队列
					resetTopic();
				}
		        ConsumerRecords<String,byte[]> records = consumerClient.poll(100);
		        if(records!=null && !records.isEmpty()){
                      //按照分区来消费消息,在手动提交偏移量时推荐使用按照分区来消费消息
		        	 for(Iterator<TopicPartition>iterator=records.partitions().iterator();iterator.hasNext();){
		        		 TopicPartition topicPartition=  iterator.next();
	        			 List<ConsumerRecord<String, byte[]>> partitionRecordLists=records.records(topicPartition);
	        			 if(partitionRecordLists!=null && !partitionRecordLists.isEmpty()){
	        				 for (ConsumerRecord<String,byte[]> record : partitionRecordLists){
	    			        	 BinlogEventInfo info = serializer.deserialize(record.value());
	    			        	 String infoStr = JSON.toJSONString(info);
	    			        	 logger.info("partition ={}, offset ={}, key ={}, value ={}", record.partition(),record.offset(), record.key(), infoStr);
	    			        	 Data data = new Data();
                                   //自定义实体类,主要记录这几个属性值
	    			        	 TopicPartitionOffset topicPartitionOffset=new TopicPartitionOffset(
                                     	    topicPartition.topic(),
                                            topicPartition.partition(),
	    			        			 record.offset()+1,true);
	    			        	 data.setIdentity(topicPartitionOffset);
	    			        	 data.setData(Arrays.asList(info));
                                   //这里只是将消息保存到缓存队列中,并将数据与offset关联
                                  //待最终消费完数据再放进KafkaBuffer的map中(ConcurrentMap<String, OffsetAndMetadata>,key为主题 + 分区)
	    			        	 KafkaBuffer.put(data);
	    		        	 }  
	        			 }
		        	 }
		        }
		         //处理偏移量
		        processOffset();
			}catch(Exception e){
				logger.error("KafkaHighLevelConsumer-->fetchData 发生错误e={}",e);
			}
		}
	}
}


public class TopicPartitionOffset {

	private int hash = 0;
    private final int partition;
    private final String topic;
    private long offset;
    private boolean wantCommit = false;
    
    //...
}    

```

提交偏移量

```java
//只展示偏移量相关方法
@Service
public class KafkaAutoFetchDataThread extends Thread{
	
	private Logger logger = LoggerFactory.getLogger(KafkaAutoFetchDataThread.class);
	@Autowired
	private SysParamService sysParamService;
	@Autowired
	private IndexConfigRepository repository;
	private  KafkaConsumer<String,byte[]> consumerClient =null;
	private UdpSerializer serializer = new UdpSerializerImpl();
	private int commitIntervalms = 1000 * 10; //提交偏移量间隔
	private long lastCommitTimestampe = 0; //最近提交offset时间
	private long oldOffsetMapCRC32Value=0L;//旧偏移量的crc32值
	public static volatile boolean isChangeTopic = false; //是否有topic改变
	
	/**
	 * 处理偏移量
	 */
	private void processOffset(){
		ConcurrentMap<String, OffsetAndMetadata>  topicPartitionMap = KafkaBuffer.getTopicPartitionMap();
		if(isAbleCommit(topicPartitionMap)){
			commitOffset(topicPartitionMap);
		}
	}
    
	/**
	 * 根据上次提交时间差以及是否有记录提交来判断是否能提交偏移量
	 */
	private boolean isAbleCommit(ConcurrentMap<String, OffsetAndMetadata>  topicPartitionMap){
		long diff = System.currentTimeMillis()-lastCommitTimestampe;
		if( !topicPartitionMap.isEmpty() && diff > commitIntervalms){
			if(offsetIsUpdate(topicPartitionMap)){
				return true;
			}
		}
		return false;
	}
        
	/**
	 * 提交偏移量
	 */
	private void commitOffset(ConcurrentMap<String, OffsetAndMetadata>  topicPartitionMap){
		Map<TopicPartition, OffsetAndMetadata> offsetMap = new HashMap<TopicPartition, OffsetAndMetadata>();
		Set<Map.Entry<String, OffsetAndMetadata>> entrySet = topicPartitionMap.entrySet();
		Iterator<Map.Entry<String, OffsetAndMetadata>> entryIt = entrySet.iterator();
		while(entryIt.hasNext()){
			Map.Entry<String, OffsetAndMetadata> entry = entryIt.next();
			String key = entry.getKey();
			String str[] = key.split(ETLConstants.TOPIC_PARTITION_SPLIT_CHAR);
			String topic = str[0];
			int partition = Integer.parseInt(str[1]);
			OffsetAndMetadata offsetAndMetadata = entry.getValue();;
			TopicPartitionOffset offsetDbRecord = new TopicPartitionOffset(topic,partition,offsetAndMetadata.offset(),true);
			offsetMap.put(new TopicPartition(topic,partition),offsetAndMetadata);
			int reuslt = repository.updatePartitionOffset(offsetDbRecord);
			if(reuslt<1){
				repository.insertPartitionOffset(offsetDbRecord);
			}
		}
		//提交到kafka
		consumerClient.commitSync(offsetMap);
		this.lastCommitTimestampe=System.currentTimeMillis();
	}
    
	/**
	 * 根据offset的crc32来判断是否有需要更新偏移量
	 */
	public boolean offsetIsUpdate(ConcurrentMap<String, OffsetAndMetadata>  topicPartitionMap){
		boolean result = false;
		long newCrc32Value = crc32Value(topicPartitionMap);
		if(this.oldOffsetMapCRC32Value!=newCrc32Value){
			result = true;
			this.oldOffsetMapCRC32Value = newCrc32Value;
		}
		return result;
	}
    
	/**
	 * 计算crc32值
	 */
	public long crc32Value(Object obj){
		ByteArrayOutputStream  bos= null;
		ObjectOutputStream oos = null;
		long newCrc32Value =0L;
		try {
			if(obj!=null){
				CRC32 crc32 = new CRC32();
				bos = new ByteArrayOutputStream();
				oos = new ObjectOutputStream(bos);
			    oos.writeObject(obj);
			    byte [] offsetBytes = bos.toByteArray();
			    crc32.update(offsetBytes);
			    newCrc32Value = crc32.getValue();
			}
		} catch (IOException e) {
			logger.error("KafkaFetchDataThread-->crc32Value,发生错误e={}",e);
		}finally{
			try{
				if(oos!=null)
					oos.close();
				if(bos!=null)
					bos.close();
			}catch(Exception e){
				logger.error("KafkaFetchDataThread-->crc32Value 关闭流错误，e={}",e);
			}
		}
		return newCrc32Value;
	}
}

```

