## 记一次Mysql优化学习

#### 背景

Mybatis根据list列表数据来批量update数据.



#### 知识点

+ CASE WHEN THEN
+ 多个字段的in查询



#### 具体实践

第一版使用`CASE WHEN THEN END`来设置字段值,然后单个字段的查询条件使用for循环来拼凑in条件.

第二版因需要使用到两个条件来组合匹配,所以查询到`IN`也有组合用法.

```xml
 <update id="batchUpdate" parameterType="java.util.List">
        UPDATE  tableName
        <trim prefix="set" suffixOverrides=",">    
            <trim prefix="A = case" suffix="end,">
                <foreach collection="list" item="item" index="index">
                    <if test="item.A != null" >
                        WHEN B = #{item.B,jdbcType=VARCHAR} THEN #{item.A,jdbcType=VARCHAR}
                    </if>
                </foreach>
            </trim>
        </trim>
        WHERE 
        (`B`,`C`) IN
        <foreach collection="list" item="item" index="index" separator="," open="(" close=")">
            (#{item.B,jdbcType=VARCHAR},#{item.C,jdbcType=VARCHAR})
        </foreach>
    </update>
```



```sql
-- 最骚的是根据条件筛选出来的每条记录都能自己匹配条件更改
-- 然后in操作的联合in也长知识了,比较少用
UPDATE 
 	 tableName 
SET
  A = 
  CASE
    WHEN B = ? 
    	THEN val1 
    WHEN B = ? 
    	THEN val2
  END
WHERE 
  (
    `B`,
    `C`
  ) IN (
    (?, ?),
    (?, ?)
    );
```

```sql
-- 骚操作直接使用,如下句sql的作用是筛选出符合IN条件的记录..然后在这些记录里面,每条记录根据属性值不同去设置对应的值
-- 比如这里最终的操作就是table表中,A01对应记录的B字段的值为1,而A02记录对应的B字段对应的值为2
UPDATE `table`  a
	SET   a.`B` =
	CASE 
		WHEN A.`id` = "A01"
			THEN "1"
		WHEN A.`id`  ="A02"
			THEN "2"
	END
WHERE id IN ("A01","A02");

```

