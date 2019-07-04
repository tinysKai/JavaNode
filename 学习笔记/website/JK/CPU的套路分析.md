## CPU的套路分析

#### CPU性能指标

+ CPU使用率
  + 用户CPU
  + 系统CPU
  + 等待IO_CPU
  + 软中断
  + 硬中断
+ 平均负载
  + 理想情况下，平均负载等于逻辑 CPU 个数，这表示每个 CPU 都恰好被充分利用。
+ 进程上下文
  + 无法获取资源而导致的自愿上下文切换
  + 被系统强制调度导致的非自愿上下文切换

**指标汇总图**

![https://s2.ax1x.com/2019/07/02/ZJLsiT.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4lsddlb47j20p40oiq4y.jpg)

#### 性能指标图

![https://s2.ax1x.com/2019/07/02/ZJLxTP.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4lsifv25qj20lr0tmwll.jpg)

#### 工具指标指南

![https://s2.ax1x.com/2019/07/02/ZJOf1g.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4lswcbrqrj20pm0qggt6.jpg)

#### CPU分析指南

![https://s2.ax1x.com/2019/07/02/ZJXCAx.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4lt0wrthhj20pe0sywgm.jpg)