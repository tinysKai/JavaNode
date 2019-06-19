#### unable to create new native thread的分析
+ max user processers较小时,影响系统线程数目的是max user processers的设置
+ max user processers充足时,影响系统线程数目的是系统的内存

使用`ulimit -u 65535`命令或者直接修改limits.conf文件，都可修改max user process参数

#### 线程内存
在Java中每new一个线程，JVM都是向操作系统请求new一个本地线程，此时操作系统会使用剩余的内存空间来为线程分配内存，而不是使用JVM的内存。  
这样，当操作系统的可用内存越少，则JVM可用创建的新线程也就越少  
线程内存使用`-Xss`参数配置


#### jvm垃圾收集算法
+ 标记-清除算法(存在大量不连续的内存碎片)
+ 复制算法(年轻代使用的算法)
+ 标记-整理算法(老年代算法)
+ 分代算法(jvm分成年轻代和老年代)  

*https://blog.csdn.net/yyywyr/article/details/39347803*


#### 垃圾收集器
+ Serial(单线程,年轻代)
+ ParNew(多线程,年轻代,多搭配CMS老年代收集器)
+ Parallel Scavenge(多线程,年轻代,注重吞吐量)
+ Serial Old(单线程,老年代)
+ Parallel Old(多线程,老年代,注重吞吐量或CPU资源敏感的场合，可以优先考虑Parallel Scavenge收集器 + Parallel Old收集器)
+ CMS(多线程并发收集器,老年代,对CPU资源敏感)
+ G1(不区分年轻代或老年代了)

*https://blog.csdn.net/a19881029/article/details/12910163*

#### CMS收集过程
1，初始标记 CMS initial mark：该步会stop the world，但耗时非常短，标记GC Root直接关联的对象  
2，并发标记 CMS concurrent mark：耗时较长，用户线程可同时运行，标记至GC Root有可达路径的对象  
3，重新标记 CMS remark该步会`stop the world`,但耗时非常短。
    &amp;&amp;&amp;&amp;由于步骤2中用户线程会同步运行，此时主要修正因步骤2中用户线程同步运行产生的对象标记变动  
4，并发清除 CMS concurrent sweep：耗时较长，用户线程可同时运行

在耗时很长的并发标记阶段和并发清除阶段用户线程和收集线程都可同时工作，故而总体上来说，CMS收集器的内存回收是与用户线程一起并发执行的


#### jvm下常用命令
+ jps(等同ps -ef)
+ jstack(查看线程的运行情况和线程当前状态)
+ jmap(监视进程运行中的jvm物理内存的占用情况，该进程内存内，所有对象的情况，例如产生了哪些对象，对象数量)
+ jinfo(观察进程运行环境参数，包括Java System属性和JVM命令行参数)
+ jstat(利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控)   
   &amp;&amp;如 :  jstat -gc 17970 2000 20 (每隔2秒监控一次，共20次)

*http://www.cnblogs.com/alipayhutu/archive/2012/08/20/2647353.html*


#### 查看jvm某线程的堆栈信息
jstack 进程号 | grep -A30 [16进制的线程号]
```
"http-8081-11" daemon prio=0x00002aab049a18000x52f1 in Object.wait() [0x0000000042c75000
java.lang.Thread.State: WAITING (on object monitor)       
at java.lang.Object.wait(Native Method)  
at java.lang.Object.wait(Object.java:
at org.apache.tomcat.util.net.JIoEndpoint$Worker.await(JIoEndpoint.java:
```

#### jvm常见配置值
+ -Xms(jvm最小虚拟内存)
+ -Xmx(jvm最大内存)
+ -XX:PermSize(最小堆大小)
+ -XX:MaxPermSize(最大堆大小)
+ -Xmn(年轻代大小)
+ -XX:SurvivorRatio(控制年轻代中Eden区与survivor区的比例)
+ -XX:NewRatio(年轻代与老年代的占比,比如值为3时,400M的内存,年轻代为100M,老年代为300M) 
+ -Xss(设置每个线程的堆栈大小)


#### jvm内存关系
虚拟机内存=堆内存(Xmx) +  永久区（MaxPermSize）  
整个JVM内存大小=年轻代大小(Xmn) + 年老代大小+ 持久代大小[永久区]  
堆内存 = 年轻代大小(Xmn) + 年老代大小  
方法区[永久区]用于存放Class的相关信息，如类名，访问修饰符，常量池，字段描述，方法描述等。。

