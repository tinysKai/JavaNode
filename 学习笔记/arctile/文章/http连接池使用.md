>使用org.apache.http.impl.conn.PoolingClientConnectionManager类即可创建httpclient连接池

>这个类建议设置最大连接数和最大路由连接数.最大路由连接数可理解为连接池维持的网站地址数目.
默认的连接池配置是最大连接是20，每个路由最大连接是2
```
    public final static int MAX_TOTAL_CONNECTIONS = 400; 
    public final static int MAX_ROUTE_CONNECTIONS = 200; 
    PoolingClientConnectionManager cm = new PoolingClientConnectionManager();  
    cm.setMaxTotal(MAX_TOTAL_CONNECTIONS);  
    cm.setDefaultMaxPerRoute(MAX_ROUTE_CONNECTIONS); 
```

>使用连接池配置超时时间,一个是连接超时一个是读超时
```
httpParams = new BasicHttpParams();  
httpParams.setParameter(CoreConnectionPNames.CONNECTION_TIMEOUT,CONNECT_TIMEOUT);  
httpParams.setParameter(CoreConnectionPNames.SO_TIMEOUT, READ_TIMEOUT);  
```

>注意httplient设置的最大连接数绝对不能超过tomcat设置的最大连接数，否则tomcat的连接就会被httpclient连接池一直占用，直到系统挂掉。

参考链接 : https://blog.csdn.net/mawming/article/details/49617829