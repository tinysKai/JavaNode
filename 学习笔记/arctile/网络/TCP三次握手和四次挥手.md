#### TCP三次握手
![tcp](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/tcp-3.gif)

#### TCP传输
![tcp](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/tcp-4.gif)  

TCP链接是双工的.双方都可以主动发起数据传输。  
不过无论是哪方喊话，都需要收到对方的确认(ack)才能认为对方收到了自己的喊话。
传输过程的确认不一定需要一条条应答,可以批量确认(ack).    
虽然有批量ack,但传输双方还是会需协商好的合适的发送和接受速率,不然接收方可能短时间内无法消化太多消息,这就是`TCP窗口大小` 


#### TCP四次挥手
![tcp](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/tcp-5.gif)  

TCP挥手需要四次一个主要原因是因为服务方接收关闭请求时无法立刻关闭连接(可能还有需回复给接收方的消息),  
故先回复一个收到关闭请求的应答,待确实关闭了连接时再发送fin信号  

time_wait状态是主动关闭的一方在回复完对方的挥手后进入的一个长期状态，这个状态标准的持续时间是4分钟.4分钟后才会进入到closed状态，释放套接字资源  
time_wait的作用是重传最后一个ack报文，确保对方可以收到。因为如果对方没有收到ack的话，会重传fin报文，处于time_wait状态的套接字会立即向对方重发ack报文   


*http://dwz.cn/BNhOWojt*


