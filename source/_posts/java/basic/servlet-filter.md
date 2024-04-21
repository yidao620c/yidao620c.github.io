---
title: servlet filter详解
date: 2015-04-06 21:08:25 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

在写一个springmvc项目中想对用户的请求进行拦截，只有登录用户才能访问资源。
这时候可以使用到SpringMVC的拦截器Intercepter，但是这个只能局限在SpringMVC中使用，
如果想更加通用一点，最好使用Servlet Filter实现这个需求。

本文将通过几个实际的例子展示下Servlet中的Filter的使用。
<!-- more -->

### 我们为什么需要使用Filter？

通常我们会使用session来保存登录用户的信息，通过从session中取出保存的属性值来判断用户是否登录，
但是如果我们有大量的请求方法，每个人方法中都这样去判断就会有很多重复代码，将来我们想改动下逻辑，
那得改动所有的请求方法实现，所以这个是不可取的。

这时候就是使用Servlet Filter的时候了，它是可插拔的，对于普通的action方法来讲是透明的。
它会在执行其他方法之前或将结果返回给客户端之前来执行其他逻辑。

以下几种场景下我们会使用到Servlet Filter：

- 将请求的参数写入日志文件
- 对于资源的访问进行统一的授权与验证
- 在请求到达实际Servlet之前格式化请求内容或请求头
- 压缩返回数据后发送给客户端
- 修改返回内容，增加一些cookie、header等信息

前面提到过，Servlet是可插拔的，可以通过在web.xml中配置是否使用。如果我们定义了多个Filter，就会形成一个过滤器链。
通过实现接口javax.servlet.Filter来创建一个过过滤器。

### Filter接口

Filter接口包含了三个跟生命周期有关的方法，并且由Servlet容器来管理。它们分别是：

```java
void init(FilterConfig paramFilterConfig)
```

当容器初始化这个Filter的时候被调用，并且只会被调用一次。因此在这个方法里面我们可以初始化一些资源。
FilterConfig会被容器用来给Filter提供初始化参数以及Servlet Context对象。
我们可以在这个方法中抛出ServletException异常。

```java
doFilter(ServletRequest paramServletRequest, ServletResponse paramServletResponse, FilterChain paramFilterChain)
```

这个方法在每次执行过滤的时候被调用，request和response被作为参数传递进来，FilterChain表示过滤器链，这是典型的责任链模式的实现例子。

```java
void destroy()
```

这个方法再容器卸载掉filter的时候被调用。因此我们可以在里面将filter使用到的一些资源给释放掉。

### WebFilter注解

在Servlet3.0中引入了注解javax.servlet.annotation.WebFilter。无需配置，简单实用，
不过如果你需要经常改动Filter逻辑的话，还是建议你在web.xml文件中配置，因为代码中修改后必须重写编译发布才能生效。

### web.xml中的Filter配置详解

像下面这样声明一个filter：

```xml

<filter>
    <filter-name>RequestLoggingFilter</filter-name> <!-- mandatory -->
    <filter-class>com.journaldev.servlet.filters.RequestLoggingFilter</filter-class> <!-- mandatory -->
    <init-param> <!-- optional -->
        <param-name>test</param-name>
        <param-value>testValue</param-value>
    </init-param>
</filter>
```

然后定义一个mapping：

```xml

<filter-mapping>
    <filter-name>RequestLoggingFilter</filter-name> <!-- 必填 -->
    <url-pattern>/*</url-pattern> <!-- url-pattern 或 servlet-name必须指定至少一个 -->
    <servlet-name>LoginServlet</servlet-name>
    <dispatcher>REQUEST</dispatcher>
</filter-mapping>
```

### 日志和登录验证的filter示例

下面是login.html页面：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="US-ASCII">
    <title>Login Page</title>
</head>
<body>
<form action="LoginServlet" method="post">
    Username: <input type="text" name="user">
    <br>
    Password: <input type="password" name="pwd">
    <br>
    <input type="submit" value="Login">
</form>
</body>
</html>
```

LoginServlet负责验证客户端是否已经登录：

```java LoginServlet.java
package com.journaldev.servlet.session;
  
import java.io.IOException;
import java.io.PrintWriter;
  
import javax.servlet.RequestDispatcher;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

/**
 * Servlet implementation class LoginServlet
 */
@WebServlet("/LoginServlet")
public class LoginServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private final String userID = "admin";
    private final String password = "password";
  
    protected void doPost(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException {
  
        // get request parameters for userID and password
        String user = request.getParameter("user");
        String pwd = request.getParameter("pwd");
          
        if(userID.equals(user) && password.equals(pwd)){
            HttpSession session = request.getSession();
            session.setAttribute("user", "Pankaj");
            //setting session to expiry in 30 mins
            session.setMaxInactiveInterval(30*60);
            Cookie userName = new Cookie("user", user);
            userName.setMaxAge(30*60);
            response.addCookie(userName);
            response.sendRedirect("LoginSuccess.jsp");
        }else{
            RequestDispatcher rd = getServletContext().getRequestDispatcher("/login.html");
            PrintWriter out= response.getWriter();
            out.println("<font color=red>Either user name or password is wrong.</font>");
            rd.include(request, response);
        }
    }
}
```

验证通过后跳转到LoginSuccess.jsp：

```jsp LoginSuccess.jsp
<%@ page language="java" contentType="text/html; charset=US-ASCII"
    pageEncoding="US-ASCII"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=US-ASCII">
