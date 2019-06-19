# Spring实战读书笔记



### Spring之旅

为了降低Java开发的复杂性，Spring采取了以下4种关键策略：

- ​      基于POJO的轻量级和最小侵入性编程
- ​      通过依赖注入和面向接口实现松耦合
- ​      基于切面和惯例进行声明式编程
- ​      通过切面和模板减少样板式代码



**spring容器**

spring容器并不是只有一个,spring自带了多个容器的实现,可以归类为两种 :

- bean工厂(由org.springframework. beans.factory.eanFactory接口定义)
- 应用上下文（由org.springframework.context.ApplicationContext 接口定义）

bean工厂太简单低级了,一般情况下都会使用**应用上下文**.



**常见的应用上下文**

- AnnotationConfigApplicationContext：从一个或多个基于Java的配置类中加载Spring应用上下文。
- AnnotationConfigWebApplicationContext：从一个或多个基于Java的配置类中加载Spring Web应用上下文。
- ClassPathXmlApplicationContext：从类路径下的一个或多个XML配置文件中加载上下文定义，把应用上下文的定义文件作为类资源。
- FileSystemXmlapplicationcontext：从文件系统下的一个或多个XML配置文件中加载上下文定义。
- XmlWebApplicationContext：从Web应用下的一个或多个XML配置文件中加载上下文定义。



**bean装载到Spring应用上下文中的一个典型的生命周期过程**

1．Spring对bean进行实例化；  
2．Spring将值和bean的引用注入到bean对应的属性中；  
3．如果bean实现了BeanNameAware接口，Spring将bean的ID传递给setBean-Name()方法；  
4．如果bean实现了BeanFactoryAware接口，Spring将调用setBeanFactory()方法，将BeanFactory容器实例传入；  
5．如果bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext()方法，将bean所在的应用上下文的引用传入进来；  
6．如果bean实现了BeanPostProcessor接口，Spring将调用它们的post-ProcessBeforeInitialization()方法；  
7．如果bean实现了InitializingBean接口，Spring将调用它们的after-PropertiesSet()方法。类似地，如果bean使用init-method声明了初始化方法，该方法也会被调用；  
8．如果bean实现了BeanPostProcessor接口，Spring将调用它们的post-ProcessAfterInitialization()方法；  
9．此时，bean已经准备就绪，可以被应用程序使用了，它们将一直驻留在应用上下文中，直到该应用上下文被销毁；  
10．如果bean实现了DisposableBean接口，Spring将调用它的destroy()接口方法。同样，如果bean使用destroy-method声明了销毁方法，该方法也会被调用。  


**spring框架结构**

