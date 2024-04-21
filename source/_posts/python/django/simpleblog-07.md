---
title: Django1.9开发博客07- 实现功能
date: 2015-08-16 12:02:56 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

到目前为止我们已经完成了一个django应用的所有基础部分。
包括url配置、视图、模型和模板。接下来开始继续完善我们的博客系统了。

首先我们需要一个显示每篇文章的详细页面，对不？
<!-- more -->

## 文章详情

对于首页每一篇文章，我们希望点击标题后可以进入该文章的阅读页面。修改post_list.html中的标题href如下：

```html
<h1><a href="@% url 'blog.views.post_detail' pk=post.pk %@">@@ post.title @@</a></h1>
```

我来详细解释下这个@% raw %@@% url ‘blog.views.post_detail’ pk=post.pk %@，@% %@@% endraw %@
表示使用django模板标签而不是普通的HTML文字，这里我们使用了url标签来生成真正的url链接。
blog.views.post_detail是视图的全路径。

### url配置

我们希望文章详细页面的链接类似这样：http://127.0.0.1:8000/post/1/

修改blog/urls.py为下面的这样：

```python
from django.conf.urls import patterns, include, url
from . import views

urlpatterns = patterns('',
                       url(r'^$', views.post_list),
                       url(r'^post/(?P<pk>[0-9]+)/$', views.post_detail),
                       )
```

这个看起来有点复杂，我们来解释下：

1. ^ 代表的是开始
1. post/表示URL开始必须是post/
1. (?P[0-9]+) – 这部分比较复杂。它表示一个命名参数pk，
   它会捕获url中的这部分然后将它赋值给pk参数传递给视图。
   [0-9]表示这部分必须是数字，+表示至少1个数字，也可以多个数字。
1. / – 然后后面接/
1. $ – URL的结尾

### post_detail视图

现在去访问还会报错，因为我们还没有post_detail这个视图。现在我们开始定义它。

修改文件blog/views.py如下：

```python
from django.shortcuts import render, get_object_or_404


def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})
```

### post_detail模板

然后再增加模板blog/templates/blog/post_detail.html：

```html
@% extends 'blog/base.html' %@

@% block content %@
<div class="date">
    @% if post.published_date %@
    @@ post.published_date @@
    @% endif %@
</div>
<h1>@@ post.title @@</h1>
<p>@@ post.text|linebreaks @@</p>
@% endblock %@
```

这次我们还是采用模板继承方式，这里我们还用到了模板标签if，这是一个条件判断的标签。

OK，一切都已准备就绪，现在打开首页，然后点击任意一篇文章标题看下结果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj017.jpg)

搞定！！

## 创建文章

最后一件事就是要实现文章创建和更新操作，这是博客系统最核心的功能。django自带的admin很酷，但是确很难定制和美化。
forms非常强大和自由，我们可以好好的利用它来实现我们需要的功能。

forms一个很好的特性就是它既能从头定义一个表单，也能创建一个ModelForm来将表单结果保存为一个模型。
而这正是我们想要的功能，我们可以创建一个ModelForm来将表单转换为一个Post模型。

表单对象的定义放在forms.py文件中。我们需要在blog文件夹中创建forms.py文件，结构如下：

    blog
       └── forms.py

在里面写入如下内容：

```python
from django import forms
from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ('title', 'text',)
```

PostForm需要继承自forms.ModelForm，这样django就能实现某些神奇的效果。
在里面我们定义了元类Meta，然后指定model为Post，还有字段为title和text。
因为我们只需要对外暴露标题和内容，至于作者就是登陆用户了，而发布日期和创建日期就是提交时间。

下面我们要做的就是在view中使用我们的form，并在template中显示它。

我们继续走四个步骤：页面上添加链接, 外加"三部曲"。基本上每个新功能的增加时只需要增加这四个东东就行了。

### 增加链接

打开blog/templates/blog/base.html，在名字为page-header的div中添加一个新增文章的链接：

```html
<a href="@% url 'blog.views.post_new' %@" class="top-menu"><span class="glyphicon glyphicon-plus"></span></a>
```

这时候你的base.html应该是这样的：

```html
@% load staticfiles %@
<html>
<head>
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css">
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap-theme.min.css">
    <script src="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/js/bootstrap.min.js"></script>
    <link href="http://fonts.googleapis.com/css?family=Lobster&subset=latin,latin-ext" rel="stylesheet" type="text/css">
    <link rel="stylesheet" href="@% static 'css/blog.css' %@">
    <title>Django Girls Blog</title>
</head>
<body>
<div class="page-header">
    <a href="@% url 'blog.views.post_new' %@" class="top-menu"><span class="glyphicon glyphicon-plus"></span></a>
    <h1><a href="/">Django Girls Blog</a></h1>
</div>
<div class="content">
    <div class="row">
        <div class="col-md-8">
            @% block content %@
            @% endblock %@
        </div>
    </div>
</div>
</body>
</html>
```

