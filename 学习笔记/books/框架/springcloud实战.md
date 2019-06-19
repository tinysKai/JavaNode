# Spring cloud微服务实战学习笔记

![springCloud](https://github.com/tinysKai/Note/blob/master/image/article/2018/1012/2018101601.png)

## 服务治理 Eureka

#### HelloWorld

**启动注册中心**

1. 创建spring boot项目

2. 在`Spring Initializr`选项中选择`Cloud Discovey`选项的Eureka Server等模块

3. 配种pom.xml

   ```xml
   <dependencies>
       <dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   		</dependency>
   </dependencies>    
   
   <dependencyManagement>
   		<dependencies>
   			<dependency>
   				<groupId>org.springframework.cloud</groupId>
   				<artifactId>spring-cloud-dependencies</artifactId>
   				<version>${spring-cloud.version}</version>
   				<type>pom</type>
   				<scope>import</scope>
   			</dependency>
   		</dependencies>
   </dependencyManagement>
   ```

4. 配置property文件

   ```properties
   #设置当前环境,一般不是放这里的
   spring.profiles.active=dev
   
   #设置eureka的环境
   eureka.environment=dev
   
   #服务注册中心端口号
   server.port=8888
   
   #服务注册中心实例的主机名
   eureka.instance.hostname=localhost
   
   #是否向服务注册中心注册自己
   eureka.client.register-with-eureka=false
   
   #是否检索服务
   eureka.client.fetch-registry=false
   
   #服务注册中心的配置内容，指定服务注册中心的位置
   eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
   
   ```

5. 启动文件加`EnableEurekaServer`注解

   ```java
   @EnableEurekaServer
   @SpringBootApplication
   public class EurekaApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(EurekaApplication.class, args);
   	}
   }
   ```

6. 运行项目,并在浏览器登录`localhost:8888`查看



**注册服务提供者**

1. 在启动类添加注解`@EnableEurekaClient`

   ```java
   @EnableEurekaClient
   @SpringBootApplication
   public class EurekarRegisterApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(EurekarRegisterApplication.class, args);
   	}
   }
   ```

2. 在配置文件添加服务名以及注册路径

   ```properties
      server.port=7777
          
      #设置当前环境,一般不是放这里的
      spring.profiles.active=dev
   
      #设置eureka的环境
      eureka.environment=dev
   
      #服务名
      spring.application.name=helloClient
   
      #注册路径
      eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka/
       
   ```



​    **高可用注册中心**

1. 修改注册中心配置

    注册中心1

   ```properties
   #服务注册中心端口号
   server.port=8888
   
   #服务注册中心实例的主机名
   eureka.instance.hostname=localhost
   
   #同一个注册中心的服务名相同
   spring.application.name=helloServer
   
   #是否向服务注册中心注册自己
   #eureka.client.register-with-eureka=false
   
   #是否检索服务
   #eureka.client.fetch-registry=false
   
   #服务注册中心的配置内容，指定服务注册中心的位置,向集群的另一个注册中心注册自己
   eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:8889/eureka/
   ```

   注册中心2

   ```properties
   #服务注册中心端口号
   server.port=8889
   
   #服务注册中心实例的主机名
   eureka.instance.hostname=localhost
   
   spring.application.name=helloServer
   
   #是否向服务注册中心注册自己
   #eureka.client.register-with-eureka=false
   
   #是否检索服务
   #eureka.client.fetch-registry=false
   
   #服务注册中心的配置内容，指定服务注册中心的位置,向集群的另一个注册中心注册自己
   eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:8888/eureka/
   ```

2. 修改服务提供方配置

   ```properties
   server.port=7777
   
   #服务名
   spring.application.name=helloClient
   #服务方需向集群中的所有注册中心注册
   eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka/,http://localhost:8889/eureka/,
   ```

   

   **服务发现与消费**

1. 启动两个服务者

2. 新建项目consume,并需依赖ribbon模块

   ```xml
   	<dependencies>
   		<dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
   		</dependency>
   
   		<dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   		</dependency>
   
   		<dependency>
   			<groupId>org.springframework.boot</groupId>
   			<artifactId>spring-boot-starter-test</artifactId>
   			<scope>test</scope>
   		</dependency>
   	</dependencies>
   ```

   

3. 在consume项目搭建restTemplate

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumeApplication {

    
	@Bean
	@LoadBalanced //开启客户端负载均衡
	RestTemplate restTemplate(){
		return  new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumeApplication.class, args);
	}
}
```

```java
@RestController
public class ConsumeController {
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/ribbon-consume")
    public String helloConsume(){
        //通过服务名而不是IP或其他信息调用
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }
}
```

```properties
server.port=9991

