---
title: SpringBoot系列 - 集成Shiro权限管理
date: 2017-07-07 09:56:08 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

`Apache Shiro`是Java的一个安全框架。目前，使用`Apache Shiro`的人越来越多，相比`Spring Security`而言相当简单，
可能没有`Spring Security`做的功能强大，但是在实际工作时可能并不需要那么复杂的东西，
所以使用小而简单的Shiro就足够了。对于它俩到底哪个好，这个不必纠结，能更简单的解决项目问题就好了。

本教程只介绍基本的Shiro使用，不会过多分析源码等，重在使用。
<!-- more -->

## Shiro架构

Shiro可以非常容易的开发出足够好的应用，其不仅可以用在JavaSE环境，也可以用在JavaEE环境。
Shiro可以帮助我们完成：认证、授权、加密、会话管理、与Web集成、缓存等。这不就是我们想要的嘛，
而且Shiro的API也是非常简单；其基本功能点如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro01.png)

* Authentication：身份认证/登录，验证用户是不是拥有相应的身份；
* Authorization：授权，即权限验证，验证某个已认证的用户是否拥有某个权限；即判断用户是否能做事情，常见的如：验证某个用户是否拥有某个角色。或者细粒度的验证某个用户对某个资源是否具有某个权限；
* Session Manager：会话管理，即用户登录后就是一次会话，在没有退出之前，它的所有信息都在会话中；会话可以是普通JavaSE环境的，也可以是如Web环境的；
* Cryptography：加密，保护数据的安全性，如密码加密存储到数据库，而不是明文存储；
* Web Support：Web支持，可以非常容易的集成到Web环境；
* Caching：缓存，比如用户登录后，其用户信息、拥有的角色/权限不必每次去查，这样可以提高效率；
* Concurrency：shiro支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去；
* Testing：提供测试支持；
* Run As：允许一个用户假装为另一个用户（如果他们允许）的身份进行访问；
* Remember Me：记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登录了。

记住一点，Shiro不会去维护用户、维护权限；这些需要我们自己去设计/提供；然后通过相应的接口注入给Shiro即可。

接下来我们分别从外部和内部来看看Shiro的架构，对于一个好的框架，从外部来看应该具有非常简单易于使用的API，
且API契约明确；从内部来看的话，其应该有一个可扩展的架构，即非常容易插入用户自定义实现，因为任何框架都不能满足所有需求。

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro02.png)

可以看到：应用代码直接交互的对象是Subject，也就是说Shiro的对外API核心就是Subject。

* Subject：主体，代表了当前"用户"
  ，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是Subject，如网络爬虫，机器人等；即一个抽象概念；所有Subject都绑定到SecurityManager，与Subject的所有交互都会委托给SecurityManager；可以把Subject认为是一个门面；SecurityManager才是实际的执行者；
*

SecurityManager：安全管理器；即所有与安全有关的操作都会与SecurityManager交互；且它管理着所有Subject；可以看出它是Shiro的核心，它负责与后边介绍的其他组件进行交互，如果学习过SpringMVC，你可以把它看成DispatcherServlet前端控制器；

*

Realm：域，Shiro从从Realm获取安全数据（如用户、角色、权限），就是说SecurityManager要验证用户身份，那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法；也需要从Realm得到用户相应的角色/权限进行验证用户是否能进行操作；可以把Realm看成DataSource，即安全数据源。

也就是说对于我们而言，最简单的一个Shiro应用：

1. 应用代码通过Subject来进行认证和授权，而Subject又委托给SecurityManager；
2. 我们需要给Shiro的SecurityManager注入Realm，从而让SecurityManager能得到合法的用户及其权限进行判断。

从以上也可以看出，Shiro不提供维护用户/权限，而是通过Realm让开发人员自己注入。

接下来我们来从Shiro内部来看下Shiro的架构，如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro03.png)

* Subject：主体，可以看到主体可以是任何可以与应用交互的"用户"；
*

SecurityManager：相当于SpringMVC中的DispatcherServlet或者Struts2中的FilterDispatcher；是Shiro的心脏；所有具体的交互都通过SecurityManager进行控制；它管理着所有Subject、且负责进行认证和授权、及会话、缓存的管理。

* Authenticator：认证器，负责主体认证的，这是一个扩展点，如果用户觉得Shiro默认的不好，可以自定义实现；其需要认证策略（Authentication
  Strategy），即什么情况下算用户认证通过了；
