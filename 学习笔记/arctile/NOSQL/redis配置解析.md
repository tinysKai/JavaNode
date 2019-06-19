##### 指定redis运行的端口，默认是6379  
  port 6379
  
##### 指定redis只接收来自于该IP地址的请求，如果不进行设置，那么将处理所有请求
  bind 127.0.0.1
   
##### 设置客户端连接时的超时时间，单位为秒。当客户端在这段时间内没有发出任何指令，那么关闭该连接
   timeout 0  //0是关闭此设置
   
##### 主从复制.设置该数据库为其他数据库的从数据库. 
设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步  
    slaveof masterip masterport
    
##### 最大客户端连接数
    设置同一时间最大客户端连接数，默认无限制，
    Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，
    如果设置 maxclients 0，表示不作限制。
    当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息

maxclients 128    

##### 最大内存限制
     指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key
     Redis同时也会移除空的list对象
    
     当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作
     
     注意：Redis新的vm机制，会把Key存放内存，Value会存放在swap区
    
     maxmemory的设置比较适合于把redis当作于类似memcached的缓存来使用，而不适合当做一个真实的DB。
     当把Redis当做一个真实的数据库使用的时候，内存使用将是一个很大的开销
 maxmemory <bytes>
 
 *https://blog.csdn.net/ithomer/article/details/9232891*
   