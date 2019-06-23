# AQS源码分析

## 概念

#### 定义

`AbstractQueuedSynchronizer`提供了一个基于FIFO队列，可以用于构建锁或者其他相关同步装置的基础框架。

#### 结构

1. 用 `volatile `修饰的整数类型的 state 状态，用于表示同步状态，提供 getState 和 setState 来操作同步状态
2. 提供了一个 FIFO 等待队列，实现线程间的竞争和等待，这是 AQS 的核心
3. AQS 内部提供了各种基于 CAS 原子操作方法，如 compareAndSetState 方法，并且提供了锁操作的acquire和release方法

#### 流程

**独占式加锁**

![](https://s2.ax1x.com/2019/06/22/Z983sP.png)

**共享式加锁**

![](https://s2.ax1x.com/2019/06/22/Z9tYXn.png)



## 基本代码

#### 基本数据结构

```java
//节点 
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
    	//定义为共享模式
        static final Node SHARED = new Node();
    
        /** Marker to indicate a node is waiting in exclusive mode */
    	//定义为排他模式
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
    
        /** waitStatus value to indicate successor's thread needs unparking */
    	//表示当前节点的后继节点包含的线程需要运行，也就是unpark
        static final int SIGNAL    = -1;
    
        /** waitStatus value to indicate thread is waiting on condition */
    	//表示当前节点在等待condition，也就是在condition队列中
        static final int CONDITION = -2;
    
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
    	//表示当前场景下后续的acquireShared能够得以执行
        static final int PROPAGATE = -3;
     
        volatile int waitStatus;
        
        volatile Node prev;
     
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

    	//存储condition队列中的后继节点
        Node nextWaiter;

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {}

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

#### 基本属性&方法

```java
//表示当前持有锁的线程
private transient volatile Node head;

//链表队列尾节点
private transient volatile Node tail;

//节点状态
private volatile int state;

```

#### 基本方法

```java
//添加节点
private Node addWaiter(Node mode) {
    //将当前线程保存到节点中
    Node node = new Node(Thread.currentThread(), mode);
    //将节点保存到链表末端
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //利用CAS添加节点到链表末端
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //失败则进入循环体一直追加直至成功
    enq(node);
    return node;
}

//添加节点直至成功
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

//设置头节点,能将线程设置为null的原因是此时已等于获取到锁了,线程不阻塞了能走下去了
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```



#### 状态控制的基本方法

```java
//state利用volatile保证了可见性
protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}

//利用CAS来操作状态的变更
protected final boolean compareAndSetState(int expect, int update) {
    //stateOffset是在AQS类的反射offset
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### 子类覆盖状态的基本方法

```java
//返回是否是独占模式
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}

//--------------独占模式需实现的方法------------------
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}


//---------------共享模式需实现的方法--------------
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

```



## 并发包实例

#### 重入锁实现独占例子

```java
public class ReentrantLock implements Lock{
 
    private final Sync sync;

	//公平锁与非公平锁的基类   
    abstract static class Sync extends AbstractQueuedSynchronizer {

        //加锁
        abstract void lock();

        //尝试获取锁,真正的tryAcquire由其子类去重写
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        //实现AQS并实现独占模式时需重写的方法,释放锁
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        //实现AQS并实现独占模式时需重写的方法,判断是否拥有锁
        protected final boolean isHeldExclusively() 
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

    	//是否加锁
        final boolean isLocked() {
            return getState() != 0;
        }
    }

	//非公平锁,默认实现,跟syn一样
	static final class NonfairSync extends Sync {
        
        //加锁,基本原理是将state值更新为1,并设置锁的占有者为当前线程
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //基于AQS父类的方法
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

	//公平锁
	static final class FairSync extends Sync {
        
        final void lock() {
            acquire(1);
        }

        
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
}
```

#### 共享锁例子

```java
public class CountDownLatch {
  
    //内部实现的AQS
    private static final class Sync extends AbstractQueuedSynchronizer {

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

  
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    
     public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
    
    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }
}    
```



## AQS原理解析

### 状态分析

| 状态名    | 状态值 | 含义                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| CANCELLED | 1      | 取消状态,表明节点已经等待超时或者已经被中断了，这时需要将其从等待队列中删除 |
| SIGNAL    | -1     | 声明后继节点需唤醒                                           |
| CONDITION | -2     | 声明线程在等待条件                                           |
| PROPAGATE | -3     | 状态需向后冒泡传播，表示 releaseShared 需被传播给后续节点，仅在共享锁模式下使用 |
| 默认值    | 0      | 默认值                                                       |



### 独占

#### acquire分析

**总结**

尝试获取锁成功,不用进入队列,直接修改state状态,以及锁拥有的线程.

若获取锁失败,则进入队列,挂起线程,等待唤起时的再次获取锁尝试.

```java
 public final void acquire(int arg) {
     if (!tryAcquire(arg) &&		//tryAcquire由子类实现
         acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //添加到队列
         selfInterrupt();			//中断线程
}

//尝试获取锁,获取失败则阻塞
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前节点的上一个节点
            final Node p = node.predecessor();
            //如果上一个节点时头结点并且尝试获取锁
            if (p == head && tryAcquire(arg)) {
                //获取锁成功则设置当前节点为头结点,前节点设置为null等待GC回收
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //获取锁失败,需进入挂起逻辑
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

/**
 *  根据 pred 节点状态来判断当前节点是否可以挂起，如果该方法返回 false，
    那么挂起条件还没准备好，就会重新进入acquireQueued的自旋体，重新进行判断。
    如果返回 true，那就说明当前线程可以进行挂起操作了，那么就会继续执行挂起。
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果 pred 节点为 SIGNAL 状态，返回true，说明当前节点需要挂起
    if (ws == Node.SIGNAL)
        return true;
   // 如果ws > 0,说明节点状态为CANCELLED，需要从队列中删除
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 如果是其它状态，则操作CAS统一改成SIGNAL状态
   		// 由于这里waitStatus的值只能是0或者PROPAGATE，所以我们将节点设置为SIGNAL，从新循环一次判断
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

//让线程阻塞,并返回中断状态
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```



#### release分析

```java
//释放锁
public final boolean release(int arg) {
    //尝试释放锁,由子类去实现
    if (tryRelease(arg)) {
        Node h = head;
        //当前head节点不为空且其对应的状态不为默认值0,则唤起等待者
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}


//唤起挂起的线程
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    //非取消状态下,将状态更新为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
	//获取下一个节点
    Node s = node.next;
    //下一个节点为空或者下一个节点的状态为取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从尾结点遍历,获取Node的下一个后继非取消节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //唤起链表head节点的下一个节点的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```



### 共享

**总结**

每次都先尝试获取锁,若成功则获取到锁,否则如队列

入队列时判断自己的前节点是否是头节点,若是头节点则再次尝试获取锁.

若获取锁成功,则唤起下一share节点来获取锁(并依此类推唤起下下个节点)

#### 获取锁分析

`acquireSharedInterruptibly`

```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
  	//提供中断支持
    if (Thread.interrupted())
        throw new InterruptedException();
    //statez状态为0时,返回1,否则返回-1
    if (tryAcquireShared(arg) < 0) //所以state为0时,if条件不成立
        //state不为0时,执行以下方法
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)  throws InterruptedException {
    //添加共享节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //前节点是head节点则尝试获取锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //获取锁成功则设置头节点并冒泡
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }      
            //挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//冒泡式唤起share节点
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
     
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                //唤醒其它等待信号的线程
                doReleaseShared();
        }
}

//释放head节点的后续share节点
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        //判断至少存在两个节点
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //等待通知信号的节点
            if (ws == Node.SIGNAL) {
                //CAS更新状态
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //唤醒后继节点线程
                unparkSuccessor(h);
            }
            //非SIGNAL将其设置为PROPAGATE节点,当当前队列的最后一个节点成为了头节点
            else if (ws == 0 &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //只有当前head节点不变才退出循环,否则一直唤醒后继节点,所以可以同时间多个线程来执行唤醒后继节点
        if (h == head)                   // loop if head changed
            break;
    }
}

//唤起后继节点
private void unparkSuccessor(Node node) {
 
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

   
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

#### 释放锁分析

```java
public final boolean releaseShared(int arg) {
    //尝试释放锁
    if (tryReleaseShared(arg)) {
        //方法代码如上,在释放锁时也需判断是否需释放后继节点
        doReleaseShared();
        return true;
    }
    return false;
}


```



## 可参考文章

[doReleaseShared详解](https://segmentfault.com/a/1190000016447307)