### URL

打开blog/urls.py文件，添加一条配置：

```python
url(r'^post/new/$', views.post_new, name='post_new'),
```

现在它的内容应该是这样的：

```python
from django.conf.urls import patterns, include, url
from . import views

urlpatterns = patterns('',
                       url(r'^$', views.post_list),
                       url(r'^post/(?P<pk>[0-9]+)/$', views.post_detail),
                       url(r'^post/new/$', views.post_new, name='post_new'),
                       )
```

### post_new视图

打开文件blog/views.py，先引入PostForm

```python
from .forms import PostForm
```

然后增加视图：

```python
def post_new(request):
    form = PostForm()
    return render(request, 'blog/post_edit.html', {'form': form})
```

### post_edit.html模板

在blog/templates/blog目录新建一个post_edit.html页面，然后写入下列内容：

```html
@% extends 'blog/base.html' %@

@% block content %@
<h1>New post</h1>
<form method="POST" class="post-form">@% csrf_token %@
    @@ form.as_p @@
    <button type="submit" class="save btn btn-default">Save</button>
</form>
@% endblock %@
```

保存后，刷新首页，点击加号那个链接可以看到如下页面：

![](https://xnstatic-1253397658.file.myqcloud.com/dj018.jpg)

但是当你写了文字后点击保存后会发现又跑到这个新建页面来了。
因为这个是POST提交，但是URL还是一样的，又会跑到post_new那个视图中去，
这个视图只做了页面跳转来到了这个新建页面。

那么我们需要修改下post_new视图逻辑了：

在头部先引入下面的依赖：

```python
from django.core.urlresolvers import reverse
from django.shortcuts import redirect
```

然后修改post_new视图如下：

```python
def post_new(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('blog.views.post_detail', pk=post.pk)
    else:
        form = PostForm()
    return render(request, 'blog/post_edit.html', {'form': form})
```

上面表示我添加完一篇文章后自动跳转到文章详情页面去，保存后效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj019.jpg)

### 表单验证

由于我们在Post模型中已经定义了title和text是必需的，django会自动帮我们做验证。
看下如果我们不输入title和text直接提交会怎样：

![](https://xnstatic-1253397658.file.myqcloud.com/dj020.jpg)

django已经自动帮我们做了验证，是不是很酷呢？

## 编辑文章

我们刚刚已经实现了新建文章的功能，那么如果是编辑修改文章呢。接下来我会快速的讲解这个流程，现在你应该是可以看得懂的了。

首先打开blog/templates/blog/post_detail.html，添加一行：

```html
<a class="btn btn-default" href="@% url 'post_edit' pk=post.pk %@">
    <span class="glyphicon glyphicon-pencil"></span></a>
```

现在它的内容是这样的：

```html
@% extends 'blog/base.html' %@

@% block content %@
<div class="date">
    @% if post.published_date %@
    @@ post.published_date @@
    @% endif %@
    <a class="btn btn-default" href="@% url 'post_edit' pk=post.pk %@"><span class="glyphicon glyphicon-pencil"></span></a>
</div>
<h1>@@ post.title @@</h1>
<p>@@ post.text|linebreaks @@</p>
@% endblock %@
```

然后修改blog/urls.py文件，添加一条：

```python
url(r'^post/(?P<pk>[0-9]+)/edit/$', views.post_edit, name='post_edit'),
```

我们会重用模板blog/templates/blog/post_edit.html，
因此只需要修改下view就可以了，打开文件blog/views.py，将下面的内容添加到最后：

```python
def post_edit(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.method == "POST":
        form = PostForm(request.POST, instance=post)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('blog.views.post_detail', pk=post.pk)
    else:
        form = PostForm(instance=post)
    return render(request, 'blog/post_edit.html', {'form': form})
```

这个跟post_new几乎一模一样，先通过主键查到文章：`post = get_object_or_404(Post, pk=pk)`，
然后我们在提交和进入编辑页面的时候都将post作为instance参数传递给PostForm。

OK，现在让我们来测试下效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj021.jpg)

![](https://xnstatic-1253397658.file.myqcloud.com/dj022.jpg)

![](https://xnstatic-1253397658.file.myqcloud.com/dj023.jpg)

完美，恭喜你！你的应用已经变得越来越酷了。

如果你想对django的表单有更深入的了解，
请查阅官方文档：<https://docs.djangoproject.com/en/1.9/topics/forms/>

## 还有一件事

发布到pythonanywhere上去。不要我每次提醒了吧，Enjoy it！

