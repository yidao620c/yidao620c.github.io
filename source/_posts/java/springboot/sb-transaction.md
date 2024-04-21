---
title: SpringBoot系列 - 声明式事务
date: 2017-07-26 15:55:36 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

所有数据访问技术都有事务机制，这些技术提供了API来开启事务、提交事务完成数据操作，或者在发生错误的时候回滚数据。

Spring采用统一的机制来处理不同的数据访问技术的事务，
Spring的事务提供一个`PlatformTransactionManager`的接口，不同的数据访问技术使用不同的接口实现。
<!-- more -->

 Data Tech | 实现                           
-----------|------------------------------
 JDBC      | DataSourceTransactionManager 
 JPA       | JPATransactionManager        
 Hibernate | HibernateTransactionManager  
 JDO       | JDOTransactionManager        
 分布式事务     | JtaTransactionManager        

而得益于SpringBoot的自动配置机制，为我们自动开启了声明式事务支持，
我们无需添加注解`@EnableTransactionManagement`

## 事务基础

Spring提供一个@EnableTransactionManagement注解在配置类上开启声明式事务支持，
自动扫描加了@Transactional注解的类和方法，加入事务支持。

这里重点讲下@Transactional注解的几个属性

**propagation**

事务的传播机制，主要有以下几种，默认是REQUIRED：

1. REQUIRED - 方法A调用时候没有事务新建一个事务，在方法A中调用方法B，将使用相同的事务，如果方法B发生异常需要回滚，整个事务回滚。
2. REQUIRES_NEW - 方法A调用方法B时，无论是否存在事务都开启一个新事务，这样B方法异常不会导致A的数据回滚。
3. NESTED - 和REQUIRES_NEW类似，但是只支持JDBC，不支持JPA或Hibernate
4. SUPPORTS - 方法调用时有事务就用事务，没事务就不用事务
5. NOT_SUPPORTED - 强制方法不在事务中执行，若有事务，在方法调用到结束阶段先挂起事务。
6. NEVER - 强制不能有事务，若有事务就抛出异常
7. MANDATORY - 强制必须有事务，如果没有事务就抛出异常

**isolation**

事务的隔离级别，决定了事务的完整性，主要一下几种，默认是DEFAULT：

1. READ_UNCOMMITTED - A事务修改记录但没提交，B事务可读取到修改后的值。可导致脏读、不可重复读、幻读。
2. READ_COMMITTED - A事务修改并提交后，B事务才能读取到修改后的值，阻止了脏读，但可能导致不可重复读和幻读。
3. REPEATABLE_READ - A事务读取了一条记录，B事务将不能修改这条记录，阻止脏读和不可重复读，但是可能出现幻读。
4. SERIALIZABLE - 事务是顺序执行的，可避免所有缺陷，但是开销很大。
5. DEFAULT - 使用当前数据库默认隔离级别，入Oracle、SQL Server是READ_COMMITTED，MySQL是REPEATABLE_READ

**timeout**

事务过期时间，默认是当前数据库默认事务过期时间。

**readOnly**

指定是否为只读事务，默认是false

如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持SQL执行期间的读一致性；

如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询SQL必须保证整体的读一致性，
否则，在前条SQL查询之后，后条SQL查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，
此时，应该启用只读事务支持。

只读事务与读写事务区别：

对于只读查询，可以指定事务类型为readonly，即只读事务。由于只读事务不存在数据的修改，
因此数据库将会为只读事务提供一些优化手段，例如Oracle对于只读事务，不启动回滚段，不记录回滚log。

**rollbackFor**

指定哪些异常可以导致事务回滚，默认是Throwable的子类

**noRollbackFor**

执行哪些异常不可用引起事务回滚，默认是Throwable的子类

## 实战篇

实际项目中，使用SpringBoot的默认配置就已经足够满足我们的需求了。
本篇将通过几个例子来演示如何使用@Transactional注解，在出现异常时候回滚或不回滚数据。

使用我最喜欢的DAO技术就是MyBatis进行数据访问，使用druid数据库连接池，
另外配合mybatis-plus，实现数据访问层。

引入依赖：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>${mysql-connector.version}</version>
<scope>runtime</scope>
</dependency>
<dependency>
<groupId>com.alibaba</groupId>
<artifactId>druid</artifactId>
<version>${druid.version}</version>
</dependency>
        <!-- MyBatis plus增强和springboot的集成-->
<dependency>
<groupId>com.baomidou</groupId>
<artifactId>mybatis-plus</artifactId>
<version>${mybatis-plus.version}</version>
</dependency>
<dependency>
<groupId>com.baomidou</groupId>
<artifactId>mybatisplus-spring-boot-starter</artifactId>
<version>${mybatisplus-spring-boot-starter.version}</version>
</dependency>
```

配置数据库连接：

```yaml
###################  spring配置  ###################
spring:
  profiles:
    active: dev
  datasource:
    url: jdbc:mysql://123.207.66.156:3306/test?useSSL=false&autoReconnect=true&tinyInt1isBit=false&useUnicode=true&characterEncoding=utf8
    username: mysql
    password: xxxxx
