## Hystrix

#### 定义

Hystrix 是 Netflix 开源的一款容错框架，包含常用的容错方法：线程池隔离、信号量隔离、熔断、降级回退。

#### Hystrix 的设计原则

- 防止单个服务的故障，耗尽整个系统服务的容器（比如 tomcat）的线程资源。
- 减少负载并快速失败，而不是排队。
- 在可行的情况下提供回退以保护用户免受故障。
- 使用隔离技术（如隔板，泳道和断路器模式）来限制任何一个依赖的影响。
- 通过近乎实时的指标，监控和警报来优化发现故障的时间。
- 通过配置更改的低延迟传播优化恢复时间，并支持 Hystrix 大多数方面的动态属性更改，从而允许您使用低延迟反馈循环进行实时操作修改。
- 保护整个依赖客户端执行中的故障，而不仅仅是在网络流量上进行保护降级、限流。

#### Hystrix的设计目标

- 通过 HystrixCommand 或者 HystrixObservableCommand 将所有的外部系统（或者称为依赖）包装起来，整个包装对象是单独运行在一个线程之中（这是典型的命令模式）。
- 超时请求应该超过你定义的阈值
- 为每个依赖关系维护一个小的线程池（或信号量）; 如果它变满了，那么依赖关系的请求将立即被拒绝，而不是排队等待。
- 统计成功，失败（由客户端抛出的异常），超时和线程拒绝。
- 打开断路器可以在一段时间内停止对特定服务的所有请求，如果服务的错误百分比通过阈值，手动或自动的关闭断路器。
- 当请求被拒绝、连接超时或者断路器打开，直接执行 fallback 逻辑。
- 近乎实时监控指标和配置变化。

#### Hystrix使用方式

