---
title: Django1.9开发博客09- 用户认证
date: 2015-08-20 21:47:47 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

你应该注意到了一点，当你去新建、修改和删除文章的时候并不需要登录，
这样的话任何浏览网站的用户都能随时修改和删除我的文章。这个可不是我想要的！
<!-- more -->

## 编辑和删除的认证

我们需要保护post_new, post_edit和post_publish这三个视图，只有登录用户才有权去执行。
django为我们提供了很好的帮助类，其实就是利用了python中的decorators技术。
django中认证的装饰器位于模块django.contrib.auth.decorators中，名称叫login_required。

编辑blog/views.py文件，在import部分添加如下的导入语句：

```python
from django.contrib.auth.decorators import login_required
```

然后在post_new, post_edit和post_publish这三个函数上添加@login_required，
类似下面

```python
@login_required
def post_new(request):
    [...]
```

好的，现在你再去访问下`http://localhost:8000/post/new/`，看看有啥变化。

注：如果你仍然能正常进入新建页面，那可能是你之前在admin界面登陆过。
那么你需要先退出，访问`http://localhost:8000/admin/logout/`可以退出，然后你再看下效果。

我刚刚添加的@login_required装饰器检测到你尚未登陆的时候会重定向到login页面，
但是我现在还没有定义login的模板页面，所以这时候会是404错误页面。

## 用户登录

django在用户认证方面做得很好了，我们只需要去使用它就行。

在mysite/urls.py文件中，添加下面一行

```python
url(r'^accounts/login/$', 'django.contrib.auth.views.login')
```

现在这个文件内容如下：

```python
from django.conf.urls import patterns, include, url

from django.contrib import admin

admin.autodiscover()

urlpatterns = patterns('',
                       url(r'^admin/', include(admin.site.urls)),
                       url(r'^accounts/login/$', 'django.contrib.auth.views.login'),
                       url(r'', include('blog.urls')),
                       )
```

然后我们再定义一个登陆页面，创建目录mysite/templates/registration，
并在里面新建模板文件login.html，内容如下：

```html
@% extends "mysite/base.html" %@

@% block content %@

@% if form.errors %@
<p>Your username and password didn't match. Please try again.</p>
@% endif %@

<form method="post" action="@% url 'django.contrib.auth.views.login' %@">
    @% csrf_token %@
    <table>
        <tr>
            <td>@@ form.username.label_tag @@</td>
            <td>@@ form.username @@</td>
        </tr>
        <tr>
            <td>@@ form.password.label_tag @@</td>
            <td>@@ form.password @@</td>
        </tr>
    </table>

    <input type="submit" value="login"/>
    <input type="hidden" name="next" value="@@ next @@"/>
</form>
@% endblock %@
```

你可以看到我们仍然使用到了模板继承。这个时候可以定义一个`mysite/templates/mysite/base.html`，
把`blog/templates/blog/base.html`的内容复制给它即可。

不过我们需要在`mysite/settings.py`中再添加一个urls配置：

```python
LOGIN_REDIRECT_URL = '/'
```

这样的话当用户直接访问login页面后登录成功会重定向到文章列表页面去。

## 改进显示

现在的确只有登录用户才能修改和删除文章，但是未登录用户却能看到这些按钮，
这个是很不好的体验。现在如果是未登录用户的话就把这些按钮给隐藏掉。

因此我们修改`mysite/templates/mysite/base.html`如下：

```html

<body>
<div class="page-header">
    @% if user.is_authenticated %@
    <a href="@% url 'post_new' %@" class="top-menu"><span class="glyphicon glyphicon-plus"></span></a>
    <a href="@% url 'post_draft_list' %@" class="top-menu"><span class="glyphicon glyphicon-edit"></span></a>
    @% else %@
    <a href="@% url 'django.contrib.auth.views.login' %@" class="top-menu"><span
            class="glyphicon glyphicon-lock"></span></a>
    @% endif %@
    <h1><a href="@% url 'blog.views.post_list' %@">Django Girls</a></h1>
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
```

然后修改blog/templates/blog/post_detail.html如下：

```html
@% extends 'blog/base.html' %@

@% block content %@
<div class="date">
    @% if post.published_date %@
    @@ post.published_date @@
    @% else %@
    @% if user.is_authenticated %@
    <a class="btn btn-default" href="@% url 'blog.views.post_publish' pk=post.pk %@">Publish</a>
    @% endif %@
    @% endif %@
    @% if user.is_authenticated %@
    <a class="btn btn-default" href="@% url 'post_edit' pk=post.pk %@"><span class="glyphicon glyphicon-pencil"></span></a>
    <a class="btn btn-default" href="@% url 'post_remove' pk=post.pk %@"><span
            class="glyphicon glyphicon-remove"></span></a>
    @% endif %@
</div>
<h1>@@ post.title @@</h1>
<p>@@ post.text|linebreaks @@</p>
@% endblock %@
```

## 用户注销

当用户登录后显示欢迎语句，Hello ，然后后面跟一个logout链接。还是依靠django帮我们处理logout动作。

修改mysite/templates/mysite/base.html文件如下：

```html

<div class="page-header">
    @% if user.is_authenticated %@
    <a href="@% url 'post_new' %@" class="top-menu"><span class="glyphicon glyphicon-plus"></span></a>
    <a href="@% url 'post_draft_list' %@" class="top-menu"><span class="glyphicon glyphicon-edit"></span></a>
    <p class="top-menu">Hello @@ user.username @@<small>&nbsp;<a href="@% url 'django.contrib.auth.views.logout' %@">Log
        out</a></p>
    @% else %@
    <a href="@% url 'django.contrib.auth.views.login' %@" class="top-menu"><span
            class="glyphicon glyphicon-lock"></span></a>
    @% endif %@
    <h1><a href="@% url 'blog.views.post_list' %@">Django Girls</a></h1>
</div>
```

很显然这时候logout肯定会报错。我们还得做些事情。

对于这方面的详细文档请参考：<https://docs.djangoproject.com/en/1.9/topics/auth/default/>

打开mysite/urls.py文件，添加一个logout配置：

```python
from django.conf.urls import patterns, include, url

from django.contrib import admin

admin.autodiscover()

urlpatterns = patterns('',
                       url(r'^admin/', include(admin.site.urls)),
                       url(r'^accounts/login/$', 'django.contrib.auth.views.login'),
                       url(r'^accounts/logout/$', 'django.contrib.auth.views.logout', {'next_page': '/'}),
                       url(r'', include('blog.urls')),
                       )
```

如果访问网站时出现模板找不到错误，那么你就在`mysite/settings.py`中添加如下配置：

```python
# TEMPLATE_DIRS
TEMPLATE_DIRS = (
    os.path.join(BASE_DIR, 'mysite/templates'),
    os.path.join(BASE_DIR, 'blog/templates'),
)
```

好的，现在你已经可以达成如下的效果了：

1. 需要一个用户名和密码登录系统
1. 在添加/编辑/删除/发布文章的时候需要登录
1. 也能注销

这么说的话，这个博客系统算功能比较完善了！朋友，祝福你。
