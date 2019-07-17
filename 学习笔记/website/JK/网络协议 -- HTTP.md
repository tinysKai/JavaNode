## 网络协议 -- HTTP

#### 统一资源定位符

`http://www.163.com`是个 URL，叫作**统一资源定位符**。HTTP 称为协议，`www.163.com `是一个域名.

#### 报文

**报文结构**

+ 请求行/状态行
+ 首部
+ 正文实体

**请求报文**

![https://s2.ax1x.com/2019/07/17/ZOprQI.png](http://ww1.sinaimg.cn/large/8bb38904ly1g5363wd7jcj20ia0a3aag.jpg)

请求行 -- 版本为HTTP版本,如`HTTP 1.1`.末尾的`crlf`为换行符.

首部 -- 首部是 key value，通过冒号分隔。如**Accept-Charset**，表示**客户端可以接受的字符集**,**Content-Type**是指**正文的格式**



**返回报文**

![https://s2.ax1x.com/2019/07/17/ZOifH0.png](http://ww1.sinaimg.cn/large/8bb38904ly1g536g3ykf1j20ih0a33yx.jpg)

首部 - **Content-Type**，表示返回的是 HTML，还是 JSON