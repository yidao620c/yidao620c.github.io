---
title: CAS教程 - 实现单点登录
date: 2019-01-09 18:16:55 +0800
toc: true
categories: [网络安全]
tags: [ CAS ]
---

通过前面几篇的介绍，熟悉了CAS Server的运行和调试，这一篇演示一个实际的单点登录例子。
统一使用 SpringBoot+Meven 构建。

| 项目          | 地址                              | 说明              |
|-------------|---------------------------------|-----------------|
| cas-overlay | https://cas.server.com:8443/cas | cas-server 服务端  |
| cas-app1    | http://app1.com:8181            | cas-client 客户端1 |
| cas-app2    | http://app2.com:8282            | cas-client 客户端2 |

接下来，老夫就要开始飙车了。
<!-- more -->

## 一、配置hosts文件

```
# SSO单点登录Demo
127.0.0.1    cas.server.com
127.0.0.1    app1.com
127.0.0.1    app2.com
```

## 二、配置cas服务端

**配置json服务注册**

1. 在src/main/resources/目录下创建services文件夹
2. 在war包中，找到HTTPSandIMAPS-10000001.json文件，并copy到services文件夹中
3. 将HTTPSandIMAPS-10000001.json文件内容修改如下

```json
{
  "@class" : "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" : "^(https|http|imaps)://.*",
  "name" : "HTTPS and HTTP and IMAPS",
  "id" : 10000001,
  "description" : "This service definition authorizes all application urls that support HTTPS and HTTP and IMAPS protocols.",
  "evaluationOrder" : 10000
}
```

默认cas支持https，不支持http客户端站点来登录，所以需要手动进行配置兼容。
如果不做此操作就会出现如下图：未认证授权的服务，所以还是乖乖的听话，跟着我的飙车轨道，别跑丢了。

体json文件中代表什么含义，后续篇会详细讲解到。

**application.properties 配置**

application.properties文件中，增加配置项cas.serviceRegistry.initFromJson=true表示开启了json注册服务。

另外为了Demo尽可能的简单方便理解，我这里启用静态账号密码方式认证（非JDBC和Rest）

```properties
##
# CAS Authentication Credentials
#
cas.authn.accept.users=test::test

##
# 开启json服务注册
#
cas.serviceRegistry.initFromJson=true

##
# 登出后允许跳转到指定页面
#
cas.logout.followServiceRedirects=true
```

## 三、配置客户端

客户端 app1 和 app2 的项目逻辑相同，不多说。

**pom.xml 配置**

添加 cas-client 包依赖，我用的 3.5.1 版本

```xml
<properties>
    <cas.client.version>3.5.1</cas.client.version>
</properties>
<!--cas的客户端 -->
<dependency>
   <groupId>org.jasig.cas.client</groupId>
   <artifactId>cas-client-core</artifactId>
   <version>${java.cas.client.version}</version>
</dependency>
```

源代码我已上传至GitHub，主要几个类如下。

CasClientConfig.java是CAS 客户端配置类，这个最重要。

