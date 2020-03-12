## DB与MQ事务一致性

### 一、MQ一致性保证意义

#### 综述

微服务体系下按不同领域职能将拆分成多个服务，一个业务场景将往往涉及到多个服务的协作，协作之间如果不强调一致性：那么整个服务就不具备业务完整性、也无法跟踪定位就上述服务之间的协作（通信方式）主要就**划分为RPC和MQ**，**前者对应同步方式、后者对应异步方式：**上述的同步/异步也不是纯粹的，RPC也能支持异步、而MQ的投递也跟同步有关（跟MQ的确认机制有关）

#### MQ一致性

+ **发送消息**：需确保业务操作和发送消息是一致的——不能出现业务操作成功消息未发出或者消息发出了但是业务并没有成功的情况
+ **消费消息**：需确保接收消息和业务处理是一致的——消息可以重复消费但是业务处理需要保证幂等（MQ中间件确保至少投递一次）

#### 场景举例

+ 在支付场景中，支付成功后我们需要插入一条支付流水，需要发送一条支付完成的消息通知其他系统
+ 在支付回调订单中，订单支付回调完成后、产生一笔消息驱动订单进行拆单（拆单后也是通过消息驱动审单）
+ 用户服务完成了注册动作，向短信服务推送一条营销相关的消息。
+ 支付验证完毕异步发送短信流程

### 二、MQ一致性处理思路

执行方案 : 先本地数据库事务,再发送MQ

1.数据库先保存待发送的MQ消息记录

2.通过事件总线触发发送消息(如spring event bus)

3.监听消息投递状况,若投递成功则将数据库的消息状态改为成功,并删除数据库消息记录

4.定时任务扫描一个小时内的待同步消息来重新发送

![QQ截图20200208100227.png](http://ww1.sinaimg.cn/large/8bb38904gy1gborjoojlqj20n60cc0u7.jpg)

### 三、事务同步器

为了把保存**待发送的事务消息**和**发送消息到RabbitMQ**两个动作从调用方感知角度合并为一个动作，可使用Spring特有的事务同步器`TransactionSynchronization`，调用流程如下,

![QQ截图20200208103954.png](http://ww1.sinaimg.cn/large/8bb38904gy1gbosmupyf2j20tx0jagt5.jpg)

上图仅仅演示了事务正确提交的场景（不包含异常的场景）。这里可以明确知道，事务同步器TransactionSynchronization的afterCommit()和afterCompletion(int status)方法都在真正的事务提交点AbstractPlatformTransactionManager#doCommit()之后回调，因此可以选用这两个方法其中之一用于执行推送消息到RabbitMQ服务端，整体的伪代码如下：

```java
@Transactional
public Dto businessMethod(){
    //业务代码
    saveBussinessDbRecord();
    // 保存事务消息
    [saveTransactionMessageRecord()]
    // 注册事务同步器 - 在afterCommit()方法中推送消息到RabbitMQ
    //[register TransactionSynchronization,send message in method afterCommit()]
    // 兼容无论是否有有事务
    if(TransactionSynchronizationManager.isActualTransactionActive()) {
        // 注册事务同步器
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
             @Override
                public void afterCommit() {
                    //发布spring event bus事件触发发送消息
                }
        });
    }else{
        //发布spring event bus事件触发发送消息
    }     
}
```

