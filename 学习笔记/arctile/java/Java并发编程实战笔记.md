## 死锁学习笔记

### 死锁

#### 定义

一组互相竞争资源的线程因互相等待，导致“永久”阻塞的现象。

#### 发生的条件

+ 互斥
+ 占有且等待
+ 不剥夺
+ 循环等待



#### 预防死锁

互斥这个条件我们没有办法避免，因为我们用锁为的就是互斥。所以我们一般得从其他三个来避免死锁.预防死锁主要是破坏三个条件中的一个.

**破坏占有且等待可由一次性申请所有资源来解决**

因使用while轮询方式判断,若并发高或者冲突多,则调用成本高,因此一般不推荐

```java
class Allocator {
  private List<Object> als = new ArrayList<>();
  // 一次性申请所有资源
  synchronized boolean apply(Object from, Object to){
    if(als.contains(from) || als.contains(to)){
      return false;  
    } else {
      als.add(from);
      als.add(to);  
    }
    return true;
  }
  // 归还资源
  synchronized void free(Object from, Object to){
    als.remove(from);
    als.remove(to);
  }
}

class Account {
  // actr应该为单例
  private Allocator actr;
  private int balance;
  // 转账
  void transfer(Account target, int amt){
    // 一次性申请转出账户和转入账户，直到成功
    while(!actr.apply(this, target))
      ；
    try{
      // 锁定转出账户
      synchronized(this){              
        // 锁定转入账户
        synchronized(target){           
          if (this.balance > amt){
            this.balance -= amt;
            target.balance += amt;
          }
        }
      }
    } finally {
      actr.free(this, target)
    }
  } 
}
```

可使用等待通知方式修改上面的调用

```java

class Allocator {
  private List<Object> als;
  // 一次性申请所有资源
  synchronized void apply(Object from, Object to){
    // 经典写法
    while(als.contains(from) || als.contains(to)){
      try{
        wait();
      }catch(Exception e){
      }   
    } 
    als.add(from);
    als.add(to);  
  }
  // 归还资源
  synchronized void free(Object from, Object to){
    als.remove(from);
    als.remove(to);
    notifyAll();
  }
}

class Account {
  // actr应该为单例
  private Allocator actr;
  private int balance;
  // 转账
  void transfer(Account target, int amt){
      //获取锁,不成功则wait等待并释放锁,直到被重新通知获取到锁
      actr.apply(this, target));

      try{
          // 锁定转出账户
          synchronized(this){              
            // 锁定转入账户
            synchronized(target){           
              if (this.balance > amt){
                this.balance -= amt;
                target.balance += amt;
              }
             }
           }
        } finally {
          actr.free(this, target)
        }
  } 
}
```



**破坏不剥夺**

可使用Lock的tryLock方法

**破坏循环等待**

按照一定规则保证加锁顺序,即不会出现循环

```java
class Account {
  private int id;
  private int balance;
      
  // 转账
  void transfer(Account target, int amt){
    Account left = this        
    Account right = target;    
    if (this.id > target.id) { 
      left = target;           
      right = this;            
    }                          
    // 锁定序号小的账户
    synchronized(left){
      // 锁定序号大的账户
      synchronized(right){ 
        if (this.balance > amt){
          this.balance -= amt;
          target.balance += amt;
        }
      }
    }
  } 
}
```

