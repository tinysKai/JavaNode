## Java核心技术36讲笔记

## 基础

#### getByte方法实战建议

*getBytes和String相关的转换时根据业务需要建议指定编码方式，如果不指定则看看JVM参数里有没有指定file.encoding参数，如果JVM没有指定，那使用的默认编码就是运行的操作系统环境的编码了，那这个编码就变得不确定了。常见的编码iso8859-1是单字节编码，UTF-8是变长的编码。*

intern方法目的是提示 JVM 把相应字符串缓存起来，以备重复使用。Intern是一种显式地排重机制,需明确调用,但每一个字符串都显式调用时非常麻烦的,甚至是代码污染.

#### 自动装箱与拆箱

javac替我们自动把装箱转换为 Integer.valueOf()，把拆箱替换为 Integer.intValue().而`valueOf`方法使用了缓存

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```



#### Error & Exception

Throwable

+ exception

  + checked exception
    + IOException
    + InterruptedException
  + unchecked exception(RuntimeException及其子类)
    + NullPointerException
    + ArrayIndexOutofBoundsException
    + NoSuchMethodException 
    + ClassCastException
    + ClassNotFoundException  
    + FileNotFoundException
    + IndexOutOfBoundsException

+ error

  + OutOfMemoryError

  + StackOverflowError

  + NoClassDefFoundError

#### Mysql的默认隔离级别

+ 未提交读(会脏读)
+ 可提交读(不会出现脏读,会出现不可重复读)
+ 可重复读(默认innodb隔离级别,可重复读,并且由于MVCC机制不会出现幻读)
+ 串行化读



#### 从架构的角度,mybatis分为几层,都有哪些模块?

  mybatis架构自下而上分为基础支撑层、数据处理层、API接口层这三层。

+ 基础支撑层，主要是用来做连接管理、事务管理、配置加载、缓存管理等最基础组件，为上层提供最基础的支撑
+ 数据处理层，主要是用来做参数映射、sql解析、sql执行、结果映射等处理，可以理解为请求到达，完成一次数据库操作的流程。
+ API接口层，主要对外提供API，提供诸如数据的增删改查、获取配置等接口。  