## Spring MVC初始化流程分析

#### 核心类层级结构

![https://ws3.sinaimg.cn/large/005BYqpggy1g2i6mgog10j30p30e7aah.jpg](https://s2.ax1x.com/2019/04/28/EM6dSK.png)



#### 流程描述

基于`GenericServlet`在`HttpServletBean`的init方法来执行初始化流程.

`FrameworkServlet`基于父类`HttpServletBean`在`initServletBean`方法里开始创建web引用的上下文

`DispatcherServlet`里调用了`onRefresh`来初始化策略



#### 流程图

![https://ws3.sinaimg.cn/large/005BYqpggy1g2iolmkpn1j30qp0dd74s.jpg](https://s2.ax1x.com/2019/04/28/ElCvyq.png)



#### 声明组件时用到的配置文件

```properties
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```



#### 设计模式

+ 多处使用了模板方法将细节定义交由子类去实现.
+ 使用简单工厂来返回实例



#### 源码分析

`<https://www.cnblogs.com/tengyunhao/p/7518481.html>`

`<https://www.cnblogs.com/tengyunhao/p/7658952.html>`

