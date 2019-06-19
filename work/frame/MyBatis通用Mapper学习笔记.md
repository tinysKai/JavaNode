# MyBatis通用Mapper学习笔记

### 快速入门

**依赖**

```xml
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper</artifactId>
    <version>最新版本</version>
</dependency>
```

**实体类**

```java
//@Table(name = "country")
public class Country implements Serializable {
    private static final long serialVersionUID = 1L;
    @Id
    @KeySql(useGeneratedKeys = true)
    private Long id;
    
    //@Column(name = "`countryname`")
    private String countryname;
    private String countrycode;
    
    //使用@Transient来配置非数据库字段
    @Transient
	private String otherThings; //非数据库表中字段
    
    //配置乐观锁
    @Version
  	private Integer version;

    //setter 和 getter 方法
}

//项目使用的例子
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
public class User {
	/**
	 * 自增主键
	 */
	@Id
	@GeneratedValue(generator = "JDBC")
	private Long id;

    
	private String name;
    
    //setter or getter
}    


```

经过上面简单的配置后，相当于就有了 MyBatis 中的 `<resultMap>` 关系映射了，**特别注意，这个映射关系只对通用 Mapper 有效，自己手写方法时，需要自己处理映射关系**。 

**Mapper接口**

```java
import tk.mybatis.mapper.common.Mapper;

public interface CountryMapper extends Mapper<Country> {}

//项目使用的例子
@Repository
public interface UserRepository extends Mapper<User> {
}
```

这里继承了 `tk.mybatis.mapper.common.Mapper` 接口，在接口上指定了泛型类型 `Country`。当你继承了 `Mapper` 接口后，此时就已经有了针对 `Country` 的大量方法 



**配置通用 Mapper**

```xml
<!--注意官方的包名和这里的包名的区别是开头为tk-->
<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="扫描包名"/>
    <!-- 其他配置 -->
</bean>

<!--项目使用的例子-->
<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage"
                  value="com.##.csm.workorder.**.repository,
                         com.##.csm.common.seq.db.repository"/>
        <property name="sqlSessionFactoryBeanName" value="##SqlSessionFactory"/>
        <property name="annotationClass" value="org.springframework.stereotype.Repository"/>
    </bean>
```



**参考链接**

快速入门指引 : https://blog.csdn.net/isea533/article/details/83045335#commentBox

实例 : https://github.com/abel533/Mapper/wiki/2.1-simple

wiki　：　https://github.com/abel533/Mapper/wiki



### 对象关系映射

**通用 Mapper 中，默认情况下是将实体类字段按照驼峰转下划线形式的表名列名进行转换。** 

映射参考文档 : https://github.com/abel533/Mapper/wiki/2.orm



##### 主键策略

**JDBC 支持通过 getGeneratedKeys 方法取回主键的情况**

```java
@Id
@GeneratedValue(generator = "JDBC")
private Long id;
```



##### 乐观锁

**使用乐观锁的方式**

```java
public class User {
 	private Long id;
  	private String name;
    
    //在属性配置一个映射数据库的字段作为乐观锁字段
    @Version
  	private Integer version;
  	//setter and getter
}
```



**支持的方法**

- delete
- deleteByPrimaryKey
- updateByPrimaryKey
- updateByPrimaryKeySelective
- updateByExample
- updateByExampleSelective

**注意**

在使用乐观锁时，由于通用 Mapper 是内置的实现，不是通过 **拦截器** 方式实现的，因此当执行上面支持的方法时，如果**版本**不一致，那么执行结果**影响的行数**可能就是 0。**这种情况下也不会报错！** 

所以在 Java6,7中使用时，你需要自己在调用方法后进行判断是否执行成功。 

在 Java8+ 中，可以通过默认方法来增加能够自动报错（抛异常）的方法，例如： 

