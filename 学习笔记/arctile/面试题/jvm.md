>GC算法
+ 标记-清除   
   - 最原始的基本没用,碎片化严重
+ 复制        
   - 用于年轻代 两个sur,一个eden
+ 标记-整理   
   - 用于老年代
+ 分代

>CMS的垃圾收集器的收集过程 :  

     1.初始标记 --> stop the world 耗时比较短  
     2.并发标记  
     3.重新标记 --> stop the world 耗时非常短  
     4.并发清除
     
>JVM的分类
+ 堆
+ 栈
+ 计数器
+ 常量池
+ 永久代  
+ 本地方法栈
   
   
>Minor GC ，Full GC 触发条件

      Minor GC触发条件：当Eden区满时，触发Minor GC。
      
 
      Full GC触发条件：
      （1）调用System.gc时，系统建议执行Full GC，但是不必然执行
      （2）老年代空间不足
      （3）永久代空间不足
      （4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存
      （5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小
         
         
         
>Java的类加载器

     1.bootstrap class loader
          用于加载最根本的类加载器..用于加载JDK里面的jre..是用汇编或者C写的
     2.ext class loader 
          用于加载jre/lib/ext的类
     3.application class loader
          用于加载用户自己写的类..
     4. other class loader(包括其他各种类加载器)
     
     补充:类加载器也是需要bootstrap加载器来加载..
     每个类加载器在加载一个类时会询问它的父加载器有没加载这个类.如果已加载就不会再加载多一次了.


#### happen-before
>happens-before原则定义
如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。  
两个操作之间存在happens-before关系，并不意味着一定要按照happens-before原则制定的顺序来执行。如果重排序之后的执行结果与按照happens-before关系来执行的结果一致，那么这种重排序并不非法。  

>happens-before原则规则：  

     程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；
     锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作；
     volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
     传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
     线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作；
     线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
     线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
     对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始；

