# Redis的高级数据结构

## 位图

#### 背景

使用序号来表示顺序,`0/1`来表示状态的一种数据结构.主要用来节约存储空间.比如百度贴吧的连续点赞数,如果要记录用户的连续点赞数,假设用户是点赞一年这么久,那么如果我们使用普通的key/val数据结构,需记录365条记录.

而如果我们使用位图,每天的点赞记录只占据一个位,365天就是46个字节,这就能大效率的提升存储空间了.

#### 组成/本质

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。

#### 基本命令

+ setbit key index val
+ getbit key index val

**注意事项**

位图的序号是从`0`开始的,而且val值只能是`0/1`.

针对使用位设置值进去的值,因为位图本质是字符串,所以我们也可以使用`get/set`指令来操作



#### 统计和查找

Redis 提供了位图统计指令` bitcount `和位图查找指令 `bitpos`

+ bitcount key [start end]
+ bitpos key [start  end]

`bitcount` 用来统计指定位置范围内 1 的个数，`bitpos` 用来查找指定范围内出现的第一个 0 或 1。

`bitcount`与`bitpos`提供了[start,end]参数来指定范围参数,但参数值的单位为字节(1字节等于8位),所以我们无法在位的粒度上进行统计

以下为简单使用命令

```
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitcount w			#w字符串的中位数组为1的个数
(integer) 21
127.0.0.1:6379> bitcount w 0 0  	#第一个字符中 1 的位数
(integer) 3
127.0.0.1:6379> bitcount w 0 1  	#前两个字符中 1 的位数
(integer) 7
127.0.0.1:6379> bitpos w 0  		#第一个 0 位
(integer) 0
127.0.0.1:6379> bitpos w 1  		#第一个 1 位
(integer) 1
127.0.0.1:6379> bitpos w 1 1 1  	#从第二个字符算起，第一个 1 位
(integer) 9
127.0.0.1:6379> bitpos w 1 2 2  	#从第三个字符算起，第一个 1 位
(integer) 17
```

#### 魔术指令bitfield

bitfield指令可原子性的对多个位进行操作

bitfield目前提供了三个子命令

+ get
+ set
+ incrby

它们都可以对指定位片段进行读写，但是最多只能处理 64 个连续的位，如果超过 64 位，就得使用多个子指令，bitfield 可以一次执行多个子指令。注意序号是从0开始的

