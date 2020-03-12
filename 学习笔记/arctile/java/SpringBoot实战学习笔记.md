## SpringBoot实战学习笔记

### 核心

+ 自动配置
+ 起步依赖
+ 命令行界面
+ Actuator : 提供在运行时检视应用程序内部情况的能力。

### 属性配置加载顺序

1. 命令行参数
2.  java:comp/env里的JNDI属性
3. JVM系统属性
4. 操作系统环境变量
5. 随机生成的带random.*前缀的属性    （在设置其他属性时，   可以引用它们，比如${random.long}）
6.  应用程序以外的application.properties或者appliaction.yml文件
7. 打包在应用程序内的application.properties或者appliaction.yml文件
8.  通过@PropertySource标注的属性源
9.  默认属性

任何在高优先级属性源里设置的属性都会覆盖低优先级的相同属性。例如，命令行参数会覆盖其他属性源里的属性。

**application.properties和application.yml文件能放在以下四个位置。**

1. 外置，在相对于应用程序运行目录的/config子目录里。

2.  外置，在应用程序运行的目录里。

3. 内置，在config包内。

4.  内置，在Classpath根目录。

   

   同样，这个列表按照优先级排序。也就是说，/config子目录里的application.properties会覆盖

应用程序Classpath里的application.properties中的相同属性。此外，如果你在同一优先级位置同时有application.properties和application.yml，那么**application.yml里的属性会覆盖application.properties里的属性**。

> 需注意目前YAML还无法通过@PropertySource注解来加载配置

### 属性配置

使用`@ConfigurationProperties`注解来加载配置属性

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

//使用bean属性来配置属性,可在`application.properties`,环境变量或命令行中使用`amazon.associateId`来配置属性
@Component
@ConfigurationProperties("amazon")               
public class AmazonProperties {
                                                 
  private String associateId;
    
  public void setAssociateId(String associateId) {
    this.associateId = associateId;
  }
  public String getAssociateId() {
    return associateId;
  }
}

```



### 单元测试

Spring的`SpringJUnit4ClassRunner`可以在基于JUnit的应用程序测试里加载Spring应用程序上下文。在测试Spring Boot应用程序时，Spring Boot除了拥有Spring的集成测试支持，还开启了自动配置和Web服务器，并提供了不少实用的测试辅助工具。

```java
@RunWith(SpringJUnit4ClassRunner.class)    //在spring4.2中可使用SpringClassRule和SpringMethodRule来代替SpringJUnit4ClassRunner                                                
@SpringApplicationConfiguration(classes=AddressBookConfiguration.class) 
//SpringApplication不仅加载应用程序上下文，还会开启日志、（application.properties或application.yml） 以及其他Spring Boot特性。加载外部属性                                           
public class AddressServiceTests {
                                                     
  @Autowired
  private AddressService addressService;
    
    
  @Test
  public void testService() {
    Address address = addressService.findByLastName("Sheman");
    assertEquals("P", address.getFirstName());
  }
}

```



### 模拟Spring MVC的单元测试

 要在测试中测试MVC,可使用`MockMvcBuilders`的静态`webAppContextSetup`方法：使用Spring应用程序上下文来构建Mock MVC，该上下文里 可以包含一个或多个配置好的控制器。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = ReadingListApplication.class)
@WebAppConfiguration
public class MockMvcWebTests {
@Autowired
private WebApplicationContext webContext;
private MockMvc mockMvc;
    
//产生一个mockMvc实例供单元测试时使用    
@Before
public void setupMockMvc() {
  mockMvc = MockMvcBuilders
      .webAppContextSetup(webContext)
      .build();
  }
}

@Test
public void homePage() throws Exception {
  mockMvc.perform(MockMvcRequestBuilders.get("/readingList"))
      .andExpect(MockMvcResultMatchers.status().isOk())
      .andExpect(MockMvcResultMatchers.view().name("readingList"))
      .andExpect(MockMvcResultMatchers.model().attributeExists("books"))
      .andExpect(MockMvcResultMatchers.model().attribute("books",
          Matchers.is(Matchers.empty())));
}

```

@WebAppConfiguration注解声明，由SpringJUnit4ClassRunner创建的应用程序上下文应该是一个WebApplicationContext（相对于基本的非WebApplicationContext）。

###  Actuator

 Actuator端点

