##  Linux性能优化 - CPU实战

### 平均负载

#### 定义

 	平均负载是指单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数，它和 CPU 使用率并没有直接关系。可以简单理解为平均负载其实就是平均活跃进程数。

`uptime`或`top`命令均可查看当前系统负载

```shell
[tinys@localhost ~]$ uptime
 01:06:56 up 1 min,  1 user,  load average: 0.34, 0.22, 0.09

#执行结果解释
# 01:06:56 当前时间
# up 1 min 启动一分钟
# 1 user 当前系统上有一个用户
# load average: 0.34, 0.22, 0.09 平均负载,依次是一分钟,五分钟,十五分钟的平均负载

[tinys@localhost ~]$ grep 'model name' /proc/cpuinfo | wc -l
1

# 查看当前系统的逻辑CPU数(即核数),得出逻辑CPU数为1,即表示平均负载为1时系统刚好完全被占用
```



#### 平均负载与 CPU 使用率

平均负载是指单位时间内，处于可运行状态和不可中断状态的进程数。所以，它不仅包括了**正在使用 CPU** 的进程，还包括**等待 CPU** 和**等待 I/O** 的进程。而 CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。

比如：

- CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
- I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
- 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

### 实战

#### 工具准备

`stress `是一个 Linux 系统压力测试工具，这里我们用作异常进程模拟平均负载升高的场景。而 `sysstat` 包含了常用的 Linux 性能工具，用来监控和分析系统的性能。

```shell
# 安装软件
# 安装stress,先到`http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el7/en/x86_64/rpmforge/RPMS/stress-1.0.2-1.el7.rf.x86_64.rpm`下载rpm文件
# 执行`rpm -Uvh stress-1.0.2-1.el7.rf.x86_64.rpm`命令
[root@localhost ~]# rpm -Uvh stress-1.0.2-1.el7.rf.x86_64.rpm
warning: stress-1.0.2-1.el7.rf.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 6b8d79e6: NOKEY
Preparing...                ########################################### [100%]
   1:stress                 ########################################### [100%]
   
# 安装sysstat
[root@localhost ~]$ wget http://pagesperso-orange.fr/sebastien.godard/sysstat-11.5.5.tar.gz
[root@localhost ~]$ tar -xvf sysstat-11.0.0.tar.gz 
[root@localhost ~]$ cd sysstat-11.0.0
[root@localhost sysstat-11.0.0]$ ./configure
[root@localhost sysstat-11.0.0]$ make
[root@localhost sysstat-11.0.0]$ make install

# 检查版本号
[root@localhost sysstat-11.0.0]$ mpstat -V
sysstat version 9.0.4
(C) Sebastien Godard (sysstat <at> orange.fr)
[root@localhost sysstat-11.0.0]$ 

```



#### CPU密集场景

```shell
#模拟cpu 100%的场景
[root@localhost sysstat-11.0.0]$ stress --cpu 1 --timeout 600
```

使用`uptime`持续观察平均负载

```shell
[root@localhost sysstat-11.0.0]$ watch -d uptime
#输入命令后会跳到到以下显示
Every 2.0s: uptime                                                                                              Sat Jun 29 02:39:07 2019

 02:39:07 up  1:34,  4 users,  load average: 0.86, 0.75, 0.13
```

使用`mpstat`观察所有cpu使用情况

如果是cpu密集型,则`usr`列会彪高;若是io密集型,则`iowait`会彪高

```shell
# -P ALL 表示监控所有 CPU，后面数字 5 表示间隔 5 秒后输出一组数据
[tinys@localhost ~]$ mpstat -P ALL 5
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/2019 	_x86_64_	(1 CPU)

02:32:55 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
02:33:00 AM  all   97.98    0.00    1.52    0.00    0.00    0.51    0.00    0.00    0.00
02:33:00 AM    0   97.98    0.00    1.52    0.00    0.00    0.51    0.00    0.00    0.00

# 第二组数据观察到有一组cpu的usr(cpu使用率)达到100%,而iowait为0,说明平均负载的升高正是由于 CPU 使用率为 100% 。
02:33:00 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
02:33:05 AM  all  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
02:33:05 AM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00



```

使用`pidstat`来查询时哪一个进程导致的cpu彪高

