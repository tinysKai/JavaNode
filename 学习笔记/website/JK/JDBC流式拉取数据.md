## JDBC流式拉取数据

#### 背景

mysql一次性拉取数据过多会出现内存溢出,因为默认情况下会将数据一次性从客户端拉取放在客户端内存中.

#### 解决方案

使用流式拉取数据

![](http://ww1.sinaimg.cn/large/8bb38904ly1g5zj7yez54j20km05sq3d.jpg)

```java
statement = conn.createStatement(ResultSet.TYPE_FORWARD_ONLY,ResultSet.CONCUR_READ_ONLY);
statement.setFetchSize(1000);
resultSet = statement.executeQuery(sb.toString());
```

