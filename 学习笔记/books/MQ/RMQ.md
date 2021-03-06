《RabbitMQ实战》读书笔记
=

RMQ是什么
-
>RQM是一个开源的消息代理和队列服务器,用来通过普通协议在完全不同的应用之间共享数据,或者简单地将作业排队以便分布式服务器进行处理

概念
-
>生产者

      生产者负责创建消息,然后发布(发送)到代理服务器(RMQ)
  
>消息
    
      消息包含两部分 : 有效载荷(payload)和标签(label)
      有效载荷就是你想要传输的数据
      标签描述了有效载荷,并且RMQ用它来决定谁将获得消息的拷贝(交换器名称和可选的主题标记)

>消费者
    
      连接了代理服务器,并订阅到队列.
      消费者接收到消息时,只能接收到消息的有效载荷
      
      ☆当RMQ队列有多个消费者时,队列收到的消息将以循环的方式发送到消费者.每条消息只会发送给一个订阅的消费者
      ☆如果消费者接收到一条消息,然后确认前冲RMQ断开连接,RMQ会认为这条消息木有分发,然后重新分发给下一个订阅的消费者
      ☆RMQ给同一个消费者发送下一条消息的前提是,此消费者上一条消息已经应答了
    
    
>信道
     
     信道是建立在"真实的"TCP连接内的虚拟连接.AMQP命令都是通过信道发送出去的.
     使用信道发送消息而不是直接通过TCP发送AMQP的原因 : 操作系统内建立和销毁TCP会话是非常昂贵的开销.


>AMQP消息路由的组成     
+ 交换器
+ 队列
+ 绑定
  
  
>交换器的类型
+ direct : 如果路由键匹配的话,消息就被投递到对应的队列.默认交换器类型就是direct类型. 
+ fanout : 当你发送一条消息到fanout交换器时,它会把消息投递给所有附加在此交换器上的队列,广播类型的交换器.
+ topic  : 一种使用类似正则匹配的交换器,匹配类型就发送给该队列.匹配规则,使用"."作为分隔符,使用"*"匹配特定位置的任意文本,使用"#"匹配所有规则
+ headers(基本不会用到,用来匹配AMQP的header而非路由键.其它跟direct差不多,但性能差,)


>队列绑定到交换器上,消息发送到交换器上,RMQ会根据交换器类型以及路由键决定将消息投递到哪个队列上


>vhost : 虚拟主机.vhost是AMQP的基础概念,必须在连接时进行指定.RMQ默认的vhost为"/".

>默认情况下交换器,队列,消息都是非持久化的

>消息持久化的必要条件,以下条件缺一不可
+ 交换器是持久化的
+ 队列是持久化的
+ 消息时持久化的



编码与模式
-

>使用RMQ实现RPC
    
    当使用RMQ实现RPC时,客户端只是简单地发布消息而已.RMQ会负责使用绑定来路由消息到合适的队列, PRC服务器会从这些队列上消费消息
    RMQ替你处理了对多台RPC服务器消息的负载均衡,RPC服务器崩溃时得重发送消息处理
    客户端发送消息时,使用AMQP消息头里面的"reply_to"字段来确定队列名称以及监听队列等待应答.
    然后接收消息的RPC服务器能检查"reply_to"字段,并创建包含应答内容的新的消息,并以队列名称作为路由键.
    所有RPC客户端需要做的就是声明临时的,排他的,匿名队列,并将该队列名称包含到RPC消息的"reply_to"头中.
   

RMQ集群
-
  
>RMQ内建集群的设计用于完成两个目标 :
+ 允许消费者和生产者在RMQ节点崩溃时继续运行
+ 可通过添加更多的节点来线性扩展消息通信的吞吐量
  

>集群队列数据

    当多个节点组成集群时,并非每一个节点都有所有队列的完全拷贝.
    在集群中创建队列的话.集群只会在单个节点而不是所有节点上创建完整的队列消息(元数据,状态,内容).
    结果只是队列的所有者节点知道有关队列的所有消息,所有其他非所有者节点只知道队列的元数据和指向该队列存在的那个节点的指针.
    因此当集群节点崩溃时,该节点的队列和关联的绑定就都消失了.附加在那些队列上的消费者丢失了其消息,并且任何匹配该队列绑定信息的新消息也都丢失了.
    想要该指定队列重回集群的唯一方式是恢复故障节点.
    如果集群中其他节点尝试创建一个持久化的该队列,会返回404错误.除非新声明的队列为非持久化的.
    
    
>集群数据不分布在所有节点的原因
+ 存储空间 - 如果每个集群节点都拥有所有队列的完整拷贝,那么添加新的节点不会给你带来更多存储空间
+ 性能     -  消息的发布需要将消息复制到每一个集群节点.对于持久化消息,每一条消息都会触发磁盘活动.每次新增节点,网络和磁盘负载都会增加,最终只能保持集群性能的稳定.    


>集群交换器数据
集群中的每一个节点都拥有每个交换器的所有消息

>内存节点&磁盘节点
    
    RMQ只要求集群中至少有一个磁盘节点,所有其他节点可以是内存节点.
    当节点加入或离开集群时,它们必须要将该变更通知到至少一个磁盘节点.
    如果集群中所有磁盘节点崩溃了,那么集群可以继续路由消息,但不能创建队列,创建交换器,创建绑定,添加用户等更改.
    
    
>镜像队列

    RabbitMQ集群是由多个broker节点构成的，那么从服务的整体可用性上来讲，该集群对于单点失效是有弹性的，
    但是同时也需要注意：尽管exchange和binding能够在单点失效问题上幸免于难，但是queue和其上持有的message却不行，
    这是因为queue及其内容仅仅存储于单个节点之上，所以一个节点的失效表现为其对应的queue不可用.
    
    引入RabbitMQ的镜像队列机制，将queue镜像到cluster中其他的节点之上。
    在该实现下，如果集群中的一个节点失效了，queue能自动地切换到镜像中的另一个节点以保证服务的可用性.
    
    参考链接
    https://blog.csdn.net/u013256816/article/details/71097186
    
    
故障恢复
-

>在检查到故障并进行重连之后的首要任务是构造交换器,队列和绑定.

>生产者只是短暂运作,所以不需要任何花哨的故障处理代码.这是因为生产者的每次调用都会重新建立新的连接.

>另一种集群模式
+ warren 主备模式 + 负载均衡
+ shovel 队列内容复制插件


性能
-

>速度的影响因素
+ 消息是否持久化
+ ack消息确认
+ 路由算法和绑定规则

>RMQSSL配置

    非对象性加密中公钥的作用
      * 解密私钥加密的数据
      * 验证消息签名
      
      