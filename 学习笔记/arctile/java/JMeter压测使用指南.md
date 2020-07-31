## JMeter压测使用指南

### 使用JMeter压测HTTP请求

1.添加线程组

![QQ截图20200610101746.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfmz79zdgfj20mu0c23z3.jpg)

2.添加HTTP请求取样器

![QQ截图20200610101913.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfmz8o2nbsj20me0i1gmr.jpg)

3.配置http请求信息

![QQ截图20200610102048.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfmza9pj9dj21670gv3zg.jpg)

4.添加监听器(查看结果树,汇总报告,聚合报告等)

![QQ截图20200610102147.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfmzbb0s67j20jk0jwdgt.jpg)



###  使用JMeter压测添加动态数据源

1.添加数据文件

![QQ图片20200609174106.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfm6ff1ajdj20q40n1ace.jpg)

2.配置数据文件指向

![QQ截图20200609174657.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfm6kajk4ej211d0in75j.jpg)

3.数据变量使用`${}`来引用

![QQ截图20200609175918.png](http://ww1.sinaimg.cn/large/8bb38904gy1gfm6x1hh3aj20t60jq3zh.jpg)



### 压测Dubbo接口

这里主要简单介绍一下，开源dubbo压测插件[jmeter-plugins-for-apache-dubbo](https://github.com/thubbo/jmeter-plugins-for-apache-dubbo)的使用。使用流程上基本跟上述的http接口压测一致，区别在于不同的取样器。

- 安装dubbo压测插件及相关依赖

- - 将插件jar包放到JMeter的lib/ext目录下 
  - 将dubbo.jar包放到JMeter的lib目录下

- 添加线程组
- 添加Dubbo Sample