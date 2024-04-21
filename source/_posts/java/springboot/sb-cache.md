---
title: SpringBoot系列 - 缓存
date: 2017-07-28 20:12:33 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

内存的速度远远大于硬盘的速度，当我们需要重复获取相同的数据的时候，一次又一次的请求数据库或远程服务，
导致大量时间都消耗在数据库查询或远程方法调用上面，性能下降，这时候就需要使用到缓存技术了。

本文介绍SpringBoot 如何使用redis做缓存，如何对redis缓存进行定制化配置(如key的有效期)以及初始化redis做缓存。
使用具体的代码介绍了`@Cacheable`，`@CacheEvict`，`@CachePut`，`@CacheConfig`等注解及其属性的用法。
<!-- more -->

## Spring缓存支持

Spring定义了`org.springframework.cache.CacheManager` 和 `org.springframework.cache.Cache` 接口来统一不同缓存技术。
其中CacheManager是Spring提供的各种缓存技术抽象接口，内部使用Cache接口进行缓存的增删改查操作，我们一般不会直接和Cache打交道。

针对不同的缓存技术，Spring有不同的CacheManager实现类，定义如下表：

 CacheManager              | 描述                                                
---------------------------|---------------------------------------------------
 SimpleCacheManager        | 使用简单的Collection存储缓存数据，用来做测试用                      
 ConcurrentMapCacheManager | 使用ConcurrentMap存储缓存数据                             
 EhCacheCacheManager       | 使用EhCache作为缓存技术                                   
 GuavaCacheManager         | 使用Google Guava的GuavaCache作为缓存技术                   
 JCacheCacheManager        | 使用JCache(JSR-107)标准的实现作为缓存技术，比如Apache Commons JCS 
 RedisCacheManager         | 使用Redis作为缓存技术                                     

在我们使用任意一个实现的CacheManager的时候，需要注册实现Bean：

```java
/**
 * EhCache的配置
 */
@Bean
public EhCacheCacheManager cacheManager(CacheManager cacheManager) {
    return new EhCacheCacheManager(cacheManager);
}
```

当然，各种缓存技术都有很多其他配置，但是配置cacheManager是必不可少的。

## 声明式缓存注解

Spring提供4个注解来声明缓存规则，如下表所示：

 注解          | 说明                                            
-------------|-----------------------------------------------
 @Cacheable  | 方法执行前先看缓存中是否有数据，如果有直接返回。如果没有就调用方法，并将方法返回值放入缓存 
 @CachePut   | 无论怎样都会执行方法，并将方法返回值放入缓存                        
 @CacheEvict | 将数据从缓存中删除                                     
 @Caching    | 可通过此注解组合多个注解策略在一个方法上面                         

@Cacheable 、@CachePut 、@CacheEvict都有value属性，指定要使用的缓存名称，而key属性指定缓存中存储的键。

## 集成Redis缓存

接下来将讲解如何集成redis来实现缓存，现在使用的是最新的SpringBoot 2，改动还是比较大的。

### 安装redis

安装和配置redis服务器网上很多教程，这里就不多讲了。在linux服务器上面安装一个redis，启动后端口号为默认的6379。

### 添加maven依赖

SpringBoot 2开始默认的Redis客户端实现是Lettuce，同时你需要添加commons-pool2的依赖。

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
<groupId>org.apache.commons</groupId>
<artifactId>commons-pool2</artifactId>
</dependency>
```

### 配置application.yml

* 指定缓存的类型
* 配置redis的服务器信息
* 配置lettuce连接池信息

```yaml
spring:
  profiles: dev
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

这里我还指定了缓存名称列表：userCache,allUsersCache

### 缓存配置类

重新配置`RedisCacheManager`，使用新的自定义配置值：

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private Environment env;

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisConf = new RedisStandaloneConfiguration();
        redisConf.setHostName(env.getProperty("spring.redis.host"));
        redisConf.setPort(Integer.parseInt(env.getProperty("spring.redis.port")));
        redisConf.setPassword(RedisPassword.of(env.getProperty("spring.redis.password")));
        return new LettuceConnectionFactory(redisConf);
    }

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        RedisCacheConfiguration cacheConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofSeconds(600))
                .disableCachingNullValues();
        return cacheConfig;
    }

    @Bean
    public RedisCacheManager cacheManager() {
        RedisCacheManager rcm = RedisCacheManager.builder(redisConnectionFactory())
                .cacheDefaults(cacheConfiguration())
                .transactionAware()
                .build();
        return rcm;
    }
}
```

### keyGenerator

一般来讲我们使用key属性就可以满足大部分要求，但是如果你还想更好的自定义key，可以实现keyGenerator。

这个属性为定义key生成的类，和key属性不能同时存在。

在`RedisCacheConfig`配置类中添加我自定义的KeyGenerator：

```java
/**
 * 自定义缓存key的生成类实现
 */
