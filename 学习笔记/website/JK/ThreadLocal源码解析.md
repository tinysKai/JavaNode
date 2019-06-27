##  ThreadLocal源码解析

### 总结

#### 定义/目标

ThreadLocal类用来提供**线程内部的局部变量**。这种变量在多线程环境下访问(通过get或set方法访问)时能**保证各个线程里的变量相对独立于其他线程内的变量**。



#### 实现原理

将数据跟线程绑定.数据关联在`Thread`类的成员变量`threadLocals`下.这个变量是一个`ThreadLocalMap`的引用,用来保存同个线程下的多个`ThreadLocal`.



#### 弱引用问题

![](https://s2.ax1x.com/2019/06/26/ZezlUs.png)

Entry的key是弱引用,而val是强引用..当key被回收线程扫描到回收时,Entry的val由于是强引用,而且此时还存在着`ThreadLocalMap`的Entry的引用,所以可能存在内存泄漏

解决方案

- 避免在线程池中使用ThreadLocal
- 养成使用完remove的习惯

**ThreadLocal清理无效Entry的时机**

+ set(添加新值时|扩容时)
+ get(获取到一个Entry不为空但key为空的Entry)
+ remove(删除掉某个entry时)



#### 哈希冲突解决方法

开放地址-线性探测法,将哈希冲突的entry放置到当前数组后第一个为空的数组下标中



#### 类关系图

![](https://s2.ax1x.com/2019/06/26/ZeOcJf.png)



#### 扩容原理

新建一个数组大小为2倍的新数组,将原数组元素逐个遍历设置到新数组中.最终设置完毕后将entry数组的引用指向性数组.由于ThreadLocal是本线程操作的,所以无并发问题

#### 疑问点

为啥需要Entry数组而不是一个单独一个entry?

因为`ThreadLocalMap`可以对应多个`ThreadLocal`,每个ThreadLocal在初始化时便会初始化hash码.并且一直保持不变



### 源码

#### 基本方法

```java
public T get() {
    Thread t = Thread.currentThread();
    //根据当前线程获取ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //计算对应的Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

//初始化方法
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //设置初始值
        map.set(this, value);
    else
        //创建ThreadLocalMap
        createMap(t, value);
    return value;
}

//设置值
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //设置当前threadlocal对应一个val
        map.set(this, value);
    else
        //创建ThreadLocalMap
        createMap(t, value);
}


//返回当前线程的ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

//创建当前线程的ThreadLocalMap
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

####  线程关联

```java
public class Thread implements Runnable { 
     //ThreadLocal关联Thread的这个属性
	ThreadLocal.ThreadLocalMap threadLocals = null;
}    
```

#### ThreadLocalMapd的基本数据结构

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);//key是虚引用
        value = v;
    }
}
```

#### 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);//取模
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

#### 哈希冲突相关方法

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    //遍历entry数组
    while (e != null) {
        ThreadLocal<?> k = e.get();
        //由于使用了线性探测法,所以key可能就存在于下一个数组下标
        if (k == key)
            return e;
        //弱引用的键,被jvm清理了
        if (k == null)
            //释放此key对应的entry的val,并且遍历entry数组释放之后的空key的entry以及rehash遍历到的key
            expungeStaleEntry(i);
        else
            //开始下一下标
            i = nextIndex(i, len);
        e = tab[i];
    }
    //如果entry为空则直接返回空
    return null;
}


private static int nextIndex(int i, int len) {
   return ((i + 1 < len) ? i + 1 : 0);
}

//清理无效entry以及rehash值
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //将entry的val置空以及entry对象置空,帮助gc回收
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null; //每次判断时都会赋值给entry
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        //遍历到空则清理
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //否则重新哈希
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
			  //重新计算下标值,从hash值计算第一个非空值
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}

//调用至少log2N次的expungeStaleEntry
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            //每触发一次清理都将n值重置
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

#### 关键方法

```java
//根据key来
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        //当前entry不为空而且不相等,由于使用的是开放地址法,所以开始循环遍历Entry数组直到Entry为空
        return getEntryAfterMiss(key, i, e);
}

//设置值到ThreadLocalMap
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    //计算数组下标
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];	//从计算的数组下标开始迭代
         e != null;
         e = tab[i = nextIndex(i, len)]) {	//每次数组下标+1
        ThreadLocal<?> k = e.get();

        //key相同则直接更新val值
        if (k == key) {
            e.value = value;
            return;
        }
		//key为空则替换对应的entry	
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        //扩容与重哈希
        rehash();
}

//清空对应的entry
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            //清空entry
            e.clear();
            //清除后续的无效entry
            expungeStaleEntry(i);
            return;
        }
    }
}
```

#### 扩容

```java
//重hash
private void rehash() {
    //清理整个entry数组的无效键值
    expungeStaleEntries();

    // size > 3/4的threshold时触发扩容
    if (size >= threshold - threshold / 4)
        resize();
}

//清理整个entry数组的空key的entry
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}

//扩容为原entry数组的两倍
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    //最后将entry数组引用切换到新数组
    table = newTab;
}
```







































































