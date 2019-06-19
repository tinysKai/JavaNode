《图解HTTP》读书笔记
=

#### TCP/IP协议族分层
+ 应用层 : 应用层决定了向用户提供应用服务时通信的活动 [FTP,DNS,HTTP]
+ 传输层 : 传输层对上层应用层，提供处于网络连接中的两台计算机之间的数据传输 [TCP,UDP]
+ 网络层 : 处理在网络上流动的数据包.该层规定了通过怎样的路径（所谓的传输路线）到达对方计算机，并把数据包传送给对方
+ 数据链路层 : 处理连接网络的硬件部分.包括控制操作系统、硬件的设备驱,网卡等.硬件上的范畴都在链路层的作用范围之内

TCP/IP通信传输流  
![tcp](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/tcp01.png) 

 
数据信息封装  
![tcp](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/tcp02.png)   

URI用字符串标识某一互联网资源,而URL表示资源的地点.可见 URL 是 URI 的子集.


#### HTTP协议

##### HTTP报文
请求报文是由请求方法、请求 URI、协议版本、可选的请求首部字段和内容实体构成的。  
  
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http01.png)  
 
 
响应报文基本上由协议版本、状态码（表示请求成功或失败的数字代码）、用以解释状态码的原因短语、可选的响应首部字段以及实体主体构成。  

![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http02.png)    


HTTP 是一种不保存状态，即无状态（stateless）协议.  


##### 持久连接  
持久连接的特点是，只要任意一端没有明确提出断开连接，则保持 TCP 连接状态.  
持久连接旨在建立 1 次 TCP 连接后进行多次请求和响应的交互.  
在 HTTP/1.1 中，所有的连接默认都是持久连接.    
持久连接使得管道化技术得到发展.以前请求需发送一个请求等待一个响应,现在能同时做到并行发送多个请求,而不需要一个接一个地等待响应了.  


##### 首部类型
+ 通用首部(请求报文和响应报文两方都会使用的首部)
   - Cache-Control(控制缓存的行为)
   - Connection(逐跳首部、连接的管理)
   - Date(创建报文的日期时间)
+ 请求首部
    - Accept(用户代理可处理的媒体类型)
    - Accept-Encoding(优先的内容编码)
    - Host
    - User-Agent
    - Range
+ 响应首部
    - Location(令客户端重定向至指定URI)
    - Server(HTTP服务器的安装信息)
+ 实体首部(针对请求报文和响应报文的实体部分使用的首部)
    - Content-Length(实体主体的大小（单位：字节）) 
    - Content-Type(实体主体的媒体类型)
    - Expires(实体主体过期的日期时间)
    - Last-Modified(资源的最后修改日期时间)

##### 范围请求(Range Request) 
对一份 10 000 字节大小的资源，如果使用范围请求，可以只请求5001~10 000 字节内的资源。  
执行范围请求时，会用到首部字段 Range 来指定资源的 byte 范围.  
针对范围请求，响应会返回状态码为 206 Partial Content 的响应报文.  
如果服务器端无法响应范围请求，则会返回状态码 200 OK 和完整的实体内容。  

#### web服务器
网关
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http03.png)   
网关能使通信线路上的服务器提供非 HTTP 协议服务。  
利用网关能提高通信的安全性，因为可以在客户端与网关之间的通信线路上加密以确保连接的安全
  

##### 通用首部
Cache-Control
```
Cache-Control: no-cache
 
使用 no-cache 指令的目的是为了防止从缓存中返回过期的资源  
客户端发送的请求中如果包含 no-cache 指令，则表示客户端将不会接收缓存过的响应。
于是，“中间”的缓存服务器必须把客户端请求转发给源服务器。  

 
如果服务器返回的响应中包含 no-cache 指令，那么缓存服务器不能对资源进行缓存。
源服务器以后也将不再对缓存服务器请求中提出的资源有效性进行确认，
且禁止其对响应资源进行缓存操作。

```


```
Cache-Control: no-cache=Location  
 
若响应字段中对no-cache字段具体指定了某个值,则客户端不能此字段进行缓存. 
其它无声明的字段依旧可以进行缓存操作.
```


max-age
```
客户端角度
    当客户端发送的请求中包含 max-age 指令时，如果判定缓存资源的缓
  存时间数值比指定时间的数值更小，那么客户端就接收缓存的资源。
  另外，当指定 max-age 值为 0，那么缓存服务器通常需要将请求转发给源服务器。  
 
服务端角度 
    当服务器返回的响应中包含 max-age 指令时，缓存服务器将不对资源
  的有效性再作确认，而 max-age 数值代表资源保存为缓存的最长时间。
 

```



Connection首部字段作用
+ 控制不再转发给代理的首部字段
+ 管理持久连接

```
HTTP/1.1 版本的默认连接都是持久连接。为此，客户端会在持久连接上连续发送请求。
当服务器端想明确断开连接时，则指定Connection 首部字段的值为 Close。
```


#### HTTPS
http的缺点
+ 通信使用明文（不加密），内容可能会被窃听
+ 不验证通信方的身份，因此有可能遭遇伪装
+ 无法证明报文的完整性，所以有可能已遭篡改  

##### 内容加密
HTTP通过和 SSL（Secure Socket Layer，安全套接层）或TLS（Transport Layer Security，安全层传输协议）
的组合使用加密HTTP的通信内容。  
用 SSL 建立安全通信线路之后，就可以在这条线路上进行 HTTP通信了。
与 SSL 组合使用的 HTTP 被称为 HTTPS（HTTPSecure，超文本传输安全协议）或 HTTP over SSL。


##### HTTPS = HTTP+ 加密 + 认证 + 完整性保护
通常，HTTP 直接和 TCP 通信。当使用 SSL 时，则演变成先和 SSL通信，再由 SSL 和 TCP 通信了。  
简言之，所谓 HTTPS，其实就是身披SSL协议这层外壳的HTTP。  
在采用 SSL 后，HTTP 就拥有了 HTTPS 的加密、证书和完整性保护这些功能。

##### 服务端发布公钥流程
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http04.png)     
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http05.png)     

##### https交互流程
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http06.png)


#### 身份认证
HTTP认证方式
+ BASIC认证(基本认证,服务端返回401状态码要求客户端进行验证,客户端输入用户,密码进行base64加密传递验证,安全性差不常用)
+ DIGEST认证(摘要认证,不传递密码而是采用传递密码与随机值得摘要来验证身份)
+ SSL客户端认证
+ FormBase认证(基于表单认证)  

SSL认证  
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/http07.png)


#### WebSocket
WebSocket，即 Web 浏览器与 Web 服务器之间全双工通信标准。  
由于是建立在 HTTP 基础上的协议，因此连接的发起方仍是客户端，
而一旦确立 WebSocket 通信连接，不论服务器还是客户端，任意一方都可直接向对方发送报文。  

WebSocket主要特点
+ 推送功能(支持由服务器向客户端推送数据的推送功能)
+ 减少通信量(只要建立起 WebSocket 连接，就希望一直保持连接状态, WebSocket的首部信息很小,通信量也相应减少了)
 
 
为了实现WebSocket通信，在HTTP连接建立之后，需要完成一次“握手”（Handshaking）的步骤。  
为了实现WebSocket通信，需要用到HTTP的Upgrade首部字段，告知服务器通信协议发生改变，以达到握手的目的。  
成功握手确立 WebSocket 连接之后，通信时不再使用HTTP的数据帧，而采用 WebSocket 独立的数据帧。  
![http](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/websocket01.png)
 