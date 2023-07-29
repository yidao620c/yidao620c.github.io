---
title: nginx笔记 - 配置HTTPS
date: 2017-01-12 20:16:16 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

TLS(transport layer security), 和它的后继者SSL是一个安全套接字层协议，是为了给普通的网络传输内容加密传输而来。

网站转成https是大势所趋。但是在国内，推进的过程显然要比国外慢很多。

现阶段如果将自己的网站改成https以后，会碰到这样的尴尬现象：如果在页面上引用了http://的链接或者图片，
用户在浏览器上会看到类似该网站是非安全网站的警告，对于网站运营者来说可以说非常冤。由于很多链接是第三方的，没有办法去控制。
对于api接口类的网站，就不存在混合的问题，所以首先应该从api后台接口部分开始用https。(ios已经强制要求接口地址必须为https了)

本章专门来讲解在nginx中如何配置https访问。

首先要明确的是要支持https就必须要有证书，这个证书可以是自己生成的，也可以从CA机构申请，在测试阶段可以自己生成，
但是如果要部署到生产环境，最好去CA机构申请，有免费和收费的证书，同时还分不同的证书种类。
<!-- more -->

## SSL证书类型

* DV（域名型SSL证书）: 最简单的, 只显示公司域名，一般来说搞这个就可以了。
* OV（企业型SSL证书）: 给企业用，显示公司名称
* EV（增强型SSL证书）: 地址栏上显示公司名，显示公司的详细信息。

三者的主要区别有以下几点：

1. 审核标准不一样，DV审核域名所有权，OV审核企业的身份，EV是审核最严格的证书；
2. 证书中显示信息不一样，DV只显示公司域名，OV证书显示公司名称，EV证书中显示公司的详细信息，
   并且通过IE7.0等高安全浏览器访问时，地址栏会变为绿色，表示诚信，防止钓鱼和欺诈网站。

证书申请流程大致为：填表签合同、付款、生成CSR，并且提交公司营业执照副本复印件并发送给我们、审核确认后，颁发证书。
申请DV证书无需人工审核，快速颁发。

申请OV证书需要贵公司提交申请证书的域名、企业营业执照副本复印件(有年检章和公章)、
填写OV证书申请表并由贵公司经理级别以上领导签字盖章、生成证书签名请求（CSR），一般只要资料准确，2-3个工作日就可以颁发证书。

生成CSR时主要有5个字段：

1. Common Name(通用名称)：填写您的服务器的全名(网址)，必须一个字不差；
1. Organization(机构名称)：您的机构的名称全名，不要填写缩写；
1. Organization Unit(申请机构的部门名称)：City or Locality(机构所在的城市)；
1. Province(机构所在的省份)；
1. Country(国家)：必须填写国家的两个字母简称,如中国就填 CN。

## 自签名证书

先讲解如何生成自签名的证书。

Nginx的安装和防火墙设置我前面已经说过，这里就不说了。

### 生成SSL证书

```
sudo mkdir /etc/ssl/private
sudo chmod 700 /etc/ssl/private
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```

这条命令会同时生成一个key文件和一个证书文件，这期间会提示很多问题让你填写，大部分都可以忽略，不过最重要的`Common Name `
不能忽略，
你需要填写你的域名。类似下面这样：

```
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:New York
Locality Name (eg, city) []:New York City
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Bouncy Castles, Inc.
Organizational Unit Name (eg, section) []:Ministry of Water Slides
Common Name (e.g. server FQDN or YOUR name) []:api.enzhico.net
Email Address []:admin@your_domain.com
```

生成的文件会在`/etc/ssl`相应的子目录下面。

另外我们还要生成一个 Diffie-Hellman 组：

