#### 常用命令
```
1.启动nginx：
    进入sbin目录下，./nginx

2.停止nginx：
    i.从容停止
    kill -QUIT nginx master进程号
    ii.快速停止
    kill -TERM nginx master进程号
    kill -INT nginx master进程号
    iii.强制停止
    kill -9 nginx master进程号

3.重启nginx：
    ./nginx -s reload

    sudo service nginx restart

4.测试配置文件是否有错
    ./nginx -t

```

#### 日志配置
日志格式 log_format指令

    语法: log_format name string …;  
    默认值: log_format combined “…”;  
    配置段: http

一个日志格式例子
```
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for" "$upstream_addr" '
                          '$upstream_response_time $request_time'; 

    
    main表示这个日志格式的名字
    
    解释$request_time和$upstream_response_time
    
    $request_time
    官网描述：request processing time in seconds with a milliseconds resolution; 
                time elapsed between the first bytes were read from the client and the log write after the last bytes were sent to the client 
    翻译 : 指的就是从接受用户请求的第一个字节到发送完响应数据的时间，即包括接收请求数据时间、程序响应时间、输出响应数据时间。
    我译 : 从nginx接收到请求到nginx响应返回结束
    
    $upstream_response_time
    官网描述：keeps times of responses obtained from upstream servers; times are kept in seconds with a milliseconds resolution. 
                Several response times are separated by commas and colons like addresses in the $upstream_addr variable
    翻译 : 是指从Nginx向后端建立连接开始到接受完数据然后关闭连接为止的时间
    我译 : nginx与后端的整个交付时间
    
    
    nginx的整个交付流程如下 : 
         前端 --1-->nginx --2-->后端
         
         前端 <--4--nginx <--3--后端
    
    $request_time是1到4的时间;$upstream_response_time是2到3的流程

```

>日志格式生效需在日志中声明日志格式的名字,如  
access_log  log/access.log  main;  
access_log的默认配置 :access_log logs/access.log combined;


#### 缓存模块

代理过程中，有两个连接的速度会影响客户端体验：
+ 从客户端到Nginx的连接
+ 从Nginx到后端的连接


没有缓冲的情况下，数据直接从后端服务器发送给客户端。如果客户端的连接速度快，则可以关闭缓冲以提高数据发送速度。
缓冲的作用是在Nginx上临时存储来自后端服务器的处理结果，从而可以提早关闭Nginx到后端的连接，这比较适合客户端连接较慢的情况。

Nginx默认启用缓冲，因为客户端的连接速度一般来说是差别很大的.  
缓冲的具体配置可以通过如下条目修改，这些条目可以放在http server或location内容块下。
需要注意的是，涉及缓冲大小的条目是针对请求配置的，如果设置的比较高，则请求数很多的时候容易造成性能问题：

	* proxy_buffering：控制本内容块下（包括子内容块）是否启用缓冲，默认为“on”。
	* proxy_buffers：有两个参数，第一个控制缓冲区请求数量，第二个控制缓冲区大小。默认值为8个、一页（一般是4k或8k）。这个值越大，缓冲的内容越多。
	* proxy_buffer_size：后端回复结果的首段（包含header的部分）是单独缓冲的，本条目定义这部分的缓冲区大小。这个值默认与proxy_buffer的值相同，我们可以把它设置的小一些，因为header内容一般比较少。
	* proxy_busy_buffers_size：设置被标记为“client-ready”（客户端就绪）的缓冲区大小。客户端一次只能从一个缓冲读取数据，而缓冲是按照队列次序被分批发送给客户端的。本条目设置的值就是这个队列的大小。
	* proxy_max_temp_file_size：每个请求可以存储临时文件的最大大小。如果上游发来的结果太大以至于无法放入一个缓冲，则Nginx会为其创建临时文件。
	* proxy_temp_path：定义Nginx存储临时文件的路径。


如上所述，Nginx提供了不少关于缓冲的配置项。大部分配置项我们都不必太在意，可能就是proxy_buffers和proxy_buffer_size值得调整一下。

#### 超时配置
```
    proxy_connect_timeout
    语法 proxy_connect_timeout time 
    默认值 60s
    后端服务器连接的超时时间_发起握手等候响应超时时间
    
    
    proxy_read_timeout
    语法 proxy_read_timeout time 
    默认值 60s
    连接成功后_等候后端服务器响应时间_其实已经进入后端的排队之中等候处理（也可以说是后端服务器处理请求的时间）
    
    proxy_send_timeout
    语法 proxy_send_timeout time 
    默认值 60s
    后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据
    
    proxy_next_upstream
    默认值  proxy_next_upstream http_502 http_504 error timeout invalid_header;
    出现值里面的错误进行在集群的另一台机重试...
    http://phl.iteye.com/blog/2247162
    注意 : 其中有一个参数值 timeout，这个参数代表如果超时，则尝试其他节点。如果要确保超时不重新链接则去掉这个值

```
    
    
##### TCP
    TCP_NODELAY : 设置不缓存立刻传输请求
    tcp_nopush : 配置一次发送数据的包大小,当数据超过该大小时才发送数据..在 nginx 中，tcp_nopush 必须和 sendfile 搭配使用。
    sendfile : 一种IO优化策略
    http://blog.csdn.net/liuxiao723846/article/details/52634622
    
    
    
##### rewrite的flag标志
+ last
+ break
+ redirect
+ permanent 

last     :  停止在本location继续处理,将处理完的URI重新在所有location继续处理  
break    :  将重写的URI继续在本location中继续执行,不会跳转到其它location中
redirect :  将重写后的URI返回给客户端,状态码为302,指明是临时重定向URI  
permanent:  将重写后的URI返回给客户端,状态码为301,指明是永久重定向URI


##### proxy_pass的注意事项
```
server{
    listen 80;
    server_name : www.abc.com
    location /context/
    {
        ...
        proxy_pass http://192.168.1.1;    //注意这里结尾没"/"
    }
}

如果这时候使用"http://www.abc.com/context"访问发起请求,转向地址为"http://192.168.1.1/context" 
```

```
server{
    listen 80;
    server_name : www.abc.com
    location /context/
    {
        ...
        proxy_pass http://192.168.1.1/anotherContext/;    //注意这里结尾有"/"
    }
}   
如果这时候使用"http://www.abc.com/context"访问发起请求,转向地址为"http://192.168.1.1/anotherContext/" 
```

总结 : 如果不想改变原地址中的context就不要在proxy_pass末尾加"/"


#### location配置
语法规则： location [=|~|~*|^~] /uri/ { … }
+ = 开头表示精确匹配
+ ^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。
+ ~ 开头表示区分大小写的正则匹配
+ ~* 开头表示不区分大小写的正则匹配
+ !~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
+ / 通用匹配，任何请求都会匹配到。

顺序优先级  

(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)