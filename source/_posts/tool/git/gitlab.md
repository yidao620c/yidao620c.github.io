---
title: centos7安装gitlab8.8
date: 2016-09-09 22:22:22 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

内部需要搭建一个源码管理控制环境，选择开源的gitlab，环境为centos7。这个平台类似于github，使用起来非常方便。
现在将搭建的步骤记录下来，因为官网上面提供的是ubuntu的流程。

全部命令都是在 root 用户下执行。安装操作系统 (CentOS 7 Minimal)，先配置好网卡和DNS，保证网络没问题。

安装和添加基础工具

```bash
yum install wget
```

安装EPEL源

```bash
yum install epel-release
wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6 https://www.fedoraproject.org/static/0608B895.txt
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
```

<!-- more -->

添加RemiRPM仓库

```bash
wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-remi http://rpms.famillecollet.com/RPM-GPG-KEY-remi
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-remi
```

确认是否添加成功

```
rpm -qa gpg*
# 查看输出是否包含
# gpg-pubkey-00f97f56-467e318a
```

然后安装，这样就可以添加remi-safe仓库了

```
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

查看是否添加进去了

```
yum repolist
```

如果能看到输出，那么使用

```
yum -y install yum-utils
yum-config-manager --enable epel --enable PUIAS_6_computational
```

最后，更新源缓存

```
yum clean all && yum makecache
```

安装必要的工具

```bash
yum -y update
yum -y groupinstall 'Development Tools'
yum -y install readline readline-devel ncurses-devel gdbm-devel glibc-devel tcl-devel openssl-devel curl-devel expat-devel db4-devel byacc sqlite-devel libyaml libyaml-devel libffi libffi-devel libxml2 libxml2-devel libxslt libxslt-devel libicu libicu-devel system-config-firewall-tui redis sudo wget crontabs logwatch logrotate perl-Time-HiRes git cmake libcom_err-devel.i686 libcom_err-devel.x86_64 ntp nodejs python-docutils
```

gitlab 8.0 之后的版本需要依赖 nodejs，不然安装 gitlab-shell 的时候会出现没有javascript runtime

安装vim

```bash
yum -y install vim-enhanced
update-alternatives --set editor /usr/bin/vim.basic
```

设置NTP时间同步

```
# 删除当前默认时区
rm -rf /etc/localtime
# 复制替换默认时区为上海
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
yum install -y ntp        # 安装时间同步服务（组件）
ntpdate us.pool.ntp.org   # 设置同步服务器
date                      # 查看当前时间
```

配置默认编辑器

```bash
ln -s /usr/bin/vim /usr/bin/editor
```

从源代码安装 Git

```bash
yum install -y zlib-devel perl-CPAN gettext curl-devel expat-devel gettext-devel openssl-devel
yum -y remove git
```

下载解压：

```bash
mkdir /tmp/git && cd /tmp/git
curl --progress https://www.kernel.org/pub/software/scm/git/git-2.8.4.tar.gz | tar xz
cd git-2.8.4/
./configure
make
make prefix=/usr/local install
```

将git加入到PATH中

```
which git
vi /etc/profile
# 最后 export PATH=$PATH:/usr/local/bin/git
source /etc/profile
```

Note: 编辑config/gitlab.yml (step 6), bin_path改成/usr/local/bin/git.

## 安装Ruby和Go

GitLab 需要 2.1 以上版本的 Ruby，但是当前不兼容 2.2 和 2.3，先删除旧的

```bash
yum remove -y ruby
```

如果没有安装ruby，上述删除的步骤可以跳过

下载安装

```
mkdir /tmp/ruby && cd /tmp/ruby
curl --progress ftp://ftp.ruby-lang.org/pub/ruby/2.1/ruby-2.1.8.tar.gz | tar xz
tar -xzf ruby-2.1.8.tar.gz
cd ruby-2.1.8
./configure --disable-install-rdoc
make
make prefix=/usr/local install
```

安装Bundler Gem:

```bash
gem sources --remove https://rubygems.org/
gem sources -a https://ruby.taobao.org/
gem sources -l
gem install bundler --no-doc
```

同样将`which ruby`加入到PATH中，和前面git类似

```
which ruby
# /usr/local/bin/ruby
ruby -v
```

*安装 go （gitlab 8.0 以后的版本需要go语言的支持）*

```bash
mkdir /tmp/go && cd /tmp/go
URL='https://storage.googleapis.com/golang/' && wget -c `curl -s $URL|xmllint --format - |awk -PF'[><]' '{if ($3~/linux/ && $3!~/(beta|rc)[0-9]+|armv6l|386/)a=$3}END{print "'$URL'"a}'`
# 上面下载的是个md5验证，可以自己先wget压缩包后手动解压缩安装
tar xf go*.linux-amd64.tar.gz -C /usr/local/
echo "PATH=/usr/local/go/bin:\$PATH" >/etc/profile.d/go.sh
. /etc/profile.d/go.sh
go version
```

## 系统用户

为gitlab创建一个linux系统用户git

```bash
adduser --system --shell /bin/bash --comment 'GitLab' --create-home --home-dir /home/git/ git
visudo
# Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
# 改为
# Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
```

## 数据库

CentOS 7 版本将 MySQL 数据库软件从默认的程序列表中移除，用 Mariadb 代替了

```bash
yum install -y mariadb mariadb-server mariadb-devel
systemctl enable mariadb  #设置开机启动
systemctl start mariadb #启动MariaDB
```

mariadb数据库的相关命令是：

```bash
systemctl start mariadb  #启动MariaDB
systemctl stop mariadb  #停止MariaDB
systemctl restart mariadb  #重启MariaDB
systemctl enable mariadb  #设置开机启动

