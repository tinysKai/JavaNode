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



```

#### vmstat

```shell
# 默认查看CPU性能
[root@wujy-tggqj ~]# vmstat 1 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0 1895096 129832    460 5134900    0    0     0     1    0    0  0  0 100  0  0
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
```

