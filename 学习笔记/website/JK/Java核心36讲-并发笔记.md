## Java核心36讲-并发笔记

#### 并发包提供的线程安全常见集合接口

![](https://s2.ax1x.com/2019/06/22/ZpKVmQ.png)

#### 并发包的线程安全队列

![](https://s2.ax1x.com/2019/06/22/ZpKnkn.png)



#### 并发包中的 ConcurrentLinkedQueue 和 LinkedBlockingQueue 有什么区别？

- Concurrent 类型基于 lock-free，在常见的多线程访问场景，一般可以提供较高吞吐量。
- 而 LinkedBlockingQueue 内部则是基于锁，并提供了 BlockingQueue 的等待性方法。



#### synchronized 和 ReentrantLock 有什么区别？

重入锁支持中断,支持带超时获取锁,支持尝试性获取锁,支持公平性竞争锁,需耗费额外的堆内存



#### 线程安全基本特性

- 原子性
- 可见性(volatile或加锁)
- 有序性(避免指令重排)

#### JVM的Synchronized基于Monitor机制的三种不同的实现锁

- 偏斜锁(偏斜锁适用于并发数少的场景,关闭[偏向锁`-XX:-UseBiasedLocking`)
- 轻量级锁
- 重量级锁

#### 线程状态

![](https://s2.ax1x.com/2019/06/20/Vx9JRP.png)

#### ThreadLocal

**定义**

 Java 提供的一种保存线程私有信息的机制，因为其在整个线程生命周期内有效，所以可以方便地在一个线程关联的不同业务模块之间传递信息，比如事务 ID、Cookie 等上下文相关信息。 

数据存储于线程相关的 ThreadLocalMap，其内部条目是弱引用,所以最佳实践是一定要自己负责remove，并且不要和线程池配合，因为 线程池的worker线程往往是不会退出的

#### 死锁

概念 : 死锁就是占着资源的双方互相等着对方释放资源的过程.

**死锁形成的条件**

- 互斥资源
- 请求与保持条件
- 循环等待条件
- 不剥夺条件

**破坏死锁**

 1.释放资源 -->sqlserver数据库
 2.尝试使用定时锁 并发包的tryLock
 3.避免一个锁同时占用两个资源



**死锁分析**

`jstack  pid | grep [16进制的线程号]`

查看到线程状态在阻塞`BLOCKER,`并显示等待锁,如下jstack日志显示

![](https://s2.ax1x.com/2019/06/21/VzSugf.png)

#### Java 内存模型中的 happen-before

Happen-before 关系，是 Java 内存模型中保证多线程操作可见性的机制。利用强制将cpu上某核的缓存数据刷新到任意核都能访问到的主内存来实现可见性.

![](https://s2.ax1x.com/2019/06/23/ZCB47d.png)

#### 集合类

#### LinkedHashMap保存插入顺序的原理

```java
//参考 https://segmentfault.com/a/1190000012964859
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> { 
        //内部定义了一个Entry来继承Node,并定义了前后节点,基本思路很清晰,问题在于在哪里维持的这份顺序
        static class Entry<K,V> extends HashMap.Node<K,V> {
            Entry<K,V> before, after;
            Entry(int hash, K key, V value, Node<K,V> next) {
                super(hash, key, value, next);
            }
        }

        //维持顺序是在添加的时候维持的,jdk1.8主要实现是在LinkedHashMap覆盖了父类的newNode方法

        // HashMap 中实现
        public V put(K key, V value) {
            return putVal(hash(key), key, value, false, true);
        }

        // HashMap 中实现
        final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            if ((tab = table) == null || (n = tab.length) == 0) {...}
            // 通过节点 hash 定位节点所在的桶位置，并检测桶中是否包含节点引用
            if ((p = tab[i = (n - 1) & hash]) == null) {...}
            else {
                Node<K,V> e; K k;
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                else if (p instanceof TreeNode) {...}
                else {
                    // 遍历链表，并统计链表长度
                    for (int binCount = 0; ; ++binCount) {
                        // 未在单链表中找到要插入的节点，将新节点接在单链表的后面
                        if ((e = p.next) == null) {
                            //重点在于这里
                            p.next = newNode(hash, key, value, null);
                            if (binCount >= TREEIFY_THRESHOLD - 1) {...}
                            break;
                        }
                        // 插入的节点已经存在于单链表中
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        p = e;
                    }
                }
                if (e != null) { // existing mapping for key
                    V oldValue = e.value;
                    if (!onlyIfAbsent || oldValue == null) {...}
                    afterNodeAccess(e);    // 回调方法，后续说明
                    return oldValue;
                }
            }
            ++modCount;
            if (++size > threshold) {...}
            afterNodeInsertion(evict);    // 回调方法，后续说明
            return null;
        }

        // HashMap 中实现
        Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
            return new Node<>(hash, key, value, next);
        }

        // LinkedHashMap 中覆写
        Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
            LinkedHashMap.Entry<K,V> p =  new LinkedHashMap.Entry<K,V>(hash, key, value, e);
            // 将 Entry 接在双向链表的尾部
            linkNodeLast(p);
            return p;
        }

        // LinkedHashMap 中实现
        private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
            LinkedHashMap.Entry<K,V> last = tail;
            tail = p;
            // last 为 null，表明链表还未建立
            if (last == null)
                head = p;
            else {
                // 将新节点 p 接在链表尾部
                p.before = last;
                last.after = p;
            }
        }
}    
```

