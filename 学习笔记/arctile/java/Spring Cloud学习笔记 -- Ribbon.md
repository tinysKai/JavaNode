# Spring Cloud学习笔记 -- Ribbon

## 定义

Spring Cloud Ribbon是一个基于HTTP和TCP的**客户端负载均衡工具**，它基于NetflixRibbon实现。

## 负载均衡策略

+ 随机
+ 轮询
+ 基于轮询的重试策略
+ 权重
+ 选最空闲服务策略(并发请求数最少的那个)

## Spring Cloud默认的Ribbon配置

| Bean Type                  | Bean Name                 | Class Name                       |
| -------------------------- | ------------------------- | -------------------------------- |
| `IClientConfig`            | `ribbonClientConfig`      | `DefaultClientConfigImpl`        |
| `IRule`                    | `ribbonRule`              | `ZoneAvoidanceRule`              |
| `IPing`                    | `ribbonPing`              | `DummyPing`                      |
| `ServerList<Server>`       | `ribbonServerList`        | `ConfigurationBasedServerList`   |
| `ServerListFilter<Server>` | `ribbonServerListFilter`  | `ZonePreferenceServerListFilter` |
| `ILoadBalancer`            | `ribbonLoadBalancer`      | `ZoneAwareLoadBalancer`          |
| `ServerListUpdater`        | `ribbonServerListUpdater` | `PollingServerListUpdater`       |

若想覆盖其中的个别组件,可使用如下方式

```java
@Configuration
protected static class FooConfiguration {

	@Bean
	public ZonePreferenceServerListFilter serverListFilter() {
		ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
		filter.setZone("myTestZone");
		return filter;
	}

	@Bean
	public IPing ribbonPing() {
		return new PingUrl();
	}

}
```

## 使用属性配置来设置Ribbon

- `<clientName>.ribbon.NFLoadBalancerClassName`: Should implement `ILoadBalancer`
- `<clientName>.ribbon.NFLoadBalancerRuleClassName`: Should implement `IRule`
- `<clientName>.ribbon.NFLoadBalancerPingClassName`: Should implement `IPing`
- `<clientName>.ribbon.NIWSServerListClassName`: Should implement `ServerList`
- `<clientName>.ribbon.NIWSServerListFilterClassName`: Should implement `ServerListFilter`

如若你想为服务users设置IRule,则如下

```yml
users:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```

## 配合Eureka时的默认Ribbon配置

| Bean Type                  | Bean Name                 | Class Name                       |
| -------------------------- | ------------------------- | -------------------------------- |
| `IClientConfig`            | `ribbonClientConfig`      | `DefaultClientConfigImpl`        |
| `IRule`                    | `ribbonRule`              | `ZoneAvoidanceRule`              |
| `IPing`                    | `ribbonPing`              | `NIWSDiscoveryPing`              |
| `ServerList<Server>`       | `ribbonServerList`        | `DiscoveryEnabledNIWSServerList` |
| `ServerListFilter<Server>` | `ribbonServerListFilter`  | `ZonePreferenceServerListFilter` |
| `ILoadBalancer`            | `ribbonLoadBalancer`      | `ZoneAwareLoadBalancer`          |
| `ServerListUpdater`        | `ribbonServerListUpdater` | `PollingServerListUpdater`       |

使用方式为直接在`RestTemplate`上使用注解`@LoadBalanced`

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumeApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return  new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumeApplication.class, args);
	}
}
```





## Ribbon中禁用Eureka

```yaml
ribbon:
  eureka:
   enabled: false
```



## 基本原理

![QQ截图20200312232346.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcrk5apvu9j20ku080mxh.jpg)