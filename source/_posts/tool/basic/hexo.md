---
title: 使用hexo搭建github博客
date: 2016-03-06 19:47:53
toc: true
categories: [ 开发工具 ]
tags: [ hexo ]
---

最今天我又折腾了我的博客，将它从octopress迁移到hexo上来。之前还专门写了一篇怎样利用octopress搭建博客的文章，
最近试用了一下hexo，毫不犹豫的迁移过来了，实在是忍受不了octopress的速度，还有稳定性，经常莫名其妙的出错。

hexo是一个台湾人做的基于Node.js的静态博客程序，优势是生成静态文件的速度非常快，支持markdown，
我最终选定它的原因是它速度快而且不容易出错，并且可以一键部署到github或者其它静态服务器上去。折腾了一天总算搞定。
<!-- more -->

## 安装

我这个教程hexo3版本，本地使用Windows7系统，在IDEA上面写Markdown的博客，爽歪歪。

### 安装依赖软件

首先安装Nodejs环境，参考我的博客《Nodejs的安装和配置》即可。

安装[Git客户端](http://git-scm.com/download/win)，用于把本地的hexo内容提交到github上去

### 安装hexo

利用 npm 命令即可安装。打开窗口控制台，输入安装hexo命令：

```bash
cnpm install -g hexo
```

### 初始化

安装完成后，在你喜爱的文件夹下（如D:\hexo），控制台执行以下指令，hexo会自动在目标文件夹建立网站所需要的所有文件。

```bash
hexo init
```

安装依赖包：

```bash
cnpm install
```

一切准备就绪让我们试验下，在D:\hexo内执行以下命令：

```bash
hexo g
hexo s
```

然后用浏览器访问http://localhost:4000， 此时，你应该看到了一个漂亮的博客了，当然这个博客只是在本地的，别人是看不到的，hexo3.0使用的默认主题是landscape。

## 发布到github上面

首先你得有github账号，如果没有就去注册个，很简单的步骤。

### 创建repository

repository相当于一个仓库，用来放置你的代码文件。登陆进入Github，并进入个人页面，选择repositories，然后New一个repository。
创建时，只需要填写Repository name即可。格式必须为yourname.github.io，比如我的是yidao620c.github.io

### 将本地博客部署上去

先修改D:\hexo下的_config.yml文件,记得一点，hexo的配置文件中任何`:`后面都是带一个空格的

```yaml
deploy:
  type: git
  repository: https://github.com/yidao620c/yidao620c.github.io.git
  branch: master
```

如果你是第一次使用Github或者是已经使用过，但没有配置过SSH，则可能需要配置一下SSH。
在Git Bash输入以下指令（任意位置点击鼠标右键），检查是否已经存在了SSH keys。

```bash
ls -al ~/.ssh
```

如果不存在就没有关系，如果存在的话，直接删除.ssh文件夹里面所有文件

输入以下指令（邮箱就是你注册Github时候的邮箱）后回车，出现提示让你输入的时候一直按回车：

```bash
ssh-keygen -t rsa -C "yidao620@gmail.com"
```

然后键入以下指令：

```bash
eval `ssh-agent -s`
ssh-add
```

到了这一步，就可以添加SSH key到你的Github账户了。键入以下指令，拷贝Key（先拷贝了，等一下可以直接粘贴，不放心的在执行下面命令后，先黏贴在记事本上）：

```bash
clip < ~/.ssh/id_rsa.pub
```

然后到Github里面，点击右上角的设置图标Settings,找到SSH keys,Ttile随便你命名，Key就黏贴上你刚才复制的key,然后点Add SSH
key，最后会让你重新输入下gitHub的密码
最后还是测试一下吧，键入以下命令：

```bash
ssh -T git@github.com
```

你可能会看到有警告，输入"yes"就好

还要安装hexo-deployer-git这个模块:

```bash
npm install hexo-deployer-git --save
```

以上就表示SSH配置好了，执行以下命令部署到Github上。

```bash
hexo g
hexo d
```

提示输入gitHub的账号密码，就能访问你得博客网站了。我的是: yidao620c.github.io

## 发布到自己的VPS

如果你觉得github不是你想要的，你有一个自己的云主机，那么还可以将hexo博客自动发布到自己的服务器上去。

比如我自己有一条腾讯云主机，操作系统是CentOS7.4，我将博客迁移到这台虚拟机上去。步骤如下：

### 源码安装 git2

1,检测系统中git的版本，并卸载低版本

```bash
yum info git
yum reomve git
```

2,下载最新的git

```bash
wget -P /opt/ https://www.kernel.org/pub/software/scm/git/git-2.14.1.tar.gz
```

3,解压

```bash
tar xzvf git-2.14.1.tar.gz
```

5,安装依赖lib

```bash
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install  gcc perl-ExtUtils-MakeMaker
```

4,指定安装目录

```bash
./configure --prefix=/usr/local/git
```

6,编译安装

```bash
make && make install
```

7,配置环境变量，编辑/etc/profile

```bash
#git settings
GIT_HOME=/usr/local/git
export PATH=$GIT_HOME/bin:$PATH
```

8,配置生效

```bash
source /etc/profile
```

9,检查git是否安装成功

```bash
git --version
```

10,创建git账户, 并设置密码

```bash
useradd git
passwd git
```

11,使用git登录并进入git目录

```bash
su - git
cd ~
```

12,在/home/git目录下创建.ssh目录,并进入

```bash
mkdir .ssh && cd .ssh
```

13,创建一个存储所有登录用户的公钥(id_rsa.pub),一行一个用户

```bash
touch authorized_keys
```

14,初始化git仓库

```bash
mkdir /var/repo && cd /var/repo
git init --bare hexo.git
chown -R git:git hexo.git
```

15,禁用shell

```bash
vi /etc/passwd
git:x:1000:1000::/home/git:/usr/bin/git-shell
```

### yum安装git2

还有一种更方便的方法安装git2：

```bash
sudo yum remove git
sudo yum install epel-release
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
sudo yum install git2u
```

### 安装nginx

参考 <https://blog.csdn.net/oldguncm/article/details/78855000>

```bash
su - root
mkdir -p /opt/www/hexo
chown -R nginx:nginx /opt/www
chown -R git:git /opt/www/hexo
```

nginx.conf配置网站文件目录

```bash
root /opt/www/hexo;
```

### 配置git hooks

这步非常重要，将代码push到git仓库后会自动执行钩子脚本，将网站文件推送到nginx的root目录，就能自动更新网站了。

```bash
su - git
$ cd /var/repo/hexo.git/hooks
$ vim post-receive
```

写入如下内容：

```bash
#!/bin/bash
git --work-tree=/opt/www/hexo --git-dir=/var/repo/hexo.git checkout -f
```

至此服务器配置完成。

然后你的hexo的`_config.yml`配置文件修改如下：

```yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@xx.xx.xx.xx:/var/repo/hexo.git
  branch: master
```

## 发表一篇文章

在你的D:\hexo目录下面，控制台执行命令：

```bash
hexo new "Hello World"
```

会自动在D:\hexo\source\_posts文件夹生成一个hello-world.md文件，用编辑器打开，在里面写文章。
记住hexo中写文章使用的是Markdown语法，自己去google下这个语法吧，很方便很强大。里面的初始内容

```
title: Hello World #可以改成中文的，如"新文章"
date: 2016-03-06 16:04:09 #发表日期，一般不改动
categories: blog #文章文类
tags: [文章, 测试] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
#这里是正文，用markdown写，你可以选择写一段显示在首页的简介后，加上
<!--more-->，在<!--more-->之前的内容会显示在首页，之后的内容会被隐藏，当游客点击Read more才能看到。
```

写完文章后，你可以使用`hexo g`生成静态文件。`hexo s`在本地预览效果。`hexo d`同步到github

## hexo主题及其配置

如果你不喜欢默认主题，那么可以去hexo官网找更多的[主题](https://hexo.io/themes/)。
我这里选择NexT主题说明一下，这个是一个非常简洁的主题，我喜欢简单的东西。

Next主题官网：<http://theme-next.iissnan.com/>

### 安装主题

```bash
$ cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```

### 启用主题

修改你的博客根目录下的config.yml配置文件中的theme属性，将其设置为next

### 更新主题

```bash
cd themes/next
git pull
```

更新好后，本地启动起来效果

```bash
hexo server -g  #生成加预览
```

### 主题的_config.yml配置

具体配置请直接参考[开始使用NexT主题](http://theme-next.iissnan.com/getting-started.html)。
同时我对这个主题进行了很多的修改让它看上去更加符合自己的审美观，如果对我博客的主题感兴趣可以直接在我的github页面拉取即可。

### 畅言评论

一开始我用了多说，后来多说关闭不能使用了。然后换到disqus，发现还是有不少问题，首先是https因为不少认证过的，在浏览器里面有警告。
另外匿名评论我在管理后台看不到，不知道什么原因。最后还是切换至国内的畅言评论。

参考这篇 <http://www.jianshu.com/p/5888bd91d070>

先申请畅言，并且通过审核。获取到畅言评论的`APP ID` 和`APP KEY`，复制下来。

主题配置文件`_config.yml`中添加一个简单的配置即可：

```yaml
# changyan
changyan:
  enable: true
  appid: your_appid
  appkey: your_appkey
```

安装畅言后遇到几个问题，打开页面控制台后一直报错误，页面循环计数，另外有评论的文章，在首页评论数永远是0，在文章里面计数正常，
虽然不影响使用，但是看着真是不舒服。必须解决，参考了一篇文章（后面贴了链接）解决方案。

找到`next/layout/_third-party/comments/changyan.swig:1~4行`

```html
@% if theme.changyan.enable and theme.changyan.appid and theme.changyan.appkey %@
@% if is_home() %@
<script id="cy_cmt_num"
        src="https://changyan.sohu.com/upload/plugins/plugins.list.count.js?clientId=@@theme.changyan.appid@@"></script>
@% else %@
...省略
```
只需要将
```
@% else %@
```
改成
```
@% elseif page.comments %@
```

然后再找到`next/layout/_mavro/post.swig:175～178行`
```html
<a href="@@ url_for(post.path) @@#SOHUCS" itemprop="discussionUrl">
    <span id="url::@@ post.permalink @@" class="cy_cmt_count" data-xid="@@ post.path @@"
          itemprop="commentsCount"></span>
</a>
```

只需要将`post.permalink`这个值包一层encodeURI方法，如下修改：
```html
<span id="url::@@ encodeURI(post.permalink) @@" class="cy_cmt_count" data-xid="@@ post.path @@"
      itemprop="commentsCount"></span>
```

另外如果部署在自己服务器上面的话，你可以在带www的文章下评论完，然后去不带www的文章下看看，评论已经不存在，我做了nginx
301重定向解决。

## 添加网站的RSS订阅

### 安装hexo-generator-feed
```bash
cnpm install hexo-generator-feed --save
```

安装完后，会在node_modules目录下生成hexo-generator-feed目录

### 配置根目录的_config.yml

```yaml
# Extensions
## Plugins: http://hexo.io/plugins/
#RSS订阅
plugin:
  - hexo-generator-feed
#Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
```

其中，feed是可选项，可配可不配！然后根据各个主题的说明添加一个RSS订阅链接即可

## 自定义分页

安装三个插件：归档，标签，分类。然后将每个分页显示文章数都变大点

```yaml
npm install hexo-generator-archive --save
npm install hexo-generator-tag --save
npm install hexo-generator-category --save
```

三者各自的配置：

```yaml
archive_generator:
  per_page: 100
  yearly: true
  monthly: true
  daily: false
tag_generator:
  per_page: 100
category_generator:
  per_page: 100
```

## 绑定自己的域名
在GitHub Page中绑定自定义域名，目前看用处不大。

## 站内搜索

最简单是直接开启`Local Search`

安装 `hexo-generator-searchdb`，在站点的根目录下执行以下命令：

```bash
cnpm install hexo-generator-search --save
cnpm install hexo-generator-searchdb --save
```

全局配置文件_config.yml中定义搜索页面：

```yaml
search:
  path: search.xml
  field: post
  format: html
  limit: 7000
```

在NexT的`source/_data/next.yml`配置中开启这个本地搜索：

```yaml
# Local Search
# Dependencies: https://github.com/theme-next/hexo-generator-searchdb
local_search:
  enable: true
  # If auto, trigger search by changing input.
  # If manual, trigger search by pressing enter key or search button.
  trigger: auto
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 1
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false
```

## algolia搜索

algolia在国内访问经常很卡，所以用这个插件的时候请三思。

第一步，先注册Algolia，创建Index

前往 [Algolia 注册页面](https://www.algolia.com/)，注册一个新账户。可以使用 GitHub 或者 Google 账户直接登录，
注册后的 14 天内拥有所有功能（包括收费类别的）。之后若未续费会自动降级为免费账户，免费账户总共有 10,000 条记录，
每月有 100,000 的可以操作数。注册完成后，创建一个新的 Index，这个 Index 将在后面使用。

第二步，安装Algolia

```bash
npm install --save hexo-algolia
```

第三步，获取Key，更新站点配置

在 Algolia 服务站点上找到需要使用的一些配置的值，包括 `ApplicationID`、`Search-Only API Key`、`min API Key`。
注意，Admin API Key 需要保密保存。点击ALL API KEYS 找到新建INDEX对应的key， 编辑权限，
在弹出框中找到ACL选择勾选`Search`、`Add records`、`Delete records`、`List indices`、`Delete index`权限，点击update更新。

![](https://xnstatic-1253397658.file.myqcloud.com/hexo20.png)

编辑站点配置文件，新增以下配置：

```yaml
algolia:
  applicationID: 'IQUD4IM8MM'
  indexName: 'blogindex'
  chunkSize: 5000
```

替换除了 chunkSize 以外的其他字段为在 Algolia 获取到的值。

第四步，更新Index

当配置完成，新增一个环境变量，windwos和linux配置环境变量我就不讲了。

```
HEXO_ALGOLIA_INDEXING_KEY=Search-Only API key
```

在站点根目录下执行

```bash
$ hexo algolia
```

来更新 Index，请注意观察命令的输出。

```bash
$ hexo algolia
INFO  [Algolia] Testing HEXO_ALGOLIA_INDEXING_KEY permissions.
INFO  Start processing
INFO  [Algolia] Identified 152 pages and posts to index.
INFO  [Algolia] Indexing chunk 1 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 2 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 3 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 4 of 4 (50 items each)
INFO  [Algolia] Indexing done.
```

注意，每次更新文章后记得执行`hexo algolia`更新索引哦。

第五步，主题集成

更改主题配置文件，找到 Algolia Search 配置部分：

```yaml
# Algolia Search
algolia_search:
  enable: true
  hits:
    per_page: 10
  labels:
    input_placeholder: Search for Posts
    hits_empty: "We didn't find any results for the search: ${query}"
    hits_stats: "${hits} results found in ${time} ms"
```

将 `enable` 改为 `true` 即可，根据需要你可以调整 `labels` 中的文本。

## 阅读次数统计

使用LeanCloud为文章添加阅读次数统计，
请参考文章[第三方服务集成](https://theme-next.iissnan.com/third-party-services.html) 中的阅读次数统计（LeanCloud)部分。

另外有个安全补丁需要打上，参考<https://github.com/theme-next/hexo-leancloud-counter-security>

## 多台电脑同时维护博客

之前利用了 Hexo + Github搭建了自己的博客网站，但问题来了：如何在多台电脑上对此博客进行维护？

### 思路

* 对于一个已创建的博客目录，其已通过hexo建立了public目录下的内容与远程仓库yidao620c@github.com的master分支的连接关系，通过hexo的hexo
  deploy命令即可保持更新；
*
因此只需要在此仓库yidao620c@github.com上再创建一个分支如source，将其与hexo下其他文件如node_modules、themes、scaffolds等（即除了public目录及.gitignore包含的文件外）进行同步绑定。
* 在不同的电脑下设置好hexo环境，通过hexo命令维护master分支，通过git命令维护source分支即可。

### 具体步骤

（假定最初创建博客为A，其他另一个为B)

1) 在A中的git_blog目录下，建立source分支：

```bash
$ git branch source // 创建source分支
$ git checkout source // 切换到source分支
$ git remote -v //查看远程分支名字
 // 我的内容为如下，说明已绑定
 // origin  git@github.com:yidao620c/yidao620c.github.com.git (fetch)
 // origin	git@github.com:yidao620c/yidao620c.github.com.git (push)
修改.gitignore文件，添加"public/"字段至其中。
$ git push origin source // 将当前git_blog下的内容push到Github上的远程仓库的source分支（会自动创建）上
```

2) 在电脑B中将source分支clone下来

```bash
$ git clone -b source git@github.com:yidao620c/yidao620c.github.com.git
$ cd yidao620c.github.com
$ cnpm install hexo
$ cnpm install
```

**重要记录**

别执行`hexo init`这个命令，同时`Next`主题的安装步骤还是需要的。

在任意一台机器上面操作，都需先切换并保持在source分支上。使用git命令管理source文件；使用hexo命令进行同步至远程master分支，无需处理本地master分支

1. 在A中使用Hexo的new、g、d方法添加、生成、部署新的博客，内容都会被同步自动放到Github的master分支上
2. 在A中使用git命令的`push origin source`同步source到远程source分支
3. 同样保证B的git当前在git_blog下的source分支下

先使用：

```bash
git pull origin source
```

获得Github的source分支上的最新版本，再使用：

```bash
$ git add -u
$ git commit -m ""
$ git push origin source
```

将新内容提交至Github的source分支上，完成source管理。

## 代码高亮插件

把代码高亮禁用掉后再编译，出现` Error: template not found: blog/base.html`的异常。

这是因为代码中包含`{紧挨着{`这种双大括号引起的，特别是django模板里面大量的这样代码。然后我把所有的大括号改成了@。

后面又出现：` pandoc exited with code null`，发现所有博客头部不能有引号。我勒个去，于是把所有的引号去掉。
后来又经常出现这个错误，是因为hexo-renderer-pandoc的包的问题，直接卸载，整个世界清静了：

```bash
npm remove --save hexo-renderer-pandoc
```

参考这篇文章：<https://www.zfl9.com/hexo-code.html>

hexo 默认的代码高亮插件为 highlight.js，highlight.js 的代码高亮个人感觉不是很好看（主要是配色方面，感觉不够"高亮"），
然后偶然之间发现了一个还不错的代码高亮插件：prism.js，所以就琢磨着如何将 hexo 的 highlight.js 替换为 prism.js。

### 禁用 highlight.js

修改站点根目录下的 _config.yml 配置文件，具体如下：

```yaml
highlight:
  enable: false
  line_number: false
  auto_detect: false
  tab_replace:
```

### 获取 prism.js

下载页面：<https://prismjs.com/download.html>；选择 theme 主题、language 支持的语言（不要选太多，够用就好）、plugin 插件；
然后点击下载按钮就行了。下载到本地之后，将它们重命名为 prism.js、prism.css，
然后将它们放置到 $HEXO/themes/hexo-theme-snippet/source/js/prism/ 目录下（prism 文件夹需要自己新建）

### 配置 prism.js

1、修改 `$HEXO/themes/hexo-theme-snippet/layout/_partials/head.ejs`，在尾部添加以下代码：

```html

<link rel="stylesheet" href="/js/prism/prism.css">
```

2、修改 `$HEXO/themes/hexo-theme-snippet/layout/_partials/footer.ejs`，在尾部添加以下代码：

```html

<script src="/js/prism/clipboard.min.js"></script>
<script src="/js/prism/prism.js" async></script>
```

如果你选择了 `Copy to Clipboard Button prism.js` 插件，
则还需下载 [clipboard.js](https://cdn.bootcss.com/clipboard.js/1.7.1/clipboard.min.js)，
因为这个插件需要使用 clipboard.js 里面的函数，如果不这样做，在 Chrome 浏览器中将无法正常显示代码块，
将 clipboard.js 放到 `$HEXO/themes/hexo-theme-snippet/source/js/prism/` 目录下。

3、修改 `$HEXO/node_modules/marked/lib/marked.js`

搜索 `<pre><code class="` 关键字（应该只有一处），该行的内容为：`return '<pre><code class="'`

将这行语句改为：如果你下载的 prism.js 未选择 Line Numbers 插件，则去掉 line-numbers（注意后面还有个空格，也要去掉）：

```
return '<pre><code class="line-numbers language-'
```

4、关于代码横向滚动条一直出不来的原因

原来style.css里面设置了`::-webkit-scrollbar`，将其注释掉即可：

```css
::-webkit-scrollbar {
    -webkit-appearance: none;
    height: 5px;
    width: 5px;
}
```

## Gitalk评论

Gitalk是一个基于 Github Issue 和 Preact 开发的评论插件，详情Demo可见：https://gitalk.github.io/。
使用方法超级简单，按照官网文档配置一下就OK。<https://theme-next.org/docs/third-party-services/comments>

## 不蒜子阅读统计

```yaml
# Show Views / Visitors of the website / page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi
busuanzi_count:
  enable: true
  total_visitors: true
  total_visitors_icon: user
  total_views: true
  total_views_icon: eye
  post_views: true
  post_views_icon: eye
```

## 升级指南
直接先升级hexo-cli，然后再升级next主题。

把next最新包下载下来，建一个新文件夹，然后修改全局_config.yml指向新的next版本。
对于next的自定义配置、自定义样式放到`source\_date\`目录下面。配置就新建一个next.yml，第一行加入`override: false`。
然后将自定义的放里面，其他默认的不变，这样子后面升级就只需要全局替换原有的next主题文件夹即可，自定义配置保留。
样式的话，就新建一个`styles.styl`文件，把自己定义样式放进去即可。 另外还必须在next.yml打开自定义配置项

```yml
custom_file_path:
  style: source/_data/styles.styl
```

### 问题记录

中文目录单击不跳转问题解决方案

生成文章的中文目录点击后不会自动跳转到相应位置。在next github 上已经提出了该问题并给出了
[解决方案](https://github.com/theme-next/hexo-theme-next/pull/1540/files)

![20230624-01.png](https://xnstatic-1253397658.file.myqcloud.com/20230624-01.png)

主要修改了如下函数（next/source/js/utils.js）：

```js
registerSidebarTOC: function () {
    const navItems = document.querySelectorAll('.post-toc li');
    const sections = [...navItems].map(element => {
        var link = element.querySelector('a.nav-link');
        var target = document.getElementById(decodeURI(link.getAttribute('href')).replace('#', ''));
        // TOC item animation navigate.
        link.addEventListener('click', event => {
            event.preventDefault();
            var offset = target.getBoundingClientRect().top + window.scrollY;
            window.anime({
                targets: document.scrollingElement,
                duration: 500,
                easing: 'linear',
                scrollTop: offset + 10
            });
        });
        return target;
    });
```

## FAQ

* 遇到有大括号的代码块，如果多行的不用管，如果单行的就单个反引号，并且在里面加raw标签，比如@% raw %@ `@@test@@` @% endraw %@
* 关闭hexo的将回车当换行做法是用正常的markdown两个回车当换行，在全局_config.yml中添加配置

```yaml
marked:
  gfm: true
  breaks: false #这里表示不要自动添加换行，按照markdown语法来
  pedantic: false
  smartLists: true
  smartypants: true
  modifyAnchors: 0
  autolink: true
  sanitizeUrl: false
  headerIds: true
  prependRoot: false
  external_link:
    enable: false
    exclude: [ ]
    nofollow: false
```

* 对于行内代码引用使用单个反引号的时候，如果代码长度过长的时候，代码会自动换行，导致上面文字撑开很难看。
  解决办法是修改css文件`/themes/next/source/css/_common/scaffolding/highlight/highlight.styl`

```css
code {
    word-break: break-all;
/ / 这一行是增加的，为了防止inline代码引用过长换行后样式变差
}
```

* `<!--more-->`不能直接放到table后面，所有table后面都应该空一行。

## 参考

* [hexo干货系列](http://tengj.top/tags/hexo/)
* [大道至简——Hexo简洁主题推荐](https://www.haomwei.com/technology/maupassant-hexo.html)
* [NEXT主题中使用畅言的坑](http://www.chaixuhong.com/article/NEXT%E4%BD%BF%E7%94%A8%E7%95%85%E8%A8%80%E7%9A%84%E9%97%AE%E9%A2%98.html)

