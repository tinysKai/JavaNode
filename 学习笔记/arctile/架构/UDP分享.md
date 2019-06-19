# UDP分享

## UDP模块定义

- udp-admin(后台)
- udp-center(控制中心)
- udp(抽数工作者)
- udp-etl(数据清洗中心)
- udp-api



## 逻辑架构图

![](https://s2.ax1x.com/2019/05/20/Ex3hGj.png)

## 总体流程

![](https://s2.ax1x.com/2019/05/20/Ex8PeK.png)





## UDP-ADMIN

后台管理系统,主要菜单如下

![](https://s2.ax1x.com/2019/05/20/ExGdgA.png)

主要功能 :

+ 添加监听库表
+ 索引配置
+ 数据与索引关联
+ 事件查询
+ center状态查询
+ 权限控制





##  UDP-CENTER

+ 任务管理
  + 任务下发(/UDP/INSTANCE/TASK)
  + 任务自平衡
+ 工作者(UDP)管理
+ 选主(基于Curator的LeaderSelector)
+ 记录UDP上送事件
+ 重试机制
  + 任务重试
  + MQ渠道探测



#### 工作者管理

基于zk的临时节点

center当选主成功后会监听`/UDP/INSTANCE`子节点的变化

udp启动时会在`/UDP/INSTANCE/`路径下创建节点,如`/UDP/INSTANCE/192.168.1.1`

节点数据内容如` { version:"节点的版本号", ip : "节点ip"}`

分析

+ udp挂了的情况,zk节点会丢失,从而触发center将该工作者的任务转移
+ udp启动则触发添加工作者,成为`新工作者`甚至有可能触发自平衡事件
+ center重启不影响工作者状态,新主分配任务也是基于`/UDP/INSTANCE`下来分发的



#### 任务管理

基于zk的临时节点,不使用其它的方案如下 : 

+ 不远程调用下游的原因是需基于center自己控制任务平衡,而远程调用是随机调用下游.

+ 不使用永久节点,让任务的下发只跟udp有关而跟center选主无关的原因是因为若只是udp挂掉,确实可以通过center将任务重新转移到其它udp实例上,或者center挂掉重新选主,,udp的数据监听依旧有效(上送不了binlog数据),但如果udp和center同时挂掉,那么任务的重新派发的触发时机将无法确定.

udp会监听`/UDP/TASK/UDP_IP`下的数据节点,如`/UDP/TASK/192.168.1.1/host_port`,从而实现任务的监听

center则会按照较平均的方式将任务信息写入到`/UDP/TASK/UDP_IP`下

基于zk的模式下发的任务在大多数情况下都是木有问题的,但是当出现因网络抖动时,udp在Instance节点的临时节点可能会失效,但udp实例却还是在运行中,而此时center检测到udp注册的zk节点失效,将其任务转移到其它udp实例上,故此时将会有两个udp实例同时监听同一个mysql实例的场景.而且即使udp后面因zk恢复了,也会因为无重新启动而没写入Instance节点而无法接收新的任务...//TODO

增量节点数据

```json
{
    "serverName": "mysql5_7", 
    "password": "###", 
    "host": "10.199.245.60", 
    "port": "3306", 
    "userName": "root", 
    "type": "全量/增量", 
    "taskId": "10.199.245.60_3306", 
    "status": "initial", 
    "binlogFileName": "mysql-bin.000017", 
    "binlogPosition": "194", 
    "gtidSet": "e59cde60-542f-11e8-867e-fa163e98d8af:1-16814170"
}
```

全量节点数据

```json
{
    "databaseName": "mytest", 
    "fullScaleDeliver": { //全量的下游信息
        "routingKey": "*", 
        "sendTopic": "queue.beifu.udp", 
        "sendType": "vms_rabbitmq", //如果这里是etl,则其它参数为空
        "vmsChannel": "channel.beifu.udp"
    }, 
    "host": "10.199.245.60", 
    "password": "###", 
    "port": "3306", 
    "status": "initial", 
    "tableName": "t2", 
    "taskId": "20190523104404f481d1ff4da045", 
    "userName": "root", 
    "where": ""
}
```



#### 表格监听管理

单个zk节点可保存单个mysql实例的所有监听表格以及下游信息

此节点目前采用的是临时节点,但使用了curator的封装类,变成了临时永久节点(网络断开后会重新写一次数据),但最终确认了此节点可直接使用永久节点,每次center启动时删除该节点重新下发即可.即不需担心网络断了导致的数据丢失问题.

节点路径如`/UDP/MAPPING/10.199.245.60_3306`,节点内容如下,

```json
[
    {
        "sendType": "rabbitmq", 
        "sendTopic": "queue.beifu.udp", 
        "routingKey": "queue.beifu.udp", 
        "vmsChannel": "", 
        "databaseName": "mytest", 
        "tableName": "table1"
    }, 
    {
        "sendType": "vms_rabbitmq", 
        "sendTopic": "queue.beifu.udp", 
        "routingKey": "*", 
        "vmsChannel": "channel.beifu.udp", 
        "databaseName": "mytest", 
        "tableName": "table2"
    }
]
```

center选主成功后会下发Mapping节点信息,所有的`UDP`实例会监听到所有的数据库实例的Mapping信息.

udp在启动时会去重新拉取Mapping节点信息,并且监听mapping信息的变化(`TreeCacheListener`**监听会首先拉取该节点的所有数据,触发了一堆ADD事件**)

```java
public class DeliveryManager implements TreeCacheListener {
    @PostConstruct
	public void init() {
		this.zookeeperManager.addChildWatcher(this.mappingPath, this);
	}
      
    @Override
    public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
        if (!this.acceptEventSet.contains(event.getType())) {
            //非TreeCacheEvent.Type.NODE_ADDED,NODE_REMOVED,NODE_UPDATED事件拒绝
            return;
        }
        String[] paths = event.getData().getPath().split("/");
        if (event.getData().getPath().startsWith(this.mappingPath) && paths.length == 5) {
            String identity = paths[4];
            if (event.getType().equals(TreeCacheEvent.Type.NODE_UPDATED) || event.getType() .equals(TreeCacheEvent.Type.NODE_ADDED)) {
                List<Map> list = JsonUtils.fromJsonArray(new String(event.getData().getData(), "UTF-8"), Map.class);
                modifyFilterRule(identity, list);
            } else if (event.getType().equals(TreeCacheEvent.Type.NODE_REMOVED)) {
                removeFilterRule(identity);
            }
        }
    }
}	

```



#### 自平衡

平衡维度是基于mysql实例数的.没基于数据流量

策略

+ 每次分发任务时选取任务数最少的UDP实例
  + 基于ZK的TASK节点的任务数(新启动的UDP比较特殊,可能会在TASK节点无子节点,直接将任务给它)
+ UDP的上下线若触发阈值则会相应的转移任务
  + 上线时将其它机器的任务转移到新上线的机器(筛选任务过多,任务过少的工作者,然后将任务较平均分配)
  + 下线时将该机器的任务转移到其它机器(先保存下线机器的任务再执行任务分配)



#### 选主

启动center时会去进行竞争主尝试,原理是在竞争抢"LOCK"路径下的节点,抢占成功则写入Master节点,不管成功失败都回调,然后都监听Master子节点的变化

```java
public void leaderElect(LeaderSelectorListener leaderSelectorListener){
    LeaderSelector selector = new LeaderSelector(client, Constants.MASTER_LOCK , leaderSelectorListener);
    selector.start();
}
```



## UDP

从数据库抽取/拉取数据发送到下游渠道

增量任务

基于`mysql binlog`的来获取数据,这里使用了阿里开源的`canal`来拉取binlog数据的,我们使用的是row模式的binlog

原理如下

![https://ws3.sinaimg.cn/large/005BYqpggy1g3n5mier1cj30e209fmzr.jpg](https://s2.ax1x.com/2019/06/02/VGHd2D.png)	  

全量任务

​	基于mysql的`select *`拉取数据 

配置mysql用户权限`GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'udp_etl'@'10.%'`  



#### 全量的流式JDBC

拉取全量数据时,若是使用常规模式则会一次性拉取全部数据,内存溢出

```java
//statement模式
//光标只能向前移动,只读结果集..createStatement()默认配置就是此类型
statement = conn.createStatement(ResultSet.TYPE_FORWARD_ONLY,ResultSet.CONCUR_READ_ONLY);
//一次只拉取1000条数据
statement.setFetchSize(1000);
resultSet = statement.executeQuery(sql);

//pre模式
//ps = (PreparedStatement) con.prepareStatement(sql,
					//ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
//ps.setFetchSize(1000);
//ps.setFetchDirection(ResultSet.FETCH_REVERSE);

 List<SqlQueryEventRequest> request  =new ArrayList<SqlQueryEventRequest>();
 while (resultSet.next()){
 	request.add(dataResult);
    if(result.size() % 500 == 0){
        sendMessage(request);
        request.clear();
    } 
 }
 sendMessage(request);
```



#### 消息丢失

基于binlog/gtid上送数据位置到center,下次下发位置是基于已确认的位置



#### 消息顺序性

UDP是否需保证消息发送的顺序性是基于下游是否能乱序接收消息

如果下游能保证,如我们自己的`UDP-ETL`,则可多线程异步发送

若下游无法保证,则需根据下游来定制方案

+ 基于kafka的单个分区有序性,,控制每个表分配到一个相同的分区



## ETL

#### 消费数据并投递到ES

流程 : 查找虚拟数据源 --> 查找索引 --> 数据映射到es属性 --> 更新es

增量模式 - 消息拉取信息后使用bulk模式投递到ES

全量模式 - 接收osp的请求并将请求投递到ES



#### 消息重复投递问题/消息乱序问题

投递消息时每条记录都包含了主键以及mysql主库binlog记录时间,执行es操作时先读取es记录的操作时间,版本号等信息,在更新时判断是否是新的消息记录,并且每次更新时都会带上版本号,使用乐观锁来更新记录,失败的记录会进入重试

常见的update语句先于insert先到,那么此时update语句会失败,只执行insert语句,待重试任务将update语句更新到es

若只有update语句,而拉取的insert语句刚好在拉取的binlog点之前,那么insert语句丢失,那么此时update语句在最后一波重试时会变为insert类型插入数据



#### 消息重试

ETL中的重试主要是将执行失败的记录进行重试.

若是内部流程异常,则直接进行flow调用重试(重新走一遍寻找数据源,寻找索引,属性匹配等流程)

若是在最后一部更新es报错,则将数据进行es重试

主要es报错类型为`version_conflict_engine_exception`(版本冲突不减少重试次数),`document_missing_exception`



## UDP-API

基于业务方提供查询ES的OSP接口





## 踩过的ZK坑

`Mapping`节点在网络中断重连后的数据丢失,表现为UDP全部数据都不只拉取但都不发送

上一个版本使用临时节点,在center选主成功或者udp实例启动成功时去拉取数据

这种情况在主没重新选主,而udp实例没重新启动的情况下,只是单单网络中断的话,zk节点会数据丢失

解决办法是监听节点变化或直接使用curator的`PersistentEphemeralNode`类

```java
/**
  * 写一个永久的临时节点
  * 其实`PersistentEphemeralNode`类就是添加了一个在`ConnectionState.RECONNECTED`事件下的重新创建节点
  */
private void createPersistentEphemeralNode(String path,Object data) throws Exception {
    try {
        Stat pathStat = client.checkExists().forPath(path);
        long sessionId = client.getZookeeperClient().getZooKeeper().getSessionId();
        if (pathStat != null) {
            if (sessionId != pathStat.getEphemeralOwner()) {
                logger.info("[Delete for previous seesion Ephemeral node][Path:" + path + "]");
                client.delete().forPath(path);
            }
        }

        logger.info("[Create for current session Ephemeral File][Path:" + path + "]");
        PersistentEphemeralNode node = 
            	new PersistentEphemeralNode(client, 
                                            PersistentEphemeralNode.Mode.EPHEMERAL, 
                                            path,
                                            JsonUtils.toJson(data).getBytes());
        node.start();
    } catch (Exception e) {
        logger.error("createPersistentEphemeralNode-->操作zk异常", e);
        throw e;
    }

}
```













