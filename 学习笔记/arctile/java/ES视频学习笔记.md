# ES视频学习笔记

## 初始ES

### URI Search

```
# 一个比较全面的查询例子
GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
{
           profile : true
}

# 所有字段查询某个值
GET /movies/_search?q=2012

# 针对指定字段查询值
GET /movies/_search?q=title:2012

# 特定两个组合按序出现
GET /movies/_search?q=title:"Beautiful Mind"

# 需出现以下两个单词 不一定需要连在一起 但需都有
GET /movies/_search?q=title:(Beautiful Mind)

# AND关联
GET /movies/_search?q=title:(Beautiful AND Mind)

# NOT 有Beautiful木有Mind的
GET /movies/_search?q=title:(Beautiful AND Mind)

# 范围查询
GET /movies/_search?q=year:>=1980
GET /movies/_search?q=year:{2017 TO 2018]
GET /movies/_search?q=year:[* TO 2018]
GET /movies/_search?q=year:(>2010 && <=2018)
GET /movies/_search?q=year:(+>2010  +<=2018)


# 通配符查询
GET /movies/_search?q=title:bea*

# 正则匹配
GET /movies/_search?q=title:[bt]oy

#模糊匹配 ,可支持单词输错时的匹配,如下的`beautifl`可以搜出`Beautiful`
GET /movies/_search?q=title:beautifl~1
```

> q指定查询语句，使用Query String Syntax
>
> df默认字段，不指定时
>
> Sort排序/from和size用于分页
>
> Profile可以查看查询是如何被执行的

### DSL

```
GET /${INDEX}/_search
{
	"sort" : [{"${field}":"desc"}]
	"from" : 0 ,
	"size" : 10 , 
	"query" : {
	   "match_all" :{}
	}
}

# OR关系
GET /${INDEX}/_search
{
	"query" : {
	   "match" :{
		"${field}" :  "${val1} ${val2}"	   
	   }
}


# 针对多个单词查询若需在match中是都有的情况需使用AND查询,否则就是OR查询
GET /${INDEX}/_search
{
	"query" : {
	   "match" :{
	   	  "${field}":{
	   	  	  "query" : "${val1} ${val2}",
	   	  	  "operator" : "AND"
	   	  }
	   }
	}
}

```

### Mapping

```
# 定义索引mapping
PUT users
{
    "mappings" : {
      "properties" : {
        "firstName" : {
          "type" : "text"
        },
        "lastName" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "text",
          "index": false  # 支持该字段不进行索引无法被搜索,意味着不会对该字段进行倒排索引,能有效减少储存空间
        }
      }
    }
}

# 设置NULL值 针对一些无值情况下的查询情况(只有keyword类型支持)
PUT users
{
    "mappings" : {
      "properties" : {
        "firstName" : {
          "type" : "text"
        },
        "lastName" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "keyword",
          "null_value": "NULL"
        }

      }
    }
}

PUT users/_doc/1
{
  "firstName":"Ruan",
  "lastName": "Yiming",
  "mobile": null
}


PUT users/_doc/2
{
  "firstName":"Ruan2",
  "lastName": "Yiming2"
}

GET users/_search
{
  "query": {
    "match": {
      "mobile":"NULL"
    }
  }
}



```

### Index Template

#### 定义

`Index Templates`帮助你设定Mappings和Settings,并按照一定的规则,自动匹配到新创建的索引之上

> 模版仅在一个索引被新创建时，才会产生作用。修改模版不会影响已创建的索引

```
#Create a default template
PUT _template/template_default
{
  "index_patterns": ["*"],   # 适配所有新创建的索引
  "order" : 0,			    # 优先级 
  "version": 1,
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas":1
  }
}

# 针对特定类型的索引进行索引配置
PUT /_template/template_test
{
    "index_patterns" : ["test*"], # 适配test开头的索引
    "order" : 1,
    "settings" : {
    	"number_of_shards": 1,
        "number_of_replicas" : 2
    },
    "mappings" : {				# 索引mapping的一些默认设置
    	"date_detection": false,	# 不开启日期推测
    	"numeric_detection": true   # 开启数字类型推测
    }
}
```

### 聚合入门

