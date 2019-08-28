## Resilience4j -  熔断器学习笔记

#### 定义

​	一个轻量级的,易用的熔断器

#### Hystrix对比

+ Hystrix调用必须被封装到HystrixCommand里，而resilience4j以装饰器的方式提供对函数式接口、lambda表达式等的嵌套装饰，因此你可以用简洁的方式组合多种高可用机制
+ Hystrix的频次统计采用滑动窗口的方式，而resilience4j采用环状缓冲区的方式
+ 关于熔断器在半开状态时的状态转换，Hystrix仅使用一次执行判定是否进行状态转换，而resilience4j则采用可配置的执行次数与阈值，来决定是否进行状态转换，这种方式提高了熔断机制的稳定性
+ 关于隔离机制，Hystrix提供基于线程池和信号量的隔离，而resilience4j只提供基于信号量的隔离

---------------------
#### 熔断状态流转

在状态环未满时,熔断器的状态是未知的.只有请求次数达到了设置的环的数量才会触发熔断器计算.

当请求数超过了环数且失败率大于配置值时,熔断器状态为`OPEN`,此时会持续一段时间来拒绝请求.

当达到`HALF_OPEN`状态时,熔断器允许少量请求通过并且此时会使用另外一个状态环来计算失败率,若失败率低于或等于阈值则熔断器状态变为`CLOSED`,若高于则熔断器状态为`OPEN`

