## Linux性能优化 - CPU问题分析

#### CPU使用率

#### 定义

![http://ww1.sinaimg.cn/large/8bb38904ly1g4ivsyq29xj20ft02eglr.jpg](https://s2.ax1x.com/2019/06/30/Zl5IgO.png)

#### 查看cpu使用率

**使用top观察cpu的总使用情况**

top 默认显示的是所有 CPU 的平均值，这个时候你只需要按下数字 1 ，就可以切换到每个 CPU 的使用率了。

![http://ww1.sinaimg.cn/large/8bb38904ly1g4iw0486foj20mt06zq3d.jpg](https://s2.ax1x.com/2019/06/30/Zl5LVA.png)

**使用pidstat观察具体的cpu使用分布情况**

下面的命令表示每2秒输出一次,共输出5次.在最下面还算了下期间的平均值

![http://ww1.sinaimg.cn/large/8bb38904ly1g4iw48whdbj20rj0dtt9n.jpg](https://s2.ax1x.com/2019/06/30/Zl5x8f.png)

**CPU使用率情况分析**

- 用户 CPU 和 Nice CPU 高，说明用户态进程占用了较多的 CPU，所以应该着重排查进程的性能问题
- 系统 CPU 高，说明内核态占用了较多的 CPU，所以应该着重排查内核线程或者系统调用的性能问题
- I/O 等待 CPU 高，说明等待 I/O 的时间比较长，所以应该着重排查系统存储是不是出现了 I/O 问题
- 软中断和硬中断高，说明软中断或硬中断的处理程序占用了较多的 CPU，所以应该着重排查内核中的中断服务程序

**查找进程的父进程**
`pstree` 

`pstree | grep ${keyword}`

### 不可中断进程排查

#### 定义

中断是系统用来响应硬件设备请求的一种机制，它会打断进程的正常调度和执行，然后调用内核中的中断处理程序来响应设备的请求。

#### 进程状态

top命令就能显示

```shell
top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
28961 root      20   0   43816   3148   4040 R   3.2  0.0   0:00.01 top
  620 root      20   0   37280  33676    908 D   0.3  0.4   0:00.01 app
    1 root      20   0  160072   9416   6752 S   0.0  0.1   0:37.64 systemd
 1896 root      20   0       0      0      0 Z   0.0  0.0   0:00.00 devapp
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.10 kthreadd
    4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
    6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 mm_percpu_wq
    7 root      20   0       0      0      0 S   0.0  0.0   0:06.37 ksoftirqd/0

```

- **R** 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。
- **D** 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。
- **Z** 是 Zombie 的缩写，它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。
- **S** 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。
- **I** 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。

####  dstat

**安装**

```shell
#yum直接安装报错
[root@localhost ~]$ yum install dstat
Loaded plugins: fastestmirror, refresh-packagekit, security
Loading mirror speeds from cached hostfile
Error: Cannot retrieve metalink for repository: epel. Please verify its path and try again
[root@localhost ~]$ dstat
-bash: dstat: command not found

#修改yum源
vim /etc/yum.repos.d/epel.repo
#将以下mirrorlist行注释掉,开启baseurl的代码
[epel]
name=Extra Packages for Enterprise Linux 6 - $basearch
baseurl=http://download.fedoraproject.org/pub/epel/6/$basearch
#mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-6&arch=$basearch
failovermethod=priority

#重新安装
[root@localhost ~]$ yum clean all
Loaded plugins: fastestmirror, refresh-packagekit, security
Cleaning repos: akopytov_sysbench akopytov_sysbench-source base epel extras updates
Cleaning up Everything
Cleaning up list of fastest mirrors
[root@localhost ~]$ yum install dstat
```

**用处**

dstat ，它的好处是，可以同时查看 CPU 和 I/O 这两种资源的使用情况，便于对比分析。

![http://ww1.sinaimg.cn/large/8bb38904ly1g4j582w9uij20ls051aa8.jpg](https://s2.ax1x.com/2019/06/30/Z1eszt.png)

查看某个进程的资源具体使用情况可以使用`pidstat`

```shell
# -d 展示 I/O 统计数据，-p 指定进程号，间隔 1 秒输出 3 组数据
$ pidstat -d -p 4344 1 3
06:38:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:38:51        0      4344      0.00      0.00      0.00       0  app
06:38:52        0      4344      0.00      0.00      0.00       0  app
06:38:53        0      4344      0.00      0.00      0.00       0  app

```

#### perf

**定义**

linux内核提供的性能监测调优工具。

**安装**

`yum install perf`

**使用**

```shell
perf top
perf record
perf report

#性能调优
# Sample on-CPU functions for the specified command, at 99 Hertz:
perf record -F 99 command

# Sample on-CPU functions for the specified PID, at 99 Hertz, until Ctrl-C:
perf record -F 99 -p PID

# Sample on-CPU functions for the specified PID, at 99 Hertz, for 10 seconds:
perf record -F 99 -p PID sleep 10

# 对所有cpu(-a)的以99HZ(-F 99)频率采样30s(-- sleep 30)，采样信息包括stack,记录调用栈(-g)
perf record -F 99 -a -g -- sleep 30             
```

**学习资料**

[perf](https://github.com/patpatbear/notes/blob/master/linux-perf.md)

### 软中断

#### 定义

为了解决中断处理程序执行过长和中断丢失的问题，Linux 将中断处理过程分成了两个阶段

- **上半部用来快速处理中断**，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作
- **下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行**

上半部直接处理硬件请求，也就是我们常说的**硬中断**，特点是快速执行；而下半部则是由内核触发，也就是我们常说的**软中断**，特点是延迟执行。

#### 查看软中断和内核线程

proc 文件系统是一种内核空间和用户空间进行通信的机制，可以用来查看内核的数据结构，或者用来动态修改内核的配置。可通过`proc`文件系统来查看中断情况

- /proc/softirqs 提供了软中断的运行情况；
- /proc/interrupts 提供了硬中断的运行情况。

```shell
cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:     811613    1972736			#定时中断
      NET_TX:         49          7			#网络发送中断
      NET_RX:    1136736    1506885			#网络接收中断
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     304787       3691
       SCHED:     689718    1897539			#内核调度
     HRTIMER:          0          0
         RCU:    1330771    1354737			#RCU锁

```

#### 命令实战

**sar**

sar 可以用来查看系统的网络收发情况，还有一个好处是，不仅可以观察网络收发的吞吐量（BPS，每秒收发的字节数），还可以观察网络收发的 PPS，即每秒收发的网络帧数。

```shell
# -n DEV 表示显示网络收发的报告，间隔 1 秒输出一组数据
$ sar -n DEV 1
15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
15:03:47    veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05

# IFACE表示网卡
# rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是 PPS
# rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是 BPS

```

**tcpdump**

```shell
# -i eth0 只抓取 eth0 网卡，-n 不解析协议名和主机名
# tcp port 80 表示只抓取 tcp 协议并且端口号为 80 的网络帧
$ tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.0.2.18238 > 192.168.0.30.80: Flags [S], seq 458303614, win 512, length 0
...

# Flags [S] 则表示这是一个 SYN 包
```

