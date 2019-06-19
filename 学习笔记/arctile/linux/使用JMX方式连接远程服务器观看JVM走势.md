## 使用JMX方式连接远程服务器观看JVM走势

#### JVM配置

**服务配置**

```
-XX:+PrintCommandLineFlags -XX:-OmitStackTraceInFastThrow -XX:-UseBiasedLocking 
-XX:-UseCounterDecay -XX:AutoBoxCacheMax=20000 -XX:+PerfDisableSharedMem 
-Djava.security.egd=file:/dev/./urandom -Djava.net.preferIPv4Stack=true 
-Djava.awt.headless=true -javaagent:/apps/dat/web/working/osp-udp.bfintra.com/mercurylib/
aspectjweaver-1.8.6.jar -XX:-TieredCompilation -XX:+CMSParallelInitialMarkEnabled 
-Dio.netty.leakDetectionLevel=disabled -Dio.netty.recycler.maxCapacity.default=0 
-server -Xms4096m -Xmx4096m -XX:MaxDirectMemorySize=2048m -XX:NewRatio=1 
-XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 
-XX:+UseCMSInitiatingOccupancyOnly -XX:MaxTenuringThreshold=4 
-XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=1024 -XX:+ParallelRefProcEnabled 
-XX:+ExplicitGCInvokesConcurrent -XX:+AlwaysPreTouch -XX:+PrintPromotionFailure 
-XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m -XX:ReservedCodeCacheSize=240M 
-Xloggc:/dev/shm/gc-osp-8014.log -XX:+PrintGCApplicationStoppedTime 
-XX:+PrintGCDateStamps -XX:+PrintGCDetails -Dcom.sun.management.jmxremote.port=8015 
-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
-Dcom.sun.management.jmxremote.ssl=false 
-Dsun.rmi.transport.tcp.threadKeepAliveTime=75000 -Djava.rmi.server.hostname=127.0.0.1 
-XX:ErrorFile=/apps/logs/osp/hs_err_%p.log -XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/apps/logs/osp/ -Dspring.profiles.active=integratetest  -Dcom.sun.management.jmxremote=true 
-Djava.rmi.server.hostname=10.197.204.106 -Dcom.sun.management.jmxremote.port=12306 -Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false  
-port 8014 -restport 8013
```

**JMX连接的关键配置**

```
//开启jmx端口
-Dcom.sun.management.jmxremote=true 
//设置开发机器远程能连接的远程机器IP,默认是127.0.0.1,
-Djava.rmi.server.hostname=10.197.204.106
//开启远程端口
-Dcom.sun.management.jmxremote.port=12306 
//开启不需要SSL连接
-Dcom.sun.management.jmxremote.ssl=false 
//开启不需要认证
-Dcom.sun.management.jmxremote.authenticate=false  
```



#### 使用JVM自带的工具观察

+ jvisualvm
+ jconsole

配置完上面的host,port,则可以直接使用jmx远程端口方式连接到对应的JVM服务.

