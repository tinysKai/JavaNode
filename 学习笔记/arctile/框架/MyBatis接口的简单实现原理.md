## MyBatis接口的简单实现原理

用过MyBatis3的人可能会觉得为什么MyBatis的Mapper接口没有实现类，但是可以直接用？

那是因为MyBatis使用Java动态代理实现的接口。

这里仅仅举个简单例子来说明原理，不是完全针对MyBatis的，这种思想我们也可以应用在其他地方。

#### 定义一个接口

```java
public interface MethodInterface {
    String helloWorld();
}

```



#### 实现动态代理接口

```java
public class MethodProxy<T> implements InvocationHandler {

    private Class<T> methodInterface;

    public MethodProxy(Class<T> methodInterface) {
        this.methodInterface = methodInterface;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("=========================");
        System.out.println("方法名:" + method.getName());
        //针对不同的方法进行不同的操作
        return null;
    }
}
```


**这里针对invoke方法简单说说MyBatis的实现原理，在该方法中，我们通过Method能够获取接口和方法名，接口的全名相当于MyBatis XML中的namespace，方法名相当于具体一个方法中的id。也就是说通过动态代理后，可以通过SqlSession来通过namespace.id方式来调用相应的方法。使用接口更方便，但是是一种间接的方式。**

#### 动态代理工厂类

```java
public class MethodProxyFactory {
    public static <T> T newInstance(Class<T> methodInterface) {
        final MethodProxy<T> methodProxy = new MethodProxy<T>(methodInterface);
        return (T) Proxy.newProxyInstance(
                Thread.currentThread().getContextClassLoader(), 
                new Class[]{methodInterface}, 
                methodProxy);
    }
}
```

通过该工厂类可以生成任意接口的动态代理类。

#### 测试

```java
MethodInterface method = MethodProxyFactory.newInstance(MethodInterface.class);
method.helloWorld();
```

可以看到MethodInterface没有实现类也可以执行。

#### 总结

一般谈到动态代理我们通常的用法都是处理事务、日志或者记录方法执行效率等方面的应用。都是对实现类方法的前置或者后置的特殊处理。



#### 参考链接

https://blog.csdn.net/isea533/article/details/48296849

