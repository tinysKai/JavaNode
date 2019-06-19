## Mybatis使用到的设计模式

#### 简单工厂模式

**使用到工程模式的类**

+ DefaultSqlSessionFactory
+ MapperProxyFactory

![](https://s2.ax1x.com/2019/04/25/EeZ3Q0.png)

#### 代理模式

使用动态代理来创建接口方法实现

![](https://s2.ax1x.com/2019/04/25/EeeHjx.png)

#### 装饰模式

CachingExecutor使用委托机制委托SimpleExecutor来实现缓存功能

![](https://s2.ax1x.com/2019/04/25/EemBqK.png)

#### Build模式

`ParameterMapping`使用build模式来构建入参映射

使用了嵌套类`ParameterMapping.Builder`



#### 模板方法

BaseExecutor使用模板方法来委托子类处理具体查询工作

![](https://s2.ax1x.com/2019/04/25/EemzZT.png)



#### 单例模式

ErrorContext使用ThreadLocal来实现的单例

```java
public class ErrorContext {
    private static final ThreadLocal<ErrorContext> LOCAL = new ThreadLocal<ErrorContext>();
    
    public static ErrorContext instance() {
    ErrorContext context = LOCAL.get();
    if (context == null) {
      context = new ErrorContext();
      LOCAL.set(context);
    }
    return context;
  }
}    
```

