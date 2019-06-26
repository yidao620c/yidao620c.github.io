---
layout: post
title: 使用Octopress搭建GitHub博客
date: '2015-03-18 17:41:21 +0800'
comments: true
toc: true
categories: 朝花夕拾
tags:
  - octopress
abbrlink: 7370
---

经过iteye博客——WordPress——github的三重进化，时间来到了2015年，
荒废了两年的Blog大业再次东山再起，于是经过一天的配置，本博客终于展现在你的眼前。
本博客基于Octopress+Github Pages搭建，可以搭建全免费、稳定运行的个人博客。
本文将简述在Windows平台（Win7 64bit）上搭建这套系统的全流程和要点。
在继续看本文之前，请先确定你具有折腾的精神。<!--more-->

### 介绍
[Octopress](https://github.com/imathis/octopress)是一个基于Ruby语言的开源静态网站框架，
所谓静态，是指网站的所有内容都是生成好的静态HTML，不含任何后台处理程序，也没有数据库。
这样的好处是网站的加载速度会非常快，整体程序规模也非常轻量级。

[Github Pages](https://pages.github.com/)是Github上的一项服务，
注册用户可以申请一个和自己账号关联的二级域名，
在上面可以托管一个静态网站，网站内容本身就是Github的一个repository也就是项目，
维护这个项目的代码就是在维护自己的网站。

此外，用户撰写日志使用的是Markdown语法。这是一种极简化的语法，
它的好处在于可以以纯文本形式表现文章，用户不用关心排版的问题。
基本上来说它相当于HTML标签的最小子集做了一个转义。

综上所述，使用本文所述的配置环境，你将得到一个如下的个人网站：

* 完全免费的空间和域名
* 在本地撰写日志和管理网站，不用考虑备份问题
* 无需管理数据库
* 满足你的折腾欲

### 安装

#### Ruby环境的安装

Octopress是基于Ruby的，我们需要安装一个Ruby环境和一个开发包。

首先我们[下载Ruby](http://rubyinstaller.org/downloads/)环境。这里要注意一下版本，
请选择和当前Ocotopress官网的建议相同的版本。
本文目前要下载的是[Ruby 1.93][]。
安装时注意选中“Add Ruby executables to your PATH”的选项，
将Ruby程序加入环境变量，这样才能在Windows任意位置使用cmd运行Ruby命令。

安装结束后，可以在cmd输入如下命令检测ruby是否安装成功：
```
ruby --version
```

下面需要安装[DevKit][]。DevKit是专在Windows上为了方便使用Ruby的一个开发包，
在它主页的Requirements一段有写根据你的Ruby版本，对应的DevKits版本也不同。
本文需下载[DevKit 4.5.2][]版本。

下载解压到没有空格的路径，然后在此路径打开cmd，执行以下命令：

```
ruby dk.rb init
ruby dk.rb install
```

#### Octopress的安装
进入[OCtopress页面](https://github.com/imathis/octopress)，点击右边的Downloadzip。
会用Github的玩家可以直接git clone，不多叙述。
下载下来之后解压，把目录名字改成octopress。

下面我们先做个预备操作：更改gem的更新源。
由于后续安装需要gem自己从服务器上拿资源，默认的服务器地址速度很慢，因此我们改成淘宝的源。

```
gem sources -a http://ruby.taobao.org/
gem sources -r http://rubygems.org/
gem sources -l
```
三行命令的作用分别是：添加淘宝源；删除默认源；显示当前源列表。显示淘宝地址就表示成功。

然后，我们还要改一下Octopress的gem更新源

在octopress的目录下，打开Gemfile文件，
查找source "http://rubygems.org/"改为source "http://ruby.taobao.org/"

好了下面我们进入本节的最后一步：安装Octopress依赖。在Octopress的目录下执行命令：
```
gem install bundler
bundler install
```
这一步需要等待一段时间，耐心等待出现Complete以后，执行以下命令安装Octopress的默认主题：
```
rake install
```

#### 测试
现在我们本地已经环境全部安装完毕了，你可以在自己的电脑上开始耍了。

由于网站是静态的，所以每次改动（修改网页，写日志等）之后需要使用以下命令生成页面：
```
rake generate
```
生成完毕后，使用以下命令启动网站进程，默认占用4000端口：
```
rake preview
```
注意这个命令运行之后当前窗口会一直以阻塞模式监听http请求和显示状态，所以要输入新命令请新开一个cmd。

之后在浏览器输入http://localhost:4000就可以看到网站的界面了。

### 配置

#### 基础配置

网站的通用配置文件是根目录下的_config.yml，内有注释已经将配置分成几个部分。
我会把我改动过的部分列出来，其余的部分略过，可以自己研究。

**1) 主配置Main Configs**
```
url: http://yidao620c.github.io
title: 笨跑的一刀
subtitle: Code is sexy, like Amy
author: 熊能
simple_search: https://www.google.com.hk/search
description:
date_format: "%Y年%m月%d日"
```

**2) 第三方插件 3rd Party Settings**

这里设置第三方插件的设定，譬如Github个人资料、Twitter、G+、Pinboard、facebook等。
总之基本上要打开就改成true否则就false，要填用户名就填就可以了。

由于是纯静态网站，因此网站评论必须使用第三方插件。我选择了国内的一个同类产品——多说（其实应该是山寨Disqus的）。

**多说插件配置**

这里详细讲解下多说这个插件的配置。其实很简单

1) 申请多说账号

去<http://duoshuo.com/>上面申请一个账号，一般都可以使用QQ等社交账号来绑定。
然后去管理后台添加一个新站点，设置下域名，获取你的short_name，就是你的二级域名前缀。
比如`shortname.duoshuo.com`。

2) 配置_config.yml

