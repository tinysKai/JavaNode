## Dubbo学习笔记

### 定义



### 原理



### 架构

![QQ截图20200321125641.png](http://ww1.sinaimg.cn/large/8bb38904gy1gd1glk0hrlj20eb08o408.jpg)

> 服务提供者在启动时向注册服务中心注册,服务消费者在启动时向注册者拉取需消费服务的地址列表
>
> 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者.服务提供者和消费者只在启动(服务提供者宕机)时与注册中心交互，注册中心不转发请求，压力较小
>
> 注册中心通过**长连接**感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者
>
> 服务消费者从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
>
> 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。
>
> 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者

#### 架构特点

- 连通性
- 健壮性
- 伸缩性
- 升级性



### XML快速启动

#### 服务方

java代码

````java
public interface DemoService {
    String sayHello(String name);
}

public class DemoServiceImpl implements DemoService {
    public String sayHello(String name) {
        return "Hello " + name;
    }
}
````

配置项

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
 
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />
 
    <!-- 注册中心地址 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
 
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
 
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="#.DemoService" ref="demoService" />
 
    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="#.provider.DemoServiceImpl" />
</beans>
```

启动

```java
public class Provider {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = 
            new ClassPathXmlApplicationContext(new String[] {"http://#/provider.xml"});
        context.start();
        System.in.read(); // 按任意键退出
    }
}
```



#### 消费方

配置项

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
 
    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />
 
    <!-- 注册中心地址 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
 
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <!-- Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认 check="true" -->
    <!-- 可以通过 check="false" 关闭检查，比如，测试时，有些服务不关心，或者出现了循环依赖，必须有一方先启动。 -->
    <dubbo:reference id="demoService" interface="#.DemoService" check="false"/>
</beans>
```

代码调用

```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = 
            new ClassPathXmlApplicationContext(new String[] {"http://#/consumer.xml"});
        context.start();
        DemoService demoService = (DemoService)context.getBean("demoService"); // 获取远程服务代理
        String hello = demoService.sayHello("world"); // 执行远程方法
        System.out.println( hello ); // 显示调用结果
    }
}
```



### 注解快速启动

#### 服务方

```java
@Service
public class AnnotationServiceImpl implements AnnotationService {
    @Override
    public String sayHello(String name) {
        return "annotation: hello, " + name;
    }
}

// 指定Spring扫描路径
@Configuration
@EnableDubbo(scanBasePackages = "#.annotation.impl")
@PropertySource("classpath:/spring/dubbo-provider.properties")
static public class ProviderConfiguration {
       
}
```



配置项

```properties
# dubbo-provider.properties
dubbo.application.name=annotation-provider
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880
```

#### 消费方

```java
@Component
public class AnnotationAction {

    @Reference
    private AnnotationService annotationService;
    
    public String doSayHello(String name) {
        return annotationService.sayHello(name);
    }
}

// 指定Spring扫描路径
@Configuration
@EnableDubbo(scanBasePackages = "#.annotation.action")
@PropertySource("classpath:/spring/dubbo-consumer.properties")
@ComponentScan(value = {"#.annotation.action"})
static public class ConsumerConfiguration {

}
```

调用服务

```java
public static void main(String[] args) throws Exception {
    AnnotationConfigApplicationContext context = 
        new AnnotationConfigApplicationContext(ConsumerConfiguration.class);
    context.start();
    final AnnotationAction annotationAction = (AnnotationAction) context.getBean("annotationAction");
    String hello = annotationAction.doSayHello("world");
}
```

配置项

```properties
# dubbo-consumer.properties
dubbo.application.name=annotation-consumer
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.consumer.timeout=3000
```





### 配置项

#### 不同粒度的配置覆盖关系

- 方法级优先，接口级次之，全局配置再次之。
- 如果级别一样，则消费方优先，提供方次之。

如下的优先级依次降低,每一个下层都被其上层配置覆盖