#服务名
spring.application.name=helloConsume
#服务方需向集群中的所有注册中心注册
eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka/,http://localhost:8889/eureka/,
```



  **概念**

服务治理三个核心要素

- 注册中心
- 服务提供者
- 服务消费者

**配置详解**

Eureka客户端配置分类

- 注册相关配置信息 : 注册中心地址,服务获取的间隔时间,可用区域等
- 实例相关配置信息 : 实例名称,IP地址,端口,健康检查地址

Eureka的服务治理机制强调了CAP原理中的`AP`,即可用性与可靠性.

比如,在服务注册中心的网络发生故障断开时,由于所有的服务实例无法维持续约心跳,在强调`CP(一致性,可靠性)`的服务中将会把所有服务都剔除掉,而在Eureka中则会因为超过85%的实例丢失心跳而出发保护机制,注册中心将会保留此时的所有节点,以实现服务间依旧可以互相调用的场景.



## 客户端负载均衡-Ribbon



## 服务容错保护-Hystrix

#### 使用

1.pom.xml引入

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

2.修改启动类,添加`@EnableHystrix`注解

```java
@EnableHystrix
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumeHystrixApplication {
	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return  new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumeHystrixApplication.class, args);
	}
}
```

3.增加service类,使用`@HystrixCommand`注解指定了服务降级的方法

```java
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;


    @HystrixCommand(fallbackMethod = "helloFallback")
    public String helloConsume(){
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }

    public String helloFallback(){
        return  "error";
    }
}
```

4.修改Controller类

```java
@RestController
public class ConsumeController {

    @Autowired
    HelloService helloService;

    @RequestMapping("/ribbon-consume")
    public String helloConsume(){
        return  helloService.helloConsume();
    }
}

```

5.请求上面的链接时,会根据服务的状态进行熔断时得返回



#### 不需要进行降级的场景

- 执行写操作的命令 当Hystrix命令是执行写操作而不是返回信息时,通常返回时void,这时服务降级意义不大
- 执行批处理或离线计算命令



#### 异常传播

在`HystrixCommand`注解中抛出异常时,除`HystrixBadRequestException`外,其它异常均会被认为方法执行失败并触发降级逻辑.在使用Hystrix时,可支持忽略指定异常功能,如下使用`ignoreExceptions`属性表明忽略的异常

```java
  @HystrixCommand(fallbackMethod = "helloFallback",ignoreExceptions = SQLDataException.class)
    public String helloConsume(){
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }
```

如上代码在抛出SQLDataException异常时,会被Hystrix将此异常包装在`HystrixBadRequestException`抛出而不会触发后面的fallback逻辑.

#### 异常获取

针对不同的异常进行不同的降级策略

```java
	@HystrixCommand(fallbackMethod = "helloFallback")
    public String helloConsume(){
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }

	//实现了在回调方法里面传异常的参数
    public String helloFallback(Throwable e){
        logger.error("发生异常",e);
        return  "error";
    }
```



#### 请求缓存

在高并发场景下,Hystrix提供了请求缓存的功能,可以方便的开启和使用请求缓存来优化系统,达到减轻高并发时得请求线程消耗,降低请求响应时间的效果

| 注解         | 描述                                                         | 属性                      |
| ------------ | ------------------------------------------------------------ | ------------------------- |
| @CacheResult | 标记请求命令返回的结果应该缓存,必须与@HystrixCommand注解结合使用 | cacheKeyMethod            |
| @CacheRemove | 让请求命令的缓存失效,失效的缓存根据定义的key决定             | commandKey,cacheKeyMethod |
| @CacheKey    | 用来在请求命令的参数上标记,使其作为缓存的key值,如果木有标注则会使用所有参数.如果同时还是使用了@CacheResult和CacheRemove注解的cacheKeyMethod方法指导缓存key的生成,那么该注解不会起作用 | value                     |

```java
    @CacheResult(cacheKeyMethod = "getKeyIdMethod")
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String helloConsume(String name){
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }

    public String getKeyIdMethod(String name){
        return name;
    }

	//或者使用优先级较低的
	@CacheResult
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String helloConsume(@CacheKey("name") String name){
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }
```



#### 请求合并

Hystrix提供了HystrixCollapser来实现请求的合并,以减少通信消耗和线程数的占用.

HystrixCollapser实现了在HystrixCommand之前放置一个合并处理器,将处于很短的时间窗(默认10毫秒) 内对`同一依赖服务`的多个请求进行合并以批量方式发起请求的功能(服务提供方也需提供相应的批量实现接口).

```java
@HystrixCollapser(batchMethod = "findAll",collapserProperties = {
    		//设置窗口时间为100ms
            @HystrixProperty(name="timerDelayInMilliseconds",value="100")
    })
    public String find(Long id){
        return null;
    }

    @HystrixCommand
    public List<String> findAll(List<Long> ids){
        return restTemplate.getForObject("http://service/users?ids={1}",List.class, StringUtils.join(ids,","));
    }
