## ZooKeeper学习笔记

### 定义

ZooKeeper旨在减轻构建健壮的分布式系统的任务。Zookeeper的主线是在分布式系统中协作多个任务。

ZooKeeper使用共享存储模型来实现应用间的协作和同步原语。

### CAP

一致性（Consistency）、可用性（Availability）和分区容错性（Partition-tolerance）,没有一个系统能同时满足这三个属性.

ZooKeeper的设计尽可能满足一致性和可用性，当然，在发生网络分区时ZooKeeper也提供了只读能力。

### zk集群

zk集群使用`超半数`的方式来避免脑裂.即只有获取超半数的投票才能成为leader.

服务器使用内部协议来保持客户端之间状态的同步，对客户端呈现一致性视图。

#### 写操作流程

![https://imgchr.com/i/mQLJzj](http://ww1.sinaimg.cn/large/8bb38904ly1g644c1rg3tj209x0dc0w2.jpg)

#### 集群角色 

+ 群首(leader)
+ 追随者(follower) - 需参与选主以及投票,提供读能力 
+ 观察者(observer) - 不参与选主或投票,只提供读功能,zk后期提供的提供扩展性的角色

zk集群中有一个节点,作为中心点处理所有对ZooKeeper系统变更的请求,我们称之为群首(leader).它就像一个定序器，建立了所有对ZooKeeper状态的更新的顺序，追随者接收群首所发出更新操作请求，并对这些请求进行处理，以此来保障状态更新操作不会发生碰撞。



### 节点类型

+ 临时节点
+ 持久节点
+ 持久有序节点
+ 临时有序节点

### 通知机制

zk的监听通知是单次的,意味着在接收到监听后你需重新监听该节点.

通知机制的一个重要保障是，对同一个znode的操作，先向客户端传送通知，然后再对该节点进行变更。

zk本身提供了保序性(ordering guarantee)：即客户端只有首先看到了监视事件后，才会感知到它所设置监视的 znode 发生了变化(a client will never see a change for which it has set a watch until it first sees the watch event). 

当与一个服务器失去连接的时候，是无法接收到 watch 的。而当 client 重新连接时，如果需要的话，所有先前注册过的 watch都会被重新注册。通常这是完全透明的。只有在一个特殊情况下，watch 可能会丢失：对于一个未创建的 znode 的 exist watch，如果在客户端断开连接期间被创建了，并且随后在客户端连接上之前又删除了，这种情况下，这个 watch 事件可能会被丢失。

单次触发可能会丢失事件,但因为任何在接收通知与注册新监视点之间的变化情况，均可以通过读取ZooKeeper的状态信息来获得。所以丢失事件通知通常不是问题.

ZooKeeper的API中的所有读操作：getData、getChildren和exists，均可以选择在读取的znode节点上设置监视点。使用监视点机制，我们需要实现Watcher接口类，实现其中的process方法,

```java
public void process(WatchedEvent event);
```

**WatchedEvent数据结构包括以下信息：**

ZooKeeper会话状态（KeeperState）

+ Disconnected
+ SyncConnected
+ AuthFailed
+ ConnectedReadOnly
+ SaslAuthenticated
+ Expired

事件类型（EventType）

+ NodeCreated
+ NodeDeleted
+ NodeDataChanged
+ NodeChildrenChanged
+ None - 表示无事件发生，而是ZooKeeper的会话状态发生了变化。

监视点一旦被设置即无法被删除,只有触发这个监视点或者会话过期/关闭才能移除这个监视点.

### 解决观察到某节点下全部子节点部分更新的问题

如我们想在/config下更新其下所有子节点的某个值配置,而又不想让客户端只观察到部分子节点的变化.想原子性的观察到其所有子节点的变化,则只需添加一个额外的节点如/configProcessing,在操作config子节点前先创建一个/configProcessing,然后在读取config全部子节点配置时先判断/configProcessing是否存在,存在就不读相应的配置.而当/configProcessing节点的删除通知到了时,即可一次性读取全部的配置节点信息.



### 乐观锁机制

每一个znode都有一个版本号，它随着每次数据变化而自增。两个API操作可以有条件地执行：setData和delete。

当以上方法的版本号与zk服务器一致时才能调用成功.



### 会话

#### 定义

在对ZooKeeper集合执行任何请求前，一个客户端必须先与服务建立会话。当一个会话因某种原因而中止时，在这个会话期间创建的临时节点将会消失。

客户端通过**TCP协议**与服务器进行连接并通信，但当会话无法与当前连接的服务器继续通信时，会话就可能转移到另一
个服务器上。ZooKeeper客户端库透明地转移一个会话到不同的服务器。

会话提供了顺序保障，这就意味着同一个会话中的请求会以FIFO（先进先出）顺序执行。即使是同一个客户端的连续两个不同会话,客户端请求在这种不同会话情况下也不一定能保证FIFO顺序.

#### 会话状态

+ CONNECTING
+ CONNECTED
+ CLOSED
+ NOT_CONNECTED

![https://imgchr.com/i/mnXuDK](http://ww1.sinaimg.cn/large/8bb38904ly1g62og3yvtbj215s0f1juv.jpg)

#### 会话超时时间

`会话超时时间`设置了ZooKeeper服务器允许会话被声明为超时之前存活的时间。如果经过时间t之后服务接收不到这个会话的任何消息，服务就会声明会话过期。

客户端在经过t/3的时间未收到任何消息,客户端则会主动发送心跳信息.在经过2t/3后仍未收到任何消息时,客户端将寻找其它zk服务器来创建会话.



### 事务标志符

zk会根据每一个更新建立的顺序来分配事务标识符,如果客户端当前观察到的事务标志符为i,那如果此时因网络或其它问题需连接到其它zk服务器时,该客户端只能连接到事务标识符大于等于i的zk服务器上.

zk在实现分布式锁时,也会存在多个进程同时用有锁的情况,如某zk客户端在获取锁后执行任务时因gc时间过长导致会话超时,此时另一个客户端监听触发后重新获取到锁,此时将会有两个进程可能会更新相同的资源.解决方案时通过创建节点时zk返回的stat信息的事务标志符(czxid)来做隔离,在更新共享资源时带上该czxid做乐观锁.如下图,

![](http://ww1.sinaimg.cn/large/8bb38904ly1g63oyn361dj211n0drwhi.jpg)

###  zk客户端的连接管理

ZooKeeper客户端库会监控与服务之间的连接，客户端库不仅告诉我们连接发生问题，还会主动尝试重新建立通信。所以不要关闭会话后再启动一个新的会话，这样会增加系统负载，并导致更长时间的中断。所以当遇到`Disconnected`事件时，只需等待zk库自己重连后变成`SyncConnected`事件。

### 顺序性保证

ZooKeeper声明对一个会话中所有客户端操作提供顺序性的保障，但还是会存在ZooKeeper控制之外某些情况，可能会改变客户端操作的顺序。

+ 链接丢失时,因重试导致的本来先发的请求可能慢于后面的请求
+ 多线程提交同步操作时可能因线程调度原因破坏了顺序性
+ 同步与异步混用,如先后发现了两个Aask1,Aask2的异步请求,但在接收到Aask1未收到Aask2的回调时,在Aask1的回调中调用同步请求Sask,则异步回调Aask2需等待同步调用处理完毕后才能收到响应

