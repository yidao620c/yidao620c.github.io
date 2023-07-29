---
title: Django1.9开发博客04- 三部曲
date: 2015-08-09 20:16:02 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

其实在django中实现一个功能只需要三个步骤即可，这里我姑且叫它三部曲。

这三部曲就是：

1. 定义urls映射
1. 定义views
1. 定义templates

<!-- more -->

### 什么是URL？

URL就算一个WEB地址，你在浏览器输入这个地址，然后浏览器返回相应的网页给你。
比如`http://djangogirls.com`是一个URL，而`127.0.0.1:8000`同样也是个URL，默认就是http协议的。

### Django中的URL工作原理

我们打开mysite/urls.py文件，会发现类似下面这样：

```python
from django.conf.urls import patterns, include, url

from django.contrib import admin
admin.autodiscover()

urlpatterns = patterns('',
    # Examples:
    # url(r'^$', 'mysite.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^admin/', include(admin.site.urls)),
)
```

上面的两行注释先不要管，这个以后再用到。
django默认已经为我们添加了admin的URL配置。
当django碰到以admin/开头的URL的时候会去admin.site.urls里面去寻找对应的匹配。
所有和admin相关的urls配置都写在一个文件中，这样就便于管理了。

### 正则表达式

你可以看到上面的url用到了正则表达式，比如’^admin/’、’^$’等等，
django是通过正则式来匹配URL的。关于正则式这里不想展开太多。可以参考相关数据和教程。

### 第一个django url配置

现在我们要将`http://127.0.0.1:8000/`这个首页地址映射到一个显示最新文章列表的页面上面去。一般的博客首页基本都是这样的。

为了保持mysite/urls.py配置文件的简介，我们最好将博客的url配置放到单独的文件中。在mysite/urls.py中去将它引进来即可。

那么你的mysite/urls.py文件现在类似于这样了：

```python
from django.conf.urls import patterns, include, url

from django.contrib import admin
admin.autodiscover()

urlpatterns = patterns('',
    url(r'^admin/', include(admin.site.urls)),
    url(r'', include('blog.urls')),
)
```

### blog.urls

创建文件blog/urls.py，然后加入下列内容

```python
from django.conf.urls import patterns, include, url
from . import views

urlpatterns = patterns('',
    url(r'^$', views.post_list),
)
```

现在我们将r’^$’的url映射到视图views.post_list。

不过你要是现在就访问首页`http://127.0.0.1:8000/`的话会报错的。

