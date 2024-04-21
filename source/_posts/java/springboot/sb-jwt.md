---
title: SpringBoot系列 - 集成JWT实现接口权限认证
date: 2017-07-09 20:12:22 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---


一般来讲，对于RESTful API都会有认证(Authentication)和授权(Authorization)过程，保证API的安全性。

Authentication指的是确定这个用户的身份，Authorization是确定该用户拥有什么操作权限。

认证方式一般有三种

*Basic Authentication*

这种方式是直接将用户名和密码放到Header中，使用`Authorization: Basic Zm9vOmJhcg==`，使用最简单但是最不安全。

*TOKEN认证*

这种方式也是再HTTP头中，使用`Authorization: Bearer <token>`，使用最广泛的TOKEN是JWT，通过签名过的TOKEN。

*OAuth2.0*

这种方式安全等级最高，但是也是最复杂的。如果不是大型API平台或者需要给第三方APP使用的，没必要整这么复杂。

一般项目中的RESTful API使用JWT来做认证就足够了。

关于JWT的介绍请参考我的另一篇文章：<https://www.xncoding.com/2017/06/22/security/jwt.html>
<!-- more -->

## SpringBoot集成JWT

简要的说明下我们为什么要用JWT，因为我们要实现完全的前后端分离，所以不可能使用session，cookie的方式进行鉴权，
所以JWT就被派上了用场，可以通过一个加密密钥来进行前后端的鉴权。

程序逻辑:

1. 我们POST用户名与密码到/login进行登入，如果成功返回一个加密token，失败的话直接返回401错误。
1. 之后用户访问每一个需要权限的网址请求必须在header中添加Authorization字段，例如Authorization: token，token为密钥。
1. 后台会进行token的校验，如果不通过直接返回401。