* Authorizer：授权器，或者访问控制器，用来决定主体是否有权限进行相应的操作；即控制着用户能访问应用中的哪些功能；
*

Realm：可以有1个或多个Realm，可以认为是安全实体数据源，即用于获取安全实体的；可以是JDBC实现，也可以是LDAP实现，或者内存实现等等；由用户提供；注意：Shiro不知道你的用户/权限存储在哪及以何种格式存储；所以我们一般在应用中都需要实现自己的Realm；

*

SessionManager：如果写过Servlet就应该知道Session的概念，Session呢需要有人去管理它的生命周期，这个组件就是SessionManager；而Shiro并不仅仅可以用在Web环境，也可以用在如普通的JavaSE环境、EJB等环境；所有呢，Shiro就抽象了一个自己的Session来管理主体与应用之间交互的数据；这样的话，比如我们在Web环境用，刚开始是一台Web服务器；接着又上了台EJB服务器；这时想把两台服务器的会话数据放到一个地方，这个时候就可以实现自己的分布式会话（如把数据放到Memcached服务器）；

*

SessionDAO：DAO大家都用过，数据访问对象，用于会话的CRUD，比如我们想把Session保存到数据库，那么可以实现自己的SessionDAO，通过如JDBC写到数据库；比如想把Session放到Memcached中，可以实现自己的Memcached
SessionDAO；另外SessionDAO中可以使用Cache进行缓存，以提高性能；

* CacheManager：缓存控制器，来管理如用户、角色、权限等的缓存的；因为这些数据基本上很少去改变，放到缓存中后可以提高访问的性能
* Cryptography：密码模块，Shiro提高了一些常见的加密组件用于如密码加密/解密的。

