## Linux性能 - IO初篇

#### 文件系统

文件系统是对存储设备上的文件，进行组织管理的机制。组织方式不同，就会形成不同的文件系统。

为了方便管理，Linux 文件系统为每个文件都分配两个数据结构，索引节点（index node）和目录项（directory entry）。它们主要用来记录文件的元信息和目录结构。索引节点是每个文件的唯一标志，而目录项维护的正是文件系统的树状结构。

- 索引节点，简称为 inode，用来记录文件的元数据，比如 inode 编号、文件大小、访问权限、修改日期、数据的位置等。索引节点和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘中。所以记住，索引节点同样占用磁盘空间。
- 目录项，简称为 dentry，用来记录文件的名字、索引节点指针以及与其他目录项的关联关系。多个关联的目录项，就构成了文件系统的目录结构。不过，不同于索引节点，目录项是由内核维护的一个内存数据结构，所以通常也被叫做目录项缓存。

![https://s2.ax1x.com/2019/07/06/Zwe1Ej.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4pvbl5qm2j20n70e7gmm.jpg)

#### 虚拟文件系统

**磁盘内的分类**

- 超级块，存储整个文件系统的状态。
- 索引节点区，用来存储索引节点。
- 数据块区，则用来存储文件数据。

目录项、索引节点、逻辑块以及超级块，构成了 Linux 文件系统的四大基本要素。为了支持各种不同的文件系统，Linux 内核在用户进程和文件系统的中间，又引入了一个抽象层，也就是虚拟文件系统 VFS（Virtual File System）。VFS 定义了一组所有文件系统都支持的数据结构和标准接口。这样，用户进程和内核中的其他子系统，就只需要跟 VFS 提供的统一接口进行交互。

####  Linux 文件系统的架构图

![https://s2.ax1x.com/2019/07/06/ZweXqg.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4pvh3m1oyj20en0g9q3t.jpg)

#### 命令

查询磁盘容量

```shell
#常见的查询方式
df -h 

# -i 参数，查看索引节点的使用情况
df -hi
```



#### IO栈

 Linux 存储系统的 I/O 栈，由上到下分为三个层次，分别是文件系统层、通用块层和设备层。

- 文件系统层，包括虚拟文件系统和其他各种文件系统的具体实现。它为上层的应用程序，提供标准的文件访问接口；对下会通过通用块层，来存储和管理磁盘数据。
- 通用块层，包括块设备 I/O 队列和 I/O 调度器。它会对文件系统的 I/O 请求进行排队，再通过重新排序和请求合并，然后才要发送给下一级的设备层。
- 设备层，包括存储设备和相应的驱动程序，负责最终物理设备的 I/O 操作。

通用块层是 Linux 磁盘 I/O 的核心。向上，它为文件系统和应用程序，提供访问了块设备的标准接口；向下，把各种异构的磁盘设备，抽象为统一的块设备，并会对文件系统和应用程序发来的 I/O 请求进行重新排序、请求合并等，提高了磁盘访问的效率。

#### IO分类

**阻塞 / 非阻塞针对的是 I/O 调用者（即应用程序），而同步 / 异步针对的是 I/O 执行者（即系统）。**

- 所谓阻塞 I/O，是指应用程序在执行 I/O 操作后，如果没有获得响应，就会阻塞当前线程，不能执行其他任务。
- 所谓非阻塞 I/O，是指应用程序在执行 I/O 操作后，不会阻塞当前的线程，可以继续执行其他的任务。

- 所谓同步 I/O，是指收到 I/O 请求后，系统不会立刻响应应用程序；等到处理完成，系统才会通过系统调用的方式，告诉应用程序 I/O 结果。
- 所谓异步 I/O，是指收到 I/O 请求后，系统会先告诉应用程序 I/O 请求已经收到，随后再去异步处理；等处理完成后，系统再通过事件通知的方式，告诉应用程序结果。



#### Linux对IO的优化

+ 优化文件访问的性能，会使用页缓存、索引节点缓存、目录项缓存等多种缓存机制，以减少对下层块设备的直接调用。

+ 优化块设备的访问效率，会使用缓冲区，来缓存块设备的数据。

#### 磁盘性能指标

- 使用率，是指磁盘处理 I/O 的时间百分比。过高的使用率（比如超过 80%），通常意味着磁盘 I/O 存在性能瓶颈。
- 饱和度，是指磁盘处理 I/O 的繁忙程度。过高的饱和度，意味着磁盘存在严重的性能瓶颈。当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。
- IOPS（Input/Output Per Second），是指每秒的 I/O 请求数。
- 吞吐量，是指每秒的 I/O 请求大小。
- 响应时间，是指 I/O 请求从发出到收到响应的间隔时间。

在数据库、大量小文件等这类随机读写比较多的场景中，IOPS 更能反映系统的整体性能；而在多媒体等顺序读写较多的场景中，吞吐量才更能反映系统的整体性能。



#### 磁盘性能检测

```shell
# c选项输出CPU相关数据
[tinys@localhost ~]$ iostat -c 
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	07/06/19 	_x86_64_	(1 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.43    0.00    1.41    3.06    0.00   95.10

# d选项表示输出磁盘相关数据
[tinys@localhost ~]$ iostat -d 
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	07/06/19 	_x86_64_	(1 CPU)

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               8.28       242.74        23.06     303301      28814

# 使用x选项输出统计信息
[tinys@localhost ~]$ iostat -d -x 1
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	07/06/19 	_x86_64_	(1 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              14.77     5.59    9.58    2.24   355.88    31.27    65.54     0.13   10.87    8.07   22.86   4.11   4.85

[tinys@localhost ~]$ iostat -c -x
Linux 2.6.32-431.el6.x86_64 (localhost.localdomain) 	07/06/19 	_x86_64_	(1 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.39    0.00    1.28    2.74    0.00   95.59

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              10.45     4.16    6.78    1.80   251.83    23.81    64.28     0.09   10.83    8.07   21.21   4.04   3.47           

```

**指标解释**

- %util ，就是我们前面提到的磁盘 I/O 使用率；
- r/s+ w/s ，就是 IOPS；
- rkB/s+wkB/s ，就是吞吐量；
- r_await+w_await ，就是响应时间。

![https://s2.ax1x.com/2019/07/06/Zw7I7d.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4q5guofj9j20pp0qe45z.jpg)

#### 进程IO观测

**实时观测的pidstat**

```shell
# kB_rd/s每秒读取的数据大小,单位是KB
# kB_wr/s每秒发出的写请求数据大小,单位是KB
# 每秒取消的写请求数据大小（kB_ccwr/s） ，单位是 KB。
# 块 I/O 延迟（iodelay），包括等待同步块 I/O 和换入块 I/O 结束的时间，单位是时钟周期。
pidstat -d 1 
13:39:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
13:39:52      102       916      0.00      4.00      0.00       0  rsyslogd

```

**进程排序的iotop**

```shell
# 前两行分别表示，进程的磁盘读写大小总数和磁盘真实的读写大小总数。
# 因为缓存、缓冲区、I/O 合并等因素的影响，它们可能并不相等。

# 剩下的部分，则是从各个角度来分别表示进程的 I/O 情况，
# 包括线程 ID、I/O 优先级、每秒读磁盘的大小、每秒写磁盘的大小、换入和等待 I/O 的时钟百分比等。

iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       7.85 K/s 
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s 
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND 
15055 be/3 root        0.00 B/s    7.85 K/s  0.00 %  0.00 % systemd-journald 

```

**lsof观察具体进程打开了哪些文件**

```shell
# -p 指定进程id

#输出参数解释
#	FD 文件描述符
#	TYPE 文件类型
#	NAME 文件路径

## 注意 FD中最后一个`3w`表示它的文件描述符是 3 号，而 3 后面的 w ，表示以写的方式打开
lsof -p 18940 
COMMAND   PID USER   FD   TYPE DEVICE  SIZE/OFF    NODE NAME 
python  18940 root  cwd    DIR   0,50      4096 1549389 / 
python  18940 root  rtd    DIR   0,50      4096 1549389 / 
… 
python  18940 root    2u   CHR  136,0       0t0       3 /dev/pts/0 
python  18940 root    3w   REG    8,1 117944320     303 /tmp/logtest.txt 

```

**具体查询buffer与cache具体各占用了多少的命令**

```shell
[root@localhost ~]# cat /proc/meminfo  | head -5
MemTotal:        1012352 kB
MemFree:          507836 kB
Buffers:           61392 kB
Cached:           252092 kB
SwapCached:            0 kB
```