```

确保你的版本是5.5.14 或以上

```bash
mysql --version
```

设置数据库 root 用户密码，并设置相关的安全配置

```bash
mysql_secure_installation
```

因为是刚安装完数据库，因此没有 root 账户的密码，按回车后，会开始让设置密码。
设置完密码后，会问是否删除匿名用户（不需要密码就能登录），选择 y

如果失败了，就使用另外方法

```bash
stop the mysql daemon.
mysqld_safe --skip-grant-tables
mysql --user=root mysql
update user set Password=PASSWORD('new-password') where user='root';
flush privileges;
restart mysql
```

然后`mysql -u root -p`登陆，使用刚刚的密码，为gitlab创建一个数据库用户git：

```bash
CREATE USER 'git'@'localhost' IDENTIFIED BY 'git';
SET storage_engine=INNODB;
# 创建生成环境数据库
CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
# 授权
GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'git'@'localhost';
\q
# 试着连连
sudo -u git -H mysql -u git -p -D gitlabhq_production
\q
```

配置 `MySQL max_allowed_packet` 的大小，避免POST太大的内容导致出现 500 错误，
例如 GitLab 发出 `MergeRequest` 的时候返回 500 错误。`vim /etc/my.cnf`，
在 `mysqld` 中添加 `max_allowed_packet`，调整值，加大为一个合适的数字即可。

```
[mysqld]
max_allowed_packet=512M
```

然后重启：systemctl restart mariadb

## Redis

开机就启动redis

```bash
chkconfig redis on
```

配置socket

```bash
cp /etc/redis.conf /etc/redis.conf.orig
# 禁止redis监听tcp请求，port设置成0
sed 's/^port .*/port 0/' /etc/redis.conf.orig | sudo tee /etc/redis.conf
echo 'unixsocket /var/run/redis/redis.sock' | sudo tee -a /etc/redis.conf
echo -e 'unixsocketperm 0775' | sudo tee -a /etc/redis.conf
service redis restart
usermod -aG redis git
```

## 安装GitLab

我们把gitlab安装到git用户的home目录

```bash
cd /home/git
# 先切换到git账户
su - git
# 再clone代码
git clone https://github.com/gitlabhq/gitlabhq.git -b 8-8-stable gitlab
# Go to GitLab installation folder
cd /home/git/gitlab
vi /etc/sudoers
# 最后添加：git ALL=(ALL)  NOPASSWD: ALL
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git -H vim config/gitlab.yml

```

配置如下信息

```
host: 192.168.217.161
port: 8008
email_from: xiongneng@winhong.com
default_theme: 1
```

确保 GitLab 可以写 log/ and tmp/ 目录，配置其他文件夹权限

```
chown -R git log/
chown -R git tmp/
chmod -R u+rwX log/
chmod -R u+rwX tmp/
sudo -u git -H mkdir /home/git/gitlab-satellites
chmod u+rwx,g=rx,o-rwx /home/git/gitlab-satellites
chmod -R u+rwX tmp/pids/
chmod -R u+rwX tmp/sockets/
sudo -u git mkdir /home/git/gitlab/public/uploads
chmod -R u+rwX  /home/git/gitlab/public/uploads
```

Unicorn 配置：

```bash
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
nproc  # 看看有几个核心
# Enable cluster mode if you expect to have a high load instance
# Ex. change amount of workers to 3 for 2GB RAM server
# Set the number of workers to at least the number of cores
# listen "0.0.0.0:8008", :tcp_nopush => true
sudo -u git -H vim config/unicorn.rb

```

复制Rack attack 配置

```
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
```

然后配置git全局配置

```bash
sudo -u git -H git config --global user.name "GitLab"
sudo -u git -H git config --global user.email "xiongneng@winhong.com"
sudo -u git -H git config --global core.autocrlf input

