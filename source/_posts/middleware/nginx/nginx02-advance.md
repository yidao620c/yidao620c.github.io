---
title: nginx笔记 - 进阶篇
date: 2017-01-09 12:25:16 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

本篇介绍nginx的一些进阶使用方法，包括反向代理、虚拟主机、负载均衡、页面缓存等等。

反向代理（Reverse Proxy）方式是指用代理服务器来接受网络上的连接请求，然后将请求转发给内部网络上的服务器，
并将从服务器上得到的结果返回给请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。

举个例子，一个用户访问 `http://www.xiongneng.cc/readme`，但是`www.xiongneng.cc`上并不存在readme页面，
它是从另外一台服务器上取回来，然后作为自己的内容返回给用户。但是用户并不知情这个过程。对用户来说，
就像是直接从`www.xiongneng.cc`获取readme页面一样。
这里`www.xiongneng.cc`这个域名对应的服务器就设置了反向代理功能。
<!-- more -->

反向代理服务器，对于客户端而言它就像是原始服务器，并且客户端不需要进行任何特别的设置。
客户端向反向代理的命名空间中的内容发送普通请求，接着反向代理将判断向何处(原始服务器)转交请求，
并将获得的内容返回给客户端，就像这些内容原本就是它自己的一样。如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/nginx-proxy.png)

## 应用场景

反向代理的典型用途是将防火墙后面的服务器提供给Internet用户访问，加强安全防护。
反向代理还可以为后端的多台服务器提供负载均衡，或为后端较慢的服务器提供缓冲服务。
另外，反向代理还可以启用高级URL策略和管理技术，
从而使处于不同web服务器系统的web页面同时存在于同一个URL空间下。

## 操作实例

Nginx 的其中一个用途是做 HTTP 反向代理，下面简单介绍 Nginx 作为反向代理服务器的方法。

> 场景描述：访问服务器上的`README.md`文件`http://www.xiongneng.cc/README.md`，
> 服务器进行反向代理，从`https://github.com/yidao620c/scrapy-cookbook/blob/master/README.md`获取页面内容。

`nginx.conf`配置示例：

```nginx
    ## 下面配置反向代理的参数
    server {
        listen    80;
        ## 1. 用户访问 http://ip:port，则反向代理到 https://github.com
        location / {
            proxy_pass  https://github.com;
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

        ## 2.用户访问 http://ip:port/README.md，则反向代理到
        ##   https://github.com/.../README.md
        location /README.md {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://github.com/yidao620c/scrapy-cookbook/blob/master/README.md;
        }
    }
```

重启报错，因为我的nginx没有配置https

## 配置HTTPS

使用https保证机器上安装了openssl和openssl-devel

```bash
yum -y install openssl*
```

颁发证书给自己

```bash
openssl genrsa -des3 -out server.key 1024            \\ 用于生成rsa私钥文件
openssl req -new -key server.key -out server.csr     \\ openssl req 用于生成证书请求
openssl rsa -in server.key -out server_nopwd.key      \\利用openssl进行RSA为公钥加密
openssl x509 -req -days 365 -in server.csr -signkey server_nopwd.key -out server.crt
mv server.crt server_nopwd.key /usr/local/nginx/conf/
```

3，添加nginx证书

```nginx
server {
    listen 443 ssl;
    ssl_certificate      server.crt; //证书路径，却对和相对路径都可以
    ssl_certificate_key  server_nopwd.key；
｝
```

保存退出

执行`nginx -t`检查，如果没问题的话就重启nginx

```
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:99
```

原因也很简单，nginx缺少http_ssl_module模块，编译安装的时候带上`--with-http_ssl_module`配置就行了，
但是现在的情况是我的nginx已经安装过了，怎么添加模块，其实也很简单，往下看：

做个说明，我的nginx的安装目录是`/usr/local/nginx`这个目录，我的源码包在`/root/nginx-1.10.3`目录

切换到源码包`cd /root/nginx-1.10.3`，查看nginx原有的模块

```bash
/usr/local/nginx/sbin/nginx -V
```

结果如下：

```
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
configure arguments:
```

那么我们的新配置信息就应该这样写，在configure arguments基础上面加：

```bash
./configure --prefix=/usr/local/nginx --with-http_ssl_module
```

运行上面的命令即可，等配置完，运行命令:

```bash
make
```

这里不要进行`make install`，否则就是覆盖安装。

然后备份原有已安装好的nginx

```bash
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```

然后将刚刚编译好的nginx覆盖掉原有的nginx（这个时候nginx要停止状态）:

```bash
systemctl stop nginx.service
cp ./objs/nginx /usr/local/nginx/sbin/
```

