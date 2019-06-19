#### 背景
在redis内部有一个哨兵(sentinel)监测master和slave的状态，当有一个master节点宕机的时候，  
sentinel就会在剩下的slave中选择一个合适的节点作为master  

>raft协议在redis中只运用于选主,没运用到分布式锁或分布式事务中


#### raft协议动画
*http://thesecretlivesofdata.com/raft/*


#### redis哨兵的选主过程
>一台哨兵发现主崩溃-主观崩溃  
![raft](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/raft1.png)  
  

>半数以上哨兵发现主崩溃-客观崩溃  
![raft](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/raft2.png)


>故障转移-哨兵投票  
首先slave11和slave12变成候选者，等待一个0到1s内的随机值，然后向sentinel集群的每一个节点发送求票信息  
![raft](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/raft3.png)
 

>选主结束-主从关系转移  
![raft](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/raft4.png)  

>持续监听原故障主节点-恢复时转换为从监听新主  
![raft](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/raft5.png)

*https://blog.csdn.net/sanwenyublog/article/details/53385616*