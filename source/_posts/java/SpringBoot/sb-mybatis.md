---
title: SpringBoot系列 - 集成MyBatis
date: 2017-07-02 10:23:53 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。
MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs映射成数据库中的记录。

使用MyBatis的时候需要自己手动编写SQL语句，也有代码自动生成工具来简化开发，我一般会使用Mybatis-Plus增强工具包来简化MyBatis的开发。

Mybatis-Plus官网：<https://github.com/baomidou/mybatis-plus>

同时它还提供了与SpringBoot的集成starter，非常的方便，本篇我将讲解如何在SpringBoot中集成MyBatis-Plus。
<!-- more -->

## maven依赖

```xml

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <druid.version>1.1.2</druid.version>
    <mysql-connector.version>8.0.7-dmr</mysql-connector.version>
    <mybatis-plus.version>2.1.8</mybatis-plus.version>
    <mybatisplus-spring-boot-starter.version>1.0.5</mybatisplus-spring-boot-starter.version>
</properties>

<dependencies>
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

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest-all</artifactId>
    <version>1.3</version>
    <scope>test</scope>
</dependency>
</dependencies>
```

这里我使用mysql数据库，阿里的druid连接池。

## 初始化表

关于mysql数据库的安装我这里就不讲了，只提供schema.sql文件初始化一个用户表，用作演示：

```sql
CREATE
DATABASE IF NOT EXISTS pos default charset utf8 COLLATE utf8_general_ci;
SET
FOREIGN_KEY_CHECKS=0;
USE
pos;

-- 用户表
DROP TABLE IF EXISTS `t_user`;
CREATE TABLE `t_user`
(
    `id`           INT(11) PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    `username`     VARCHAR(32) NOT NULL COMMENT '账号',
    `name`         VARCHAR(16)  DEFAULT '' COMMENT '名字',
    `password`     VARCHAR(128) DEFAULT '' COMMENT '密码',
    `salt`         VARCHAR(64)  DEFAULT '' COMMENT 'md5密码盐',
    `phone`        VARCHAR(32)  DEFAULT '' COMMENT '联系电话',
    `tips`         VARCHAR(255) COMMENT '备注',
    `state`        TINYINT(1) DEFAULT 1 COMMENT '状态 1:正常 2:禁用',
    `created_time` DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_time` DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='用户表';
INSERT INTO `t_user`
VALUES (1, 'admin', '系统管理员', '123456', 'www', '17890908889', '系统管理员', 1, '2017-12-12 09:46:12',
        '2017-12-12 09:46:12');
