---
title: Nginx重新编译添加模块
date: 2018-03-20 15:11:03 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

一般的应用都是通过Nginx来做为反向代理，并且Nginx还可能是多层的。如果想在内部服务里面获取最原始的客户端IP地址。 则需要做一些配置。

<!-- more -->

## 最外层Nginx

为了防止`X-Forwarded-For`头的伪造，可在最外层Nginx将`X-Forwarded-For`设置为真实IP `$remote_addr`。
`$remote_addr`是获取的是直接TCP连接的客户端IP（类似于Java中的`request.getRemoteAddr()`）， 这个是无法伪造的，即使客户端伪造也会被覆盖掉，而不是追加。

```nginx
upstream innerservice {
    server 192.168.12.33:9001;
    server 192.168.12.34:9002;
}
server {
    listen    9000;
    server_name  192.168.12.22;
    root         /usr/share/nginx/html;
    include /etc/nginx/default.d/*.conf;
    location / {
        proxy_pass http://innerservice;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

## 后续Nginx配置

```nginx
upstream innerservice {
    server 192.168.12.33:9001;
    server 192.168.12.34:9002;
}
server {
    listen    9000;
    location / {
        proxy_pass http://innerservice;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Java中获取真实IP

``` java
/**
 * Description：客户端信息获取过滤器
 */
public class ClientContextFilter extends OncePerRequestFilter {

    /**
     * Nginx默认增加的IP地址头
     */
    public static final String HEADER_X_FORWARDED_FOR = "X-Forwarded-For";

    /**
     * 内部微服务之间调用增加的IP地址头
     */
    public static final String HEADER_X_REMOTE_USER_IP = "X-Remote-User-IP";

    /**
     * IP响应头的分割符号
     */
    private static final String IP_HEADER_DELIMITER = ",";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            String remoteUserIP = loadRemoteUserIP(request);
            ClientContext context = new ClientContext();
            context.setClientIP(request.getRemoteAddr());
            context.setRemoteUserIP(remoteUserIP);
            ClientContextHolder.setContext(context);
            filterChain.doFilter(request, response);
        } finally {
            ClientContextHolder.clearContext();
        }
    }

    private String loadRemoteUserIP(HttpServletRequest request) {
        String xForwardedHeader = request.getHeader(HEADER_X_FORWARDED_FOR);
        // 先尝试从X-Forwarded-For获取
        if (xForwardedHeader != null && xForwardedHeader.trim().length() > 0) {
            String[] ips = xForwardedHeader.split(IP_HEADER_DELIMITER);
            return ips[0].trim();
        } else {
            // 尝试从内部自定义头上获取
            String internalIPHeader = request.getHeader(HEADER_X_REMOTE_USER_IP);
            if (null != internalIPHeader && internalIPHeader.trim().length() > 0) {
                return internalIPHeader.trim();
            } else {
                return null;
            }
        }
    }
}
```

原理是在服务往后面传递的时候，定义一个自定义头，将真实IP放到这个header中。 后台服务可取两个IP地址

* `request.getRemoteAddr()` - 获取调用方服务的内网IP地址
* `loadRemoteUserIP` - 获取最源头的真实客户端IP地址
