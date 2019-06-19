## Kafka消费者特性

#### 再均衡

再均衡是指分区的所属权从一个消费者转移到另一消费者的行为，它为消费组具备高可用性和伸缩性提供保障，使我们可以既方便又安全地删除消费组内的消费者或往消费组内添加消费者。不过在再均衡发生期间，消费组内的消费者是无法读取消息的。也就是说，在再均衡发生期间的这一小段时间内，消费组会变得不可用。

再均衡监听器用来设定发生再均衡动作前后的一些准备或收尾的动作。ConsumerRebalanceListener 是一个接口，包含2个方法，具体的释义如下：

```java
//这个方法会在再均衡开始之前和消费者停止读取消息之后被调用。
//可以通过这个回调方法来处理消费位移的提交，以此来避免一些不必要的重复消费现象的发生。
//参数 partitions 表示再均衡前所分配到的分区。
void onPartitionsRevoked(Collection partitions);
//这个方法会在重新分配分区之后和消费者开始读取消费之前被调用。
//参数 partitions 表示再均衡后所分配到的分区。
void onPartitionsAssigned(Collection partitions); 

//配合外部存储使用的再平衡监听器
consumer.subscribe(Arrays.asList(topic), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        //store offset in DB （storeOffsetToDB）
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        for(TopicPartition tp: partitions){
            consumer.seek(tp, getOffsetFromDB(tp));//从DB中读取消费位移
        }
    }
});
```



#### 消费者拦截器

消费者拦截器主要在消费到消息或在提交消费位移时进行一些定制化的操作。

ConsumerInterceptor接口

```java
//在 poll() 方法返回之前调用拦截器的 onConsume() 方法来对消息进行相应的定制化操作
//比如修改返回的消息内容、按照某种规则过滤消息（可能会减少 poll() 方法返回的消息的个数）。
//如果 onConsume() 方法中抛出异常，那么会被捕获并记录到日志中，但是异常不会再向上传递。
public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records)；

//在提交完消费位移之后调用拦截器的 onCommit() 方法
//使用这个方法来记录跟踪所提交的位移信息
//比如当消费者使用commitSync的无参方法时，我们不知道提交的消费位移的具体细节，而使用拦截器的onCommit()方法却可以做到这一点
public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets)；
public void close();
```

自定义消息过滤,将超过10s的消息忽略掉

```java
public class ConsumerInterceptorTTL implements ConsumerInterceptor<String, String> {
    private static final long EXPIRE_INTERVAL = 10 * 1000;

    @Override
    public ConsumerRecords<String, String> onConsume(ConsumerRecords<String, String> records) {
        long now = System.currentTimeMillis();
        Map<TopicPartition, List<ConsumerRecord<String, String>>> newRecords = new HashMap<>();
        for (TopicPartition tp : records.partitions()) {
            List<ConsumerRecord<String, String>> tpRecords = records.records(tp);
            List<ConsumerRecord<String, String>> newTpRecords = new ArrayList<>();
            for (ConsumerRecord<String, String> record : tpRecords) {
                //判断消息的在有效期内,在使用ProducerRecord时可指定定义消息的时间戳
                if (now - record.timestamp() < EXPIRE_INTERVAL) {
                    newTpRecords.add(record);
                }
            }
            if (!newTpRecords.isEmpty()) {
                newRecords.put(tp, newTpRecords);
            }
        }
        return new ConsumerRecords<>(newRecords);
    }

    @Override
    public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets) {
        offsets.forEach((tp, offset) -> 
                System.out.println(tp + ":" + offset.offset()));
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

自定义消费者拦截器配置

```java
props.put(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
          ConsumerInterceptorTTL.class.getName());
```



#### 消费者特别参数

`exclude.internal.topics`

Kafka 中有两个内部的主题： __consumer_offsets 和 __transaction_state。exclude.internal.topics 用来指定 Kafka 中的内部主题是否可以向消费者公开，默认值为 true。如果设置为 true，那么只能使用 subscribe(Collection)的方式而不能使用 subscribe(Pattern)的方式来订阅内部主题，设置为 false 则没有这个限制。

`max.poll.records`

这个参数用来配置 Consumer 在一次拉取请求中拉取的最大消息数，默认值为500（条）。如果消息的大小都比较小，则可以适当调大这个参数值来提升一定的消费速度。