![https://imgchr.com/i/Gwq0Z4](http://ww1.sinaimg.cn/large/8bb38904gy1gdhvf8d5g0j217n0p4dl1.jpg)

![https://imgchr.com/i/GwqsiR](http://ww1.sinaimg.cn/large/8bb38904gy1gdhvh7mtvvj218r0p77cf.jpg)

## 深入ES

### Term查询

在 ES 中，Term 查询，对输入不做分词。会将输入作为一个整体，在倒排索引中查找准确的词项，并且使用相关度算分公式为每个包含该词项的文档进行相关度算分.

> 即便是对 Keyword 进行 Term 查询，同样会进行算分.可以将查询转为 Filtering，取消相关性算分的环节，以提升性能

```
POST /products/_bulk
{ "index": { "_id": 1 }}
{ "productID" : "XHDK-A-1293-#fJ3","desc":"iPhone" }
{ "index": { "_id": 2 }}
{ "productID" : "KDKE-B-9947-#kL5","desc":"iPad" }


# 由于分词器中存在大小写转换,使用原来的大写`iPhone`会匹配不到值,需使用`iphone来查找`
POST /products/_search
{
  "query": {
    "term": {
      "desc": {
        //"value": "iPhone"
        "value":"iphone"
      }
    }
  }
}

# 可使用`${field}.keyword`来做完全匹配查询
POST /products/_search
{
  "query": {
    "term": {
      "productID.keyword": {
        "value": "XHDK-A-1293-#fJ3"
      }
    }
  }
}

# 使用filter过滤,不进行算分,减少算分的开销
POST /products/_search
{
  "explain": true,
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "productID.keyword": "XHDK-A-1293-#fJ3"
        }
      }

    }
  }
}
```

### 全文查询

#### 方式

+ Match Query 
+  Match Phrase Query 
+  Query String Query

#### 过程

​	查询时候，先会对输入的查询进行分词，然后每个词项逐个进行底层的查询，最终将结果进行合
并。并为每个文档生成一个算分。- 例如查 “Matrix reloaded”，会查到包括 Matrix 或者 reload
的所有结果。

#### 例子

```
GET /${INDEX}/_search
{
	"query" : {
	  
      "match" :{
		"${field}" :  "${val1} ${val2}"	   
	   }
}
```

#### 结构化查询

结构化搜索（Structured search） 是指对结构化数据的搜索

> 日期，布尔类型和数字都是结构化的,文本也可以是结构化的。

```
# 日期 range范围查询
# 其它常见变量 年-y,月-M,周-w,日-d,时-H/h,分-m,秒-s
POST products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "date" : {
                      "gte" : "now-1y"  # 支持当前时间减一年的操作
                    }
                }
            }
        }
    }
}

#exists查询
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "exists": {
          "field": "#{field}"
        }
      }
    }
  }
}


#exists 反向查询
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "bool" :{
          "must_not":{
            "exists": {
              "field": "#{field}"
            }
          }  
        }
      }
    }
  }
}

#处理多值字段，term 查询是包含，而不是等于
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "terms": {
          "productID.keyword": [
            "QQPX-R-3956-#aD8",
            "JODL-X-1937-#pV7"
          ]
        }
      }
    }
  }
}
```

### Query & Filter

#### bool查询

| must     | 必须匹配,贡献得分                    |
| -------- | ------------------------------------ |
| should   | 选择匹配,贡献得分                    |
| must_not | Filter Context,必须不能匹配          |
| filter   | Filter Context,必须匹配,但不贡献得分 |

```
#基本语法
POST /products/_search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "price" : "30" }
      },
      "filter": {
        "term" : { "avaliable" : "true" }
      },
      "must_not" : {
        "range" : {
          "price" : { "lte" : 10 }
        }
      },
      "should" : [
        { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
        { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
      ],
      "minimum_should_match" :1
    }
  }
}

#Filtering Context 不计算得分
POST _search
{
  "query": {
    "bool" : {
      "filter": {
        "term" : { "avaliable" : "true" }
      },
      "must_not" : {
        "range" : {
          "price" : { "lte" : 10 }
        }
      }
    }
  }
}

#同一层级下的竞争字段，具有有相同的权重,通过嵌套 bool 查询，可以改变对算分的影响
# 如下四个term相同的计算得分影响
POST /animals/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "brown" }},
        { "term": { "text": "red" }},
        { "term": { "text": "quick"   }},
        { "term": { "text": "dog"   }}
      ]
    }
  }
}

# 我们将颜色分类的这一项使用bool的符合查询来改变对计分的影响
POST /animals/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "quick" }},
        { "term": { "text": "dog"   }},
        {
          "bool":{
            "should":[
               { "term": { "text": "brown" }},
                 { "term": { "text": "brown" }},
            ]
          }

        }
      ]
    }
  }
}
```

Boosting 是控制相关度的一种手段参数 boost的含义

+ 当 boost > 1 时，打分的相关度相对性提升

+ 当 0 < boost < 1 时，打分的权重相对性降低

+ 当 boost < 0 时，贡献负分

```
POST blogs/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {
          "title": {
            "query": "apple,ipad",
            "boost": 4
          }
        }},

        {"match": {
          "content": {
            "query": "apple,ipad",
            "boost":  1
          }
        }}
      ]
    }
  }
}
```

使用`Boosting Query`来影响计分方式

```
POST news/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "content": "apple"
        }
      },
      "negative": {
        "match": {
          "content": "pie"
        }
      },
      "negative_boost": 0.5
    }
  }
}

```

#### 单字符串多字段查询

三种场景

+ 最佳字段(默认)
+ 多数字段
+ 混合字段

```
POST blogs/_search
{
  "query": {
    "multi_match": {
      "type": "best_fields",
      "query": "Quick pets",
      "fields": ["title","body"],
      "tie_breaker": 0.2,
      "minimum_should_match": "20%"
    }
  }
}

```

![https://imgchr.com/i/GBhsRU](http://ww1.sinaimg.cn/large/8bb38904gy1gditt4wa84j20rk0mu77u.jpg)

#### 搜索脚本

```
# 先定义一个搜索脚本,`{{q}}`表示一个变量入参
POST _scripts/tmdb
{
  "script": {
    "lang": "mustache",
    "source": {
      "_source": [
        "title","overview"
      ],
      "size": 20,
      "query": {
        "multi_match": {
          "query": "{{q}}",
          "fields": ["title","overview"]
        }
      }
    }
  }
}


DELETE _scripts/tmdb

GET _scripts/tmdb

# 使用脚本进行搜索,其实类似于定义了一个方法
POST tmdb/_search/template
{
    "id":"tmdb",
    "params": {
        "q": "basketball with cartoon aliens"
    }
}
```

#### 别名定义

```
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-latest"
      }
    }
  ]
}

# 可直接进行针对数据进行过滤后再进行起别名
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-lastest-highrate",
        "filter": {
          "range": {
            "rating": {
              "gte": 4
            }
          }
        }
      }
    }
  ]
}

# 使用上述定义的别名直接全部查询就是已经经过过滤的数据了
POST movies-lastest-highrate/_search
{
  "query": {
    "match_all": {}
  }
}
```

###  分布式搜索

#### 文档操作

更新操作

![https://imgchr.com/i/GD65yn](http://ww1.sinaimg.cn/large/8bb38904gy1gdj269r0y0j21au0l3403.jpg)

删除操作

![https://imgchr.com/i/GD6ILq](http://ww1.sinaimg.cn/large/8bb38904gy1gdj26so956j215n0mswfz.jpg)

#### 分片及生命周期

ES 中最小的工作单元是一个 Lucene 的 Index

Segment的不可变性

![https://imgchr.com/i/GDOMWj](http://ww1.sinaimg.cn/large/8bb38904gy1gdj4zyyhvej20oc0dbq36.jpg)

记录的储存转换过程

![https://imgchr.com/i/GDO0p9](http://ww1.sinaimg.cn/large/8bb38904gy1gdj50zxnrbj20lh0afwf5.jpg)

#### 排序

Elasticsearch 默认采用相关性算分对结果进行降序排序

```
# 多字段排序
POST /${Index}/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}},
    {"_doc":{"order": "asc"}},
    {"_score":{ "order": "desc"}}
  ]
}

#打开 text的 fielddata
PUT ${index}/_mapping
{
  "properties": {
    "${field}" : {
          "type" : "text",
          "fielddata": true,
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
  }
}


# 针对明确不需要排序和聚合分析的字段可关闭 keyword的 doc values
PUT ${index}/_mapping
{
  "properties": {
    "${field}":{
      "type": "keyword",
      "doc_values":false
    }
  }
}
```

排序是针对字段`原始内容`进行的。 倒排索引无法发挥作用,需要用到正排索引。通过文档 Id 和字段快速得到字段原始内容

Elasticsearch 有两种实现方法

+ fieldData
+ doc values(列式储存,对text无效)

![https://imgchr.com/i/Gsma0f](http://ww1.sinaimg.cn/large/8bb38904gy1gdju9lgw6wj21br0nydi1.jpg)

#### 分页

##### from/size

常见的es分页机制`from`,`size`来控制分页数据,ES默认限制(from + size之和不超过10000)

> 由于es本身的分布式机制,当进行`深度分页`时(如from-1000,size-10),会在每个分片上查询大量的数据(此时为1010条数据),之后再由协调节点进行排序后返回最终的10条,性能不理想

##### search after

`search after`机制使用基于上一次查询结果返回的数据来进行对下一次数据的查询,类似游标查询,可避免`深度分页`造成的内存问题,但有以下几个限制 :

+ 遍历单向性,只能往后遍历
+ 需指定sort并保证值唯一(可再添加`_id`来保证唯一)
+ 除了第一次的查询.之后的每一个查询都基于上一次查询返回的`sort`返回唯一值进行下一次查询
+ 每次查询的记录条数是固定的(基于你设置的size值)

```
# 第一次查询
POST users/_search
{
    "size": 1,
    "query": {
        "match_all": {}
    },
    "sort": [
        {"age": "desc"} ,     # 使用age来进行降序排序
        {"_id": "asc"}        # 同时基于id来保证此排序结果唯一
    ] 
}

# 之后的查询
POST users/_search
{
    "size": 1,
    "query": {
        "match_all": {}
    },
    "search_after":   			# 使用上文查询返回的sort返回值进行下一次查询
        [
          10,
          "ZQ0vYGsBrR8X3IP75QqX"
         ],
    "sort": [
        {"age": "desc"} ,
        {"_id": "asc"}    
    ]
}
```

##### Scrol API

`Scrol API`基于一个创建的快照来对数据进行滚动查询,适用于查询全部数据时使用,有以下特点:

+ 在快照生成之后的数据无法查询到(查询的数据就是生成快照那一刻的数据)
+ 每一次查询都依赖上一次查询返回的`scroll_id`

```
POST /${index}/_search?scroll=5m   # 定义此快照的有效期为5分钟
{
    "size": 1,
    "query": {
        "match_all" : {
        }
    }
}

POST /_search/scroll
{
    "scroll" : "1m",		# 定义此次快照的有效期为1分钟
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAWAWbWdoQXR2d3ZUd2kzSThwVTh4bVE0QQ=="
}

```

#### 并发控制

ES使用乐观锁机制来解决并发问题

ES 中的文档是不可变更的。如果你更新一个文档，会将旧文档标记为删除，同时增加一个全新的文档。同时文档
的 version 字段加 1

ES有两种版本控制, 

+ 内部版本(旧版本使用version,新版本使用`If_seq_no + If_primary_term`)
+ 外部版本(version + version_type=external)

```
# 内部版本控制,使用基于查询返回的`if_seq_no=1&if_primary_term=1`来进行并发控制
PUT products/_doc/1?if_seq_no=1&if_primary_term=1
{
  "title":"iphone",
  "count":100
}


# 外部版本控制
PUT products/_doc/1?version=30000&version_type=external
{
  "title":"iphone",
  "count":100
}
```

### 聚合分析

##### 聚合

```
# 同时查询多个聚合结果
POST employees/_search
{
  "size": 0,
  "aggs": {
    "max_salary": {
      "max": {
        "field": "salary"
      }
    },
    "min_salary": {
      "min": {
        "field": "salary"
      }
    },
    "avg_salary": {
      "avg": {
        "field": "salary"
      }
    }
  }
}

# 通过一个字段查询多个聚合值
POST employees/_search
{
  "size": 0,
  "aggs": {
    "stats_salary": {
      "stats": {
        "field":"salary"
      }
    }
  }
}

```

##### 分桶

注意针对text分桶时会进行分词,而对keyword进行分桶并不会进行分词

```
# 对keword 进行聚合
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"
      }
    }
  }
}



# 对 Text 字段打开 fielddata，支持terms aggregation
PUT employees/_mapping
{
  "properties" : {
    "job":{
       "type":     "text",
       "fielddata": true
    }
  }
}


# 对 Text 字段进行 terms 分词。分词后的terms
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job"
      }
    }
  }
}


# 对job.keyword 进行 terms 聚合,`cardinality`会进行去重
POST employees/_search
{
  "size": 0,
  "aggs": {
    "cardinate": {
      "cardinality": {
        "field": "job.keyword"
      }
    }
  }
}

# 对 性别的 keyword 进行聚合
POST employees/_search
{
  "size": 0,
  "aggs": {
    "gender": {
      "terms": {
        "field":"gender"
      }
    }
  }
}
```

##### 复杂聚合查询

```
# 指定size，不同工种中，年纪最大的3个员工的具体信息
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"
      },
      "aggs":{				# 针对查询出来的每个bucket再进行聚合操作,此处为排序筛选
        "old_employee":{
          "top_hits":{
            "size":3,
            "sort":[
              {
                "age":{
                  "order":"desc"
                }
              }
            ]
          }
        }
      }
    }
  }
}

```

##### Range分桶

```
#Salary Ranges 分桶，可以自己定义 key
POST employees/_search
{
  "size": 0,
  "aggs": {
    "salary_range": {
      "range": {
        "field":"salary",
        "ranges":[
          {
            "to":10000
          },
          {
            "from":10000,
            "to":20000
          },
          {
            "key":">20000",
            "from":20000
          }
        ]
      }
    }
  }
}
```

##### 直方图分桶

```
#Salary Histogram,工资0到10万，以 5000一个区间进行分桶
POST employees/_search
{
  "size": 0,
  "aggs": {
    "salary_histrogram": {
      "histogram": {
        "field":"salary",
        "interval":5000,	# 5000一个区间
        "extended_bounds":{
          "min":0,
          "max":100000

        }
      }
    }
  }
}
```

#### Pipeline

支持对聚合分析的结果，再次进行聚合分析.Pipeline 的分析结果会输出到原结果中，根据位置的不同，分为两类 :

+ Sibling - 结果和现有分析结果同级
  + Max，min，Avg & Sum Bucket
  + Stats，Extended Status Bucket
  + Percentiles Bucket
+ Parent - 结果内嵌到现有的聚合分析结果之中
  +    Derivative （求导）
  +    Cumultive Sum （累计求和）
  +    Moving Function (滑动窗口)
    

```
# 平均工资最低的工作类型
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field": "job.keyword",
        "size": 10
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        }
      }
    },
    "min_salary_by_job":{
      "min_bucket": {
        "buckets_path": "jobs>avg_salary"
      }
    }
  }
}


#按照年龄对平均工资求导
POST employees/_search
{
  "size": 0,
  "aggs": {
    "age": {
      "histogram": {
        "field": "age",
        "min_doc_count": 1,
        "interval": 1
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        },
        "derivative_avg_salary":{
          "derivative": {
            "buckets_path": "avg_salary"
          }
        }
      }
    }
  }
}

```

#### 聚合的作用范围

ES 聚合分析的默认作用范围是 query 的查询结果集,同时 ES 还支持以下方式改变聚合的作用范围

+ filter -- Filter 只对当前的子聚合语句生效
+ post_filter  -- 对聚合分析后的文档进行再次过滤
+ global

```
# Query 默认是基于query的结果集的
POST employees/_search
{
  "size": 0,
  "query": {
    "range": {
      "age": {
        "gte": 20
      }
    }
  },
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword",
        "order":[			# 排序
          {"_count":"asc"},
          {"_key":"desc"}
          ]
      }
    }
  }
}


#Filter
POST employees/_search
{
  "size": 0,
  "aggs": {
    "older_person": {
      "filter":{		# Filter 只对当前的子聚合语句生效
        "range":{
          "age":{
            "from":35
          }
        }
      },
      "aggs":{
         "jobs":{
           "terms": {
       		 "field":"job.keyword"
     		 }
      }
    }},
    "all_jobs": {	# all_jobs 还是基于query的作用范围
      "terms": {
        "field":"job.keyword"
      }
    }
  }
}


#Post field. 一条语句，找出所有的job类型。还能找到聚合后符合条件的结果
POST employees/_search   # 无需使用size
{
  "aggs": {
    "jobs": {
      "terms": {
        "field": "job.keyword"
      }
    }
  },
  "post_filter": {
    "match": {
      "job.keyword": "Dev Manager"
    }
  }
}


#global
POST employees/_search
{
  "size": 0,
  "query": {
    "range": {
      "age": {
        "gte": 40
      }
    }
  },
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"
        
      }
    },
    
    "all":{
      "global":{},	# 无视query的查询条件,对全部文档进行统计
      "aggs":{
        "salary_avg":{
          "avg":{
            "field":"salary"
          }
        }
      }
    }
  }
}
```



