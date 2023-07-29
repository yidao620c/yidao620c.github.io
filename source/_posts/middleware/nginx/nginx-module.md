---
title: Nginx重新编译添加模块
date: 2018-03-17 22:47:02 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

编译安装Nginx的时候，有些模块默认并不会安装，比如http_ssl_module，那么为了让Nginx支持HTTPS，必须添加这个模块。

下面讲解如何在已经安装过后再次添加新的模块。

1、找到安装nginx的源码根目录(即安装包存放目录)，如果没有的话下载新的源码并解压

```bash
cd software
ls
nginx-1.10.2  nginx-1.10.2.tar.gz
```

<!-- more -->

2、查看nginx版本极其编译参数

```bash
/usr/local/nginx/sbin/nginx -V
```

3、进入nginx源码目录

```bash
cd nginx-1.10.2
```

4、重新编译的代码和模块

```bash
./configure --prefix=/usr/local/nginx --with-http_ssl_module
```

5、执行`make`（注意：千万别`make install`，否则就覆盖安装了），
make完之后在/software/nginx-1.10.2/objs目录下就多了个nginx，这个就是新版本的程序了。

6、备份旧的nginx程序

```bash
cd /usr/local/nginx/sbin/
mv nginx nginx_bak
```

7、把新的nginx程序复制到/usr/local/nginx/sbin/下

```bash
cp /software/nginx-1.10.2/objs/nginx /usr/local/nginx/sbin/
```

8、测试新的nginx程序是否正确

```bash
/usr/local/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

9、平滑启动服务

```bash
/usr/local/nginx/sbin/nginx -s reload
```

10、查看模块是否已安装

```bash
/usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.10.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-4) (GCC)
built with OpenSSL 1.0.1e-fips 11 Feb 2013
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --with-http_ssl_module
```

11、重启Nginx

```bash
./nginx -s quit
./nginx
```

nginx重新加载模块完成！

