## Java性能调优学习笔记

#### 系统常见性能指标

+ 响应时间
+ 吞吐量
+ 计算机资源使用率
  + cpu
  + 内存
  + IO
  + 磁盘
+ 负载承受能力



#### String优化

**使用intern来节省内存**

字符串常量默认会将对象放入常量池中;在字符串变量中，对象是会创建在堆内存中，同时也会在常量池中创建一个字符串对象，String 对象中的 char 数组将会引用常量池中的 char 数组，并返回堆内存对象引用。

如果调用 `intern` 方法，会去查看字符串常量池中是否有等于该对象的字符串的引用.如果常量池中木有该字符串引用,则将它添加到常量池中;如果有,则返回常量池中的字符串引用;

#### LinkedList遍历

使用Iterator来遍历链表,如使用for循环来遍历链表的话,每循环一次就需要遍历一次指定节点前的数据,源码如下

```java
// 获取双向链表中指定位置的节点
private Entry<E> entry(int index) {
    if (index < 0 || index >= size)
        throw new IndexOutOfBoundsException("Index: "+index+", Size: "+size);
    Entry<E> e = header;
    // 获取index处的节点。
    // 若index < 双向链表长度的1/2,则从前先后查找;
    // 否则，从后向前查找。
    if (index < (size >> 1)) {
        for (int i = 0; i <= index; i++)
            e = e.next; //重点在这里的循环遍历
    } else {
        for (int i = size; i > index; i--)
            e = e.previous; //重点在这里的循环遍历
    }
    return e;
}
```

而iterator在第一次拿到一个数据后，之后的循环中会使用Iterator中的next()方法采用的是顺序访问。



#### ab压测工具

安装 `yum-y install httpd-tools`

**说明文档如下**:

