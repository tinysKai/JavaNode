## CMS收集器产生GC时间过长排查

#### 背景

**JVM配置**

```
 -XX:+UnlockDiagnosticVMOptions -XX:+PrintCommandLineFlags -XX:-OmitStackTraceInFastThrow 
 -XX:-UseBiasedLocking -XX:-UseCounterDecay -XX:AutoBoxCacheMax=20000 
 -XX:+PerfDisableSharedMem -Djava.security.egd=file:/dev/./urandom 
 -Djava.net.preferIPv4Stack=true -Djava.awt.headless=true  -XX:-TieredCompilation 
 -Dio.netty.leakDetectionLevel=disabled -Dio.netty.recycler.maxCapacity.default=0 
 -Dio.netty.allocator.numHeapArenas=20 -server  
 -XX:MaxDirectMemorySize=4096m -XX:+AlwaysPreTouch -XX:MetaspaceSize=256m 
 -XX:ReservedCodeCacheSize=240M -XX:+PrintPromotionFailure 
 -XX:NewRatio=1 -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 
 -XX:+UseCMSInitiatingOccupancyOnly -XX:MaxTenuringThreshold=4 
 -XX:ParGCCardsPerStrideChunk=1024 -XX:+ParallelRefProcEnabled 
 -XX:+ExplicitGCInvokesConcurrent -XX:+CMSParallelInitialMarkEnabled 
 -Xloggc:/dev/shm/gc-osp-8080.log -XX:+PrintGCApplicationStoppedTime 
 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Dcom.sun.management.jmxremote.port=8060 
 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
 -Dcom.sun.management.jmxremote.ssl=false -Dsun.rmi.transport.tcp.threadKeepAliveTime=75000
 -Djava.rmi.server.hostname=127.0.0.1 -XX:ErrorFile=/apps/logs/osp//hs_err_%p.log
 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/apps/logs/osp// 
 -Dspring.profiles.active=production -Xmx16g -Xms16g -Xmn4g -XX:MetaspaceSize=1024m
 -XX:MaxMetaspaceSize=1024m -XX:+PrintGCCause -XX:+CMSParallelInitialMarkEnabled  
```

重要参数解释

任何一个JVM参数的默认值可以通过`java -XX:+PrintFlagsFinal -version |grep JVMParamName`获取，例如：`java -XX:+PrintFlagsFinal -version |grep MetaspaceSize`

```less
-XX:-UseBiasedLocking   //取消偏向锁
-XX:AutoBoxCacheMax=20000  //自动装箱的范围到20000
-XX:+PrintCommandLineFlags //这个参数的作用是显示出JVM初始化完毕后所有跟最初的默认值不同的参数及它们的值
-Djava.net.preferIPv4Stack=true  //禁用IPV6
-XX:MaxDirectMemorySize  //设置堆外内存,永久区的大小
-XX:+PrintPromotionFailure //能得到多大的新生代对象晋升到老生代失败从而引发Full GC的
-XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly //老年代达到75%时即触发CMS回收老年代,这两个参数需一起用才生效
-XX:MaxTenuringThreshold=4 //对象经过多少次young gc后晋升老年代
-Xloggc:/dev/shm/gc-osp-8080.log -XX:+PrintGCDateStamps -XX:+PrintGCDetails //打印gc日志
-XX:+PrintGCCause   //打印产生GC的原因，比如AllocationFailure什么的，在JDK8已默认打开，JDK7要显式打开一下
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/apps/logs/osp  //内存溢出时设置生成hprof的路径
-XX:+ExplicitGCInvokesConcurrent  //让full gc时使用CMS算法，不是全程停顿，必选
-XX: PermSize=128m -XX:MaxPermSize=512m （JDK7）
-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m（JDK8） //JDK8的永生代几乎可用完机器的所有内存，同样设一个128M的初始值，512M的最大值保护一下。

-XX:+CMSScavengeBeforeRemark
//默认为关闭，在CMS remark前，先执行一次minor GC将新生代清掉，这样从老生代的对象引用到的新生代对象的个数就少了，停止全世界的CMS remark阶段就短一些。
//但如果打开了，会让一次YGC紧接着一次CMS GC，使得停顿的总时间加长了。
//又一个真实案例，CMS GC的时间和新生代的使用量成比例，新生代较小时很快完成，新生代快满时CMS GC的停顿时间超过2秒，这时候就还是打开了划算。

-XX:+PerfDisableSharedMem //设置了此值就用不了jps以及jstat
//jvm在启动的时候都会分配一块内存来存PerfData，只是说这个PerfData是不是其他进程可见的问题
//如果设置了这个PerfDisableSharedMem参数，说明不能被共享，此时其他进程将访问不了该内存

```





**JVM参数解析**

内存碎片问题相关参数

+ `-XX：UseCompactAtFullCollection`每次执行FULL GC时执行碎片整理
+ `-XX：CMSFullGCsBeforeCompaction=n`执行n次FULL GC后执行碎片处理

**GC日志**

