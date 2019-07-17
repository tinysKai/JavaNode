## Linux性能优化 - 内核线程

### 特殊的内核线程

Linux 在启动过程中，有三个特殊的进程，也就是 PID 号最小的三个进程。

- 0 号进程为 idle 进程，这也是系统创建的第一个进程，它在初始化 1 号和 2 号进程后，演变为空闲任务。当 CPU 上没有其他任务执行时，就会运行它。
- 1 号进程为 init 进程，通常是 systemd 进程，在用户态运行，用来管理其他用户态进程。
- 2 号进程为 kthreadd 进程，在内核态运行，用来管理内核线程。

**查找内核线程的方法**

```shell
ps -ef | grep "\[.*\]"
root         2     0  0 08:14 ?        00:00:00 [kthreadd]
root         3     2  0 08:14 ?        00:00:00 [rcu_gp]
root         4     2  0 08:14 ?        00:00:00 [rcu_par_gp]
...
```

### 常见的内核线程

- **ksoftirqd**：用来处理软中断的内核线程，并且每个 CPU 上都有一个。
- **kswapd0**：用于内存回收。在 [Swap 变高](https://time.geekbang.org/column/article/75797) 案例中，我曾介绍过它的工作原理。
- **kworker**：用于执行内核工作队列，分为绑定 CPU （名称格式为 kworker/CPU86330）和未绑定 CPU（名称格式为 kworker/uPOOL86330）两类。
- **migration**：在负载均衡过程中，把进程迁移到 CPU 上。每个 CPU 都有一个 migration 内核线程。
- **jbd2**/sda1-8：jbd 是 Journaling Block Device 的缩写，用来为文件系统提供日志功能，以保证数据的完整性；名称中的 sda1-8，表示磁盘分区名称和设备号。每个使用了 ext4 文件系统的磁盘分区，都会有一个 jbd2 内核线程。
- **pdflush**：用于将内存中的脏页（被修改过，但还未写入磁盘的文件页）写入磁盘（已经在 3.10 中合并入了 kworker 中）。



### 火焰图

#### 定义

单纯使用`perf`来分析线程调用会比较难判断分析,可使用图形化的火焰图分析内核线程的调用栈

![https://s2.ax1x.com/2019/07/11/ZRNYUf.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4w7m910z0j20zv0lkwnr.jpg)

**横轴与纵轴的含义**

- **横轴表示采样数和采样比例**。一个函数占用的横轴越宽，就代表它的执行时间越长。同一层的多个函数，则是按照字母来排序。
- **纵轴表示调用栈**，由下往上根据调用关系逐个展开。换句话说，上下相邻的两个函数中，下面的函数，是上面函数的父函数。这样，调用栈越深，纵轴就越高。

另外，要注意图中的颜色，并没有特殊含义，只是用来区分不同的函数。

#### 火焰图分析

首先，我们需要生成火焰图。我们先下载几个能从 perf record 记录生成火焰图的工具，这些工具都放在 <https://github.com/brendangregg/FlameGraph> 上面。你可以执行下面的命令来下载：

```shell
$ git clone https://github.com/brendangregg/FlameGraph
$ cd FlameGraph
```

安装好工具后，要生成火焰图，其实主要需要三个步骤：

1. 执行 perf script ，将 perf record 的记录转换成可读的采样记录；
2. 执行 stackcollapse-perf.pl 脚本，合并调用栈信息；
3. 执行 flamegraph.pl 脚本，生成火焰图。

```shell
# 使用管道来一步完成生成火焰图
perf script -i /root/perf.data | ./stackcollapse-perf.pl --all |  ./flamegraph.pl > ksoftirqd.svg
```