```shell
# 间隔 5 秒后输出一组数据,-u表示cpu指标
[tinys@localhost ~]$ pidstat -u 5 1
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/2019 	_x86_64_	(1 CPU)
#这里我们看到有一个进程的cpu使用率达到了400%
02:34:42 AM       PID    %usr %system  %guest    %CPU   CPU  Command
02:34:47 AM      5004  433.04    0.87    0.00  433.91     0  stress

Average:          PID    %usr %system  %guest    %CPU   CPU  Command
Average:         5004  433.04    0.87    0.00  433.91     -  stress


#新版本的sysstat的pidstat会带有%wait,建议源码安装时选择比较新的版本
[root@localhost sysstat-11.5.5]# pidstat -u 5 1
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/19 	_x86_64_	(1 CPU)

03:05:15      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
03:05:21      500      2320    0.00    0.20    0.00    0.00    0.20     0  sshd
03:05:21      500      5007    0.20    0.00    0.00    0.00    0.20     0  watch
03:05:21        0      7947    0.00    0.40    0.00    0.00    0.40     0  pidstat
```



### CPU上下文

#### 定义

`CPU 寄存器`，是 CPU 内置的容量小、但速度极快的内存。而`程序计数器`，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 **CPU 上下文**。

`CPU 上下文切换`就是先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。而这些保存下来的上下文，会存储在系统内核中，并在任务重新调度执行时再次加载进来。这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。

#### CPU 的上下文切换场景

+ 进程上下文切换
+ 线程上下文切换
+ 中断上下文切换

#### 系统调用

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间.从用户态到内核态的转变，需要通过**系统调用**来完成。比如，当我们查看文件内容时，就需要多次系统调用来完成：首先调用 open() 打开文件，然后调用 read() 读取文件内容，并调用 write() 将内容写到标准输出，最后再调用 close() 关闭文件。

一次系统调用的过程，其实是发生了两次 CPU 上下文切换。先是从用户态切换到内核态,再从内核态切换到用户态.

#### 进程上下文切换

进程上下文切换，是指从一个进程切换到另一个进程运行。而系统调用过程中一直是同一个进程在运行。

进程的上下文切换就比系统调用时多了一步：在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。

#### 线程上下文切换

**线程是调度的基本单位，而进程则是资源拥有的基本单位**。

各种情况的线程上下文切换

- 当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换时是不需要修改的。不过线程间的私有栈和寄存器等不共享数据是需要切换的
- 不同进程里的线程切换,因为资源不共享，所以切换过程就跟进程上下文切换是一样

#### 中断上下文切换

为了快速响应硬件的事件，**中断处理会打断进程的正常调度和执行**，转而调用中断处理程序，响应设备事件。而在打断其他进程时，就需要将进程当前的状态保存下来，这样在中断结束后，进程仍然可以从原来的状态恢复运行。

跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等。

#### 查看上下文切换情况

**查看系统整体的上下文情况**

```shell
[tinys@localhost ~]$ vmstat 3
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 684748  44736 109240    0    0   993    54  194  308  3  8 77 12  0	
 0  0      0 684724  44736 109240    0    0     0     0   20   28  0  0 100  0  0	
 0  0      0 684724  44736 109240    0    0     0     0   21   32  0  0 100  0  0	

# cs（context switch）是每秒上下文切换的次数。

# in（interrupt）则是每秒中断的次数。

# r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。

# b（Blocked）则是处于不可中断睡眠状态的进程数。
```

**查看各个进程的上下文切换**

```shell
[tinys@localhost ~]$ pidstat -w 5
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/19 	_x86_64_	(1 CPU)

06:12:19      UID       PID   cswch/s nvcswch/s  Command
06:12:24        0         4      1.00      0.00  ksoftirqd/0
06:12:24        0         7      2.19      0.00  events/0
06:12:24        0        13      0.20      0.00  sync_supers
06:12:24       42      2254      0.20      0.00  gnome-power-man
06:12:24      500      2495      0.20      0.40  pidstat

# cswch ，表示每秒自愿上下文切换（voluntary context switches）的次数，
# nvcswch ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。
```

 **上下文切换种类**

- **自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换**。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
- **非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换**。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

#### 实战

安装`sysbench`

```shell
# 相应的命令
wget http://imysql.com/wp-content/uploads/2014/09/sysbench-0.4.12-1.1.tgz
 tar -vzxf sysbench-0.4.12-1.1.tgz
 cd sysbench-0.4.12-1.1
 ./configure --without-mysql
 make 
 make install
```

**压测前空闲状态下的数据**

```shell
[tinys@localhost ~]$ vmstat 2
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 541648  50228 214252    0    0    68    16   41   61  1  1 98  1  0	
 0  0      0 541508  50228 214252    0    0     0    38   27   41  0  1 100  0  0	
 0  0      0 541516  50228 214252    0    0     0     0   23   32  0  0 100  0  0	
```

**压测脚本**

