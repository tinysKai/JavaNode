## Spring Boot - Redis入门

### 使用JedisPool

#### 启动类

```java
@SpringBootApplication
@Slf4j
public class DemoApplication implements CommandLineRunner{
	@Autowired
	private JedisPool jedisPool;
	@Autowired
	private JedisPoolConfig jedisPoolConfig;

	//声明JedisPoolConfig
	@Bean
	@ConfigurationProperties("redis") //表明使用配置文件redis开头的配置
	public JedisPoolConfig jedisPoolConfig() {
		return new JedisPoolConfig();
	}


	/**
	 * 声明redis连接池
	 * 使用destroyMethod声明bean注销掉时调用close方法
	 */
	@Bean(destroyMethod = "close")
	public JedisPool jedisPool(@Value("${redis.host}") String host) {
		return new JedisPool(jedisPoolConfig(), host);
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

	//如果使用虚拟机,记得redis去`bind ip`,然后关闭防火墙`service iptables stop`
	@Override
	public void run(String... args) throws Exception {
		log.info(jedisPoolConfig.toString());

		//使用try-with-resource来自动关闭资源
		try (Jedis jedis = jedisPool.getResource()) {
			String key = "name";
			jedis.set(key,"tinys");
			log.info("读取redis的值{}",jedis.get(key));
		}
	}
}
```

#### POM依赖

```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>

		<dependency>
			<groupId>redis.clients</groupId>
			<artifactId>jedis</artifactId>
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
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
</dependencies>
```

#### Properties属性

```properties
redis.host=192.168.198.128
redis.maxTotal=5
redis.maxIdle=5
redis.testOnBorrow=true
```

### Spring Cache入门

#### 启动类

```java
@SpringBootApplication
@Slf4j
@EnableCaching(proxyTargetClass = true)
public class JvmCacheApplication  implements CommandLineRunner{
    @Autowired
    StudentService studentService;
    public static void main(String[] args) {
        SpringApplication.run(JvmCacheApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        selectAllWithFiveTimes();
    }

    private void selectAllWithFiveTimes() {
        for (int i = 0; i < 5; i++) {
            studentService.list();
        }
    }
}

```

#### 实体类

```java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name = "t_student")
@Builder
@Data
@ToString()
@NoArgsConstructor
@AllArgsConstructor
public class Student {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
}

```

#### Dao类

```java
public interface StudentDao extends JpaRepository<Student, Long> {}
```

#### Service类

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.List;

@Slf4j
@Service
@CacheConfig(cacheNames = "student")
public class StudentService {
    @Autowired
    private StudentDao studentDao;

    //使用缓存
    @Cacheable
    public List<Student> list(){
        return studentDao.findAll();
    }


    //清除缓存
    @CacheEvict
    public void deleteAll(){
        studentDao.deleteAll();
    }
}
```

#### 属性文件

```properties
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
```

#### 数据库文件

```sql
-- schema.sql
CREATE TABLE student (ID INT IDENTITY, name VARCHAR(16));

-- data.sql
INSERT INTO student (ID, `name`) VALUES (1, 'aaa');
INSERT INTO student (ID, `name`) VALUES (2, 'bbb');
```

### 使用redis的Cache 缓存

#### POM文件

```xml
<!--引入以下依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

#### 配置文件

```properties
# 新加以下cache 属性
spring.cache.type=redis
spring.cache.cache-names=student
spring.cache.redis.time-to-live=5000
spring.cache.redis.cache-null-values=false
spring.redis.host=192.168.198.128
```

### 使用redisTemplate & JPA

#### 启动类

```java
@Slf4j
@SpringBootApplication
public class RedisTemplateApplication implements CommandLineRunner{

    @Autowired
    RedisTemplate<String,String> redisTemplate;

    @Autowired
    StudentCacheDao studentCacheDao;

    @Bean
    public RedisTemplate<String, Student> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Student> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    public static void main(String[] args) {
        SpringApplication.run(RedisTemplateApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        redisTemplate();
        jpaRedis();
    }

    private void jpaRedis() {
        log.info("使用JPA");
        StudentCache cache = StudentCache.builder().id(1L).name("tinys").build();
        studentCacheDao.save(cache);
        log.info(studentCacheDao.findAll().toString());
    }

    private void redisTemplate() {
        log.info("使用redisTemplate");
        boolean flag =  redisTemplate.hasKey("tinys");
        log.info("redis是否存在tinys这个键{}",flag);
        redisTemplate.opsForValue().set("key","val");
        log.info("string模式的key对应的值为{}",redisTemplate.opsForValue().get("key"));

        redisTemplate.opsForHash().put("tinys","age","29");
        redisTemplate.expire("tinys",1, TimeUnit.SECONDS);//设置超时时间
        log.info(redisTemplate.opsForHash().get("tinys","age").toString());
    }
}
```

#### 实体类

```java
import org.springframework.data.redis.core.RedisHash;
import org.springframework.data.redis.core.index.Indexed;

import javax.persistence.Id;


@RedisHash(value = "student", timeToLive = 60)
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class StudentCache {
    @Id
    private Long id;
    @Indexed
    private String name;
}
```

#### Dao层

```java
public interface StudentCacheDao  extends CrudRepository<StudentCache,Long>{}
```

