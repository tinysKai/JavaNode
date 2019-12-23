## 使用GenericObjectPool创建FTP连接池

#### ftp线程池类

```java
import org.apache.commons.net.ftp.FTPClient;
import org.apache.commons.pool2.impl.GenericObjectPool;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * FTP连接池
 */
@Service
public class FTPClientPool implements InitializingBean, DisposableBean {

    private final static Logger log = LoggerFactory.getLogger(FTPClientPool.class);

    @Autowired
    private PropertyManager propertyManager;

    //FTP连接池map
    private final Map<String, GenericObjectPool<FTPClient>> internalPools = new ConcurrentHashMap<>();

    /**
     * 初始化FTP连接池
     * @throws Exception
     */
    @Override
    public void afterPropertiesSet() throws Exception {
   
        /*try {

        } catch (Exception e) {
            log.error("初始化FTP连接池失败", e);
        }
        log.info("初始化FTP连接池完成");*/
    }

    /**
     * 销毁FTP连接池
     */
    @Override
    public void destroy() throws Exception {
        try {
            //销毁所有FTP连接池
            for (Map.Entry<String, GenericObjectPool<FTPClient>> entry: internalPools.entrySet()){
                GenericObjectPool pool = entry.getValue();
                pool.close();
            }
        } catch (Exception e) {
            log.error("销毁FTP连接池异常：", e);
        }
        log.info("销毁FTP连接池完成");
    }




    /**
     * 获取FTPClient
     */
    public FTPClient getFTPClient(FTPConfig ftpConfig) {
        FTPClient ftpClient = null;
        String key = getFTPClientPoolKey(ftpConfig);
        GenericObjectPool<FTPClient> pool = internalPools.get(key);
        if (null == pool) {
            synchronized (FTPClientPool.class) {
                pool = internalPools.get(key);
                if (null == pool) {
                    //创建FTP连接池
                    pool = new GenericObjectPool<FTPClient>(new FTPClientFactory(ftpConfig), ftpConfig);
                    internalPools.put(key,  pool);
                }
            }
        }
        try {
            //从连接池中获取FTPClient
            ftpClient = pool.borrowObject();
        } catch (Exception e) {
            throw new ConnectionException(e.getMessage(), e, CallRespMessage.connectionTimeout);
        }
        return ftpClient;
    }

    /**
     * 归还FTP连接
     */
    public void returnFTPClient(FTPConfig ftpConfig, FTPClient ftpClient) {
        try {
            String key = getFTPClientPoolKey(ftpConfig);
            GenericObjectPool<FTPClient> pool = internalPools.get(key);
            pool.returnObject(ftpClient);
        } catch (Exception e) {
            log.error("close ftp connection catch error, {}", e.getMessage());
        }
    }

    /**
     * 创建FTP配置
     */
   public FTPConfig createFTPConfig(String host, int port, String username, String password,
           int connectTimeout, int sockeTimeout, boolean passiveMode){
       FTPConfig ftpConfig = new FTPConfig(host, port, username, password);
       //FTP相关配置
       ftpConfig.setConnectTimeout(connectTimeout); //连接超时
       ftpConfig.setSockeTimeout(sockeTimeout);//读写超时
       ftpConfig.setPassiveMode(passiveMode); //是否开启被动模式。默认是主动

       //连接池相关配置
       int minPoolSize = new Integer(propertyManager.getProperty("ftpMinPoolSize", "30")).intValue();
       int maxPoolSize = new Integer(propertyManager.getProperty("ftpMaxPoolSize", "100")).intValue();
       int timeBetweenEvictionRunsMillis = new Integer(propertyManager.getProperty("ftpTimeBetweenEvictionRunsMillis", "20000")).intValue();//空闲检查定时器20s执行一次
       int minEvictableIdleTimeMillis = new Integer(propertyManager.getProperty("ftpMinEvictableIdleTimeMillis", "40000")).intValue();//连接存活时间为40s
    
       ftpConfig.setMinIdle(minPoolSize); //最小连接数
       ftpConfig.setMaxIdle(maxPoolSize); //最大连接数
       ftpConfig.setMaxTotal(maxPoolSize);//最大连接数
       ftpConfig.setMaxWaitMillis(5000);// 不设置默认无限等待,当连接池资源耗尽时，等待时间，超出则抛异常，默认为-1即永不超时
       ftpConfig.setTestWhileIdle(true); //在空闲的时候检查连接的有效性
       ftpConfig.setTimeBetweenEvictionRunsMillis(timeBetweenEvicti onRunsMillis); //空闲检查定时器20s执行一次
       ftpConfig.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);  //连接存活时间为40s
       ftpConfig.setNumTestsPerEvictionRun(100); //每次检测20个连接。默认是3
       return  ftpConfig;
   }


    /**
     * 获取FTP连接池key
     */
    public static String getFTPClientPoolKey(FTPConfig ftpConfig){
        //FTP连接池的key组成：主机+端口+用户名+密码
        String key = ftpConfig.getHost()+ftpConfig.getPort()+ftpConfig.getUsername()+ftpConfig.getPassword();
        return key;
    }



}

```

