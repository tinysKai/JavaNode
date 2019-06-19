## ConcurrentHashMap源码解析

#### JDK7版本的设计简要概括

基于继承了`ReentrantLock`的`Segment`实现分段锁.



#### 基本属性

```java
transient volatile Node<K,V>[] table;
//仅在扩容更新的时候不为空,扩容时会慢慢把数据移动到这个数组
private transient volatile Node<K,V>[] nextTable;
//元素个数值
private transient volatile long baseCount;
//表示已经分配给扩容线程的table数组索引位置,主要用来协调多个线程间迁移任务的并发安全.
//扩容在声明为新数组长度为2倍后会设置此值为原来的数组大小n
private transient volatile int transferIndex;
/**
   * The number of bits used for generation stamp in sizeCtl.
   * Must be at least 6 for 32bit arrays.
*/
private static int RESIZE_STAMP_BITS = 16;
/**
  * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;


/**
  + sizeCtl > 0时可分为两种情况:
		- 未初始化时,sizeCtl表示初始容量.
		- 初始化后表示扩容的阈值,为当前数组长度length*0.75
		
  + sizeCtl = -1: 表示正在初始化或者扩容阶段.
  
   + sizeCtl < -1 : sizeCtl承担起了扩容时标识符(高16位)和参与线程数目(低16位)的存储
		- 在addCount和helpTransfer的方法代码中,如果需要帮助扩容,则会CAS替换为sizeCtl+1
		- 在完成当前扩容内容,且没有再分配的区域时,线程会退出扩容,此时会CAS替换为sizeCtl-1
*/
private transient volatile int sizeCtl;

private static final sun.misc.Unsafe U;
private static final long SIZECTL;
private static final long BASECOUNT;
private static final long ABASE;
private static final int ASHIFT;

//静态方法对Unsafe操作的一些属性进行读取赋值
static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            SIZECTL = U.objectFieldOffset(k.getDeclaredField("sizeCtl"));
            BASECOUNT = U.objectFieldOffset(k.getDeclaredField("baseCount"));
            Class<?> ak = Node[].class;
            // 获取数组中第一个元素的偏移地址
            ABASE = U.arrayBaseOffset(ak);
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

#### 关键属性sizeCtl说明

- 未初始化： 
  - sizeCtl=0：表示没有指定初始容量。
  - sizeCtl>0：表示初始容量。

- 初始化中：
  - sizeCtl=-1,标记作用，告知其他线程，正在初始化
- 正常状态：
  - sizeCtl=0.75n ,扩容阈值
- 扩容中:
  - sizeCtl < 0 : 表示有其他线程正在执行扩容
  - sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2    : 表示此时只有一个线程在执行扩容

**状态流转图**

![https://ws3.sinaimg.cn/large/005BYqpggy1g434drxnuhj30qj06i0uf.jpg](https://s2.ax1x.com/2019/06/16/VTq8ER.png)



#### 并发辅助方法

```java
//获得在i位置上的Node节点
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

//cas更新Node节点的值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
//利用volatile方法设置节点位置的值
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```



#### 基本数据结构

```java
 static class Node<K,V> implements Map.Entry<K,V> {
     final int hash;
     final K key;
     //注意val的可见性
     volatile V val;
     //注意可见性
     volatile Node<K,V> next;
     
     //不允许执行set值操作
     public final V setValue(V value) {
            throw new UnsupportedOperationException();
     }
     
     //查找方法,子类会覆盖此方法
     Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
    }
 }   

//转发节点,仅仅存活在扩容阶段,作为一个标记节点放在桶的首位,并且指向是nextTable(扩容的中间数组)
static final class ForwardingNode<K,V> extends Node<K,V> {
    //关联了nextTable,扩容期间可以通过find方法，访问已经迁移到了nextTable中的数据
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        //hash值设置为-1,其它属性都为空
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
    
    //通过此方法，访问被迁移到nextTable中的数据
     Node<K,V> find(int h, Object k) {
           //...
     }
}    
```



#### 基本方法

```java
//初始化数组
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //小于0代表有其它线程在进行初始化,自旋等待
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //利用CAS将`sizeCtl`的值设置为-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            //此方法进来一定会执行下面的方法一次并且只执行一次
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //由构造方法确保sc值肯定是大于等于DEFAULT_CAPACITY的2次方的数字
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //sc的值为 数组大小 - 1/4数组大小 = 3/4数组大小,是扩容的阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

//hash干扰函数,将高16位与低16位异或后,确保非负数
static final int spread(int h) {
     return (h ^ (h >>> 16)) & HASH_BITS;
}

