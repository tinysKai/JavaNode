## Linux性能常见命令

#### sar

```shell
# 默认不指定参数时查看cpu, 1代表每秒输出1次, 3代表一共输出3次
#默认查看cpu选项等同于加上参数 `-u`
[root@wujy-tggqj ~]# sar 1 3
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

09:59:09 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
09:59:10 AM     all      0.38      0.00      2.50      0.00      0.00     97.12
09:59:11 AM     all      0.00      0.00      0.25      0.00      0.00     99.75
09:59:12 AM     all      0.13      0.00      0.13      0.00      0.00     99.75
Average:        all      0.17      0.00      0.96      0.00      0.00     98.87


# 加上参数d表示查看磁盘数据
[root@wujy-tggqj ~]# sar -d 2 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:00:13 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
10:00:15 AM  dev253-0      4.00      0.00     32.00      8.00      0.00      0.25      0.12      0.05
10:00:15 AM dev253-16      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:00:15 AM dev253-32      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

Average:          DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
Average:     dev253-0      3.33      0.00     26.67      8.00      0.00      0.40      0.20      0.07
Average:    dev253-16      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:    dev253-32      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

# 加上`-n DEV`查看网络信息
# IFACE表示网卡
# rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是 PPS
# rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是 BPS
[root@wujy-tggqj ~]# sar -n DEV 1 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:01:30 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
10:01:31 AM       br0     48.00      0.00      2.43      0.00      0.00      0.00      0.00
10:01:31 AM ovs-system      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:01:31 AM    br-int      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:01:31 AM      eth0     61.00     15.00      3.93      1.09      0.00      0.00      0.00
10:01:31 AM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:01:31 AM     mgmt0     61.00     15.00      3.10      1.01      0.00      0.00      0.00
10:01:31 AM genev_sys_6081      0.00      0.00      0.00      0.00      0.00      0.00      0.00

Average:        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
Average:          br0     46.33      0.00      2.37      0.00      0.00      0.00      0.00
Average:    ovs-system      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:       br-int      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:         eth0     56.67     11.33      3.66      1.29      0.00      0.00      0.00
Average:           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:        mgmt0     56.67     11.33      2.89      1.25      0.00      0.00      0.00
Average:    genev_sys_6081      0.00      0.00      0.00      0.00      0.00      0.00      0

# 加上`-b`参数查看io方面的统计情况
[root@wujy-tggqj ~]# sar -b 1 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:03:39 AM       tps      rtps      wtps   bread/s   bwrtn/s
10:03:40 AM     38.00      0.00     38.00      0.00    304.00
Average:        38.00      0.00     38.00      0.00    304.00


# 间隔 1 秒输出一组数据
# -r 表示显示内存使用情况，-S 表示显示 Swap 使用情况
$ sar -r -S 1
04:39:56    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:57      6249676   6839824   1919632     23.50    740512     67316   1691736     10.22    815156    841868         4

04:39:56    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:57      8388604         0      0.00         0      0.00


# kbcommit，表示当前系统负载需要的内存。它实际上是为了保证系统内存不溢出，
#     对需要内存的估计值。%commit，就是这个值相对总内存的百分比。

# kbactive，表示活跃内存，也就是最近使用过的内存，一般不会被系统回收。

# kbinact，表示非活跃内存，也就是不常访问的内存，有可能会被系统回收。
```

#### vmstat

```shell
# 默认查看CPU性能
[root@wujy-tggqj ~]# vmstat 1 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0 1895096 129832    460 5134900    0    0     0     1    0    0  0  0 100  0  0
 
# cs（context switch）是每秒上下文切换的次数。

# in（interrupt）则是每秒中断的次数。

# r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。

# b（Blocked）则是处于不可中断睡眠状态的进程数。 
```

#### pidstat