@Bean(name = "myKeyGenerator")
public KeyGenerator myKeyGenerator() {
    return new KeyGenerator() {
        @Override
        public Object generate(Object o, Method method, Object... params) {
            logger.info("自定义缓存，使用第一参数作为缓存key，params = " + Arrays.toString(params));
            // 仅仅用于测试，实际不可能这么写
            return params[0];
        }
    };
}
```

经过以上配置后，redis缓存管理对象已经生成，下面简单介绍如何使用。

## 缓存注解

缓存是通过注解实现的，这里详细讲解一下这几个缓存注解，以及注解参数含义。

### @Cacheable

这个注解含义是方法结果会被放入缓存，并且一旦缓存后，下一次调用此方法，会通过key去查找缓存是否存在，如果存在就直接取缓存值，不再执行方法。

这个注解有几个参数值，定义如下

 参数           | 解释                             
--------------|--------------------------------
 cacheNames   | 缓存名称                           
 value        | 缓存名称的别名                        
 condition    | Spring SpEL 表达式，用来确定是否缓存       
 key          | SpEL 表达式，用来动态计算key             
 keyGenerator | Bean 名字，用来自定义key生成算法，跟key不能同时用 
 unless       | SpEL 表达式，用来否决缓存，作用跟condition相反 
 sync         | 多线程同时访问时候进行同步                  

在计算key、condition或者unless的值得时候，可以使用到以下的特有的SpEL表达式

 表达式               | 解释        
-------------------|-----------
 #result           | 表示方法的返回结果 
 #root.method      | 当前方法      
 #root.target      | 目标对象      
 #root.caches      | 被影响到的缓存列表 
 #root.methodName  | 方法名称简称    
 #root.targetClass | 目标类       
 #root.args[x]     | 方法的第x个参数  

### @CachePut

该注解在执行完方法后会触发一次缓存put操作，参数跟@Cacheable一致

### @CacheEvict

该注解在执行完方法后会触发一次缓存evict操作，参数除了@Cacheable里的外，还有个特殊的`allEntries`，
表示将清空缓存中所有的值。

## 使用

在service中定义增删改的几个常见方法，通过注解实现缓存：

```java
@Service
@Transactional
public class UserService {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Resource
    private UserMapper userMapper;

    /**
     * cacheNames 设置缓存的值
     * key：指定缓存的key，这是指参数id值。key可以使用spEl表达式
     *
     * @param id
     * @return
     */
    @Cacheable(value = "userCache", key = "#id", unless="#result == null")
    public User getById(int id) {
        logger.info("获取用户start...");
        return userMapper.selectById(id);
    }

    @Cacheable(value = "allUsersCache", unless = "#result.size() == 0")
    public List<User> getAllUsers() {
        logger.info("获取所有用户列表");
        return userMapper.selectList(null);
    }

    /**
     * 创建用户，同时使用新的返回值的替换缓存中的值
     * 创建用户后会将allUsersCache缓存全部清空
     */
    @Caching(
            put = {@CachePut(value = "userCache", key = "#user.id")},
            evict = {@CacheEvict(value = "allUsersCache", allEntries = true)}
    )
    public User createUser(User user) {
        logger.info("创建用户start..., user.id=" + user.getId());
        userMapper.insert(user);
        return user;
    }

    /**
     * 更新用户，同时使用新的返回值的替换缓存中的值
     * 更新用户后会将allUsersCache缓存全部清空
     */
    @Caching(
            put = {@CachePut(value = "userCache", key = "#user.id")},
            evict = {@CacheEvict(value = "allUsersCache", allEntries = true)}
    )
    public User updateUser(User user) {
        logger.info("更新用户start...");
        userMapper.updateById(user);
        return user;
    }

