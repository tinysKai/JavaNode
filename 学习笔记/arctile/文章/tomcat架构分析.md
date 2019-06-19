#### tomcat顶层结构图
+ server server掌管着整个Tomcat的生死大权(一个tomcat只有一个server)
+ service 对外提供服务(一个server能有多个service)
   + Connector 用于处理连接相关的事情，并提供Socket与Request和Response相关的转化(一个service可以有多个Connector,如https,http分别对应不同的connector)
   + Container 用于封装和管理Servlet，以及具体处理Request请求(一个service只有一个Container)


![tomcat](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/tomcat.jpg)  
![tomcat-connect](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/tomcat-connect.jpg)

##### Connector架构图
Connector就是使用ProtocolHandler来处理请求,不同的ProtocolHandler代表不同的连接类型，比如：Http11Protocol使用的是普通Socket来连接的，Http11NioProtocol使用的是NioSocket来连接的。  
>ProtocolHandler的组成
+ Endpoint处理底层的socket网络连接
+ Processor将Endpoint接收到的Socket封装成Request
+ Adapter用于将Request交给Container进行具体的处理
![connector](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/connector.jpg)


#### contain架构图
+ engine 引擎,用来管理站点(一个contain只有一个engine)
+ host   站点,虚拟主机
+ context 代表一个应用
+ wapper  代表一个servlet  

![contain](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/contain.jpg)

*https://zhuanlan.zhihu.com/p/35398064*