```verilog
//以下是发生Minor GC的日志例子
2019-05-13T16:30:41.922+0800: 351678.795: [GC (CMS Initial Mark) [1 CMS-initial-mark: 11685845K(12582912K)] 15346701K(16357824K), 0.1520733 secs] [Times: user=2.70 sys=0.00, real=0.16 secs] 
2019-05-13T16:30:42.075+0800: 351678.947: Total time for which application threads were stopped: 0.1534123 seconds, Stopping threads took: 0.0001098 seconds
2019-05-13T16:30:42.075+0800: 351678.947: [CMS-concurrent-mark-start]
2019-05-13T16:30:44.899+0800: 351681.772: [GC (Allocation Failure) 2019-05-13T16:30:44.900+0800: 351681.772: [ParNew: 3774912K->365634K(3774912K), 1.5980074 secs] 15460757K->12281907K(16357824K), 1.5983630 secs] [Times: user=3.58 sys=0.00, real=1.60 secs] 
2019-05-13T16:30:46.498+0800: 351683.371: Total time for which application threads were stopped: 1.5996209 seconds, Stopping threads took: 0.0001083 seconds
2019-05-13T16:30:47.299+0800: 351684.171: [CMS-concurrent-mark: 3.624/5.224 secs] [Times: user=22.32 sys=0.00, real=5.22 secs] 
2019-05-13T16:30:47.299+0800: 351684.171: [CMS-concurrent-preclean-start]
2019-05-13T16:30:47.499+0800: 351684.372: Total time for which application threads were stopped: 0.0011321 seconds, Stopping threads took: 0.0001062 seconds
2019-05-13T16:30:47.675+0800: 351684.548: [CMS-concurrent-preclean: 0.366/0.376 secs] [Times: user=0.41 sys=0.00, real=0.38 secs] 
2019-05-13T16:30:47.675+0800: 351684.548: [CMS-concurrent-abortable-preclean-start]
 CMS: abort preclean due to time 2019-05-13T16:30:53.088+0800: 351689.960: [CMS-concurrent-abortable-preclean: 5.412/5.413 secs] [Times: user=6.19 sys=0.00, real=5.41 secs] 
2019-05-13T16:30:53.089+0800: 351689.962: [GC (CMS Final Remark) [YG occupancy: 810536 K (3774912 K)]2019-05-13T16:30:53.089+0800: 351689.962: [Rescan (parallel) , 0.0502591 secs]2019-05-13T16:30:53.139+0800: 351690.012: [weak refs processing, 0.0025081 secs]2019-05-13T16:30:53.142+0800: 351690.014: [class unloading, 0.0178253 secs]2019-05-13T16:30:53.160+0800: 351690.032: [scrub symbol table, 0.0110555 secs]2019-05-13T16:30:53.171+0800: 351690.043: [scrub string table, 0.0015640 secs][1 CMS-remark: 11916273K(12582912K)] 12726810K(16357824K), 0.0839412 secs] [Times: user=0.94 sys=0.00, real=0.08 secs] 
2019-05-13T16:30:53.173+0800: 351690.046: Total time for which application threads were stopped: 0.0851026 seconds, Stopping threads took: 0.0001201 seconds
2019-05-13T16:30:53.173+0800: 351690.046: [CMS-concurrent-sweep-start]
2019-05-13T16:30:59.297+0800: 351696.170: [CMS-concurrent-sweep: 6.124/6.124 secs] [Times: user=7.15 sys=0.00, real=6.13 secs] 
2019-05-13T16:30:59.297+0800: 351696.170: [CMS-concurrent-reset-start]
2019-05-13T16:30:59.340+0800: 351696.213: [CMS-concurrent-reset: 0.043/0.043 secs] [Times: user=0.04 sys=0.00, real=0.04 secs] 
    

//以下日志是触发了`concurrent mode failure`的例子    
2019-05-13T18:16:23.897+0800: 358020.769: [GC (CMS Initial Mark) [1 CMS-initial-mark: 12149436K(12582912K)] 15381749K(16357824K), 0.1664505 secs] [Times: user=2.96 sys=0.00, real=0.17 secs] 
2019-05-13T18:16:24.063+0800: 358020.936: Total time for which application threads were stopped: 0.1678487 seconds, Stopping threads took: 0.0001188 seconds
2019-05-13T18:16:24.063+0800: 358020.936: [CMS-concurrent-mark-start]
2019-05-13T18:16:27.979+0800: 358024.852: [CMS-concurrent-mark: 3.916/3.916 secs] [Times: user=20.20 sys=0.00, real=3.92 secs] 
2019-05-13T18:16:27.979+0800: 358024.852: [CMS-concurrent-preclean-start]
2019-05-13T18:16:28.009+0800: 358024.881: [CMS-concurrent-preclean: 0.029/0.029 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
2019-05-13T18:16:28.009+0800: 358024.881: [CMS-concurrent-abortable-preclean-start]
2019-05-13T18:16:28.009+0800: 358024.881: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2019-05-13T18:16:28.010+0800: 358024.882: [GC (CMS Final Remark) [YG occupancy: 3401424 K (3774912 K)]2019-05-13T18:16:28.010+0800: 358024.883: [Rescan (parallel) , 0.1787776 secs]2019-05-13T18:16:28.189+0800: 358025.061: [weak refs processing, 0.0026205 secs]2019-05-13T18:16:28.191+0800: 358025.064: [class unloading, 0.0182134 secs]2019-05-13T18:16:28.210+0800: 358025.082: [scrub symbol table, 0.0113039 secs]2019-05-13T18:16:28.221+0800: 358025.094: [scrub string table, 0.0015662 secs][1 CMS-remark: 12149436K(12582912K)] 15550861K(16357824K), 0.2132539 secs] [Times: user=3.24 sys=0.00, real=0.21 secs] 
2019-05-13T18:16:28.223+0800: 358025.096: Total time for which application threads were stopped: 0.2145090 seconds, Stopping threads took: 0.0001171 seconds
2019-05-13T18:16:28.223+0800: 358025.096: [CMS-concurrent-sweep-start]
2019-05-13T18:16:34.468+0800: 358031.341: [CMS-concurrent-sweep: 6.245/6.245 secs] [Times: user=7.30 sys=0.00, real=6.24 secs] 
2019-05-13T18:16:34.469+0800: 358031.341: [CMS-concurrent-reset-start]
2019-05-13T18:16:34.518+0800: 358031.390: [CMS-concurrent-reset: 0.049/0.049 secs] [Times: user=0.06 sys=0.00, real=0.05 secs] 
2019-05-13T18:16:36.518+0800: 358033.391: [GC (CMS Initial Mark) [1 CMS-initial-mark: 12149243K(12582912K)] 15904204K(16357824K), 0.1884803 secs] [Times: user=3.36 sys=0.00, real=0.19 secs] 
2019-05-13T18:16:36.707+0800: 358033.579: Total time for which application threads were stopped: 0.1896868 seconds, Stopping threads took: 0.0000894 seconds
2019-05-13T18:16:36.707+0800: 358033.579: [CMS-concurrent-mark-start]
2019-05-13T18:16:37.817+0800: 358034.690: [GC (Allocation Failure) 2019-05-13T18:16:37.817+0800: 358034.690: [ParNew: 3774912K->3774912K(3774912K), 0.0000360 secs]2019-05-13T18:16:37.817+0800: 358034.690: [CMS2019-05-13T18:16:40.611+0800: 358037.484: [CMS-concurrent-mark: 3.903/3.904 secs] [Times: user=19.59 sys=0.00, real=3.91 secs] 
 (concurrent mode failure): 12149243K->12582911K(12582912K), 42.0340808 secs] 15924155K->12729771K(16357824K), [Metaspace: 95589K->95589K(1134592K)], 42.0344497 secs] [Times: user=53.25 sys=0.00, real=42.03 secs] 
    
```

