# MyBatis中的OGNL教程

## MyBatis常用OGNL表达式

```
e1 or e2
e1 and e2
e1 == e2,e1 eq e2
e1 != e2,e1 neq e2
e1 lt e2：小于
e1 lte e2：小于等于，其他gt（大于）,gte（大于等于）
e1 in e2
e1 not in e2
e1 + e2,e1 * e2,e1/e2,e1 - e2,e1%e2
!e,not e：非，求反
e.method(args)调用对象方法
e.property对象属性值
e1[ e2 ]按索引取值，List,数组和Map
@class@method(args)调用类的静态方法
@class@field调用类的静态字段值
```

上述内容只是合适在MyBatis中使用的OGNL表达式，完整的表达式点击

[这里]: https://commons.apache.org/proper/commons-ognl/language-guide.html



## MyBatis中什么地方可以使用OGNL

MyBatis中可以使用OGNL的地方有两处：

- 动态SQL表达式中
- ${param}参数中

上面这两处地方在MyBatis中处理的时候都是使用OGNL处理的。



下面通过举例来说明这两种情况的用法。

#### 动态SQL表达式中

例一，MySql like 查询:

```sql
<select id="xxx" ...>
    select id,name,... from country
    <where>
        <if test="name != null and name != ''">
            name like concat('%', #{name}, '%')
        </if>
    </where>
</select>
```

上面代码中test的值会使用OGNL计算结果。

例二，通用 like 查询：

```java
<select id="xxx" ...>
    select id,name,... from country
    <bind name="nameLike" value="'%' + name + '%'"/>
    <where>
        <if test="name != null and name != ''">
            name like #{nameLike}
        </if>
    </where>
</select>
```

这里`<bind>`的`value`值会使用OGNL计算。

注：对`<bind`参数的调用可以通过`#{}`或 `${}` 方式获取，`#{}`可以防止注入。

#### ${param}参数中

```sql
<select id="xxx" ...>
    select id,name,... from country
    <where>
        <if test="name != null and name != ''">
            name like '${'%' + name + '%'}'
        </if>
    </where>
</select>
```

这里注意写的是${'%' + name + '%'}，而不是%${name}%，这两种方式的结果一样，但是处理过程不一样。

在MyBatis中处理${}的时候，只是使用OGNL计算这个结果值，然后替换SQL中对应的${xxx}，OGNL处理的只是${这里的表达式}。

这里表达式可以是OGNL支持的所有表达式，可以写的很复杂，可以调用静态方法返回值，也可以调用静态的属性值。



#### 参考链接

<https://blog.csdn.net/isea533/article/details/50061705> 

