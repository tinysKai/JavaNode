#### mybatis是什么,有什么优点
MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架。MyBatis不是ORM工具，是一种查询映射工具。
MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。  
Mybatis自我总结 : 是一个SQL映射的框架

##### mybatis的核心组件
+ SqlSessionFactoryBuilder（构造器）  : 它会根据配置信息或者代码生成SqlSessionFactory
+ SqlSessionFactory（工厂接口）       : 依靠工厂来生成SqlSession
+ SqlSession（会话接口）              : 即可以发送SQL去执行并返回结果,也可以获取Mapper的接口
+ SQL Mapper（映射器）                : 新组件,有一个接口以及XML文件组成,需要给出对应的SQL和映射关系/它负责发送SQL并返回结果

