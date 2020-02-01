## 使用`factory-method`来实现弱依赖创建Reddison

#### XML声明

```xml
<bean id="redissonClient" class="#.redisson.RedissonFactory" factory-method="create">
    <constructor-arg name="config" ref="redissonConfig"/>
    <constructor-arg name="checkInitialDelay" value="15"/>
    <constructor-arg name="checkPeriod" value="5"/>
</bean>
```

#### Java代码

```java
import #.NamedThreadFactory;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;

import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * redisson factory 避免启动强依赖,导致应用不能启动,并自动检测redis恢复,恢复后自动创建client并重新register到spring
 */
public class RedissonFactory {
    private static final Logger LOGGER = LoggerFactory.getLogger(RedissonFactory.class);
    private static final ScheduledExecutorService scheduledExecutorService =  new ScheduledThreadPoolExecutor(1, new NamedThreadFactory("redisson-initial-check", true));

    /**
     * Create sync/async Redisson instance with provided config
     */
    public static RedissonClient create(Config config, Long checkInitialDelay, Long checkPeriod) {
        RedissonClient redissonClient = null;
        try {
            redissonClient = Redisson.create(config);
        } catch (Exception e) {
            LOGGER.error("create redisson exception.", e);
            //创建异常的话创建一个定时器来定时检测重新创建
            scheduledExecutorService.scheduleWithFixedDelay(new Checker(config), checkInitialDelay, checkPeriod, TimeUnit.SECONDS);
        }
        return redissonClient;
    }

    final static class Checker implements Runnable {

        private Config config;

        public Checker(Config config) {
            this.config = config;
        }
        @Override
        public void run() {
            LOGGER.debug("Checking Redisson register.");
            try {
                RedissonClient redissonClient = SpringContextHolder.getBean(RedissonClient.class);
                if (redissonClient == null) {
                    createAndRegister();
                } else {
                    LOGGER.info("Redisson initial success, shutdown checker scheduler.");
                    scheduledExecutorService.shutdown();
                }
            } catch (NoSuchBeanDefinitionException e) {
                LOGGER.error("找不到bean", e);
                createAndRegister();
            } catch (Exception e) {
                LOGGER.error("", e);
            }
        }

        /**
         * 尝试创建redisson client并注册到spring
         */
        private void createAndRegister() {
            DefaultListableBeanFactory fty = (DefaultListableBeanFactory) SpringContextHolder.getApplicationContext().getAutowireCapableBeanFactory();
            BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Redisson.class);
            beanDefinitionBuilder.setFactoryMethod("create");
            beanDefinitionBuilder.addConstructorArgValue(config);
            fty.registerBeanDefinition("redissonClient", beanDefinitionBuilder.getBeanDefinition());
        }
    }

}

```

```java
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 定义一个ThreadFactory
*/
public class NamedThreadFactory implements ThreadFactory{
	private static final AtomicInteger POOL_SEQ = new AtomicInteger(1);

	private final AtomicInteger mThreadNum = new AtomicInteger(1);

	private final String mPrefix;

	private final boolean mDaemo;

	private final ThreadGroup mGroup;

	public NamedThreadFactory()
	{
		this("pool-" + POOL_SEQ.getAndIncrement(),false);
	}

	public NamedThreadFactory(String prefix)
	{
		this(prefix,false);
	}

	public NamedThreadFactory(String prefix,boolean daemo)
	{
		mPrefix = prefix + "-thread-";
		mDaemo = daemo;
        SecurityManager s = System.getSecurityManager();
        mGroup = ( s == null ) ? Thread.currentThread().getThreadGroup() : s.getThreadGroup();
	}

	public Thread newThread(Runnable runnable)
	{
		String name = mPrefix + mThreadNum.getAndIncrement();
        Thread ret = new Thread(mGroup,runnable,name,0);
        ret.setDaemon(mDaemo);
        return ret;
	}

	public ThreadGroup getThreadGroup()
	{
		return mGroup;
	}
}

```



#### 总结

本篇的重点在于如何使用java代码向spring注册bean.