参考 [Shiro官网](http://shiro.apache.org/reference.html)

## SpringBoot集成

实现一个最长见的验证码登录、记住我、权限自定义（or），缓存功能的SpringBoot应用，模板使用Thymeleaf 3

Maven依赖：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
        <!-- thymeleaf模板中shiro标签-->
<dependency>
<groupId>com.github.theborakompanioni</groupId>
<artifactId>thymeleaf-extras-shiro</artifactId>
<version>2.0.0</version>
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
        <!--验证码框架-->
<dependency>
<groupId>com.github.axet</groupId>
<artifactId>kaptcha</artifactId>
<version>0.0.9</version>
</dependency>
```

新建一个配置类`ShiroConfig.java`，内容如下：

```java
/**
 * Description  : Apache Shiro 核心通过 Filter 来实现，就好像SpringMvc 通过DispachServlet 来主控制一样。
 * 既然是使用 Filter 一般也就能猜到，是通过URL规则来进行过滤和权限校验，所以我们需要定义一系列关于URL的规则和访问权限。
 */
@Configuration
@Order(1)
public class ShiroConfig {

    //配置kaptcha图片验证码框架提供的Servlet,,这是个坑,很多人忘记注册(注意)
    @Bean
    public ServletRegistrationBean kaptchaServlet() {
        ServletRegistrationBean servlet = new ServletRegistrationBean(new KaptchaServlet(), "/kaptcha.jpg");
        servlet.addInitParameter(Constants.KAPTCHA_SESSION_CONFIG_KEY, Constants.KAPTCHA_SESSION_KEY);//session key
        servlet.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_SIZE, "50");//字体大小
        servlet.addInitParameter(Constants.KAPTCHA_BORDER, "no");
        servlet.addInitParameter(Constants.KAPTCHA_BORDER_COLOR, "105,179,90");
        servlet.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_SIZE, "45");
        servlet.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_CHAR_LENGTH, "4");
        servlet.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_NAMES, "宋体,楷体,微软雅黑");
        servlet.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_COLOR, "blue");
        servlet.addInitParameter(Constants.KAPTCHA_IMAGE_WIDTH, "125");
        servlet.addInitParameter(Constants.KAPTCHA_IMAGE_HEIGHT, "60");
        //可以设置很多属性,具体看com.google.code.kaptcha.Constants
        return servlet;
    }

    //注入异常处理类
    @Bean
    public MyExceptionResolver myExceptionResolver() {
        return new MyExceptionResolver();
    }

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
        KaptchaFilter kaptchaFilter = new KaptchaFilter();
        filtersMap.put("kaptchaFilter", kaptchaFilter);
        //实现自己规则roles,这是为了实现or的效果
        //RoleFilter roleFilter = new RoleFilter();
        //filtersMap.put("roles", roleFilter);
        shiroFilterFactoryBean.setFilters(filtersMap);
        // 拦截器
        //rest：比如/admins/user/**=rest[user],根据请求的方法，相当于/admins/user/**=perms[user：method] ,其中method为post，get，delete等。
        //port：比如/admins/user/**=port[8081],当请求的url的端口不是8081是跳转到schemal：//serverName：8081?queryString,其中schmal是协议http或https等，serverName是你访问的host,8081是url配置里port的端口，queryString是你访问的url里的？后面的参数。
        //perms：比如/admins/user/**=perms[user：add：*],perms参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，比如/admins/user/**=perms["user：add：*,user：modify：*"]，当有多个参数时必须每个参数都通过才通过，想当于isPermitedAll()方法。
        //roles：比如/admins/user/**=roles[admin],参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，当有多个参数时，比如/admins/user/**=roles["admin,guest"],每个参数通过才算通过，相当于hasAllRoles()方法。//要实现or的效果看http://zgzty.blog.163.com/blog/static/83831226201302983358670/
        //anon：比如/admins/**=anon 没有参数，表示可以匿名使用。
        //authc：比如/admins/user/**=authc表示需要认证才能使用，没有参数
        //authcBasic：比如/admins/user/**=authcBasic没有参数表示httpBasic认证
        //ssl：比如/admins/user/**=ssl没有参数，表示安全的url请求，协议为https
        //user：比如/admins/user/**=user没有参数表示必须存在用户，当登入操作时不做检查
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
        // 配置退出过滤器,其中的具体的退出代码Shiro已经替我们实现了
        filterChainDefinitionMap.put("/logout", "logout");
        //配置记住我或认证通过可以访问的地址
        filterChainDefinitionMap.put("/index", "user");
        filterChainDefinitionMap.put("/", "user");
        filterChainDefinitionMap.put("/login", "kaptchaFilter");
        // <!-- 过滤链定义，从上向下顺序执行，一般将 /**放在最为下边 -->:这是一个坑呢，一不小心代码就不好使了;
        //这段是配合 actuator框架使用的，配置相应的角色才能访问
        // filterChainDefinitionMap.put("/health", "roles[aix]");//服务器健康状况页面
        // filterChainDefinitionMap.put("/info", "roles[aix]");//服务器信息页面
        // filterChainDefinitionMap.put("/env", "roles[aix]");//应用程序的环境变量
        // filterChainDefinitionMap.put("/metrics", "roles[aix]");
        // filterChainDefinitionMap.put("/configprops", "roles[aix]");
        //开放的静态资源
        filterChainDefinitionMap.put("/favicon.ico", "anon");//网站图标
        filterChainDefinitionMap.put("/static/**", "anon");//配置static文件下资源能被访问的，这是个例子
        filterChainDefinitionMap.put("/kaptcha.jpg", "anon");//图片验证码(kaptcha框架)

        filterChainDefinitionMap.put("/api/v1/**", "anon");//API接口

        // swagger接口文档
        filterChainDefinitionMap.put("/v2/api-docs", "anon");
        filterChainDefinitionMap.put("/webjars/**", "anon");
        filterChainDefinitionMap.put("/swagger-resources/**", "anon");
        filterChainDefinitionMap.put("/swagger-ui.html", "anon");

        // 其他的
        filterChainDefinitionMap.put("/**", "authc");

        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        // 未授权界面，不生效(详情原因看MyExceptionResolver)
        shiroFilterFactoryBean.setUnauthorizedUrl("/errorView/403_error.html");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }

    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 设置realm.
        securityManager.setRealm(myShiroRealm());
        //注入缓存管理器
        securityManager.setCacheManager(ehCacheManager());//这个如果执行多次，也是同样的一个对象;
        //注入记住我管理器;
        securityManager.setRememberMeManager(rememberMeManager());
        return securityManager;
    }

    /**
     * 身份认证realm; (这个需要自己写，账号密码校验；权限等)
     */
    @Bean
    public MyShiroRealm myShiroRealm() {
        MyShiroRealm myShiroRealm = new MyShiroRealm();
        myShiroRealm.setCredentialsMatcher(hashedCredentialsMatcher());
        return myShiroRealm;
    }

    /**
     * 凭证匹配器 （由于我们的密码校验交给Shiro的SimpleAuthenticationInfo进行处理了
     * 所以我们需要修改下doGetAuthenticationInfo中的代码; @return
     */
    @Bean
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
        hashedCredentialsMatcher.setHashAlgorithmName("md5");// 散列算法:这里使用MD5算法;
        hashedCredentialsMatcher.setHashIterations(2);// 散列的次数，比如散列两次，相当于md5(md5(""));
        hashedCredentialsMatcher.setStoredCredentialsHexEncoded(true);//表示是否存储散列后的密码为16进制，需要和生成密码时的一样，默认是base64；
        return hashedCredentialsMatcher;
    }

    /**
     * 开启shiro aop注解支持. 使用代理方式; 所以需要开启代码支持;
     *
     * @param securityManager
     * @return
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

    /**
     * cookie对象;
     *
     * @return
     */
    @Bean
    public SimpleCookie rememberMeCookie() {
        //System.out.println("ShiroConfiguration.rememberMeCookie()");
        //这个参数是cookie的名称，对应前端的checkbox的name = rememberMe
        SimpleCookie simpleCookie = new SimpleCookie("rememberMe");
        //<!-- 记住我cookie生效时间30天 ,单位秒;-->
        simpleCookie.setMaxAge(259200);
        return simpleCookie;
    }

    /**
     * cookie管理对象;
     *
     * @return
     */
    @Bean
    public CookieRememberMeManager rememberMeManager() {
        //System.out.println("ShiroConfiguration.rememberMeManager()");
        CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
        cookieRememberMeManager.setCookie(rememberMeCookie());
        return cookieRememberMeManager;
    }

    @Bean(name = "shiroDialect")
    public ShiroDialect shiroDialect() {
        return new ShiroDialect();
    }
}
```

代码中解释都非常清楚。

`MyShiroRealm.java`的内容如下：

```java
public class MyShiroRealm extends AuthorizingRealm {

