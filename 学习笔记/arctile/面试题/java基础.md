>常见集合对比  

Arraylist
+ 基于数组,线性集合
+ 适合查找更新操作  
     - 数组查找 o(1)
+ 不适合添加删除操作
    -  添加到末尾,如果不需要扩容,O(1).需要扩容,调用Arrays.copyOf(elementData, newCapacity)扩容  
    - 添加到中间,需要移动(size-index)个元素  
    - remove(index),需移动size-index-1的元素
 
 LinkedList
+ 基于链表
+ 适合删除,插入                        
+ 不适合查找         
     
     
>谈一下HashMap

HashMap继承AbstractMap,而AbstractMap直接继承Object..相比于其它容器(如ArrayList,继承AbstractCollection之前先继承了AbstractList)是继承AbstractCollection再继承Object.  
基于哈希的散列到不同的桶(Entry),然后如果出现哈希碰撞的话再通过链表的方式来解决冲突.  

哈希碰撞的解决方案 : 
+ 链表
+ 再哈希


HashMap是允许key和value的值为null的...

关于寻找值的,都是先找到对应的Entry,然后再通过链表去寻找对应的值..
 ```
    Entry<K,V> e = table[indexFor(hash, table.length);
    if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
    return e.value;
```

关于扩容的部分
threshold = (int)(capacity * loadFactor);  
当map大小(size)达到threadhold时就扩容,默认的初始容量为16(也就是 Entry table = new Entry[16]),加载因子为0.75f..  
扩容的过程是 :  创建一个新的Entry数组 :  Entry[] newTable = new Entry[2 * oldTable.length]

     
>谈一下CurrentHashMap


      CurrentHashMap的实现(1.6版本)
      基于Segment的实现..Segment继承了ReentrantLock..然后Segment里面通过HashEntry来散列,再通过链表来解决哈希冲突...Segment用来实现分段锁..
      HashEntry的value值为volatile..确保了可见性...
           class HashEntry<K,V> {
              final K key;
              final int hash;
              volatile V value;
              final HashEntry<K,V> next;
           }
 
      isEmpty方法..通过遍历整个Segment[]..如果存在一个值大于0返回false,否则通过modcount来判断..
      size方法..先通过modCount来尝试不加锁的遍历..若两次遍历读取的大小一致,以及Segment的修改次数一致的话..则返回该值..否则加锁读取...
      containsValue类似试着方法..
      containsKey因为key是哈希的关键..所以可以直接判断
           public boolean containsKey(Object key) {
              int hash = hash(key.hashCode());
              return segmentFor(hash).containsKey(key, hash);
          }
 
      1.7版本引入了Unsafe类..
      1.8引入了解决哈希碰撞从链表可能变为红黑树...  
      

>深拷贝,浅拷贝  

     浅拷贝是指拷贝对象时仅仅拷贝对象本身（包括对象中的基本变量），而不拷贝对象包含的引用指向的对象。
     深拷贝不仅拷贝对象本身，而且拷贝对象包含的引用指向的所有对象。
     举例来说更加清楚：对象A1中包含对B1的引用，B1中包含对C1的引用。浅拷贝A1得到A2，A2中依然包含对B1的引用，B1中依然包含对C1的引用。
     深拷贝则是对浅拷贝的递归，深拷贝A1得到A2，A2中包含对B2（B1的copy）的引用，B2 中包含对C2（C1的copy）的引用。
     
     
>comparable接口和comparator接口实现比较的区别

     接口方法
        Comparable接口的方法：compareTo(Object o)
  
        Comparator接口的方法：compare(T o1, To2)
       --------------
      实现方式
         Comparable接口的实现：直接用需要进行排序的类实现即可。例如：public class User implements Comparable<User>
  
         Comparator接口的实现：需要新建一个类实现此接口。例如：public class UserComparator implements Comparator<User>
       --------------
      使用
         Comparable接口：Arrays.sort(List<T> list);
  
         Comparator接口的实现：需要新建一个类实现此接口。例如：public Comparator接口：Arrays.sort(List<T> list, Comparator<? super T> c);
         
         
         
>java中弱引用

      java中引用分为四种 :常见的强引用，软引用，弱引用，虚引用
      软引用 ： 只要ｊｖｍ有足够的内存，就不会回收软引用．．但当内存不足时，会清除软引用的内存．．适合做缓存．．
      弱引用 ： 如果垃圾回收线程（优先级别较低的线程）检查到弱引用对象的存在，则会回收该对象的内存．． 
         
         
>Java历代版本的变化  

  1.4 
  + 正则表达式
  + 日志类
  + NIO
  + XML解析器
  
 1.5 
  + 自动装箱
  + 泛型
  + 枚举
  + 可变长参数
  + foreach
  + 并发包
  
 java7
 + 泛型实例创建时的类型推断
 + fork join并发框架
 + switch支持
 + try-with-resources 语句
 + 同时捕获多个异常处理
 + Files Paths文件操作
         
         
>线程池的线程数的数量怎么定的?
            
      首先看下线程池的执行任务是IO密集型的(对应linux的sy),还是CPU密集型的(对应linux的us)...
      如果是IO密集型,则可以相应的调高线程池数量,保证当部分资源不可达时线程阻塞时,还有足够资源可用.
      若是CPU密集型的线程池,则可以减少相应的线程数,确保每个线程都充足的工作而且不平频繁的线程切换..
         
         