![https://ws3.sinaimg.cn/large/005BYqpggy1g2vgthgbwgj30il0afq4l.jpg](https://s2.ax1x.com/2019/05/09/E2ZqAO.png)

接下来我们对照着上面的图看个简单的例子:

```
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w get u4 0  # 从第一个位开始取 4 个位，结果是无符号数 (u)
(integer) 6
127.0.0.1:6379> bitfield w get u3 2  # 从第三个位开始取 3 个位，结果是无符号数 (u)
(integer) 5
127.0.0.1:6379> bitfield w get i4 0  # 从第一个位开始取 4 个位，结果是有符号数 (i)
1) (integer) 6
127.0.0.1:6379> bitfield w get i3 2  # 从第三个位开始取 3 个位，结果是有符号数 (i)
1) (integer) -3

//一次执行多次命令
127.0.0.1:6379> bitfield w get u4 0 get u3 2 get i4 0 get i3 2
1) (integer) 6
2) (integer) 5
3) (integer) 6
4) (integer) -3

//设置值
127.0.0.1:6379> bitfield w set u8 8 97  # 从第 9 个位开始，将接下来的 8 个位用无符号数 97 替换
1) (integer) 101
127.0.0.1:6379> get w
"hallo"

//incrby指令
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w incrby u4 2 1  # 从第三个位开始，对接下来的 4 位无符号数 +1
1) (integer) 11
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w incrby u4 2 1  # 溢出折返了
1) (integer) 0
```

bitfield 指令提供了溢出策略子指令 overflow，用户可以选择溢出行为，默认是折返 (wrap)，还可以选择失败 (fail) 报错不执行[返回nil]，以及饱和截断 (sat)[一直保持最大值]，超过了范围就停留在最大最小值。overflow 指令只影响接下来的第一条指令，这条指令执行完后溢出策略会变成默认值折返 (wrap)。



#### 适用场景

+ 网站用户的签到天数

+ 统计批量的电话号码中号码的个数(电话号码最大值为99999999,大概需要1亿个位,需要`100000000/8/1024/1024   =12M左右`)

  

------



## HyperLogLog

#### 定义/目标

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

**基数定义** 

*比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。*



#### 特点

+ HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素
+ HyperLogLog 提供不精确的去重计数方案，标准误差是 0.81%，这样的精确度已经可以满足上面的 UV 统计需求了。

#### 基本指令

+ pfadd
+ pfcount
+ pfmerge

```
127.0.0.1:6379> pfadd visitcount tinys
(integer) 1
127.0.0.1:6379> pfcount visitcount
(integer) 1
127.0.0.1:6379> pfadd visitcount tinys1
(integer) 1
127.0.0.1:6379> pfcount visitcount
(integer) 2
127.0.0.1:6379> pfadd visitcount tinys2
(integer) 1
127.0.0.1:6379> pfcount visitcount
(integer) 3
127.0.0.1:6379> pfadd visitcount tinys3
(integer) 1
127.0.0.1:6379> pfcount visitcount
(integer) 4
127.0.0.1:6379> 
```

#### 常用场景

+ 统计某个网站的UV(用户浏览数,需去重,如果userId是连续自增的也可以使用bitmap`setbit key userId 1`)
+ 用户搜索网站的关键词数量

------



#### 布隆过滤器(Bloom Filter)

#### 定义/目标

布隆过滤器可在数据集中`不精确`判断某个对象是否存在

当布隆过滤器判断某个元素不存在于数据集中时,元素一定不存在于数据集中

当布隆过滤器判断元素存在于数据集中时,元素有可能不存在于数据集中

Bloom Filter是Bit-map思想的一种扩展，它可以在允许低错误率的场景下，大大地进行空间压缩，是一种拿错误率换取空间的数据结构。

#### 基本使用

+ bf.add
+ bf.exists
+ bf.madd
+ bf.mexists
+ bf.reserve

```
127.0.0.1:6379> bf.add name tinys
(integer) 1
127.0.0.1:6379> bf.add name user2
(integer) 1
127.0.0.1:6379> bf.add name user3
(integer) 1
127.0.0.1:6379> bf.exists name tinys
(integer) 1
127.0.0.1:6379> bf.exists name user4
(integer) 0
```

redis提供了对布隆过滤器的自定义参数设置但我们需要在添加key之前即创建时就想声明这些参数,否则声明会报错

`bf.reserve`指令有三个参数,分别是`key`,`error_rate`和`initial_size`

错误率越低,所需的空间越大.而且当实际放入过滤器的元素数量超过初始化大小时,误判率会上升

[布隆计算器]: https://krisives.github.io/bloom-calculator/	"布隆计算器"

关于过滤器的空间大小举例

+ 100W的数据,错误率在1%的话,空间需大概1M
+ 1000W的数据,错误率在1%的话,空间需大概10M
+ 100W的数据,错误率在1%的话,空间需大概1.7M
+ 1000W的数据,错误率在0.1%的话,空间需大概17M

注意,redis4.0才为redis提供了插件,需升级到redis并安装插件才能正常使用布隆过滤器相关功能

#### 原理

![https://ws3.sinaimg.cn/large/005BYqpggy1g2x7dt94coj30h805yaak.jpg](https://s2.ax1x.com/2019/05/11/EWI71s.png)

使用一个大型的数组(bitmap)组成基本的数据元素,然后使用多个无偏的hash函数对新添加的key进行一一哈希计算,并将算出的索引值置为1,所以一个key会有多个索引值置为1.

判断一个新key是否存在于布隆过滤器中时,会将key的多个hash值到数据中的索引中匹配,如果发现其中一个索引值为0,则该key肯定不存在.如果全部的索引值都为1,那么新key可能存在于过滤器中(因为有可能是其它的key的多个hash值设置的)





#### 适用场景

+ 用户浏览地址/文章的判重
+ 其它不需要高精度的判重场景
+ 缓存穿透的解决方案(将所有可能的值缓存到过滤器中,不存在的值过滤掉,存在误判率)

#### 局限性

jedis目前并木有提供布隆过滤器的相关实现,需依赖`lettuce`

<https://github.com/lettuce-io/lettuce-core>