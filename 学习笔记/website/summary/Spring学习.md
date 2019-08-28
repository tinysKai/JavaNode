## Spring学习

### Spring组成

Spring，始于框架，但不限于框架.

+ Spring Framework
+ Spring Boot
+ Spring Clound
+ 其它整个spring家族

#### Spring框架结构

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5djfigyejj20oz0im411.jpg)

#### Spring Clound架构

**主要功能点**

+  配置管理
+ 服务注册与发现
+  熔断
+ 服务追踪
+ ....

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5djitxfplj21000iyacg.jpg)

### Spring事务

#### Spring事务抽象

**PlatformTransactionManager**

+ DataSourceTransactionManager
+  HibernateTransactionManager
+  JtaTransactionManager

**TransactionDenition**

+ Propagation
+ Isolation
+ Timeout
+ Read-only status