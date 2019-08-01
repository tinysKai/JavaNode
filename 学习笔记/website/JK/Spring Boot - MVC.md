## Spring Boot - MVC

#### pom

```xml
<!--必须引入web模块-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### 控制器

```java
@RestController
public class HelloControler {
    @RequestMapping("/hello")
    public String hello(){
        return "Hello World";
    }
}
```

#### 拦截器

```java
@Slf4j
public class LogInteceptor implements HandlerInterceptor {
    @Override
    public  boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String method = handler.getClass().getSimpleName();
        if (handler instanceof HandlerMethod) {
            String beanType = ((HandlerMethod) handler).getBeanType().getName();
            String methodName = ((HandlerMethod) handler).getMethod().getName();
            method = beanType + "." + methodName;
        }
        log.info("请求路径为{},映射到{}方法", request.getRequestURI(), method);
        return true;
    }
}
```

#### 启动器

实现WebMvcConfigurer并添加拦截器

```java
@SpringBootApplication
public class InterceptorApplication implements WebMvcConfigurer {
    
    //注册拦截器以及其对应的拦截方法
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInteceptor())
                .addPathPatterns("/hello/**");
    }

    public static void main(String[] args) {
        SpringApplication.run(InterceptorApplication.class, args);
    }
}
```