```java
public interface MyMapper<T> extends Mapper<T> {
  
  default int deleteWithVersion(T t){
    int result = delete(t);
    if(result == 0){
      throw new RuntimeException("删除失败!");
    }
    return result;
  }
  
  default int updateByPrimaryKeyWithVersion(Object t){
    int result = updateByPrimaryKey(t);
    if(result == 0){
      throw new RuntimeException("更新失败!");
    }
    return result;
  }
  
  //...
  
}
```



### 配置介绍

如果你需要对通用 Mapper 进行特殊配置，可以按下面的方式进行配置： 

```xml
<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="tk.mybatis.mapper.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    <property name="properties">
        <value>
            参数名=值
            参数名2=值2
            ...
        </value>
    </property>
</bean>
```



#### 以下为重要参数罗列

**notEmpty**

`insertSelective` 和 `updateByPrimaryKeySelective` 中，是否判断字符串类型 `!=''`。

配置方式：

```properties
notEmpty=true
```



**useSimpleType**

默认 `true`，启用后判断实体类属性是否为表字段时校验字段是否为简单类型，如果不是就忽略该属性，这个配置优先级高于所有注解。

**注意：byte, short, int, long, float, double, char, boolean 由于存在默认值，这里不会作为简单类型对待！也就是默认情况下，这些字段不会和表字段进行映射。2.2.5 中也强调了这一点。**

配置方式如下：

```properties
useSimpleType=true
```



**usePrimitiveType**

为了方便部分还在使用基本类型的实体，增加了该属性，只有配置该属性，并且设置为 `true` 才会生效，启用后，会扫描 8 种基本类型。

配置方式如下：

```properties
usePrimitiveType=true
```



**safeDelete**

配置为 true 后，delete 和 deleteByExample 都必须设置查询条件才能删除，否则会抛出异常。

配置如下：

```properties
safeDelete=true
```

**safeUpdate**

配置为 true 后，updateByExample 和 updateByExampleSelective 都必须设置查询条件才能删除，否则会抛出异常（`org.apache.ibatis.exceptions.PersistenceException`）。

> updateByPrimaryKey 和 updateByPrimaryKeySelective 由于要求必须使用主键，不存在这个问题。

配置如下：

```properties
safeUpdate=true
```



参考链接 : https://github.com/abel533/Mapper/wiki/3.config



### 代码生成

**maven方式生成代码**

1.新生成一个maven项目,修改pom.xml如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tinys</groupId>
    <artifactId>code-gen</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!--  编译jdk版本  -->
        <jdk.version>1.8.0_112</jdk.version>
        <!--  依赖版本  -->
        <mybatis.version>3.4.6</mybatis.version>
        <mapper.version>4.0.3</mapper.version>
        <pagehelper.version>5.1.4</pagehelper.version>
        <mysql.version>5.1.29</mysql.version>
        <spring.version>4.1.2.RELEASE</spring.version>
        <mybatis.spring.version>1.3.2</mybatis.spring.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>${mybatis.version}</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>${mybatis.spring.version}</version>
        </dependency>
        <!-- Mybatis Generator -->
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.2</version>
            <scope>compile</scope>
            <optional>true</optional>
        </dependency>
        <!--分页插件-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper</artifactId>
            <version>${pagehelper.version}</version>
        </dependency>
        <!--通用Mapper-->
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper</artifactId>
            <version>${mapper.version}</version>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>nexus</id>
            <name>local private nexus</name>
            <url>http://maven.oschina.net/content/groups/public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>sonatype-nexus-releases</id>
            <name>Sonatype Nexus Releases</name>
            <url>http://oss.sonatype.org/content/repositories/releases</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>sonatype-nexus-snapshots</id>
            <name>Sonatype Nexus Snapshots</name>
            <url>http://oss.sonatype.org/content/repositories/snapshots</url>
            <releases>
                <enabled>false</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
    <plugins>
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8.0_112</source>
                <target>1.8.0_112</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.6</version>
            <configuration>
                <configurationFile>
                    ${basedir}/src/main/resources/generatorConfig.xml
                </configurationFile>
                <overwrite>true</overwrite>
                <verbose>true</verbose>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>5.1.29</version>
                </dependency>
                <dependency>
                    <groupId>tk.mybatis</groupId>
                    <artifactId>mapper</artifactId>
                    <version>4.0.0</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
    </build>
