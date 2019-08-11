## Spring Boot实战

#### Spring Boot核心

- 自动配置
- 起步依赖
- 命令行界面 - Spring Boot CLI
- Actuator - 提供在运行时检视应用程序内部情况的能力

#### 常见的自动化注解

+ @ConditionOnBean - 配置了某个指定类
+ @ConditionOnMissingBean - 木有配置某个指定类
+ @ConditionOnClass - classPath有指定的类
+ @ConditionOnMissingClass - classpath里缺少指定的类
+ @ConditionOnProperty - 指定的配置属性中有一个明确的值

#### Spring加载bean顺序

先加载应用级别的bean再加载自动化的bean

#### Spring Boot属性优先级

(1) 命令行参数
(2) java:comp/env里的JNDI属性
(3) JVM系统属性
(4) 操作系统环境变量
(5) 应用程序以外的application.properties或者appliaction.yml文件
(6) 打包在应用程序内的application.properties或者appliaction.yml文件
(7) 通过@PropertySource标注的属性源
(8) 默认属性

按照以上顺序优先级依次降低.第一个的优先级最高.如果你在同一优先级位置同时有application.properties和application.yml，那么application.yml里的属性会覆盖application.properties里的属性。

#### Bean的属性外配置

想在属性文件中配置bean属性可以通过在bean上加上`@ConfigurationProperties(prefix="${preStr}")`来实现

```java
@Component
@ConfigurationProperties(prefix="hello")
public class HelloWorldProperties {
    String key;
    //setter
}

//通过以上配置即可在application.property中添加属性,如hello.key
```

#### 针对环境配置变量

可在bean上使用`@Profile`注解来定义此bean在何环境下才生效.

针对具体的属性文件可使用`application-{profile}.properties`方式来区分不同环境的属性定义.而使用`application.properties`则是所有环境共用的.

#### 单元测试

​    Spring的SpringJUnit4ClassRunner可以在基于JUnit的应用程序测试里加载Spring应用程序上下文。在测试Spring Boot应用程序时，Spring Boot除了拥有Spring的集成测试支持，还开启了自动配置和Web服务器，并提供了不少实用的测试辅助工具。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes=HelloWorldConfiguration.class)
public class HelloWorldServiceTests {
  ...
}
```

#### Actuator

**端点信息**

| 路径            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| /autoconfig     | 提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过 |
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

**引入**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

```

**关闭端点**

默认情况下，所有端点（除了/shutdown）都启用。若想禁用部分端点可在配置文件中声明如`endpoints.shutdown.enabled`为false即可关闭shutdown端点,` endpoints.metrics.enabled `设置为false禁用metrics,如`endpoints.enabled`设置为false就能禁用Actuator的全部端点 

**创建自定义追踪仓库**

​    默认情况下，/trace端点报告的跟踪信息都存储在内存仓库里，100个条目封顶。一旦仓库满了，就开始移除老的条目，给新的条目腾出空间。可使用下面方式来修改默认值.

```java
package readinglist;
import org.springframework.boot.actuate.trace.InMemoryTraceRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ActuatorConfig {
  @Bean
  public InMemoryTraceRepository traceRepository() {
    InMemoryTraceRepository traceRepo = new InMemoryTraceRepository();
    traceRepo.setCapacity(1000);
    return traceRepo;
  }
}

```

若以上修改条数不满足需求的话,可使用持久化信息到外部系统中去,只须实现Spring Boot的TraceRepository接口即可.

```java
@Service
public class MongoTraceRepository implements TraceRepository {
  //依赖于spring boot的自动化配置
  private MongoOperations mongoOps;
   
  @Autowired
  public MongoTraceRepository(MongoOperations mongoOps) {
    this.mongoOps = mongoOps;
  }
  @Override
  //获取所有跟踪条目
  public List<Trace> findAll() {
    return mongoOps.findAll(Trace.class);
  }
  @Override
  //保存一个跟踪条目
  public void add(Map<String, Object> traceInfo) {
    mongoOps.save(new Trace(new Date(), traceInfo));
  }
}

```

#### 部署

**部署方式**

+ IDE中运行程序
+ 使用Maven的spring-boot:run或Gradle的bootRun，在命令行里运行
+ 使用Maven或Gradle生成可运行的JAR文件，随后在命令行中运行。
+ 使用Spring Boot CLI在命令行中运行Groovy脚本。
+ 使用Spring Boot CLI来生成可运行的JAR文件，随后在命令行中运行。

**部署选项**

| 部署产物 | 产生方式                     | 目标环境            |
| -------- | ---------------------------- | ------------------- |
| jar      | maven/gradle/spring boot cli | 云环境              |
| war      | maven/gradle                 | java运用环境/云环境 |

**构建war文件**

1. 脚本文件修改
2. 更新spring boot容器为provided
3. 更新war文件打包方式为war

**脚本修改**

```
//maven项目下,修改jar为war
<packaging>jar</packaging>改为<packaging>war</packaging>

//gradle下
apply plugin: 'war' //添加此插件
//修改jar方式为war
war {
    baseName = 'readinglist'
    version = '0.0.1-SNAPSHOT'
}

```

**排除内嵌服务**

```xml
<!--通过provided属性确保只在编译时会使用内嵌服务器,部署时不会-->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-tomcat</artifactId>
   <scope>provided</scope>
</dependency>
```

```java
@SpringBootApplication
//继承SpringBootServletInitializer子类
public class Demo2Application extends SpringBootServletInitializer {

    //重写configure方法
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Demo2Application.class);
    }
    public static void main(String[] args) {
        SpringApplication.run(Demo2Application.class, args);
    }
}
```

#### Spring Boot开发者工具

+ 自动重启：当Classpath里的文件发生变化时，自动重启运行中的应用程序
+ LiveReload支持：对资源的修改自动触发浏览器刷新。
+  远程开发：远程部署时支持自动重启和LiveReload。
+ 默认的开发时属性值：为一些属性提供有意义的默认开发时属性值。

**添加开发者工具引入**

```xml
<!--    
当应用程序以完整打包好的JAR或WAR文件形式运行时，开发者工具会被禁用，所以没有必
要在构建生产部署包前移除这个依赖。
-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
</dependency>
```

**自动重启**

​	在激活了开发者工具后，Classpath里对文件做任何修改都会触发应用程序重启。为了让重启速度够快，不会修改的类（比如第三方JAR文件里的类）都加载到了基础类加载器里，而应用程序的代码则会加载到一个单独的重启类加载器里。检测到变更时，只有重启类加载器重启。
​	可以设置`spring.devtools.restart.exclude`属性来覆盖默认的重启排除目录,如`spring.devtools.restart. exclude` = /static/\*\*,   /templates/\*\* .如想彻底关闭自动重启可将`spring.devtools.restart.enabled`设置为false.    

​	若想自己控制重启,则可设置一个触发文件,当修改了该文件后,IDE就会重新编译修改的文件.如`spring.devtools.restart.trigger-file` = trigger                                                     

默认的开发时属性

开启开发者工具后,一些缓存相关的属性将设置为false.如

-  spring.thymeleaf.cache
- spring.freemarker.cache
- spring.velocity.cache
- spring.mustache.cache
- spring.groovy.template.cache

