```



#### Dashboard

配置

1.pom配置

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-actuator</artifactId>
</dependency>
```

2.确保启动类使用了`@EnableHystrixDashboard`注解

3.访问`http://localhost:port/hystrix`

4.注意如果项目的spring boot版本为2.0需要配置下以下bean

```java
@Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }      
```

5.配置完即可访问以下链接 : http://localhost:7991/hystrix

  在其链接中可访问某个服务的的`http://127.0.0.1:9992/hystrix.stream `的指标



#### Turbine集群控制





## 声明式服务调用 Feign

**基于Netflix Feign实现,整合了Ribbon与Hystrix,还提供了声明式的web服务端定义方式**



#### 快速入门

1.pom配置

```xml
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
		</dependency>

```

2.启动类

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumeApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumeApplication.class, args);
	}
}
```

3.添加一个调用服务方的接口

```java
@FeignClient("helloClient") //括号里为要调用的服务方名字
public interface  HelloService {
    @RequestMapping("/hello") //调用方法的路径
    String hello();
}
```

4.控制器

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumeApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumeApplication.class, args);
	}
}
```



 此实现方式主要是省略了重复的RestTemplate等模板代码,通过接口封装的方式来优化



#### 参数绑定

1.服务方代码

```java
 @RequestMapping("/hello")
    public String index(){
        return "HelloWorld";
    }

    //普通属性使用的是RequestParam注解或省略
    @RequestMapping("/hello1")
    public String hello1(String name ){
        return "HelloWorld" + name;
    }

	//对象使用的是RequestBody注解
    @RequestMapping("/hello2")
    public String hello2(@RequestBody  User user){
        return "HelloWorld" + user.getName() + ", you are " + user.getAge();
```



2.调用方代码.

```java
@FeignClient("helloClient")
public interface  HelloService {
    @RequestMapping("/hello")
    String hello();

    //注解不能省略,而且注解里面的value值也不能省
    @RequestMapping("/hello1")
    String hello1(@RequestParam("name") String name);

    //对象的注解是RequestBody
    @RequestMapping("/hello2")
    String hello2(@RequestBody User user);
}
```



#### 继承特性

封装实体类,定义API工程来存放实体类



#### Ribbon配置

Feign的负载均衡是通过Ribbon实现的,所以我们可以直接通过配置Ribbon客户端的方式来自定义各个服务端调用的参数



- 全局配置  ribbon.key = value
- 特定配置 clientName.ribbon.key = value



#### Hystrix配置

**全局配置**

​    使用默认配置前缀hystrix.command.default.# 

注意在进行配置前,需要确认`feign.hystrix.enabled`参数有无被设置为false,若被设置为false则会关闭Feign客户端的Hystrix支持.



**禁用Hystrix**

全局关闭 

`feign.hystrix.enabled` = false

部分服务关闭

```java
//配置指定的配置类
@Configuration
public class DisableHystrixConfiguration {
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder(){
        return Feign.builder();
    }
}

//在不想使用Hystrix的服务上引入上面的配置
@FeignClient(value = "helloClient",configuration = DisableHystrixConfiguration.class)
public interface  HelloService {
    @RequestMapping("/hello")
    String hello();
}
```



#### 服务降级配置

1.实现一个降级类

```java
@Component
public class HelloServiceFallback implements  HelloService {
    @Override
    public String hello() {
        return "error";
    }

    @Override
    public String hello1(String name) {
        return "error1";
    }

    @Override
    public String hello2(User user) {
        return "error2";
    }
}
```

