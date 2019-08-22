## VJTool使用建议

#### jstack出线程栈后可使用在线线程分析网站来分析

[fastthread在线分析网站](https://fastthread.io/)

网站的梳理点

+ 线程按照组分类
+ 区分线程状态,如`RUNNING`,`WAITING`,`TIMED_WAITED`
+ 按照线程堆栈区分
+ 消耗CPU的分类
+ 分析死锁
+ 火焰图

效果展示

![](http://ww1.sinaimg.cn/large/8bb38904gy1g664s42okzj218j0m476n.jpg)

#### GC log分析

[gceasy在线分析gc日志](http://www.gceasy.io/)

网站的梳理点

+ JVM信息
+ GC暂停时间统计
+ GC效果展示(Interactive Graphs)
+ 内存泄漏分析
+ 长暂停记录

效果展示

![](http://ww1.sinaimg.cn/large/8bb38904gy1g665c735u4j20bd0ck74p.jpg)

![](http://ww1.sinaimg.cn/large/8bb38904gy1g665e4osazj20yb0mbwgl.jpg)



#### Vjtop

作用 : 观察JVM进程指标及其繁忙线程

优势 : 对比于“先top -H 列出线程，再执行jstack拿到全部线程，再手工换算十与十六进制的线程号”的繁琐过程，vjtop既方便，又可以连续跟踪，更不会因为jstack造成JVM停顿。

[vjtop](https://github.com/vipshop/vjtools/tree/master/vjtop)

常用命令

```shell
#按时间区间内，线程占用的CPU排序，默认显示前10的线程，默认每10秒打印一次
./vjtop.sh <PID>

#按时间区间内，线程占用的CPU排序，默认显示前10的线程，默认每M秒打印一次,一共打印N次
./vjtop.sh <PID> -i M -n N

```

输出示例

输出有用值

+ cpu使用率
+ 内存使用情况
+ 线程使用情况
+ CPU使用率最高的线程

```shell
 PID: 191082 - 17:43:12 JVM: 1.7.0_79 USER: calvin UPTIME: 2d02h
 PROCESS: 685.00% cpu(28.54% of 24 core), 787 thread
 MEMORY: 6626m rss, 6711m peak, 0m swap | DISK: 0B read, 13mB write
 THREAD: 756 live, 749 daemon, 1212 peak, 0 new | CLASS: 15176 loaded, 161 unloaded, 0 new
 HEAP: 630m/1638m eden, 5m/204m sur, 339m/2048m old
 NON-HEAP: 80m/256m/512m perm, 13m/13m/240m codeCache
 OFF-HEAP: 0m/0m direct(max=2048m), 0m/0m map(count=0), 756m threadStack
 GC: 6/66ms/11ms ygc, 0/0ms fgc | SAFE-POINT: 6 count, 66ms time, 5ms syncTime


    TID NAME                                                      STATE    CPU SYSCPU  TOTAL TOLSYS
     23 AsyncAppender-Worker-ACCESSFILE-ASYNC                   WAITING 23.56%  6.68%  2.73%  0.72%
    560 OSP-Server-Worker-4-5                                  RUNNABLE 22.58% 10.67%  1.08%  0.48%
   9218 OSP-Server-Worker-4-14                                 RUNNABLE 22.37% 11.45%  0.84%  0.40%
   8290 OSP-Server-Worker-4-10                                 RUNNABLE 22.36% 11.24%  0.88%  0.41%
   8425 OSP-Server-Worker-4-12                                 RUNNABLE 22.24% 10.72%  0.98%  0.47%
   8132 OSP-Server-Worker-4-9                                  RUNNABLE 22.00% 10.68%  0.90%  0.42%
   8291 OSP-Server-Worker-4-11                                 RUNNABLE 21.80% 10.09%  0.89%  0.41%
   8131 OSP-Server-Worker-4-8                                  RUNNABLE 21.68%  9.77%  0.93%  0.44%
   9219 OSP-Server-Worker-4-15                                 RUNNABLE 21.56% 10.43%  0.90%  0.41%
   8426 OSP-Server-Worker-4-13                                 RUNNABLE 21.35% 10.42%  0.66%  0.31%

 Total  : 668.56% cpu(user=473.25%, sys=195.31%) by 526 atcive threads(which cpu>0.05%)
 Setting: top 10 threads order by CPU, flush every 10s
 Input command (h for help):
```

#### VJDump

[VjDump文档](https://github.com/vipshop/vjtools/tree/master/vjdump)

VJDump的作用是线上JVM数据紧急收集脚本。收集的数据存放在`/tmp/vjtools/vjdump`下.

输出文件列表

+ gc-osp-1080.log
+ jmap_heap-16113-20190820102524.log
+ jmap_histo_live-16113-20190820102524.log
+ jinfo-flags-16113-20190820102524.log
+ jmap_histo-16113-20190820102524.log
+ jstack-16113-20190820102524.log
+ jmap_dump_live-16113-20190820170612.bin(live命令获取的文件)

常用命令

```shell
# 对指定的进程PID进行急诊
vjdump.sh $pid

# 额外收集heap dump信息（jmap -dump:live的信息）
vjdump.sh --liveheap $pid
```

