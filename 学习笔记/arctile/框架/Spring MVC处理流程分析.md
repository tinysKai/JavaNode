# Spring MVC处理请求流程分析

## 主流程

#### 关键代码

```java
//我去除了一些异步,文件类型的request代码,只保留主流程代码
public class DispatcherServlet extends FrameworkServlet {
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		try {
			ModelAndView mv = null;
			Exception dispatchException = null;
			try {		
				//首先基本的HandleMapping在onRefresh时内部已经被初始化了
                  //这里是获取一个执行器链
				mappedHandler = getHandler(processedRequest);
				//获取真正的执行器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
                 //执行拦截器的预处理,拦截器是在getHandle方法中添加的
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}
				//触发调用逻辑
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                 //设置视图
				applyDefaultViewName(processedRequest, mv); 
                  //在视图返回前,执行拦截器的postHandle处理
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
             //渲染视图后,并触发拦截器的`afterCompletion`操作
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}	
	}
    
    //根据HandlerMapping找到合适的HandlerMethod,并包装成HandlerExecutionChain
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}
}   
```

#### 流程图

![https://ws3.sinaimg.cn/large/005BYqpggy1g2lgra9eu3j313a0r0tai.jpg](https://s2.ax1x.com/2019/05/01/EJFNvD.png)



## 子流程分析

### 获取执行器链

#### 关键代码

```java
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport implements HandlerMapping, Ordered, BeanNameAware {
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        //调用父类AbstractHandlerMethodMapping获取HandleMethod
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		//如果返回的类型是字符串,则通过bean工厂获取
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
        //获取执行器链
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		return executionChain;
	}
    
    
    //获取执行器链
    protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ? HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

         //添加拦截器
		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			else {
				chain.addInterceptor(interceptor);
			}
		}
		return chain;
	}
}		
```

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
	
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
         //Notice,下面两行代码加了读写锁,加锁代码以及try/finally代码被我省略了
         //根据调用路径找到匹配的HandlerMethod
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		
	}
}    
```

```java
public class HandlerMethod {
	public HandlerMethod createWithResolvedBean() {
		Object handler = this.bean;
         //如果HandlerMethod的bean是字符串,则从bean工厂获取
		if (this.bean instanceof String) {
			String beanName = (String) this.bean;
			handler = this.beanFactory.getBean(beanName);
		}
		return new HandlerMethod(this, handler);
	}
    
    //内部构造方法
    private HandlerMethod(HandlerMethod handlerMethod, Object handler) {
		this.bean = handler;
		this.beanFactory = handlerMethod.beanFactory;
		this.beanType = handlerMethod.beanType;
		this.method = handlerMethod.method;
         //注意这个属性,是之后执行业务逻辑方法位置的依据
		this.bridgedMethod = handlerMethod.bridgedMethod;
		this.parameters = handlerMethod.parameters;
		this.responseStatus = handlerMethod.responseStatus;
		this.responseStatusReason = handlerMethod.responseStatusReason;
		this.resolvedFromHandlerMethod = handlerMethod;
	}
}   
```

**HandleMethod的初始化流程**

![https://ws3.sinaimg.cn/large/005BYqpggy1g2ll0arfq6j30qz0i875a.jpg](https://s2.ax1x.com/2019/05/01/EJZ59I.png)

------

### Adaptor执行器

#### 关键代码

```java
//入口是在`DispatcherServlet`的`doDispatch`方法里面的`ha.handle(processedRequest, response, mappedHandler.getHandler())` 
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware, InitializingBean {	
	 //真正执行业务代码的方法,创建了`ServletInvocableHandlerMethod`用以下面调用
     protected ModelAndView invokeHandlerMethod(HttpServletRequest request,HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
			
            //创建ServletInvocableHandlerMethod,其实HandleMethod的子类
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
			//关键代码入口
			invocableMethod.invokeAndHandle(webRequest, mavContainer);

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
 }   
```



```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,Object... providedArgs) throws Exception {
		//关键代码,真正执行请求代码的方法
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		try {
            //处理返回数据
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			throw ex;
		}
	}
    
    
    public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,Object... providedArgs) throws Exception {
		//获取方法参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		return doInvoke(args);
	}
    
    protected Object doInvoke(Object... args) throws Exception {
        //使用反射将方法设置方法访问级别
		ReflectionUtils.makeAccessible(getBridgedMethod());
		try {
            //使用反射执行`RequestMapping`对应的方法
			return getBridgedMethod().invoke(getBean(), args);
		}
		catch (IllegalArgumentException ex) {
			assertTargetBean(getBridgedMethod(), getBean(), args);
			String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
			throw new IllegalStateException(formatInvokeError(text, args), ex);
		}
		catch (InvocationTargetException ex) {
			// Unwrap for HandlerExceptionResolvers ...
			//我省略了异常包装以及抛出
		}
	}

}    
```



#### 流程图

![<https://ws3.sinaimg.cn/large/005BYqpggy1g2m2b4vuesj30ze0kv76f.jpg>](https://s2.ax1x.com/2019/05/01/EYulTO.png)