## ApplicationListener & ApplicationEvent

#### 定义

​    ApplicationContext事件机制是`观察者设计模式`的实现，通过ApplicationEvent类和ApplicationListener接口，可以实现ApplicationContext事件处理。如果容器中有一个ApplicationListener Bean，每当ApplicationContext发布ApplicationEvent时，ApplicationListener Bean将自动被触发。



#### 组成

Spring的事件框架有如下两个重要的成员：

- ApplicationEvent：容器事件，由ApplicationContext发布(继承了`ApplicationEventPublisher`类)
- ApplicationListener：监听器，可由容器中的任何监听器Bean担任

Spring提供如下几个内置事件：

- **ContextRefreshedEvent**：ApplicationContext容器初始化或刷新时触发该事件。此处的初始化是指：所有的Bean被成功装载，后处理Bean被检测并激活，所有Singleton Bean 被预实例化，ApplicationContext容器已就绪可用

- **RequestHandledEvent**：Web相关事件，只能应用于使用DispatcherServlet的Web应用。在使用Spring作为前端的MVC控制器时，当Spring处理用户请求结束后，系统会自动触发该事件。

- ContextStartedEvent：当使用ConfigurableApplicationContext(ApplicationContext的子接口）接口的start()方法启动ApplicationContext容器时触发该事件。容器管理声明周期的Bean实例将获得一个指定的启动信号，这在经常需要停止后重新启动的场合比较常见

- ContextClosedEvent：当使用ConfigurableApplicationContext接口的close()方法关闭ApplicationContext时触发该事件

- ContextStoppedEvent：当使用ConfigurableApplicationContext接口的stop()方法使ApplicationContext容器停止时触发该事件。此处的停止，意味着容器管理生命周期的Bean实例将获得一个指定的停止信号，被停止的Spring容器可再次调用start()方法重新启动

  

#### 使用场景

**常见使用场景**

​	在一些业务场景中，当容器初始化完成之后，需要处理一些操作，比如一些数据的加载、初始化缓存、特定任务的注册等等。这个时候我们就可以使用Spring提供的ApplicationListener来进行操作。

**常规的使用场景**

​	适用于项目内部的`业务解耦`.比如用户注册成功后需要发送邮箱,此时在登记完数据库后可通过事件发布注册用户事件来处理发邮件的功能.类似`guava的eventbus`.

​	跨项目或者大对象可能会堆积引起内存问题才考虑使用MQ.



**注意默认的事件驱动是同步的,可配置为使用线程池的异步模式**

```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
    
    @Override
    public void multicastEvent(final ApplicationEvent event) {
        for (final ApplicationListener listener : getApplicationListeners(event)) {
            Executor executor = getTaskExecutor();
            //如果线程池不为空,则使用线程池模式
            if (executor != null) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        listener.onApplicationEvent(event);
                    }
                });
            }
            else {
                //默认使用同步调用模式
                listener.onApplicationEvent(event);
            }
        }
    }
}    
```





#### 代码例子

**事件定义**

```java
public abstract class ApplicationEvent extends EventObject {
	private final long timestamp;
    
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}

	public final long getTimestamp() {
		return this.timestamp;
	}
}

public class DemoEvent extends ApplicationEvent{

    private String message;

    public DemoEvent(Object source,String message){
        super(source);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
 }

```



**事件发布**

```java
@Component
public class DemoPublisher {
    //容器注入
    @Autowired
    ApplicationContext applicationContext;

    public void publish(String message){
        //发布事件
        applicationContext.publishEvent(new DemoEvent(this, message));
    }
}

```



**事件监听**

```java
//容器初始化完成事件
@Component
public class ServerInitListener implements ApplicationListener<ContextRefreshedEvent> {

	@Override
	public void onApplicationEvent(ContextRefreshedEvent event) {
		//防止重复执行。
         //方式二判断 : event.getApplicationContext().getDisplayName().equals("Root WebApplicationContext")
		if (event.getApplicationContext().getParent() == null) {
			doSomething();
		}
        //如果是自定义的事件类型,则可以使用`event instanceof CustomEvent`来判断
	}
}
```

**注册事件类型为异步事件**

```java
@Service
@Qualifier(value = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
public class ServiceEventMulticaster extends SimpleApplicationEventMulticaster {
    
    public ServiceEventMulticaster() {
        ThreadPoolExecutor tpe = new ThreadPoolExecutor(
                3, 15, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(), new ThreadFactory() {
            int num = 1;
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("Mythread-" + num);
                num++;
                return thread;
            }
        }
        );
        //设置了线程池来异步处理事件
        this.setTaskExecutor(tpe);
    }

    @Override
    public void multicastEvent(ApplicationEvent event) {
        this.getApplicationListeners(event);
    }

    @PreDestroy
    public void beforeDestroy() {
        //关闭事件
        ThreadPoolExecutor executor = (ThreadPoolExecutor) this.getTaskExecutor();
        executor.shutdown();
    }
}

```





#### 源码设计调研

**主要类**

+ AbstractApplicationContext
+ SimpleApplicationEventMulticaster
+ ApplicationListener实现类
+ ApplicationEvent实现类

**代码流程**

1.注册`ApplicationEventMulticaster`

```java
//这个注册就是上面ServiceEventMulticaster能成为异步事件的原因
public abstract class AbstractApplicationContext extends DefaultResourceLoader 	implements ConfigurableApplicationContext, DisposableBean {
    //在refresh方法中来注册多路广播
    protected void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        }
        else {
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
            beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster); 
        }
    }
}
```

2.注册`Listeners`

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext, DisposableBean {
	protected void registerListeners() {
		//感觉这个循环没啥用,主要是依赖下边来添加Listeners
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}
		//根据接口来下来加载Listeners
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String lisName : listenerBeanNames) {
             //添加Listener到Multicaster
			getApplicationEventMulticaster().addApplicationListenerBean(lisName);
		}
	}
}    
```

3.发送事件

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext, DisposableBean {
	@Override
	public void publishEvent(ApplicationEvent event) {
         //将发布的事件通知到监听者
		getApplicationEventMulticaster().multicastEvent(event);
		if (this.parent != null) {
			this.parent.publishEvent(event);
		}
	}	
}    
```



