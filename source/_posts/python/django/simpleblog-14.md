---
title: Django1.9开发博客14- 集成Xadmin
date: 2015-08-26 21:45:29 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

xadmin是一个django的管理后台实现，使用了更加灵活的架构设计及Bootstrap UI框架，
目的是替换现有的admin，国人开发，有许多新的特性：

* 兼容 Django Admin
* 使用 Bootstrap 作为 UI 框架
* 编辑页面灵活布局
* 主页面仪表盘及小部件
* 过滤器强化
* 数据导出
* 强大的插件机制

项目主页：<http://sshwsfc.github.io/django-xadmin/>

在线demo: <http://demo.xadmin.io/>
<!-- more -->

### 与django的集成

本篇以simpleblog项目为例，介绍下怎样在django中集成xadmin

#### python2.7环境切换

注意，前面的教程都是在python3.4环境下开放的。
而目前为止xadmin还只能支持python2，所以我们要在此项目基础上新建一个分支py27，
然后我们创建一个python2.7的virtual environment，切换到此环境下面即可。

#### 添加依赖

在requirements.txt中添加如下的依赖，注意：要用到xadmin的django1.9分支

```
django-reversion==1.8.5
xlwt==0.7.5
git+https://github.com/sshwsfc/django-xadmin.git@django1.9
```

#### 修改settings.py

增加xadmin的配置如下

```python
ADMINS = (
    # ('Your Name', 'your_email@example.com'),
)
MANAGERS = ADMINS

# Application definition
INSTALLED_APPS = (
    # ...
    'xadmin',
    'crispy_forms',
    'reversion',
    # ...
)
```

#### 添加/xadmin的链接

修改`mysite/urls.py`如下

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
from django.conf.urls import patterns, include, url
# version模块自动注册需要版本控制的 Model
from xadmin.plugins import xversion
xversion.register_models()

urlpatterns = patterns(
    '',
    url(r'xadmin/', include(xadmin.site.urls), name='xadmin'),
    # ...
)
```

#### 创建adminx.py

在blog/目录下创建adminx.py，内容如下

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
"""
Topic: adminx定制类
Desc :
"""
import xadmin
import xadmin.views as xviews
from .models import Tag, Category, Post, Comment, Evaluate, Page
from xadmin.layout import Main, TabHolder, Tab, Fieldset, Row, Col, AppendedText, Side


class BaseSetting(object):
    enable_themes = True
    use_bootswatch = True


xadmin.site.register(xviews.BaseAdminView, BaseSetting)


class AdminSettings(object):
    # 设置base_site.html的Title
    site_title = '博客管理后台'
    # 设置base_site.html的Footer
    site_footer = 'Winhong Inc.'
    menu_style = 'default'

    # 菜单设置
    def get_site_menu(self):
        return (
            {'title': '博客管理', 'perm': self.get_model_perm(Page, 'change'), 'menus': (
                {'title': '所有页面', 'icon': 'fa fa-vimeo-square'
                    , 'url': self.get_model_url(Page, 'changelist')},
                {'title': '分类目录', 'icon': 'fa fa-vimeo-square'
                    , 'url': self.get_model_url(Category, 'changelist')},
            )},
        )


xadmin.site.register(xviews.CommAdminView, AdminSettings)

xadmin.site.register(Page)
xadmin.site.register(Category)
# xadmin.site.register(Tag)
# xadmin.site.register(Post)
# xadmin.site.register(Comment)
# xadmin.site.register(Evaluate)
```

在这里，我们将所有的model都注册到xadmin中去，这样后台就能自动管理它们了。
并且自定义了后台的一些菜单、标题等等。具体的定制方法可以参考xadmin的官方文档。

#### 添加管理后台链接

在`mysite/templates/mysite/base.html`模板中添加/xamdin的管理后台链接：

```html
<div id="meta-2" class="widget widget_meta">
    <h3>功能</h3>
    <ul>
        @% if user.is_superuser %@
        <li><a href="/xadmin">管理站点</a></li>
        @% endif %@
        @% if user.is_authenticated %@
        <li>
            <a href="@% url 'django.contrib.auth.views.logout' %@">登出</a>
        </li>
        @% else %@
        <li>
            <a href="@% url 'django.contrib.auth.views.login' %@">登录</a>
        </li>
        @% endif %@
        <li><a href="#">文章<abbr title="RSS">RSS</abbr></a></li>
    </ul>
</div>
```

#### 自定义后台登陆页面

新建`mysite/templates/registration/login.html`模板，将xadmin模块中的login.html复制过来，
修改其内容，改成自己想要的形式即可

```html
@% load staticfiles %@
@% load i18n %@
<html>
<head>
    <meta name="viewport"
          content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0">
    <meta name="description" content="">
    <meta name="author" content="">
    <meta name="robots" content="NONE,NOARCHIVE">
    <title>用户登录 - SimpleBlog</title>
    <!--...中间省略...-->
    <script type="text/javascript" src="@% static 'xadmin/vendor/jquery/jquery.js' %@"></script>
</head>
<body class="login">
<div class="container">
    <form method="post" action="@% url 'django.contrib.auth.views.login' %@">
        @% csrf_token %@
        <div class="panel panel-default panel-single" id="panel-login">
            <div class="panel-heading">
                <h2 class="form-signin-heading">请登录</h2>
            </div>
            <!--...中间省略...-->
        </div>
    </form>

</div>
<!--...中间省略...-->
<script type="text/javascript" src="@% static 'xadmin/js/xadmin.main.js' %@"></script>
<script type="text/javascript" src="@% static 'xadmin/js/xadmin.responsive.js' %@"></script>
<script type="text/javascript"
        src="@% static 'xadmin/vendor/jquery-ui/jquery.ui.effect.js' %@"></script>
<script type="text/javascript" src="@% static 'xadmin/js/xadmin.plugin.themes.js' %@"></script>

</body>
</html>
```

这些完成后，我们打开应用，访问`管理站点`链接，应该可以看到如下的登录页面

![](https://xnstatic-1253397658.file.myqcloud.com/dj110.png)

登录后的效果

![](https://xnstatic-1253397658.file.myqcloud.com/dj111.jpg)