# Configure Redis connection settings
sudo -u git -H cp config/resque.yml.example config/resque.yml

```

如果不使用Redis的默认端口，则需要配置

```
sudo -u git -H vim config/resque.yml
```

注意：同时你还需要配置gitlab.yml和unicorn.rb

配置一下权限，如果这里repositories还不存在，那么等后面再来执行：

```
sudo chmod 700 /home/git/gitlab/public/uploads
sudo chmod -R ug+rwX,o-rwx /home/git/repositories/
sudo chmod -R ug-s /home/git/repositories/
sudo find /home/git/repositories/ -type d -print0 | sudo xargs -0 chmod g+s
```

配置gitlab数据库

```
sudo -u git cp config/database.yml.mysql config/database.yml
```

配置数据库的密码，修改为正确的用户名和密码，分别修改git用户和root用户

```
sudo -u git -H vim config/database.yml
sudo -u git -H chmod o-rwx config/database.yml
```

*安装Gems*

```
cd /home/git/gitlab
# 修改为淘宝的ruby源
vi Gemfile
# 修改为：source "https://ruby.taobao.org/"
# 下面这一步的时间会等很久
sudo -u git -H bundle install --deployment --without development test postgres aws
```

*安装GitLab shell*

```
su - git
cd /home/git/gitlab
ln -s /usr/local/bin/git /usr/bin/git
bundle exec rake gitlab:shell:install[v$(cat GITLAB_SHELL_VERSION)] REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production
logout

# 查看下gitlab-shell配置
sudo -u git -H editor /home/git/gitlab-shell/config.yml
```

*安装 gitlab-workhorse*

```
su - git
cd /home/git/
git clone https://gitlab.com/gitlab-org/gitlab-workhorse.git
logout

cd /home/git/gitlab-workhorse
ln -s /usr/local/go/bin/go /usr/bin/go
sudo -u git -H make
```

初始化数据并且激活高级特性

```
cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
```

按yes来创建数据库，创建后管理员用户也会创建了，并且默认密码是空，第一次打开会让你新建密码。输出类似下面：

```
Administrator account created:

login:    root
password: You'll be prompted to create one on your first visit.
```

*安装初始化脚本*

```
sudo cp /home/git/gitlab/lib/support/init.d/gitlab /etc/init.d/gitlab
chmod +x /etc/init.d/gitlab
chkconfig --add gitlab
chkconfig gitlab on
cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
```

*检查应用状态*

如果前面的gitlab正确配置好了，那么现在可以查看它的状态了：

```
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
```

编译静态文件

```
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
```

*启动gitlab服务*

```
service gitlab start
```

## 配置web服务器

官方推荐使用nginx服务器

```
yum update
yum -y install nginx
chkconfig nginx on
```

使用SSl

```
cp /home/git/gitlab/lib/support/nginx/gitlab-ssl /etc/nginx/conf.d/gitlab.conf
```

不使用SSL

```
cp /home/git/gitlab/lib/support/nginx/gitlab /etc/nginx/conf.d/gitlab.conf
```

我这里不使用SSL，编辑vim /etc/nginx/conf.d/gitlab.conf，将git.example.com替换成你的地址

```
listen 0.0.0.0:80;
##listen [::]:80 default_server;
server_name 192.168.217.161; ## Replace
server_tokens off; ## Don't show
location / {
    client_max_body_size 512m;
```

另外还要加个gitlab的upstream，里面默认就一个gitlab-workhorse：

```
upstream gitlab {
  server unix:/home/git/gitlab/tmp/sockets/gitlab.socket fail_timeout=0;
}

upstream gitlab-workhorse {
  server unix:/home/git/gitlab/tmp/sockets/gitlab-workhorse.socket fail_timeout=0;
}
```

测试配置：

```
nginx -t
```

修改权限

```
usermod -a -G git nginx
chmod g+rx /home/git/
```

去修改`/etc/selinux/conf`，将selinux关闭，否则会出现 nginx 访问错误 (13: Permission denied)，HTTP显示502

将`SELINUX=enforcing` 改为 `SELINUX=disabled`,重启机器即可

`setenforce 0` 只是临时关闭，重启后问题仍然出现

然后执行：

```
service nginx start
shutdown -r now
service gitlab restart
```

## 配置防火墙

直接禁用防火墙最简单

```
systemctl stop firewalld
systemctl mask firewalld
```

设置开放端口到服务器外访问，这里以8008为例子，需要替换为真实的端口

```
/sbin/iptables -I INPUT -p tcp --dport 8008 -j ACCEPT
service iptables restart
```

## 最后检查

为了确保所有流程都正确完成，执行下面的检查：

```bash
cd /home/git/gitlab
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
```

如果都是绿色的，表示安装成功。
如果出现类似表找不到的错误，那么就是前面的数据库初始化没成功，再执行一次：

```
sudo -u gitlab -H bundle exec rake db:migrate RAILS_ENV=production
```

然后浏览器直接访问：<http://192.168.217.161/>

## Gitlab 备份

官网的备份说明

<https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md>

查看备份设置

```
vim /home/git/gitlab/config/gitlab.yml
```

检查Backup Settings设置项。
默认情况下，备份文件是存放在/home/git/gitlab/tmp/backups/

执行备份

```
sudo service gitlab stop # 先停止Gitlab，可以不暂停
cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
```

执行完成后，会在/home/git/gitlab/tmp/backups/目录下创建一个备份俄文件，
以时间戳_gitlab_backup命名如 1417040627_gitlab_backup.tar

重新启动

```
sudo service gitlab start
sudo service nginx restart
```

还原

```
chmod o+wrx /home/git/.ssh/authorized_keys.lock
# 需要使用 git 用户来执行，否则会没有权限操作 git 目录下的文件，timestamp_of_backup为时间戳如 1417040627
sudo service gitlab stop
cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:backup:restore BACKUP=timestamp_of_backup RAILS_ENV=production
```

如果是从全新部署的 gitlab 还原，需要执行这一步

```
sudo -u git -H bundle exec rake gitlab:satellites:create RAILS_ENV=production
```

重启 gitlab

```
sudo service gitlab start
sudo service nginx restart

sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
```

设置自动备份

```
sudo service gitlab stop;
cd /home/git/gitlab;
sudo -u git -H editor config/gitlab.yml;
# Enable keep_time in the backup section to automatically delete old backups
```

keep_time参数默认是604800（单位是秒），因此会保留最近7天内的备份

```
sudo -u git crontab -e # Edit the crontab for the git user
```

将如下内容添加到文件末尾

```
# 每天凌晨2点自动备份
0 2 * * * cd /home/git/gitlab && PATH=/usr/local/bin:/usr/bin:/bin bundle exec rake gitlab:backup:create RAILS_ENV=production CRON=1
```

重新启动

```
sudo service gitlab start;
sudo service nginx restart;
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production;
```

## 忘记管理员密码

Gitlab 服务器上使用

```
# Gitlab 安装路径
cd /home/git/gitlab
# 进入Rails控制台
sudo -u git -H bundle exec rails console production
# 进入控制台，如果知道需要修改用户的邮箱，使用如下，直接修改
user = User.find_by(email: 'admin@example.com')
user.password = 'secret_password'
user.password_confirmation = 'secret_password'
user.save
# 如果不知道具体邮箱，可以通过find来查找邮箱
user = User.find(1)
```

## 中文汉化

如果不习惯英文界面，可以汉化

克隆 GitLab.com 仓库git clone

```
git clone https://gitlab.com/larryli/gitlab.git gitlab_cn
```

确认版本号，安装对应版本的汉化包：

```
cat /home/git/gitlab/VERSION
```

进入前面用git拉取的目录gitlab_cn

```
cd gitlab_cn
```

先停止gitlab

```
service gitlab stop
```

8.8 版本的汉化补丁（8-8-stable是英文稳定版，8-8-zh是中文版，两个 diff 结果便是汉化补丁）

```
git diff origin/8-8-stable origin/8-8-zh > /tmp/8.8.diff
```

应用汉化补丁

```
cd /home/git/gitlab/
git apply /tmp/8.8.diff
```

启动gitlab

```
service gitlab start
```

## 简单使用

这里我使用的是SSH协议，需要复制你的ssh-key到gitlab上面。

**客户端生成ssh-key**

如果已经有sshkey，可用之前的。

在客户端执行命令 ssh-keygen -t rsa -C "for xxx"，-C 选项后是备注，可随意。

命令执行后会要求输入key存储的文件名和passphrase：

输入一个特有的文件名，否则使用默认的 id_rsa。
passphrase。不输入也可以。输入之后，提交的时候要输入这个passphrase
完成后在 ~/.ssh/ 会生成2个文件。id_rsa 和 id_rsa.pub。前者是私钥，注意保管，后者是公钥。

**添加ssh-key到gitlab**

登录之后: Profile Settings => SSH-Keys => Add SSH key。

输入之前生成的公钥，标题随意。

## FAQ

1. 内网请使用SSH协议的地址来pull/push，profile settings中SSH Keys添加相应的pubkey
2. 公网的话就使用https协议了，这个暂时还没研究
3. win7上面push的时候报了一个`libcurl-4.dll`找不到，去网上下载后放到`C:\Windows\SysWOW64\`下面

## 参考文档

* <https://gitlab.com/help/install/installation.md>
* <https://gitlab.com/gitlab-org/gitlab-recipes/tree/master/install/centos>