```shell
[tinys@localhost ~]$ sysbench --num-threads=10 --max-time=300 --max-requests=100000 --test=threads run
sysbench 0.5:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 10
Random number generator seed is 0 and will be ignored


Threads started!
```

**压测时的观察数据**

发现`cs`暴增,`in`显著增加..同时`r`列也开始增加了..表示运行中的任务数,因为我们开了10个线程.所以数字接近10.

然后cpu相关的`us`以及`sy`之和接近100了

```shell
[tinys@localhost ~]$ vmstat 2
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs       us sy id wa st
 7  0      0 541444  50228 214252    0    0    62    15   44 7242  1    1  97  1  0	
 8  0      0 541404  50228 214252    0    0     0     0 1008 2093172    11 90  0  0  0	
 9  0      0 541156  50228 214252    0    0     0     0 1011 1985582    10 91  0  0  0	
 8  0      0 541156  50228 214252    0    0     0     0 1008 2044638    10 91  0  0  0	
 8  0      0 541156  50228 214252    0    0     0     0 1007 2030035    10 91  0  0  0	
```

**观察具体的进程**

```shell
# 每隔 1 秒输出 1 组数据（需要 Ctrl+C 才结束）
# -w 参数表示输出进程切换指标，而 -u 参数则表示输出 CPU 使用指标
[tinys@localhost ~]$ pidstat -w -u 1
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/19 	_x86_64_	(1 CPU)

07:14:43      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
07:14:44        0      1905    0.00    0.60    0.00    1.20    0.60     0  tpvmlp
07:14:44      500     15005    6.02   54.22    0.00    0.00   60.24     0  sysbench
07:14:44      500     15011    0.00    0.60    0.00    0.60    0.60     0  pidstat

#发现下面列出的上下文切换远不及vmstat汇总的几百万次
07:14:43      UID       PID   cswch/s nvcswch/s  Command
07:14:44        0         7      3.01      0.00  events/0
07:14:44        0        13      0.60      0.00  sync_supers
07:14:44        0       155      0.60      0.00  mpt_poll_0
07:14:44        0       795      1.20      0.00  vmmemctl
07:14:44        0      1317      9.04      0.00  vmtoolsd
07:14:44        0      1905      1.20      0.60  tpvmlp
07:14:44       42      2242      0.60      0.00  gnome-settings-
07:14:44       42      2254      0.60      0.00  gnome-power-man
07:14:44      500     15011      0.60      1.81  pidstat

# 想到sysbench使用的是线程,我们pidstat查看的是进程还是线程呢?
# 每隔 1 秒输出一组数据（需要 Ctrl+C 才结束）
# -wt 参数表示输出线程的上下文切换指标
[tinys@localhost ~]$ pidstat -wt  1
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	06/29/19 	_x86_64_	(1 CPU)

07:20:03      UID      TGID       TID   cswch/s nvcswch/s  Command
07:20:04        0         4         -      1.74      0.00  ksoftirqd/0
07:20:04        0         -         4      1.74      0.00  |__ksoftirqd/0
07:20:04        0         7         -      1.74      0.00  events/0
07:20:04        0         -         7      1.74      0.00  |__events/0
07:20:04        0       155         -      0.87      0.00  mpt_poll_0
07:20:04        0         -       155      0.87      0.00  |__mpt_poll_0
07:20:04        0       795         -      0.87      0.00  vmmemctl
07:20:04        0         -       795      0.87      0.00  |__vmmemctl
07:20:04        0      1317         -     10.43      0.00  vmtoolsd
07:20:04        0         -      1317     10.43      0.00  |__vmtoolsd
07:20:04       42      2242         -      0.87      0.87  gnome-settings-
07:20:04       42         -      2242      0.87      0.87  |__gnome-settings-
07:20:04       42      2254         -      0.87      0.00  gnome-power-man
07:20:04       42         -      2254      0.87      0.00  |__gnome-power-man
07:20:04      499         -      2272      0.87      0.87  |__rtkit-daemon
07:20:04      499         -      2273      0.87      0.00  |__rtkit-daemon
07:20:04      500         -     15034      0.00 482781.74  |__sysbench
07:20:04      500         -     15035      0.00 480595.65  |__sysbench
07:20:04      500         -     15036      0.00 477966.09  |__sysbench
07:20:04      500         -     15037      0.00 476346.96  |__sysbench
07:20:04      500         -     15038      0.00 481623.48  |__sysbench
07:20:04      500     15043         -      0.87     12.17  pidstat
07:20:04      500         -     15043      0.87     12.17  |__pidstat

# 这一次很明显监控线程之后发现确实是sysbench导致的上下文切换
```

#### 上下文切换问题汇总

- 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；
- 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；
- 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型。