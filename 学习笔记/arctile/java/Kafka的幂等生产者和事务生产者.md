## Kafka的幂等生产者和事务生产者

### 幂等性生产者

Kafka在 0.11 之后，指定 Producer 幂等性的方法很简单，仅需要设置一个参数即可，即 

```java
props.put(“enable.idempotence”, ture);
//或
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG， true);
```

#### 特点

1.只能保证单分区上的幂等性，即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。

2.只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了。

#### 原理

**区分producer会话**
producer每次启动后，首先向broker申请一个全局唯一的pid，用来标识本次会话。

**消息检测**
message_v2 增加了sequence number字段，producer每发一批消息，seq就加1。

broker在内存维护(pid,seq)映射，收到消息后检查seq，如果，


new_seq=old_seq+1: 正常消息；

new_seq<=old_seq : 重复消息；

new_seq>old_seq+1: 消息丢失；

### 事务

Kafka 自 0.11 版本开始也提供了对事务的支持，目前主要是在 read committed 隔离级别上做事情。它能保证多条消息原子性地写入到目标分区，同时也能保证 Consumer 只能看到事务成功提交的消息。事务型 Producer 能够保证将消息原子性地写入到多个分区中。这批消息要么全部写入成功，要么全部失败.

kafka事务消息提供了多个消息原子写的保证，但它不保证原子读。比如原子写入完后一个分区挂了,则会丢失多个消息的原子性.

#### 使用

设置事务型 Producer 的方法也很简单，满足两个要求即可：

+ 和幂等性 Producer 一样，开启 enable.idempotence = true。
+ 设置 Producer 端参数 transactional. id。最好为其设置一个有意义的名字。

```java

producer.initTransactions();
try {
    //这段代码能够保证 Record1 和 Record2 被当作一个事务统一提交到 Kafka，
    //要么它们全部提交成功，要么全部写入失败。
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```

实际上即使写入失败，Kafka 也会把它们写入到底层的日志中，也就是说 Consumer 还是会看到这些消息。因此在 Consumer 端，读取事务型 Producer 发送的消息也是需要一些变更的。修改起来也很简单，设置 isolation.level 参数的值即可。当前这个参数有两个取值：

+ **read_uncommitted**：这是默认值，表明 Consumer 能够读取到 Kafka 写入的任何消息，不论事务型 Producer 提交事务还是终止事务，其写入的消息都可以读取。很显然，如果你用了事务型 Producer，那么对应的 Consumer 就不要使用这个值。
+ **read_committed**：表明 Consumer 只会读取事务型 Producer 成功提交事务写入的消息。当然了，它也能看到非事务型 Producer 写入的所有消息。

### 总结

幂等性 Producer 只能保证单分区、单会话上的消息幂等性；而事务能够保证跨分区、跨会话间的幂等性。

### 参考文章

[文章参考](https://www.jianshu.com/p/f77ade3f41fd)