```java
@Configuration
public class CasClientConfig {

    /*========================== SSO配置-开始  ============================*/

    /**
     * SingleSignOutFilter 登出过滤器
     * 该过滤器用于实现单点登出功能，可选配置
     *
     * @return
     */
    @Bean
    public FilterRegistrationBean filterSingleRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new SingleSignOutFilter());
        // 设定匹配的路径
        registration.addUrlPatterns("/*");
        Map<String, String> initParameters = new HashMap();
        initParameters.put("casServerUrlPrefix", CasConfig.CAS_SERVER_LOGIN_PATH);
        registration.setInitParameters(initParameters);
        // 设定加载的顺序
        registration.setOrder(1);
        return registration;
    }

    /**
     * SingleSignOutHttpSessionListener 添加监听器
     * 用于单点退出，该过滤器用于实现单点登出功能，可选配置
     *
     * @return
     */
    @Bean
    public ServletListenerRegistrationBean<EventListener> singleSignOutListenerRegistration() {
        ServletListenerRegistrationBean<EventListener> registrationBean = new ServletListenerRegistrationBean<EventListener>();
        registrationBean.setListener(new SingleSignOutHttpSessionListener());
        registrationBean.setOrder(1);
        return registrationBean;
    }

    /**
     * Cas30ProxyReceivingTicketValidationFilter 验证过滤器
     * 该过滤器负责对Ticket的校验工作，必须启用它
     *
     * @return
     */
    @Bean
    public FilterRegistrationBean filterValidationRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new Cas30ProxyReceivingTicketValidationFilter());

        // 设定匹配的路径
        registration.addUrlPatterns("/*");
        Map<String, String> initParameters = new HashMap();
        initParameters.put("casServerUrlPrefix", CasConfig.CAS_SERVER_PATH);
        initParameters.put("serverName", CasConfig.SERVER_NAME);

        // 是否对serviceUrl进行编码，默认true：设置false可以在302对URL跳转时取消显示;jsessionid=xxx的字符串
        // 观察CommonUtils.constructServiceUrl方法可以看到
        initParameters.put("encodeServiceUrl", "false");

        registration.setInitParameters(initParameters);
        // 设定加载的顺序
        registration.setOrder(1);
        return registration;
    }

    /**
     * AuthenticationFilter 授权过滤器
     *
     * @return
     */
    @Bean
    public FilterRegistrationBean filterAuthenticationRegistration() {

        FilterRegistrationBean registration = new FilterRegistrationBean();
        Map<String, String> initParameters = new HashMap();

        registration.setFilter(new AuthenticationFilter());
        registration.addUrlPatterns("/*");
        initParameters.put("casServerLoginUrl", CasConfig.CAS_SERVER_LOGIN_PATH);
        initParameters.put("serverName", CasConfig.SERVER_NAME);

        // 不拦截的请求 .* 有后缀的文件
        initParameters.put("ignorePattern", ".*");

        // 表示过滤所有
        initParameters.put("ignoreUrlPatternType", "com.xncoding.cas.config.SimpleUrlPatternMatcherStrategy");

        registration.setInitParameters(initParameters);
        // 设定加载的顺序
        registration.setOrder(1);
        return registration;
    }

    /**
     * AssertionThreadLocalFilter
     * <p>
     * 该过滤器使得开发者可以通过org.jasig.cas.client.util.AssertionHolder来获取用户的登录名。
     * 比如AssertionHolder.getAssertion().getPrincipal().getName()。
     *
     * @return
     */
    @Bean
    public FilterRegistrationBean filterAssertionThreadLocalRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new AssertionThreadLocalFilter());
        // 设定匹配的路径
        registration.addUrlPatterns("/*");
        // 设定加载的顺序
        registration.setOrder(1);
        return registration;
    }

    /*
     * HttpServletRequestWrapperFilter wraper过滤器
     * 该过滤器负责实现HttpServletRequest请求的包裹，
     * 比如允许开发者通过HttpServletRequest的getRemoteUser()方法获得SSO登录用户的登录名，可选配置。
     *
     * @return
     */
    @Bean
    public FilterRegistrationBean filterWrapperRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new HttpServletRequestWrapperFilter());
        // 设定匹配的路径
        registration.addUrlPatterns("/*");
        // 设定加载的顺序
        registration.setOrder(1);
        return registration;
    }

    /* =========================   SSO配置-结束   ========================*/
}

```

URL拦截匹配规则类：

```java
public class SimpleUrlPatternMatcherStrategy implements UrlPatternMatcherStrategy {

    /**
     * 机能概要: 判断是否匹配这个字符串
     *
     * @param url 用户请求的连接
     * @return true : 不拦截
     * false :必须得登录了
     */
    @Override
    public boolean matches(String url) {

        if (url.contains("/logout")) {
            return true;
        }

        List<String> list = Arrays.asList(
                "/",
                "/index",
                "/favicon.ico"
        );

        String name = url.substring(url.lastIndexOf("/"));
        if (name.contains("?")) {
            name = name.substring(0, name.indexOf("?"));
        }

        System.out.println("name：" + name);
        boolean result = list.contains(name);
        if (!result) {
            System.out.println("拦截URL：" + url);
        }
        return result;
    }

    /**
     * 正则表达式的规则，这个地方可以是web传递过来的
     */
    @Override
    public void setPattern(String pattern) {

    }
}
```

Cas的一些配置项，这里我写成常量了，其实作为Properties配置文件才是最好：

```java
public class CasConfig {

    /**
     * 当前应用程序的baseUrl（注意最后面的斜线）
     */
    public static String SERVER_NAME = "http://app1.com:8080/";


    /**
     * App1 登出成功url
     */
    public static String APP_LOGOUT_PATH = SERVER_NAME + "user/logout/success";


    /**
     * CAS服务器地址
     */
    public static String CAS_SERVER_PATH = "https://cas.server.com:8443/cas";

    /**
     * CAS登陆服务器地址
     */
    public static String CAS_SERVER_LOGIN_PATH = "https://cas.server.com:8443/cas/login";

    /**
     * CAS登出服务器地址
     */
    public static String CAS_SERVER_LOGOUT_PATH = "https://cas.server.com:8443/cas/logout";

}
```

然后就是几个Controller接口类，定义的/books是需要登录才能访问，而/index不需要拦截。这里就不贴了。

## 几个常用配置项

1. cas.logout.followServiceRedirects=true 是否允许客户端Logout后重定向到service参数指定的资源
2. tgt.maxTimeToLiveInSeconds=28800 指定Session的最大有效时间,即从生成到指定时间后就将超时,默认28800s,即8小时
3. tgt.timeToKillInSeconds=7200 指定用户操作的超时时间,即用户在多久不操作后就超时,默认7200s,即2小时。
   还要注意客户端web.xml配置的超时时间，即只有客户端配置超时时间不大于tgt.timeToKillInSeconds时才能看见服务端设置的效果
4. st.timeToKillInSeconds=10 指定service ticket的有效时间,默认10s，
   这也是debug追踪CAS应用认证过程中经常会失败的原因,因为追踪的时候service ticket已经过了10秒有效期了
5. slo.callbacks.disabled=false 是否禁用单点登出




