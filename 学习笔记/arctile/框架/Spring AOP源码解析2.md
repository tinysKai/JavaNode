# Spring AOP源码解析

## AOP定义

**基本概念**

+ Aspect : 切面(包括advice以及pointcut)
+ JoinPoint : 连接点(spring aop中，一个连接点通常代表一个方法的调用)
+ Advice : 通知(包括`Before`,`after`等通知)
+ pointcut : 切入点

**整体概念概括**(图片来源于网络)

![https://ws3.sinaimg.cn/large/005BYqpggy1g2ncgjv4j2j30ry0bbjsu.jpg](https://s2.ax1x.com/2019/05/02/ENnmdK.png)



## 创建代理对象的时机

我们先看一张Spring AOP源码中,创建代理关键类的层级图

![https://ws3.sinaimg.cn/large/005BYqpggy1g2n09do8z7j313o0gujsb.jpg](https://s2.ax1x.com/2019/05/02/Etliyd.png)

注意右上方的`BeanPostProcessor`接口

```java
//BeanPostProcessor接口作用是：
//如果我们需要在Spring容器完成Bean的实例化、配置和其他的初始化前后添加一些自己的逻辑处理，
//我们就可以定义一个或者多个BeanPostProcessor接口的实现，然后注册到容器中。
public interface BeanPostProcessor {

	//在实例化及依赖注入完成后、在任何初始化代码（比如配置文件中的init-method）调用之前调用
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	//在初始化代码调用之后调用
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

附Spring bean的实例化过程

![https://ws3.sinaimg.cn/large/005BYqpggy1g2nbedwrkrj30nf0960tz.jpg](https://s2.ax1x.com/2019/05/02/ENAGqO.jpg)

因此,代码的创建时在`初始化代码调用之后`创建的.



##  Spring AOP源码分析

整个创建代理流程分为以下两个子流程 : 

+ 查找拦截器并组装AOP基本组件(Aspect,pointcut等)
+ 创建代理

![https://ws3.sinaimg.cn/large/005BYqpggy1g2nc0les84j30ut0i4abi.jpg](https://s2.ax1x.com/2019/05/02/ENVkHU.png)

#### AOP代码入口分析

**方式**

+ 通过xml的`<aop/>`标签,在对应的aop的jar包中找到`spring.handle`文件中声明的处理类
+ 全局搜索`aspectj-autoproxy`来查找有这个的类

最终我们定位到以下`AopNamespaceHandler`中的`AspectJAutoProxyBeanDefinitionParser`类

```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
         //声明aop
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

```java
class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {

    //初始化一个AOP专用的Bean，并且注册到Spring容器中
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
		extendBeanDefinition(element, parserContext);
		return null;
	}
}	
```

```java
public abstract class AopNamespaceUtils {
 	public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(ParserContext parserContext, Element sourceElement) {
		//把应用了注解@Aspect的bean注册成BeanDefiniton,这里最终会创建一个`AnnotationAwareAspectJAutoProxyCreator`类
		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
		//解析配置元素，决定代理的模式。(JDK动态代理或CGLIB代理)
        useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
        //注册组件并通知监听器
		registerComponentIfNecessary(beanDefinition, parserContext);
	}   
}   
```

以下的代码跳转最终会去创建注册一个叫做`AnnotationAwareAspectJAutoProxyCreator`的类,就是上面提到的关键类.

注意`wrapIfNecessary`方法.

```java
//下面的两个方法都是继承其父类`AbstractAutoProxyCreator`的
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
	
    @Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                 //关键入口方法
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
    
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		//如果已经处理过，直接返回
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
        //无需增强
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
         //如果是基础类比如 advice pointcut advisor 这些基础aop类则直接返回
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
         //子流程1,寻找advice
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
             //子流程2,创建动态代理
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
}    
```



#### 寻找拦截器代码分析

```java
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {	
	@Override
	protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
		//注意`AnnotationAwareAspectJAutoProxyCreator`子类覆盖了父类的这个方法,所以这里需从父类的这个方法看起
         //其实一开始创建的关键类就是`AnnotationAwareAspectJAutoProxyCreator`,父类中的方法其实是通过子类来体现的
        List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
    
    //Find all eligible Advisors for auto-proxying this class.
    //这里是寻找xml配置文件的通知
    protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
}    

public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
	@Override
	protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
         //先调用父类的xml配置文件寻找通知
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
             //添加使用注解的通知
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}
}    
```

下面的代码是在当前的BeanFactory寻找Aspect注解的bean,然后转换为Advise返回.

```java
public class BeanFactoryAspectJAdvisorsBuilder {	
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;
		
        //第一次进入无缓存状态时,则去加载
		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new ArrayList<>();
					aspectNames = new ArrayList<>();
                      //获取所有的bean的名字
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
					for (String beanName : beanNames) {
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
					   //获取bean类型	
                        Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
                          //判断是否是切面
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
								//解析Aspect注解中的拦截器方法(比如 @After @Before @Aroud等)
                                   List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
        
        //后面的请求使用上面缓存的结果返回
		List<Advisor> advisors = new ArrayList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
}    
```

```java
public class ReflectiveAspectJAdvisorFactory extends AbstractAspectJAdvisorFactory
	@Override
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new ArrayList<>();
		// 获取切面拦截器的所有拦截方法
		for (Method method : getAdvisorMethods(aspectClass)) {
			///获取拦截器方法(增强)并包装成advisor
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}

	@Override	
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

        //获取Pointcut表达式
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
		//根据pointcut生成拦截器对象
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}

	private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
		//寻找方法上的通知注解
        AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		if (this.beanFactory != null) {
			ajexp.setBeanFactory(this.beanFactory);
		}
		return ajexp;
	}
}
```

上面一长串的代码主要是做组装`Advise`列表使用,以及其中的多个子步骤 : 

+ 获取切面的所有方法
+ 生成Pointcut对象
+ 生成拦截器对象(InstantiationModelAwarePointcutAdvisorImpl)



#### 生成代理对象

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		//通过`ProxyFactory`来生成代理对象
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
}	
```

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}    
```

生成代理对象有两种模式 :

+ 基于接口的JDK动态代理
+ 基于Cglib的动态代理

**Spring的代理选择规则**

+ 如果是实现了接口
  - 默认配置时,使用JDK的动态代理
  - 如果配置了`<aop:aspectj-autoproxy proxy-target- class=“true”/>`,则强制使用CGLIb动态代理
  - 如果ProxyConfig配置了属性`optimize`为`true`,则也使用Cglib动态代理(Cglib基于ASM生成字节码性能好)
+ 如果无实现接口
  - 使用Cglib动态代理



#### 生成的代理之后的调用关联

`BeanPostProcessor`是容器级别的接口,实现了该接口的类将可以在该容器的bean实例化前后做一些自定义处理并返回新的对象来覆盖spring生成的对象.

对应于本篇文章就是`postProcessAfterInitialization`方法会接收到此容器内所有bean实例化,初始化之后的回调(每个bean调用一次),然后spring aop模块使用自己新生成的代码返回给spring从而覆盖了原来的bean对象.但此时这个对象的beanName以及beanId都没变化,只是实例变化了.下次调用时获取到的bean变成了代理后的bean.

再次附`postProcessAfterInitialization`方法定义

```java
    //java8的方法定义,默认方法是返回本身的bean
    //参数解释 : 
	// 		  bean : 注入的bean对象
	//		  beanName : bean的id或者name(xml中bean标签的id或者是使用注解注入的默认的类名首字母小写转换)
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
```



## 参考链接

<https://blog.csdn.net/c_unclezhang/article/details/78769426>