2.在接口类的注解引入降价属性

```java
@FeignClient(name = "helloClient",fallback = HelloServiceFallback.class)
public interface  HelloService {
    @RequestMapping("/hello")
    String hello();

    @RequestMapping("/hello1")
    String hello1(@RequestParam("name") String name);

    @RequestMapping("/hello2")
    String hello2(@RequestBody User user);
}
```

3.注意feign的版本号

  I.可能需要添加以下属性 `feign.hystrix.enabled=true`来开启feign的hystrix功能

  II.引入hystrix的jar

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```



#### 其它配置

压缩配置

```properties
#请求和响应GZIP压缩支持
feign.compression.request.enabled=true
feign.compression.response.enabled=true


#指定压缩的请求数据类型 mime types
feign.compression.request.mime-types=text/xml,application/xml,application/json

#设置压缩请求的最小下限,只有请求大小大于这个size才会触发压缩
feign.compression.request.min-request-size=2048


```



#### 日志配置

1.配置文件定义类或包的日志级别

```properties
#logging.level.<FeignClient>  = DEBUG,  <FeignClient>为Feign客户端定义接口的完整路径  
logging.level.com.tinys.cloud.HelloService = DEBUG
```



2.若想创建全局日志

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumeApplication {

	@Bean
	Logger.Level feignLoggerLevel(){
		return Logger.Level.FULL;
	}

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumeApplication.class, args);
	}
}
```

3.若想针对服务创建特定的日志级别

```java
@Configuration
public class FullLogConfiguration {
    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}

//声明配置级别并在注解中引用
//@FeignClient(name = "helloClient",fallback = HelloServiceFallback.class,configuration = FullLogConfiguration.class)
```



## API网关服务 Zuul ​

#### 快速入门

1.pom.xml

```xml
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>		
```

2.启动类

```java
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
	public static void main(String[] args) {
		SpringApplication.run(GatewayApplication.class, args);
	}
}
```

3.配置文件

```properties
server.port=9994

#设置当前环境,一般不是放这里的
spring.profiles.active=dev

#设置eureka的环境
eureka.environment=dev

spring.application.name=helloGateway
eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka/,http://localhost:8889/eureka/,
#配置路由规则,当路由有前缀api-a时转发到helloClient服务
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.serviceId=helloClient

#配置路由规则,当路由有前缀api-b时转发到helloConsumeFeign服务
zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.serviceId=helloConsumeFeign
```

4.访问测试

```
访问http://localhost:9994/api-a/hello时会转发到helloClient服务下的hello路径

访问http://localhost:9994/api-b/feignConsume会转发到helloConsumeFeign服务下的feignConsume

```



#### 请求过滤

**实现**

1.写一个实现了ZuulFilter的过滤器

```java
package com.tinys.cloud;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.http.HttpServletRequest;

/**
 * Created by asus on 2018/10/13.
 */
public class AsscessFilter extends ZuulFilter {
    private Logger logger = LoggerFactory.getLogger(AsscessFilter.class);

    /**
     * 定义过滤器的类型
     * 顺序类型 :
     		pre : 请求路由之前调用
     		routing : 在路由请求时被调用
     		post    : 在routing和error过滤器之后被使用
     		error   : 处理请求时发生错误时被调用
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 定义过滤器的执行顺序,数字越小优先级越高
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 判断过滤器是否需要执行
     * 这里我们直接返回了true,代表对所有请求都需执行
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器的具体执行逻辑
     */
    @Override
    public Object run() throws ZuulException {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
        logger.info("send {} request to {}",
                request.getMethod(),
                request.getRequestURL().toString());

        Object token = request.getParameter("asscessToken");
        if (token == null){
            context.setSendZuulResponse(false);
            context.setResponseStatusCode(401);
        }
        return null;
    }
}

```

2.在启动类中增加该过滤器

```java
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
	public static void main(String[] args) {
		SpringApplication.run(GatewayApplication.class, args);
	}

	@Bean
	public  AsscessFilter asscessFilter(){
		return new AsscessFilter();
	}
}
```

3.访问

按照之前的请求地址访问 : http://localhost:9994/api-a/hello ,会返回401状态

按照过滤器的规则添加一个asscessToken参数则可访问成功,http://localhost:9994/api-a/hello?asscessToken=1



#### 网关在微服务中的作用