    private static final Logger _logger = LoggerFactory.getLogger(MyShiroRealm.class);

    @Autowired
    ManagerInfoService managerInfoService;

    /**
     * 认证信息.(身份验证)
     * Authentication 是用来验证用户身份
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
            throws AuthenticationException {

        _logger.info("MyShiroRealm.doGetAuthenticationInfo()");

        //获取用户的输入的账号.
        String username = (String) token.getPrincipal();
        //_logger.info("用户的账号:"+username);

        //通过username从数据库中查找 ManagerInfo对象
        //实际项目中，这里可以根据实际情况做缓存，如果不做，Shiro自己也是有时间间隔机制，2分钟内不会重复执行该方法
        ManagerInfo managerInfo = managerInfoService.findByUsername(username);

        if (managerInfo == null) {
            return null;
        }

        //交给AuthenticatingRealm使用CredentialsMatcher进行密码匹配，如果觉得人家的不好可以自定义实现
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
                managerInfo, //用户
                managerInfo.getPassword(), //密码
                ByteSource.Util.bytes(managerInfo.getCredentialsSalt()),//salt=username+salt
                getName()  //realm name
        );

        //明文: 若存在，将此用户存放到登录认证info中，无需自己做密码对比，Shiro会为我们进行密码对比校验
//        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
//                managerInfo, //用户名
//                managerInfo.getPassword(), //密码
//                getName()  //realm name
//        );
        return authenticationInfo;
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
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        ManagerInfo managerInfo = (ManagerInfo) principals.getPrimaryPrincipal();

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

    /**
     * 设置认证加密方式
     */
    @Override
    public void setCredentialsMatcher(CredentialsMatcher credentialsMatcher) {
        HashedCredentialsMatcher md5CredentialsMatcher = new HashedCredentialsMatcher();
        md5CredentialsMatcher.setHashAlgorithmName(ShiroKit.HASH_ALGORITHM_NAME);
        md5CredentialsMatcher.setHashIterations(ShiroKit.HASH_ITERATIONS);
        super.setCredentialsMatcher(md5CredentialsMatcher);
    }
}

```

自定义异常处理类`MyExceptionResolver.java` :

```java
public class MyExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        //如果是shiro无权操作，因为shiro 在操作auno等一部分不进行转发至无权限url
        if (ex instanceof UnauthorizedException) {
            return new ModelAndView("error/shiro_403");
        }
        return null;
    }
}
```

## 接口控制

配置好了Shiro后就可以通过注解方式来限制某些接口调用需要相应的角色或权限了：

```java
@RequestMapping(value = "/index")
@RequiresRoles("admin")
public String index(HttpServletRequest request, Model model) {
    _logger.info("进入项目管理首页...");
}
```

其他的注解请参考官网的 [Shiro注解](https://shiro.apache.org/java-annotations-list.html)

## Thymeleaf的shiro标签

可以在Thymeleaf模板中使用shiro的权限标签来控制某些菜单或按钮是否显示。

maven中添加依赖，这个前面已经有了：

```xml
<!-- thymeleaf模板中shiro标签-->
<dependency>
    <groupId>com.github.theborakompanioni</groupId>
    <artifactId>thymeleaf-extras-shiro</artifactId>
    <version>2.0.0</version>
