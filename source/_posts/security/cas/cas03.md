---
title: CAS教程 - 定制开发
date: 2019-01-05 21:15:34 +0800
toc: true
categories: [ 网络安全 ]
tags: [ CAS ]
---

现在开始对CasServer进行二次开发，比如如何设置数据库连接，如何使用数据库的用户名和密码登录，
如何使用Restful API方式实现SSO，如何自定义服务，如何自定义登陆界面等等。接下来将逐步介绍。

Cas官方说明，如果你想对它默认项目有所更改，那么就使用覆盖它路径的方式进行。

更改CAS的配置既可以修改cas.properties文件，也能修改默认的application.properties。
为了配置集中处理，我会把默认的application.properties配置文件提出来，只修改它，并把cas.properties内容都移到里面去。
<!-- more -->

## 修改默认用户名和密码

在cas-overlay-template-master项目中，新建一个`src/main/resources`目录，将resources目录设置成资源目录。
然后将`target/war/work/org.apereo.cas/cas-server-webapp-tomcat/WEB-INF/classes/application.properties`
复制到resources目录中。并将前面修改的`cas.properties`内容复制到里面去。

然后找到最后的`cas.authn.accept.users=casuser::Mellon`，改成我自己的：

```
cas.authn.accept.users=xiongneng::xiongneng
```

重新build后放到tomcat容器中，再看看效果。使用新的用户名密码登录成功！

## yml配置文件

将war包下的`application.properties`文件、`application.yml`都移到resources目录，然后将properties文件置空。
cas默认会先读取properties文件，然后再读取yml文件，这样就完美的解决了通过yml启动cas了。

下一步将properties文件内容转成yml格式复制到yml中，推荐转换网站：<http://www.toyaml.com>

```yaml
cas:
  adminPagesSecurity:
    ip: 127.0.0.1
  authn:
    accept:
      users: xiongneng::xiongneng
  server:
    name: https://cas.server.com:8443
    prefix: https://cas.server.com:8443/cas
  后面省略》。。
```

重新build后启动再访问看看。

## 单点登录JDBC认证

之前介绍过在application.properties文件中，修改默认的静态用户名和密码。这节分析下，取消静态登陆方式，
该用从数据库链接获取用户进行登录认证。

CAS支持各种数据库，我这里选用MySQL进行实践。CAS支持多种JDBC的认证方案，这里推荐方案是通过盐等手段进行加密再进行匹配。

常用单向加密算法：MD5、SHA、HMAC，推荐加密策略为单向加密算法(密码+动态盐+私有盐)*加密次数。

先创建cas数据库：

```sql
CREATE
DATABASE IF NOT EXISTS cas default charset utf8 COLLATE utf8_general_ci;
```

然后创建用户表：

```sql
create table sys_user
(
    `id`       int(11) not null auto_increment,
    `username` varchar(30) not null,
    `password` varchar(64) not null,
    `email`    varchar(50),
    `address`  varchar(100),
    `age`      int,
    `expired`  int,
    #是否过期  0否 1是
        `disabled` int,
    #是否禁用  0否 1是
        `locked` int,
    #是否锁定  0否 1是
        primary key (`id`)
) engine=innodb auto_increment=1;
```

插入几条数据，这里密码就采用简单点的MD5散列值：

```sql
/*123*/
insert into sys_user
values ('1', 'admin', '202cb962ac59075b964b07152d234b70', 'admin@foxmail.com', '广州天河', 24, 0, 0, 0);
/*12345678*/
insert into sys_user
values ('2', 'zhangsan', '25d55ad283aa400af464c76d713c07ad', 'zhangsan@foxmail.com', '广州越秀', 26, 0, 0, 0);
/*1234*/
/*禁用账户*/
insert into sys_user
values ('3', 'zhaosi', '81dc9bdb52d04dc20036dbd8313ed055', 'zhaosi@foxmail.com', '广州海珠', 25, 0, 1, 0);
/*12345*/
/*过期账户*/
insert into sys_user
values ('4', 'wangwu', '827ccb0eea8a706c4c34a16891f84e7b', 'wangwu@foxmail.com', '广州番禺', 27, 1, 0, 0);
/*123*/
/*锁定账户*/
insert into sys_user
values ('5', 'boss', '202cb962ac59075b964b07152d234b70', 'boss@foxmail.com', '深圳', 30, 0, 0, 1);
```

我们需要在原来的pom.xml基础上，添加数据库驱动，以及jdbc的支持：