``HystrixCommand`或`HystrixObservableCommand`

#### Hystrix执行方法

+ execute - 阻塞等待结果或接收到一个异常
+ queue - 异步执行任务,配合`Future`使用
+ observe - 订阅 Observable 代表依赖关系的响应，并返回一个 Observable，该 Observable 会复制该来源 Observable
+ toObservable - 返回一个 Observable，当您订阅它时，将执行 Hystrix 命令并发出其响应

```java
K             value   = command.execute();
Future<K>     fValue  = command.queue();
Observable<K> ohValue = command.observe();         //hot observable
Observable<K> ocValue = command.toObservable();    //cold observable
```

`execute`和`queue`只适用于`HystrixCommand`,`observe`和 `toObservable` 是 HystrixObservableCommand 中的方法。

#### Hystrix隔离方式

+ 线程池隔离
+ 信号量隔离

**线程池方式下业务请求线程和执行依赖的服务的线程不是同一个线程；信号量方式下业务请求线程和执行依赖服务的线程是同一个线程**

#### Hystrix整体流程

![](http://ww1.sinaimg.cn/large/8bb38904gy1g5lf5qn3b5j211w0i9tak.jpg)

#### 熔断器参数

+ **circuitBreaker.enabled** - 是否启用熔断器，默认是 TURE。
+ **circuitBreaker.forceOpen** - 熔断器强制打开，始终保持打开状态。默认值 FLASE
+ **circuitBreaker.forceClosed** - 熔断器强制关闭，始终保持关闭状态。默认值 FLASE
+ **circuitBreaker.errorThresholdPercentage** - 设定错误百分比，默认值 50%，例如一段时间（10s）内有 100 个请求，其中有 55 个超时或者异常返回了，那么这段时间内的错误百分比是 55%，大于了默认值 50%，这种情况下触发熔断器 - 打开。
+ **circuitBreaker.requestVolumeThreshold** - 默认值 20. 意思是至少有 20 个请求才进行 errorThresholdPercentage 错误百分比计算。
+ **circuitBreaker.sleepWindowInMilliseconds** - 熔断器开启后多久进行半熔断状态,默认为5000ms

#### 熔断器指标统计

Hystrix 会报告成功、失败、拒绝和超时的指标给回路器，回路器包含了一系列的**滑动窗口数据**，并通过该数据进行统计。

它使用这些统计数据来决定回路器是否应该熔断，如果需要熔断，将在一定的时间内不在请求依赖 [短路请求]（译者：这一定的时候可以通过配置指定），当再一次检查请求的健康的话会重新关闭回路器。

`hystrix.command.default.metrics.rollingStats.timeInMilliseconds` 设置统计的时间窗口值的，毫秒值，circuit break 的打开会根据1个rolling window的统计来计算。若rolling window被设为10000毫秒，则rolling window会被分成n个buckets，每个bucket包含success，failure，timeout，rejection的次数的统计信息。默认10000即10秒.

#### Fallback执行时机

当命令执行失败时，Hystrix 会尝试执行自定义的 Fallback 逻辑：

- 当`construct()`或者`run()`方法执行过程中抛出异常。
- 当回路器打开，命令的执行进入了熔断状态。
- 当命令执行的线程池和队列或者信号量已经满容。
- 命令执行超时。

#### `HystrixCommand`与``HystrixCircuitBroker``进行交互

![](http://ww1.sinaimg.cn/large/8bb38904gy1g5y6f6ub06j20m80i375m.jpg)

回路器打开和关闭有如下几种情况：

- 假设回路中的请求满足了一定的阈值（`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()`）
- 假设错误发生的百分比超过了设定的错误发生的阈值`HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()`
- 回路器状态由`CLOSE`变换成`OPEN`
- 如果回路器打开，所有的请求都会被回路器所熔断。
- 一定时间之后`HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()`，下一个的请求会被通过（处于半打开状态），如果该请求执行失败，回路器会在睡眠窗口期间返回`OPEN`，如果请求成功，回路器会被置为关闭状态，重新开启`1`步骤的逻辑(**只通过一个请求来决定是否打开熔断器风险略高**)。



#### 代码

```java
public class HystrixTest extends HystrixCommand<String> {
    public HystrixTest(String name){
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ThreadPoolTestGroup"))
                        .andCommandKey(HystrixCommandKey.Factory.asKey("testCommandKey"))
                        .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(name))
                        .andCommandPropertiesDefaults(  //设置熔断器变量
                                HystrixCommandProperties.Setter()
                                        .withCircuitBreakerEnabled(true)//默认是true，本例中为了展现该参数
                                        .withCircuitBreakerForceOpen(false)//默认是false，本例中为了展现该参数
                                        .withCircuitBreakerForceClosed(false)//默认是false，本例中为了展现该参数
                                        .withCircuitBreakerErrorThresholdPercentage(5)//(1)错误百分比超过5%
                                        .withCircuitBreakerRequestVolumeThreshold(10)//(2)时间窗口内错误数量达到某个值时会触发熔断器打开
                                        .withCircuitBreakerSleepWindowInMilliseconds(1000)//隔1s之后，熔断器会尝试半开(关闭)，重新放进来请求
                        )
                        .andThreadPoolPropertiesDefaults(
                                HystrixThreadPoolProperties.Setter()
                                        .withMaxQueueSize(10)   //配置队列大小
                                        .withCoreSize(2)    // 配置线程池里的线程数
                        )
        );
    }

    @Override
    protected String run() throws Exception {
        Random rand = new Random();
        //模拟错误百分比(方式比较粗鲁但可以证明问题)
        if(1==rand.nextInt(5)){
            throw new Exception("make exception");
        }
        return "running:  ";
    }

    @Override
    protected String getFallback() {
        return "fallback: ";
    }

    public static void main(String[] args) throws InterruptedException {
        for(int i=0;i<25;i++){
            Thread.sleep(100);
            HystrixCommand<String> command = new HystrixTest("testCircuitBreaker");
            String result = command.execute();
            //本例子中从第11次，熔断器开始打开
            System.out.println("call times:"+(i+1)+"   result:"+result +" isCircuitBreakerOpen: "+command.isCircuitBreakerOpen());
            //本例子中5s以后，熔断器尝试关闭，放开新的请求进来
        }
    }


}
```







#### 参考文档

[使用文档](https://github.com/Netflix/Hystrix/wiki/How-To-Use)

[原理](https://github.com/Netflix/Hystrix/wiki/How-it-Works)

[中文原理文档](https://segmentfault.com/a/1190000012439580)