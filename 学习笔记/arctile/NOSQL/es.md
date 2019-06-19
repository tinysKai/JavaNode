#### 定义
Elasticsearch 是一个分布式可扩展的实时搜索和分析引擎.  
Elasticsearch是一个建立在全文搜索引擎 Apache Lucene(TM) 基础上的搜索引擎

#### 类比数据库
关系数据库 ⇒ 数据库 ⇒ 表 ⇒ 行 ⇒ 列(Columns)

Elasticsearch ⇒ 索引 ⇒ 类型 ⇒ 文档 ⇒ 字段(Fields)


#### Lucene抽象架构
![Lucene](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/lucence.jpg)

#### Lucene数据模型
+ Index：索引，由很多的Document组成。
+ Document：由很多的Field组成，是Index和Search的最小单位。
+ Field：由很多的Term组成，包括Field Name和Field Value。
+ Term：由很多的字节组成，可以分词


#### Lucene的不足
+ Lucene是一个单机的搜索库，如何能以分布式形式支持海量数据?(路由)
+ Lucene中没有更新，每次都是Append一个新文档，如何做部分字段的更新？("_version"字段)
+ Lucene中没有主键索引，如何处理同一个Doc的多次写入？(es新增了一个"_id"字段)
+ 在稀疏列数据中，如何判断某些文档是否存在特定字段？("_source"字段)
+ Lucene中生成完整Segment后，该Segment就不能再被更改，此时该Segment才能被搜索，这种情况下，如何做实时搜索？

*https://zhuanlan.zhihu.com/p/34852646*



#### ES集群节点类型
+ master 是否可参与选主节点
+ data   是否存储数据  

以上两个选项共能组成4种不同的节点类型.既非master又非data的节点充当着proxy节点可负责转发请求  

![node](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/es-node.jpg)


#### ES选主
>ES集群为避免脑裂,选择多数派(quorum)方案来避免  

>ES的选举方案

    I.节点版本号越高的节点优先级更高
    II.节点版本号相同的,节点序号越小的优先级越高

>ES选主依然存在脑裂现象的分析

    多数派之所以能成立的前提是每个候选节点只能投一票,若这个前提被破坏了,则这个方案也就不成立了.
    那什么时候会出现节点投递多票呢?
    在节点投递票之后迟迟等不到应选节点的选主通知,转而发起下一轮选主,而此时恰好发现另一个优先级高的候选节点转而投递了其它节点
    当然,这种脑裂现象会很快恢复,因为脑裂后其中一个会少于多数派支持,会重新发起投票最终选主正确的

*https://zhuanlan.zhihu.com/p/35291900*


#### ES分片模型
>背景

    ES中每个Index会划分为多个Shard，Shard分布在不同的Node上，以此来实现分布式的存储和查询，支撑大规模的数据集。  
    对于每个Shard，又会有多个Shard的副本，其中一个为Primary，其余的一个或多个为Replica。  
    数据在写入时，会先写入Primary，由Primary将数据再同步给Replica。在读取时，为了提高读取能力，Primary和Replica都会接受读请求  

>数据复制流程
  
ES写入流程为先写入Primary，再并发写入Replica，最后应答客户端


#### ES特性
+ 数据高可靠：数据具有多个副本。
+ 服务高可用：Primary挂掉之后，可以从Replica中选出新的Primary提供服务。
+ 读能力扩展：Primary和Replica都可以承担读请求。
+ 故障恢复能力：Primary或Replica挂掉都会导致副本数不足，此时可以由新的Primary通过复制数据产生新的副本。

*https://zhuanlan.zhihu.com/p/35299145*


#### ES写入
>写入请求到达Shard后，先写Lucene文件，创建好索引，此时索引还在内存里面，接着去写TransLog，写完TransLog后，刷新TransLog数据到磁盘上，写磁盘成功后，请求返回给用户
![es](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/es-flush.jpg)      
```
这里有几个关键点，
一是和数据库不同，数据库是先写CommitLog，然后再写内存，而Elasticsearch是先写内存，最后才写TransLog，一种可能的原因是Lucene的内存写入会有很复杂的逻辑，很容易失败，比如分词，字段长度超过限制等，比较重，为了避免TransLog中有大量无效记录，减少recover的复杂度和提高速度，所以就把写Lucene放在了最前面。
二是写Lucene内存后，并不是可被搜索的，需要通过Refresh把内存的对象转成完整的Segment后，然后再次reopen后才能被搜索，一般这个时间设置为1秒钟，导致写入Elasticsearch的文档，最快要1秒钟才可被从搜索到，所以Elasticsearch在搜索方面是NRT（Near Real Time）近实时的系统。
三是当Elasticsearch作为NoSQL数据库时，查询方式是GetById，这种查询可以直接从TransLog中查询，这时候就成了RT（Real Time）实时系统。
四是每隔一段比较长的时间，比如30分钟后，Lucene会把内存中生成的新Segment刷新到磁盘上，刷新后索引文件已经持久化了，历史的TransLog就没用了，会清空掉旧的TransLog。
```


#### 在集群内部创建,更新,删除请求的过程
![es](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/es1.png)


#### 在集群内部查询的请求过程
![es](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/es2.png)










#### es索引的精髓
一切设计都是为了提高搜索的性能,为了提高搜索的性能，难免会牺牲比如插入/更新的速度,所以又称为`近实时`查询

#### 倒排索引
根据属性的值来查找记录

es针对倒排索引的优化
+ 根据fields建立对主键的映射关系(也就是所谓的倒排索引)
+ 针对每个filed的倒排索引的属性进行排序(这样就能快读定位到需要查询的field值,二分查找,效率为logN)
+ 为上面的filed排序再做查找索引(所以是先使用内存定位field值的大概位置,然后再到磁盘的倒排索引去查询)
+ 压缩倒排索引中主键的值(旧版本使用bitmap方式,新版本使用Roaring bitmaps模式[商,余数]方式)  

![index](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/index.png)  

![index](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/indexid.png)  

![index](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/indexid2.png)

*https://neway6655.github.io/elasticsearch/2015/09/11/elasticsearch-study-notes.html*