```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

这个命令可能会运行比较长时间，耐心等待。生成的DH组文件放在`/etc/ssl/certs/dhparam.pem`，待会我们会用到。

### 配置nginx使用SSL

一般我们配置nginx会单独生成一个配置文件，创建`/etc/nginx/conf.d/ssl.conf`，内容如下：

```
server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name api.enzhico.net;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
}
```

另外还可以多加点配置选项：

```
server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name api.enzhico.net;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    
    ########################################################################
    # from https://cipherli.st/                                            #
    # and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html #
    ########################################################################

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    # Disable preloading HSTS for now.  You can use the commented out header line that includes
    # the "preload" directive if you understand the implications.
    #add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    ##################################
    # END https://cipherli.st/ BLOCK #
    ##################################
    
    root /usr/share/nginx/html;
    
    location / {
    }

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

### HTTP重定向到HTTPS

有时候你想让别人访问你的网址的时候，无论使用什么方式访问，最后都是HTTPS。创建`/etc/nginx/default.d/ssl-redirect.conf`，内容如下：

```
return 301 https://$host$request_uri/;
```

通过命令`nginx -t`测试一下配置是否正确，然后重启nginx。浏览器打开https网址看看，一般来讲浏览器会报一个警告，你信任此网址继续访问即可。

## Lets Encrypt 证书

大牌提供商的SSL证书可不便宜，对于大公司也许不算什么，但是对于小公司及个人来说贵了。
现在国外出现的免费SSL服务商`Let's Encrypt`，绝对是小公司或者开发者的福音。

这里整理了在CentOS7 + nginx安装和使用`Let's Encrypt`的完整过程。

官方网站：<https://letsencrypt.org>

申请let's encript 证书可以有三种方式：

