## Spring MVC参数绑定

### 常用的注解

+ PathVariable 
+ RequestHeader 
+ CookieValue 
+ RequestParam
+ RequestBody



### 注解解释

#### @PathVariable

`@PathVariable` 是用来获得请求url中的动态参数的，可以将URL中的变量映射到功能处理方法的参数上，其中URL 中的 {xxx} 占位符可以通过@PathVariable(“xxx“) 绑定到操作方法的入参中。

```java
//可直接使用Get方法
@ResponseBody
@RequestMapping("/test/{param1}/{param2}")
public void test(HttpServletRequest request, @PathVariable String param1, @PathVariable String param2) {
        System.out.println("通过PathVariable获取的参数param1=" + param1);
        System.out.println("通过PathVariable获取的参数param2=" + param2);
    }
```



#### @RequestHeader 

@RequestHeader 注解，可以把Request请求header部分的值绑定到方法的参数上。

```java
 @ResponseBody
 @RequestMapping("/testHeaderParam")
 public void testHeaderParam(HttpServletRequest request, @RequestHeader String param1) {
        System.out.println("通过RequestHeader获取的参数param1=" + param1);
 }
```

Postman使用方式

![https://mmbiz.qpic.cn/mmbiz_png/OXnTwHqwxnZ6eR0ZmRcA808Bb7WLhUySD2y2h8haaqH74zASBLZpdSl0ZDXibZR8kvIic2CnHNG9kbK2cwTsiaeRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://s2.ax1x.com/2019/04/26/Em8TNq.png)



#### @CookieValue 

@CookieValue 可以把Request header中关于cookie的值绑定到方法的参数上。

```java
 @ResponseBody
 @RequestMapping("/testCookieParam")
 public void testCookieParam(HttpServletRequest request, HttpServletResponse response, @CookieValue String sessionid) {
        System.out.println("通过CookieValue获取的参数sessionid=" + sessionid);
 }
```

Postman使用方式

![https://mmbiz.qpic.cn/mmbiz_png/OXnTwHqwxnZ6eR0ZmRcA808Bb7WLhUySW99Tj5PlwoiaU54orANia9KRSQEtoSX9lpg52MsE81QGQpFs4aLgiaHjg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://s2.ax1x.com/2019/04/26/EmG94x.png)



#### @RequestParam

@RequestParam注解用来处理Content-Type: 为 application/x-www-form-urlencoded编码的内容。提交方式为get或post。（Http协议中，form的enctype属性为编码方式，常用有两种：application/x-www-form-urlencoded和multipart/form-data，默认为application/x-www-form-urlencoded）；

@RequestParam注解实质是将Request.getParameter() 中的Key-Value参数Map利用Spring的转化机制ConversionService配置，转化成参数接收对象或字段，get方式中queryString的值，和post方式中body data的值都会被Servlet接受到并转化到Request.getParameter()参数集中，所以@RequestParam可以获取的到；

该注解有三个属性： value、required、defaultValue； value用来指定要传入值的id名称，required用来指示参数是否必录，defaultValue表示参数不传时候的默认值。

```java
@ResponseBody
@RequestMapping("/testRequestParam")
public void testRequestParam(HttpServletRequest request, @RequestParam(value = "num", required = true, defaultValue = "0") int num) {
        System.out.println("通过RequestParam获取的参数num=" + num);
}
```

Postman使用方式

![https://mmbiz.qpic.cn/mmbiz_png/OXnTwHqwxnZ6eR0ZmRcA808Bb7WLhUySV4asKw6vdoa4kTshvXq6yp51LONd7n2PPWDaduSiaULEic1H41ACRa5Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://s2.ax1x.com/2019/04/26/EmGwGV.png)



#### @RequestBody

@RequestBody注解用来处理HttpEntity（请求体）传递过来的数据，一般用来处理非Content-Type: application/x-www-form-urlencoded编码格式的数据；

GET请求中，因为没有HttpEntity，所以@RequestBody并不适用；

POST请求中，通过HttpEntity传递的参数，必须要在请求头中声明数据的类型Content-Type，SpringMVC通过使用HandlerAdapter配置的HttpMessageConverters来解析HttpEntity中的数据，然后绑定到相应的bean上。

```java
 @ResponseBody
 @RequestMapping("/testRequestBody")
 public void testRequestBody(HttpServletRequest request, @RequestBody String bodyStr){
        System.out.println("通过RequestBody获取的参数bodyStr=" + bodyStr);
 }
```

Postman使用方式

![https://mmbiz.qpic.cn/mmbiz_png/OXnTwHqwxnZ6eR0ZmRcA808Bb7WLhUySTZO1E5po2gZgSyibibmAiaYCicjVaADnsqoItLmiangyGpOVZ4X5xWSkNuw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://s2.ax1x.com/2019/04/26/EmGTqH.png)

如果@RequestBody使用在对象上,则可以使用raw的json模式来发送数据

![https://ws3.sinaimg.cn/large/005BYqpggy1g2fut6nn00j30ht04udfu.jpg](https://s2.ax1x.com/2019/04/26/EmJPds.png)