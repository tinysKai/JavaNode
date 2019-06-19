>关于把名单导入到数据库时检查手机好是否重复怎么做比较好?

    先将新名单自己去重.
    将去重后的新名单插入到一个临时表temp.
    删除新名单中存在于旧名单的数据
        delete * from temp where exists (select 1 from target where temp.id = target.id)
    将不重复的名单插入到名单表中
        insert into target select * from temp

        
         
>数据库         
![SQL的关联关系](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709//SQL_JOIN.png)

>写SQL的原则
+ 少关联,让关系数据库无关联
+ 若关联,则小表关联大表,尽量缩小每一个结果集
+ 查询尽量走索引



