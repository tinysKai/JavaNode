## 使用CountDownLatch和CyclicBarrier优化项目

####  需求

对账系统的处理逻辑是首先查询订单，然后查询派送单，之后对比订单和派送单，将差异写入差异库。

![068418bdc371b8a1b4b740428a3b3ffe.png](http://ww1.sinaimg.cn/large/8bb38904gy1gasy7bs8dkj20vq0hen0j.jpg)

原处理如下,但速度较慢,需优化

```java
while(存在未对账订单){
  // 查询未对账订单
  pos = getPOrders();
  // 查询派送单
  dos = getDOrders();
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
} 
```

#### 并行查询数据库

由于两个数据库查询是无依赖的,因此使用多线程来并发查询.但后面的对账又依赖于这两个查询,故可使用CountDownLatch来执行主线程等待.

```java
// 创建2个线程的线程池
Executor executor = Executors.newFixedThreadPool(2);
while(存在未对账订单){
  // 计数器初始化为2
  CountDownLatch latch = new CountDownLatch(2);
  // 查询未对账订单
  executor.execute(()-> {
    pos = getPOrders();
    latch.countDown();
  });
  // 查询派送单
  executor.execute(()-> {
    dos = getDOrders();
    latch.countDown();
  });
  
  // 等待两个查询操作结束
  latch.await();
  
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
}
```

####  将数据库查询与对账操作并行化

再细想业务场景,其实在对账的时候,是可以准备下一次的对账数据的.即当前批次的对账操作是可以跟下一批次的数据查询并行操作的.而此时这个模型就有点类似于`生产者-消费者`模型了,数据库查询的订单和派送单作为生产者的数据供对账这个消费者来消费.

此处可使用两个队列来分别保存订单队列以及派送单队列,并且保持同一批次的订单的下标一致,如下图

![22e8ba1c04a3bc2605b98376ed6832da.png](http://ww1.sinaimg.cn/large/8bb38904gy1gasygypzjij20vq0clwfo.jpg)

此处使用CyclicBarrier来实现回调触发,即当两个查询完成后触发对账,代码实例如下

```java
// 订单队列
Vector<P> pos;
// 派送单队列
Vector<D> dos;
// 执行回调的线程池,只有一个固定线程的原因是保存对账时获取的下标一致(即remove的操作一次只有一个) 
Executor executor = Executors.newFixedThreadPool(1);
final CyclicBarrier barrier =
  new CyclicBarrier(2, ()->{
    executor.execute(()->check());
  });
  
void check(){
  P p = pos.remove(0);
  D d = dos.remove(0);
  // 执行对账操作
  diff = check(p, d);
  // 差异写入差异库
  save(diff);
}
  
//CyclicBarrier 的计数器有自动重置的功能，当减到 0 的时候，会自动重置你设置的初始值。
void checkAll(){
  // 循环查询订单库
  Thread T1 = new Thread(()->{
    while(存在未对账订单){
      // 查询订单库
      pos.add(getPOrders());
      // 等待
      barrier.await();
    }
  });
  T1.start();  
  // 循环查询运单库
  Thread T2 = new Thread(()->{
    while(存在未对账订单){
      // 查询运单库
      dos.add(getDOrders());
      // 等待
      barrier.await();
    }
  });
  T2.start();
}
```

#### 总结

CountDownLatch 主要用来解决一个线程等待多个线程的场景,而 CyclicBarrier 是一组线程之间互相等待。

CyclicBarrier 的计数器是可以循环利用的，而且具备自动重置的功能，一旦计数器减到 0 会自动重置到你设置的初始值。除此之外，CyclicBarrier 还可以设置回调函数

