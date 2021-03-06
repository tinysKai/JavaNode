《Elasticsearch服务器开发》读书笔记
=

概念
-

##### Lucene基本概念
 + 文档（document）:索引和搜索时使用的主要数据载体，包含一个或多个存有数据的字段。
 + 字段（field）:文档的一部分，包含名称和值两部分。
 + 词（term）: 一个搜索单元，表示文本中的一个词。
 + 标记（token）:表示在字段文本中出现的词，由这个词的文本、开始和结束偏移量以及类型组成。
 

>每个索引分为多个段.  
建立索引时,一个段写入磁盘后就不能再更新了.因此,被删除的文档信息存储在一个单独的文件中,但该段自身不被更新.  
然后多个段可以通过`段合并`合并到一起.合并时,会删除不再需要的信息(如被删除的文档).  
检索大段比检索小段要快,因小段的寻找还得合并结果.

>数据转换的过程称为分析.  
分析的工作由分析器完成，它由一个分词器(tokenizer)和零个或多个标记过滤器(token filter)组成，也可以有零个或多个字符映射器(character mapper).  
Lucene中的分词器把文本分割成多个标记，基本就是词加上一些额外信息，比如该词在原始文本中的位置和长度。分词器的处理结果称为标记流（token stream），它是一个接一个的标记，准备被过滤器处理。


索引
-

###### 属性设置
>设置索引的自动创建  
+ 简单模式  : action.auto_create_index: false/true
+ 复杂模式  : action.auto_create_index: -an*,+a*,-* (允许自动创建以a开头的索引,但以an开头的索引则不允许)[注意定义模式的顺序很重要,es的规则是第一匹配原则]

>在创建索引时禁用字段类型猜测,可以把dynamic属性设置为false.设置完后,在创建索引未提到的字段将会被es忽略.

>es索引中字段的核心类型
+ string 
+ number
+ date
+ boolean
+ binary



搜索
-

##### 搜索类型
+ query_then_fetch 分为发散以及手机阶段.分散阶段-先到每个分片查询对应的标识符以及得分,收集阶段-到相应分片搜索结果.  
  &amp;&amp;返回结果的最大数量等于size参数的值,默认搜索类型
+ query_and_fetch  所有分片都返回等于size值得结果数,  
  &amp;&amp;返回文档的最大数量等于size的值乘以分片的数量,最快也是最简单的搜索类型实现.
+ count  只返回匹配查询的文档数
+ scan   类似数据库的游标查询,只有在查询返回大量结果时使用
 

##### 搜索执行偏好
+ _primary 只在主分片搜索,不使用副本
+ _primary_first 优先使用主分片
+ _local 可能的情况下,只在发送请求的节点的可用分片上执行搜索
+ _only_node:node_id 只在某节点上搜索
+ _prefer_node:node_id 优先在某节点上搜索
+ _shards:1,2 在某几个分片上执行搜索

  
  