    /**
     * 对符合key条件的记录从缓存中移除
     * 删除用户后会将allUsersCache缓存全部清空
     */
    @Caching(
            evict = {
                    @CacheEvict(value = "userCache", key = "#id"),
                    @CacheEvict(value = "allUsersCache", allEntries = true)
            }
    )
    public void deleteById(int id) {
        logger.info("删除用户start...");
        userMapper.deleteById(id);
    }

}
```

然后写个测试类：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@Transactional
public class UserServiceTest {
    @Autowired
    private UserService userService;
    @Test
    public void testCache() {
        // 创建一个用户admin
        int id = new Random().nextInt(1000);
        User user = new User(id, "admin", "admin");
        userService.createUser(user);

        // 再创建一个用户xiong
        int id2 = new Random().nextInt(1000);
        User user2 = new User(id2, "xiong", "neng");
        userService.createUser(user2);

        // 查询所有用户列表
        List<User> list = userService.getAllUsers();
        assertEquals(list.size(), 2);

        // 两次访问看看缓存命中情况
        User user3 = userService.getById(id); // 第1次访问
        assertEquals(user3.getPassword(), "admin");
        User user4 = userService.getById(id); // 第2次访问
        assertEquals(user4.getPassword(), "admin");

        // 更新用户密码
        user4.setPassword("123456");
        userService.updateUser(user4);

        // 更新完成后再次访问用户
        User user5 = userService.getById(id); // 第4次访问
        assertEquals(user5.getPassword(), "123456");

        // 删除用户admin
        userService.deleteById(id);
        assertNull(userService.getById(id));
    }
}
```

下面是测试的打印日志一部分：

```
创建用户start..., user.id=444
==>  Preparing: INSERT INTO t_user ( id, username, `password` ) VALUES ( ?, ?, ? )
==> Parameters: 444(Integer), admin(String), admin(String)
<==    Updates: 1
创建用户start..., user.id=895
==>  Preparing: INSERT INTO t_user ( id, username, `password` ) VALUES ( ?, ?, ? )
==> Parameters: 895(Integer), xiong(String), neng(String)
<==    Updates: 1
Starting without optional epoll library
Starting without optional kqueue library
获取所有用户列表
==>  Preparing: SELECT id AS id,username,`password` FROM t_user
==> Parameters:
<==      Total: 2
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 444(Integer)
<==      Total: 1
获取用户start...
更新用户start...
==>  Preparing: UPDATE t_user SET username=?, `password`=? WHERE id=?
==> Parameters: admin(String), 123456(String), 444(Integer)
<==    Updates: 1
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 444(Integer)
<==      Total: 1
删除用户start...
==>  Preparing: DELETE FROM t_user WHERE id=?
==> Parameters: 444(Integer)
<==    Updates: 1
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` FROM t_user WHERE id=?
==> Parameters: 444(Integer)
<==      Total: 0
Rolled back transaction for test: [DefaultTestContext@44ebcd03 testClass = UserServiceTest,
Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@1a5a4e19:
{dataSource-1} closed
```

可以看到，第2次、第3次获取的时候并没有执行方法，说明缓存生效了。后面更新会同时更新缓存，取出来的也是更新后的数据。

## 切换缓存技术

得益于SpringBoot的自动配置机制，切换缓存技术除了替换相关maven依赖包和配置Bean外，使用方式和实例中一样，
不需要修改业务代码。如果你要切换到其他缓存技术非常简单。

**EhCache**

当我们需要使用EhCache作为缓存技术的时候，只需要在pom.xml中添加EhCache的依赖：

```xml

<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcahe</artifactId>
</dependency>
```

EhCache的配置文件ehcache.xml只需要放到类路径下面，SpringBoot会自动扫描，例如：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false" monitoring="autodetect"
         dynamicConfig="true">

    <diskStore path="java.io.tmpdir/ehcache"/>

    <defaultCache
            maxElementsInMemory="50000"
            eternal="false"
            timeToIdleSeconds="3600"
            timeToLiveSeconds="3600"
            overflowToDisk="true"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
    />

    <cache name="authorizationCache"
           maxEntriesLocalHeap="2000"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="3600"
           overflowToDisk="false"
           statistics="true">
    </cache>
</ehcache>
```

SpringBoot会为我们自动配置`EhCacheCacheManager`这个Bean，不过你也可以自己定义。

**Guava**

当我们需要Guava作为缓存技术的时候，只需要在pom.xml中增加Guava的依赖即可：

```xml

<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>18.0</version>
</dependency>
```

SpringBoot会为我们自动配置`GuavaCacheManager`这个Bean。

**Redis**

最后还提一点，本篇采用Redis作为缓存技术，添加了依赖：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

SpringBoot会为我们自动配置`RedisCacheManager`这个Bean，同时还会配置`RedisTemplate`这个Bean。
后面这个Bean就是下一篇要讲解的操作Redis数据库用，这个就比单纯注解缓存强大和灵活的多了。

## 参考文章

[Spring Boot Redis Cache](https://www.concretepage.com/spring-boot/spring-boot-redis-cache)

## GitHub源码

[springboot-cache](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-cache)