</project>
```

2.在resource文件夹下添加generatorConfig.xml文件

```xml
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>

    <context id="Mysql" targetRuntime="MyBatis3Simple" defaultModelType="flat">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <plugin type="tk.mybatis.mapper.generator.MapperPlugin">
            <property name="mappers" value="tk.mybatis.mapper.common.Mapper"/>
            <property name="caseSensitive" value="true"/>
        </plugin>
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://10.199.245.60:3306/mytest"
                        userId="root"
                        password="root">
        </jdbcConnection>

        <javaModelGenerator targetPackage="com.tinys.mybatis.model"
                            targetProject="src/main/java"/>

        <sqlMapGenerator targetPackage="mapper"
                         targetProject="src/main/resources"/>

        <javaClientGenerator targetPackage="com.tinys.mybatis.mapper"
                             targetProject="src/main/java"
                             type="XMLMAPPER"/>

        <table tableName="payment_order_test">
            <generatedKey column="id" sqlStatement="JDBC"/>
        </table>
    </context>
</generatorConfiguration>
```

3.在maven窗口执行`mybatis-generator:generate` 命令

4.生成文件

+ 实体类entity
+ 接口类Mapper
+ Mapper.xml文件



参考项目例子 : https://github.com/abel533/Mybatis-Spring



### Example用法

通用 Mapper 中的 Example 方法有两大类定义，一个参数和两个参数的，例如下面两个： 

```java
List<T> selectByExample(Object example);

int updateByExampleSelective(@Param("record") T record, @Param("example") Object example);
```

**用法例子**

查询

```java
Example example = new Example(Country.class);
example.createCriteria().andGreaterThan("id", 100).andLessThan("id",151);
example.or().andLessThan("id", 41);
List<Country> countries = mapper.selectByExample(example);

//对应日志
//DEBUG [main] - ==>  Preparing: SELECT id,countryname,countrycode FROM country WHERE ( id > ? and id < ? ) or ( id < ? ) ORDER BY id desc FOR UPDATE 
//DEBUG [main] - ==> Parameters: 100(Integer), 151(Integer), 41(Integer)
//DEBUG [main] - <==      Total: 90
```

排序

```java
Example example = new Example(Country.class);
example.orderBy("id").desc().orderBy("countryname").orderBy("countrycode").asc();
List<Country> countries = mapper.selectByExample(example);

//DEBUG [main] - ==>  Preparing: SELECT id,countryname,countrycode FROM country order by id DESC,countryname,countrycode ASC 
```

builder模式

```java
Example example = Example.builder(Country.class)
        .select("countryname")
        .where(Sqls.custom().andGreaterThan("id", 100))
        .orderByAsc("countrycode")
        .build();
List<Country> countries = mapper.selectByExample(example);

//对应SQL
//SELECT countryname FROM country WHERE ( id > ? ) order by countrycode Asc
```

项目例子

```java
  WorkOrderHeader header = new WorkOrderHeader();
  BeanUtil.copyProperties(request, header);

  Example headerExample = new Example(WorkOrderHeader.class);
  Example.Criteria headerCriteria = headerExample.createCriteria()
          .andEqualTo(header)
          .andGreaterThan("createTime", request.getCreateTimeBegin())
          .andLessThan("createTime", request.getCreateTimeEnd());
  headerExample.orderBy("expectFinishTime").asc()
                        .orderBy("priority").desc();

  workOrderHeaderRepository.selectByExampleAndRowBounds(example, rowBounds);

```