</dependency>
```

在ShiroConfig中添加一个Bean配置：

```java
@Bean(name = "shiroDialect")
public ShiroDialect shiroDialect() {
    return new ShiroDialect();
}
```

在html页面添加如下内容：

```html

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org"
      xmlns:shiro="http://www.pollix.at/thymeleaf/shiro">
```

添加完后在html页面调用如下：

```html
<!-- 认证通过或已记住的用户。 -->
<p shiro:user="">
    Welcome back John! Not John? Click <a href="login.html">here</a> to login.
</p>
<p shiro:notAuthenticated="">
    未身份验证（包括记住我）
</p>
<p shiro:guest=""><span style="white-space:pre;"> </span>Please <a href="login.html">login5555</a></p>  
```

第二种：

```html

<shiro:guest>
    <a>登录</a> <a>注册</a>
</shiro:guest>
<shiro:user>
    欢迎
    <shiro:principal property="name"/>
</shiro:user>
```

## 权限数据库设计

一般来讲都会讲用户的角色和权限保存到数据库，这里设计一种最通用的模型，
使用RBAC（Role-Based Access Control，基于角色的访问控制）模型设计用户，角色和权限间的关系。简单地说，
一个用户拥有若干角色，每一个角色拥有若干权限。这样，就构造成"用户-角色-权限"的授权模型。

在这种模型中，用户与角色之间，角色与权限之间，一般者是多对多的关系。如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro04.png)

根据这个模型，设计数据库表，并插入一些测试数据，具体可参考源码中的`schema.sql`，后面我将`t_user`表名字改成`t_manager`

然后我们通过MyBatis实现`ManagerInfoService`，

```java
/**
 * 后台用户管理
 */
@Service
public class ManagerInfoService {
    @Resource
    private ManagerInfoDao managerInfoDao;

    public ManagerInfo findByUsername(String username) {
        return managerInfoDao.findByUsername(username);
    }
}
```

然后对应的`ManagerInfoDao.xml`如下：

```xml

<resultMap id="ManagerInfoMap" type="managerInfo">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="name" column="name"/>
    <result property="password" column="password"/>
    <result property="salt" column="salt"/>
    <result property="state" column="state"/>
    <collection property="roles" ofType="sysRole">
        <id property="id" column="role_id"/>
        <result property="role" column="role_role"/>
        <collection property="permissions" ofType="permission">
            <id property="id" column="perm_id"/>
            <result property="permission" column="perm_permission"/>
        </collection>
    </collection>
</resultMap>

<select id="findByUsername" resultMap="ManagerInfoMap">
SELECT DISTINCT
A.id AS id,
A.username AS username,
A.name AS name,
A.password AS password,
A.salt AS salt,
A.state AS state,
C.id AS role_id,
C.role AS role_role,
E.id AS perm_id,
E.permission AS perm_permission
FROM t_manager A
LEFT JOIN t_manager_role B ON A.id=B.manager_id
LEFT JOIN t_role C ON B.role_id=C.id
LEFT JOIN t_role_permission D ON C.id=D.role_id
LEFT JOIN t_permission E ON D.permission_Id=E.id
WHERE username=#{username}
LIMIT 1
</select>
```

`ManagerInfo.java`如下：

```java
public class ManagerInfo extends Manager implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 一个管理员具有多个角色
     */
    private List<SysRole> roles;// 一个用户具有多个角色
}
```

最后来几张效果图：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro05.png)

admin角色登录之后：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-shiro06.png)

## 参考文章

* [Spring boot 中使用Shiro](https://www.jianshu.com/p/0f2049a3983b)
* [Apache Shiro和Spring boot的结合使用](http://weiqingfei.iteye.com/blog/2307860)
* [Spring Boot Shiro权限控制](https://mrbird.cc/Spring-Boot-Shiro%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6.html)

## GitHub源码

[springboot-shiro](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-shiro)
