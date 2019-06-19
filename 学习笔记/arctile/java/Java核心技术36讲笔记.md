## Java核心技术36讲笔记

## 基础

#### getByte方法实战建议

*getBytes和String相关的转换时根据业务需要建议指定编码方式，如果不指定则看看JVM参数里有没有指定file.encoding参数，如果JVM没有指定，那使用的默认编码就是运行的操作系统环境的编码了，那这个编码就变得不确定了。常见的编码iso8859-1是单字节编码，UTF-8是变长的编码。*

intern方法目的是提示 JVM 把相应字符串缓存起来，以备重复使用。Intern是一种显式地排重机制,需明确调用,但每一个字符串都显式调用时非常麻烦的,甚至是代码污染.

#### 自动装箱与拆箱

javac替我们自动把装箱转换为 Integer.valueOf()，把拆箱替换为 Integer.intValue().而`valueOf`方法使用了缓存

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```





## JVM

**解释执行**

我们开发的 Java 的源代码，首先通过 Javac 编译成为字节码（bytecode），然后，在运行时，通过 Java 虚拟机（JVM）内嵌的解释器将字节码转换成为最终的机器码。但是常见的 JVM，比如我们大多数情况使用的 Oracle JDK 提供的 Hotspot JVM，都提供了 JIT（Just-In-Time）编译器，也就是通常所说的动态编译器，JIT 能够在运行时将热点代码编译成机器码，这种情况下部分热点代码就属于编译执行.



#### 类加载器与双亲委托机制

+ Bootstrp loader
+ ExtClassLoader  
+ AppClassLoader 
+ CustomerClassLoader

![](https://s2.ax1x.com/2019/06/09/VrnUZn.png)

双亲委托机制即当类加载器加载时先到父加载器查询是否有此类(父加载器会去其父加载器查询),若有则从父加载器加载,若无则加载到自身中,以便下次加载时缓存直接返回.

若怀疑应用存在引用（或 finalize）导致的回收问题,可使用JVM 自身便提供了明确的选项（PrintReferenceGC）去获取相关信息

```
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintReferenceGC

0.403: [GC (Allocation Failure) 0.871: [SoftReference, 0 refs, 0.0000393 secs]0.871: [WeakReference, 8 refs, 0.0000138 secs]0.871: [FinalReference, 4 refs, 0.0000094 secs]0.871: [PhantomReference, 0 refs, 0 refs, 0.0000085 secs]0.871: [JNI Weak Reference, 0.0000071 secs][PSYoungGen: 76272K->10720K(141824K)] 128286K->128422K(316928K), 0.4683919 secs] [Times: user=1.17 sys=0.03, real=0.47 secs] 

```





类加载过程 : 加载,验证,链接,初始化





#### NoClassDefFoundError 和 ClassNotFoundException有什么区别?

`NoClassDefFoundError `是error,而`ClassNotFoundException`是exception.前者是指要查找的类在编译时期是存在，运行期间却找不到该对象对应的类，这个时候就会导致NoClassDeFoundError错误。而后者指反射路径错误导致的异常,常见情况如反射` Class c = Class.forName("com.demo.Test")`.

#### Error & Exception

Throwable

+ exception

  + checked exception
    + IOException
    + InterruptedException
  + unchecked exception(RuntimeException及其子类)
    + NullPointerException
    + ArrayIndexOutofBoundsException
    + NoSuchMethodException 
    + ClassCastException
    + ClassNotFoundException  
    + FileNotFoundException
    + IndexOutOfBoundsException

+ error

  + OutOfMemoryError

  + StackOverflowError

  + NoClassDefFoundError

------



### 集合类

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





