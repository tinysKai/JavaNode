《Elasticsearch技术解析与实战》读书笔记
=

概念
-

##### 定义
>ES是一个基于Lucene构建的开源,分布式,RESTFUL接口全文搜索引擎.  
ES还是一个分布式文档数据库,其中每个字段均是被索引的数据且可被搜索.

>全文检索是指计算机搜索程序通过扫描文章中的每一个词,对每一个词建立一个索引,指明该词在文章中出现的次数和位置,  
当用户查询时,搜索程序就根据事先建立的索引进行查找,并将查找的结果反馈给用户.

>Lucene是Apache下一个全文搜索引擎工具包,是一个全文搜索引擎的架构,包括了完整的查询引擎和索引引擎,部分文本分析引擎.  
Lucene的目的是为开发人员提供一个简单易用的工具包,以方便在目标系统中实现全文检索的功能,或是以此为基础建立完整的全文搜索引擎.

>倒排索引源于实际应用中需要根据属性值来查找记录.  
这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址.  
由于不是由记录来确定属性值,而是由属性值来确定记录的位置,因而称为倒排索引.  

>当你存储一个文档的时候,系统会首先存储在主分片中,然后复制到不同的副本中.  
默认情况下,一个索引有5个分片.`当分片一旦建立,则分片的数量不能修改`.  
每一个主分片有零个或多个副本.副本主要是主分片的复制.  
副本分片的目的 :   
  &nbsp;&nbsp;增加高可用性  
  &nbsp;&nbsp;提高性能(查询也可以到副本分片查询)
  
>默认情况下,每个索引分配5个分片和一个副本.这意味着你的集群节点至少要有两个节点,你将拥有5个主分片和5个副本分片共10个分片
  

>索引是具有相同结构的文档集合





##### 优点
+ 横向可扩展性
+ 分片机制(同一个索引分成多个分片)
+ 高可用(提供复制机制,一个分片可以设置多个复制)


后序章节
-
由于太多API相关的说明  
略