>HashMap的缺点
 
 	 不适合遍历 Entry数组+链接结构  
 
          map的常见种遍历
            for (String key : map.keySet()) ;
            Iterator<Map.Entry<String, String>> it = map.entrySet().iterator();
            for (Map.Entry<String, String> entry : map.entrySet()) ;
            
 	 浪费内存 
        1.entry数组的定义难免有空槽..
        2.使用链表解决哈希..链表有什么缺点呢？链表中的每一个节点，存储在内存条的不同物理位置上，这意味着对内存的访问是随机性的，而不是连续性的，而对内存条的随机访问的低效的..
        为什么对内存的随机访问的低效的？CPU从内存条中读数据，需要通过L2/L3 cache，如果cache能够命中所需要的数据，就不再从内存载入，否则L2/L3 cache是一次性将一片连续的内存（比如：64字节）载入，再将计算单元用到的部分交给计算单元，而这两种操作速度有10倍的差距，我的i7 CPU，L3 cache的只有区区4MB，只能保存内存中数据的极小一部分，所以如果对内存访问不是连续的，就需要频繁的换入换出内存数据
 
 
 >ConcurrentHashmap有什么缺点
 
       弱一致性..比如在map中清除所有元素时,在清除完某个segment之后(假设第i个segment),那么i之前的segment目前是没加锁的,是有可能被添加数据的..所以是弱一致性的..
       
       
         public void clear() {
              final Segment<K,V>[] segments = this.segments;
              for (int j = 0; j < segments.length; ++j) {
              Segment<K,V> s = segmentAt(segments, j);
              if (s != null)
                   s.clear();
              }
          }
        
          final void clear() {
              lock();
              try {
                   HashEntry<K,V>[] tab = table;
                   for (int i = 0; i < tab.length ; i++)
                        setEntryAt(tab, i, null);
                        ++modCount;
                        count = 0;
                   } 
                }finally {
                   unlock();
               }
          }
        
           
       
       单独的读写操作(get,put,putIfAbsent...)是完全线程安全且一致的，但是迭代时候则不是强一致的，迭代所遍历的不是迭代时刻的快照，而是各个segement的真是数据，
       所以迭代期间如有数据发生变更，如果变更的是已经遍历的segement则迭代过程不再感知这个变化，但如果变化发生在未遍历的segement，则本次迭代会感知到这个元素。
       另外一个基本常识，任意的组合操作，比如先get, 然后put，则不能保证强一致。
       此外这个类还有个特点，迭代过程中即使元素被修改，也不会抛出异常，其他一些集合则会抛出ConcurrentModificationException
 

 >hashmap的容量为什么是2的指数倍,为什么hashmap扩容是原数组长度乘以2?
 
       HashMap中的数据结构是数组+单链表的组合，我们希望的是元素存放的更均匀，
       最理想的效果是，Entry数组中每个位置都只有一个元素，这样，查询的时候效率最高，不需要遍历单链表，也不需要通过equals去比较K，而且空间利用率最大。
       那如何计算才会分布最均匀呢？我们首先想到的就是%运算，哈希值%容量=bucketIndex，SUN的大师们是否也是如此做的呢？我们阅读一下这段源码：
   
        static int indexFor(int h, int length) {
            return h & (length-1);
       }
  
      这里h是通过K的hashCode最终计算出来的哈希值，并不是hashCode本身，而是在hashCode之上又经过一层运算的hash值，length是目前容量。
      这块的处理很有玄机，与容量一定为2的幂环环相扣，当容量一定是2^n时，h & (length - 1) == h % length，它俩是等价不等效的，位运算效率非常高，
      实际开发中，很多的数值运算以及逻辑判断都可以转换成位运算，但是位运算通常是难以理解的，因为其本身就是给电脑运算的，运算的是二进制，而不是给人类运算的，人类运算的是十进制，这也是位运算在普遍的开发者中间不太流行的原因(门槛太高)。
      这个等式实际上可以推理出来，2^n转换成二进制就是1+n个0，减1之后就是0+n个1，如16 -> 10000，15 -> 01111，那根据&位运算的规则，都为1(真)时，才为1，那0≤运算后的结果≤15，假设h <= 15，那么运算后的结果就是h本身，h >15，运算后的结果就是最后三位二进制做&运算后的值，最终，就是%运算后的余数，我想，这就是容量必须为2的幂的原因。




>并发环境下用两个线程对ArrayList进行10000次添加操作的结果
+ 完全正常
+ 抛出数组越界(内部扩容时数组长度与数组个数一致性被破坏)
+ list的个数小于20000个(由于并发两个线程对同一个位置进行赋值导致)

>线程池参数
![线程池配置](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/poll.png)




>CopyOnWrite的缺点
+ 内存占用问题
+ 数据一致性问题(只确保最终一致性)

>各种map的null问题
![map的null解析](https://github.com/tinysKai/JavaNode/blob/master/image/article/java/map_null.png)


#### 超时时间
+ connected timeout(连接超时)
+ so timeout(读超时)  
*https://my.oschina.net/shipley/blog/715196*


#### 一个ArrayList在循环过程中删除，会不会出问题，为什么  
会抛出ConcurrentModificationException异常或NoSuchElementException异常.
因为ArrayList并非线程安全的,所以为了响应"fail-fast"使用了modcount机制来统计修改次数,若发现迭代过程有变化则快速失败
 