1. [通过certbot脚本](https://certbot.eff.org/)
2. [通过支持Letencript的虚拟主机提供商](https://community.letsencrypt.org/t/web-hosting-who-support-lets-encrypt/6920)
3. [手工申请manual mode](https://certbot.eff.org/docs/using.html#manual)

没有特殊情况，首选采用certbot脚本方式。

`Let's Encrypt`证书的有效期只有90天，需要长期使用的话，需要在失效前进行延长申请。用certbot脚本工具，
可以将延期申请的脚本写到定时任务来自动完成，非常方便。

### 前提条件

* 拥有一个域名，例如`enzhico.net` (在国内主机的用的话，还需要通过ICP备案)
* 在域名服务器创建一条A记录，指向云主机的公网IP地址。例如`api.enzhico.net`指向`xxx.xxx.xxx.xxx`的IP地址
* 要等到新创建的域名解析能在公网上被解析到。

### 使用certbot-auto脚本安装Certbot

`certbot-auto`脚本会安装Certbot，并且能够自己解决RPM包和Python包依赖问题，同样非常方便。
同时`certbot-auto`是对certbot的封装，即`certbot-auto`提供certbot的所有功能。

执行完成后就自动获得了HTTPS配置，并且还能设置成HTTP自动转HTTPS，非常方便，好喜欢哦。

1）获取certbot-auto脚本

```
wget https://dl.eff.org/certbot-auto
```

2）使`certbot-auto`脚本可执行

```
chmod a+x ./certbot-auto
```

接下来有两种方法申请证书，一个是申请单个域名证书，一个适合申请通配符域名证书。

### 单个域名证书

先配置好80端口的ngnix站点

例如，假设为`api.enzhico.net`快速配置一个最简单的nginx站点

```
mkdir /opt/www/api.enzhico.net -p
chown -R nginx:nginx /opt/www/api.enzhico.net/
vi /etc/nginx/conf.d/api.enzhico.net.conf
```

将以下内容复制到该文件中:

```
server {
    listen 80;
    server_name api.enzhico.net;
    charset utf-8;
    
    root /opt/www/api.enzhico.net;
    index index.html index.htm;
    
    access_log  /var/log/nginx/api.enzhico.net_access.log;
    error_log  /var/log/nginx/api.enzhico.net_error.log;
}
```

启动nginx服务，然后访问`http://api.enzhico.net`，没有错误的话nginx站点配置完成。

运行`certbot-auto`，安装Certbot

```
./certbot-auto certonly --email yidao620@gmail.com --domains api.enzhico.net
```

执行过程中可能会告警：

```
Failed to find executable apachectl in PATH...
```

这种告警信息意识是没有按照apache软件，可以忽略，不过你要是不像看到这种告警，也可以按照以下相应的依赖：

```
sudo yum install httpd mod_ssl
```

如果你想同时为多个domain申请证书，上面的`--domains`
参数后面接逗号隔开的域名，比如`--domains api.enzhico.net,dev.enzhico.net`

联系人email地址要填写真实有效的，`let's encrypt`会在证书在过期以前发送预告的通知邮件。申请成功后，会显示以下Congratulations信息

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/admtest.enzhico.cn/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/admtest.enzhico.cn/privkey.pem
   Your cert will expire on 2018-03-28. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```

证书的保存位置在：`/etc/letsencrypt/live/api.enzhico.net/`

### 通配符证书

现在Lets Encrypt 推出通配符证书，简直爽到不要不要的。接下来我将演示申请通配符证书方法，以我自己的域名`xncoding.com`为例子

运行`certbot-auto`，安装Certbot

```
./certbot-auto --server https://acme-v02.api.letsencrypt.org/directory -d "*.xncoding.com" -d "xncoding.com" --manual --preferred-challenges dns-01 certonly
```

联系人email地址要填写真实有效的，`let's encrypt`会在证书在过期以前发送预告的通知邮件。

注意，申请通配符证书是要经过DNS认证的，按照提示，前往域名后台添加对应的DNS TXT记录。

这里我把主域名也添加上了，所以需要增加两个DNS TXT记录，注意，是增加不是修改。DNS后台就有两条TXT记录了。

添加之后，不要心急着按回车，先执行`dig _acme-challenge.xncoding.com txt`确认解析记录是否生效，生效之后再回去按回车确认。

申请成功后，会显示以下Congratulations信息

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/xncoding.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/xncoding.com/privkey.pem
   Your cert will expire on 2018-06-24. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

证书的保存位置在：`/etc/letsencrypt/live/xncoding.com/`

### 查看证书有效期的命令

````
openssl x509 -noout -dates -in /etc/letsencrypt/live/xncoding.com/cert.pem
````

### 设置定时任务自动更新证书

letsencrypt证书的有效期是90天，但是可以用脚本去更新。

```
# 如果不需要返回的信息，可以用静默方式：
./certbot-auto renew --no-self-upgrade --quiet
```

注意：更新证书时候网站必须是能访问到的，也就是说http的80端口的root目录必须配置好。

也可以通过crontab定期更新证书，先查看crond服务是否开启：

```
systemctl status crond
```

续期命令检测到续期时间小于30天时，会重新请求生成新证书，官方建议每天凌晨执行一次。

将`certbot-auto`复制到`/usr/bin/`目录中，然后编辑`/etc/crontab`，后面添加如下内容：

```
# 每天凌晨3点执行执行一次更新，并重启nginx服务器
00 03 * * * root /usr/bin/certbot-auto renew --no-self-upgrade --quiet --renew-hook "/bin/systemctl restart nginx"
# 每天凌晨3点执行执行一次更新，并重启nginx服务器
00 03 * * * root /usr/bin/certbot-auto renew --no-self-upgrade --quiet --renew-hook "/etc/init.d/nginx restart"
# 每天凌晨3点执行执行一次更新，并重启nginx服务器
00 03 * * * root /usr/bin/certbot-auto renew --no-self-upgrade --quiet --renew-hook "/usr/local/nginx/sbin/nginx -s reload"
```

### 配置nginx使用证书开通https站点

生成`Perfect Forward Security（PFS）`键值

```
mkdir /etc/ssl/private/ -p
cd /etc/ssl/private/
openssl dhparam 2048 -out dhparam.pem
```

`Perfect Forward Security（PFS)`是个什么东西，中文翻译成完美前向保密，一两句话也说不清楚，
反正是这几年才提倡的加强安全性的技术。如果本地还没有生成这个键值，需要先执行生成的命令。
生成的过程还挺花时间的，喝杯咖啡歇会儿吧。

配置nginx站点，例如`/etc/nginx/conf.d/api.enzhico.net.conf`，样例内容如下：

```
server {
    listen 80;
    server_name api.enzhico.net;
    rewrite ^ https://$server_name$request_uri? permanent;
}
server {
    listen 443 ssl;
    server_name api.enzhico.net;
    
    charset utf-8;
    root /opt/www/api.enzhico.net;
    index index.html index.htm;
    
    access_log  /var/log/nginx/api.enzhico.net_access.log;
    error_log  /var/log/nginx/api.enzhico.net_error.log;
    
    # letsencrypt生成的文件
    ssl_certificate /etc/letsencrypt/live/api.enzhico.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.enzhico.net/privkey.pem;
    
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets on;
    
    ssl_dhparam /etc/ssl/private/dhparam.pem;
    
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    # 一般推荐使用的ssl_ciphers值: https://wiki.mozilla.org/Security/Server_Side_TLS
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128:AES256:AES:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK';
    ssl_prefer_server_ciphers on;
}
```

### 大功告成

在浏览器打开`http://api.enzhico.net`, 如果正常跳转到`https://api.enzhico.net`，就算成功了。
如果是chrome浏览器，在地址栏点击小锁的图标，可以查看证书的详情。

## 国内SSL证书

除了`Let's Encrypt`外，国内也有很多SLL证书产商，比如腾讯云、阿里云、七牛之类的，申请的过程也很简单。
比如七牛的SSL证书就有免费版本的，只需要申请后填写资料完善后，一天就能申请通过了。

七牛SSL证书申请地址：<https://developer.qiniu.com/ssl>

## FAQ

### ssl_certificate没有配置

今天为线上Nginx更换SSL证书的过程中遇到了一个诡异的错误，折腾了挺长时间，事后回想起来还是挺容易跳坑的，
所以想记录下来，一是留作以后查看，二来也许可以帮助一帮跳坑的小伙伴。

nginx错误日志如下：

```
[error] 24267#0: *34 no "ssl_certificate" is defined in server listening on SSL port while SSL handshaking,
[error] 24267#0: *35 no "ssl_certificate" is defined in server listening on SSL port while SSL handshaking,
[error] 24267#0: *36 no "ssl_certificate" is defined in server listening on SSL port while SSL handshaking,
[error] 24267#0: *37 no "ssl_certificate" is defined in server listening on SSL port while SSL handshaking,
```

看错误提示意思是ssl_certificate没有配置，可是检查nginx配置，ssl_certificate和ssl_certificate_key两个选项明明都已经配置了啊，
再检查配置的证书路径、权限也都没有任何问题，真是百思不得其解啊。

最后在网上搜了一通，终于在StackOverflow一个问题找到了答案（问题地址懒得找了，就不贴了哈），
大概意思是说ssl_certificate必须在http段中先定义， 在server段才配置ssl_certificate已经来不及了，
检查我的nginx配置，ssl_certificate确实只在server段定义，而在http段未定义，加到http段即可。

### 找不到libpython2.7.so.1.0

```
locate libpython2.7.so.1.0
```

In my case, it show out put:

```
/opt/rh/python27/root/usr/lib64/libpython2.7.so.1.0
```

vi编辑/etc/ld.so.conf，最后面加一行：

```
/opt/rh/python27/root/usr/lib64
```

然后运行：

```
/sbin/ldconfig
```

## 参考

* [ssl 证书的三种类型](https://siwei.me/blog/posts/ssl-https-http-7)
* [centos7生成自签名证书](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-on-centos-7)
* [CentOS7下Nginx安装使用Let’s Encrypt](https://ruby-china.org/topics/31942)
