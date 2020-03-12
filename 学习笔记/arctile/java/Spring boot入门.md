## Spring boot入门

### Hello World

#### 初始化spring boot项目

idea下打开`File --> Project --> Spring Initializr`来初始化一个Spring boot项目.注意此次demo需选择的模块有

+ H2
+ jdbc
+ lombok
+ web
+ acttutor

#### 主程序

```java
@SpringBootApplication
@Slf4j
public class SpringdatasourceApplication implements CommandLineRunner {
	@Autowired
	@Qualifier("dataSource")
	private DataSource dataSource;

	@Autowired
	JdbcTemplate jdbcTemplate;

	public static void main(String[] args) {
		SpringApplication.run(SpringdatasourceApplication.class, args);
	}

	/**
	 * Callback used to run the bean.
	 */
	@Override
	public void run(String... args) throws Exception {
		showConnection();
		showData();
	}

	/**
	 * 这里有个点是关于spring boot提供了在启动时执行ddl以及dml的功能
	 * 通过在resource文件夹下的schema.sql以及data.sql来实现
	 *
	 * 展示数据
	 */
	private void showData() {
		jdbcTemplate.queryForList("SELECT * FROM student")
				.forEach(row -> log.info(row.toString()));
	}


	/**
	 * 展示数据源信息
	 */
	private void showConnection() throws SQLException {
		log.info("数据源 : {}", dataSource.toString());
		Connection conn = dataSource.getConnection();
		log.info("连接{}",conn.toString());
		conn.close();
	}

}

```

#### pom.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.6.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.tinys</groupId>
	<artifactId>springdatasource</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>springdatasource</name>
	<description>Demo project for Spring Boot</description>

	<properties>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web-services</artifactId>
		</dependency>

		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```

#### SQL文件

```sql
-- scheme.sql
CREATE TABLE student (ID INT IDENTITY, name VARCHAR(16));
```

```sql
-- data.sql
INSERT INTO student (ID, `name`) VALUES (1, 'aaa');
INSERT INTO student (ID, `name`) VALUES (2, 'bbb');
```

#### 配置属性(可选)

```properties
#通用
spring.datasource.url=jdbc:mysql://localhost/test

spring.datasource.username=dbuser

spring.datasource.password=dbpass

spring.datasource.driver-class-name=com.mysql.jdbc.Driver（可选）

#初始化内嵌数据库
spring.datasource.initialization-mode=embedded|always|never

spring.datasource.schema与spring.datasource.data确定初始化SQL文件

spring.datasource.platform=hsqldb | h2 | oracle | mysql | postgresql（与前者对应）

```

### 配置多个数据源

```java
/**
 * 排除掉默认的自动配置DataSource等Configuration,自己实现多数据源配置
 */
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class,
        DataSourceTransactionManagerAutoConfiguration.class,
        JdbcTemplateAutoConfiguration.class})
@Slf4j
public class MultiDataSourceDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(MultiDataSourceDemoApplication.class, args);
    }

    /**
     * 配置读取的属性文件是以`foo.datasource`开头的
     */
    @Bean
    @ConfigurationProperties("foo.datasource")
    public DataSourceProperties fooDataSourceProperties() {
        return new DataSourceProperties();
    }

    /**
     * 声明一个dataSource,bean的名字为方法名
     */
    @Bean
    public DataSource fooDataSource() {
        DataSourceProperties dataSourceProperties = fooDataSourceProperties();
        log.info("foo datasource: {}", dataSourceProperties.getUrl());
        return dataSourceProperties.initializeDataSourceBuilder().build();
    }

    /**
     * 注入事务管理器
     */
    @Bean
    @Resource
    public PlatformTransactionManager fooTxManager(DataSource fooDataSource) {
        return new DataSourceTransactionManager(fooDataSource);
    }

    @Bean
    @ConfigurationProperties("bar.datasource")
    public DataSourceProperties barDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource barDataSource() {
        DataSourceProperties dataSourceProperties = barDataSourceProperties();
        log.info("bar datasource: {}", dataSourceProperties.getUrl());
        return dataSourceProperties.initializeDataSourceBuilder().build();
    }

    @Bean
    @Resource
    public PlatformTransactionManager barTxManager(DataSource barDataSource) {
        return new DataSourceTransactionManager(barDataSource);
    }
}

```

属性文件

```properties
management.endpoints.web.exposure.include=*
# 多彩输出
spring.output.ansi.enabled=ALWAYS

# foo数据源的配置
foo.datasource.url=jdbc:h2:mem:foo
foo.datasource.username=sa
foo.datasource.password=

# bar数据源的配置
bar.datasource.url=jdbc:h2:mem:bar
bar.datasource.username=sa
bar.datasource.password=
```

### 连接池

#### 默认连接池

`Sring boot 2.0`默认的连接池就是`HikariCP`,其常用配置如下

```properties
#常用配置
spring.datasource.hikari.maximumPoolSize=10

spring.datasource.hikari.minimumIdle=10

spring.datasource.hikari.idleTimeout=600000

spring.datasource.hikari.connectionTimeout=30000

spring.datasource.hikari.maxLifetime=1800000

```

#### 修改为Druid连接池

启动类

```java
@SpringBootApplication
@Slf4j
public class DruidDemoApplication implements CommandLineRunner {
	@Autowired
	@Qualifier("dataSource")
	private DataSource dataSource;
    
	@Autowired
	private JdbcTemplate jdbcTemplate;

	public static void main(String[] args) {
		SpringApplication.run(DruidDemoApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		log.info(dataSource.toString());
	}
}
```

pom文件

```xml
<!-- 排除默认的Hikaricp连接池 -->
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web-services</artifactId>
			<!--添加排除掉的默认的HikariCp连接池-->
			<exclusions>
				<exclusion>
					<artifactId>HikariCP</artifactId>
					<groupId>com.zaxxer</groupId>
				</exclusion>
			</exclusions>
</dependency>

<!-- 添加druid连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.10</version>
</dependency>
```



properties文件

```properties
#多彩输出
spring.output.ansi.enabled=ALWAYS
# 数据源配置
spring.datasource.url=jdbc:h2:mem:tinys
spring.datasource.username=sa
spring.datasource.password=

# 配置druid的特性
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.filters=conn,config,stat,slf4j

# 启用ConfigFilter,配置后本示例中的ConnectionLogFilter会生效
spring.datasource.druid.filter.config.enabled=true

# 连接池配置
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
spring.datasource.druid.test-while-idle=true
```

Filter

```java
@Slf4j
public class ConnectionLogFilter extends FilterEventAdapter {

    @Override
    public void connection_connectBefore(FilterChain chain, Properties info) {
        log.info("BEFORE CONNECTION!");
    }

    @Override
    public void connection_connectAfter(ConnectionProxy connection) {
        log.info("AFTER CONNECTION!");
    }
}
```

过滤器配置

```properties
# druid-filter.properties文件
druid.filters.conn=com.tinys.ConnectionLogFilter
```

![<https://imgchr.com/i/eKF7j0>](http://ww1.sinaimg.cn/large/8bb38904ly1g5e6gi8rkhj20cd0ct0t5.jpg)