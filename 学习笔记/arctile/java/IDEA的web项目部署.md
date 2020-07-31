## IDEA的web项目部署

1.创建项目

![QQ截图20200718153857.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggv615iojrj20p60gh75i.jpg)

2.设置默认输出路径

![QQ截图20200718185256.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbmyzj35j216p0p5tbh.jpg)

3.设置项目结构

![QQ截图20200718185421.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbofkppvj21gq0ktdi2.jpg)

![QQ截图20200718185700.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbr6axmhj21ap0lr3zs.jpg)

![QQ截图20200718185504.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbp5m3dwj21gt0n80v1.jpg)

![QQ截图20200718185601.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbq65z7bj21aq0lhta6.jpg)

4.设置web项目部署包输出结构

![QQ截图20200718185817.png](http://ww1.sinaimg.cn/large/8bb38904gy1ggvbspijhbj21ao0liwg2.jpg)

5.设置基础页面的访问

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:tx="http://www.springframework.org/schema/tx"
	   xmlns:util="http://www.springframework.org/schema/util" xmlns:mvc="http://www.springframework.org/schema/mvc"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx-3.2.xsd
         http://www.springframework.org/schema/util
    	http://www.springframework.org/schema/util/spring-util-3.0.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.2.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    
		<!-- 自动扫描的包名 -->
		<context:component-scan base-package="com.*"/>
		<import resource="classpath*:spring-mybatis.xml" />

		<!--这句主要是能不能访问到后端服务的请求-->
		<mvc:annotation-driven></mvc:annotation-driven>
		<!--这句话关系到能不能直接访问login.html-->
		<mvc:resources mapping="/*.html" location="/view/" /

</beans>
```

