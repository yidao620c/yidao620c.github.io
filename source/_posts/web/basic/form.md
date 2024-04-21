---
title: 防止表单重复提交
date: 2016-07-29 09:22:08 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

在平时开发中，如果网速比较慢的情况下，用户提交表单后，发现服务器半天都没有响应，
那么用户可能会以为是自己没有提交表单，就会再点击提交按钮重复提交表单，我们在开发中必须防止表单重复提交。
<!-- more -->

## 重复提交的几个场景

**场景一**

在网络延迟的情况下让用户有时间点击多次submit按钮导致表单重复提交

**场景二**

表单提交后用户点击【刷新】按钮导致表单重复提交，点击浏览器的刷新按钮，
就是把浏览器上次做的事情再做一次，因为这样也会导致表单重复提交。

**场景三**

用户提交表单后，使用浏览器【后退】按钮重复之前的操作，导致重复提交表单。

## 解决办法

### JavaScript方案

针对【场景一】，可通过页面前端的JavaScript方案来解决重复提交问题。

可通过javascript来防止表单重复提交，表单提交之后，
将提交按钮设置为不可用，让用户没有机会点击第二次提交按钮，代码如下：

```js
function dosubmit(){
    //获取表单提交按钮
    var btnSubmit = document.getElementById("submit");
    //将表单提交按钮设置为不可用，这样就可以避免用户再次点击提交按钮
    btnSubmit.disabled= "disabled";
    //返回true让表单可以正常提交
    return true;
}
```

### Session方案

对于【场景二】和【场景三】导致表单重复提交的问题，既然客户端无法解决，
那么就在服务器端解决，在服务器端解决就需要用到session了。

具体的做法：在服务器端生成一个唯一的随机标识号，专业术语称为Token(令牌)，同时在当前用户的Session域中保存这个Token。
然后将Token发送到客户端的Form表单中，在Form表单中使用隐藏域来存储这个Token，
表单提交的时候连同这个Token一起提交到服务器端，然后在服务器端判断客户端提交上来的Token与服务器端生成的Token是否一致，
如果不一致，那就是重复提交了，此时服务器端就可以不处理重复提交的表单。如果相同则处理表单提交，
处理完后清除当前用户的Session域中存储的标识号。

在下列情况下，服务器程序将拒绝处理用户提交的表单请求：

1. 存储Session域中的Token(令牌)与表单提交的Token(令牌)不同。
2. 当前用户的Session中不存在Token(令牌)。
3. 用户提交的表单数据中没有Token(令牌)。

具体的例子：

1.创建FormServlet，用于生成Token(令牌)和跳转到form.jsp页面

```java
package xdp.gacl.session;

import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class FormServlet extends HttpServlet {
    private static final long serialVersionUID = -884689940866074733L;

    public void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        String token = TokenProccessor.getInstance().makeToken();//创建令牌
        System.out.println("在FormServlet中生成的token："+token);
        request.getSession().setAttribute("token", token);  //在服务器使用session保存token(令牌)
        request.getRequestDispatcher("/form.jsp").forward(request, response);//跳转到form.jsp页面
    }

    public void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response);
    }
}
```

生成Token的工具类TokenProccessor：

```java
package xdp.gacl.session;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Random;
import sun.misc.BASE64Encoder;

public class TokenProccessor {

    /*
     *单例设计模式（保证类的对象在内存中只有一个）
     *1、把类的构造函数私有
     *2、自己创建一个类的对象
     *3、对外提供一个公共的方法，返回类的对象
     */
    private TokenProccessor(){}
    
    private static final TokenProccessor instance = new TokenProccessor();
    
    /**
     * 返回类的对象
     * @return
     */
    public static TokenProccessor getInstance(){
        return instance;
    }
    
    /**
     * 生成Token
     * Token：Nv6RRuGEVvmGjB+jimI/gw==
     * @return
     */
    public String makeToken(){  //checkException
        //  7346734837483  834u938493493849384  43434384
        String token = (System.currentTimeMillis() + new Random().nextInt(999999999)) + "";
        //数据指纹   128位长   16个字节  md5
        try {
            MessageDigest md = MessageDigest.getInstance("md5");
            byte md5[] =  md.digest(token.getBytes());
            //base64编码--任意二进制编码明文字符   adfsdfsdfsf
            BASE64Encoder encoder = new BASE64Encoder();
            return encoder.encode(md5);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}
```

2.在form.jsp中使用隐藏域来存储Token(令牌)

```jsp
<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
<title>form表单</title>
</head>
<body>
    <form action="${pageContext.request.contextPath}/servlet/DoFormServlet" method="post">
        <%--使用EL表达式取出存储在session中的token--%>
        <input type="hidden" name="token" value="${token}"/> 
        用户名：<input type="text" name="username"> 
        <input type="submit" value="提交">
    </form>
</body>
</html>
```

3.DoFormServlet处理表单提交

```java
package xdp.gacl.session;

import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class DoFormServlet extends HttpServlet {

    public void doGet(HttpServletRequest request, HttpServletResponse response)
                throws ServletException, IOException {

            boolean b = isRepeatSubmit(request);//判断用户是否是重复提交
            if(b==true){
                System.out.println("请不要重复提交");
                return;
            }
            request.getSession().removeAttribute("token");//移除session中的token
            System.out.println("处理用户提交请求！！");
        }
        
        /**
         * 判断客户端提交上来的令牌和服务器端生成的令牌是否一致
         * @param request
         * @return 
         *         true 用户重复提交了表单 
         *         false 用户没有重复提交表单
         */
        private boolean isRepeatSubmit(HttpServletRequest request) {
            String client_token = request.getParameter("token");
            //1、如果用户提交的表单数据中没有token，则用户是重复提交了表单
            if(client_token==null){
                return true;
            }
            //取出存储在Session中的token
            String server_token = (String) request.getSession().getAttribute("token");
            //2、如果当前用户的Session中不存在Token(令牌)，则用户是重复提交了表单
            if(server_token==null){
                return true;
            }
            //3、存储在Session中的Token(令牌)与表单提交的Token(令牌)不同，则用户是重复提交了表单
            if(!client_token.equals(server_token)){
                return true;
            }
            
            return false;
        }

    public void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        doGet(request, response);
    }
}
```

### 使用Post/Redirect/Get模式

针对【场景二】和【场景三】，除了使用Session方案，还有一种方案可以实现。
这就是所谓的 [Post-Redirect-Get (PRG)模式](http://www.theserverside.com/news/1365146/Redirect-After-Post)。

简言之，当用户提交了表单后，你去执行一个客户端的重定向，转到提交成功信息页面。
这能避免用户按F5导致的重复提交，而其也不会出现浏览器表单重复提交的警告，也能消除按浏览器前进和后退按导致的同样问题。


