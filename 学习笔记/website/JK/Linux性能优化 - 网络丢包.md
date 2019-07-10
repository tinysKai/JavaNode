## Linux性能优化 - 网络丢包追踪

### 定义

​     丢包是指在网络数据的收发过程中，由于种种原因，数据包还没传输到应用程序中，就被丢弃了。这些被丢弃包的数量，除以总的传输包数，也就是我们常说的**丢包率**。丢包率是网络性能中最核心的指标之一。

​    丢包通常会带来严重的性能下降，特别是对 TCP 来说，丢包通常意味着网络拥塞和重传，进而还会导致网络延迟增大、吞吐降低。

### 网络丢包的原理

![https://s2.ax1x.com/2019/07/10/Zg8KZF.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4v1t5bvc2j20zz0l4n1x.jpg)

### 丢包分析

#### 链路层

当缓冲区溢出等原因导致网卡丢包时，Linux 会在网卡收发数据的统计信息中，记录下收发错误的次数。

```shell
root@nginx:/# netstat -i
Kernel Interface table
Iface      MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0       100       31      0      0 0             8      0      0      0 BMRU
lo       65536        0      0      0 0             0      0      0      0 LRU

# RX-OK、接收时的总包数
# RX-ERR、总错误数
# RX-DRP、进入 Ring Buffer 后因其他原因（如内存不足）导致的丢包数
# RX-OVR ，Ring Buffer 溢出导致的丢包数。
```

#### 网络层与传输层

使用 `netstat -s` 命令查看协议的收发汇总，以及错误信息.

netstat 汇总了 IP、ICMP、TCP、UDP 等各种协议的收发统计信息。不过，我们的目的是排查丢包问题，所以这里主要观察的是错误数、丢包数以及重传数。

```shell
root@nginx:/# netstat -s
Ip:
    Forwarding: 1					// 开启转发
    31 total packets received		// 总收包数
    0 forwarded						// 转发包数
    0 incoming packets discarded	// 接收丢包数
    25 incoming packets delivered	// 接收的数据包数
    15 requests sent out			// 发出的数据包数
Icmp:
    0 ICMP messages received		// 收到的 ICMP 包数
    0 input ICMP message failed		// 收到 ICMP 失败数
    ICMP input histogram:
    0 ICMP messages sent			//ICMP 发送数
    0 ICMP messages failed			//ICMP 失败数
    ICMP output histogram:
Tcp:
    0 active connection openings	// 主动连接数
    0 passive connection openings	// 被动连接数
    11 failed connection attempts	// 失败连接尝试数
    0 connection resets received	// 接收的连接重置数
    0 connections established		// 建立连接数
    25 segments received			// 已接收报文数
    21 segments sent out			// 已发送报文数
    4 segments retransmitted		// 重传报文数
    0 bad segments received			// 错误报文数
    0 resets sent					// 发出的连接重置数
Udp:
    0 packets received
    ...
TcpExt:
    11 resets received for embryonic SYN_RECV sockets	// 半连接重置数
    0 packet headers predicted
    TCPTimeouts: 7		// 超时数
    TCPSynRetrans: 4	//SYN 重传数
	...

# 观察如上输出发现只有 TCP 协议发生了丢包和重传，分别是：

# 11 次连接失败重试（11 failed connection attempts）
# 4 次重传（4 segments retransmitted）
# 11 次半连接重置（11 resets received for embryonic SYN_RECV sockets）
# 4 次 SYN 重传（TCPSynRetrans）
# 7 次超时（TCPTimeouts）

#这个结果告诉我们，TCP 协议有多次超时和失败重试，
# 并且主要错误是半连接重置。换句话说，主要的失败，都是三次握手失败。
```