```xml

<properties>
    <cas.version>5.3.8</cas.version>
    <mysql.driver.version>8.0.15</mysql.driver.version>
</properties>

        <!--数据库认证相关 start-->
<dependency>
<groupId>org.apereo.cas</groupId>
<artifactId>cas-server-support-jdbc-drivers</artifactId>
<version>${cas.version}</version>
</dependency>
<dependency>
<groupId>org.apereo.cas</groupId>
<artifactId>cas-server-support-jdbc</artifactId>
<version>${cas.version}</version>
</dependency>
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>${mysql.driver.version}</version>
</dependency>
```

里面有个依赖找不到，我就手动下载安装到本地：

```bash
mvn install:install-file -Dfile=D:/download/xmlsectool-2.0.0.jar \
-DgroupId=net.shibboleth.tool -DartifactId=xmlsectool -Dversion=2.0.0 -Dpackaging=jar
```

接下来配置application.yml文件。

1 首先禁用默认的用户名和密码。

2 增加如下配置

```yaml
cas:
  adminPagesSecurity:
    ip: 127.0.0.1
  authn:
    #    accept:
    #      users: test::test
    jdbc:
      query:
        - dialect: org.hibernate.dialect.MySQLDialect
          driverClass: com.mysql.cj.jdbc.Driver
          fieldDisabled: disabled
          fieldExpired: expired
          fieldPassword: password
          sql: select * from sys_user where username=?
          url: jdbc:mysql://127.0.0.1:3306/cas?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
          user: test
          password: test
          passwordEncoder:
            type: DEFAULT
            characterEncoding: UTF-8
            encodingAlgorithm: MD5
```

以上配置，如驱动，查询数据库等等需要根据不同的场景进行调整

1. 若密码无加密，调整passwordEncoder.type=NONE
1. 若密码加密策略为SHA，调整passwordEncoder.encodingAlgorithm=SHA
1. 若算法为自定义，实现`org.springframework.security.crypto.password.PasswordEncoder`接口，
   并且把类名配置在`passwordEncoder.type`

**Encode Database Authentication 编码加密**

对密码进行盐值处理再加密，增加了反查难度，如上面的例子，对密码只是简单的加密，不同的帐号有可能相同的值，
能判断出密码是一致，但通过此方案，大大增加了难度，所以安全系数也高了许多，推荐策略。

数据库新增表：

```sql
/*
 * 账号加盐表
 */
create table sys_user_encode
(
    `id`       int(11) not null auto_increment,
    `username` varchar(30) not null,
    `password` varchar(64) not null,
    `email`    varchar(50),
    `address`  varchar(100),
    `age`      int,
    `expired`  int,
    #是否过期  0否 1是
        `disabled` int,
    #是否禁用  0否 1是
        `locked` int,
    #是否锁定  0否 1是
        primary key (`id`)
) engine=innodb auto_increment=1;
```

插入数据：

```sql
#
明文密码 123
insert into sys_user_encode values ('1', 'admin_en', 'bfb194d5bd84a5fc77c1d303aefd36c3', 'huang.wenbin@foxmail.com', '江门蓬江', 24, 0, 0, 0);
#
明文密码 12345678
insert into sys_user_encode values ('2', 'zhangsan_en', '68ae075edf004353a0403ee681e45056',  'zhangsan@foxmail.com', '深圳宝安', 21, 0, 0, 0);
#明文密码
1234
insert into sys_user_encode values ('4', 'wangwu_en', '44b907d6fee23a552348eabf5fcf1ac7',  'wangwu@foxmail.com', '佛山顺德', 19, 1, 0, 0);
```

将之前jdbc.query的配置改成jdbc.encode：

```yaml
cas:
  adminPagesSecurity:
    ip: 127.0.0.1
  authn:
    #    accept:
    #      users: test::test
    jdbc:
      encode:
        - dialect: org.hibernate.dialect.MySQLDialect
          driverClass: com.mysql.cj.jdbc.Driver
          disabledFieldName: disabled
          expiredFieldName: expired
          passwordFieldName: password
          sql: select * from sys_user_encode where username=?
          url: jdbc:mysql://127.0.0.1:3306/cas?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
          user: test
          password: test
          algorithmName: MD5 #对处理盐值后的算法
          numberOfIterations: 2 #加密迭代次数
          # numberOfIterationsFieldName: #该列名的值可替代上面的值，但对密码加密时必须取该值进行处理
          saltFieldName: username #盐值固定列
          staticSalt: . #静态盐值
```