提示是否覆盖，输入y即可

然后启动nginx，仍可以通过命令查看是否已经加入成功

```bash
/usr/local/nginx/sbin/nginx -V
```

结果:

```
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
built with OpenSSL 1.0.1e-fips 11 Feb 2013
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --with-http_ssl_module
```

重启 nginx 后，我们打开浏览器，验证下反向代理的效果。
在浏览器地址栏中输入`http://www.xiongneng.cc/README.md`，返回的结果是我的GitHub上面的README页面:

![](https://xnstatic-1253397658.file.myqcloud.com/nginx-09.png)

## HTTPS配置实例

```nginx
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  api.enzhico.cn;

    location / {
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header 'Access-Control-Allow-Origin' '*';
            proxy_set_header 'Access-Control-Allow-Credentials' 'true';
            proxy_set_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,X-Requested-With';
            proxy_set_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
            proxy_pass http://119.29.12.177:8094/;
    }

    ssl_certificate "server.crt";
    ssl_certificate_key "server_nopwd.key";

}
```

## 虚拟主机

有时候我们需要在同一台主机上面托管多个应用，每个应用访问域名不同，这里我们可以使用基于域名的虚拟主机来实现。

假设我们在本地开发有3个项目，分别在hosts里映射到本地的127.0.0.1上：

```
127.0.0.1 www.xiongneng.com xiongneng.com
127.0.0.1 api.xiongneng.com
127.0.0.1 admin.xiongneng.com
```

有这样3个项目，分别对应于web根目录下的3个文件夹，我们用域名对应文件夹名字，这样子好记：

```
/home/xiongneng/www/www.xiongneng.com/
/home/xiongneng/www/api.xiongneng.com/
/home/xiongneng/www/admin.xiongneng.com/
```

每个目录下都有一个index.php文件，内容就是都是简单的输入自己的域名

下面我们就来搭建这3个域名的虚拟主机，很显然，我们要新建3个server来完成。建议将对虚拟主机进行配置的内容写进另外一个文件，
然后通过include指令包含进来，这样更便于维护和管理。不会使得这个nginx.conf内容太多：

```nginx
main
events {
    ....
}
http {
    ....
    include vhost/www.xiongneng.conf;
    include vhost/api.xiongneng.conf;
    include vhost/admin.xiongneng.conf;
    # 或者用 *.conf  包含
    # include vhost/*.conf
}
```

include：主模块指令，实现对配置文件所包含的文件的设定，可以减少主配置文件的复杂度。

既然每一个conf都是一个server，前面已经学习了一个完整的server写的了。下面就开始：

```nginx
# www.xiongneng.conf
server {
    listen 80;
    server_name www.xiongneng.com xiongneng.com;

    root /home/xiongneng/www/www.xiongneng.com/;
    index index.html index.htm;

    access_log /usr/local/var/log/nginx/www.xiongneng.access.log main;
    error_log /usr/local/var/log/nginx/www.xiongneng.error.log error;

    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        include        fastcgi.conf;
    }
}
```

其他两个类似就省略了，这样3个很精简的虚拟域名就搭建好了。重启下nginx，
然后打开浏览器访问一下这3个域名，就能看到对应的域名内容了。

## 负载均衡

别被这个名字给吓住了，其实做个服务器调度策略，通过集群部署来提供高可用，
对外的IP和域名是一样的，外界看起来就好像一台机器一样。

nginx通过`upstream`模块提供负载均衡配置，使用起来也非常简单。

有四种调度算法:

> **weight 轮询（默认）**<br/>
> 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端某台服务器宕机，故障系统被自动剔除，
> 使用户访问不受影响。weight。指定轮询权值，weight值越大，分配到的访问机率越高，
> 主要用于后端每个服务器性能不均的情况下。<br/>
> **ip_hash**<br/>
> 每个请求按访问IP的hash结果分配，这样来自同一个IP的访客固定访问一个后端服务器，
> 有效解决了动态网页存在的session共享问题。<br/>
> **fair（第三方）**<br/>
> 比上面两个更加智能的负载均衡算法。此种算法可以依据页面大小和加载时间长短智能地进行负载均衡，
> 也就是根据后端服务器的响应时间来分配请求，响应时间短的优先分配。Nginx本身是不支持fair的，
> 如果需要使用这种调度算法，必须下载Nginx的upstream_fair模块。<br/>
> **url_hash（第三方）**<br/>
> 按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率。
> Nginx本身是不支持url_hash的，如果需要使用这种调度算法，必须安装Nginx的hash软件包。

### 基于weight权重的负载

先来一个最简单的，weight权重的：

```nginx
upstream webservers{
  server 192.168.212.200 down;
  server 192.168.212.201 weight=10 max_fails=2 fail_timeout=30s;
  server 192.168.212.202 backup;
}
server {
  listen 80;
  server_name upstream.iyangyi.com;

  access_log /usr/local/var/log/nginx/upstream.xn.access.log main;
  error_log /usr/local/var/log/nginx/upstream.xn.error.log error;

  location / {
      proxy_pass http://webservers;
      proxy_set_header  X-Real-IP  $remote_addr;
  }
}
```

我们看几个参数:

> **max_fails**: 允许请求失败的次数，默认为1。当超过最大次数时，
> 返回proxy_next_upstream模块定义的错误。
>
> **fail_timeout**: 在经历了max_fails次失败后，暂停服务的时间。
> max_fails可以和fail_timeout一起使用，进行健康状态检查。
>
> **down**: 表示这台机器暂时不参与负载均衡，相当于注释掉了。
>
> **backup**: 表示这台机器是备用机器，是其他的机器不能用的时候，这台机器才会被使用，俗称备胎

### 基于ip_hash的负载

看配置文件先:

```nginx
upstream webservers{
  ip_hash;
  server 192.168.212.200 weight=1 max_fails=2 fail_timeout=30s;
  server 192.168.212.201 weight=1 max_fails=2 fail_timeout=30s;
  server 192.168.212.202 down;
}
```

注意几点:

1. ip_hash 模式下，最好不要设置weight参数，因为你设置了，
   就相当于手动设置了，将会导致很多的流量分配不均匀。
2. ip_hash 模式下，backup参数不可用，加了会报错，为啥呢？
   因为，本身我们的访问就是固定的了，其实，备用已经不管什么作用了。

## 页面缓存

现代web应用使用了大量的缓冲技术来加速访问，凡是遵循2/8原则的都应该用缓冲技术。
nginx提供反向代理的缓存功能，只需要简单配置下，就能将指定的一个页面缓存起来。

它的原理也很简单，就是匹配当前访问的url, hash加密后，去指定的缓存目录找，
看有没有，有的话就说明匹配到缓存了。

我们先来看一下一个简单的页面缓存的配置：

```nginx
http {
    proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=cache_zone:10m inactive=1d max_size=100m;
    upstream myproject {
        .....
    }
    server  {
        ....
        location ~ *\.php$ {
            proxy_cache cache_zone; #keys_zone的名字
            proxy_cache_key $host$uri$is_args$args; #缓存规则
            proxy_cache_valid any 1d;
            proxy_pass http://127.0.0.1:8080;
        }
    }
    ....
}
```

首先需要在http中加入`proxy_cache_path`，用来制定缓存的目录以及缓存目录深度制定等。它的格式如下：

```nginx
proxy_cache_path path [levels=number] keys_zone=zone_name:zone_size [inactive=time] [max_size=size];
```

> path是用来指定缓存在磁盘的路径地址。比如：`/data/nginx/cache`。那以后生存的缓存文件就会存在这个目录下。
>
> levels用来指定缓存文件夹的级数，可以是：levels=1, levels=1:1, levels=1:2, levels=1:2:3
> 可以使用任意的1位或2位数字作为目录结构分割符，如 X, X:X,或 X:X:X 例如: 2, 2:2, 1:1:2，但是最多只能是三级目录。

那这个里面的数字是什么意思呢。表示取hash值的个数。比如：

> 现在根据请求地址`localhost/index.php?a=4`用md5进行哈希，得到`e0bd86606797639426a92306b1b98ad9`
>
> levels=1:2 表示建立2级目录，把hash最后1位(9)拿出建一个目录，然后再把9前面的2位(ad)拿来建一个目录,
> 那么缓存文件的路径就是`/data/nginx/cache/9/ad/e0bd86606797639426a92306b1b98ad9`
>
> 以此类推：levels=1:1:2表示建立3级目录，把hash最后1位(9)拿出建一个目录，然后再把9前面的1位(d)建一个目录,
> 最后把d前面的2位(8a)拿出来建一个目录 那么缓存文件的路径就是`/data/nginx/cache/9/d/8a/e0bd86606797639426a92306b1b98ad9`

keys_zone 所有活动的key和元数据存储在共享的内存池中，这个区域用keys_zone参数指定。zone_name指的是共享池的名称，
zone_size指的是共享池的大小。注意每一个定义的内存池必须是不重复的路径，例如：

```nginx
proxy_cache_path  /data/nginx/cache/one  levels=1      keys_zone=one:10m;
proxy_cache_path  /data/nginx/cache/two  levels=2:2    keys_zone=two:100m;
proxy_cache_path  /data/nginx/cache/three  levels=1:1:2  keys_zone=three:1000m;
```

> inactive 表示指定的时间内缓存的数据没有被请求则被删除，默认inactive为10分钟。inactive=1d 1小时。inactive=30m 30分钟。
>
> max_size 表示单个文件最大不超过的大小。它被用来删除不活动的缓存和控制缓存大小，当目前缓存的值超出max_size指定的值之后，
> 超过其大小后最少使用数据（LRU替换算法）将被删除。max_size=10g表示当缓存池超过10g就会清除不常用的缓存文件。
>
> clean_time 表示每间隔自动清除的时间。clean_time=1m 1分钟清除一次缓存。

说完了这个很重要的参数。我们再来说在server模块里的几个配置参数：

> proxy_cache用来指定用哪个keys_zone的名字，也就是用哪个目录下的缓存。
> 上面我们指定了三个one,two,three。比如，我现在想用one这个缓存目录: proxy_cache one
>
> proxy_cache_key 这个其实蛮重要的，它用来指定生成hash的url地址的格式。根据这个key映射成一个hash值，
> 然后存入到本地文件。`proxy_cache_key $host$uri`表示无论后面跟的什么参数，都会访问一个文件，不会再生成新的文件。
> 而如果`proxy_cache_key $is_args$args`，那么传入的参数`localhost/index.php?a=4`与
> `localhost/index.php?a=44`将映射成两个不同hash值的文件。
>
> proxy_cache_key 默认是 "$scheme$host$request_uri"。但是一般我们会把它设置成：
> `$host$uri$is_args$args`一个完整的url路径。
>
> proxy_cache_valid 它是用来为不同的http响应状态码设置不同的缓存时间。

```nginx
proxy_cache_valid  200 302 10m;
proxy_cache_valid  301 1h;
proxy_cache_valid  any 1h; #所有的状态都缓存1小时
```

表示为http status code 为200和302的设置缓存时间为10分钟，404代码缓存1分钟。如果只定义时间：

## URL重写

先看一下是怎么用的，看2个例子，然后我们再一点一点讲每个的使用方法：

```nginx
location /download/ {
    if ($forbidden) {
        return   403;
    }
    if ($slow) {
        limit_rate  10k;
    }
    rewrite ^/(download/.*)/media/(.*)\..*$  /$1/mp3/$2.mp3 break;
    ......
}
```

```nginx
location / {
    root   html;
    index  index.html index.htm;
    rewrite ^/bbs/(.*)$ http://192.168.18.201/forum/$1;
}
```

上面2个例子就是利用rewrite来完成URL重写的。我们慢慢来看它的用法。

### break

break和编程语言中的用法一样，就是跳出某个逻辑。

> 语法：break
>
> 默认值：none
>
> 使用字段：server, location, if

```nginx
if (!-f $request_filename) {
  break;
}
```

上面这个例子就是在if里面使用break,意思是如果访问的文件名不存在，就跳出。后续会有更多的例子。

### if

if 判断一个条件，如果条件成立，则后面的大括号内的语句将执行，相关配置从上级继承。

```nginx
if (!-f $request_filename) {
  break;
  proxy_pass  http://127.0.0.1;
}
```

### return

这个指令结束执行配置语句并为客户端返回状态代码，可以使用下列的值：
204，400，402-406，408，410, 411, 413, 416与500-504。此外，非标准代码444将关闭连接并且不发送任何的头部。

> 语法：return code
>
> 默认值：none
>
> 使用字段：server, location, if

### rewrite

> 语法：rewrite regex replacement flag
>
> 默认值：none
>
> 使用字段：server, location, if

regex 表示用来匹配的正则，replacement 表示用来替换的，flag 是尾部的标记

flag可以是以下的值：

> last - url重写后，马上发起一个新的请求，再次进入server块，重试location匹配，超过10次匹配不到报500错误，地址栏url不变
>
> break - url重写后，直接使用当前资源，不再执行location里余下的语句，完成本次请求，地址栏url不变
>
> redirect - 返回302临时重定向，url会跳转，爬虫不会更新url。
>
> permanent - 返回301永久重定向。url会跳转，爬虫会更新url。
>
> 为空 - URL 不会变，但是内容已经变化，也是永久性的重定向。

下面是一些简单的常见的重写：

```nginx
rewrite ^/js/base.core.v3.js /js/base.core.v3.dev.js redirect;
rewrite ^/js/comment.frame.js /js/comment.frame.dev.js redirect;
rewrite ^/live-static/(.*)$ http://live.bilibili.com/public/$1 last;
```

