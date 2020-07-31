## DUBBO单元测试因配置错误而耗费了很久时间记录

### Dubbo服务代码单元测试

#### 背景

因需测试core包下的MQ通知,所以加上了所有的包,测试完毕后开始测试dubbo接口时发现调用失败.

#### 调用测试类

```java
@SpringBootApplication(exclude = {
        ShardingsDataSourceAutoConfiguration.class,
        MongoAutoConfiguration.class,
        MongoDataAutoConfiguration.class,
        DataSourceAutoConfiguration.class,
        HibernateJpaAutoConfiguration.class,
        DataSourceTransactionManagerAutoConfiguration.class
        },
        scanBasePackages = {"#.unifiedpay.dubbo.test"
        //这里不小心添加了全部的类,导致启动的时候也加载了全部的类,所以全部东东都需要加载,配置    
        //scanBasePackages = {"#.unifiedpay.dubbo.test","#.unifiedpay","#.common"
        } )
@EnableDubbo
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}

```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = ClientApplication.class)
@Slf4j
public class CashierServiceTest {
    public static final String YJ_101 = "YJ101";
    @Reference(version = "1.0.0", protocol = "dubbo", registry = "yyp-reg-consul")
    private CashierService cashierService;

     @Test
    public void cashierUrlTest(){}
}    
```



### 分析

dubbo服务单元测试时只是模拟一个dubbo的consumer端即可,所以只需引入test包,对应的配置也只需引入consume端所依赖的最少配置即可

### 总结

`@SpringBootTest`的classes属性`classes`可指定加载的上下文

### 单元测试整个服务的其它spring类

需配置整个服务的相关属性依赖,将启动的类所依赖的东东都配置上

#### 调用类

```java
@SpringBootApplication(
        scanBasePackages = {"#.unifiedpay.dubbo.test","#.common","#.unifiedpay"
        } )
@EnableDubbo
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = ClientApplication.class)
@Slf4j
public class PayNotifyHandlerTest {
    @Autowired
    @Qualifier("payNotifyHandler")
    private NotifyHandler notifyHandler;


    @Test
    public void handleMessageTest() throws BusinessException {}
}
```

### 总结

spring单独的测试类可看情况屏蔽相关类



### spring类测试最终版

直接在单元测试类写SpringBootApplication,平常则注释掉,需要测试时才开启

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
/*@SpringBootApplication(
        scanBasePackages = {"#.unifiedpay.dubbo.test","#.common","#.unifiedpay"
        } )*/
@Slf4j
public class PayNotifyHandlerTest {
    @Autowired
    @Qualifier("payNotifyHandler")
    private NotifyHandler notifyHandler;


    @Test
    public void handleMessageTest() throws BusinessException {
        String notifyStr = "{...}";
        PayNotifyBo payNotifyBo = GsonUtils.fromJson(notifyStr, PayNotifyBo.class);
        notifyHandler.handleMessage(payNotifyBo);
    }
}
```

