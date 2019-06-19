#### linux系统 IO复路模型

	select,poll,epoll都是同步IO..
	select,poll有限制连接数..epoll木有..
	select,poll是基于遍历查询fd的,epoll是回调通知,查询速度快..
	epoll是共享内存,而select与poll都需要将内核数据拷贝到用户空间
	

#### IO多路复用
+ select 单个进程能够监视的文件描述符的数量存在最大限制,因使用轮询的方式所以IO的效率不会随着监视fd的数量的增长而下降
+ poll 单个进程能够监视的文件描述符的数量存在最大限制,因使用轮询的方式所以IO的效率不会随着监视fd的数量的增长而下降
+ epoll  监视的描述符数量不受限制,基于回调来查找文件描述符
	

#### 网络模式方案
- 阻塞 I/O（blocking IO）
- 非阻塞 I/O（nonblocking IO）
- I/O 多路复用（ IO multiplexing）
- 异步 I/O（asynchronous IO）[用的也不多]	
- 信号驱动 I/O（ signal driven IO）[实际中基本不用]  





