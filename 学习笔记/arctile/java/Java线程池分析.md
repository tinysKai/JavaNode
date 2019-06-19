# Java线程池分析

## 定义

 一种线程使用模式。线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建与销毁线程的代价。线程池不仅能够保证内核的充分利用，还能防止过分调度。

### 线程池参数

**corePoolSize** - 核心线程数

**maximumPoolSize** - 最大线程数

**keepAliveTime** - 当线程数大于核心线程数时,多余线程等待新任务的最大空闲时间

**unit**

**workQueue** - 保存任务的线程队列

**threadFactory** - 声明一个线程工厂,自定义的线程工厂可定制线程池名字以及创建的线程的名字等属性

**RejectedExecutionHandler** - 拒绝策略,默认的策略是抛出异常



#### 策略类型

抛异常 - AbortPolicy

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
   throw new RejectedExecutionException("Task " + r.toString() +" rejected from " +e.toString());
}
```

抛弃任务 - DiscardPolicy

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {}
```

抛弃最旧任务 - DiscardOldestPolicy

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```

直接同步运行线程,而不通过线程池执行 - CallerRunsPolicy

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```



## 线程池好处

- 降低资源消耗 通过重复利用已创建的线程,降低创建和销毁线程造成的系统资源消耗
- 提高响应速度 当任务到达时,任务可以不需要等到线程创建就能立即执行
- 提高线程的可管理性 线程是稀缺资源,如果过多地创建,不仅会消耗系统资源，还会降低系统的稳定性，导致使用线程池可以进行统一分配、调优和监控



## Java自带线程池分析

### newFixedThreadPool

**创建声明**

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

**主要问题**

 	阻塞队列是LinkedBlockingQueue,其队列大小为Integer.MAX_VALUE,有内存溢出风险

### newCachedThreadPool

**创建声明**

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

**主要问题**

​	最大线程数为Integer.MAX_VALUE,容易线程数爆掉

### newSingleThreadExecutor

**创建声明**

```java
  public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

**主要问题**

​	阻塞队列是LinkedBlockingQueue,其队列大小为Integer.MAX_VALUE,有内存溢出风险

### newScheduledThreadPool

**创建声明**

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

**主要问题**

- 阻塞队列是LinkedBlockingQueue,其队列大小为Integer.MAX_VALUE,有内存溢出风险
- 不回收工作线程



## 自定义线程池

```java
 ExecutorService executorService = new ThreadPoolExecutor(8, 8,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(1000),
                new ThreadFactory() {
                    AtomicInteger count = new AtomicInteger(0);
                    @Override
                    public Thread newThread(Runnable r) {
                        int c = count.incrementAndGet();
                        Thread thread = new Thread(r);
                        thread.setName("exec-pool-name" + c);
                        return thread;
                    }
                },
                 new ThreadPoolExecutor.CallerRunsPolicy());
```





## 线程池定义拓展

常见使用方式 - 定义一个子类类实现ThreadPoolExecutor并覆盖其几个方法 : 

- beforeExecute
- afterExecute
- terminated

```java
public class TimingThreadPool extends ThreadPoolExecutor {
    private final ThreadLocal<Long> startTime = new ThreadLocal<Long>();
    private final Logger log = Logger.getAnonymousLogger();
    private final AtomicLong numTasks = new AtomicLong();
    private final AtomicLong totalTime = new AtomicLong();

    public TimingThreadPool(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
    }

    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        log.info(String.format("Thread %s: start %s", t, r));
        startTime.set(System.nanoTime());
    }

    protected void afterExecute(Runnable r, Throwable t) {
        try {
            long endTime = System.nanoTime();
            long taskTime = endTime - startTime.get();
            numTasks.incrementAndGet();
            totalTime.addAndGet(taskTime);
            log.info(String.format("Thread %s: end %s, time=%dns", t, r, taskTime));
        } finally {
            super.afterExecute(r, t);
        }
    }

    protected void terminated() {
        try {
            log.info(String.format("Terminated: avg time=%dns", totalTime.get() / numTasks.get()));
        } finally {
            super.terminated();
        }
    }
}

