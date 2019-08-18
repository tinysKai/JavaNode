## Curator学习笔记

#### 定义

Curator作为ZooKeeper的一个高层次封装库，为开发人员封装了ZooKeeper的一组开发库，Curator的核心目标就是为你管理ZooKeeper的相关操作，将连接管理的复杂操作部分隐藏起来.

#### 功能点

+ 锁（lock）
+ 屏障（barrier）
+ 缓存（cache）
+ 命名空间
+ 自动重连

#### 引入

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```

#### 创建链接

```java
private CuratorFramework  client;//定义curator客户端

@PostConstruct
public void init(){		
    logger.info("zookeeper path : {}" , manager.getProperty("ZOOKEEPER_CONNECT_STRING"));
    this.connectString = manager.getProperty("ZOOKEEPER_CONNECT_STRING");
    client = CuratorFrameworkFactory.builder()
        .namespace("${namespace}")
        .connectString(this.connectString)
        .sessionTimeoutMs(60000) //默认60s
        .connectionTimeoutMs(10000) //默认15s
        .retryPolicy(new RetryNTimes(5, 3000)) //重试策略
        .build();
    client.start(); 
    try {
        client.blockUntilConnected(); //阻塞直到连接成功
    } catch (InterruptedException e) {
        logger.error("connect zookeeper error : {}" , e);
        System.exit(0);
    }		
}
```

#### 节点操作

```java
//创建节点
public void createPathData(String path, Object obj, CreateMode createModel) throws Exception{
    if(obj != null){
        client.create().withMode(createModel).forPath(path, JsonUtils.toJson(obj).getBytes());
    }else{
        client.create().withMode(createModel).forPath(path);
    }
}

//删除节点
public void removeNode(String path) {
    try {
        this.client.delete().deletingChildrenIfNeeded().forPath(path);
    } catch (Exception e) {
        logger.info("delete node error : {}", e);
    }
}


//节点不为空时设置节点数据
public void setData(String path, String nodeName, Object obj) throws Exception {
    if(this.getPathStat(path) != null){
        client.setData().forPath(path + Constants.SPLIT_CHAR + nodeName, JsonUtils.toJson(obj).getBytes());
    }
}


//获取节点数据
public byte[] getData(String path) {
    try {
        return this.client.getData().forPath(path);
    } catch (Exception e) {

    }
    return null;
}

//检查是否有子节点
public boolean checkExistChildren(String path) throws Exception{
    Stat stat = client.checkExists().forPath(path);
    if(stat != null && stat.getNumChildren() > 0){
        return true;
    }else{
        return false;
    }
}

//获取子节点
public List<String> getChildren(String parentPath) throws Exception {
    return this.client.getChildren().forPath(parentPath);
}

//删除某节点下的子节点
public void removeChild(String path) {
    List<String> list;
    try {
        list = this.client.getChildren().forPath(path);
        if( list != null ){
            for(String child : list){
                this.client.delete().deletingChildrenIfNeeded().forPath(path + "/" + child);
            }
        }
    } catch (Exception e) {
        logger.error("remove child error : {}", e);
    }

}

//获取节点的stat信息
public Stat getPathStat(String path) throws Exception {
    return this.client.checkExists().forPath(path);
}

//写一个永久的临时节点,会覆盖原有节点,因使用了会话来判断
public void createNode(String path, Object data) throws UdpException {
    try {
        byte[] byteData = data instanceof String ? ((String) data).getBytes() : objMapper.writeValueAsBytes(data);

        Stat pathStat = zk.checkExists().forPath(path);

        //判断会话id是否相同
        long sessionId = zk.getZookeeperClient().getZooKeeper().getSessionId();
        if (pathStat != null) {
            if (sessionId != pathStat.getEphemeralOwner()) {
                logger.info("[Delete for previous seesion Ephemeral node][Path:" + path + "]");
                zk.delete().forPath(path);
            }
        }

        logger.info("[Create for current session Ephemeral File][Path:" + path + "]");
        //永久临时节点不会因网络抖动而丢失,主要是针对 ConnectionState.RECONNECTED状态来进行重新创建
        /**
         * 通过以下方式来进行网络重连时的重新创建节点
         *   client.getConnectionStateListenable().addListener(connectionStateListener);
               private final ConnectionStateListener connectionStateListener = new ConnectionStateListener()
                    {
                        @Override
                        public void stateChanged(CuratorFramework client, ConnectionState newState)
                        {
                            if ( newState == ConnectionState.RECONNECTED )
                            {
                                createNode();
                            }
                        }
                    };
        
        */
        PersistentEphemeralNode node = 
            new PersistentEphemeralNode(zk, 
                                        PersistentEphemeralNode.Mode.EPHEMERAL,
                                        path,
                                        byteData);
        node.start();

    } catch (Exception e) {
        e.getStackTrace();
        throw new UdpException(e);
    }
}

```

#### 选主

```java
LeaderSelectorListener listener = new LeaderSelectorListenerAdapter()
{
    public void takeLeadership(CuratorFramework client) throws Exception
    {
        // this callback will get called when you are the leader
        // do whatever leader work you need to and only exit
        // this method when you want to relinquish leadership
    }
};

