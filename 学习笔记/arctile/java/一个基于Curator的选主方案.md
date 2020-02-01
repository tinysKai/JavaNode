## 一个基于Curator的选主方案

#### 选主流程

![993b4ce6-6d81-4660-b1fc-1eba05b5ca1c-2575962.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gb7kf6b59rj20sj0j9dh2.jpg)

#### 代码实现

curator对象创建

```java
@Service
public class ZookeeperClient {

	private Logger logger = LoggerFactory.getLogger(ZookeeperClient.class);
	private static final ObjectMapper objMapper = new ObjectMapper();
	
	private CuratorFramework client;
	
	@Autowired
	private PropertyManager propertyManager;
	
	private String connectString;
	
	private String rootNode;
	
	public ZookeeperClient(){}

	@PostConstruct
	public void init(){		
		logger.info("ZookeeperClient -->init zookeeper path={}" , propertyManager.getProperty("ZOOKEEPER_CONNECT_STRING"));
		rootNode = propertyManager.getProperty("ROOT_NODE","UDP");
		logger.info("ZookeeperClient -->init rootNode={}" , rootNode);
		this.connectString = propertyManager.getProperty("ZOOKEEPER_CONNECT_STRING");
		client = CuratorFrameworkFactory.builder()
				.namespace(this.rootNode)
				.connectString(this.connectString)
				.sessionTimeoutMs(1000 * 60)
				.connectionTimeoutMs(10000)
				.retryPolicy(new RetryNTimes(5, 3000))
				.build();
		client.start();
		try {
			client.blockUntilConnected();
		} catch (InterruptedException e) {
			logger.error("connect zookeeper error : {}" , e);
			System.exit(0);
		}		
	}
	
	public void leaderElect(LeaderSelectorListener leaderSelectorListener){
		LeaderSelector selector = new LeaderSelector(client, Constants.MASTER_LOCK , leaderSelectorListener);
		selector.start();
	}
    
    public void watchChildrenPathChange(String path, CuratorWatcher watcher) throws Exception {		
			client.getChildren().usingWatcher(watcher).forPath(path);
	}
    
    public Stat getPathStat(String path) throws Exception {
		return this.client.checkExists().forPath(path);
	}
}    
```



竞争选主代码

```java
//在bean初始化后或spring完成后可竞争主
public void leaderElect(LeaderSelectorListener leaderSelectorListener){
    	//client为CuratorFramework对象
		LeaderSelector selector = new LeaderSelector(client, Constants.MASTER_LOCK , leaderSelectorListener);
		selector.start();
}
```

回调通知

```java
	//上面传入的LeaderSelectorListener接口的回调
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
			client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(Constants.MASTER + "/" + this.localIp, null);			
			this.status = PROCESSING;
			this.masterIp = this.localIp;
			zookeeperClient.watchChildrenPathChange(Constants.MASTER, this);
			//do component init
			init();
			
			Thread.sleep(2000L);
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

监听节点变化

```java
//在实现了CuratorWatcher接口之后接收上面的watch通知
@Override
public void process(WatchedEvent event) {
    // the master shutdown , do the leadership elect again
    if(event.getType() == EventType.NodeChildrenChanged){
        try {
            Stat stat = this.zookeeperClient.getPathStat(Constants.MASTER);
            // no master to do the next elect
           leaderElect(this);
        } catch (Exception e) {
            logger.error("get path stat error {}", e);
        }
    }
}
```

#### LeaderSeletor解析

**当回调方法处理完毕释放后,leader会重新释放出来**,其它之前阻塞的线程可重新获取到锁来判断,因此若想一直占有leader需特殊处理.

LeaderSeletor源码中的处理流程是 : 

阻塞获取锁 --> 获取到锁时调用回调方法 --> 释放锁 --> 判断当前任务是否重新进入竞选

autoRequque方法用于重复竞选主,若不设置则只竞选一次