INSERT INTO `t_user`
VALUES (2, 'aix', '张三', '123456', 'eee', '17859569358', '', 1, '2017-12-12 09:46:12', '2017-12-12 09:46:12');
```

## 配置文件

修改application.yml配置文件，注意是mysql连接信息和mybatis-plus的配置：

```yaml
###################  mybatis-plus配置  ###################
mybatis-plus:
  mapper-locations: classpath*:com/xncoding/pos/common/dao/repository/mapping/*.xml
  typeAliasesPackage: >
    com.xncoding.pos.common.dao.entity
  global-config:
    id-type: 0  # 0:数据库ID自增   1:用户输入id  2:全局唯一id(IdWorker)  3:全局唯一ID(uuid)
    db-column-underline: false
    refresh-mapper: true
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: true #配置的缓存的全局开关
    lazyLoadingEnabled: true #延时加载的开关
    multipleResultSetsEnabled: true #开启的话，延时加载一个属性时会加载该对象全部属性，否则按需加载属性

spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/pos?useSSL=false&autoReconnect=true&tinyInt1isBit=false&useUnicode=true&characterEncoding=utf8
    username: root
    password: 123456
```

主要解释两个配置：

1. mapper-locations 这个是你定义的mapping文件，支持通配符，如果有多个用逗号隔开。
2. typeAliasesPackage 这个定义实体类所在的包名，或者其他能使用别名的类所在的包。

## 配置类

由于使用到的Druid连接池的配置项非常多，可先定义一个`DruidProperties`属性类，具体的数据源配置我就不贴了。
然后再定义`MybatisPlusConfig`如下：

```java
@Configuration
@EnableTransactionManagement(order = 2)
@MapperScan(basePackages = {
        "com.xncoding.pos.common.dao.repository",
        "com.xncoding.pos.dao.repository"})
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

    /**
     * mybatis-plus分页插件
     */
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}
```

这里要说明一下：

@EnableTransactionManagement开启了事务支持。

@MapperScan注解扫描DAO类，支持多个包，用逗号隔开。

然后在里面定义了一个数据源的类，另外还定义了分页插件，这个用在分页查询上面，本篇演示还用不到。

## 实体类

```java
@TableName(value = "t_user")
public class User extends Model<User> {

private static final long serialVersionUID = 1L;
    /**
     * 主键ID
     */
    @TableId(value="id", type= IdType.AUTO)
    private Integer id;
    /**
     * 账号
     */
    private String username;
    /**
     * 名字
     */
    private String name;
    /**
     * 密码
     */
    private String password;
    /**
     * md5密码盐
     */
    private String salt;
    /**
     * 联系电话
     */
    private String phone;
    /**
     * 备注
     */
    private String tips;
    /**
     * 状态 1:正常 2:禁用
     */
    private Integer state;
    /**
     * 创建时间
     */
    private Date createdTime;
    /**
     * 更新时间
     */
    private Date updatedTime;
    
    // 省略getter/setter
}
```

## DAO类

接下来定义DAO操作类：

```java
public interface UserMapper extends BaseMapper<User> {

}
```

这个接口继承 `BaseMapper<User>`后即可获得常用的增删改查的方法，
如果有其他复杂的操作，就再定义自己的方法，并同时定义mapping文件即可。

这里演示比较简单，无需自己再去写mapping文件。

## 业务逻辑Servcie

接下来定义业务逻辑处理类，并将DAO类注入进去：

```java
@Service
public class UserService {

    @Resource
    private UserMapper userMapper;

    /**
     * 通过ID查找用户
     * @param id
     * @return
     */
    public User findById(Integer id) {
        return userMapper.selectById(id);
    }

    /**
     * 新增用户
     * @param user
     */
    public void insertUser(User user) {
        userMapper.insert(user);
    }

    /**
     * 修改用户
     * @param user
     */
    public void updateUser(User user) {
        userMapper.updateById(user);
    }

    /**
     * 删除用户
     * @param id
     */
    public void deleteUser(Integer id) {
        userMapper.deleteById(id);
    }

}
```

## 测试类

一切搞定后，接下来开始编写测试用例试试看：

```java
/**
 * 测试
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    private static final Logger log = LoggerFactory.getLogger(ApplicationTests.class);

    @Resource
    private UserService userService;

    /**
     * 测试增删改查
     */
    @Test
    public void test() {
        User user = new User();
        user.setUsername("xiaoxx");
        user.setName("小星星");
        user.setPassword("222222");
        user.setPhone("13890907676");
        userService.insertUser(user);

        User user1 = userService.findById(user.getId());
        assertThat(user1.getUsername(), is("xiaoxx"));
        assertThat(user1.getName(), is("小星星"));

        user1.setPassword("888888");
        userService.updateUser(user1);
        User user2 = userService.findById(user.getId());
        assertThat(user2.getPassword(), is("888888"));

        userService.deleteUser(user.getId());

        User user3 = userService.findById(user.getId());
        assertThat(user3, nullValue());

    }
}
```

先创建一个用户，插入到表中，使用了数据库的自增主键，调用`insertUser`成功后，user的id会被自动设置上。
然后再通过id查询用户，确保能查询到。后面更新用户并确认，删除用户并确认。

运行测试用例，结果是green bar，测试通过。

## GitHub源码

[springboot-mybatis](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-mybatis)

