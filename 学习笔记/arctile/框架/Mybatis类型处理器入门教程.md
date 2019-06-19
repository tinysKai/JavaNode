## Mybatis类型处理器入门教程

#### 自定义一个类型处理器类

```java

@MappedJdbcTypes(JdbcType.VARCHAR)
//通过类型处理器的泛型，MyBatis 可以得知该类型处理器处理的 Java 类型,比如下面这个类就是Date
public class ExampleTypeHandler extends BaseTypeHandler<Date> {
    @Override
    public Date getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String sqlTimetamp = rs.getString(columnName);
        if (null != sqlTimetamp){
            //将时间戳转为为日期Date类型
            return new Date(Long.valueOf(sqlTimetamp));
        }
        return null;
    }

    @Override
    public Date getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String sqlTimetamp = rs.getString(columnIndex);
        if (null != sqlTimetamp){
            //将时间戳转为为日期Date类型
            return new Date(Long.valueOf(sqlTimetamp));
        }
        return null;
    }

    @Override
    public Date getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String sqlTimetamp = cs.getString(columnIndex);
        if (null != sqlTimetamp){
            return new Date(Long.valueOf(sqlTimetamp));
        }
        return null;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Date parameter,JdbcType jdbcType) throws SQLException {
        //将Date类型保存为时间戳的字符串
        ps.setString(i, String.valueOf(parameter.getTime()));
    }

}
```

#### 实体类

```java

public class User {
    private int id;
    private Date date;
    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public Date getDate() {
        return date;
    }
    public void setDate(Date date) {
        this.date = date;
    }
}
```

#### Mapper类

```java
public interface UserMapper {
    Note query(int id);
    void insert(User user);
}
```



#### 配置声明类型处理器

**mybatis-config.xml中注册TypeHandler**

```xml
<configuration>
	<!-- 注意configuration中各个属性配置的顺序应为：properties,settings,typeAliases,typeHandlers,objectFactory,objectWrapperFactory,reflectorFactory,plugins,environments,databaseIdProvider,mappers)-->
    
    <typeHandlers>
        <typeHandler handler=".../ExampleTypeHandler" javaType="java.util.Date" jdbcType="VARCHAR"/>
         <!--或者直接使用包模式-->
         <!--
			<package name="org.mybatis.example.typeHandler"/>
		-->
    </typeHandlers>
      
    <mappers>
        <mapper resource=".../mapper/UserMapper.xml"/>
    </mappers>    
    
</configuration>

```

Mapper文件声明

```xml
<mapper namespace="com.mybatis.mapper.UserMapper">
    <resultMap type="com.mybatis.pojo.User" id="base">
        <result property="id" column="id"/>
        <result property="date" column="date" typeHandler="com.mybatis.util.ExampleTypeHandler"/>
    </resultMap>
    
    <select id="query" parameterType="int" resultMap="base">
        select * from user where id = #{id}
    </select>
    <insert id="insert" parameterType="com.mybatis.pojo.User">
        insert into user (id, date) values(#{id}, #{date, typeHandler=com.mybatis.util.ExampleTypeHandler})　　<!--使用自定义的TypeHandler插入数据-->
    </insert>
</mapper>
```



#### 总结

使用上面的`TypeHandler`就能在使用Date类型保存数据时自动转换为`Varchar`类型的时间戳.

当在 ResultMap 中决定使用哪种类型处理器时，此时 Java 类型是已知的（从结果类型中获得），但是 JDBC 类型是未知
因此 Mybatis 使用 javaType=[Java 类型], jdbcType=null 的组合来选择一个类型处理器。
这意味着使用 @MappedJdbcTypes 注解可以限制类型处理器的范围，同时除非显式的设置，否则类型处理器在 ResultMap 中将是无效的。 
如果希望在 ResultMap 中使用类型处理器，那么设置 @MappedJdbcTypes 注解的 includeNullJdbcType=true 即可。 
然而从 Mybatis 3.4.0 开始，如果只有一个注册的类型处理器来处理 Java 类型，那么它将是 ResultMap 使用 Java 类型时的默认值（即使没有 includeNullJdbcType=true）。 









































