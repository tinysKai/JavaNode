## Java核心36讲-JVM笔记

#### 解释执行

我们开发的 Java 的源代码，首先通过 Javac 编译成为字节码（bytecode），然后，在运行时，通过 Java 虚拟机（JVM）内嵌的解释器将字节码转换成为最终的机器码。但是常见的 JVM，比如我们大多数情况使用的 Oracle JDK 提供的 Hotspot JVM，都提供了 JIT（Just-In-Time）编译器，也就是通常所说的动态编译器，JIT 能够在运行时将热点代码编译成机器码，这种情况下部分热点代码就属于编译执行.

#### Java引用类型

- 强引用
- 软引用(内存不足时会优先回收此类对象)
- 弱引用(执行GC时会回收此类对象)
- 虚对象

####  JVM 内存区域的划分

![](https://s2.ax1x.com/2019/06/23/ZCkf6H.png)

#### 垃圾收集的算法

- 复制算法(年轻代)
- 标记-清理(老年代-CMS,会有碎片化问题)
- 标记-整理(老年代,整理需耗费多余的时间,serial old)



#### 类加载器与双亲委托机制

- Bootstrp loader
- ExtClassLoader  
- AppClassLoader 
- CustomerClassLoader

![](https://s2.ax1x.com/2019/06/23/ZCkpWD.png)

双亲委托机制即当类加载器加载时先到父加载器查询是否有此类(父加载器会去其父加载器查询),若有则从父加载器加载,若无则加载到自身中,以便下次加载时缓存直接返回.

双亲委托机制不适合的场景如我们希望一个jvm能够同时加载某类的不同版本，那么双亲委派就不合适了，需要的是在不同范围内（例如模块）单独加载.

若怀疑应用存在引用（或 finalize）导致的回收问题,可使用JVM 自身便提供了明确的选项（PrintReferenceGC）去获取相关信息

```
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintReferenceGC

0.403: [GC (Allocation Failure) 0.871: [SoftReference, 0 refs, 0.0000393 secs]0.871: 
[WeakReference, 8 refs, 0.0000138 secs]0.871: 
[FinalReference, 4 refs, 0.0000094 secs]0.871: 
[PhantomReference, 0 refs, 0 refs, 0.0000085 secs]0.871: 
[JNI Weak Reference, 0.0000071 secs]
[PSYoungGen: 76272K->10720K(141824K)] 128286K->128422K(316928K), 0.4683919 secs] [Times: user=1.17 sys=0.03, real=0.47 secs] 

```



#### 类加载过程 

- 加载
- 链接
  - 验证
  - 准备
  - 解析
- 初始化

```
1.首先是加载阶段（Loading），它是 Java 将字节码数据从不同的数据源读取到 JVM 中，并映射为 JVM 认可的数据结构（Class 对象），
	这里的数据源可能是各种各样的形态，如 jar 文件、class 文件，甚至是网络数据源等；
	如果输入数据不是 ClassFile 的结构，则会抛出 ClassFormatError。
	加载阶段是用户参与的阶段，我们可以自定义类加载器，去实现自己的类加载过程。

2.第二阶段是链接（Linking），这是核心的步骤，简单说是把原始的类定义信息平滑地转化入 JVM 运行的过程中。这里可进一步细分为三个步骤：

    验证（Verification），这是虚拟机安全的重要保障，JVM 需要核验字节信息是符合 Java 虚拟机规范的，否则就被认为是 VerifyError，
     这样就防止了恶意信息或者不合规的信息危害 JVM 的运行，验证阶段有可能触发更多 class 的加载。

    准备（Preparation），创建类或接口中的静态变量，并初始化静态变量的初始值。
    但这里的“初始化”和下面的显式初始化阶段是有区别的，侧重点在于分配所需要的内存空间，不会去执行更进一步的 JVM 指令。

    解析（Resolution），在这一步会将常量池中的符号引用（symbolic reference）替换为直接引用。
    在Java 虚拟机规范中，详细介绍了类、接口、方法和字段等各个方面的解析。

3.最后是初始化阶段（initialization），这一步真正去执行类初始化的代码逻辑，包括静态字段赋值的动作，
	以及执行类定义中的静态初始化块内的逻辑，编译器在编译阶段就会把这部分逻辑整理好，父类型的初始化逻辑优先于当前类型的逻辑。
```

#### JVM调优

对于 GC 调优来说，首先就需要清楚调优的目标是什么？

从性能的角度看，通常关注三个方面，内存占用（footprint）、延时（latency）和吞吐量（throughput），大多数情况下调优会侧重于其中一个或者两个方面的目标，很少有情况可以兼顾三个不同的角度。



#### NoClassDefFoundError 和 ClassNotFoundException有什么区别?

`NoClassDefFoundError `是error,而`ClassNotFoundException`是exception.前者是指要查找的类在编译时期是存在，运行期间却找不到该对象对应的类，这个时候就会导致NoClassDeFoundError错误。而后者指反射路径错误导致的异常,常见情况如反射` Class c = Class.forName("com.demo.Test")`.