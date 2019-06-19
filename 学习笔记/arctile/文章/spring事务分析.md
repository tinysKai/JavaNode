>Spring 对事务控制的支持统一在 TransactionDefinition 类中描述，该类有以下几个重要的接口方法：
+ int getPropagationBehavior()：事务的传播行为
+ int getIsolationLevel()：事务的隔离级别
+ int getTimeout()：事务的过期时间
+ boolean isReadOnly()：事务的读写特性。  

除了事务的传播行为外，事务的其它特性 Spring 是借助底层资源的功能来完成的，Spring 无非只充当个代理的角色。

>在传统的编程中，DAO必须执有一个Connection，而Connection即是状态化的对象。所以传统的DAO不能做成单实例的，每次要用时都必须new一个新的实例。传统的Service由于将有状态的DAO作为成员变量，所以传统的Service本身也是有状态的。  
但是在 Spring 中，DAO 和 Service 都以单实例的方式存在。Spring 是通过 ThreadLocal 将有状态的变量（如 Connection 等）本地线程化，达到另一个层面上的“线程无关”，从而实现线程安全。Spring 不遗余力地将状态化的对象无状态化，就是要达到单实例化 Bean 的目的。

>在相同线程中进行相互嵌套调用的事务方法工作于相同的事务中。
如果这些相互嵌套调用的方法工作在不同的线程中，不同线程下的事务方法工作在独立的事务中

>混合使用ORM(hibernate) + JDBC(mybatis,spring jdbc) 技术会造成有可能造成hibernate的缓存会覆盖掉mybatis的提交.  
使用 Hibernate 事务管理器后，可以混合使用 Hibernate 和 Spring JDBC 数据访问技术，它们将工作于同一事务上下文中。但是使用 Spring JDBC 访问数据时，Hibernate 的一级或二级缓存得不到同步，此外，一级缓存延迟数据同步机制可能会覆盖 Spring JDBC 数据更改的结果  
所以这种搭配模式下,最好采取hibernate进行读写操作,mybatis进行读操作


>由于 Spring 事务管理是基于接口代理或动态字节码技术，通过 AOP 实施事务增强的  
对于基于接口动态代理的AOP事务增强来说，由于接口的方法是public的,这就要求实现类的实现方法必须是public的(不能是protected,private等),
同时不能使用 static 的修饰符。所以，可以实施接口动态代理的方法只能是使用“public”或“public final”修饰符的方法，其它方法不可能被动态代理，相应的也就不能实施 AOP 增强，也不能进行 Spring 事务增强了。


参考文章 :  

https://www.ibm.com/developerworks/cn/java/j-lo-spring-ts1/index.html?ca=drs  
https://www.ibm.com/developerworks/cn/java/j-lo-spring-ts2/index.html?ca=dat-cn-0325#ibm-pcon
