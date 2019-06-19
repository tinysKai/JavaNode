# ES数据建模

## ES嵌套文档梳理

#### 特点

+ 嵌套文档是隐藏存储的,我们不能直接获取
+ 如果要增删改一个嵌套对象,我们必须把整个文档重新索引才可以
+ 查询的时候返回的是整个文档,而不是嵌套文档本身 



#### 注意

由于嵌套对象 被索引在独立隐藏的文档中，我们无法直接查询它们。 相应地，我们必须使用 [`nested` 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-nested-query.html)去获取它们 .

```js
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "eggs" 
          }
        },
        {
          "nested": {
            "path": "comments", 
            "query": {
              "bool": {
                "must": [ 
                  {
                    "match": {
                      "comments.name": "john"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 28
                    }
                  }
                ]
              }
            }
          }
        }
      ]
}}}
```



#### 嵌套的排序

```javascript
//数据
PUT /my_index/blogpost/2
{
  "title": "Investment secrets",
  "body":  "What they don't tell you ...",
  "tags":  [ "shares", "equities" ],
  "comments": [
    {
      "name":    "Mary Brown",
      "comment": "Lies, lies, lies",
      "age":     42,
      "stars":   1,
      "date":    "2014-10-18"
    },
    {
      "name":    "John Smith",
      "comment": "You're making it up!",
      "age":     28,
      "stars":   2,
      "date":    "2014-10-16"
    }
  ]
}

//查询
GET /_search
{
  "query": {
    "nested": { 
      "path": "comments",
      "filter": {
        "range": {
          "comments.date": {
            "gte": "2014-10-01",
            "lt":  "2014-11-01"
          }
        }
      }
    }
  },
  "sort": {
    "comments.stars": { 
      "order": "asc",   
      "mode":  "min",   
      "nested_path": "comments", 
      "nested_filter": {
        "range": {
          "comments.date": {
            "gte": "2014-10-01",
            "lt":  "2014-11-01"
          }
        }
      }
    }
  }
}

```



#### 嵌套对象的使用时机

**嵌套模型的缺点**

+  当对嵌套文档做增加、修改或者删除时，整个文档都要重新被索引 
+  查询结果返回的是整个文档，而不仅仅是匹配的嵌套文档 



#### 嵌套文档的使用例子

```javascript
//创建映射
PUT /payment1
{
  "mappings":{  
   "order":{  
      "properties":{  
         "id": {
                     "type": "long"
                   },
                   "userId": {
                    "type": "keyword"
                   },
         "transaction":{  
            "type":"nested",
            "properties":{  
                "orderId": {
              "type": "long"
          },
          "payType": {
              "type": "keyword"
          },
          "amount": {
                    "type": "long"
           }
            }
         }
      }
   }
}
}

//添加数据
PUT payment1/order/1
{
  "id" : 1,
  "userId":"11111",
  "transaction" : [ 
    { "orderId" : 1, "payType" :  "4","amount":111},
    { "orderId" : 1, "payType" :  "4","amount":112}
  ]
}

//获取整个文档
GET payment1/_search

//聚合查询例子
POST payment1/_search
{
  "aggs":{	//声明聚合运算
    "stat" : {
                        "nested" : {
                            "path" : "transaction" //声明内嵌对象的字段
                        },
                        "aggs" : {	//聚合求和
                         "sumAmount" : { "sum" : { "field" : "transaction.amount" } }
                        }
        }
  }
}

//多个聚合查询的例子
POST payment1/_search
{
  "aggs":{
    "transaction" : {
                        "nested" : {
                            "path" : "transaction"
                        },
                        "aggs" : {
                         "sumAmount" : { "sum" : { "field" : "transaction.amount" } },
                         "sumId":  {"sum" : { "field" : "transaction.orderId" } }
                        }
        },

     "sumId":  {"sum" : { "field" : "id" } }
               
  }
}
```



## 父子文档

#### 定义

允许将一个对象实体和另外一个对象实体关联起来 .但在父子文档中,父对象和子对象都是`完全独立`的文档。 

父-子关系的主要作用是允许把一个 type 的文档和另外一个 type 的文档关联起来，构成一对多的关系：一个父文档可以对应多个子文档 。 

Elasticsearch 维护了一个父文档和子文档的映射关系，得益于这个映射，父-子文档关联查询操作非常快。但是这个映射也对父-子文档关系有个限制条件：**父文档和其所有子文档，都必须要存储在同一个分片中**。 

尽量少地使用父子关系，仅在子文档远多于父文档时使用 



#### 对比

与嵌套对象相比,父子文档有以下优势:

+ 更新父文档时，不会重新索引子文档。
+ 创建，修改或删除子文档时，不会影响父文档或其他子文档。 
+ 子文档可以作为搜索结果独立返回。
+ 查询速度会比同等的嵌套查询慢5到10倍