```
# duoshuo comments
duoshuo_comments: true
duoshuo_short_name: shortname
duoshuo_asides_num: 10      # 侧边栏评论显示条目数
duoshuo_asides_avatars: 1   # 侧边栏评论是否显示头像
duoshuo_asides_time: 1      # 侧边栏评论是否显示时间
duoshuo_asides_title: 1     # 侧边栏评论是否显示标题
duoshuo_asides_admin: 0     # 侧边栏评论是否显示作者评论
duoshuo_asides_length: 45   # 侧边栏评论截取的长度
```

3) 新增source/blog/comments/index.html文件，内容如下

```
---
layout: page
title: "最新评论"
date: 2015-05-06 11:29
comments: false
sharing: false
footer: false
---

<section>
    <ul class="ds-recent-comments"
        data-num-items="{{ site.duoshuo_asides_num }}"
        data-show-avatars="{{ site.duoshuo_asides_avatars }}"
        data-show-time="{{ site.duoshuo_asides_time }}"
        data-show-title="{{ site.duoshuo_asides_title }}"
        data-show-admin="{{ site.duoshuo_asides_admin }}"
        data-excerpt-length="{{ site.duoshuo_asides_length }}"></ul>
    <!--多说js加载开始，一个页面只需要加载一次 -->
    <script type="text/javascript">
        var duoshuoQuery = {short_name:"{{ site.duoshuo_short_name }}"};
        (function() {
            var ds = document.createElement('script');
            ds.type = 'text/javascript';ds.async = true;
            ds.src = 'http://static.duoshuo.com/embed.js';
            ds.charset = 'UTF-8';
            (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(ds);
        })();
    </script>
    <!--多说js加载结束，一个页面只需要加载一次 -->
</section>
```
4) 修改`source/_includes/custom/navigation.html`，在首页新增一个链接：

``` html
<li><a href="{{ root_url }}/blog/comments">最新评论</a></li>
```

**3) 国内访问加速**

由于防火长城的原因，脸书、推、G+的服务在国内都没法用，加上Octopress引用的一些资源都来自Google，
导致默认的网站加载起来非常非常慢（因为加载资源超时拖慢了整体速度）。
因此我们需要对这些引用做一些改动以加快速度。

1) 修改jQuery的源
打开source/_includes/head.html，找到如下字串
``` html
<script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
```
改为
``` html
<script src="//ajax.aspnetcdn.com/ajax/jQuery/jquery-1.9.1.min.js"></script>
```
2）修改字体源