ThreadPoolExecutor  exec = 
    new TimingThreadPool(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
```



------



## 源码分析

#### ThreadPoolExecutor 关键方法

+ execute
+ addWorker
+ runWorker
+ shutdown
+ shutdownNow

#### 执行逻辑描述

- 当线程数少于核心线程数时,创建新线程来执行任务
- 当线程数大于等于核心线程数时,而新任务可排在队列时,入队等待(阻塞队列是针对整个线程池的)
- 当线程数大于核心线程数而且无法入队等待时,创建新线程来执行任务
- 当线程数等于最大任务数时并且此时任务无法入队列时,执行拒绝策略

#### CTL状态变量解释

![https://s2.ax1x.com/2019/05/12/E4iEa8.png](https://ws3.sinaimg.cn/large/005BYqpggy1g2yvb7i6qjj30hv0lg403.jpg)

#### Worker类分析

特点

- Worker是实现了Runnable接口的

- 线程池的线程是基于HashSet<Worker>来实现的

- Worker是基于AQS的独占模式

- AQS的主要使用方法

  - lock - 在执行任务时会先锁住任务

  - unlock - 执行完任务之后会释放任务

  - tryLock - 在关闭线程池时会中断所有的空闲线程,而执行任务的Worker由于加锁了所以尝试获取失败

    

执行中的任务的关闭时机

`runWorker`方法获取不到新任务时会执行`processWorkerExit`方法.`processWorkerExit`会尝试tryTerminate去判断是否需关闭线程池



#### 添加任务模块代码分析

```java
 public void execute(Runnable command) {
 	 //获取控制器 
     int c = ctl.get();
     //当前线程数小于核心线程数时
     if (workerCountOf(c) < corePoolSize) {
         //增加线程
         if (addWorker(command, true))
             return;
         c = ctl.get();
     }
     //线程数大于核心线程数时尝试向阻塞队列添加任务    
     //这里的offer添加完任务后,是在runWorker方法去获取任务的  
     if (isRunning(c) && workQueue.offer(command)) {
         //二次检查控制器
         int recheck = ctl.get();
         //检查线程池是否突然关闭
         if (! isRunning(recheck) && remove(command))
             reject(command);
         //检查线程池是否突然木有线程了
         else if (workerCountOf(recheck) == 0)
             addWorker(null, false);
     }
     //当阻塞队列添加不了任务时,尝试直接添加线程
     else if (!addWorker(command, false))
         //线程添加失败时,触发拒绝策略
         reject(command);
    }
```

```java
//所以线程池的主要逻辑依赖于addWorker 
private boolean addWorker(Runnable firstTask, boolean core) {
     retry:
     for (;;) {
         int c = ctl.get();
         int rs = runStateOf(c);

         // Check if queue empty only if necessary.
         if (rs >= SHUTDOWN &&
             ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
             return false;

         for (;;) {
             int wc = workerCountOf(c);
              //这里决定了能添加线程的条件是:                
             //	1.线程数小于核心线程数               
             //	2.线程数大于核心线程数阻塞队列已满而且目前线程数小于最大线程数 
             if (wc >= CAPACITY ||
                 wc >= (core ? corePoolSize : maximumPoolSize))
                 return false;
             //满足上面条件的情况下尝试CAS添加新线程
             if (compareAndIncrementWorkerCount(c))
                 //添加线程数成功的情况下跳出两层循环
                 break retry;
             c = ctl.get();  // Re-read ctl
             if (runStateOf(c) != rs)
                 continue retry;
             // else CAS failed due to workerCount change; retry inner loop
         }
     }

     boolean workerStarted = false;
     boolean workerAdded = false;
     Worker w = null;
     try {
         w = new Worker(firstTask);
         final Thread t = w.thread;
         if (t != null) {
              //重入锁主要作用是控制线程数的操作 
             final ReentrantLock mainLock = this.mainLock;
             mainLock.lock();
             try {
                 //加锁后再检查一下线程池状态
                 int rs = runStateOf(ctl.get());

                 if (rs < SHUTDOWN ||
                     (rs == SHUTDOWN && firstTask == null)) {
                     if (t.isAlive()) // precheck that t is startable
                         throw new IllegalThreadStateException();
                     workers.add(w);
                     int s = workers.size();
                     if (s > largestPoolSize)
                         largestPoolSize = s;
                     workerAdded = true;
                 }
             } finally {
                 mainLock.unlock();
             }
             if (workerAdded) {
                  //添加Worker成功,启动线程 
                 t.start();//这里其实就是调用Worker的run方法,也就是调用runWorker方法 
                 workerStarted = true;
             }
         }
     } finally {
         if (! workerStarted)
             //添加线程失败回滚计数   
             addWorkerFailed(w);
     }
     return workerStarted;
 }
```

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    //这里置空Worker的第一个任务,线程后面的任务是从阻塞队列获取的        
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //getTask也是一个核心的方法
        //注意这里当大于核心线程数时,有可能返回null的任务此时的线程就等待消灭,使用阻塞队列的poll(time,unit)            
        //而如果是小于核心线程数,则会一直阻塞,则到获取到下一个任务,使用阻塞队列的take方法阻塞
        while (task != null || (task = getTask()) != null) {
            w.lock();
            //...这里有响应检查中断
            try {
                //线程执行前触发的钩子
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行线程任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
         //线程退出
        processWorkerExit(w, completedAbruptly);
    }
}
```

#### 结束任务板块分析

**shutdown**

- 原理
  - 遍历workers集合，每个worker线程去尝试获取锁，获取到锁证明是空闲线程，可以中断
- 注意
  - shutdown调用完不是立刻结束线程池的,需等待队列中的任务执行完.如果需要判断线程池状态需配合awaitTermination方法查询
- 疑问
  - 在执行任务的worker线程最后怎么关闭的
    - runWorker方法获取不到新任务时会执行processWorkerExit方法
    - processWorkerExit会尝试tryTerminate去判断是否需关闭线程池,最终任务数为0并且工作线程未0时关闭线程池,执行线程池状态修改

**shutdownNow**

原理

- 遍历workers集合去中断线程,并返回未执行的任务
- 直接设置线程池状态为STOP
- 最后调用tryTerminate去关闭线程池设置状态为TERMINATED