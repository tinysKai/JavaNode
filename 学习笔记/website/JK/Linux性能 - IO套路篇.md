## Linux性能 - IO套路篇

#### 性能指标

- **使用率**，是指磁盘忙处理 I/O 请求的百分比。过高的使用率（比如超过 60%）通常意味着磁盘 I/O 存在性能瓶颈。
- **IOPS**（Input/Output Per Second），是指每秒的 I/O 请求数。
- **吞吐量**，是指每秒的 I/O 请求大小。
- **响应时间**，是指从发出 I/O 请求到收到响应的间隔时间

![https://s2.ax1x.com/2019/07/07/ZBmmJs.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4r5r20h2oj21bc0dm41c.jpg)

#### 性能工具

![https://s2.ax1x.com/2019/07/07/ZBnUAg.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4r60r79esj20o60qo7al.jpg)

![https://s2.ax1x.com/2019/07/07/ZBn5g1.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4r64ye3psj20im0qon1v.jpg)



#### 性能分析套路

![https://s2.ax1x.com/2019/07/07/ZBKDmV.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4r6rqiabpj21h30qfn19.jpg)



#### 命令

```shell
# 查看索引节点容量,使用容量以及剩余容量
[tinys@localhost ~]$ df -i
Filesystem      Inodes  IUsed  IFree IUse% Mounted on
/dev/sda2      1164592 184863 979729   16% /
tmpfs           126544      3 126541    1% /dev/shm
/dev/sda1        76912     38  76874    1% /boot
```

