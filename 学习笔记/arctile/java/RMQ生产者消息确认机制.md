## RMQ生产者消息确认机制

#### 消息可靠性

若想保证消息的可靠性,需对消息进行持久化,而RMQ的消息持久化需以下三个方面都持久化

+ exchange
+ queue
+ messages

因此,需保证消息可达需让消息到达exchange而且能路由到queue,而RMQ的生产者api是没保证消息可达的,因此需使用以下方式来确保

1. 通过AMQP提供的事务机制实现；

2. 使用发送者确认模式实现；

   

   

#### 使用事务

基于以下三个方法来确保消息可达

1. channel.txSelect()声明启动事务模式；
2. channel.txComment()提交事务；
3. channel.txRollback()回滚事务；

```java
ConnectionFactory factory = new ConnectionFactory();
//...设置参数
Connection conn = factory.newConnection();

Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(_queueName, true, false, false, null);
String message = String.format("消息时间 => %s", new Date().getTime());
try {
    channel.txSelect(); // 声明事务
    // 发送消息
    channel.basicPublish("", _queueName, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
    channel.txCommit(); // 提交事务
} catch (Exception e) {
    channel.txRollback();
} finally {
    channel.close();
    conn.close();
}
```

注意事务模式性能非常差,因此**不建议使用**

#### conform确认模式

confirm实现方式

+ 方式一：channel.waitForConfirms()普通发送方确认模式；

+ 方式二：channel.waitForConfirmsOrDie()批量确认模式；

+ 方式三：channel.addConfirmListener()异步监听发送方确认模式；

**方式一 - 普通confirm模式**

只需要在推送消息之前，channel.confirmSelect()声明开启发送方确认模式，再使用channel.waitForConfirms()等待消息被服务器确认即可

```java
ConnectionFactory factory = new ConnectionFactory();
Connection conn = factory.newConnection();

Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
String message = String.format("消息时间 => %s", new Date().getTime());
channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
if (channel.waitForConfirms()) {
    System.out.println("消息发送成功" );
}
```

**方式二 - 批量confirm模式**

channel.waitForConfirmsOrDie()，使用同步方式等所有的消息发送之后才会执行后面代码，只要有一个消息未被确认就会抛出IOException异常

```java
ConnectionFactory factory = new ConnectionFactory();
Connection conn = factory.newConnection();

Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
    String message = String.format("消息时间 => %s", new Date().getTime());
    channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
channel.waitForConfirmsOrDie(); //直到所有信息都发布，只要有一个未确认就会IOException
System.out.println("全部执行完成");
```

**方式三 - 异步conform模式**

```java
ConnectionFactory factory = new ConnectionFactory();
Connection conn = factory.newConnection();

Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
    String message = String.format("消息时间 => %s", new Date().getTime());
    channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
//异步监听确认和未确认的消息
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("未确认消息，标识：" + deliveryTag);
    }
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        System.out.println(String.format("已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
    }
});
```



#### 基于Spring template的Callback方式

RMQ主要有两个针对发送消息后的回调接口`ConfirmCallback`和`ReturnCallback`两个接口

+ ConfirmCallback：当消息成功到达exchange的时候触发的ack回调。
+ ReturnCallback：当消息成功到达exchange，但是没有队列与之绑定的时候触发的ack回调。发生网络分区会出现这种情况。

生产者端使用ConfirmCallback和ReturnCallback回调机制，最大限度的保证消息不丢失

常见的XML配置

```xml
<!-- 给模板指定转换器 --><!-- mandatory必须设置true,return callback才生效 -->
<!--returnCallBackListener publisher发送到exchange后，异步收到失败消息 -->
<!--confirmCallBackListener publisher发送消息到exchange,queue,consumer收到消息后才会收到异步收到消息 -->
<rabbit:template id="amqpTemplate" connection-factory="connectionFactory" exchange="#.exchange"
                 return-callback="returnCallBackListener"
                 confirm-callback="${callback}"
                 mandatory="true"
        />
```