## 自定义密码验证

加密的类，必须实现`org.springframework.security.crypto.password.PasswordEncoder`，因为验证的时候，cas就是调用接口，然后验证是否正确。
现在自己写一个实现了这个接口的类：

```java
package com.xncoding.cas;

import org.springframework.security.crypto.password.PasswordEncoder;
import java.math.BigInteger;
import java.security.MessageDigest;

/**
 * 自定义加密类
 */
public class CustomPasswordEncoder implements PasswordEncoder {

    public String encode(CharSequence password) {
        try {
            //给数据进行md5加密
            MessageDigest md = MessageDigest.getInstance("MD5");
            md.update(password.toString().getBytes());
            String pwd = new BigInteger(1, md.digest()).toString(16);
            System.out.println("encode方法：加密前（" + password + "），加密后（" + pwd + "）");
            return pwd;
        } catch (Exception e) {
            return null;
        }
    }

    /**
     * 调用这个方法来判断密码是否匹配
     */
    @Override
    public boolean matches(CharSequence rawPassword, String encodePassword) {
        // 判断密码是否存在
        if (rawPassword == null) {
            return false;
        }

        //通过md5加密后的密码
        String pass = this.encode(rawPassword.toString());

        System.out.println("matches方法：rawPassword：" + rawPassword + "，encodePassword：" + encodePassword + "，pass：" + pass);
        //比较密码是否相等的问题
        return pass.equals(encodePassword);
    }
}
```

注册加密的类：

```yaml
cas:
  adminPagesSecurity:
    ip: 127.0.0.1
  authn:
    #    accept:
    #      users: test::test
    jdbc:
      query:
        - dialect: org.hibernate.dialect.MySQLDialect
          driverClass: com.mysql.cj.jdbc.Driver
          fieldDisabled: disabled
          fieldExpired: expired
          fieldPassword: password
          sql: select * from sys_user where username=?
          url: jdbc:mysql://127.0.0.1:3306/cas?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
          user: test
          password: test
          passwordEncoder:
            type: com.xncoding.cas.CustomPasswordEncoder
            characterEncoding: UTF-8
            encodingAlgorithm: MD5
```

## Rest认证

通过REST数据接口对用户进行认证，通过请求接口，返回固定格式，进行对密码匹配，判断用户是否合法。

什么场景下用rest认证？

用户数据存在远端、不允许cas直接访问数据库、cas不希望你知道帐号数据的表结构

由于Rest方式并不需要直接通过JDBC链接数据库，所以在上一节文章中介绍的JDBC和mysql驱动依赖项就可以删掉了，保留一个就可以了

```xml

<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-rest-authentication</artifactId>
    <version>${cas.version}</version>
</dependency>
```

同样的，把之前的jdbc配置可以删掉了，配置如下即可：

```properties
##
# REST 认证开始, 请求远程调用接口
#
cas.authn.rest.uri=http://localhost:8080/cas_db/user/login
cas.authn.rest.passwordEncoder.type=DEFAULT
cas.authn.rest.passwordEncoder.characterEncoding=UTF-8
cas.authn.rest.passwordEncoder.encodingAlgorithm=MD5
```

REST认证流程是这样的，当用户点击登录后，cas会发送post请求到 http://localhost:8080/cas_db/login
并且把用户信息以"用户名:密码"进行Base64编码放在authorization请求头中。

若输入用户名密码为：admin/123；那么请求头包括：authorization=Basic Base64(admin:MD5(123))。

那么发送后客户端必须响应一下数据，cas明确规定如下：

* cas 服务端会通过post请求，并且把用户信息以"用户名:密码"进行Base64编码放在authorization请求头中
* 200状态码：并且格式为{"@class":"org.apereo.cas.authentication.principal.SimplePrincipal","id":"casuser","attributes":{}
  }是成功的；
* 403状态码：用户不可用；
* 404状态码：账号不存在；
* 423状态码：账户被锁定；
* 428状态码：过期；
* 其他登录失败

接下来编写REST认证服务，使用SpringBoot+Maven+MyBatis构建：

用户类