#### 构建

为父文档创建索引与为普通文档创建索引没有区别。父文档并不需要知道它有哪些子文档。 

创建子文档时，用户必须要通过 `parent` 参数来指定该子文档的父文档 ID .父文档 ID 有两个作用：创建了父文档和子文档之间的关系，并且保证了父文档和子文档都在同一个分片上。 

在执行单文档的请求时需要指定父文档的 ID，单文档请求包括：通过 `GET` 请求获取一个子文档；创建、更新或删除一个子文档。

而执行搜索请求时是不需要指定父文档的ID，这是因为搜索请求是向一个索引中的所有分片发起请求，而单文档的操作是只会向存储该文档的分片发送请求。因此，如果操作单个子文档时不指定父文档的 ID，那么很有可能会把请求发送到错误的分片上。 



#### 查询

父子文档一个关键的特性是可以通过子文档的内容来查询父文档.或者通过父文档来查询子文档.

使用 `has_child` 语句可以基于子文档来查询父文档，使用 `has_parent` 语句可以基于父文档来查询子文档 

`has_child` 的查询和过滤都可以接受这两个参数：`min_children` 和 `max_children` 。 使用这两个参数时，只有当子文档数量在指定范围内时，才会返回父文档。 



```javascript
//通过子文档查询父文档
GET /company/branch/_search
{
  "query": {
    "has_child": {
      "type": "employee",
      "query": {
        "range": {
          "dob": {
            "gte": "1980-01-01"
          }
        }
      }
    }
  }
}

//has_child的过滤
GET /company/branch/_search
{
  "query": {
    "has_child": {
      "type":         "employee",
      "min_children": 2, //至少有两个雇员的分公司才会符合查询条件
      "query": {
        "match_all": {}
      }
    }
  }
}

//通过父文档查询子文档
GET /company/employee/_search
{
  "query": {
    "has_parent": {
      "type": "branch", 
      "query": {
        "match": {
          "country": "UK"
        }
      }
    }
  }
}
```



#### 父子文档聚合

???在父-子文档中支持 [子文档聚合](http://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-children-aggregation.html)，这一点和 [嵌套聚合](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-aggregation.html) 类似。但是，对于父文档的聚合查询是不支持的 .



#### 父子文档使用例子

```javascript
//构建结构体
PUT /payment
{
  "mappings": {
    "order": {
        "properties": {
                  "id": {
                     "type": "long"
                   },
                   "userId": {
                    "type": "keyword"
                   }
              }
    },
    "transaction": {
      "_parent": {
        "type": "order"
      },
      "_routing": {
          "required": true
        },
      "properties": {
          "orderId": {
              "type": "long"
          },
          "payType": {
              "type": "keyword"
          },
          "amount": {
                    "type": "long"
           }
      }
    }
  }
}

//赋值
POST /payment/order/_bulk
{ "index": { "_id": 1 }}
{ "id": 1, "userId": "123"}
{ "index": { "_id": 2 }}
{ "id": 2, "userId": "234"}

PUT /payment/transaction/1?parent=1
{
  "orderId":  "1",
  "payType":   "4",
  "amount" : "11" 
}

PUT /payment/transaction/3?parent=1
{
  "orderId":  "1",
  "payType":   "4",
  "amount" : "13" 
}

PUT /payment/transaction/2?parent=2
{
  "orderId":  "2",
  "payType":   "5",
  "amount" : "12" 
}

//聚合查询
POST /payment/transaction/_search
{
   "query" : {
        "match": {"payType": "4"}
    },
    "aggs" : {
        "sumAmount" : { "sum" : { "field" : "amount" } }
    }
}


//构建结构体
PUT /merchant
{
  "mappings": {
    "custom": {
        "properties": {
                  "id": {
                     "type": "keyword"
                   },
                   "status": {
                    "type": "keyword"
                   }
              }
    },

    "user": {
      "_parent": {
        "type": "custom" 
      },
      "_routing": {
          "required": true
        },
      "properties": {
          "customerId": {
              "type": "keyword"
          },
          "userName": {
              "type": "keyword"
          },
          "userStatus": {
                    "type": "keyword"
           },
          "id": {
              "type": "keyword"
          }

      }
    }
  }
}

//父子文档查询
GET /merchant/user/_search
{
  "query": {
    "bool" :{
      "must":[
        {
          "has_parent": {
            "type": "custom", 
            "query": {
              "match": {
                "status": "4"
                        }
                    }
                      }
        }
        
        ]
    }
    
  }
}

//父子文档查询
GET /merchant/user/_search
{
  "query": {
    "bool" :{
      "must":[
        {
          "has_parent": {
            "type": "custom", 
            "query": {
              "match": {
                "status": "0"
                        }
                    }
                      }
        },{
          "match": {
            "userStatus": "0"
          }
        }
        ]
    }
    
  }
}
```





