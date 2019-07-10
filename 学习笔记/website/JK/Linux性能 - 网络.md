## Linux性能 - 网络

### 网络性能

低层协议是其上的各层网络协议的基础。自然，低层协议的性能，也就决定了高层的网络性能。所以一般情况下，我们需要从上到下，对每个协议层进行性能测试

+ 网络接口层和网络层，它们主要负责网络包的封装、寻址、路由以及发送和接收。在这两个网络协议层中，每秒可处理的网络包数 PPS，就是最重要的性能指标。(转发性能)
+ 传输层的话是主要是测试TCP/UDP一段时间内的平均吞吐量
+ 应用层主要测试 HTTP 服务的每秒请求数、请求延迟、吞吐量以及请求延迟的分布情况等。

### 网络性能分析

+ tcpdump
+ wireshark

#### tcpdump

![https://s2.ax1x.com/2019/07/08/ZDyazd.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4s2vppbpwj21000l3gph.jpg)

![https://s2.ax1x.com/2019/07/08/ZDywQA.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4s2x66eytj20z80nxtd6.jpg)

**tcpdump的输出格式**

`时间戳 协议 源地址. 源端口 > 目的地址. 目的端口 网络包详细信息`

#### wireshark

Wireshark 也是最流行的一个网络分析工具，它最大的好处就是提供了跨平台的图形界面。跟 tcpdump 类似，Wireshark 也提供了强大的过滤规则表达式，同时，还内置了一系列的汇总分析工具。

```shell
# 执行下面的命令，把抓取的网络包保存到 ping.pcap 文件中,然后用wireshark打开它
tcpdump -nn udp port 53 or host 35.190.27.188 -w ping.pcap
```

![https://s2.ax1x.com/2019/07/08/ZDyBLt.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4s30qnxpyj20zu064aec.jpg)

![https://s2.ax1x.com/2019/07/08/ZDyyo8.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4s33ffqi7j2103063792.jpg)

HTTP的三次握手与四次挥手

![https://s2.ax1x.com/2019/07/08/ZDy2WQ.png](http://ww1.sinaimg.cn/large/8bb38904ly1g4s35am7huj20nw0qrk4o.jpg)

### DDOS

#### 定点IP投递的DDOS攻击

找出源IP,丢掉相关的包

```shell
iptables -I INPUT -s 192.168.0.2 -p tcp -j REJECT
```

#### 针对非固定IP

```shell
# 限制 syn 并发数为每秒 1 次
$ iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT

# 限制单个 IP 在 60 秒新建立的连接数为 10
$ iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT

#增大半链接的数量
sysctl -w net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_max_syn_backlog = 1024

# 减少半链接重试次数
sysctl -w net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_synack_retries = 1
```

#### TCP SYN Cookies 

TCP SYN Cookies 也是一种专门防御 SYN Flood 攻击的方法。SYN Cookies 基于连接信息（包括源地址、源端口、目的地址、目的端口等）以及一个加密种子（如系统启动时间），计算出一个哈希值（SHA1），这个哈希值称为 cookie。然后，这个 cookie 就被用作序列号，来应答 SYN+ACK 包，并释放连接状态。当客户端发送完三次握手的最后一次 ACK 后，服务器就会再次计算这个哈希值，确认是上次返回的 SYN+ACK 的返回包，才会进入 TCP 的连接状态。

```shell
# 开启TCP SYN Cookies,临时性 
sysctl -w net.ipv4.tcp_syncookies=1
net.ipv4.tcp_syncookies = 1

# 持久化的配置,需要执行`sysctl -p` 命令后，才会动态生效
cat /etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_max_syn_backlog = 1024

```