<title>Login Success Page</title>
</head>
<body>
<%
//allow access only if session exists
String user = (String) session.getAttribute("user");
String userName = null;
String sessionID = null;
Cookie[] cookies = request.getCookies();
if(cookies !=null){
for(Cookie cookie : cookies){
    if(cookie.getName().equals("user")) userName = cookie.getValue();
    if(cookie.getName().equals("JSESSIONID")) sessionID = cookie.getValue();
}
}
%>
<h3>Hi <%=userName %>, Login successful. Your Session ID=<%=sessionID %></h3>
<br>
User=<%=user %>
<br>
<a href="CheckoutPage.jsp">Checkout Page</a>
<form action="LogoutServlet" method="post">
<input type="submit" value="Logout" >
</form>
</body>
</html>
```

退出时我们并不需要进行验证，退出页面为：

```jsp CheckoutPage.jsp
<%@ page language="java" contentType="text/html; charset=US-ASCII"
    pageEncoding="US-ASCII"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=US-ASCII">
<title>Login Success Page</title>
</head>
<body>
<%
String userName = null;
String sessionID = null;
Cookie[] cookies = request.getCookies();
if(cookies !=null){
for(Cookie cookie : cookies){
    if(cookie.getName().equals("user")) userName = cookie.getValue();
}
}
%>
<h3>Hi <%=userName %>, do the checkout.</h3>
<br>
<form action="LogoutServlet" method="post">
<input type="submit" value="Logout" >
</form>
</body>
</html>
```

LogoutServlet在用户点击退出按钮时执行：

```java LogoutServlet.java
package com.journaldev.servlet.session;
  
import java.io.IOException;
  
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
  
/**
 * Servlet implementation class LogoutServlet
 */
@WebServlet("/LogoutServlet")
public class LogoutServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
         
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/html");
        Cookie[] cookies = request.getCookies();
        if(cookies != null){
        for(Cookie cookie : cookies){
            if(cookie.getName().equals("JSESSIONID")){
                System.out.println("JSESSIONID="+cookie.getValue());
                break;
            }
        }
        }
        //invalidate the session if exists
        HttpSession session = request.getSession(false);
        System.out.println("User="+session.getAttribute("user"));
        if(session != null){
            session.invalidate();
        }
        response.sendRedirect("login.html");
    }
  
} 
```

现在我们来创建日志和登录认证的两个filter：

```java RequestLoggingFilter.java
package com.journaldev.servlet.filters;

import java.io.IOException;
import java.util.Enumeration;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
  
/**
 * Servlet Filter implementation class RequestLoggingFilter
 */
@WebFilter("/RequestLoggingFilter")
public class RequestLoggingFilter implements Filter {
  
    private ServletContext context;
      
    public void init(FilterConfig fConfig) throws ServletException {
        this.context = fConfig.getServletContext();
        this.context.log("RequestLoggingFilter initialized");
    }
  
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        Enumeration<String> params = req.getParameterNames();
        while(params.hasMoreElements()){
            String name = params.nextElement();
            String value = request.getParameter(name);
            this.context.log(req.getRemoteAddr() + "::Request Params::{"+name+"="+value+"}");
        }
          
        Cookie[] cookies = req.getCookies();
        if(cookies != null){
            for(Cookie cookie : cookies){
                this.context.log(req.getRemoteAddr() + "::Cookie::{"+cookie.getName()+","+cookie.getValue()+"}");
            }
        }
        // pass the request along the filter chain
        chain.doFilter(request, response);
    }
  
    public void destroy() {
        //we can close resources here
    }
  
}
```

然后是另一个Filter：

```java AuthenticationFilter.java
package com.journaldev.servlet.filters;
  
import java.io.IOException;
  
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
  
@WebFilter("/AuthenticationFilter")
public class AuthenticationFilter implements Filter {
  
    private ServletContext context;
      
    public void init(FilterConfig fConfig) throws ServletException {
        this.context = fConfig.getServletContext();
        this.context.log("AuthenticationFilter initialized");
    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
  
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse res = (HttpServletResponse) response;
          
        String uri = req.getRequestURI();
        this.context.log("Requested Resource::"+uri);
          
        HttpSession session = req.getSession(false);
          
        if(session == null && !(uri.endsWith("html") || uri.endsWith("LoginServlet"))){
            this.context.log("Unauthorized access request");
            res.sendRedirect("login.html");
        }else{
            // pass the request along the filter chain
            chain.doFilter(request, response);
        }
    }
  
    public void destroy() {
        //close any resources here
    }
}
```

注意我们并不会对任何静态html页面或LoginServlet进行验证，下面我们在web.xml文件中配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
    <display-name>ServletFilterExample</display-name>
    <welcome-file-list>
        <welcome-file>login.html</welcome-file>
    </welcome-file-list>

    <filter>
        <filter-name>RequestLoggingFilter</filter-name>
        <filter-class>com.journaldev.servlet.filters.RequestLoggingFilter</filter-class>
    </filter>
    <filter>
        <filter-name>AuthenticationFilter</filter-name>
        <filter-class>com.journaldev.servlet.filters.AuthenticationFilter</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>RequestLoggingFilter</filter-name>
        <url-pattern>/*</url-pattern>
        <dispatcher>REQUEST</dispatcher>
    </filter-mapping>
    <filter-mapping>
        <filter-name>AuthenticationFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```

接下来我们试验下效果。此处省略88个字……