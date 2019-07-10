## Linux性能优化 -- 网络优化套路

#### 性能目标

网络性能优化的整体目标，是降低网络延迟（如 RTT）和提高吞吐量（如 BPS 和 PPS）.但不同应用标准可能会不同.

+ NAT网关 - 通常需要达到或接近线性转发，也就是说， PPS 是最主要的性能目标
+ 数据库/缓存 - 快速完成网络收发，即低延迟，是主要的性能目标
+ WEB服务 - 需要同时兼顾吞吐量和延迟

#### 网络协议栈

![https://s2.ax1x.com/2019/07/10/Z6gNQO.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4udnjdhrej20lu0r1aau.jpg)

**由于底层是其上方各层的基础，底层性能也就决定了高层性能。**

+ `网络接口层和网络层`，它们主要负责网络包的封装、寻址、路由，以及发送和接收。每秒可处理的网络包数 PPS，就是它们最重要的性能指标（特别是在小包的情况下）。
+ `传输层的 TCP 和 UDP`，它们主要负责网络传输。对它们而言，吞吐量（BPS）、连接数以及延迟，就是最重要的性能指标。你可以用 iperf 或 netperf ，来测试传输层的性能。
+ `应用层`，最需要关注的是吞吐量（BPS）、每秒请求数以及延迟等指标。你可以用 wrk、ab 等工具，来测试应用程序的性能。

#### 网络性能工具

![https://s2.ax1x.com/2019/07/10/Z6g0wd.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4udqwcqh7j20nj0qpjw2.jpg)

![https://s2.ax1x.com/2019/07/10/Z6gsYt.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4udrzz3d2j20k10qpgp9.jpg)

#### 网络IO优化思路

+  I/O 多路复用技术 epoll，主要用来取代 select 和 poll
+ 使用异步 I/O（Asynchronous I/O，AIO）

#### 网络协议优化

- 使用长连接取代短连接，可以显著降低 TCP 建立连接的成本。在每秒请求次数较多时，这样做的效果非常明显。
- 使用内存等方式，来缓存不常变化的数据，可以降低网络 I/O 次数，同时加快应用程序的响应速度。
- 使用 Protocol Buffer 等序列化的方式，压缩网络 I/O 的数据量，可以提高应用程序的吞吐。
- 使用 DNS 缓存、预取、HTTPDNS 等方式，减少 DNS 解析的延迟，也可以提升网络 I/O 的整体速度。

#### 套接字优化

发送缓冲区大小，理想数值是吞吐量 * 延迟，这样才可以达到最大网络利用率。

![https://s2.ax1x.com/2019/07/10/Z6gblT.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4ue9wmxabj20z90jr78t.jpg)

#### 优化套接字的网络连接

- 为 TCP 连接设置 TCP_NODELAY 后，就可以禁用 Nagle 算法；
- 为 TCP 连接开启 TCP_CORK 后，可以让小包聚合成大包后再发送（注意会阻塞小包的发送）；
- 使用 SO_SNDBUF 和 SO_RCVBUF ，可以分别调整套接字发送缓冲区和接收缓冲区的大小。

####  传输层优化

传输层最重要的是 TCP 和 UDP 协议.要优化传输层协议需先熟练掌握TCP/UDP协议先.否则只是纸上谈兵.

TCP 提供了面向连接的可靠传输服务。要优化 TCP，我们首先要掌握 TCP 协议的基本原理，比如流量控制、慢启动、拥塞避免、延迟确认以及状态流图（如下图所示）等。

![https://s2.ax1x.com/2019/07/10/Zg30g0.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4v1juezf1j20kt0qv79y.jpg)

#### 网络层

网络层，负责网络包的封装、寻址和路由，包括 IP、ICMP 等常见协议。在网络层，最主要的优化，其实就是对路由、 IP 分片以及 ICMP 等进行调优。

优化略

