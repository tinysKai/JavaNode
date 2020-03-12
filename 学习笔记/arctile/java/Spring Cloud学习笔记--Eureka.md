# Spring Cloud学习笔记 -- Eureka



## 定义

`Spring Cloud Eureka`基于`Netflix Eureka`做了二次封装，主要负责完成微服务架构中的服务治理功能。我们只需通过简单引入依赖和注解配置就能让SpringBoot构建的微服务应用轻松地与Eureka服务治理体系进行整合。

## 服务治理基本元素

+ 注册中心

+ 服务注册

+ 服务下线

+ 服务续约

+ 服务提供者

+ 服务消费者

  

## 基本架构

![QQ截图20200312095449.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcqwrmwr7ej20vn0e0dhp.jpg)

## 数据存储结构

![61a26fffaeaf63d5dd5c0c2c5dd7852a.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcqxmlomqij20sg0lvdkk.jpg)

Eureka 的数据存储分了两层：缓存层和数据存储层

Eureka Client 在拉取服务信息时，先从缓存层获取，如果获取不到，先把数据存储层的数据加载到缓存中，再从缓存中获取。值得注意的是，数据存储层的数据结构是服务信息，而缓存中保存的是经过处理加工过的、可以直接传输到 Eureka Client 的数据结构。

#### 缓存层Map解析

第一层的是一个不会过期的ConcurrentHashMap,由定时任务将二级缓存的内容同步到一级缓存(包含删除以及更新缓存)

第二层是一个基于guava的缓存,会失效,并自动加载

#### 整体的缓存更新策略

![2dcf60f2fa8ec888f2b3262e6e9de5c3.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcqy5cjf18j20sg0cq0uc.jpg)



## ZooKeepre与Eureka对比

- ZK 的设计原则是 CP，即强一致性和分区容错性。他保证数据的强一致性，但舍弃了可用性，**如果出现网络问题可能会影响 ZK 的选举，导致 ZK 注册中心的不可用**。
- Eureka 的设计原则是 AP，即可用性和分区容错性。他保证了注册中心的可用性，但舍弃了数据一致性，**各节点上的数据有可能是不一致的（会最终一致）**。
- ZK 是将服务信息保存在树形节点上,Eureka是采取`ConcurrentHashMap<String,Map<String,Lease<InstanceInfo>>>`





## 注册中心

pom.xml配置

```xml
<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Finchley.SR1</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
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

配置属性

```properties
#设置当前环境,一般不是放这里的
spring.profiles.active=dev

#设置eureka的环境
eureka.environment=dev

#服务注册中心端口号
server.port=8889

#服务注册中心实例的主机名
eureka.instance.hostname=localhost

#命令服务名
spring.application.name=helloServer

#是否向服务注册中心注册自己(实现高可用的注册中心时可注册其它注册中心的地址),默认为true
#eureka.client.register-with-eureka=false

#是否检索服务
#eureka.client.fetch-registry=false

#服务注册中心的配置内容，指定服务注册中心的位置(实现高可用注册中心时会用到)
#eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

//使用`EnableEurekaServer`注解来启动一个服务注册中心
@EnableEurekaServer
@SpringBootApplication
public class Eureka2Application {

	public static void main(String[] args) {
		SpringApplication.run(Eureka2Application.class, args);
	}
}
```

## 服务注册

> pom.xml基本如上

启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

//使用`EnableEurekaClient`注解来注册client服务
@EnableEurekaClient
@SpringBootApplication
public class EurekarRegisterApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekarRegisterApplication.class, args);
	}
}
```

配置文件

```properties
server.port=7777

#设置当前环境,一般不是放这里的
spring.profiles.active=dev

#设置eureka的环境
eureka.environment=dev

#服务名
spring.application.name=helloClient

#注册中心地址
eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka

```

>  注册完服务后后续在其它的consume服务会直接通过类似以下方式调用`restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();`

## 服务消费者

服务发现的任务由Eureka的客户端完成，而服务消费的任务由Ribbon完成。Ribbon是一个基于HTTP和TCP的客户端负载均衡器，它可以在通过客户端中配置的ribbonServerList服务端列表去轮询访问以达到均衡负载的作用。

>  pom.xml的配置如上

启动类

```java
//使用`EnableDiscoveryClient`注解来发现服务
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumeApplication {

	@Bean
	@LoadBalanced  //开启客户端负载均衡
	RestTemplate restTemplate(){
		return  new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumeApplication.class, args);
	}
}
```

配置文件

```properties
server.port=9991

#服务名
spring.application.name=helloConsume
#服务方需向集群中的所有注册中心注册
eureka.client.serviceUrl.defaultZone=http://localhost:8888/eureka
```

调用代码

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;


@RestController
public class ConsumeController {
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/ribbon-consume")
    public String helloConsume(){
        //重点代码在此,通过服务名直接调用
        return restTemplate.getForEntity("http://helloClient/hello",String.class).getBody();
    }
}

```

## 常见治理配置参数

```properties
#服务续约任务的调用时间间隔,默认为30s
eureka.instance.lease-renewal-interval-in-seconds=30
#定义服务失效时间,默认为90s
eureka.instance.lease-expiration-duration-in-seconds=90

#服务消费者的服务缓存更新时间,默认为30s
eureka.client.registry-fetch-interval-seconds=30

#关闭注册中心的保护机制,以确保注册中心可以将不可用的实例正确剔除。
eureka.server.enable-self-preservation=false
```



