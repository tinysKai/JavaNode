### springboot-boot安装
地址 : http://repo.spring.io/release/org/springframework/boot/spring-boot-cli/

安装步骤 :
+ 下载spring-boot-cli-2.0.4.RELEASE-bin.zip压缩包
+ 解压到任意路径,并将其bin路径添加到path系统环境变量]
+ cmd控制台输出`spring --version`


### 创建项目
使用IDEA创建springboot项目  
使用`Spring Initializr`来初始化项目  

过程:  
New --> Project -->  Spring Initializr


### 条件化配置  
条件化配置允许配置存在于应用程序中，但在满足某些特定条件之前都忽略这个配置。  
在Spring里可以很方便地编写你自己的条件，你所要做的就是实现Condition接口，覆盖它的matches()方法。

```java
package readinglist;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
public class JdbcTemplateCondition implements Condition {
  @Override
  public boolean matches(ConditionContext context,
                         AnnotatedTypeMetadata metadata) {
    try {
      context.getClassLoader().loadClass(
             "org.springframework.jdbc.core.JdbcTemplate");
      return true;
    } catch (Exception e) {
      return false;
    }
  }
}

```


```
@Conditional(JdbcTemplateCondition.class)
public MyService myService() {
    ...
}


在这个例子里，只有当JdbcTemplateCondition类的条件成立时才会创建MyService这个Bean。
也就是说MyService Bean创建的条件是Classpath里有JdbcTemplate。否则，这个Bean的声明就会被忽略掉。

```



















```