![https://imgchr.com/i/8WWmr9](http://ww1.sinaimg.cn/large/8bb38904gy1gd1iiahnddj20ic0isteb.jpg)

### 启动时检查

spring配置

```xml
<!-- 关闭某个服务的启动时检查  -->
<dubbo:reference interface="com.foo.BarService" check="false" />

<!-- 关闭所有服务的启动时检查  -->
<dubbo:consumer check="false" />

<!-- 关闭注册中心启动时检查 -->
<dubbo:registry check="false" />
```

properties文件配置

```properties
dubbo.reference.com.foo.BarService.check=false
dubbo.reference.check=false
dubbo.consumer.check=false
dubbo.registry.check=false
```

jvm配置

```
java -Ddubbo.reference.com.foo.BarService.check=false
java -Ddubbo.reference.check=false
java -Ddubbo.consumer.check=false 
java -Ddubbo.registry.check=false
```

#### 配置的含义

`dubbo.reference.check=false`，强制改变所有 reference 的 check 值，就算配置中有声明，也会被覆盖。

`dubbo.consumer.check=false`，是设置 check 的缺省值，如果配置中有显式的声明，如：`<dubbo:reference check="true"/>`，不会受影响。

`dubbo.registry.check=false`，前面两个都是指订阅成功，但提供者列表是否为空是否报错，如果注册订阅失败时，也允许启动，需使用此选项，将在后台定时重试。

### 负载均衡算法

+ Random (随机,默认)
+ RoundRobin (轮询)
+ LeastActive(最少活跃调用)
+ ConsistentHash (**一致性 Hash**，相同参数的请求总是发到同一提供者)

### 线程模型

事件处理模型

![https://imgchr.com/i/8fJNPP](http://ww1.sinaimg.cn/large/8bb38904gy1gd1n86vlbxj20jg03wt8y.jpg)

> 如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。

Dispatcher

- `all` 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
- `direct` 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
- `message` 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- `execution` 只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- `connection` 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。

配置示例

```xml
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```



### 单元测试时的tips

#### 直连服务

为方便测试时的接口单元测试,可直接指定某服务对应的提供方

最常见的是在 JVM 启动参数中加入-D参数映射服务地址 :

> java -Dpackage.XxxService=dubbo://localhost:20890

#### 服务方只订阅

若开发与测试环境共用一个注册中心,则当在开发时,为防止正在修改的服务影响到测试环境可让该服务方只订阅服务不注册服务(因该服务可能需依赖其他服务)

**需要注意此方法需要开发完属性改回来**

> <dubbo:registry address="${ip}:${port}" register="false" />



### Dubbo协议

Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。

反之，Dubbo 缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

![https://imgchr.com/i/8fJNPP](http://ww1.sinaimg.cn/large/8bb38904gy1gd1n86vlbxj20jg03wt8y.jpg)

- Transporter: mina, netty, grizzy
- Serialization: dubbo, hessian2, java, json
- Dispatcher: all, direct, message, execution, connection
- ThreadPool: fixed, cached

特性

- 连接个数：单连接
- 连接方式：长连接
- 传输协议：TCP
- 传输方式：NIO 异步传输
- 序列化：Hessian 二进制序列化
- 适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用 dubbo 协议传输大文件或超大字符串。
- 适用场景：常规远程服务方法调用

### 多协议

Dubbo 允许配置多协议，在不同服务上支持不同协议或者同一服务上同时支持多种协议。不同服务在性能上适用不同协议进行传输，比如大数据用短连接协议，小数据大并发用长连接协议

不同服务不同协议

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd"> 
    <dubbo:application name="world"  />
    <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
    <!-- 多协议配置 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="rmi" port="1099" />
    <!-- 使用dubbo协议暴露服务 -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" protocol="dubbo" />
    <!-- 使用rmi协议暴露服务 -->
    <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" protocol="rmi" /> 
</beans>
```

多协议服务

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="world"  />
    <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
    <!-- 多协议配置 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <dubbo:protocol name="hessian" port="8080" />
    <!-- 使用多个协议暴露服务 -->
    <dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
</beans>
```

### 异步调用

从v2.7.0开始，Dubbo的所有异步编程接口开始以CompletableFuture为基础

Provider端异步执行和Consumer端异步调用是相互独立的，你可以任意正交组合两端配置

- Consumer同步 - Provider同步
- Consumer异步 - Provider同步
- Consumer同步 - Provider异步
- Consumer异步 - Provider异步

**Dubbo 实现同步和异步调用比较关键的一点就在于由谁调用 ResponseFuture 的 get 方法。同步调用模式下，由框架自身调用 ResponseFuture 的 get 方法。异步调用模式下，则由用户调用该方法。**具体参考[服务调用内部解析](http://dubbo.apache.org/zh-cn/docs/source_code_guide/service-invoking-process.html)

#### 服务定义

接口声明

```java
public interface AsyncService {
    //注意这里返回的是CompletableFuture
    CompletableFuture<String> sayHello(String name);
}
```

配置项

```xml
<dubbo:reference id="asyncService" timeout="3000" interface="#.AsyncService"/>
```

#### 消费方

```java
// 调用直接返回CompletableFuture
CompletableFuture<String> future = asyncService.sayHello("async call request");
// 增加回调
future.whenComplete((v, t) -> {
    if (t != null) {
        t.printStackTrace();
    } else {
        System.out.println("Response: " + v);
    }
});
// 早于结果输出
System.out.println("Executed before response return.");
```

#### 消费方异步调用同步的服务方

```xml
<dubbo:reference id="asyncService" interface="#.AsyncService">
      <!-- sent="true" 等待消息发出，消息发送失败将抛出异常 -->
      <dubbo:method name="sayHello" async="true" send="true"/>
</dubbo:reference>
```

调用代码

```java
// 此调用会立即返回null
asyncService.sayHello("world");
// 拿到调用的Future引用，当结果返回后，会被通知和设置到此Future
CompletableFuture<String> helloFuture = RpcContext.getContext().getCompletableFuture();
// 为Future添加回调
helloFuture.whenComplete((retValue, exception) -> {
    if (exception == null) {
        System.out.println(retValue);
    } else {
        exception.printStackTrace();
    }
});

//或使用以下方式调用
CompletableFuture<String> future = RpcContext.getContext().asyncCall(
    () -> {
        asyncService.sayHello("oneway call request1");
    }
);

future.get();
```

#### 服务方异步调用

Provider端异步执行将阻塞的业务从Dubbo内部线程池切换到业务自定义线程，避免Dubbo线程池的过度占用，有助于避免不同服务间的互相影响。

```java
// 接口定义
public interface AsyncService {
    CompletableFuture<String> sayHello(String name);
}

//接口实现
public class AsyncServiceImpl implements AsyncService {
    @Override
    public CompletableFuture<String> sayHello(String name) {
        RpcContext savedContext = RpcContext.getContext();
        // 建议为supplyAsync提供自定义线程池，避免使用JDK公用线程池
        //通过return CompletableFuture.supplyAsync()，业务执行已从Dubbo线程切换到业务线程，避免了对Dubbo线程池的阻塞。
        return CompletableFuture.supplyAsync(() -> {
            System.out.println(savedContext.getAttachment("consumer-key1"));
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "async response from provider.";
        });
    }
}
```

### 事件通知

在调用之前、调用之后、出现异常时，会触发 `oninvoke`、`onreturn`、`onthrow` 三个事件，可以配置当事件发生时，通知哪个类的哪个方法

#### 服务方

```java
interface IDemoService {
    public Person get(int id);
}

class NormalDemoService implements IDemoService {
    public Person get(int id) {
        return new Person(id, "tinys", 4);
    }
}
```

```xml
<dubbo:application name="rpc-callback-demo" />
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<bean id="demoService" class="#.NormalDemoService" />
<dubbo:service interface="#.IDemoService" ref="demoService" version="1.0.0" group="cn"/>
```

#### 消费方

```java
//通知接口
interface Notify {
    public void onreturn(Person msg, Integer id);
    public void onthrow(Throwable ex, Integer id);
}

class NotifyImpl implements Notify {
    public Map<Integer, Person>    ret    = new HashMap<Integer, Person>();
    public Map<Integer, Throwable> errors = new HashMap<Integer, Throwable>();
    
    public void onreturn(Person msg, Integer id) {
        System.out.println("onreturn:" + msg);
        ret.put(id, msg);
    }
    
    public void onthrow(Throwable ex, Integer id) {
        errors.put(id, ex);
    }
}
```

```xml
<bean id ="demoCallback" class = "#.NofifyImpl" />
<dubbo:reference id="demoService" interface="#.IDemoService" version="1.0.0" group="cn" >
       <!--配置服务的回调方法-->
      <dubbo:method name="get" async="true" onreturn = "demoCallback.onreturn" onthrow="demoCallback.onthrow" />
</dubbo:reference>
```

### 服务降级(调用失败时的Mock行为)

配置项

```xml
<dubbo:reference interface="com.foo.BarService" mock="com.foo.BarServiceMock" />
```

```java
package com.foo;
public class BarServiceMock implements BarService {
    public String sayHello(String name) {
        // 你可以伪造容错数据，此方法只在出现RpcException时被执行
        return "容错数据";
    }
}
```

如果服务的消费方经常需要 try-catch 捕获异常，如下所示请考虑改为 Mock 实现，并在 Mock 实现中 return null。

```java
Offer offer = null;
try {
    offer = offerService.findOffer(offerId);
} catch (RpcException e) {
   logger.error(e);
}
```

### 并发控制

`executes`控制服务端的并发量,`actives`控制客户端的并发量

```xml
<!-- 限制 com.foo.BarService 的每个方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个 -->
<dubbo:service interface="com.foo.BarService" executes="10" />

<!--限制 com.foo.BarService 的每个方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个 -->
<dubbo:service interface="com.foo.BarService" actives="10" />
```

### 推荐用法

#### 在 Provider 端尽量多配置 Consumer 端属性

建议在 Provider 端配置的 Consumer 端属性有：

1. `timeout`：方法调用的超时时间
2. `retries`：失败重试次数，缺省是 2 [[2\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn2)
3. `loadbalance`：负载均衡算法 [[3\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn3)，缺省是随机 `random`。还可以配置轮询 `roundrobin`、最不活跃优先 [[4\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn4) `leastactive` 和一致性哈希 `consistenthash` 等
4. `actives`：消费者端的最大并发调用限制，即当 Consumer 对一个服务的并发调用到上限后，新调用会阻塞直到超时，在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

#### 在 Provider 端配置合理的 Provider 端属性

建议在 Provider 端配置的 Provider 端属性有：

1. `threads`：服务线程池大小
2. `executes`：一个服务提供者并行执行请求上限，即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

### 参考文章

[原理解析](http://dubbo.apache.org/zh-cn/docs/source_code_guide/service-invoking-process.html)

[快速启动](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)