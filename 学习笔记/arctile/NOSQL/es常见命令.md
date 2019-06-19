#### DDL
>查询所有的索引  

 	GET /_cat/indices?v
 	
>创建索引

 	PUT /index?pretty

 	
>查询索引结构
    	
    	GET index_aliases/_mapping
    	
>删除整个索引
    	
    	DELETE /index?pretty
    	 	
#### 别名操作
>添加别名

    POST /_aliases
    {
      "actions": [
        {
          "add": {
            "index": "work_order_header",
            "alias": "work_order_header_aliases"
          }
        }
      ]
    }


>删除别名

    POST /_aliases
    {
      "actions": [
        {
          "remove": {
            "index": "work_order_header",
            "alias": "work_order_header_aliases"
          }
        }
      ]
    }    
        	 	
    	 	
#### DML    	 	

>索引赋值/更新索引值

    PUT /index/type/id?pretty
	{
	  "key": value
	}
 	
>文档部分更新

 	POST /index/type/id/_update
    {
       "doc" : {
          "key1" : val1,
          "key2" : val2 
       }
    }
    

>分页

    GET /index/_search
    	{
    	  "query": { "match_all": {} },
    	  "from": 0,
    	  "size": 10
    	}
    	
>查询get

    	GET /index/type/id?pretty
    	

    	
>更新某索引具体id的某个值
    	
    	POST /index/type/id/_update?pretty
        	{
        	  "doc": { "key1": value1, "key2": value2,....}
        	}
        	
        	
>删除索引的某个id的值

     DELETE /customer/type/id?pretty
        	

>只获取部分属性

    GET /index/_search
    {
      "query": { "match_all": {} },
      "_source": ["field"]
    }
    
>查询筛选(查询的value值有多个值时以空格分开)

    GET /index/_search
    {
      "query": { "match": { "key": value } }
    }
    
> 精确搜索包含"word1 word2"的记录

       GET /index/type/_search
       {
           "query" : {
               "match_phrase" : {
                   "key" : "word1 word2"
               }
           }
       }
       
#### 复合查询
>and

    GET /person/_search
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
    
>or
    
    GET /person/_search
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
    
    
>neither not都不符合的例子

    GET /person/_search
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
    
    
>多条件布尔查询: 筛选出名字为tinys但年龄不等于21的记录

    GET /person/_search
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


>范围查询

    GET /person/_search
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


>聚合操作

    GET /bank/_search
    {
      "size": 0,
      "aggs": {
        "group_by_state": {
          "terms": {
            "field": "state.keyword"
          }
        }
      }
    }       
 等价于SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC
 
 
>单范围查询
    
    GET binding_card_request_aliases/_search
    {
      "query": {
        "bool": {
          "must": { "match_all": {} },
          "should": [
                      {"range": {
                          "bindingTime": {
                                          "gte": "2010-01-01 00:00:00"
                                        }
                                }
                      }
                    ]
                }
              }
    }
    
    

    
#### 批量操作
>批量更新

	POST /index/type/_bulk?pretty
	{"index":{"_id":"1"}}
	{"key": value,...}
	{"index":{"_id":"2"}}
	{"key": value,...}

>批量更新的例子

	POST /customer/customer/_bulk?pretty
	{"index":{"_id":"1"}}
	{"name": "tinys11" }
	{"index":{"_id":"2"}}
	{"name": "tinys22" }


>一个同时更新id为1以及删除id为2的批量操作

	POST /customer/customer/_bulk?pretty
	{"update":{"_id":"1"}}
	{"doc": { "name": "tiny11111" } }
	{"delete":{"_id":"2"}}    
    
    
    