这里我讲一下如何在SpringBoot中使用JWT来做接口权限认证，安全框架依旧使用Shiro，JWT的实现使用 [jjwt](https://github.com/jwtk/jjwt)

### 添加Maven依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
<dependency>
<groupId>com.auth0</groupId>
<artifactId>java-jwt</artifactId>
<version>3.4.0</version>
</dependency>
        <!-- shiro 权限控制 -->
<dependency>
<groupId>org.apache.shiro</groupId>
<artifactId>shiro-spring</artifactId>
<version>1.4.0</version>
<exclusions>
    <exclusion>
        <artifactId>slf4j-api</artifactId>
        <groupId>org.slf4j</groupId>
    </exclusion>
</exclusions>
</dependency>
        <!-- shiro ehcache (shiro缓存)-->
<dependency>
<groupId>org.apache.shiro</groupId>
<artifactId>shiro-ehcache</artifactId>
<version>1.4.0</version>
<exclusions>
    <exclusion>
        <artifactId>slf4j-api</artifactId>
        <groupId>org.slf4j</groupId>
    </exclusion>
</exclusions>
</dependency>
```

### 创建用户Service

这个在shiro一节讲过如果创建角色权限表，添加用户Service来执行查找用户操作，这里就不多讲具体实现了，只列出关键代码：

```java
/**
 * 通过名称查找用户
 *
 * @param username
 * @return
 */
public ManagerInfo findByUsername(String username) {
    ManagerInfo managerInfo = managerInfoDao.findByUsername(username);
    if (managerInfo == null) {
        throw new UnknownAccountException();
    }
    return managerInfo;
}
```

用户信息类：

```java
public class ManagerInfo implements Serializable {
    private static final long serialVersionUID = 1L;
    /**
     * 主键ID
     */
    private Integer id;
    /**
     * 账号
     */
    private String username;
    /**
     * 密码
     */
    private String password;
    /**
     * md5密码盐
     */
    private String salt;
    /**
     * 一个管理员具有多个角色
     */
    private List<SysRole> roles;
```

### JWT工具类

我们写一个简单的JWT加密，校验工具，并且使用用户自己的密码充当加密密钥，
这样保证了token 即使被他人截获也无法破解。并且我们在token中附带了username信息，并且设置密钥5分钟就会过期。

```java
public class JWTUtil {

    private static final Logger log = LoggerFactory.getLogger(JWTUtil.class);

    // 过期时间5分钟
    private static final long EXPIRE_TIME = 5 * 60 * 1000;

    /**
     * 生成签名,5min后过期
     *
     * @param username 用户名
     * @param secret   用户的密码
     * @return 加密的token
     */
    public static String sign(String username, String secret) {
        Date date = new Date(System.currentTimeMillis() + EXPIRE_TIME);
        Algorithm algorithm = Algorithm.HMAC256(secret);
        // 附带username信息
        return JWT.create()
                .withClaim("username", username)
                .withExpiresAt(date)
                .sign(algorithm);
    }

    /**
     * 校验token是否正确
     *
     * @param token  密钥
     * @param secret 用户的密码
     * @return 是否正确
     */
    public static boolean verify(String token, String username, String secret) {
        Algorithm algorithm = Algorithm.HMAC256(secret);
        JWTVerifier verifier = JWT.require(algorithm)
                .withClaim("username", username)
                .build();
        DecodedJWT jwt = verifier.verify(token);
        return true;
    }

    /**
     * 获得token中的信息无需secret解密也能获得
     *
     * @return token中包含的用户名
     */
    public static String getUsername(String token) {
        DecodedJWT jwt = JWT.decode(token);
        return jwt.getClaim("username").asString();
    }

}
```

### 编写登录接口

为了让用户登录的时候获取到正确的JWT Token，需要实现登录接口，这里我编写一个`LoginController.java`：

```java
/**
 * 登录接口类
 */
@RestController
public class LoginController {

    @Resource
    private ManagerInfoService managerInfoService;

    private static final Logger _logger = LoggerFactory.getLogger(LoginController.class);

    @PostMapping("/login")
    public BaseResponse<String> login(@RequestHeader(name="Content-Type", defaultValue = "application/json") String contentType,
                                      @RequestBody LoginParam loginParam) {
        _logger.info("用户请求登录获取Token");
        String username = loginParam.getUsername();
        String password = loginParam.getPassword();
        ManagerInfo user = managerInfoService.findByUsername(username);
        //随机数盐
        String salt = user.getSalt();
        //原密码加密（通过username + salt作为盐）
        String encodedPassword = ShiroKit.md5(password, username + salt);
        if (user.getPassword().equals(encodedPassword)) {
            return new BaseResponse<>(true, "Login success", JWTUtil.sign(username, encodedPassword));
        } else {
            throw new UnauthorizedException();
        }
    }

}
```

注意上面登录的时候，我会从数据库中把这个用户取出来，密码加盐算MD5值比较，通过之后再用密码作为密钥来签名生成JWT。

### 编写RESTful接口

先编写一个通用的接口返回类：

```java
/**
 * API接口的基础返回类
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2018/1/7
 */
public class BaseResponse<T> {
    /**
     * 是否成功
     */
    private boolean success;

    /**
     * 说明
     */
    private String msg;

    /**
     * 返回数据
     */
    private T data;

    public BaseResponse() {

    }

    public BaseResponse(boolean success, String msg, T data) {
        this.success = success;
        this.msg = msg;
        this.data = data;
    }
}
```

通过SpringMVC实现RESTful接口，这里我只写一个示例方法：

```java
/**
 * 机具管理API接口类
 */
@RestController
@RequestMapping(value = "/api/v1")
public class PublicController {

    private static final Logger _logger = LoggerFactory.getLogger(PublicController.class);

    /**
     * 入网绑定查询接口
     *
     * @return 是否入网
     */
    @RequestMapping(value = "/join", method = RequestMethod.GET)
    @RequiresAuthentication
    public BaseResponse join(@RequestParam("imei") String imei) {
        _logger.info("入网查询接口 start... imei=" + imei);
        BaseResponse result = new BaseResponse();
        result.setSuccess(true);
        result.setMsg("已入网并绑定了网点");
        return result;
    }
}
```

### 自定义异常

为了实现我自己能够手动抛出异常，我自己写了一个`UnauthorizedException.java`

```java
public class UnauthorizedException extends RuntimeException {
    public UnauthorizedException(String msg) {
        super(msg);
    }

    public UnauthorizedException() {
        super();
    }
}
```

### 处理框架异常

之前说过restful要统一返回的格式，所以我们也要全局处理Spring Boot的抛出异常。利用@RestControllerAdvice能很好的实现。
注意这个统一异常处理器只对认证过的用户调用接口中的异常有作用，对AuthenticationException没有用

```java
@RestControllerAdvice
public class ExceptionController {

    // 捕捉shiro的异常
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    @ExceptionHandler(ShiroException.class)
    public BaseResponse handle401(ShiroException e) {
        return new BaseResponse(false, "shiro的异常", null);
    }
    
    // 捕捉UnauthorizedException
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    @ExceptionHandler(UnauthorizedException.class)
    public BaseResponse handle401() {
        return new BaseResponse(false, "UnauthorizedException", null);
    }

    // 捕捉其他所有异常
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public BaseResponse globalException(HttpServletRequest request, Throwable ex) {
        return new BaseResponse(false, "其他异常", null);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
        if (statusCode == null) {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
        return HttpStatus.valueOf(statusCode);
    }
}
```

### 配置Shiro

大家可以先看下官方的 [Spring-Shiro](http://shiro.apache.org/spring.html) 整合教程，有个初步的了解。
不过既然我们用了SpringBoot，那我们肯定要争取零配置文件。

*实现JWTToken*

JWTToken差不多就是Shiro用户名密码的载体。因为我们是前后端分离，服务器无需保存用户状态，所以不需要RememberMe这类功能，
我们简单的实现下AuthenticationToken接口即可。因为token自己已经包含了用户名等信息，所以这里我就弄了一个字段。
如果你喜欢钻研，可以看看官方的UsernamePasswordToken是如何实现的。

```java
public class JWTToken implements AuthenticationToken {

    // 密钥
    private String token;

    public JWTToken(String token) {
        this.token = token;
    }

    @Override
    public Object getPrincipal() {
        return token;
    }

    @Override
    public Object getCredentials() {
        return token;
    }
}
```

*实现Realm*

realm的用于处理用户是否合法的这一块，需要我们自己实现。

```java
/**
 * Description  : 身份校验核心类
 */

public class MyShiroRealm extends AuthorizingRealm {

    private static final Logger _logger = LoggerFactory.getLogger(MyShiroRealm.class);

    @Autowired
    ManagerInfoService managerInfoService;

    /**
     * JWT签名密钥，这里没用。我使用的是用户的MD5密码作为签名密钥
     */
    public static final String SECRET = "9281e268b77b7c439a20b46fd1483b9a";

    /**
     * 必须重写此方法，不然Shiro会报错
     */
    @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof JWTToken;
    }

    /**
     * 认证信息(身份验证)
     * Authentication 是用来验证用户身份
     *
     * @param auth
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken auth)
            throws AuthenticationException {
        _logger.info("MyShiroRealm.doGetAuthenticationInfo()");

        String token = (String) auth.getCredentials();
        // 解密获得username，用于和数据库进行对比
        String username = JWTUtil.getUsername(token);
        if (username == null) {
            throw new AuthenticationException("token invalid");
        }

        //通过username从数据库中查找 ManagerInfo对象
        //实际项目中，这里可以根据实际情况做缓存，如果不做，Shiro自己也是有时间间隔机制，2分钟内不会重复执行该方法
        ManagerInfo managerInfo = managerInfoService.findByUsername(username);

        if (managerInfo == null) {
            throw new AuthenticationException("User didn't existed!");
        }

        if (!JWTUtil.verify(token, username, managerInfo.getPassword())) {
            throw new AuthenticationException("Username or password error");
        }

        return new SimpleAuthenticationInfo(token, token, "my_realm");
    }

    /**
     * 此方法调用hasRole,hasPermission的时候才会进行回调.
     * <p>
     * 权限信息.(授权):
     * 1、如果用户正常退出，缓存自动清空；
     * 2、如果用户非正常退出，缓存自动清空；
     * 3、如果我们修改了用户的权限，而用户不退出系统，修改的权限无法立即生效。
     * （需要手动编程进行实现；放在service进行调用）
     * 在权限修改后调用realm中的方法，realm已经由spring管理，所以从spring中获取realm实例，调用clearCached方法；
     * :Authorization 是授权访问控制，用于对用户进行的操作授权，证明该用户是否允许进行当前操作，如访问某个链接，某个资源文件等。
     *
     * @param principals
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        /*
         * 当没有使用缓存的时候，不断刷新页面的话，这个代码会不断执行，
         * 当其实没有必要每次都重新设置权限信息，所以我们需要放到缓存中进行管理；
         * 当放到缓存中时，这样的话，doGetAuthorizationInfo就只会执行一次了，
         * 缓存过期之后会再次执行。
         */
        _logger.info("权限配置-->MyShiroRealm.doGetAuthorizationInfo()");
        String username = JWTUtil.getUsername(principals.toString());

        // 下面的可以使用缓存提升速度
        ManagerInfo managerInfo = managerInfoService.findByUsername(username);

        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();

        //设置相应角色的权限信息
        for (SysRole role : managerInfo.getRoles()) {
            //设置角色
            authorizationInfo.addRole(role.getRole());
            for (Permission p : role.getPermissions()) {
                //设置权限
                authorizationInfo.addStringPermission(p.getPermission());
            }
        }
        return authorizationInfo;
    }

}
```

在`doGetAuthenticationInfo`中用户可以自定义抛出很多异常，详情见文档。

*重写Filter*

所有的请求都会先经过Filter，所以我们继承官方的`BasicHttpAuthenticationFilter`，并且重写鉴权的方法，
另外通过重写preHandle，实现跨越访问。

代码的执行流程`preHandle->isAccessAllowed->isLoginAttempt->executeLogin`

```java
public class JWTFilter extends BasicHttpAuthenticationFilter {

    private Logger LOGGER = LoggerFactory.getLogger(this.getClass());

    /**
     * 判断用户是否想要登入。
     * 检测header里面是否包含Authorization字段即可
     */
    @Override
    protected boolean isLoginAttempt(ServletRequest request, ServletResponse response) {
        HttpServletRequest req = (HttpServletRequest) request;
        String authorization = req.getHeader("Authorization");
        return authorization != null;
    }

    /**
     *
     */
    @Override
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        String authorization = httpServletRequest.getHeader("Authorization");
        JWTToken token = new JWTToken(authorization);
        // 提交给realm进行登入，如果错误他会抛出异常并被捕获
        getSubject(request, response).login(token);
        // 如果没有抛出异常则代表登入成功，返回true
        return true;
    }

    /**
     * 这里我们详细说明下为什么最终返回的都是true，即允许访问
     * 例如我们提供一个地址 GET /article
     * 登入用户和游客看到的内容是不同的
     * 如果在这里返回了false，请求会被直接拦截，用户看不到任何东西
     * 所以我们在这里返回true，Controller中可以通过 subject.isAuthenticated() 来判断用户是否登入
     * 如果有些资源只有登入用户才能访问，我们只需要在方法上面加上 @RequiresAuthentication 注解即可
     * 但是这样做有一个缺点，就是不能够对GET,POST等请求进行分别过滤鉴权(因为我们重写了官方的方法)，但实际上对应用影响不大
     */
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        if (isLoginAttempt(request, response)) {
            try {
                executeLogin(request, response);
            } catch (Exception e) {
                response401(request, response);
            }
        }
        return true;
    }

    /**
     * 对跨域提供支持
     */
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        httpServletResponse.setHeader("Access-control-Allow-Origin", httpServletRequest.getHeader("Origin"));
        httpServletResponse.setHeader("Access-Control-Allow-Methods", "GET,POST,OPTIONS,PUT,DELETE");
        httpServletResponse.setHeader("Access-Control-Allow-Headers", httpServletRequest.getHeader("Access-Control-Request-Headers"));
        // 跨域时会首先发送一个option请求，这里我们给option请求直接返回正常状态
        if (httpServletRequest.getMethod().equals(RequestMethod.OPTIONS.name())) {
            httpServletResponse.setStatus(HttpStatus.OK.value());
            return false;
        }
        return super.preHandle(request, response);
    }

    /**
     * 将非法请求跳转到 /401
     */
    private void response401(ServletRequest req, ServletResponse resp) {
        try {
            HttpServletResponse httpServletResponse = (HttpServletResponse) resp;
            httpServletResponse.sendRedirect("/401");
        } catch (IOException e) {
            LOGGER.error(e.getMessage());
        }
    }
}
```

*编写ShiroConfig配置类*

这里我还增加了EhCache缓存管理支持，不需要每次都调用数据库做授权。

```java
@Configuration
@Order(1)
public class ShiroConfig {
    /**
     * ShiroFilterFactoryBean 处理拦截资源文件问题。
     * 注意：单独一个ShiroFilterFactoryBean配置是或报错的，以为在
     * 初始化ShiroFilterFactoryBean的时候需要注入：SecurityManager Filter Chain定义说明
     * 1、一个URL可以配置多个Filter，使用逗号分隔
     * 2、当设置多个过滤器时，全部验证通过，才视为通过
     * 3、部分过滤器可指定参数，如perms，roles
     */
    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {

        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        // 必须设置 SecurityManager
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //验证码过滤器
        Map<String, Filter> filtersMap = shiroFilterFactoryBean.getFilters();
        filtersMap.put("jwt", new JWTFilter());
        shiroFilterFactoryBean.setFilters(filtersMap);

        // 拦截器
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();

        // 其他的
        filterChainDefinitionMap.put("/**", "jwt");

        // 访问401和404页面不通过我们的Filter
        filterChainDefinitionMap.put("/401", "anon");
        filterChainDefinitionMap.put("/404", "anon");

        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }

    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 设置realm.
        securityManager.setRealm(myShiroRealm());
        //注入缓存管理器
        securityManager.setCacheManager(ehCacheManager());
        // 关闭shiro自带的session
        DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
        DefaultSessionStorageEvaluator defaultSessionStorageEvaluator = new DefaultSessionStorageEvaluator();
        defaultSessionStorageEvaluator.setSessionStorageEnabled(false);
        subjectDAO.setSessionStorageEvaluator(defaultSessionStorageEvaluator);
        securityManager.setSubjectDAO(subjectDAO);