```shell
# 默认查看CPU参数
[root@wujy-tggqj ~]# pidstat 1 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:35:34 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:35:35 AM     0       595    0.00    1.00    0.00    1.00     1  java
10:35:35 AM     0      1800    0.00    1.00    0.00    1.00     3  pidstat

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0       595    0.00    1.00    0.00    1.00     -  java
Average:        0      1800    0.00    1.00    0.00    1.00     -  pidstat

# `-d`参数查看磁盘相关信息
[root@wujy-tggqj ~]# pidstat -d 1 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:37:18 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:37:19 AM     0     20710      0.00     15.84      0.00  java

Average:      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
Average:        0     20710      0.00     15.84      0.00  java


# -d 展示 I/O 统计数据，-p 指定进程号，间隔 1 秒输出 3 组数据
$ pidstat -d -p 4344 1 3
06:38:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:38:51        0      4344      0.00      0.00      0.00       0  app
06:38:52        0      4344      0.00      0.00      0.00       0  app
06:38:53        0      4344      0.00      0.00      0.00       0  app



# `-w`查看上下文切换状况
# cswch表示自愿上下文切换次数
# nvcswch表示非自愿上下文切换次数
[root@wujy-tggqj ~]# pidstat -w 1 1
Linux 3.10.0-514.6.2.el7.x86_64 (wujy-tggqj.vclound.com) 	07/08/2019 	_x86_64_	(8 CPU)

10:40:48 AM   UID       PID   cswch/s nvcswch/s  Command
10:40:49 AM     0         3      1.98      0.00  ksoftirqd/0
10:40:49 AM     0         9     66.34      0.00  rcu_sched
10:40:49 AM     0        33      1.98      0.00  ksoftirqd/5
10:40:49 AM     0       328     20.79      0.00  xfsaild/vda5
10:40:49 AM     0       585   1000.00      0.00  rngd
10:40:49 AM   499       676      0.99      0.00  ovsdb-server
10:40:49 AM     0       711      0.99      0.00  kworker/0:2
10:40:49 AM   499       733      1.98      0.00  ovs-vswitchd
10:40:49 AM     0      1277      0.99      0.00  kworker/6:2
10:40:49 AM     0      1627      0.99      0.00  sshd
10:40:49 AM   600      1748      0.99      0.00  zabbix_agentd
10:40:49 AM     0      1791      1.98      0.00  kworker/u16:2
10:40:49 AM     0      1836      0.99      0.00  pidstat
10:40:49 AM     0      1877      1.98      0.00  python
10:40:49 AM     0     11854      0.99      0.00  kworker/1:0
10:40:49 AM     0     27428      2.97      0.00  kworker/5:0
10:40:49 AM     0     28373      1.98      0.00  kworker/7:1

Average:      UID       PID   cswch/s nvcswch/s  Command
Average:        0         3      1.98      0.00  ksoftirqd/0
Average:        0         9     66.34      0.00  rcu_sched
Average:        0        33      1.98      0.00  ksoftirqd/5
Average:        0       328     20.79      0.00  xfsaild/vda5
Average:        0       585   1000.00      0.00  rngd
Average:      499       676      0.99      0.00  ovsdb-server
Average:        0       711      0.99      0.00  kworker/0:2
Average:      499       733      1.98      0.00  ovs-vswitchd
Average:        0      1277      0.99      0.00  kworker/6:2
Average:        0      1627      0.99      0.00  sshd
Average:      600      1748      0.99      0.00  zabbix_agentd
Average:        0      1791      1.98      0.00  kworker/u16:2
Average:        0      1836      0.99      0.00  pidstat
Average:        0      1877      1.98      0.00  python
Average:        0     11854      0.99      0.00  kworker/1:0
Average:        0     27428      2.97      0.00  kworker/5:0
Average:        0     28373      1.98      0.00  kworker/7:1

# -t表示显示进程相关的
[apps@automatic-test-dfaiq ~]$ pidstat -t 1 1
Linux 2.6.32-642.13.1.el6.x86_64 (automatic-test-dfaiq.vclound.com) 	07/11/2019 	_x86_64_	(8 CPU)

09:45:40 AM      TGID       TID    %usr %system  %guest    %CPU   CPU  Command
09:45:41 AM      4468         -    0.95    0.00    0.00    0.95     7  java
09:45:41 AM         -      4660    0.95    0.00    0.00    0.95     6  |__java
09:45:41 AM     15451         -    0.95    0.00    0.00    0.95     7  java
09:45:41 AM         -     15628    0.00    0.95    0.00    0.95     2  |__java
09:45:41 AM         -     15635    0.95    0.00    0.00    0.95     4  |__java
09:45:41 AM     18567         -    0.95    0.00    0.00    0.95     2  python2.6
09:45:41 AM     23941         -    0.00    0.95    0.00    0.95     6  java
09:45:41 AM         -     24011    0.00    0.95    0.00    0.95     2  |__java
09:45:41 AM         -     24012    0.00    0.95    0.00    0.95     4  |__java
09:45:41 AM     25309         -    1.90    2.86    0.00    4.76     6  pidstat
09:45:41 AM         -     25309    1.90    2.86    0.00    4.76     6  |__pidstat
09:45:41 AM     30284         -    0.00    0.95    0.00    0.95     6  java
09:45:41 AM         -     30325    0.95    0.00    0.00    0.95     7  |__java

Average:         TGID       TID    %usr %system  %guest    %CPU   CPU  Command
Average:         4468         -    0.95    0.00    0.00    0.95     -  java
Average:            -      4660    0.95    0.00    0.00    0.95     -  |__java
Average:        15451         -    0.95    0.00    0.00    0.95     -  java
Average:            -     15628    0.00    0.95    0.00    0.95     -  |__java
Average:            -     15635    0.95    0.00    0.00    0.95     -  |__java
Average:        18567         -    0.95    0.00    0.00    0.95     -  python2.6
Average:        23941         -    0.00    0.95    0.00    0.95     -  java
Average:            -     24011    0.00    0.95    0.00    0.95     -  |__java
Average:            -     24012    0.00    0.95    0.00    0.95     -  |__java
Average:        25309         -    1.90    2.86    0.00    4.76     -  pidstat
Average:            -     25309    1.90    2.86    0.00    4.76     -  |__pidstat
Average:        30284         -    0.00    0.95    0.00    0.95     -  java
Average:            -     30325    0.95    0.00    0.00    0.95     -  |__java
```

#### iostat

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

#### lsof

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