```java
@TableName(value = "sys_user")
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
    @NotNull
    private String username;
    /**
     * 密码
     */
    @JsonIgnore
    private String password;
    /**
     * 联系电话
     */
    private String email;
    /**
     * 备注
     */
    private String address;
    /**
     * 状态 1:正常 2:禁用
     */
    private Integer age;

    private Integer expired;
    private Integer disabled;
    private Integer locked;

    //需要返回实现org.apereo.cas.authentication.principal.Principal的类名接口
    @TableField(exist=false)
    @JsonProperty("@class")
    private String clazz = "org.apereo.cas.authentication.principal.SimplePrincipal";

    @JsonProperty("attributes")
    @TableField(exist=false)
    private Map<String, Object> attributes = new HashMap<>();
    
    // 省略getter/setter
}
```

接口类：

```java
@RestController
@RequestMapping("/user")
public class SysUserController {
    private Logger logger = LogManager.getLogger(SysUserController.class);

    @Autowired
    private UserService userService;

    @PostMapping("/login")
    public Object login(@RequestHeader HttpHeaders httpHeaders) {
        logger.info("Rest api login.");
        logger.debug("request headers: " + httpHeaders);
        User user = null;
        try {
            UserTemp userTemp = obtainUserFormHeader(httpHeaders);

            //当没有 传递 参数的情况
            if (userTemp == null) {
                return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
            }

            //尝试查找用户库是否存在
            user = userService.findByUsername(userTemp.username);
            if (user != null) {
                if (!user.getPassword().equals(userTemp.password)) {
                    //密码不匹配
                    return new ResponseEntity(HttpStatus.BAD_REQUEST);
                }
                if (user.getDisabled() == 1) {
                    //禁用 403
                    return new ResponseEntity(HttpStatus.FORBIDDEN);
                }
                if (user.getLocked() == 1) {
                    //锁定 423
                    return new ResponseEntity(HttpStatus.LOCKED);
                }
                if (user.getExpired() == 1) {
                    //过期 428
                    return new ResponseEntity(HttpStatus.PRECONDITION_REQUIRED);
                }
            } else {
                //不存在 404
                return new ResponseEntity(HttpStatus.NOT_FOUND);
            }
        } catch (UnsupportedEncodingException e) {
            logger.error("", e);
            return new ResponseEntity(HttpStatus.BAD_REQUEST);
        }
        logger.info("[{" + user.getUsername() + "}] login is ok");
        logger.info(JacksonUtil.bean2Json(user));
        //成功返回json
        return user;
    }


    /**
     * 根据请求头获取用户名及密码
     *
     * @param httpHeaders
     * @return
     * @throws UnsupportedEncodingException
     */
    private UserTemp obtainUserFormHeader(HttpHeaders httpHeaders) throws UnsupportedEncodingException {
        /*
         *
         * This allows the CAS server to reach to a remote REST endpoint via a POST for verification of credentials.
         * Credentials are passed via an Authorization header whose value is Basic XYZ where XYZ is a Base64 encoded version of the credentials.
         */
        //根据官方文档，当请求过来时，会通过把用户信息放在请求头authorization中，并且通过Basic认证方式加密
        String authorization = httpHeaders.getFirst("authorization");//将得到 Basic Base64(用户名:密码)
        if (StringUtils.isEmpty(authorization)) {
            return null;
        }
        String baseCredentials = authorization.split(" ")[1];
        String usernamePassword = new String(Base64Utils.decodeFromString(baseCredentials), StandardCharsets.UTF_8);//用户名:密码
        logger.debug("login user: " + usernamePassword);
        String[] credentials = usernamePassword.split(":");
        return new UserTemp(credentials[0], credentials[1]);
    }


    /**
     * 解析请求过来的用户
     */
    private class UserTemp {
        private String username;
        private String password;

        UserTemp(String username, String password) {
            this.username = username;
            this.password = password;
        }
    }
}
```

Service类我就不贴了，启动后验证登录流程。

## 代码调试

通过官网的overlay构建会发现跟目录有build.cmd/build.sh两个文件，就是在根目录下。
其中有一段代码，不难发现是采用java -jar的方式启用了一个远程调试5000端口，当然了这个端口也是可以改的

```bash
function debug() {
   package && java -Xdebug -Xrunjdwp:transport=dt_socket,address=5000,server=y,suspend=n -jar target/cas.war
}
```

启用调试:

```bash
build debug
```

IDEA启动监听，增加一个Remote Server调试器。填写三个配置：

1. 命令行参数：-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5000
2. Host: localhost
3. Port: 5000

配置好后启动debug，看到如下说明成功一半：

```
Connected to the target VM, address: 'localhost:5000', transport: 'socket'
```

调试代码：

RestAuthenticationHandler进行调试，具体调试哪个代码按自己的实际情况。

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190309-001.jpg)