- 作为系统的统一入口，屏蔽了系统内部各个微服务的细节
- 它可以与服务治理框架结合，实现自动化的服务实例维护以及负载均衡的路由转发
- 它可以实现接口权限校验与微服务业务逻辑的解耦
- 通过网关中的过滤器，在各生命周期中去校验请求的内容，将原本在对外服务层做的校验前移，保证微服务的无状态性，届时降低了微服务的测试难度，让服务本身更集中关注业务逻辑处理

#### 服务路由的默认规则

默认情况下,大部分的路由配置规则几乎都采用了服务名作为外部请求前缀,如下

```properties
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceId=user-service
```



对于这些规则性的路由配置,zuul为我们提供了自动配置.当我们在zuul中引入Eureka后,它会为Eureka的每个服务自动创建一个路由规则,这些默认规则的path使用serviceId配置的服务名作为前缀.

对于不想对外开放的服务,我们可以使用`zuul.ignored-services`参数来设置一个服务名匹配表达式来定义不自动创建路由的规则.

比如定义`zuul.ignored-services = * `时,zuul将对所欲服务都不自动创建路由规则.

zuul还支持模式忽略,使用``zuul.ignored-patterns`参数, 比如

```properties
#会忽略所有hello接口
zuul.ignored-patterns = /**/hello/**
```



#### 禁用过滤器

```properties
#zuul.<SimpleClassName>.<filterType>.disable = true
zuul.AccessFilter.pre.disable = true
```



## 分布式配置中心 Config 
#### 服务端配置

1.pom.xml

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

2.启动类

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}
```

3.配置文件

```properties
#设置当前环境,一般不是放这里的
spring.profiles.active=dev

#设置eureka的环境
eureka.environment=dev

#服务注册中心端口号
server.port=5554

#服务注册中心实例的主机名
eureka.instance.hostname=localhost

spring.application.name=configServer

spring.cloud.config.server.git.uri = https://gitee.com/tinys/cloudConfig/
spring.cloud.config.server.git.search-paths =   repo
spring.cloud.config.server.git.username = 
spring.cloud.config.server.git.password  = 
```

4.访问

http://localhost:5554/tinys/dev

访问会返回服务端的配置信息

5.访问规则

```
在远程git服务上需根据`spring.cloud.config.server.git.search-paths`创建文件夹
然后再上面创建`application`为前缀的配置文件,比如,
tinys.properties
tinys-dev.properties
tinys-test.properties
tinys-pro.properties

如果刚刚http://localhost:5554/tinys/dev这个地址访问的是仓库的master分支,如需切其它分支则,
在其后添加分支名字,比如http://localhost:5554/tinys/dev/test
dev表示配置文件的环境,test表示git仓库的test分支

```

#### 客户端配置

1.pom.xml

```xml
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

2.在Resource文件下新建一个`bootstrap.properties`文件来存放config属性,必须放在`bootstrap.properties`内

```properties
#属性值跟Properties文件名字前缀一致
spring.application.name=tinys

spring.cloud.config.uri=http://localhost:5554/
spring.cloud.config.profile=dev
spring.cloud.config.label=master
```

3.访问配置的例子

```java
@RefreshScope
@RestController
public class TestController {
    @Value("${from}")
    private String from;   //访问配置文件的from属性

    @RequestMapping("/from")
    public String from(){
        return from;
    }
}
```



#### 失败快速响应与重试

**快速失败**

要实现客户端优先判断ConfigServer获取是否正常,并快速响应失败内容,只需在bootstrap.properties中配置参数

`spring.cloud.config.failFast=true`即可



**重试**

为避免网络波动而导致的直接启动失败,可开启自动重试功能.在开始自动重试前,请确保`spring.cloud.config.failFast=true`已配置.

然后在引入以下jar

```hxml
<dependency>
			<groupId>org.springframework.retry</groupId>
			<artifactId>spring-retry</artifactId>
</dependency>

<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

如上已完成快速失败与自动重试的配置.

#### 动态刷新配置

新增jar

```xml
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

客户端引入以上jar后,在远程配置git工程修改了值时,通过刷新`http://localhost:5553/refresh`可以实现客户端值的刷新.当然在学习完`spring cloud bus`后我们可以结合git hook钩子来实现配置变更



## 消息总线 bus

#### 消息代理

消息代理是一种消息验证,传输,路由的架构模式.