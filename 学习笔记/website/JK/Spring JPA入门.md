## Spring JPA入门

### 定义

`Java Persistence API`为对象关系映射提供了一种基于 POJO 的持久化模型.能有效的简化数据持久化代码的开发工作.

### 常用JPA注解

#### 实体

+ @Entity
+ @MappedSuperclass
+ @Table(name) 

#### 主键

+ @Id 
+  @GeneratedValue(strategy, generator)  //定义主键生成策略
+  @SequenceGenerator(name, sequenceName)  //定义序号生成器

#### 映射

+ @Column(name, nullable, length, insertable, updatable)
+ @JoinTable(name)
+ @JoinColumn(name) 

#### 关系

+ @OneToOne
+ @OneToMany
+ @ManyToOne
+ @ManyToMany 
+ @OrderBy

#### Hello World

实体类

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import javax.persistence.*;
import java.util.Date;


@Entity
@Table(name = "t_user")
@Builder
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    //声明主键
    @Id
    @GeneratedValue
    private Long id;
    
   
    private String name;

    //声明创建时间不能更改
    @Column(updatable = false)
    @CreationTimestamp
    private Date createTime;

    //声明更新时间是自动更新的
    @UpdateTimestamp
    private Date updateTime;
}

```

properties文件

```properties
# jpa特性
# 每次运行完创建完会删除表,注意工作时不能配`create`或者`create-drop`,建议使用`upadte`
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true

# 配置数据源属性
spring.datasource.url=jdbc:h2:mem:tinys;;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.username=sa
spring.datasource.password=
```

Dao类

```java
//这里不需要注入,spring会帮我们注入创建代理的
public interface UserRepository extends CrudRepository<User, Long> {
}
```

启动类

```java
@SpringBootApplication
@Slf4j
public class SpringjpaApplication  implements ApplicationRunner {
	@Autowired
	private UserRepository userRepository;


	public static void main(String[] args) {
		SpringApplication.run(SpringjpaApplication.class, args);
	}


	@Override
	public void run(ApplicationArguments args) throws Exception {
		insert();
		selectAll();
		update();
		selectAll();
	}

	private void selectAll() {
		log.info("查询结果{}",userRepository.findAll());
	}

	private void update() {
		User user = new User();
		user.setId(1L);
		user.setName("updateName");
		userRepository.save(user);
		log.info("更新数据完毕");
	}

	private void insert() {
		User user = new User();
		user.setName("tinys");
		userRepository.save(user);
		log.info("插入数据完毕");
	}
}
```

### 进阶版

实体类

```java
/**
 * 描述: 基本的实体类
 * 2019-07-28
 */
@MappedSuperclass
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BaseModel {
    @Id
    @GeneratedValue
    private Long id;
    @Column(updatable = false)
    @CreationTimestamp
    private Date createTime;
    @UpdateTimestamp
    private Date updateTime;
}

import com.tinys.enums.GradeState;
import lombok.*;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Enumerated;
import javax.persistence.Table;


@Entity
@Table(name = "t_student")
@Builder
@Data
@ToString(callSuper = true) //注意这里需声明调用了父类的toString方法
@NoArgsConstructor
@AllArgsConstructor
public class Student extends BaseModel {
    private String name;
    private Integer age;

    //声明了使用枚举
    @Enumerated
    @Column(nullable = false)
    private GradeState state;
}

//自定义的枚举,在Student中使用
public enum GradeState {
    Good(9,"优秀"),OK(1,"合格"),Bad(-1,"不及格"),Init(0,"初始化");
    private int level;
    private  String desc;

    GradeState(int level, String desc) {
        this.level = level;
        this.desc = desc;
    }
}

```

Dao层

```java
//声明继承PagingAndSortingRepository
@NoRepositoryBean
public interface BaseRepository<T, Long> extends PagingAndSortingRepository<T, Long> {

    /**
     *  JPA 提供的可根据某些规则来定义接口而不需写实现`
     *  具体参考 : https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.core-concepts
     */
    List<T> findTop3ByOrderByUpdateTimeDescIdAsc();
}

public interface StudentRepository extends  BaseRepository<Student,Long> {
     Student findStudentByName(String name);
}
```

启动类

```java
@SpringBootApplication
@Slf4j
public class SpringJpa2Application implements ApplicationRunner {
    @Autowired
    private StudentRepository studentRepository;

    public static void main(String[] args) {
        SpringApplication.run(SpringJpa2Application.class, args);
    }


    @Override
    public void run(ApplicationArguments args) throws Exception {
        insert();
        selectAll();
       selectByName();
    }

    private void selectByName() {
        log.info("根据名字来查找{}",studentRepository.findStudentByName("tinys"));
    }

    private void insert() {
        Student stu = Student.builder().name("tinys").age(15).state(GradeState.Good).build();
        studentRepository.save(stu);
        log.info("插入完毕");
    }

    private void selectAll() {
        log.info("查询结果{}",studentRepository.findTop3ByOrderByUpdateTimeDescIdAsc());
    }
}

```

#### JPA提供的自定义方法钩子

可按照一定的约束在接口中声明方法而无需实现就能有相应的功能.

**根据方法名定义查询**

+ find…By… / read…By… / query…By… / get…By… 
+ count…By… 
+ …OrderBy…[Asc / Desc] 
+ And / Or / IgnoreCase 
+ Top / First / Distinct

[参考官方文档](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.core-concepts)

简要的示例展示

![https://s2.ax1x.com/2019/07/28/eQ31Cn.png](http://ww1.sinaimg.cn/large/8bb38904ly1g5fcgtcifwj20nd0qi0u3.jpg)

**官方例子**

```java
interface PersonRepository extends Repository<User, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}

//带分页参数的例子
Page<User> findByLastname(String lastname, Pageable pageable);

Slice<User> findByLastname(String lastname, Pageable pageable);

List<User> findByLastname(String lastname, Sort sort);

List<User> findByLastname(String lastname, Pageable pageable);

//限制条数的例子
User findFirstByOrderByLastnameAsc();

User findTopByOrderByAgeDesc();

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

Slice<User> findTop3ByLastname(String lastname, Pageable pageable);

List<User> findFirst10ByLastname(String lastname, Sort sort);

List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

### 开启事务

```java
@SpringBootApplication
@Slf4j
@EnableTransactionManagement //开启事务
@EnableJpaRepositories //spring boot默认帮我们开启了
public class SpringJpa2Application implements ApplicationRunner {
    @Autowired
    private StudentRepository studentRepository;

    public static void main(String[] args) {
        SpringApplication.run(SpringJpa2Application.class, args);
    }


    @Override
    @Transactional  //声明此方法使用事务
    public void run(ApplicationArguments args) throws Exception {
        insert();
        selectAll();
        selectByName();
    }

    private void selectByName() {
        log.info("根据名字来查找{}",studentRepository.findStudentByName("tinys"));
    }

    private void insert() {
        Student stu = Student.builder().name("tinys").age(15).state(GradeState.Good).build();
        studentRepository.save(stu);
        log.info("插入完毕");
    }

    private void selectAll() {
        log.info("查询结果{}",studentRepository.findTop3ByOrderByUpdateTimeDescIdAsc());
    }
}
```