![](http://ww1.sinaimg.cn/large/8bb38904gy1g5j5yt0mtaj20bp04hab1.jpg)

#### 熔断器是线程安全的

+ 状态是存在于AtomicReference
+ 使用原子操作来更新状态
+ 更新`Ring Bit Buffer`是同步的

**需注意的是熔断器没针对调用做同步控制,所以如果当熔断器关闭时,并发过来的请求是无法控制的,此时所有的请求都会允许被执行.状态环数的大小无法限制最大的失败数.如果你需要严格限制并发数,请使用`BulkHead`.**

#### 调用流程

原理是使用装饰模式,调用方法时先判断熔断状态,执行完毕后处理执行结果更新熔断状态

![](http://ww1.sinaimg.cn/large/8bb38904gy1g5l5xw29axj20g60nx75r.jpg)

#### 熔断器声明方式

使用注册器声明一个默认的熔断器

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
```

builder模式声明自定义的熔断器

```java
//声明一个熔断器的自定义配置
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
  .failureRateThreshold(50)
  .waitDurationInOpenState(Duration.ofMillis(1000))
  .ringBufferSizeInHalfOpenState(2)
  .ringBufferSizeInClosedState(2)
  .recordFailure(e -> INTERNAL_SERVER_ERROR
                 .equals(getResponse().getStatus()))
  .recordExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .build();

//使用注册器的方式注册自定义熔断器
//这里是注册器在配置熔断器配置
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.of(circuitBreakerConfig);
//使用上面的注册器创建熔断器
CircuitBreaker bulkheadWithDefaultConfig = circuitBreakerRegistry.circuitBreaker("name1");

//直接使用注册器创建熔断器
CircuitBreaker bulkheadWithCustomConfig = circuitBreakerRegistry.circuitBreaker("name2", circuitBreakerConfig);

//不使用注册器直接声明熔断器
CircuitBreaker customCircuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
```

声明一个监听器

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
circuitBreakerRegistry.getEventPublisher()
  .onEntryAdded(entryAddedEvent -> {
    CircuitBreaker addedCircuitBreaker = entryAddedEvent.getAddedEntry();
    LOG.info("CircuitBreaker {} added", addedCircuitBreaker.getName());
  })
  .onEntryRemoved(entryRemovedEvent -> {
    CircuitBreaker removedCircuitBreaker = entryRemovedEvent.getRemovedEntry();
    LOG.info("CircuitBreaker {} removed", removedCircuitBreaker.getName());
  });
```

#### 原理

熔断器执行业务时会先获取执行许可`acquirePermission`,若执行成功则调用`onSuccess`,失败则调用`onError`



#### 代码示例1

初始化

```java
 CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig
            .custom().failureRateThreshold(90) //90%的失败率则触发熔断,默认50%
            .ringBufferSizeInClosedState(10) //半熔断时环的容量大小,默认为10
            .enableAutomaticTransitionFromOpenToHalfOpen() //自动从OPEN切换到HALF_OPEN 
             //设置当断路器处于HALF_OPEN状态下的ring buffer的大小，它存储了最近一段时间请求的成功失败状态，默认为10
     		.ringBufferSizeInHalfOpenState(10)	
             //用来指定断路器从OPEN到HALF_OPEN状态等待的时长，默认是60秒
     		.waitDurationInOpenState(Duration.ofSeconds(5)) 
            .recordExceptions(ChannelException.class) //设置统计的异常,其它异常不统计
            .build();
```

声明熔断器

```java
CircuitBreaker circuitBreaker = CircuitBreaker.of("backendName",circuitBreakerConfig);
//声明一个状态变化回调
CircuitBreaker.EventPublisher ep = circuitBreaker.getEventPublisher()
        .onStateTransition(new StatusChangeEvenConsumer());
```

```java
public class StatusChangeEvenConsumer implements EventConsumer<CircuitBreakerOnStateTransitionEvent> {
    @Override
    public void consumeEvent(CircuitBreakerOnStateTransitionEvent event){
        CircuitBreaker.StateTransition transition = event.getStateTransition();
        event.getCircuitBreakerName();
        System.out.println("------------------------" +
                                event.getCircuitBreakerName()+"transition from "+transition.getFromState()+" to"+transition.getToState() +
                            "-----------------------");
    }
}
```

执行

```java
BaseResponse result = circuitBreaker.executeSupplier(() -> service.test(finalI));
```

```java
public class BackendServiceImpl implements BackendService {

    @Override
    public BaseResponse test(Integer id) throws ChannelException {

        BaseResponse response = new BaseResponse();
        if(id <= 10){
            System.out.println("==========="+id+"success");
            response.setCode("success");
            return response;
        }else if(id >= 21 && id<= 25){
            System.out.println("==========="+id+"fail");
            response.setCode("fail");
            return response;
        }else {
            //熔断器会统计这个异常,计算为失败
            System.out.println("==========="+id+"fail");
            throw new ChannelException();
        }

    }

}
```

#### 代码示例2

启动类

```java
package tinys.breaker;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import tinys.listener.StatusChangeEvenConsumer;
import tinys.service.RandomValService;

import java.time.Duration;

/**
 * 描述 : 熔断器测试类
 * 2019/8/2
 */
public class CircuitBreakerTest {
    public static void main(String[] args) {


        CircuitBreaker circuitBreaker = CircuitBreaker.of("breaker",initCircuitBreakerConfig());
        //声明一个状态变化回调,状态类如上
        CircuitBreaker.EventPublisher ep = circuitBreaker.getEventPublisher().onStateTransition(new StatusChangeEvenConsumer());
        //注册事件的监听
        //registerEventCallback(circuitBreaker);


        //执行多次来触发熔断恢复断开等流程
        for (int i = 0; i < 30; i++) {
            try {
                Thread.sleep(100L);
                //在熔断器内执行业务方法
                circuitBreaker.executeRunnable(()->RandomValService.get());//这个是只执行的lambda
                circuitBreaker.executeSupplier(()-> RandomValService.get());//这个是可获取到值的lambda
            }catch (Exception e){
                System.out.println(e);
            }
        }
    }

    /**
     * 熔断状态监听
     */
    private static void registerEventCallback(CircuitBreaker circuitBreaker) {
        //监听事件
        circuitBreaker.getEventPublisher().onEvent(event -> System.out.println("触发事件-->" + event));
        //选择性监听事件
        circuitBreaker.getEventPublisher()
                .onSuccess(event -> System.out.println("触发成功事件-->" + event))
                .onError(event -> System.out.println("触发失败事件-->" + event));
    }

    /**
     *  声明熔断器配置参数
     */
    private static CircuitBreakerConfig initCircuitBreakerConfig() {
        return CircuitBreakerConfig
                    .custom().failureRateThreshold(50) //90%的失败率则触发熔断,默认50%
                    .ringBufferSizeInClosedState(5) //半熔断时环的容量大小,默认为10
                    .enableAutomaticTransitionFromOpenToHalfOpen() //自动从OPEN切换到HALF_OPEN
                    //设置当断路器处于HALF_OPEN状态下的ring buffer的大小，它存储了最近一段时间请求的成功失败状态，默认为10
                    .ringBufferSizeInHalfOpenState(10)
                    //用来指定断路器从OPEN到HALF_OPEN状态等待的时长，默认是60秒
                    .waitDurationInOpenState(Duration.ofSeconds(1))
                    .recordExceptions(RuntimeException.class) //设置统计的异常,其它异常不统计
                    .build();
    }

}

```

业务类

```java
public class RandomValService {
    public static  int get(){
        Random random = new Random();
        int num = random.nextInt(10);
        if (num < 5){
            //抛异常来代表执行失败
            throw  new RuntimeException("小于5触发异常");
        }
        return num;
    }
}
```



#### 参考链接

[文档](https://resilience4j.readme.io/docs/getting-started)

[github](https://github.com/resilience4j/resilience4j#circuitbreaker)

[HYSTRIX](https://github.com/Netflix/Hystrix/wiki)

[HYSTRI原理](https://github.com/Netflix/Hystrix/wiki/How-it-Works)