| 路径            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| /autoConfig     | 提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过 |
| /configprops    | 描述配置属性（包含默认值）如何注入Bean                       |
| /beans          | 描述应用程序上下文里全部的Bean，以及它们的关系               |
| /dump           | 获取线程活动的快照                                           |
| /env            | 获取全部环境属性                                             |
| /env/{name}     | 根据名称获取特定的环境属性值                                 |
| /health         | 报告应用程序的健康指标，这些值由HealthIndicator的实现类提供  |
| /info           | 获取应用程序的定制信息，这些信息由info打头的属性提供         |
| /mappings       | 描述全部的URI路径，以及它们和控制器（包含Actuator端点）的映射关系 |
| /metrics        | 报告各种应用程序度量信息，比如内存用量和HTTP请求计数         |
| /metrics/{name} | 报告指定名称的应用程序度量值                                 |
| /trace          | 提供基本的HTTP请求跟踪信息（时间戳、HTTP头等）               |
| /shutdown       | 关闭应用程序，要求endpoints.shutdown.enabled设置为true,得使用POST方法 |

要启用Actuator的端点，只需在项目中引入Actuator的起步依赖即可

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 部署

![QQ截图20200303194350.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcgz7rdf3tj214w07ngn3.jpg)

gradle项目构建war文件

首先需使用war插件`apply plugin: 'war'`,然后在build.gradle里用以下war配置替换原来的jar配置,之后使用`gradlew build`来构建

```shell
# 两者的唯一区别就是 j 换成了w。
war {
    baseName = 'readinglist'
    version = '0.0.1-SNAPSHOT'
}
```

maven项目

修改pol.xml中的package为`<packaging>war</packaging>`,最后使用`mvn target`来构建

## JAVAEE开发的颠覆者--Spring Boot实战

### 关闭特定的自动配置

@SpringBootApplication(exclude={DataSourceAutoConfiguration.class})

### 属性配置

+ 使用`@value`注解在属性上直接声明
+ 基于`@ConfigurationProperties`注解来将属性与bean关联

```properties
person.name=tinys
person.age=24
```

使用`@Value`注解来注入属性

```java
@Component
public class Person{
    @Value("${person.name}")
    String name;
    @Value("${person.age}")
    String age;
    
}
```

使用`@ConfigurationProperties`来注入属性,若以下bean不想使用`@Component`来注入,可在调用方使用`@EnableConfigurationProperties`来为其注入,示例如`@EnableConfigurationProperties(Person.class)`

```java
@Component
@ConfigurationProperties(prefix="person")
public class Person{
    String name;
    String age;
    
}
```

在application.properties中的各个参数之间可以直接通过PlaceHolder的方式来进行引用,如下

```properties
person.name=tinys
person.age=15
person.message=${person.name} is ${person.age} old
```

在application.properties中配置随机数

```properties
#随机字符串
test.random.value=${random.value}
#随机int
test.random.number=${random.int}
#随机long
test.random.bignumber=${random.long}
#10以内的随机数
test.random.test1=${random.int(10)}
# 10~20的随机数
test.random.test2=${random.int[10,20]}
```



### 日志配置

```properties
logging.file=D:/mylog.log
logging.level.${package}=DEBUG
```



### Tomcat配置

Tomcat的配置都以`server.tomcat`作为前缀,如

```properties
# 统一的servlet容器的配置
server.port=8080
server.context-path= ${默认访问路径,默认为/}

# Tomcat的配置
server.tomcat.uri-encoding = 
```



### 事务

Spring Boot无需显式开启事务,默认即开启事务(`@EnableTransactionManagement`)

### 缓存

Spring Boot支持以`spring.cache`为前缀的属性来配置缓存

```properties
spring.cache.type=redis|encache|guava
spring.ext.cache.name=${cacheName}
```

在Spring Boot环境下，使用缓存技术只需在项目中导入相关缓存技术的依赖包，并在配置类使用`@EnableCaching`开启缓存支持即可。

### 模板热部署

在SpringBoot里，模板引擎的页面默认是开启缓存的，如果修改了页面的内容，则刷新页面是得不到修改后的页面的因此，我们可以在application.properties中关闭模板引擎的缓存,如

```properties
spring.freeemarker.cache=false

spring.velocity.cache=false

```



### 打包成war部署到tomcat

#### pom.xml中修改打包形式

```xml
 <packaging>war</packaging>
```

#### 移除嵌入式tomcat插件

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 移除嵌入式tomcat插件 -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

#### 添加servlet-api的依赖

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>

 <!-- 或者 -->

<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-servlet-api</artifactId>
    <version>8.0.36</version>
    <scope>provided</scope>
</dependency>
```

#### 修改启动类，并重写初始化方法

```java

/**
 * 修改启动类，继承 SpringBootServletInitializer 并重写 configure 方法
 */
public class SpringBootStartApplication extends SpringBootServletInitializer {
 
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        // 注意这里要指向原先用main方法执行的Application启动类
        return builder.sources(Application.class);
    }
}
```

