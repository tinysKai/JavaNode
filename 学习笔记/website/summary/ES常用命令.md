## ES常用命令

#### 全文检索&短语搜索

`match`是全文检索,因此如果查询条件如`"name":"tinys kai"`,则会分词出`tinys`,`kai`来查询.

若想只查询短语`tinys kai`,则可使用`match_phrase`来查询.

#### 集群健康检查

```shell
#查看集群状态
GET /_cat/health?v

# 返回结果
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1566637870 02:11:10  elasticsearch yellow          1         1      1   1    0    0        1             0                  -                 50.0%

#status
 #- green：每个索引的 primary shard 和 replica shard 都是 active 状态的
 #- yellow：每个索引的 primary shard 都是 active 状态的，但是部分 replica shard 不是 active 状态，处于不可用的状态
 #- red：不是所有索引的 primary shard 都是 active 状态的，部分索引有数据丢失了


# unassign：未分配数量
# active_shards_percent：可用 shards 百分比





# 查看集群节点状态
GET /_cat/nodes?v
ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
127.0.0.1            4          90  20    0.94    0.72     0.30 mdi       *      b1uYKac

#master列下带`*`的符号表示为主




# 查看分片状态
GET /_cat/shards?v
index   shard prirep state      docs store ip        node
person  1     p      STARTED       0  191b 127.0.0.1 b1uYKac
person  1     r      UNASSIGNED                                         
person  2     p      STARTED       1 3.7kb 127.0.0.1 b1uYKac
person  2     r      UNASSIGNED                                 
person  0     p      STARTED       0  191b 127.0.0.1 b1uYKac
person  0     r      UNASSIGNED        
```



#### es全局查询

```shell
# 查看当前es存在哪些索引
GET /_cat/indices?v

# 返回结果
health status index   uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   .kibana ys5virQxRU6y8VRKuyymgA   1   1          1            0      3.2kb          3.2kb

# 查询索引结构
GET ${index}/_mapping

# 删除索引
DELETE /${index}?pretty
```

#### 常见查询

```json
//查询语句
GET /${index}/${type}/_search
{
  "query": {
    "match_all": {}
  }
}

//分页查询
GET /${index}/${type}/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,  //0表示第一条记录开始
  "size": 20  //查询20条
}

//筛选返回字段
GET /${index}/${type}/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["${field1}","${field2}"]  
}

//查询并过滤后排序
GET /${index}/${type}/_search
{
  "query": {
    "match_all": {}
  },
  "sort":[
    {
      "${field}": "desc"
    }
  ]
}


//不分词的短语搜索,如下查询则只会查询"tinys kai"的匹配项,而不会单独去查询"tinys"或者"kai"
GET /${index}/${type}/_search
{
  "query": {
    "match_phrase": {
      "${field}" : "tinys kai"
    }
  }
}  
```

单filter查询

单独使用filter查询需配合使用`constant_score`（恒定分数，所有分数都是 1）来实现

```
GET /${index}/${type}/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "${field}": {
            "gte": ${val}
          }
        }
      }
    }
  }
}
```

#### 不分词查询

```json
//搜索文本不分词查询
GET /${index}/${type}/_search
{
  "query": {
    "term": {
      "${field}": "${val}"
    }
  }
}

//多词不分词搜索
GET /_search
{
    "query": {
        "terms": { 
            "${filed}": [ "${val1}", "${val2}", "${val3}" ] 
        }
    }
}
```

#### 聚合查询

```json
//聚合函数语法
GET /${index}/${type}/_search
{
    "aggs": {
        "NAME": {
            "AGG_TYPE": {}
        }
    }
}

//- aggs：聚合函数,固定值
//- NAME：给这个操作取一个名字
//- AGG_TYPE：聚合类型

//聚合查询
GET /${index}/${type}/_search
{
    "aggs": {
        "${agg_name}": {
            "terms": {
                "field": "${field}"
            }
        }
    },
     "size": 0 //加上返回的size为0可只返回聚合数据
}

//如果聚合的字段为text,则需设置`"fielddata": true`
PUT /${index}/${type}/_searchg
{
  "properties": {
    "${field}":{
      "type": "text",
      "fielddata": true
    }
  }
}

//查询过滤后判重
GET /${index}/${type}/_search
{
  "query": {
    "match_all": {}
  },
   "aggs": {
        "${agg_name}": {
            "terms": {
                "field": "${field}"
            }
        }
    },
     "size": 0 //加上返回的size为0可只返回聚合数据
}

```

#### 更新语句

```json
//全部更新
PUT /${index}/${type}/id?pretty
{
	"key": value
}

//部分更新
POST /${index}/${type}/id/_update
{
   "doc" : {
      "${key1}" : val1,
      "${key2}" : val2 
   }
}

//删除索引下某个id的记录
DELETE /${index}/${type}/id?pretty
```

#### 复合布尔查询

bool 中可以放：must，must_not，should，filter

```json
//and查询例子
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "tinys" } },
        { "match": { "age": 20 } }
      ]
    }
  }
}

//or查询例子
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name": "tinys" } },
        { "match": { "age": 210 } }
      ]
    }
  }
}

//neither not都不符合的例子
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "name": "tinys" } },
        { "match": { "age": 21 } }
      ]
    }
  }
}


//多条件布尔查询: 筛选出名字为tinys但年龄不等于21的记录
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "tinys" } }
      ],
      "must_not": [
        { "match": { "age": 21 } }
      ]
    }
  }
}

//范围查询
GET /${index}/${type}/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
            "age": {
              "gte": 20,
              "lte": 30
                   }
                }
               }
          }
  }
}
```

#### 别名操作

```json
添加别名
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "${index}",
        "alias": "${index}_aliases"
      }
    }
  ]
}


删除别名
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "${index}",
        "alias": "${index}_aliases"
      }
    }
  ]
}
```

#### Mapping映射

`只能创建 index 时手动建立 mapping，或者新增 field mapping，但是不能 update field mapping`

```json
//给新创建的索引创建映射
PUT /${index}
{
  "${index}": {
    "${type}":{
      "properties": {
        "field":{
          "type": "text", // 数据类型
          "index":"",    // 索引类型
          "analyzer": "english" // 分词类型
        }
      }
    }
  }
}

//索引类型有如下值：
//- analyzed : 全文 full text
//- not_analyzed : 精准匹配 exact value
//- no ：不索引


//给已创建完的索引修改Mapping
PUT /${index}/${type}/_mapping
{
  "properties" : {
    "${new_field}" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}
```

#### 创建索引

```json
PUT /${index}
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "${type}": {
      "properties": {
        "${field}": {
          "type": "text"
        }
      }
    }
  }
}
```

