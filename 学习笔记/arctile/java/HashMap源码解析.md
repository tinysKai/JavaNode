## HashMap源码解析

#### 基本方法

```java
//jdk1.8中的节点数据结构 
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
}


//key的hash取值
static final int hash(Object key) {
    int h;
    //key.hashCode()为哈希算法，返回初始哈希值返回int型散列值
    //将h的高16位与低16位异或取值
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}


//返回最接近cap而且比cap大的2的N次方的数
//原理就是将低位的1全部或成1,然后加1,低位全部进位变为0,,直到最高位进1
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

//基于数组的结构
transient Node<K,V>[] table;
//数组元素大小
transient int size;
//hashmap结构变化计数,用于来在遍历时fail-fast
transient int modCount;
//The next size value at which to resize (capacity * load factor).
int threshold;
```



#### hash与取模分析

在数组长度length为2的N次方的情况下,`hash % length`的数值等于`hash & (length-1)`.

然后hashmap为了让数值分散,使用了扰乱函数,将高16位与低16位坐异或,得到一个新的hash值

![https://ws3.sinaimg.cn/large/005BYqpggy1g3zua6i5x0j30gb098dia.jpg](https://s2.ax1x.com/2019/06/13/VhD0FU.png)

#### 关键方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

 
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //红黑树寻找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //链表遍历寻找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}


public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

 Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
 }


/**
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        //第一次添加节点时,初始化数组
        n = (tab = resize()).length;
    
    
    if ((p = tab[i = (n - 1) & hash]) == null)
        //对应数组无值时,创建新节点并赋值给table
        tab[i] = newNode(hash, key, value, null);
    else {
        //数组有对应值时,可能是链表或者是红黑树
        Node<K,V> e; K k;
        //如果对应Node位置的第一个节点hash值相同而且key相同则覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            //红黑树结构,则添加节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //链表结构
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    //遍历到下一个节点无数据则创建节点
                    p.next = newNode(hash, key, value, null);
                    //转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历得到相同的key则停止遍历跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //若存在旧值则直接覆盖返回旧值并返回
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //若木有旧值则,modCount添加1
    ++modCount;
    //判断当前size添加1是否需要扩容,所以是提前扩容
    if (++size > threshold)
        resize();
    //给LinkHashedMap覆盖的方法,可实现删除最先添加的元素
    afterNodeInsertion(evict);
    return null;
}
```

#### 扩容

```java
//初始化或扩容两倍的大小,JDK1.8的扩容已优化避免了死循环
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
    	//原数组大小大于0,空参数构造方法只初始化加载因子
        if (oldCap > 0) {
            //数组长度最大时无法扩容,直接返回
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //在扩容前数组大小大于默认大小并且翻倍后小于最大值时,数组长度翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&oldCap >= DEFAULT_INITIAL_CAPACITY)
                //扩容触发点翻倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            //新数组长度等于threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //第一次默认空构造方法会走这里
            newCap = DEFAULT_INITIAL_CAPACITY; //数组长度为16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
    
        //声明一个新的Node数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    	//数组转移,但还没赋值,有并发问题
        table = newTab;
    	//数据转移
        if (oldTab != null) {
            //遍历原Node数组,将每个槽位的链表/红黑树数据转移
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //先置空
                    oldTab[j] = null;
                    //若Node节点无下一节点,只有单节点时则直接重新散列
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                         //树状
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 如果 table[j] 后是一个链表 ，将原链表拆分为两条链，分别放到newTab中 
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //重点在这里,看下图的解释
                            if ((e.hash & oldCap) == 0) {
                                //低位链表
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                //注意保持顺序是正向保持的,而jdk1.7是反向保存的
                                loTail = e;
                            }
                            else {
                                //高位链表
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //之所以能将单链表拆分为两个新链表的是因为数组大小都是2的N次方
                        //每次扩容时,原本链条的近似一半数据都在原链表位置,近似一半在新链表
                        if (loTail != null) {
                            //j的链表
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            //扩容后原本j的链表变为j + oldCap
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

#### resize中链表拆分分析

前提 : `单链表拆分为两个新链表的是因为数组大小都是2的N次方`

![https://ws3.sinaimg.cn/large/005BYqpggy1g41nurzs9vj30y408y0uf.jpg](https://s2.ax1x.com/2019/06/15/VItXdO.png)

(a) 是未扩容时， `key1` 和 `key2` 得出的 hash & (n - 1) 均为 5。
(b) 是扩容之后，`key1` 计算出的 newTab 角标依旧为5，但是 `key2` 由于扩容， 得出的角标加了16，即21， 16是oldTab的length，再来看`e.hash & oldCap`，oldCap.length即n 本身为 `0000 0000 0000 0000 0000 0000 0001 0000` ，**这个位与运算可以得出扩容后哪些key在扩容新增位是1，哪些是0，一个位运算替换了`rehash`过程**



#### 并发问题分析

- 扩容时新空数组赋值给Node数组时数据访问丢失

- 并发扩容时访问死循环(jdk1.8版本已优化不会出现死循环)

  ```java
  //主要问题在于resise时链表的写入顺序是反向写入的,每次都是直接操作数组Node槽节点.
  //jdk1.7参考这篇文章 `https://juejin.im/post/5a66a08d5188253dc3321da0`
  //而jdk8是采用正向的顺序来存储链表的,所以不会有死循环问题,但依旧有数据访问丢失问题
  void resize(int newCapacity) {
      Entry[] oldTable = table;
      int oldCapacity = oldTable.length;
      ...
  
      Entry[] newTable = new Entry[newCapacity];
      ...
      transfer(newTable, rehash);
      table = newTable;
      threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
  }
  
  
  void transfer(Entry[] newTable, boolean rehash) {
      int newCapacity = newTable.length;
      for (Entry<K,V> e : table) {
          while(null != e) {
              Entry<K,V> next = e.next;
              if (rehash) {
                  e.hash = null == e.key ? 0 : hash(e.key);
              }
              int i = indexFor(e.hash, newCapacity);
              e.next = newTable[i];
              newTable[i] = e;
              e = next;
          }
      }
  }
  ```

