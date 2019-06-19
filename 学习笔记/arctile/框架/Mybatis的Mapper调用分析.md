## Mybatis的Mapper调用分析

#### 调用例子

```java
 SqlSession sqlSession = sqlSessionFactory.openSession();
 // 创建Usermapper对象，mybatis自动生成mapper代理对象
 UserMapper mapper = sqlSession.getMapper(UserMapper.class);
 User user = mapper.findUserById(1);

 //纯接口
 public interface UserMapper {
    /** 根据ID查询用户信息 */
    public User findUserById(int id);
}
```



#### 原理

使用`动态代理`为接口创建实现类,在代理类中的`Method`类可以获取到被代理类对应的Pro`package`与要被代理的方法名`methodName`,对应到xml文件的`namespace.id`,则可以通过`SqlSession`中有`namespace.id`参数方法来调用

#### 约定

+ Mapper接口的包名以及签名需与XML对应

  

```xml
<!-- 比如User实体类的配置如下 -->
<!-- UserMapper.xml,主要注意其下的namespace -->
<mapper namespace="com.mybatis.mapper.UserMapper">

<!-- SqlMapConfig.xml,如果使用package模式则直接指定包名,否则指定Mapper类 -->    
 <mappers>
       <mapper resource="mapper/UserMapper.xml"/>
 </mappers>   
    
<!--Mybatis框架通过Mapper与代理工厂来映射-->    
```





#### 源码分析

获取`MapperProxyFactory`生成实例

```java
public class DefaultSqlSession implements SqlSession {
    //调用获取###Mapper方法 
    public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
}    

//到MapperRegistry去获取代理
public class MapperRegistry {
 private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
    
 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
      
   final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
   if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
   }
   try {
      //调用工厂方法来生成动态代理实例,比如UserMapper,就可以直接去调用selectList这些方法
      return mapperProxyFactory.newInstance(sqlSession);
   } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
   }
 }
}   

```



Mapper的代理工厂类

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  //返回代理类  
  protected T newInstance(MapperProxy<T> mapperProxy) {
    //针对动态代理三个参数说明下,第一个参数是classLoader,第二个参数是要被代理的接口
    //第三个是实现了InvocationHandler实现类,动态代理会调用到其重载方法invoke方法  
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

    
  public T newInstance(SqlSession sqlSession) {
    //生成InvocationHandler实现类  
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```



```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;//相应的缓存逻辑被我省略了
  }

  //所以这个代理类除了使用Map缓存之外,就是直接调用方法了
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //省略了method的Class是Object以及java8默认方法的处理逻辑
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //调用MapperMethod,如果是查询方法等同于调用调用sqlSession.selectList  
    return mapperMethod.execute(sqlSession, args);
  }
   
   //声明了MapperMethod,在其内声明了SqlCommand
   private MapperMethod cachedMapperMethod(Method method) {
    //使用Lambda来创建并缓存MapperMethod
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }  

}
```

```java
public class MapperMethod {

  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
  
  //调用的execute方法  
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          //常见的查询方法  
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
   
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
      //普通的查询的调用代码,`command.getName()`就是对应的`namespace.id`,在SqlCommand类中
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
    
}    
    
```

```java
public static class SqlCommand {
	//对应的`namespace.id`
    private final String name;
    //SQL类型
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      final String methodName = method.getName();
      final Class<?> declaringClass = method.getDeclaringClass();
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,configuration);
      if (ms == null) {
        if (method.getAnnotation(Flush.class) != null) {
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        //设置属性  
        name = ms.getId();
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }
    
    private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
        Class<?> declaringClass, Configuration configuration) {
      //关键的代码,这里通过Class的获取名字 + 方法名得到`namespace.id`  
      String statementId = mapperInterface.getName() + "." + methodName;
      if (configuration.hasStatement(statementId)) {
        //从configuration返回MappedStatement  
        return configuration.getMappedStatement(statementId);
      } else if (mapperInterface.equals(declaringClass)) {
        return null;
      }
      for (Class<?> superInterface : mapperInterface.getInterfaces()) {
        if (declaringClass.isAssignableFrom(superInterface)) {
          MappedStatement ms = resolveMappedStatement(superInterface, methodName,
              declaringClass, configuration);
          if (ms != null) {
            return ms;
          }
        }
      }
      return null;
    }
  }
    
    public String getName() {
      return name;
    }
}  
```





#### 时序图

######  获取MapperProxy时序图

![备用地址`https://images0.cnblogs.com/blog/677727/201412/041116257808987.png`](https://s2.ax1x.com/2019/04/20/EiuORf.png)

###### SQL执行时序图

![备用链接`https://images0.cnblogs.com/blog/677727/201412/041322493892818.png`](https://s2.ax1x.com/2019/04/20/EiG3EF.png)





#### 参考链接

`https://www.cnblogs.com/dongying/p/4142476.html`