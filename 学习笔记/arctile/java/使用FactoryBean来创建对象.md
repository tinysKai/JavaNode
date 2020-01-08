## 使用FactoryBean来创建对象

#### 定义具体的创建对象工厂类

```java
import lombok.Setter;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.FactoryBean;

/**
 * RedissonClient创建者
 * 不直接用Redisson的xml标签在spring做初始化的原因是因为需要把RedissonClient作为弱依赖
 */
public class RedissonFactoryBean implements FactoryBean<RedissonClient> {

    private static final Logger log = LoggerFactory.getLogger(RedissonFactoryBean.class);

    @Setter
    private Config redissonConfig;

    private RedissonClient redissonClient;

    //在xml注入的对象是通过此方法的返回值获取的对象的
    public RedissonClient getObject() throws Exception {
        try {
            redissonClient = Redisson.create(redissonConfig);
            log.info("初始化RedissonClient成功");
        } catch (Exception e) {
            log.error("初始化RedissonClient失败：" , e);
        }

        return redissonClient;
    }

    public Class<?> getObjectType() {
        return RedissonClient.class;
    }

    public boolean isSingleton() {
        return true;
    }

    public void close() {
        redissonClient.shutdown();
    }
}

```

#### XML配置

```xml
<!-- redisson cluster config -->
<!-- 
	通过工厂方法的方式来创建对象 以下是其创建对象的方法
 	public static Config fromJSON(String content) throws IOException {
        ConfigSupport support = new ConfigSupport();
        return support.fromJSON(content, Config.class);
    }
	
-->
<bean id="redissonConfig" class="org.redisson.config.Config" factory-method="fromJSON">
    <!--通过`Factory-method`传递参数可灵活构造spring对象-->
    <constructor-arg name="content" value="${redissonConfigFromJSON}"/>
</bean>

<!-- redisson client 通过实现FactoryBean来创建对象-->
<bean id="redissonClient" class="#.RedissonFactoryBean" destroy-method="close">
    <property name="redissonConfig" ref="redissonConfig" />
</bean>
```

#### Property属性文件

```properties
#redisson配置
redissonConfigFromJSON=\
{\
  "clusterServersConfig":{\
     "idleConnectionTimeout":60000,\
     "connectTimeout":300,\
     "timeout":300,\
     "readMode":"MASTER",\
     "subscriptionMode":"MASTER",\
     "retryInterval":100,\
     "nodeAddresses": [\
       "redis://${CLUSTER_REDIS_HOST1}",\
       "redis://${CLUSTER_REDIS_HOST2}",\
       "redis://${CLUSTER_REDIS_HOST3}"\
     ],\
     "slaveConnectionMinimumIdleSize":20,\
     "slaveConnectionPoolSize":100,\
     "masterConnectionMinimumIdleSize":20,\
     "masterConnectionPoolSize":100\
  }\
}
```

#### 总结

本篇文章重点在于学习如何使用FactoryBean的工厂模式来创建对象,以及使用spring的工厂方法`factory-method`来创建对象

