## Mysql - 排序

#### 案例

```sql
select city,name,age from t where city='杭州' order by name limit 1000  ;
```

![eH6Lv9.png](https://s2.ax1x.com/2019/08/09/eH6Lv9.png)

Extra 这个字段中的“Using filesort”表示的就是需要排序，MySQL 会给每个线程分配一块内存用于排序，称为 sort_buffer。



#### 全字段排序

`sort_buffer_size`，就是 MySQL 为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于` sort_buffer_size`，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

查出主键后回表查询记录,然后直接在buffer中排序

![eHcN5T.jpg](https://s2.ax1x.com/2019/08/09/eHcN5T.jpg)

####  文件排序

以下语句可确认排序是否使用了临时文件排序

```sql
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
select @b-@a;

```

查看`OPTIMIZER_TRACE`的执行结果

![eHRoL9.png](https://s2.ax1x.com/2019/08/09/eHRoL9.png)

#### rowid排序

当单行记录偏大时,mysql只会将需排序的字段以及主键放进`sort buffer`中进行排序,排序完后再一次回表获取全部所需的字段数据.

![eHf2gU.jpg](https://s2.ax1x.com/2019/08/09/eHf2gU.jpg)

#### 使用联合索引来避免排序

```sql
alter table t add index city_user(city, name);
```

![eH4R0J.png](https://s2.ax1x.com/2019/08/09/eH4R0J.png)

Extra 字段中没有` Using filesort `了，也就是不需要排序了。而且由于 (city,name) 这个联合索引本身有序，所以这个查询也不用把 4000 行全都读一遍，只要找到满足条件的前 1000 条记录就可以退出了。也就是说，在我们这个例子里，只需要扫描 1000 次。

#### 覆盖索引来避免回表

```sql
alter table t add index city_user_age(city, name, age);
```

![eH5m90.png](https://s2.ax1x.com/2019/08/09/eH5m90.png)

Extra 字段里面多了`Using index`，表示的就是使用了覆盖索引，性能上会快很多。

