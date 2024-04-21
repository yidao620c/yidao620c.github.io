---
title: SpringBoot系列 - Redis数据库
date: 2017-07-30 12:21:58 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

在互联网场景下，尤其 2C 端大流量场景下，需要将一些经常展现和不会频繁变更的数据，存放在存取速率更快的地方。
缓存就是一个存储器，在技术选型中，常用 Redis 作为缓存数据库。缓存主要是在获取资源方便性能优化的关键方面。

如果使用Redis缓存技术，SpringBoot中有两种方式实现缓存，一个是上一篇中通过CacheManager实现，
不过这个是对于简单的缓存场景，而更为强大的是通过`RedisTemplate`来直接操作Redis数据库实现缓存。

Redis 是一个高性能的 key-value 数据库。GitHub 地址：<https://github.com/antirez/redis> 。
<!-- more -->

Github 是这么描述的：

Redis is an in-memory database that persists on disk. The data model is key-value,
but many different kind of values are supported: Strings, Lists, Sets, Sorted Sets, Hashes, HyperLogLogs, Bitmaps.

缓存的应用场景有哪些呢？

比如常见的电商场景，根据商品 ID 获取商品信息时，店铺信息和商品详情信息就可以缓存在 Redis，直接从 Redis 获取。
减少了去数据库查询的次数。但会出现新的问题，就是如何对缓存进行更新？这就是下面要讲的。

## 缓存更新策略