//resizeStamp获取扩容时的一个标记
//numberOfLeadingZeros是Integer的方法,返回的是n的32位二进制形式前面的0的个数
//结果等于 获取n的有效位之前的0的个数加上2的15次方
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```



#### 关键方法

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    //Node数组不为空并且哈希值对应的槽位值不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&(e = tabAt(tab, (n - 1) & h)) != null) {
        //首节点判断
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 元素hash < 0的情况有以下三种:
        // 1. 数组正在扩容，Node的实际类型是ForwardingNode
        // 2. 节点为树的root节点，TreeNode
        // 3. 暂时保留的Hash, Node
        // 不同的Node都会调用各自的find()方法
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //链表遍历
        while ((e = e.next) != null) {
            if (e.hash == h &&((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    //查找不到返回null
    return null;
}


public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    //键值都不允许为null
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //初始化数组
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //对应的Node节点无值
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //添加节点,若CAS操作失败会重新进入循环
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //表示正在扩容
        else if ((fh = f.hash) == MOVED)
            //帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //syn锁的粒度为数组槽位,可为链表或红黑树
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //大于0表示链表
                    if (fh >= 0) {
                        //链表中的元素个数统计
                        binCount = 1;
                        //遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //发现已有KEY值,则覆盖原有值
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //无后续节点时则直接创建新节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,value, null);
                                break;
                            }
                        }
                    }
                    //树状
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            
            if (binCount != 0) {
                //转化为树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //返回旧值
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 添加元素计数,并在binCount大于0时检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

#### 扩容

**原理**

java8采用多线程扩容.整个扩容过程，通过CAS设置sizeCtl，transferIndex等变量协调多个线程进行**并发扩容**。

![https://ws3.sinaimg.cn/large/005BYqpggy1g434meigk3j30j50mrjt8.jpg](https://s2.ax1x.com/2019/06/16/VTLGLQ.png)

```java
//增加map的元素个数并检查是否需要扩容
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果计数盒子不是空 或者
    // 如果修改 baseCount 失败
    if ((as = counterCells) != null ||
        // CAS增加baseCount
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果计数盒子是空（尚未出现并发）
        // 如果随机取余一个数组位置为空 或者
        // 修改这个槽位的变量失败（出现并发了）
        // 执行 fullAddCount 方法。并结束
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    //检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
         // 1. 如果元素总数大于等于sizeCtl,表示达到了扩容阈值
         // 2. tab数组不能为空,已经初始化
         // 3. table.length小于最大容量,有扩容空间
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 根据数组长度获取一个扩容标志
            int rs = resizeStamp(n);
            //sc小于0表示有其它线程在扩容
            if (sc < 0) {
                // 如果sc的低16位不等于rs,表示标识符已经改变.				
                // 如果nextTable为空,表示扩容已经结束
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                 // CAS替换sc值为sc+1,成功则开始添加此线程来扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //调用transfer开始扩容,此时nextTable已经指定
                    transfer(tab, nt);
            }
            // `sc > 0`表示数组此时并不在扩容阶段,更新sizeCtl并开始扩容
            //注意此时的sizeCtl置为`rs << RESIZE_STAMP_SHIFT) + 2`
            else if (U.compareAndSwapInt(this, SIZECTL, sc,(rs << RESIZE_STAMP_SHIFT) + 2))
                // 调用transfer,nextTable待生成
                transfer(tab, null);
            s = sumCount();
        }
    }
}


final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        
        while (nextTab == nextTable && table == tab &&(sc = sizeCtl) < 0) {+
            // 一. 不需要帮助扩容的情况
            // 1. sc的高16位不等于rs (第一个扩容的线程，执行transfer方法之前，会设置 sizeCtl = `(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)`)
            // 2. sc等于rs+1(触发扩容的线程已退出,扩容已经完成)
            // 3. sc等于rs+MAX_RESIZERS
            // 4. transferIndex<=0, 因为扩容时会分配并减去transferIndex,小于0时表示数组的区域已分配完毕
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //添加扩容线程,所以sizectl + 1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

```

```java
//扩容的核心方法
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //计算每次迁移的node个数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            //初始化数组,数组长度为旧数组长度的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        //设置迁移的起始坐标为数组的末尾
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //初始化迁移节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    
    //下面开始进行数组的迁移
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    //bound记录了当前批次的最后一个Node下标(逆序遍历)
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //每个进入的线程都会尝试去扩容stride个Node节点,除非扩容完成或节点分配完毕,分配完毕下次就不执行这个while循环
        while (advance) {
            int nextIndex, nextBound;
            //更新迁移索引i,最后一个else分支才能往前遍历迁移Node节点,正常批次的Node节点每次遍历都会执行此方法,并返回`advance = false`
            if (--i >= bound || finishing)
                advance = false;
            //表示所有Node节点已迁移分配完
            else if ((nextIndex = transferIndex) <= 0) {
                //transferIndex<=0表示已经没有需要迁移的hash桶，将i置为-1，线程准备退出
                i = -1;
                advance = false;
            }
            //当迁移完bound这个桶后，尝试更新transferIndex，，获取下一批待迁移的hash桶
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        
        //分配完stride个Node节点后,正常情况下是执行最后一个else分支来连续执行数据迁移
        
        //退出transfer
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //迁移完成时设置值
            if (finishing) {
                //扩容保存的数组置空
                nextTable = null;
                //重新设置Node数组值为扩容后的值
                table = nextTab;
                //2n - 0.5n = 1.5n 等于原来的两倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            
          
            
            //将sizeCtl - 1表示增加多一个线程来执行扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
               /**
             	假设`(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)`值为val
                第一个扩容的线程，执行transfer方法之前，会设置 sizeCtl = val
                后续帮其扩容的线程，执行transfer方法之前，会设置 sizeCtl = sizeCtl+1
                每一个退出transfer的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1
                那么最后一个线程退出时：
                必然有sc == val，即 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT
               */
                
                //不相等，说明不到最后一个线程，直接退出transfer方法
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //重新进入循环会到上面的完成判断进入跳出
                finishing = advance = true;
                //最后退出的线程要重新check下是否全部迁移完毕
                i = n; // recheck before commit
            }
        }
        //迁移过程发现对应Node节点无数据,直接将其设置为迁移节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //表示正在迁移,感觉不会出现这种情况
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //单线程模式下的扩容,跟hashmap原理是一样的
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //设置拆分完的两条链表到新数组中
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        //标记此Node节点在原数组已迁移完成
                        setTabAt(tab, i, fwd);
                        //重新更新advanced为true
                        advance = true;
                    }
                    //树状
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```



#### 参考文章

[掘金文章的源码解读](https://juejin.im/post/5c40a1fa51882525ed5c4ac2#heading-2)

[扩容分析](https://www.jianshu.com/p/487d00afe6ca)



