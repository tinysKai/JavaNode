## Linux性能分析

#### CPU

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9cteprcj30s907rjru.jpg](https://s2.ax1x.com/2019/05/25/VFOLnA.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9e6erl1j30x40a8q3i.jpg](https://s2.ax1x.com/2019/05/25/VFOvAP.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9erxpphj30so06xdg4.jpg](https://s2.ax1x.com/2019/05/25/VFXp9S.png)



#### IO

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9jgejqbj30sq050t8r.jpg](https://s2.ax1x.com/2019/05/25/VFXyUP.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9k49bk2j30q5095js1.jpg](https://s2.ax1x.com/2019/05/25/VFX64f.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9krzn4tj30p505174c.jpg](https://s2.ax1x.com/2019/05/25/VFXWvQ.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9ljf1aej30ye0ba0tm.jpg](https://s2.ax1x.com/2019/05/25/VFX5bn.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d9ma3c1ij312k09qdgr.jpg](https://s2.ax1x.com/2019/05/25/VFXTU0.png)

#### 网络

```
查看网络信息的命令 : netstat,其常见参数

-a (all)显示所有选项，默认不显示LISTEN相关
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。
-l 仅列出有在 Listen (监听) 的服務状态

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。


提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到
```

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d7hedm76j30sp04ojri.jpg](https://s2.ax1x.com/2019/05/25/VFLSVP.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d7ibs8l3j30tt0boq3h.jpg](https://s2.ax1x.com/2019/05/25/VFLPPS.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d7iy2ru8j30ny0jwjs2.jpg](https://s2.ax1x.com/2019/05/25/VFLi8g.png)

![https://ws3.sinaimg.cn/large/005BYqpggy1g3d7jqhmxwj30ro0400sr.jpg](https://s2.ax1x.com/2019/05/25/VFLF2Q.png)



