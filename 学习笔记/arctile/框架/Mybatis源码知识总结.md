# Mybatis源码知识总结

## 流程图

![https://ws3.sinaimg.cn/large/005BYqpggy1g2faj6rloej30xh0j5js1.jpg](https://s2.ax1x.com/2019/04/26/EnWPyD.png)



![https://ws3.sinaimg.cn/large/005BYqpggy1g2g9fyt97ej312e0p2gmr.jpg](https://s2.ax1x.com/2019/04/26/EnnwSH.png)



## 知识点

- 动态代理模式来直接调用接口而无需写其实现类
- mybatis的拦截器
- TypeHandle的使用

### 拦截器使用

**官方中文文档**

`http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins`



#### 拦截器分类

默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)

#### 使用插件

使用插件的方式是实现 Interceptor 接口，并指定想要拦截的方法签名即可.

```java
//引用PageHelper插件源码
//详细说明见文档：https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/Interceptor.md
@Intercepts(
    {
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
    }
)
public class QueryInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        RowBounds rowBounds = (RowBounds) args[2];
        ResultHandler resultHandler = (ResultHandler) args[3];
        Executor executor = (Executor) invocation.getTarget();
        CacheKey cacheKey;
        BoundSql boundSql;
        //由于逻辑关系，只会进入一次
        if(args.length == 4){
            //4 个参数时
            boundSql = ms.getBoundSql(parameter);
            cacheKey = executor.createCacheKey(ms, parameter, rowBounds, boundSql);
        } else {
            //6 个参数时
            cacheKey = (CacheKey) args[4];
            boundSql = (BoundSql) args[5];
        }
        //TODO 自己要进行的各种处理
        //注：下面的方法可以根据自己的逻辑调用多次，在分页插件中，count 和 page 各调用了一次
        return executor.query(ms, parameter, rowBounds, resultHandler, cacheKey, boundSql);
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }

}
```

配置插件

```xml
<!-- mybatis-config.xml -->
<!--
	多个拦截器存在时,针对Executor的query方法的拦截,
	4 个参数的配置在 QueryInterceptor 拦截器的下面，
	6 个参数的配置在 QueryInterceptor 拦截器的上面。 
	按照这个顺序进行配置时，就能保证拦截器都执行
-->
<plugins>
    <plugin interceptor="com.github.pagehelper.QueryInterceptor"/>
</plugins>
```

------



#### 拦截器调用链的顺序

**按拦截器分类**

​	Executor拦截器     -->   StatementHandler拦截器

**相同类型拦截器执行顺序**

Mybatis的拦截器的执行代码

```java
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
        target = interceptor.plugin(target);
    }
    return target;
}
```

现假设配置以下三个Executor 类型的拦截器

```xml
<plugins>
    <plugin interceptor="com.tinys.ExecutorQueryInterceptor1"/>
    <plugin interceptor="com.tinys.ExecutorQueryInterceptor2"/>
    <plugin interceptor="com.tinys.ExecutorQueryInterceptor3"/>
</plugins>
```

上面的三个拦截器,那么对应的执行顺序是 : `3-->2-->1 -->target -->1-->2-->3`

分析过程如下

```
Interceptor3 前置处理      
Object result = Interceptor2..query();    
Interceptor3 后续处理   
return result;

Interceptor2 前置处理      
Object result = Interceptor1..query();     
Interceptor2 后续处理   
return result;

Interceptor1 前置处理      
Object result = executor.query();     
Interceptor1 后续处理   
return result;

//叠加到一起后，如下
Interceptor3 前置处理
Interceptor2 前置处理
Interceptor1 前置处理  
Object result = executor.query();     
Interceptor1 后续处理   
Interceptor2 后续处理  
Interceptor3 后续处理   
return result;
```

------



### 类型处理器

#### 自定义类型处理器

具体做法为：实现 `org.apache.ibatis.type.TypeHandler` 接口， 或继承一个很便利的类 `org.apache.ibatis.type.BaseTypeHandler`， 然后可以选择性地将它映射到一个 JDBC 类型

```java
//Mybatis源码的String转换
public class NStringTypeHandler extends BaseTypeHandler<String> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
      throws SQLException {
    ps.setNString(i, parameter);
  }

  @Override
  public String getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    return rs.getNString(columnName);
  }

  @Override
  public String getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    return rs.getNString(columnIndex);
  }

  @Override
  public String getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return cs.getNString(columnIndex);
  }

}
```

```xml
<!-- mybatis-config.xml -->
<typeHandlers>
  <typeHandler handler="org.mybatis.example.NStringTypeHandler"/>
</typeHandlers>

<!--或者使用package模式引入-->
<typeHandlers>
  <package name="org.mybatis.example"/>
</typeHandlers>
```

------

#### 动态代理

![https://ws3.sinaimg.cn/large/005BYqpggy1g2ge7jqlufj314f0o2taa.jpg](https://s2.ax1x.com/2019/04/26/EnRzJx.png)