Octopress的英文字体是加载的Google Fonts，我们将其改成国内的CDN源，
打开source/_includes/custom/head.html,
将其中的https://fonts.googleapis.com改为http://fonts.useso.com即可。

3） 关闭Twitter插件

在前面提到的_config.yml中twitter_tweet_butto改为false即可。

#### 中文化
Octopress的页面中很多显示信息都是英文的，我们可以改成中文。直接打开文件改对应字符串保存即可。

* 顶部导航在source/_includes/custom/navigation.html
* 底部版权在source/_includes/custom/footer.html
* 侧边栏的RecentPosts标题在source/_includes/asides/recent_posts.html

这里写到的只是其中的一部分，总之源码都在source目录，需要改动哪个部分找字串然后改了保存就可以汉化了。

### 撰写日志
要发布一篇新文章，在命令行中输入以下命令：
```
rake new_post["mypost"]
```
之后在source/_posts里就可以找到这个文件形如2015-03-10-mypost.markdown，打开编辑就可以写日志内容了。

文件里已经默认有一段基本属性的说明：
```
---
layout: post
title: "mypost"
date: 2015-03-10 03:02:43 +0800
comments: true
categories: octopress
---
```

title默认和你新建文章时的名字相同，这里可以修改。comments表示是否允许评论。categories是自己分类用的目录。
之后文章正文的语法就用Markdown来写。语法参考可见[这里](http://wowubuntu.com/markdown)。

写完文章之后就可以用rake generate生成静态页面。然后通过rake preview来本地预览。

如果你觉得每次都这样生成再预览比较麻烦的话，可以通过配置实时的更新你的修改而不需要重新generate，配置如下：

打开Rakefile文件，找到`task :generate do`这一段，将里面的`"jekyll build"`修改为`"jekyll build --watch"`。

另外还需要注意一下，写好的文章要保存为utf-8无bom格式，否则对中文支持会有问题。

### 发布

#### 申请空间
在Github注册一个账号。登录，创建一个新仓库（Create new repository），
Repository name一定要写成username.github.io的形式，
username是你刚注册的账号名称。其他的不用变。
申请完毕后大约几分钟，域名https://username.github.io就可以访问了。

### 本地配置

* 在Git官网下载git for windows并安装。安装完成后可在cmd输入git version来确认是否安装成功。
* 在octopress的目录下输入命令rake setup_github_pages进行一系列自动的设置。
  期间会让你输入以下你的Github账号和密码。
* 完毕以后就可以准备发布了，生成还是rake generate，发布请执行rake deploy。
* 发布完毕后查看https://username.github.io就可以看到我们的网站已经运行了。
* 发布源码
```
git add .
git commit -m "push source"
git push origin source
```
* 发布时可能会提示你 Updates were rejected because the tip of your current
  branch is behind its remote counterpart.也就是告诉你远程目录已经有内容了，
  此时请到发布的目录_deploy下，打开一个新的cmd，运行命令：git push -f，强制用本地内容覆盖远程即可。

### 加入MathJax，支持 LaTeX 公式
博客里需要插入数学公式，比较好的选择是用LaTeX写数学公式，
但是Octopress默认不支持，需要修改一下Octopress配置，加入MathJax。

**1\. 把 rdiscount 换成 kramdown**

* `gem install kramdown`
* 修改 _config.yml，把 `markdown: rdiscount` 改成 `markdown: kramdown`
* 修改 Gemfile， 把 `gem 'rdiscount'` 改成 `gem 'kramdown', '~> 1.6.0'`，kramdown 版本可以在 gem list 中看到

实际上这一步我们已经在刚开始的时候完成了。所以如果你安装我之前的步骤做过，这部分可以省略了。

**2\. 添加 MathJax 配置**

在 source/_includes/custom/head.html 文件中添加以下内容：

``` html
<!-- MathJax Configuration -->
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  MathMenu: {
      showLocale: false
  },
  jax: ["input/TeX", "output/HTML-CSS"],
  tex2jax: {
    inlineMath: [ ['$', '$'] ],
    displayMath: [ ['$$', '$$']],
    processEscapes: true,
    skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
  },
  messageStyle: "none",
  "HTML-CSS": { preferredFont: "TeX", availableFonts: ["STIX","TeX"] }
});
</script>
<script src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML" type="text/javascript"></script>
```

**3\. 测试**

3.1 整段公式：

```
$$
\begin{align}
\mbox{Union: } & A\cup B = \{x\mid x\in A \mbox{ or } x\in B\} \\
\mbox{Concatenation: } & A\circ B  = \{xy\mid x\in A \mbox{ and } y\in B\} \\
\mbox{Star: } & A^\star  = \{x_1x_2\ldots x_k \mid  k\geq 0 \mbox{ and each } x_i\in A\} \\
\end{align}
$$
```

3.2 行内公式：

```
If $a^2=b$ and $b=2$, then the solution must be
either $a=+\sqrt{2}$ or $a=-\sqrt{2}$.
```

**附：参考文章**

[Mathjax, Kramdown and Octopress](http://www.lucypark.kr/blog/2013/02/25/mathjax-kramdown-and-octopress/)

[Octopress 中使用 Latex 写数学公式](http://dreamrunner.org/blog/2014/03/09/octopresszhong-shi-yong-latexxie-shu-xue-gong-shi/)

### SEO

#### 增加统计工具

博客搭建好了以后，大家一定很想知道每天都有多少的访问量。现在有很多工具都可以帮助我们做这件事，
比如Google Analytics、百度统计、CNZZ 等

其中Google Analytics是Octopress自带的统计工具，使用方式也非常简单，只需要到Google Analytics申请一个app id，
填写到_config.yml文件中的google_analytics_tracking_id后面即可。
但Google Analytics存在翻墙的麻烦，而且百度统计功能也挺齐全，完全能满足我的需求，就选择了百度统计。

集成百度统计方式非常简单：

只需到百度统计官方网站申请一个账号，将获取的代码添加到source/_includes/custom/footer.html中，重新部署即可。

#### 搜索优化

为了让自己搭建的博客更容易被搜索引擎搜到，最好将网站地址提交给各大搜索引擎，下面有两个连接搜集了各个搜索引擎的网站提交入口：

```
http://urlc.cn/tool/addurl.html
http://tool.lusongsong.com/addurl.html
```

我试了下，添加到google以后，搜索关键字的时候自己的博客确实排名靠前了。

光是将网址添加到搜索引擎还不够，你必须得为你的文章添加关键字，才能更好地被引擎搜到，
在创建一篇新文章的时候，生成的makedown文件包含以下内容，以本文举例：

```
---
layout: post
title: "windows7上面使用Octopress搭建GitHub博客"
date: 2015-04-28 11:17:22 +0800
comments: true
categories: octopress github
---
```

实际上我们还可以为其添加以下几项，以本文举例：
```
tags: [octopress, github, seo, blog]
keywords: seo, octopress, analytics, github, blog
description: windows7上面使用Octopress搭建GitHub博客
```
这样更利于搜索引擎抓取到我们的博客。

事实上，如果我们不做上述设置，Octopress会默认将文章的前150个字作为文章的关键字，
供搜索引擎抓取，但那并不一定准确。


### 结语
正如octopress的标语所说的那样：
```
A blogging framework for hackers.
```

这是一个生来就给人折腾的博客……

欢迎来折腾。

### 问题记录
* 在引用代码的时候，如果代码中有`{ %`的标签，需要使用`{ % raw % } { % endraw % }`包围
* 在在代码片段的前后都应该有一个空号
* 引用html代码的时候，<html>标签还得有</html>闭合

[Ruby 1.93]: http://dl.bintray.com/oneclick/rubyinstaller/rubyinstaller-1.9.3-p551.exe?direct
[DevKit]: https://github.com/oneclick/rubyinstaller/wiki/Development-Kit
[DevKit 4.5.2]: https://github.com/downloads/oneclick/rubyinstaller/DevKit-tdm-32-4.5.2-20111229-1559-sfx.exe