![spring](https://github.com/tinysKai/JavaNode/blob/master/image/article/2018/0709/spring090901.png)




### 装配bean

**spring的主要装配机制**

- 在XML中进行显式配置
- 在Java中进行显式配置
- 隐式的bean发现机制和自动装配



**通过@ComponentScan 来启用组件扫描**

```java
@Configuration
@ComponentScan
public class ScanConfig{
    //如果没有其他配置的话，@ComponentScan 默认会扫描与配置类相同的包。
}

```



**设置组件扫描的基础包**

```java
@Configuration
@ComponentScan("packageName")
public class ScanConfigByPackageName{
}

@Configuration
@ComponentScan(basePackages={"packageName1","packageName2"})
public class ScanConfigByPackageNames{
    //此方式设置的基础包是以String 类型表示的,这种方法是类型不安全的,在代码重构时可能会有问题
}

@Configuration
@ComponentScan(basePackageClasses={Clazz1.class,Clazz2.class})
public class ScanConfigByPackageClass{
    //这些类所在的包将会作为组件扫描的基础包
}

```



**使用JavaConfig来显式配置spring**

```java
@Configuration
public class JavaConfig{
    @Bean
    public Keyboard keyboard(){
        //@Bean注解会告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文中的bean。
        //方法体中包含了最终产生bean实例的逻辑。
        //默认情况下，bean的ID与带有@Bean 注解的方法名是一样的,本例中bean的名字为keyboard
        return new Keyboard();
    }
    
    //依赖注入bean
    @Bean
    public Computer computer(){
        //构造方法里调用本JavaConfig的其它bean方法
        return new Computer(keyboard());
    }
    
    //另一种依赖注入bean的方式,推荐这种
    @Bean
    public Computer anotherComputert(Keyboard keyboard){
        return new Computer(keyboard);
    }
    
    //使用setter方式的注入
    @Bean
    public Computer setComputer(Keyboard keyboard){
        Computer computer = new Computer(keyboard);
        computer.setKeyBoard(keyboard);
        return computer;
    }
    
}

```



**JavaConfig的引用**

若JavaConfig中的bean数量非常多,我们需将其拆分多多个Config.而多个Config之间是通过`@Import`来关联的

```java
@Configuration
public class KeyBoardConfig{
    @Bean
    public Keyboard keyboard(){
        return new Keyboard();
    }
}

//使用Import注解来引入其它配置类
@Configuration
@Import(KeyBoardConfig.class) 
public class ComputerConfig{
     @Bean
    public Computer anotherComputert(Keyboard keyboard){
        return new Computer(keyboard);
    }
}

//更好的配置是使用一个更高级的配置类将其下的配置类组合在一起,这样就没必要在ComputerConfig中使用@Import
@Configuration
@Import(KeyBoardConfig.class,ComputerConfig.class) 
public class BaseConfig{
}

//如果多个bean的声明不是使用相同的方式,比如Keyboard使用XML,Computer使用java配置,
//则得使用@ImportResource注解
@Configuration
@Import(ComputerConfig.class) 
@ImportResource("classpath:application.xml") //声明bean的xml文件路径
public class BaseConfig{
}

```



### 高级装备

#### Profile

**环境与 profile**

在3.1版本中，Spring引入了bean profile的功能。要使用profile，你首先要将所有不同的bean定义整理到一个或多个profile之中，在将应用部署到每个环境时，要确保对应的profile处于激活（active）的状态。

在Java配置中，可以使用@Profile 注解指定某个bean属于哪一个profile。



**激活profile**

Spring在确定哪个profile处于激活状态时，需要依赖两个独立的属性：

- spring.profiles.active
- spring.profiles.default

如果设置了spring.profiles.active 属性的话，那么它的值就会用来确定哪个profile是激活的。

但如果没有设置spring.profiles.active 属性的话，那Spring将会查找spring.profiles.default 的值。

如果spring.profiles.active 和spring.profiles.default 均没有设置的话，那就没有激活的profile，因此只会创建那些没有定义在profile中的bean。



在单元测试上,可以使用@ActiveProfiles 注解设置.  



#### 条件化的Bean

使用@Conditional注解,如果给定的条件计算结果为true,就会创建这个bean，否则的话，这个bean会被忽略。

```java
@Bean
@Conditional(MySQLUrlExistCondition.class)
public class MysqlBean{
   //只有当上面注解条件成立时才实例化这个bean,上面的MySQLUrlExistCondition是一个实现了Condition接口的实现
   return new Mysql();   
}    

public interface Condition{
    boolean match(ConditionContext ctxt,AnnotatedTypeMetadata metadata);
}

public class MySQLUrlExistCondition implements Condition{
    public boolean match(ConditionContext context,AnnotatedTypeMetadata metadata){
        //判断是否存在MYSQL相关的环境变量
        Environment env = context.getEnvironment();
        return env.containsProperty("MYSQL_URL");
    }
}
```



Condition接口match方法参数解释

```java
/**
 借助getRegistry() 返回的BeanDefinitionRegistry 检查bean定义；
 借助getBeanFactory() 返回的ConfigurableListableBeanFactory 检查bean是否存在，甚至探查bean的属性
 借助getEnvironment() 返回的Environment 检查环境变量是否存在以及它的值是什么；
 读取并探查getResourceLoader() 返回的ResourceLoader所加载的资源；
 借助getClassLoader() 返回的ClassLoader 加载并检查类是否存在。
*/
public interface ConditionContext {
    BeanDefinitionRegistry getRegistry();

    ConfigurableListableBeanFactory getBeanFactory();

    Environment getEnvironment();

    ResourceLoader getResourceLoader();

    ClassLoader getClassLoader();
}

/**
  AnnotatedTypeMetadata则能够让我们检查带有@Bean注解的方法上还有什么其他的注解
*/
public interface AnnotatedTypeMetadata {
    boolean isAnnotated(String var1);

    @Nullable
    Map<String, Object> getAnnotationAttributes(String var1);

    @Nullable
    Map<String, Object> getAnnotationAttributes(String var1, boolean var2);

    @Nullable
    MultiValueMap<String, Object> getAllAnnotationAttributes(String var1);

    @Nullable
    MultiValueMap<String, Object> getAllAnnotationAttributes(String var1, boolean var2);
}
```



#### 处理自动装配的歧义性

- @Qualifier 
- @Primary

使用@Primary注解来解决@Autowired下有多个实现的情况,在遇到歧义时,spring会优先选择标有@Primary注解的实现.

使用@Qualifier直接指明注入bean的ID,但注意这里属性时字符串,如果是使用默认声明bean名字的方式,将来重构修改了类名,那么可能会引起匹配失败问题. 



#### bean的作用域

**Spring的作用域**

- 单例
- 原型(Prototype)
- 会话(session)
- 请求(request)

```java
/**
 * 一个使用session作用域的java配置例子
   当会话域或者请求域对象注入于单例对象时,因单例对象会在spring应用上下文加载时创建,
   而创建时不管是会话域的对象或者请求域对象都是还没创建的,为了解决这个问题,spring使用了代理的模式,
   在创建的时候注入一个代理对象,下例是使用了基于接口代的理的方式注入
 */
@Component
@Scope(value=WebApplicationContext.SCOPE_SESSION,ScopedProxyMode.INTERFACES)
public ShoppingCart cart(){
    return new ShoppingCart();
}
```

```xml
//使用xml的方式配置session作用域
<bean id="card" class="##" scope="session">
    <aop:scope-proxy />
</bean>
//默认情况下是使用cglib创建的代理对象,若想使用基于接口的代理则可使用proxy-target-class设置为false
<bean id="card" class="##" scope="session">
    <aop:scope-proxy proxy-target-class="false"/>
</bean>

//注 : 为使用<aop:scope-proxy />我们必须在XML配置中声明aop命名空间
```



#### 运行时注入

**注入外部的值**

在Spring中，处理外部值的最简单方式就是声明属性源并通过Spring的Environment 来检索属性。

```java
/**
* 一个使用环境变量例子
*/
@Configuration
@PropertySource("classpath:app.properties") //声明属性来源
public class MysqlConfig{
    @Autowired
    Environment env;
    
    @Bean
    public Mysql mysql(){
        return new Mysql(
        	env.getProperty("mysql.url"),
            env.getProperty("mysql.username"),
            env.getProperty("mysql.password"),
        );
    }
}
```



### 切面AOP

#### 面向切面

Spring提供了4种类型的AOP支持

- 基于代理的经典Spring AOP(笨重,复杂,现基本不用)
- 纯POJO切面
- @AspectJ注解驱动的切面
- 注入式AspectJ切面（适用于Spring各版本）

前三种都是Spring AOP实现的变体，Spring AOP构建在动态代理基础之上，

因此，Spring对AOP的支持局限于方法拦截。



 **Spring支持的Aspect指示器**

| AspectJ指示器 | 描　　述                                                     |
| ------------- | :----------------------------------------------------------- |
| arg()         | 限制连接点匹配参数为指定类型的执行方法                       |
| @args()       | 限制连接点匹配参数由指定注解标注的执行方法                   |
| execution()   | 用于匹配是连接点的执行方法                                   |
| this()        | 限制连接点匹配AOP代理的bean引用为指定类型的类                |
| target        | 限制连接点匹配目标对象为指定类型的类                         |
| @target()     | 限制连接点匹配特定的执行对象，这些对象对应的类要具有指定类型的注解 |
| within()      | 限制连接点匹配指定的类型                                     |
| @within()     | 限制连接点匹配指定注解所标注的类型                           |
| @annotation   | 限定匹配带有指定注解的连接点                                 |

除了所列的指示器外，Spring还引入了一个新的bean() 指示器，它允许我们在切点表达式中使用bean的ID来标识bean。bean() 使用bean ID或bean名称作为参数来限制切点只匹配特定的bean。

```
//表示执行某方法时执行通知,但限制bean的id
execution(* package.interface.method()) and bean("beanId")
```



一个使用注解的切面例子

```java
/**
*  使用Aspect注解声明这是一个切面
   然后使用@Before,@AfterReturning等注解声明切点,在切点前后执行切入
*/
@Aspect  
public class Audience {
  @Before("execution(** package.interface.method(..))")  //方法开始之前执行
  public void takeSeats() { 
    System.out.println("The audience is taking their seats.");
  }

  @Before("execution(** package.interface.method(..))")  //方法开始之前执行  
  public void turnOffCellPhones() { 
    System.out.println("The audience is turning off their cellphones");
  }

  @AfterReturning("execution(** package.interface.method(..))")  //方法结束之后执行
  public void applaud() { 
    System.out.println("CLAP CLAP CLAP CLAP CLAP");
  }

  @AfterThrowing("execution(** package.interface.method(..))")  //方法执行出错时执行
  public void demandRefund() { 
    System.out.println("Boo! We want our money back!");
  }
    
  /**
   * 上面的切点都是使用相同的路径,重复定义了,spring提供了切点来定义路径
  */
   @Pointcut("execution(** package.interface.method(..))") //定义命令的切点
   public  void performance(){}
    
   @AfterReturning("performance()") //使用上面定义的切点 
    public void home(){
        System.out.println("Go Home!");
    } 
}

//使用切面还需启用AspectJ自动代理
//如果使用xml装载bean的话,则需在xml添加"<aop:aspectj-autoproxy>"
//假设你使用java配置方式装载bean,则如下
@Configuration
@EnableAspectJAutoProxy //声明启动自动代理
@ComponentScan
public class ConcertConfig{
   @Bean
    public Audience audience(){
        return new Audience();
    } 
}
```



使用环绕通知重写上面的切面

```java
@Aspect  
public class Audience {
   @Pointcut("execution(** package.interface.method(..))") //定义命令的切点
   public  void performance(){}
    
    @Around("performance()")
    public void watchPerformance(ProceedingJoinPoint pjp){
        try{
            System.out.println("The audience is turning off their cellphones");
            pjp.proceed();
            System.out.println("CLAP CLAP CLAP CLAP CLAP");
        }catch(Exception e){
        	System.out.println("Boo! We want our money back!");
        }    
    } 
}    
```



XML配置切面

| aop配置元素             | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| \<aop:advisor\>         | 定义aop通知器                                                |
| \<aop:after>            | 定义aop后置通知(不管被通知的方法是否执行成功)                |
| \<aop:after-returning>  | 定义aop after-returning通知                                  |
| \<aop:after-throwing>   | 定义aop after-throwing通知                                   |
| \<aop:around>           | 定义aop环绕通知                                              |
| \<aop:aspect>           | 定义切面                                                     |
| \<aop:aspect-autoproxy> | 启动@AspectJ注解驱动的切面                                   |
| \<aop:before>           | 定义aop前置通知                                              |
| \<aop:config>           | 顶层的aop配置元素，大多数的\<aop:*>元素必须包含在\<aop:config>内 |
| \<aop:declare-parents>  | 为被通知的 对象引入的额外的接口，并透明地实现                |
| \<aop:pointcut>         | 定义切点                                                     |

```xml
<aop:config>
   <aop:aspect ref="audience">
     <aop:pointcut id="performance" expression="execution(** concert.Performance.perform(..))"/>
         <aop:before pointcut-ref="performance" method="takeSeats"/>

          <aop:before pointcut-ref="performance" method="applause"/>

          <aop:before pointcut-ref="performance" method="demandRefund"/>
   </aop:aspect>
</aop:config>
```

