## Mybatis解析SQL的过程

#### SQL执行器执行查询

```java
public class CachingExecutor implements Executor {
     public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //返回PrepareStatement所需的执行查询语句,就是将`#{}`替换为`?`,并且将参数按序保存到BoundSql的parameterMappings属性中     
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
}    
```



#### 分析过程

```java
public final class MappedStatement {
 public BoundSql getBoundSql(Object parameterObject) {
    //通过sqlSource以及传参获取BoundSql,我们具体看下`RawSqlSource`对应的构造方法来观察怎么转换SQL的 
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    //每个ParameterMapping 实例真正被使用的位置则是位于ParameterHandler接口唯一的实现者DefaultParameterHandler 中的setParameters方法中. 
    //所以我们在使用JDBC编程时进行的PreparedStatement.setXXX 操作其实是在每个 TypeHandler<T> 接口的实现类中完成的. 

    //检查传参是否为空,这里不太明白为啥需重新new一个 
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }

    return boundSql;
  }
}      
```



```java
public class RawSqlSource implements SqlSource {

  private final SqlSource sqlSource;

  public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
    this(configuration, getSql(configuration, rootSqlNode), parameterType);
  }

  public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    //使用`SqlSourceBuilder`来解析  
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<>());
  }

  private static String getSql(Configuration configuration, SqlNode rootSqlNode) {
    DynamicContext context = new DynamicContext(configuration, null);
    rootSqlNode.apply(context);
    return context.getSql();
  }

 
  //这里被`getBoundSql`方法调用, 通过`sqlSource`来获取`BoundSql`,我们通过构造方法来看这个全局变量
  public BoundSql getBoundSql(Object parameterObject) {  
    return sqlSource.getBoundSql(parameterObject);
  }

}
```

```java
//使用build模式来构造`SqlSource`
public class SqlSourceBuilder extends BaseBuilder {

  private static final String parameterProperties = "javaType,jdbcType,mode,numericScale,resultMap,typeHandler,jdbcTypeName";
    
  //重点的解析方法  
  public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
      
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    //转换占位符处理类  
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    //解析  
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }  
    
  private static class ParameterMappingTokenHandler extends BaseBuilder implements TokenHandler {

    private List<ParameterMapping> parameterMappings = new ArrayList<>();
    private Class<?> parameterType;
    private MetaObject metaParameters;

    public ParameterMappingTokenHandler(Configuration configuration, Class<?> parameterType, Map<String, Object> additionalParameters) {
      super(configuration);
      this.parameterType = parameterType;
      this.metaParameters = configuration.newMetaObject(additionalParameters);
    }


    @Override
    public String handleToken(String content) {  
      //#{}中的key属性以及相应的参数映射，比如javaType、jdbcType等信息均保存至BoundSql的parameterMappings属性中供最后的预表达式对象PrepareStatement赋值使用  
      parameterMappings.add(buildParameterMapping(content));
      return "?";
    }
}  
```

```java
//占位符转换类,占位符拆为三份,一份是前缀,一份是后缀,中间的内容
public class GenericTokenParser {
  //起始占位符,这里是`#{` 
  private final String openToken;
  //结尾占位符,这里是`}`  
  private final String closeToken;
  //内容处理器,这里是上面代码的ParameterMappingTokenHandler  
  private final TokenHandler handler;

  public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
    this.openToken = openToken;
    this.closeToken = closeToken;
    this.handler = handler;
  }
  
  //转换方法  
  public String parse(String text) {
    if (text == null || text.isEmpty()) {
      return "";
    }
    // search open token
    int start = text.indexOf(openToken);
    if (start == -1) {
      return text;
    }
    //转换为数组,方便拼凑  
    char[] src = text.toCharArray();
    //用于标记起始位置,循环下起始位置会变  
    int offset = 0;
    //SQL语句  
    final StringBuilder builder = new StringBuilder();
    //被替换的内容
    StringBuilder expression = null;
    while (start > -1) {
      if (start > 0 && src[start - 1] == '\\') {
        // this open token is escaped. remove the backslash and continue.
        builder.append(src, offset, start - offset - 1).append(openToken);
        offset = start + openToken.length();
      } else {
        // found open token. let's search close token.
        if (expression == null) {
          expression = new StringBuilder();
        } else {
          expression.setLength(0);
        }
        //builder拼凑前缀之前的字符  
        builder.append(src, offset, start - offset);
        //修改起始标记位  
        offset = start + openToken.length();
        //查找结尾占位符  
        int end = text.indexOf(closeToken, offset);
        while (end > -1) {
          if (end > offset && src[end - 1] == '\\') {
            // this close token is escaped. remove the backslash and continue.
            expression.append(src, offset, end - offset - 1).append(closeToken);
            offset = end + closeToken.length();
            end = text.indexOf(closeToken, offset);
          } else {
            //保存被替换的内容  
            expression.append(src, offset, end - offset);
            //修改下次的起始标记位  
            offset = end + closeToken.length();
            break;
          }
        }
        if (end == -1) {
          // close token was not found.
          builder.append(src, start, src.length - start);
          offset = src.length;
        } else {
          //处理被标记的内容,并且返回`?`  
          builder.append(handler.handleToken(expression.toString()));
          //修改下次的起始标记位    
          offset = end + closeToken.length();
        }
      }
      start = text.indexOf(openToken, offset);
    }
    if (offset < src.length) {
      builder.append(src, offset, src.length - offset);
    }
    return builder.toString();
  }
}
```



#### 设置值

```java
public class SimpleExecutor extends BaseExecutor {  
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      //初始化返回PreparedStatementHandler的代理类RoutingStatementHandler  
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //执行将上面的`?`与`ParameterMapping`一一对应设置值
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    //初始化statement  
    stmt = handler.prepare(connection, transaction.getTimeout());
    //设置参数,最终是调用`DefaultParameterHandler`去设置参数的  
    handler.parameterize(stmt);
    return stmt;
  }  
}
```

```java
public class DefaultParameterHandler implements ParameterHandler {
  @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          //通过类型处理器来设置值  
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            //设置值的代码  
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }

}
```

附上一个官方类型处理器的代码

```java
public class LongTypeHandler extends BaseTypeHandler<Long> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Long parameter, JdbcType jdbcType) throws SQLException {
    ps.setLong(i, parameter);
  }

  @Override
  public Long getNullableResult(ResultSet rs, String columnName) throws SQLException {
    long result = rs.getLong(columnName);
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Long getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    long result = rs.getLong(columnIndex);
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Long getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    long result = cs.getLong(columnIndex);
    return result == 0 && cs.wasNull() ? null : result;
  }
}
```