![ac58706f86ebd1d7349561ae501fca0a.png](http://ww1.sinaimg.cn/large/8bb38904gy1gc6dploxxhj20j50fpmyt.jpg)

**常见测试脚本** :

```shell
#测试并发用户数为 10、请求数量为 100 的的 post 请求输入如下,-p表示post 参数文档路径（-p 和 -T 参数要配合使用）；
ab -n 100  -c 10 -p 'post.txt' -T 'application/x-www-form-urlencoded' 'http://test.api.com/test/register'

# 测试get接口
ab -c 10 -n 100 http://www.test.api.com/test/login?userName=test&password=test
```

**结果展示如下**

![66e7cf2dafa91a3ae80405f97a91899b.png](http://ww1.sinaimg.cn/large/8bb38904gy1gc6ds0j9f2j20q20ge0u2.jpg)

以上输出中，有几项性能指标可以提供给你参考使用：

+ Requests per second：吞吐率，指某个并发用户数下单位时间内处理的请求数；
+ Time per request：上面的是用户平均请求等待时间，指处理完成所有请求数所花费的时间 /（总请求数 / 并发用户数）；
+ Time per request：下面的是服务器平均请求处理时间，指处理完成所有请求数所花费的时间 / 总请求数；
+ Percentage of the requests served within a certain time：每秒请求时间分布情况，指在整个请求中，每个请求的时间长度的分布情况，例如有 50% 的请求响应在 8ms 内，66% 的请求响应在 10ms 内，说明有 16% 的请求在 8ms~10ms 之间。

#### Stream流

![ea8dfeebeae8f05ae809ee61b3bf3094.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gc6lzrtmlkj21kk13yjvk.jpg)

#### Java序列化的缺陷

+ 无法跨语言
+ 安全性差,易被攻击
+ 序列化后流太大
+ 序列化性能差



#### Syn锁

JVM 中的同步是基于进入和退出管程（Monitor）对象实现的。因 Monitor 是依赖于底层的操作系统实现，存在用户态与内核态之间的切换，所以增加了性能开销。

为了提升性能，JDK1.6 引入了偏向锁、轻量级锁、重量级锁概念，来减少锁竞争带来的上下文切换，而正是新增的 Java 对象头实现了锁升级功能。

**对象头**

在 JDK1.6 JVM 中，对象实例在堆内存中被分为了三个部分：对象头、实例数据和对齐填充。其中 Java 对象头由 Mark Word、指向类的指针以及数组长度三部分组成。以下为Mark Word的储存结构,

![fd86f1b5cbac1f652bea58b039fbc8f8.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gc8m9rhk9kj213m0da0vp.jpg)

**锁类型**

+ 偏向锁  -- 减少同一线程多次申请同一锁时的性能开销
+ 轻量级锁(会自旋) - 利用CAS来获取锁
+ 重量级锁 -- 阻塞等待锁

偏向锁的作用就是，当一个线程再次访问这个同步代码或方法时，该线程只需去对象头的 Mark Word 中去判断一下是否有偏向锁指向它的 ID，无需再进入 Monitor 去竞争对象了。

**JVM锁参数**

```java
-XX:-UseBiasedLocking //关闭偏向锁（默认打开）
-XX:-UseSpinning //参数关闭自旋锁优化(默认打开)    
```

**锁升级**

![d9f1e7fae6996a940e9471c47a455ba2.png](http://ww1.sinaimg.cn/large/8bb38904gy1gc8n8daz6nj20t11m3436.jpg)

#### 线程池

1.可以通过调用`ThreadPoolExecutor`的`prestartAllCoreThreads`来对线程池提前预热,适合秒杀抢购等场景

2.可使用`allowCoreThreadTimeOut`属性来允许线程池的核心线程在一定时间后进行`core thread`回收,默认不不回收核心线程

#### 编译优化技术

JIT 编译运用了一些经典的编译优化技术来实现代码的优化，即通过一些例行检查优化，可以智能地编译出运行时的最优性能代码。

**1.方法内联**

调用一个方法通常要经历压栈和出栈。调用方法是将程序执行顺序转移到存储该方法的内存地址，将方法的内容执行完后，再返回到执行该方法前的位置。这种执行操作要求在执行前保护现场并记忆执行的地址，执行后要恢复现场，并按原来保存的地址继续执行。 因此，方法调用会产生一定的时间和空间方面的开销。

`方法内联的优化行为就是把目标方法的代码复制到发起调用的方法之中，避免发生真实的方法调用。`

```java
//如以下代码调用
private int add1(int x1, int x2, int x3, int x4) {
    return add2(x1, x2) + add2(x3, x4);
}
private int add2(int x1, int x2) {
    return x1 + x2;
}

//经过方法内联之后,会被优化为
private int add1(int x1, int x2, int x3, int x4) {
    return x1 + x2+ x3 + x4;
}
```

JVM 会自动识别热点方法，并对它们使用方法内联进行优化。我们可以通过 -XX:CompileThreshold 来设置热点方法的阈值。但要强调一点，热点方法不一定会被 JVM 做内联优化，如果这个方法体太大了，JVM 将不执行内联操作。而方法体的大小阈值，我们也可以通过参数设置来优化：

> 经常执行的方法，默认情况下，方法体大小小于 325 字节的都会进行内联，我们可以通过 -XX:MaxFreqInlineSize=N 来设置大小值；
>
> 不是经常执行的方法，默认情况下，方法大小小于 35 字节才会进行内联，我们也可以通过 -XX:MaxInlineSize=N 来重置大小值。

之后我们就可以通过配置 JVM 参数来查看到方法被内联的情况：

> -XX:+PrintCompilation //在控制台打印编译过程信息
>
> -XX:+UnlockDiagnosticVMOptions //解锁对JVM进行诊断的选项参数。默认是关闭的，开启后支持一些特定参数对JVM进行诊断
>
> -XX:+PrintInlining //将内联方法打印出来

一般我们可以通过以下几种方式来提高方法内联：

+ 通过设置 JVM 参数来减小热点阈值或增加方法体阈值，以便更多的方法可以进行内联，但这种方法意味着需要占用更多地内存；
+ 在编程中，避免在一个方法中写大量代码，习惯使用小方法体；
+ 尽量使用 final、private、static 关键字修饰方法，编码方法因为继承，会需要额外的类型检查。

**编译优化技术**

+ 方法内联(将目标方法优化为方法体内的方法调用,减少方法调用切换时的开销)
+ 栈上分配(逃逸分析如果发现一个对象只在方法中使用，就会将对象分配在栈上)
+ 锁消除(如局部方法中的StringBuffer被优化为StringBuilder)
+ 标量替换(逃逸分析证明一个对象不会被外部访问，如果这个对象可以被拆分的话，当程序真正执行的时候可能不创建这个对象，而直接创建它的成员变量来代替)



#### SQL调优

**Show Profile**

`Show Profile`可深入到内核中,从执行线程的状态和时间来分析

```mysql

SHOW PROFILE [type [, type] ... ]
[FOR QUERY n]
[LIMIT row_count [OFFSET offset]]

type参数：
| ALL：显示所有开销信息
| BLOCK IO：阻塞的输入输出次数
| CONTEXT SWITCHES：上下文切换相关开销信息
| CPU：显示CPU的相关开销信息 
| IPC：接收和发送消息的相关开销信息
| MEMORY ：显示内存相关的开销，目前无用
| PAGE FAULTS ：显示页面错误相关开销信息
| SOURCE ：列出相应操作对应的函数名及其在源码中的调用位置(行数) 
| SWAPS：显示swap交换次数的相关开销信息
```

常见命令

```sql
select @@have_profiling -- 查询是否支持该功能

-- 默认情况Show Profiles只显示最近15条SQL语句,我们可以重新设置 profiling_history_size 增大该存储记录，最大值为 100。

-- 常见的查询过程为通过`show profiles`来查询Query ID,再通过`Show Profile for Query ID`来查看具体的详情
```

![5488fde01df647508d60b9a77cd1f14f.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gcb26ecq4oj212e0rsjwm.jpg)

![dc7e4046ddd22438c21690e5bc38c123.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gcb2617nwnj217s0u2juq.jpg)