#### 工厂类 -- 创建client到连接池

```java
import org.apache.commons.net.ftp.FTPClient;
import org.apache.commons.pool2.BasePooledObjectFactory;
import org.apache.commons.pool2.PooledObject;
import org.apache.commons.pool2.impl.DefaultPooledObject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * FTPClient工厂
 */
public class FTPClientFactory extends BasePooledObjectFactory<FTPClient> {

    private final static Logger log = LoggerFactory.getLogger(FTPClientFactory.class);

    private FTPConfig ftpConfig;

    public FTPClientFactory() {
    }

    public FTPClientFactory(FTPConfig ftpConfig) {
        this.ftpConfig = ftpConfig;
    }

    /**
     * 创建FTP对象
     */
    @Override
    public FTPClient create() throws Exception {
        FTPClient client = null;
        try {
            client = new FTPClient();
            //设置连接超时时间
            client.setConnectTimeout(ftpConfig.getConnectTimeout());
            //连接FTP
            client.connect(ftpConfig.getHost(), ftpConfig.getPort());
            //设置读写超时时间
            client.setSoTimeout(ftpConfig.getSockeTimeout());
            //设置主动模式超时时间
            client.setDataTimeout(ftpConfig.getSockeTimeout());
            //是否开启被动模式，默认否
            if (ftpConfig.isPassiveMode()) {
                client.enterLocalPassiveMode();
            }
            //登录到FTP
            client.login(ftpConfig.getUsername(), ftpConfig.getPassword());
            log.info("创建FTP对象{} {}@{}:{} success", client, ftpConfig.getUsername(), ftpConfig.getHost(), ftpConfig.getPort());
        } catch (Exception e){
            log.error(e.getMessage(), e);
        }
        return client;
    }

    /**
     * 把FTP对象包装为池对象
     */
    @Override
    public PooledObject<FTPClient> wrap(FTPClient ftpClient) {
        return new DefaultPooledObject<FTPClient>(ftpClient);
    }

    /**
     * 检测FTP对象是否可用
     */
    @Override
    public boolean validateObject(PooledObject<FTPClient> pooledObject) {
        boolean connect = false;
        try {
            FTPClient ftpClient = pooledObject.getObject();
            connect = ftpClient.sendNoOp();
        } catch (Exception e) {
            //log.error(e.getMessage(), e);
            //探测出问题不打印ERROR日志，以免干扰正常告警
            log.warn(e.getMessage());
        }
        return connect;
    }

    /**
     * 销毁对象
     */
    @Override
    public void destroyObject(PooledObject<FTPClient> pooledObject) throws Exception {
        try {
            FTPClient ftpClient = pooledObject.getObject();
            //ftpClient.logout();
            if (ftpClient.isConnected()) {
                log.info("close ftp connection");
                ftpClient.disconnect();
            }
        } catch (Exception e) {
            // do nothing
            log.error("close ftp connection catch error, {}", e.getMessage());
        }
    }



    public FTPConfig getFtpConfig() {
        return ftpConfig;
    }

    public void setFtpConfig(FTPConfig ftpConfig) {
        this.ftpConfig = ftpConfig;
    }
}

```

#### FTP配置实体类

```java
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;

/**
 * FTP配置
 */
public class FTPConfig extends GenericObjectPoolConfig{

    public FTPConfig() {
    }

    public FTPConfig(String host, int port, String username, String password) {
        this.host = host;
        this.port = port;
        this.username = username;
        this.password = password;
    }

    private String host;  //FTP IP

    private int port;   //FTP端口

    private String username; //FTP用户名

    private String password;  //FTP密码

    private boolean passiveMode = false;  //是否采用被动模式。默认是主动模式

    private int connectTimeout = 2000;  //连接超时时间。默认2000ms

    private int sockeTimeout = 6000;  //读写超时时间。默认6000ms

    //get set
}

```

#### 使用线程池的调用

```java
FTPConfig ftpConfig = ftpClientPool.createFTPConfig(host, port, username, password,
                        getConnectionTimeout(request), getSocketTimeout(request), passiveMode);
 org.apache.commons.net.ftp.FTPClient client = ftpClientPool.getFTPClient(ftpConfig);
 client.changeWorkingDirectory(remoteFilePath);
boolean result = client.storeFile(remoteFilename, FileUtils.openInputStream(localFile));
```

#### 短连接的使用方式

```java
org.apache.commons.net.ftp.FTPClient client = ftpClientPool.getFTPClient(ftpConfig);
 client = new org.apache.commons.net.ftp.FTPClient();
client.setConnectTimeout(getConnectionTimeout(request));
client.connect(host, port);
client.setSoTimeout(getSocketTimeout(request));
//设置主动模式超时时间
client.setDataTimeout(getSocketTimeout(request));
//是否开启被动模式，默认否
//client.enterLocalPassiveMode();


client.login(username, password);
client.changeWorkingDirectory(remoteFilePath);
boolean result = client.storeFile(remoteFilename, FileUtils.openInputStream(localFile));
```



#### 总结

本篇文章的重点思想在于学习怎么利用common-pool来对一个集合对象进行池化

