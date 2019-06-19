

## Mybatis原始SQL执行源码分析

#### 调用例子

```java
 SqlSession sqlSession = sqlSessionFactory.openSession();
 User user = sqlSession.selectOne("namespace.id", userId);
```



#### 从SqlSessionFactoryBuilder中获取SqlSessionFactory

```java
public class SqlSessionFactoryBuilder {
  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      //`parse.parse()`方法会调用`parseConfiguration`去解析XML配置文件
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
     
    //利用parse方法返回的Configuration去构造工厂类
    //这里使用了build模式  
    public SqlSessionFactory build(Configuration config) {
   		return new DefaultSqlSessionFactory(config);
  	}  
  }
```





#### 获取`SqlSession`

```java
//SqlSession工厂
public class DefaultSqlSessionFactory implements SqlSessionFactory {
  

public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}   

    
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //正常来讲这里返回的是`SpringManagedTransaction`事务类
      //注意这里的 `autoCommit`因为spring事务所以自动提交都为false
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //创建Executor,并创建`Executor`拦截器链  
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```



#### 获取Session流程图

![备用链接`https://images0.cnblogs.com/blog/677727/201412/041027590613706.png`](https://s2.ax1x.com/2019/04/20/EPluHH.png)





####  SqlSession执行

```java
//开始调用查询入口
public class DefaultSqlSession implements SqlSession {
  
  //比如调用列表查询时
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //根据传入的id到Configuration查找MappedStatement
      //id是指xml文件的`namespace.id`,所有的xml中的sql语句会在初始化时加载映射到`mappedStatements`中
      MappedStatement ms = configuration.getMappedStatement(statement);
      //到CachingExecutor执行器去执行查询,最终会调用6个参数的query方法  
      //Executor在`newExecutor`方法会使用`装饰者模式`来装饰`SimpleExecutor`方法
      //所以最终是调用`SimpleExecutor`的父类`BaseExecutor`的query方法 
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
} 
}

```

```java
public class CachingExecutor implements Executor {
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //获取要执行的SQL,主要作用是将SQL中的`#{}`参数替换为`?`,并且将对应值保存到BoundSql的parameterMappings属性中
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
}    
```



```java
//创建`StementHandler`对象
public class SimpleExecutor extends BaseExecutor {
    
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds,          	ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      //创建`PreparedStatementHandler`对象来执行SQL语句,配置`StementHandler`拦截器链
      //RoutingStatementHandler使用代理模式来委托多个不同类型的StementHandler处理SQL
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, 		   			rowBounds, resultHandler, boundSql);
      //创建java.Sql.Statement对象，传递给StatementHandler对象
      //对创建的Statement对象设置参数，即设置SQL 语句中 ? 设置为指定的参数  
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
}		    
```



```java
//预编译Handle类执行查询
public class PreparedStatementHandler extends BaseStatementHandler { 
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    //执行SQL语句  
    ps.execute();
    //使用ResultHandler来处理返回结果
    return resultSetHandler.<E> handleResultSets(ps);
  }
}    
```



```java
//结果集处理
public class DefaultResultSetHandler implements ResultSetHandler {
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
  
  //如果是普通的返回单个ResultSet结果集的最终会调用这个方法
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    //处理limit跳过的记录  
    skipRows(resultSet, rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }  
}   
```

#### SQL执行时序图

![备用链接`https://images0.cnblogs.com/blog/677727/201412/041322493892818.png`](https://s2.ax1x.com/2019/04/20/EPldEj.png)





#### 参考链接

执行流程分析 : https://blog.csdn.net/luanlouis/article/details/40422941

BoundSQL :  https://www.cnblogs.com/question-sky/p/7535482.html





//注意Executor是装饰者模式



//注意StatementHandler使用的是代理模式



//TODO  研究SpringManagedTransaction类

//TODO SqlSessionManager的调用链



 