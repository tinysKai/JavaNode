#### info查询配置信息
info  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-info.png) 
 
#### 查询哨兵情况
sentinel sentinels mymaster   
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-sentinel.png)


#### 查询相应的从信息
sentinel slaves mymaster  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-slave.png)

#### 查询对应的主信息
sentinel master mymaster  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-master.png)

#### 返回所有连接到服务器的客户端信息和统计数据
client list  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-list.png)  
![redis](https://github.com/tinysKai/JavaNote/blob/master/image/article/2018/0709/redis-list1.png)

