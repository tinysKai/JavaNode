## Elasticsearch学习笔记

#### 定义

Elasticsearch 是一个分布式可扩展的实时搜索和分析引擎.

#### 启动es

```shell
#下载
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.3.tar.gz

# 2.解压
tar -xvf elasticsearch-5.6.3.tar.gz

# 3.启动
cd elasticsearch-5.6.3/bin
./elasticsearch

# 指定名字启动 :
./elasticsearch -Ecluster.name=my_cluster_name -Enode.name=my_node_name
```

```shell
#修改kibana/config下的配合文件kibana.yml
server.host: "192.168.198.128" #修改为外网可访问的地址
elasticsearch.url: "http://localhost:9200" #修改为es的地址

#进入bin文件夹启动 :
./kibana

#浏览器访问　：　http://192.168.198.128:5601/app/kibana#/dev_tools/console?_g=()
#注意关闭防火墙 `service iptables stop`
```



#### lucence

lucene，最先进、功能最强大的搜索库；直接基于 lucene 开发，非常复杂，api 复杂，需要深入理解原理（各种索引结构）.Elasticsearch 基于 lucene，隐藏复杂性，提供简单易用的 restful api 接口,是一个分布式的搜索引擎和分析引擎.

#### 倒排索引

倒排索引由在文档中出现的唯一的单词列表，以及对于每个单词在文档中的位置组成。

#### 全文检索



#### 基本元素

+ NRT(近实时,Near Realtime)

+ 集群

  + 分片(shard) - 每个 shard可创建多个 replica副本
  + 副本(replica) - replica可以在 shard 故障时提供备用服务，保证数据不丢失，多个replica还可以提升搜索操作的吞吐量和性能.默认的副本数为1

+ 数据结构

  + 索引
  + 类型
  + 文档
  + field

#### 数据类型

- string
- byte，short，integer，long
- float，double
- boolean
- date


#### 乐观锁

 es的记录中存在着一个`_version`,用来声明当前记录的版本号.创建时初始值为0,之后每次更新成功后都会在操作后将值加一.所以如果要避免记录在更新时相互覆盖的场景,则可在更新记录时先到es获取对应记录得到版本号,然后更新时带上这个版本号来实现乐观锁更新



#### Bulk批量操作

Bulk 不是原子性的，不能用它来实现事务控制。每个请求是单独处理的，因此一个请求的成功或失败不会影响其他的请求。

Bulk Request 会加载到内存里，如果太大的话，性能反而会下降，因此需要反复尝试一个最佳的 bulk size。一般从 1000~5000 条数据开始，尝试逐渐增加。另外，如果看大小的话，最好是在 5~15MB 之间。

**操作类型**

+ index - 普通的put操作，可以是创建文档，也可以是全量替换文档
+ create - `PUT /${index}/${type}/${id}/_create`，强制创建/存在则报错
+ update 
+ delete

数据格式

采用这种一行定位信息,一行doc记录的行格式而不使用json格式的原因是es的数据是分片的,一行行读取后可将记录直接就转发到相应的分片上,而使用json需解析并整个数据需耗费更多的内存,更多的gc开销

```
{"action": {"meta"}}\n
{"data"}\n
{"action": {"meta"}}\n
{"data"}\n
```





#### 路由算法

- shard = hash(routing) % number_of_primary_shards
- routing = `_id` or custom routing value

主分片默认为5个,一旦创建了就不能修改就是因为document记录是基于主分片数来hash取模的.

当然es支持手动指定路由键,比如说 `put /${index}/${type}/${id}?routing=${user_id}|`,此时可将同一个用户的数据都路由到同一个分片中,能提升批量读取时的性能



#### 分片之间的数据同步

TODO

#### 查询过程解析

1. 客户端请求到es集群的某个节点A,此时节点A作为协调节点
2. A根据路由键计算记录是分布在哪一个shard上,我们假设在分片`one`上
3. A将在分片`one`的主分片以及副本分片上随机选一个来执行这次查询

特殊情况,当查询的记录正在主分片上创建或更新时,记录还没同步到副本分片中,此时查询的结果可能是脏数据或者是查不到数据.解决方法是敏感数据只在主分片执行查询.

#### filter与query对比

- filter：仅仅只是按照搜索条件过滤出需要的数据而已，不计算任何相关度分数，对相关度没有任何影响
- query： 会去计算每个 document 相对于搜索条件的相关度，并按照相关度进行排序

一般来说，如果你是在进行搜索，需要将最匹配搜索条件的数据先返回，那么用 query；如果你只是要根据一些条件筛选出一部分数据，不关注其排序，那么用 filter；

```json
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "${field}": "${val}"
          }
        }
      ],
      "filter": {
        "range": {
          "${filed2}": {
            "gte": ${val2} //gte表示大于等于,也可使用如gt,lte等
          }
        }
      }
    }
  }
}
```

#### 写操作解析

es写入流程是写将数据写入lucene,数据载入内存后,再将数据写入translog刷新到磁盘,写磁盘成功后，请求返回给用户。

![img](https://s2.ax1x.com/2019/08/27/moVoMd.jpg)

#### 相关度得分算法

Elasticsearch 使用的是 term frequency/inverse document frequency 算法，简称为 **TF/IDF** 算法.

影响因素

+ Term frequency（不同词的频率） - 出现次数越多，就越相关
+ Inverse document frequency(相同词的频率) -  出现的次数越多，就越不相关
+ Field-length norm - field越长,相关度越弱

#### 深度查询分页常用做法

+ from/size
+ search-after
+ srcoll 

游标查询会取某个时间点的快照数据。 查询初始化之后索引上的任何变化会被它忽略。 它通过保存旧的数据文件来实现这个特性，结果就像保留初始化时的索引 '视图' 一样。

深度分页的代价根源是结果集全局排序，如果去掉全局排序的特性的话查询结果的成本就会很低。 游标查询用字段 `_doc`来排序。 这个指令让 Elasticsearch 仅仅从还有结果的分片返回下一批结果。

启用游标查询可以通过在查询的时候设置参数 `scroll` 的值为我们期望的游标查询的过期时间。 游标查询的过期时间会在每次做查询的时候刷新，所以这个时间只需要足够处理当前批的结果就可以了，而不是处理查询结果的所有文档的所需时间。 这个过期时间的参数很重要，因为保持这个游标查询窗口需要消耗资源，所以我们期望如果不再需要维护这种资源就该早点儿释放掉。 设置这个超时能够让 Elasticsearch 在稍后空闲的时候自动释放这部分资源。

```json
GET /old_index/_search?scroll=1m  //保持游标查询窗口一分钟
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], //关键字 _doc 是最有效的排序顺序
    "size":  1000
}

//这个查询的返回结果包括一个字段 _scroll_id， 它是一个base64编码的长字符串 。
//现在我们能传递字段 _scroll_id 到 _search/scroll 查询接口获取下一批结果：
GET /_search/scroll
{
    "scroll": "1m", //注意再次设置游标查询过期时间为一分钟。
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7...."
}

//注意游标查询每次返回一个新字段 _scroll_id。每次我们做下一次游标查询， 我们必须把前一次查询返回的字段 _scroll_id 传递进去。 
//当没有更多的结果返回的时候，我们就处理完所有匹配的文档了。
```