参考[《缓存更新的套路》](http://coolshell.cn/articles/17416.html) ，缓存更新的模式有四种：

1. Cache aside
2. Read through
3. Write through
4. Write behind caching

这里我们使用的是 Cache Aside 策略，从三个维度：

* 失效：应用程序先从cache取数据，没有得到，则从数据库中取数据，成功后，放到缓存中。
* 命中：应用程序从cache中取数据，取到后返回。
* 更新：先把数据存到数据库中，成功后，再让缓存失效。

大致流程如下：

获取商品详情举例

1. 从商品 Cache 中获取商品详情，如果存在，则返回获取 Cache 数据返回。
1. 如果不存在，则从商品 DB 中获取。获取成功后，将数据存到 Cache 中。则下次获取商品详情，就可以从 Cache 就可以得到商品详情数据。
1. 从商品 DB 中更新或者删除商品详情成功后，则从缓存中删除对应商品的详情缓存

![](https://xnstatic-1253397658.file.myqcloud.com/sb-redis01.png)

## 添加maven依赖

这里仍然是MyBatis做数据库DAO操作，Redis做缓存操作。

SpringBoot 2开始默认的Redis客户端实现是Lettuce，同时你需要添加commons-pool2的依赖。

```xml

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-core</artifactId>
        <version>${jackson-databind-version}</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson-databind-version}</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-annotations</artifactId>
        <version>${jackson-databind-version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>com.vaadin.external.google</groupId>
                <artifactId>android-json</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
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
</dependencies>
```

注意上面我添加了Jackson的依赖，后面要用到。

### 配置application.yml

增加 Redis 相关配置

```yaml
spring:
  cache:
    type: REDIS
    redis:
      cache-null-values: false
      time-to-live: 600000ms
      use-key-prefix: true
    cache-names: userCache,allUsersCache
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    lettuce:
      shutdown-timeout: 200ms
      pool:
        max-active: 7
        max-idle: 7
        min-idle: 2
        max-wait: -1ms
```

对应的配置类：

`org.springframework.boot.autoconfigure.data.redis.RedisProperties`

## 添加配置类

这里自定义RedisTemplate的配置类，主要是想使用Jackson替换默认的序列化机制：

```java
@Configuration
public class RedisConfig {
    /**
     * redisTemplate 默认使用JDK的序列化机制, 存储二进制字节码, 所以自定义序列化类
     * @param redisConnectionFactory redis连接工厂类
     * @return RedisTemplate
     */
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        // 使用Jackson2JsonRedisSerialize 替换默认序列化
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);

        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);

        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);

        // 设置value的序列化规则和 key的序列化规则
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
}
```

## 使用演示

MyBatis实现的DAO层，以及User实体类我就不贴在这里了，只贴Service核心增删改查操作：

```java
@Service
@Transactional
public class UserService {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Resource
    private UserMapper userMapper;

    @Resource
    private RedisTemplate<String, User> redisTemplate;


    /**
     * 创建用户
     * 不会对缓存做任何操作
     */
    public void createUser(User user) {
        logger.info("创建用户start...");
        userMapper.insert(user);
    }

    /**
     * 获取用户信息
     * 如果缓存存在，从缓存中获取城市信息
     * 如果缓存不存在，从 DB 中获取城市信息，然后插入缓存
     *
     * @param id 用户ID
     * @return 用户
     */
    public User getById(int id) {
        logger.info("获取用户start...");
        // 从缓存中获取用户信息
        String key = "user_" + id;
        ValueOperations<String, User> operations = redisTemplate.opsForValue();

        // 缓存存在
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            User user = operations.get(key);
            logger.info("从缓存中获取了用户 id = " + id);
            return user;
        }
        // 缓存不存在，从 DB 中获取
        User user = userMapper.selectById(id);
        // 插入缓存
        operations.set(key, user, 10, TimeUnit.SECONDS);

        return user;
    }

    /**
     * 更新用户
     * 如果缓存存在，删除
     * 如果缓存不存在，不操作
     *
     * @param user 用户
     */
    public void updateUser(User user) {
        logger.info("更新用户start...");
        userMapper.updateById(user);
        int userId = user.getId();
        // 缓存存在，删除缓存
        String key = "user_" + userId;
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            redisTemplate.delete(key);
            logger.info("更新用户时候，从缓存中删除用户 >> " + userId);
        }
    }

    /**
     * 删除用户
     * 如果缓存中存在，删除
     */
    public void deleteById(int id) {
        logger.info("删除用户start...");
        userMapper.deleteById(id);

        // 缓存存在，删除缓存
        String key = "user_" + id;
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            redisTemplate.delete(key);
            logger.info("删除用户时候，从缓存中删除用户 >> " + id);
        }
    }

}
```

RedisTemplate 封装了 RedisConnection，具有连接管理，序列化和 Redis 操作等功能。
还有专门针对 String 的模板对象 StringRedisTemplate。

Redis 操作视图接口类用的是 ValueOperations，对应的是 Redis String/Value 操作。

还有其他的操作视图，ListOperations、SetOperations、ZSetOperations 和 HashOperations 。
ValueOperations 插入缓存是可以设置失效时间，这里设置的失效时间是 10 s。

然后写个测试类测试运行看看效果：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@Transactional
public class UserServiceTest {
    @Autowired
    private UserService userService;
    @Test
    public void testCache() {
        int id = new Random().nextInt(1000);
        User user = new User(id, "admin", "admin");
        userService.createUser(user);
        User user1 = userService.getById(id); // 第1次访问
        assertEquals(user1.getPassword(), "admin");
        User user2 = userService.getById(id); // 第2次访问
        assertEquals(user2.getPassword(), "admin");
        user.setPassword("123456");
        userService.updateUser(user);
        User user3 = userService.getById(id); // 第3次访问
        assertEquals(user3.getPassword(), "123456");
        userService.deleteById(id);
        assertNull(userService.getById(id));
    }
}
```

运行SpringBoot集成测试，查看日志如下：

```
创建用户start...
==>  Preparing: INSERT INTO t_user ( id, username, `password` ) VALUES ( ?, ?, ? )
==> Parameters: 89(Integer), admin(String), admin(String)
<==    Updates: 1
获取用户start...
Starting without optional epoll library
Starting without optional kqueue library
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 89(Integer)
<==      Total: 1
获取用户start...
从缓存中获取了用户 id = 89
更新用户start...
==>  Preparing: UPDATE t_user SET username=?, `password`=? WHERE id=?
==> Parameters: admin(String), 123456(String), 89(Integer)
<==    Updates: 1
更新用户时候，从缓存中删除用户 >> 89
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 89(Integer)
<==      Total: 1
删除用户start...
==>  Preparing: DELETE FROM t_user WHERE id=?
==> Parameters: 89(Integer)
<==    Updates: 1
更新用户时候，从缓存中删除用户 >> 89
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 89(Integer)
<==      Total: 0
Rolled back transaction for test: [DefaultTestContext@6e20b53a testClass = UserServiceTest
Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@503f91c3
{dataSource-1} closed
```

可以看出第一次取的时候，缓存未命中，会从DB中取数据，而第2次取的时候缓存命中直接从缓存中取出来。
后面不管是更新和删除，都会从缓存中删除。再去取的时候缓存未命中，从DB中取最新的。

## GitHub源码

[springboot-redis](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-redis)