------



#### 触发FULL GC的场景

+ System.gc()

+ 永久代区空间不足时(jdk1.8永久区已独立出来,不再占用jvm内存)

+ 执行`jmap`

+ 老年代内存不足

  + 年轻代对象晋升 -- 对象要晋升到老年代中时，发现老年代的内存不足了，所以要引发一次Full GC。对象的晋升分为正常晋升和提前晋升(Premature Promotion)
  + 直接分配大对象




#### 根节点解释

+ Stack Local  -- 局部变量/方法参数
+ Thread -- 活着的线程
+ Class - 由系统类加载器(system class loader)加载的对象，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象

------



#### CMS分析

**特点**

CMS(Concurrent Mark-Sweep)是**以牺牲吞吐量为代价来获得最短回收停顿时间**的垃圾回收器。对于要求服务器响应速度的应用上，这种垃圾回收器非常适合

CMS采用的基础算法是：**标记—清除**,所以CMS不会整理、压缩堆空间,会造成大量的内存碎片.

**GC流程**

#### ![https://ws3.sinaimg.cn/large/005BYqpggy1g30mrxgbyvj31090cqta1.jpg](https://s2.ax1x.com/2019/05/14/EIVCEn.png)

+ InitialMarking(初始化标记,收集所有`GC Roots`以及其直接引用的对象,STW)
+ Marking（并发标记）
+ Precleaning（并发预清理）
+ FinalMarking（重新标记，STW)
+ ConcurrentSweeping(并发清理)
+ ConcurrentReset(并发重置)



#### CMS出现的主要问题

+ Allocation Failure
+ Promotion Failed(晋升失败)
+ Concurrent Mode Failure



`Allocation Failure`表示向年轻代申请空间给新对象时,年轻代由于剩余空间不足导致的minor gc.

`Promotion Failed`发生在进行Minor GC时,Survivor Space放不下，对象只能放入老年代，而此时老年代也放不下导致的晋升失败.`Promotion Failed`往往伴随着`concurrent mode failure`.

`concurrent mode failure`发生在CMS的GC过程中,又触发了一次Full GC，则Full GC会抢占回收执行机会，停止CMS，采用Serial Old或Parallel Old这些采用标记-整理算法的GC方式（CMS采用的是标记-清理算法）进行Full GC。通常是对象要晋升到老年代中时，发现老年代的内存不足了，所以要引发一次Full GC。对象的晋升分为正常晋升和提前晋升。











































