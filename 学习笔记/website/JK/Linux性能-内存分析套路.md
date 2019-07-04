#### Linux性能-内存分析套路

#### 内存指标

![https://s2.ax1x.com/2019/07/05/ZanmXF.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4olo2sh5ij20lw0pd0v8.jpg)

#### 指标工具指南

![https://s2.ax1x.com/2019/07/05/ZanM79.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4oltzx6agj20o60teq6k.jpg)

![https://s2.ax1x.com/2019/07/05/ZanlkR.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4olvuw9cpj20sy0tt0zq.jpg)

#### 分析

![https://s2.ax1x.com/2019/07/05/Zan1t1.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4olyzd705j21c00pcwhw.jpg)

#### 内存优化思路

1. 最好禁止 Swap。如果必须开启 Swap，降低 swappiness 的值，减少内存回收时 Swap 的使用倾向。
2. 减少内存的动态分配。比如，可以使用内存池、大页（HugePage）等。
3. 尽量使用缓存和缓冲区来访问数据。比如，可以使用堆栈明确声明内存空间，来存储需要缓存的数据；或者用 Redis 这类的外部缓存组件，优化数据的访问。
4. 使用 cgroups 等方式限制进程的内存使用情况。这样，可以确保系统内存不会被异常进程耗尽。
5. 通过 /proc/pid/oom_adj ，调整核心应用的 oom_score。这样，可以保证即使内存紧张，核心应用也不会被 OOM 杀死。