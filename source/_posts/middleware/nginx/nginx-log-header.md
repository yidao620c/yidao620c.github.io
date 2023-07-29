---
title: Nginx打印请求头和响应头
date: 2018-03-23 09:21:07 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

在Nginx调试问题的时候经常需要打印请求头和响应头信息，这里记录一下Nginx日志的配置方法。

<!-- more -->

编辑nginx.conf配置文件。

```nginx
http {
    log_format log_req_resp '$remote_addr - $remote_user [$time_local] ' '"$request" 
        $status $body_bytes_sent ' "$http_referer" "$http_user_agent"  
        $http_x_real_ip $http_x_forwarded_for $request_time $content_type 
        $host $http_authorization $http_x_sdk_date $http_x_sdk_content_sha256  
        '"$http_referer" "$http_user_agent" $request_time req_body:"$request_body"'  
        ' resp_body:"$resp_body" resp_header:"$resp_header"';
    log_format log_req_resp '$remote_addr | $http_x_real_ip | $http_x_forwarded_for ' 
        '"$request" $status' "$http_referer" "$http_user_agent"  $request_time 
        $content_type $host 'resp_header:"$resp_header"';
    server {
        access_log logs/access.log log_req_resp; 
        set $req_header "";
        set $resp_header "";
        header_filter_by_lua '
            local h = ngx.resp.get_headers()
            for k, v in pairs(h) do
                if type(v) == "table" then
                    for k, vv in pairs(v) do
                        ngx.var.resp_header = ngx.var.resp_header..k..": "..vv;
                    end
                else
                    ngx.var.resp_header = ngx.var.resp_header..k..": "..v;
                end
            end
        ';
        set $resp_body "";
        body_filter_by_lua '
          local resp_body = string.sub(ngx.arg[1], 1, 1000)
          ngx.ctx.buffered = (ngx.ctx.buffered or "") .. resp_body
          if ngx.arg[2] then
             ngx.var.resp_body = ngx.ctx.buffered
          end
        ';
    }
}
```

**注意：自定义响应头通过使用http_开头即可。**

反向代理增加请求头

```nginx
location /some/path/ {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://localhost:8000;
}
```