        return securityManager;
    }

    /**
     * 身份认证realm; (这个需要自己写，账号密码校验；权限等)
     */
    @Bean
    public MyShiroRealm myShiroRealm() {
        MyShiroRealm myShiroRealm = new MyShiroRealm();
        return myShiroRealm;
    }

    /**
     * 开启shiro aop注解支持. 使用代理方式; 所以需要开启代码支持;
     *
     * @param securityManager 安全管理器
     * @return 授权Advisor
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }

    /**
     * shiro缓存管理器;
     * 需要注入对应的其它的实体类中：
     * 1、安全管理器：securityManager
     * 可见securityManager是整个shiro的核心；
     *
     * @return
     */
    @Bean
    public EhCacheManager ehCacheManager() {
        EhCacheManager cacheManager = new EhCacheManager();
        cacheManager.setCacheManagerConfigFile("classpath:ehcache.xml");
        return cacheManager;
    }

}
```

里面URL规则自己参考文档 <http://shiro.apache.org/web.html> ，这个在shiro那篇说的很清楚了。

## 运行验证

最后是将代码跑起来验证这一切是否正常。

启动SpringBoot后，先通过POST请求登录拿到token

![](https://xnstatic-1253397658.file.myqcloud.com/jwt21.png)

然后在调用入网接口的时候在header中带上这个token认证：

![](https://xnstatic-1253397658.file.myqcloud.com/jwt22.png)

如果token认证不正确会报异常：

![](https://xnstatic-1253397658.file.myqcloud.com/jwt23.png)

如果使用普通用户登录，认证正确但是授权访问接口失败，会返回如下的未授权结果：

![](https://xnstatic-1253397658.file.myqcloud.com/jwt24.png)

## 参考文章

* [RESTful API Authentication Basics](http://blog.restcase.com/restful-api-authentication-basics/)
* [Shiro+JWT+Spring Boot Restful简易教程](https://juejin.im/post/59f1b2766fb9a0450e755993)

## GitHub源码

[springboot-jwt](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-jwt)