LeaderSelector selector = new LeaderSelector(client, path, listener);
//下面方法表示在回调方法退出后自动重新竞争主,若想自己控制选主,则可不用此方法
//selector.autoRequeue();  // not required, but this is behavior that you will probably expect
selector.start();

//选主的另一种可退出方式是写入一个其他节点,如MAASTER_LOCK节点,然后在回调中重新创建Master节点并写入ip信息
//之后监听Master节点子节点变化,每次子节点变化时都直接进行重新选主判断
@Override
public void takeLeadership(CuratorFramework client) throws Exception {
    // watch the master path for the node
    Stat masterStat = client.checkExists().forPath(Constants.MASTER); 
    if(!isLeader(masterStat)){
        // not the master
        logger.info("{} be follower", this.localIp);
        this.masterIp = zookeeperClient.getChildren(Constants.MASTER).get(0);
        zookeeperClient.watchChildrenPathChange(Constants.MASTER, this);
        this.status = FOLLOWING;
    } else {
        logger.info("{} be leader", this.localIp);
        // add the master node
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL)
            .forPath(Constants.MASTER + "/" + this.localIp, null);			
        this.status = PROCESSING;
        this.masterIp = this.localIp;
        zookeeperClient.watchChildrenPathChange(Constants.MASTER, this);
        //do component init
        init();

        Thread.sleep(2000l);
    }
}


private boolean isLeader(Stat masterStat) throws Exception {
    if(masterStat != null && masterStat.getNumChildren() > 0){
        //ip 与master 节点相同
        if(this.localIp.equals(zookeeperClient.getChildren(Constants.MASTER).get(0))){
            return true;
        } else {
            //其他不是master节点
            return false;
        }
    }
    //不存在子节点，是master 节点
    return true;
}
```



#### 分布式锁

```java
InterProcessMutex lock = new InterProcessMutex(client, lockPath);
if ( lock.acquire(maxWait, waitUnit) ) 
{
    try 
    {
        // do some work inside of the critical section here
    }
    finally
    {
        lock.release();
    }
}
```

#### 发现/监听

三种cache监听器,事件触发后会自动再监听

+ Node Cache - 只关心`update/create/delete`事件
+ Path Cache - 只关心子节点的变化
+ Tree Cache -  能监听自身以及子节点的变化

```java
public PathChildrenCache createPathChildrenCache(String path, PathChildrenCacheListener listener) throws Exception {
    //true表示客户端在接收到节点列表发生变化的同时，也能够获取到节点的数据内容
    PathChildrenCache cache = new PathChildrenCache(this.client, path, true); 
    cache.getListenable().addListener(listener);
    cache.start();
    cache.rebuild();//阻塞方法,会到zk中重新查询数据到内存
    return cache;
}

//监听回调
public class InstanceManager implements PathChildrenCacheListener{
	@Override
	public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
        logger.info("instance change event type : {}", event.getType());
        //判断类型
        if(event.getType().equals(PathChildrenCacheEvent.Type.CHILD_ADDED)){
            createInstance(event);
        }else if(event.getType().equals(PathChildrenCacheEvent.Type.CHILD_REMOVED)){
            removeInstance(event);
        }	
	}
}    
```

```java
public void addChildWatcher(String path, BaseTreeCacheListener baseTreeCacheListener) throws UdpException {
    try {
        TreeCache treeCache = new TreeCache(zk, path);
        treeCache.start();
        treeCache.getListenable().addListener(baseTreeCacheListener);
    } catch (Exception e) {
        logger.debug("UdpZookeeper addChildWatcher error : {}", e.getMessage());
        throw new UdpException(e);
    }
}

public void addDataWatcher(String path, BaseDataListener baseDataListener) throws UdpException {
    try {
        NodeCache cache = new NodeCache(zk, path);
        cache.getListenable().addListener(baseDataListener);
        cache.start(true);
    } catch (Exception e) {
        logger.debug("UdpZookeeper addDataWatcher error : {}", e.getMessage());
        throw new UdpException(e);
    }
}
```

原始的监听

原生的监听只生效一次,触发后需重新下发监听

```java
//异步方法调用的模式,针对节点的变化会回调到CuratorWatcher中
private final CuratorWatcher watcher = new CuratorWatcher(){
    @Override
    public void process(WatchedEvent event) throws Exception
    {
        if ( event.getType() == EventType.NodeDeleted )
        {
            createNode();
        }
        else if ( event.getType() == EventType.NodeDataChanged )
        {
            watchNode();
        }
    }
};

private void watchNode() throws Exception{
    if ( !isActive() )
    {
        return;
    }

    String localNodePath = nodePath.get();
    if ( localNodePath != null )
    {
        client.checkExists().usingWatcher(watcher).inBackground(checkExistsCallback).forPath(localNodePath);
    }
}

private final BackgroundCallback checkExistsCallback = new BackgroundCallback(){
    @Override
    public void processResult(CuratorFramework client, CuratorEvent event) throws Exception{
        if ( event.getResultCode() == KeeperException.Code.NONODE.intValue() ) {
                createNode();
         }
        //....
    }
}  

  
```