```

然后增加mybatis个性化配置：

```yaml
###################  mybatis-plus配置  ###################
mybatis-plus:
  mapper-locations: classpath*:com/xncoding/trans/dao/repository/mapping/*.xml
  typeAliasesPackage: >
    com.xncoding.trans.dao.entity
  global-config:
    id-type: 0  # 0:数据库ID自增   1:用户输入id  2:全局唯一id(IdWorker)  3:全局唯一ID(uuid)
    db-column-underline: false
    refresh-mapper: true
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: true #配置的缓存的全局开关
    lazyLoadingEnabled: true #延时加载的开关
    multipleResultSetsEnabled: true #开启的话，延时加载一个属性时会加载该对象全部属性，否则按需加载属性
```

增加实体类User：

```java
@TableName(value = "t_user")
public class User extends Model<User> {
    /**
     * 主键ID
     */
    @TableId(value="id", type= IdType.AUTO)
    private Integer id;

    private String username;

    private String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @Override
    protected Serializable pkVal() {
        return this.id;
    }
}
```

增加UserMapper类：

```java
public interface UserMapper extends BaseMapper<User> {
}
```

增加Mybatis配置类：

```java
@Configuration
@EnableTransactionManagement(order = 2)
@MapperScan(basePackages = {"com.xncoding.trans.dao.repository"})
public class MybatisPlusConfig {

    @Resource
    private DruidProperties druidProperties;

    /**
     * 单数据源连接池配置
     */
    @Bean
    public DruidDataSource singleDatasource() {
        DruidDataSource dataSource = new DruidDataSource();
        druidProperties.config(dataSource);
        return dataSource;
    }
}
```

定义Service，并注入UserMapper：

```java
@Service
public class UserService {
    @Resource
    private UserMapper userMapper;
}
```

增加Controller，注入Service，定义几个url来做测试用：

```java
@RestController
public class UserController {
    @Resource
    private UserService userService;
}
```

到此为止项目初始化完成，可以在Service中添加方法进行声明式事务测试了。

## 异常回滚

@Transactional注解可以放到类也可以放到方法上，如果放到类上面会对所有public方法添加注解，
不过你仍然可以在方法上面加这个注解，会覆盖类上面声明的事务注解。

先实验一个抛出异常会回滚的方法：

```java
 /**
  * 增删改要写 ReadOnly=false 为可写
  * @param user 用户
  */
 @Transactional(readOnly = false)
 public void updateUserError(User user) {
     userMapper.updateById(user);
     errMethod(); // 执行一个会抛出异常的方法
 }

 private void errMethod() {
     System.out.println("error");
     throw new RuntimeException("runtime");
 }
```

然后在Controller里面添加一个url调用此方法：

```java
@RequestMapping("/errorUpdate")
    public Object first() {
        User user = new User();
        user.setId(1);
        user.setUsername("admin");
        user.setPassword("admin");
        userService.updateUserError(user);
        return "first controller";
    }
}
```

数据库里面先插入一条数据：`1|admin|123`

启动应用后访问地址：`http://localhost:8092/errorUpdate`

控制台打印异常：

```
Caused by: java.lang.RuntimeException: runtime
	at com.xncoding.trans.service.UserService.errMethod(UserService.java:32) ~[classes/:na]
	at com.xncoding.trans.service.UserService.updateUserError(UserService.java:27) ~[classes/:na]
```

查看数据库中记录：`1|admin|123`，没有变动，说明回滚成功。

## 异常不会滚

你还可以指定特定异常不回滚，比如自定义一个MyException，抛出这个异常不回滚。

```java
public class MyException extends RuntimeException {
    public MyException() {
        super();
    }
    public MyException(String runtime) {
        super(runtime);
    }
}
```

然后通过指定这个异常不回滚：

```java
@Transactional(readOnly = false, noRollbackFor = {MyException.class})
public void updateUserError2(User user) {
    userMapper.updateById(user);
    errMethod2(); // 执行一个会抛出自定义异常的方法
}

private void errMethod2() {
    System.out.println("error");
    throw new MyException("runtime");
}
```

然后再定义一个url来验证：

```java
@RequestMapping("/errorUpdate2")
public Object second() {
    User user = new User();
    user.setId(1);
    user.setUsername("admin");
    user.setPassword("admin");
    userService.updateUserError(user);
    return "second controller";
}
```

重启服务器，访问地址：`http://localhost:8092/errorUpdate2`

控制台仍然报异常：

```
Caused by: com.xncoding.trans.exception.MyException: runtime
	at com.xncoding.trans.service.UserService.errMethod2(UserService.java:43) ~[classes/:na]
	at com.xncoding.trans.service.UserService.updateUserError2(UserService.java:34) ~[classes/:na]
```

看看数据库中记录：`1|admin|admin`，更改成功，说明抛出这个MyException异常后并不会回滚。

## GitHub源码

[springboot-transaction](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-transaction)

