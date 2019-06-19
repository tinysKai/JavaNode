##  Mybatis-PageHelper学习笔记

####  1.使用配置

**依赖**

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>最新版本</version>
</dependency>
```

spring配置

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <!-- 注意其他配置 -->
  <property name="plugins">
    <array>
      <bean class="com.github.pagehelper.PageInterceptor">
        <property name="properties">
          <!--使用下面的方式配置参数，一行配置一个 -->
          <value>
            params=value1
          </value>
        </property>
      </bean>
    </array>
  </property>
</bean>

<!--代码使用例子-->
<bean id="csmWorkflowMysqlSDKMyBatisSqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="csmWorkflowMysqlSDK"/>
        <property name="mapperLocations" value="classpath*:mapper/*Mapper.xml"/>
        <property name="plugins">
            <array>
                <bean class="com.github.pagehelper.PageInterceptor">
                    <property name="properties">
                        <value>
                            helperDialect=mysql
                            reasonable=true
                            supportMethodsArguments=true
                            autoRuntimeDialect=true
                        </value>
                    </property>
                </bean>
            </array>
        </property>
    </bean>
```



#### 2.使用例子

```java
/**
 * 分页工具类
 */
public class PagerUtil {
    /**
     * 将Pager对象转换为PageRowBounds对象
     *
     * @param pager  统一分页对象
     * @param runner 具体查询逻辑执行器
     * @param <T>    集合泛型类型
     * @return 查询结果集合
     */
    public static <T> List<T> withRowBounds(Pager pager, Function<PageRowBounds, List<T>> runner) {
        PageRowBounds rowBounds = new PageRowBounds(pager.getOffset(), pager.getPageSize());
        rowBounds.setCount(pager.isAutoCount());
        List<T> result = runner.invoke(rowBounds);
        if (rowBounds.getTotal() != null) {
            pager.setRecordCount(rowBounds.getTotal().intValue());
        }
        return result;
      }
    }

  //调用例子
  return PagerUtil.withRowBounds(pager, rowBounds -> {
            Example example = createListHeadersExample(request);
            return workOrderHeaderRepository.selectByExampleAndRowBounds(example, rowBounds);
          }).stream()
                    .map(this::toWorkOrderResponse)
                    .collect(Collectors.toList());

//补充下Function类
public interface Function<K, V> {
    V invoke(K obj);
}
```



#### 3.使用说明

**RowBounds方式的调用**

```
List<Country> list = sqlSession.selectList("x.y.selectIf", null, new RowBounds(1, 10));
```

使用这种调用方式时，你可以使用RowBounds参数进行分页，这种方式**侵入性最小**，我们可以看到，通过RowBounds方式调用只是使用了这个参数，并没有增加其他任何内容。

分页插件检测到使用了RowBounds参数时，就会对该查询进行**物理分页**。

关于这种方式的调用，有两个特殊的参数是针对 `RowBounds` 的，你可以参看上面的 **场景一** 和 **场景二**

**注：**不只有命名空间方式可以用RowBounds，使用接口的时候也可以增加RowBounds参数，例如：

```
//这种情况下也会进行物理分页查询
List<Country> selectAll(RowBounds rowBounds);  
```

**注意：** 由于默认情况下的 `RowBounds` 无法获取查询总数，分页插件提供了一个继承自 `RowBounds` 的 `PageRowBounds`，这个对象中增加了 `total` 属性，执行分页查询后，可以从该属性得到查询总数。



#### 开源地址

github地址:   https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md  

集成地址 :    https://github.com/abel533/Mybatis-Spring