![](https://xnstatic-1253397658.file.myqcloud.com/dj006.jpg)

为啥，因为你的视图views.post_list现在没有实现啊，找不到这个方法！

那么接下来我们就来讲解view的实现了。

### 什么是view？

view也叫视图，在django中它存放了实际的业务逻辑。这个跟我们通常所说的MVC中的view是不一样的。

**django的MTV模式**

这里我稍微解释下django的结构，一般我们称之为MTV模式：

1. M 代表模型（Model），即数据存取层。该层处理与数据相关的所有事务：如何存取、如何确认有效性、包含哪些行为以及数据之间的关系等。
1. T 代表模板(Template)，即表现层。该层处理与表现相关的决定：如何在页面或其他类型文档中进行显示。
1. V 代表视图（View），即业务逻辑层。该层包含存取模型及调取恰当模板的相关逻辑。你可以把它看作模型与模板之间的桥梁。

那么通常意义的控制器Controller去哪里了呢，细心的童鞋应该会猜到了，那就是我们上一节所讲的urls.py配置文件。

一句话总结：URLconf+MTV构成了django的总体架构。

**blog/views.py**

这个文件初始内容是这样的：

```python
from django.shortcuts import render

# Create your views here.
```

添加一个最简单的视图：

```python
def post_list(request):

    return render(request, 'blog/post_list.html', {})
```

我们定义了一个方法post_list，它的参数是request，使用render函数返回一个html模板blog/post_list.html。

接下来我们访问下首页，OMG，又出错了：

![](https://xnstatic-1253397658.file.myqcloud.com/dj007.jpg)

这次报的错是模板`blog/post_list.html`找不到。这个是显而易见的，因为我们根本还没有定义这个html模板。

别着急，继续沿着教程往下看就行…

### 什么是模板？

一个模板就是一个使用固定格式呈现动态内容的可重用的文件。
比如你可以使用一个模板来写邮件，每封邮件可能有不同的内容，寄给不同的人，但是它们的格式是一样的。

Django中的模板使用HTML文件，至于神马是HTML，这个去参考下W3C或者自行google下，
不过如果做web开发的人不懂HTML，请不要告诉别人我认识你。^_^

** 第一个模板 **

创建一个模板就是创建一个HTML文件。模板文件存储在blog/templates/blog目录下面，
首先在blog目录下创建templates目录，然后再在templates目录下创建blog目录，至于为啥要这么做，
先不用管，django里面很多目录都是约定好的，这个就跟maven是一样的，约定高于配置。
所以你先照着做就是了。目录结构如下：

```
blog
└───templates
    └───blog
```

然后在blog/templates/blog目录下创建一个post_list.html文件，现在里面还没有内容。

这时候再次访问首页，效果如下：

![](https://xnstatic-1253397658.file.myqcloud.com/dj008.jpg)

一片空白，但没有报错了。

在post_list.html中添加点东西：

```html
<html>
    <p>Hi there!</p>
    <p>It works!</p>
</html>
```

再次访问`http://192.168.203.95:8000/`：

![](https://xnstatic-1253397658.file.myqcloud.com/dj009.jpg)

### 动态模板

不过目前为止我们还只能显示静态的网页。怎样将文章列表在首页显示出来呢？

我们已经有了模型Post，有了模板post_list.html，怎样使得模型数据在模板中显示出来呢，
这个就是视图的功能了，实际上，django中的视图的作用就是连接模型和模板的桥梁。
在视图中，通过QuerySet将数据库中的数据检索出来，然后传递给模板，模板负责显示出来。

首先打开blog/views.py，它目前的内容是这样的：

```python
from django.shortcuts import render

def post_list(request):
    return render(request, 'blog/post_list.html', {})
```

这时候我们将Post模型导入进来

```python
from django.shortcuts import render
from .models import Post
```

注意我们还是用到了相对导入，这是python3的强大功能。

### QuerySet

是时候请出QuerySet了，在模型和ORM小节我们已经介绍过。

现在我们想要将数据库中的文章都检索出来并且按照发布日期逆序排序，使得最新的文章放前面。

```python
from django.shortcuts import render
from .models import Post

def post_list(request):
    posts = Post.objects.filter(published_date__isnull=False).order_by('-published_date')
    return render(request, 'blog/post_list.html', {'posts': posts})
```

注意render函数中最后一个参数{‘posts': posts}，这个就是用来给模板传递数据的。

### 模板标签

HTML页面只识别HTML标签，那么怎样让生成动态的内容呢？答案就是使用django自带的模板标签，
包括了判断、循环、管道等语法。我们已经获取了文章的列表了，那么可以使用for循环来生成相应的HTML页面：

```html

<html>
<head>
    <title>Django Girls Blog</title>
</head>
<body>
<div>
    <h1><a href="/">Django Girls Blog</a></h1>
</div>
@% for post in posts %@
<div>
    <p>published: @@ post.published_date @@</p>
    <h1><a href="">@@ post.title @@</a></h1>
    <p>@@ post.text|linebreaks @@</p>
</div>
@% endfor %@
</body>
</html>
```

在`@% raw %@@% for %@@% endraw %@` 和`@% raw %@@% endfor %@@% endraw %@`之间会循环每个post，然后每次生成一段

现在再次访问首页，效果如下：

![](https://xnstatic-1253397658.file.myqcloud.com/dj010.jpg)

### 别忘了一件事

别忘了把它push到pythonanywhere上面去。

```bash
git add .
git commit -m '动态文章列表首页'
git push origin master
```

恭喜你，目前为止基本的全程已经贯通了。打开admin后添加几篇文章，
记得填上发布日期，再刷新下首页，看会不会显示出来。

好了，这时候你可以出门左拐去小卖部给自己买点棒棒糖奖励下自己了！

不过我们的教程还没完，怎样让页面变得漂亮呢？请看下一节。

